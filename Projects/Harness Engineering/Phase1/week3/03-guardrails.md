# Guardrails 精读笔记

> 对应源码：`src/agents/guardrail.py`、`src/agents/tool_guardrails.py`、`src/agents/run_internal/guardrails.py`

---

## 1. 护栏的两个维度

SDK 提供**两个层次**的护栏：

| 层次 | 类 | 作用时机 | 触发异常 |
|------|----|---------|---------|
| Agent 输入护栏 | `InputGuardrail` | 第一个 Turn，调用模型前（可并行） | `InputGuardrailTripwireTriggered` |
| Agent 输出护栏 | `OutputGuardrail` | 产生 Final Output 后 | `OutputGuardrailTripwireTriggered` |
| 工具输入护栏 | `ToolInputGuardrail` | 工具调用前，解析参数后 | `ToolInputGuardrailTripwireTriggered` |
| 工具输出护栏 | `ToolOutputGuardrail` | 工具执行后，结果送给模型前 | `ToolOutputGuardrailTripwireTriggered` |

---

## 2. InputGuardrail（Agent 输入护栏）

### 2.1 核心数据结构

```python
@dataclass
class GuardrailFunctionOutput:
    output_info: Any          # 可选的诊断信息
    tripwire_triggered: bool  # True = 触发，停止 agent 执行

@dataclass
class InputGuardrailResult:
    guardrail: InputGuardrail[Any]
    output: GuardrailFunctionOutput

@dataclass
class InputGuardrail(Generic[TContext]):
    guardrail_function: Callable[
        [RunContextWrapper, Agent, str | list[InputItem]],
        MaybeAwaitable[GuardrailFunctionOutput]
    ]
    name: str | None = None
    run_in_parallel: bool = True  # 默认与 agent 并行运行
```

### 2.2 用 `@input_guardrail` 装饰器

```python
from agents import input_guardrail, GuardrailFunctionOutput

@input_guardrail
async def check_topic(ctx: RunContextWrapper, agent: Agent, input) -> GuardrailFunctionOutput:
    is_off_topic = "竞争对手" in str(input)
    return GuardrailFunctionOutput(
        output_info={"reason": "包含竞争对手信息"},
        tripwire_triggered=is_off_topic,
    )

# 带参数的装饰器
@input_guardrail(name="topic_guard", run_in_parallel=False)
async def check_topic_v2(ctx, agent, input) -> GuardrailFunctionOutput: ...
```

### 2.3 挂载到 Agent

```python
agent = Agent(
    name="Support",
    input_guardrails=[check_topic],
)

# 也可在 RunConfig 中全局设置
run_config = RunConfig(input_guardrails=[check_topic])
```

---

## 3. OutputGuardrail（Agent 输出护栏）

```python
from agents import output_guardrail

@output_guardrail
async def check_output(
    ctx: RunContextWrapper,
    agent: Agent,
    output: str           # 类型取决于 agent.output_type
) -> GuardrailFunctionOutput:
    contains_pii = detect_pii(output)
    return GuardrailFunctionOutput(
        output_info={"pii_found": contains_pii},
        tripwire_triggered=contains_pii,
    )

agent = Agent(
    name="DataAgent",
    output_guardrails=[check_output],
)
```

> **注意**：Output Guardrail 函数的第三个参数类型 = `agent.output_type`，若是 `str` 则接收字符串，若是 Pydantic model 则接收该 model 实例。

---

## 4. 执行机制：并行 vs 串行

### 4.1 `run_in_parallel=True`（默认）

```python
# run_internal/guardrails.py: run_input_guardrails()
guardrail_tasks = [
    asyncio.create_task(run_single_input_guardrail(agent, g, input, ctx))
    for g in guardrails
]
# 用 asyncio.as_completed 收集结果
# 第一个触发的 → 立即 cancel 剩余任务 → 抛出异常
for done in asyncio.as_completed(guardrail_tasks):
    result = await done
    if result.output.tripwire_triggered:
        for t in guardrail_tasks: t.cancel()
        raise InputGuardrailTripwireTriggered(result)
```

### 4.2 `run_in_parallel=False`

当 guardrail 设置 `run_in_parallel=False` 时，它在**模型调用之前**串行执行，而不是与模型调用并发。  
这保证了护栏有机会在调用 LLM 之前就拦截请求（适合计费类或高安全要求场景）。

---

## 5. 流式模式下的 Guardrail

流式模式中，输入护栏通过单独的 `asyncio.Queue` 与主流程通信：

```python
# run_internal/guardrails.py: run_input_guardrails_with_queue()
async def run_input_guardrails_with_queue(
    agent, guardrails, input, context, streamed_result, parent_span
):
    queue = streamed_result._input_guardrail_queue
    # 并发运行所有 guardrails
    # 每个结果（触发或不触发）都 put_nowait 进队列
    # 触发时：立即 cancel 其余任务，通知流式结果对象
```

调用方通过：
```python
result = Runner.run_streamed(agent, input)
async for event in result.stream_events():
    # InputGuardrailResult 事件会在流中出现
    ...
```

---

## 6. ToolInputGuardrail（工具输入护栏）

比 Agent Guardrail 更细粒度，作用于单个工具调用。

### 6.1 输出结构：三种行为

```python
@dataclass
class ToolGuardrailFunctionOutput:
    output_info: Any
    behavior: AllowBehavior | RejectContentBehavior | RaiseExceptionBehavior
    
    # 工厂方法
    @classmethod
    def allow(cls, output_info=None) -> ToolGuardrailFunctionOutput: ...
    
    @classmethod
    def reject_content(cls, message: str, output_info=None) -> ...:
        # 用 message 替换工具输出，告知模型，但不中止 agent
        ...
    
    @classmethod
    def raise_exception(cls, output_info=None) -> ...:
        # 抛出异常，中止整个 agent run
        ...
```

### 6.2 示例

```python
from agents import ToolInputGuardrail, ToolGuardrailFunctionOutput, ToolInputGuardrailData

@ToolInputGuardrail
async def block_dangerous_commands(
    ctx: ToolContext,
    data: ToolInputGuardrailData,
) -> ToolGuardrailFunctionOutput:
    if "rm -rf" in data.tool_input:
        return ToolGuardrailFunctionOutput.reject_content(
            message="危险命令已被拦截",
            output_info={"blocked_input": data.tool_input},
        )
    return ToolGuardrailFunctionOutput.allow()

@function_tool(input_guardrails=[block_dangerous_commands])
def run_shell(cmd: str) -> str:
    import subprocess
    return subprocess.check_output(cmd, shell=True).decode()
```

---

## 7. ToolOutputGuardrail（工具输出护栏）

在工具执行**完成后**、结果发送给模型之前运行：

```python
from agents import ToolOutputGuardrail, ToolOutputGuardrailData

@ToolOutputGuardrail
async def redact_secrets(
    ctx: ToolContext,
    data: ToolOutputGuardrailData,
) -> ToolGuardrailFunctionOutput:
    output_str = str(data.tool_output)
    if "SECRET_KEY" in output_str:
        return ToolGuardrailFunctionOutput.reject_content(
            message="[输出已脱敏]",
        )
    return ToolGuardrailFunctionOutput.allow()
```

---

## 8. Guardrail 在 Trace 中的体现

每个 Guardrail 执行都会创建一个 `guardrail_span`：

```python
# run_internal/guardrails.py: run_single_input_guardrail()
async def run_single_input_guardrail(...):
    with guardrail_span(guardrail.get_name()) as span:
        result = await guardrail.run(agent, input, context)
        span.span_data.triggered = result.output.tripwire_triggered
        return result
```

在 OpenAI Platform 的 Traces 面板中，可以看到每个 guardrail 的执行时间和是否触发。

---

## 9. 触发后的异常处理

```python
from agents import Runner
from agents.exceptions import InputGuardrailTripwireTriggered, OutputGuardrailTripwireTriggered

try:
    result = await Runner.run(agent, user_input)
except InputGuardrailTripwireTriggered as e:
    # e.result 是 InputGuardrailResult
    print(f"输入护栏 [{e.result.guardrail.get_name()}] 已触发")
    print(f"诊断信息: {e.result.output.output_info}")
except OutputGuardrailTripwireTriggered as e:
    # e.result 是 OutputGuardrailResult
    print(f"输出护栏 [{e.result.guardrail.get_name()}] 已触发")
```

---

## 10. 优先级与配置来源

护栏可以来自多个地方，**最终合并执行**：

```python
# 1. Agent 级别（最常用）
agent = Agent(
    input_guardrails=[guardrail_a],
    output_guardrails=[guardrail_b],
)

# 2. RunConfig 全局级别（对所有 agent 生效）
run_config = RunConfig(
    input_guardrails=[global_guardrail],
    output_guardrails=[global_output_guardrail],
)

# 执行时合并：agent.input_guardrails + run_config.input_guardrails
```

---

## 11. 关键文件速查

| 文件 | 职责 |
|------|------|
| `src/agents/guardrail.py` | `InputGuardrail`、`OutputGuardrail`、`GuardrailFunctionOutput`、装饰器 |
| `src/agents/tool_guardrails.py` | `ToolInputGuardrail`、`ToolOutputGuardrail`、`ToolGuardrailFunctionOutput` |
| `src/agents/run_internal/guardrails.py` | 运行时执行逻辑：并发调度、tripwire 检测、流式集成 |
| `src/agents/exceptions.py` | `InputGuardrailTripwireTriggered`、`OutputGuardrailTripwireTriggered`、`ToolInputGuardrailTripwireTriggered` |
| `src/agents/tracing/span_data.py` | `GuardrailSpanData`（含 `triggered` 字段） |
