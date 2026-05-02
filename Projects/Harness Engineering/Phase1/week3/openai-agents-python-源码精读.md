# OpenAI Agents SDK 完整源码精读：Guardrail + Handoff + Tracing

> 基于 `openai-agents-python` 仓库 main 分支源码逐行分析，聚焦三大核心子系统：**护栏（Guardrail）**、**交接（Handoff）**、**链路追踪（Tracing）**。

---

## 目录

- [第一部分：Guardrail 护栏系统](#第一部分guardrail-护栏系统)
  - [1.1 架构全景图](#11-架构全景图)
  - [1.2 核心数据类型](#12-核心数据类型)
  - [1.3 InputGuardrail / OutputGuardrail](#13-inputguardrail--outputguardrail)
  - [1.4 装饰器工厂模式](#14-装饰器工厂模式)
  - [1.5 运行时执行引擎](#15-运行时执行引擎)
  - [1.6 ToolGuardrail 工具级护栏](#16-toolguardrail-工具级护栏)
  - [1.7 设计总结](#17-设计总结)
- [第二部分：Handoff 交接系统](#第二部分handoff-交接系统)
  - [2.1 架构全景图](#21-架构全景图)
  - [2.2 HandoffInputData —— 交接数据包](#22-handoffinputdata--交接数据包)
  - [2.3 Handoff 数据类](#23-handoff-数据类)
  - [2.4 handoff() 工厂函数深析](#24-handoff-工厂函数深析)
  - [2.5 History 嵌套与展平机制](#25-history-嵌套与展平机制)
  - [2.6 运行时集成](#26-运行时集成)
  - [2.7 设计总结](#27-设计总结)
- [第三部分：Tracing 链路追踪系统](#第三部分tracing-链路追踪系统)
  - [3.1 分层架构](#31-分层架构)
  - [3.2 全局入口与延迟初始化](#32-全局入口与延迟初始化)
  - [3.3 TraceProvider —— 追踪工厂](#33-traceprovider--追踪工厂)
  - [3.4 Scope —— ContextVar 上下文管理](#34-scope--contextvar-上下文管理)
  - [3.5 Trace 生命周期](#35-trace-生命周期)
  - [3.6 Span 生命周期](#36-span-生命周期)
  - [3.7 SpanData 类型体系](#37-spandata-类型体系)
  - [3.8 创建工厂函数（create.py）](#38-创建工厂函数createpy)
  - [3.9 TracingProcessor 与导出链](#39-tracingprocessor-与导出链)
  - [3.10 BackendSpanExporter —— OpenAI Traces API 导出](#310-backendspanexporter--openai-traces-api-导出)
  - [3.11 TraceContext —— 运行时的追踪生命周期管理](#311-tracecontext--运行时的追踪生命周期管理)
  - [3.12 设计总结](#312-设计总结)

---

## 第一部分：Guardrail 护栏系统

### 1.1 架构全景图

SDK 提供了**三层护栏**，覆盖 Agent 运行的全生命周期：

```
                    ┌──────────────────────┐
                    │   InputGuardrail     │  ← 输入级：Agent 开始前/并行运行
                    │   (guardrail.py)     │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  ToolInputGuardrail   │  ← 工具输入级：工具调用前
                    │ (tool_guardrails.py)  │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │ ToolOutputGuardrail   │  ← 工具输出级：工具调用后
                    │ (tool_guardrails.py)  │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  OutputGuardrail     │  ← 输出级：Agent 最终输出后
                    │   (guardrail.py)     │
                    └──────────────────────┘
```

**源码文件映射**：
- `src/agents/guardrail.py` —— 数据类型定义 + `InputGuardrail` / `OutputGuardrail` 类 + 装饰器
- `src/agents/tool_guardrails.py` —— `ToolInputGuardrail` / `ToolOutputGuardrail` + 行为策略
- `src/agents/run_internal/guardrails.py` —— 运行时调度引擎

### 1.2 核心数据类型

#### GuardrailFunctionOutput（`guardrail.py:19-29`）

```python
@dataclass
class GuardrailFunctionOutput:
    output_info: Any        # 护栏检查的元信息
    tripwire_triggered: bool  # True = 触发阻断
```

这是所有护栏函数的**统一返回类型**。`tripwire_triggered` 是核心控制标志——设为 `True` 即终止 Agent 运行。

#### InputGuardrailResult / OutputGuardrailResult（`guardrail.py:35-68`）

两者结构一致：封装了 `guardrail`（原始护栏对象）+ `output`（执行结果）。`OutputGuardrailResult` 额外携带 `agent` 和 `agent_output`，用于异常抛出时的溯源。

#### ToolGuardrailFunctionOutput（`tool_guardrails.py:59-117`）

工具护栏的输出比输入/输出护栏更精细，支持三种行为策略：

| 行为 | 类型 | 效果 |
|------|------|------|
| `allow` | `AllowBehavior` | 正常执行，不干预 |
| `reject_content` | `RejectContentBehavior` | 拒绝当前工具调用，向模型返回消息，但**继续执行** |
| `raise_exception` | `RaiseExceptionBehavior` | 抛出异常，**终止执行** |

```python
@dataclass
class ToolGuardrailFunctionOutput:
    output_info: Any
    behavior: RejectContentBehavior | RaiseExceptionBehavior | AllowBehavior

    @classmethod
    def allow(cls, output_info=None): ...        # 放行
    @classmethod
    def reject_content(cls, message, ...): ...   # 软拒绝
    @classmethod
    def raise_exception(cls, output_info=None): ...  # 硬拒绝
```

**设计洞见**：`reject_content` 是工具护栏的独特能力——它不像传统护栏那样"一刀切"终止整个流程，而是允许模型在不合规的工具调用后**自我修正**，实现了更优雅的容错。

### 1.3 InputGuardrail / OutputGuardrail

#### InputGuardrail（`guardrail.py:71-131`）

```python
@dataclass
class InputGuardrail(Generic[TContext]):
    guardrail_function: Callable[
        [RunContextWrapper[TContext], Agent[Any], str | list[TResponseInputItem]],
        MaybeAwaitable[GuardrailFunctionOutput],
    ]
    name: str | None = None
    run_in_parallel: bool = True  # 核心配置
```

关键设计：
- **`run_in_parallel`**：默认为 `True`，护栏与 Agent 的 LLM 调用**并发执行**，不增加延迟。设为 `False` 时先跑护栏再启动 Agent。
- **`get_name()`**：名字用于 Tracing Span 命名。未指定时自动取函数名。
- **`run()` 方法**（`guardrail.py:111-130`）：用 `inspect.isawaitable()` 统一处理同步/异步函数，避免 `asyncio.iscoroutinefunction` 对装饰器包装函数的误判。

```python
async def run(self, agent, input, context) -> InputGuardrailResult:
    output = self.guardrail_function(context, agent, input)
    if inspect.isawaitable(output):
        return InputGuardrailResult(guardrail=self, output=await output)
    return InputGuardrailResult(guardrail=self, output=output)
```

#### OutputGuardrail（`guardrail.py:133-185`）

与 `InputGuardrail` 结构对称，区别在于：
- 函数签名接收 `agent_output: Any` 而非 `input`
- **无 `run_in_parallel`** 字段（输出护栏只在 Agent 结束后顺序执行）

### 1.4 装饰器工厂模式

`input_guardrail()` 和 `output_guardrail()` 都使用了**重载 + 双重调用模式**（`guardrail.py:200-343`）：

```python
# 用法 1: 无参数装饰器
@input_guardrail
def my_guardrail(context, agent, input): ...

# 用法 2: 带参数装饰器
@input_guardrail(name="check_toxicity", run_in_parallel=False)
async def my_guardrail(context, agent, input): ...
```

实现原理（`guardrail.py:254-270`）：

```python
def input_guardrail(func=None, *, name=None, run_in_parallel=True):
    def decorator(f):
        return InputGuardrail(
            guardrail_function=f,
            name=name if name else f.__name__,
            run_in_parallel=run_in_parallel,
        )

    if func is not None:
        return decorator(func)  # 无括号调用
    return decorator             # 带括号调用，返回装饰器
```

使用 `typing_extensions.TypeVar` + `@overload` 提供精确的类型推断，调用方可以拿到 `InputGuardrail[TContext]` 而非泛型的 `Any`。

### 1.5 运行时执行引擎（`run_internal/guardrails.py`）

这是护栏系统的**调度核心**，负责并发执行、异常传播和 Stream 集成。

#### 单护栏执行

```python
async def run_single_input_guardrail(agent, guardrail, input, context):
    with guardrail_span(guardrail.get_name()) as span_guardrail:
        result = await guardrail.run(agent, input, context)
        span_guardrail.span_data.triggered = result.output.tripwire_triggered
        return result
```

每个护栏都包裹在 `guardrail_span` 中，将 `triggered` 状态写入 Span，确保护栏触发情况在 trace 中可视化。

#### 并行调度 + 短路机制（`guardrails.py:54-105`）

```python
async def run_input_guardrails(agent, guardrails, input, context):
    guardrail_tasks = [
        asyncio.create_task(run_single_input_guardrail(...))
        for guardrail in guardrails
    ]
    for done in asyncio.as_completed(guardrail_tasks):
        result = await done
        if result.output.tripwire_triggered:
            for t in guardrail_tasks:
                t.cancel()  # 立即取消其余护栏
            raise InputGuardrailTripwireTriggered(result)
        guardrail_results.append(result)
```

核心策略：
1. **所有护栏并发启动**（`asyncio.create_task`）
2. **`asyncio.as_completed`** 按完成顺序逐个检查
3. **任一触发即短路**：取消所有未完成任务 → 附加错误到 Span → 抛出异常
4. **异常传播到 Runner**，由 Runner 决定是否通过 `RunConfig.failure_error_function` 做优雅降级

#### Streaming 模式特殊处理（`guardrails.py:54-105`）

`run_input_guardrails_with_queue` 额外将护栏结果通过 `queue.put_nowait()` 推送到 `RunResultStreaming._input_guardrail_queue`，让上层 stream consumer 能实时感知护栏触发。

### 1.6 ToolGuardrail 工具级护栏

#### 数据模型

```python
@dataclass
class ToolInputGuardrailData:
    context: ToolContext[Any]    # 工具上下文（含 tool_call_id 等）
    agent: Agent[Any]            # 当前 Agent

@dataclass
class ToolOutputGuardrailData(ToolInputGuardrailData):
    output: Any                  # 工具返回的输出
```

`ToolOutputGuardrailData` 继承 `ToolInputGuardrailData`，自然扩展了输出字段。这是面向"在已有的上下文中加上新维度的数据"场景的良好实践。

#### ToolInputGuardrail / ToolOutputGuardrail

与 `InputGuardrail` / `OutputGuardrail` 的设计完全平行，区别仅在于函数签名中不接收 `RunContextWrapper[TContext]` 而接收 `ToolInputGuardrailData` / `ToolOutputGuardrailData`。

### 1.7 设计总结

| 维度 | 策略 |
|------|------|
| **类型安全** | 全程泛型 `Generic[TContext]`，通过 `TypeVar` 推导 |
| **同步/异步统一** | `MaybeAwaitable` + `inspect.isawaitable()` 运行时判断 |
| **并行执行** | Input Guardrails 并发运行，异步完成即检查 |
| **短路机制** | tripwire 触发即取消所有 pending 任务并抛异常 |
| **可观测性** | 每个护栏包裹 `guardrail_span`，triggered 状态写入 trace |
| **Stream 兼容** | 通过 `queue.put_nowait` 将护栏结果推送到异步流 |
| **优雅降级** | `ToolGuardrailFunctionOutput.reject_content` 允许不终止流程的软拒绝 |

---

## 第二部分：Handoff 交接系统

### 2.1 架构全景图

Handoff 是 Multi-Agent 协作的核心机制：**一个 Agent 将对话控制权移交给另一个 Agent**。

```
┌─────────────────────────────────────────────────────┐
│                  Runner.run()                        │
│                                                      │
│  ┌──────────┐    tool_call     ┌──────────────────┐ │
│  │  Triage  │ ──────────────► │  Handoff Tool     │ │
│  │  Agent   │                  │  to BillingAgent  │ │
│  └──────────┘                  └──────┬───────────┘ │
│                                       │              │
│                              ┌────────▼───────────┐ │
│                              │ handoff() factory    │ │
│                              │ - on_invoke_handoff  │ │
│                              │ - input_filter       │ │
│                              │ - history mapper     │ │
│                              └────────┬───────────┘ │
│                                       │              │
│  ┌──────────┐                        │              │
│  │ Billing  │ ◄──────────────────────┘              │
│  │ Agent    │                                       │
│  └──────────┘                                       │
└─────────────────────────────────────────────────────┘
```

**源码文件映射**：
- `src/agents/handoffs/__init__.py` —— 核心 Handoff 数据类 + `handoff()` 工厂函数
- `src/agents/handoffs/history.py` —— 对话历史嵌套/展平/摘要逻辑
- `src/agents/extensions/handoff_filters.py` —— 内置 Handoff input filter 实现
- `src/agents/extensions/handoff_prompt.py` —— 推荐的 Handoff prompt 模板

### 2.2 HandoffInputData —— 交接数据包

```python
@dataclass(frozen=True)
class HandoffInputData:
    input_history: str | tuple[TResponseInputItem, ...]
    pre_handoff_items: tuple[RunItem, ...]
    new_items: tuple[RunItem, ...]
    run_context: RunContextWrapper[Any] | None = None
    input_items: tuple[RunItem, ...] | None = None
```

**三个关键字段的语义**：

| 字段 | 含义 |
|------|------|
| `input_history` | Runner.run() 调用**之前**的历史（例如来自外部 session） |
| `pre_handoff_items` | 触发 handoff 的**那一轮之前**生成的所有 RunItem |
| `new_items` | **当前轮**生成的新 items（含触发 handoff 的 tool_call + handoff tool 的 output） |

`input_items` 是后加的字段：当 `HandoffInputFilter` 想要**过滤掉某些 items 不给下一个模型看**，但又需要保证 session 历史完整性时，filtered items 放在 `input_items`，`new_items` 保持原始不变。

`frozen=True` + `clone(**kwargs)` 方法提供了**不可变数据 + 可控修改**的模式：

```python
def clone(self, **kwargs: Any) -> HandoffInputData:
    return dataclasses_replace(self, **kwargs)
```

### 2.3 Handoff 数据类

```python
@dataclass
class Handoff(Generic[TContext, TAgent]):
    tool_name: str
    tool_description: str
    input_json_schema: dict[str, Any]
    on_invoke_handoff: Callable[[RunContextWrapper[Any], str], Awaitable[TAgent]]
    agent_name: str
    input_filter: HandoffInputFilter | None = None
    nest_handoff_history: bool | None = None
    strict_json_schema: bool = True
    is_enabled: bool | Callable[[...], MaybeAwaitable[bool]] = True
    _agent_ref: weakref.ReferenceType[AgentBase[Any]] | None = None
```

关键设计决策：

**1. Handoff 本身是 Tool**

`Handoff` 不是某种"元数据"，它就是**一个 Tool**。它被注册到 Agent 的 `handoffs` 列表后，SDK 将其转换为工具定义发送给 LLM。当 LLM 调用 `transfer_to_billing_agent` 时，触发 handoff 流程。

`tool_name` 和 `tool_description` 由工厂方法自动生成：

```python
@classmethod
def default_tool_name(cls, agent):
    return _transforms.transform_string_function_style(f"transfer_to_{agent.name}")

@classmethod
def default_tool_description(cls, agent):
    return f"Handoff to the {agent.name} agent to handle the request. {agent.handoff_description or ''}"
```

**2. weakref 避免循环引用**

`_agent_ref` 是 `weakref.ReferenceType`，不在 `__init__` 中设置（`init=False`）。这是因为 `Handoff` 持有对目标 `Agent` 的引用，而 `Agent` 也可能持有其他对象引用 Handoff。weakref 防止 GC 无法回收。

**3. `is_enabled` 的动态开关**

可以是 `bool` 或 `Callable`。动态版本接收 `(RunContextWrapper, AgentBase) -> MaybeAwaitable[bool]`，允许基于运行时状态决定是否暴露某个 handoff。

**4. `strict_json_schema` 强制严格模式**

`handoff()` 工厂函数中（`__init__.py:312`）：

```python
input_json_schema = ensure_strict_json_schema(input_json_schema)
```

自动将 JSON Schema 转为 `additionalProperties: false` + `required` 全覆盖的严格模式，减少 LLM 生成不合规参数的概率。

### 2.4 handoff() 工厂函数深析

`handoff()` 是创建 Handoff 的**唯一推荐入口**（`__init__.py:222-335`）。

#### 四种调用形态

```python
# 1. 最简形态
handoff(billing_agent)

# 2. 带 on_handoff 回调（无输入类型）
handoff(billing_agent, on_handoff=log_handoff)

# 3. 带 on_handoff 回调 + 输入类型
handoff(billing_agent, on_handoff=log_handoff, input_type=HandoffParams)

# 4. 完整配置
handoff(
    billing_agent,
    tool_name_override="escale_to_billing",
    tool_description_override="...",
    on_handoff=my_callback,
    input_type=MyParams,
    input_filter=filter_sensitive_data,
    nest_handoff_history=True,
    is_enabled=lambda ctx, agent: ctx.context.get("plan") == "pro",
)
```

#### 内部 `_invoke_handoff` 闭包

核心逻辑（`__init__.py:275-305`）：

```python
async def _invoke_handoff(ctx, input_json=None) -> Agent[TContext]:
    if input_type is not None and type_adapter is not None:
        # 用 Pydantic TypeAdapter 校验 LLM 生成的 JSON
        validated_input = _json.validate_json(
            json_str=input_json, type_adapter=type_adapter, partial=False
        )
        # 调用用户提供的 on_handoff 回调（接收 context + input）
        await input_func(ctx, validated_input)
    elif on_handoff is not None:
        # 无输入回调（接收 context）
        await no_input_func(ctx)

    return agent  # 始终返回最初绑定的 agent
```

两个关键细节：
- **`partial=False`**：Pydantic 严格校验，不允缺字段
- **无论如何都返回同一个 `agent`**：`on_handoff` 只能做**副作用**（日志、存储更新等），不能动态切换目标 Agent。如需动态路由，应在调用前通过 `is_enabled` 控制可用 handoff 集合。

### 2.5 History 嵌套与展平机制

当 Agent A → Agent B → Agent C 形成交接链时，如果不做处理，每个 Agent 都会把整段对话历史注入 context。`history.py` 的核心功能就是将**前面的对话压缩为摘要消息**。

#### nest_handoff_history() 主流程（`history.py:71-112`）

```
输入
  ├── input_history (外界传入的历史)
  ├── pre_handoff_items (本轮之前的 items)
  └── new_items (本轮新 items)
      │
      ▼
1. _normalize_input_history()  → 将 input_history 转为 list
      │
2. _flatten_nested_history_messages()  → 展平嵌套的 <CONVERSATION HISTORY> 标记
      │
3. 转换 RunItem → plain input（跳过 ToolApprovalItem）
      │
4. 拼接 transcript = flattened_history + pre_items_as_inputs + new_items_as_inputs
      │
5. history_mapper(transcript)  → 生成摘要
      │
输出: HandoffInputData 的 clone，其中 input_history 被替换为摘要
```

#### 默认摘要格式（`history.py:115-158`）

```python
def default_handoff_history_mapper(transcript):
    summary_lines = [f"{idx+1}. {_format_transcript_item(item)}"
                     for idx, item in enumerate(transcript_copy)]
    content = """
For context, here is the conversation so far:
<CONVERSATION HISTORY>
1. user: 我的账单有问题
2. assistant: 让我把您转接给账单专员
...
</CONVERSATION HISTORY>
"""
    return [{"role": "assistant", "content": content}]
```

#### 嵌套展平算法（`history.py:191-223`）

当多级 handoff 发生时（A→B→C），B 收到的历史中已经包含 `<CONVERSATION HISTORY>` 标记。`_flatten_nested_history_messages` 负责解析这些标记，**逐行恢复为原始 items**，然后将它们与新的 transcript 合并后再做摘要，确保历史不会无限嵌套。

```python
def _flatten_nested_history_messages(items):
    flattened = []
    for item in items:
        nested = _extract_nested_history_transcript(item)  # 解析 <CONVERSATION HISTORY>
        if nested is not None:
            flattened.extend(nested)  # 展平插入
        else:
            flattened.append(item)
    return flattened
```

解析逻辑（`history.py:226-246`）支持从摘要行 `"1. user: Hello"` 反向还原为 `{"role": "user", "content": "Hello"}` 格式。

#### 可定制 Wrapper Markers

```python
set_conversation_history_wrappers(start="<HISTORY>", end="</HISTORY>")
reset_conversation_history_wrappers()  # 恢复 <CONVERSATION HISTORY>
```

全局可变的状态通过 `global` 变量管理，适用于自定义 prompt 模板的场景。

#### 去重过滤（`history.py:259-275`）

`_should_forward_pre_item` 和 `_should_forward_new_item` 负责决定哪些 items **不需要再发送给下一个模型**（因为它们已经包含在摘要中）：

- `pre_handoff_items` 中的 assistant 消息、function_call、function_call_output、reasoning → **不转发**
- `new_items` 中有 role 的消息 → **总是转发**

### 2.6 运行时集成

`HandoffInputFilter` 是一个可选的用户注入函数：

```python
HandoffInputFilter = Callable[[HandoffInputData], MaybeAwaitable[HandoffInputData]]
```

用户在 `input_filter` 中可以做：
- 移除敏感数据
- 截断过长历史
- 重新排序 items

注意事项（来自源码注释）：
- Streaming 模式下 filter 函数**不会产生 stream 事件**
- Server-managed conversations（`conversation_id` / `previous_response_id`）**不支持** handoff input filters

### 2.7 设计总结

| 维度 | 策略 |
|------|------|
| **Handoff = Tool** | Handoff 被模型视为可调用的工具，无缝融入 function calling 流程 |
| **不可变数据** | `HandoffInputData` 使用 `frozen=True` + `clone()` 模式 |
| **弱引用** | `weakref.ref` 避免 Agent ↔ Handoff 循环引用 |
| **历史压缩** | `nest_handoff_history` 将长对话转为摘要，避免 token 膨胀 |
| **嵌套展平** | 支持多级 handoff，自动展平 `<CONVERSATION HISTORY>` 标记 |
| **可定制摘要** | `HandoffHistoryMapper` + `set_conversation_history_wrappers` |
| **JSON Schema 严格模式** | `ensure_strict_json_schema` 强制应用，降低 LLM 出错概率 |
| **动态启用/禁用** | `is_enabled` 支持 callable，运行时决定 handoff 是否可用 |

---

## 第三部分：Tracing 链路追踪系统

### 3.1 分层架构

Tracing 系统采用**分层 + 依赖倒置**架构，自底向上：

```
┌──────────────────────────────────────────────────────────┐
│  create.py           │ 工厂函数层（trace, agent_span ...） │
├──────────────────────────────────────────────────────────┤
│  trace.py / spans.py │ 核心模型层（TraceImpl, SpanImpl）  │
├──────────────────────────────────────────────────────────┤
│  scope.py             │ 上下文管理层（ContextVar）         │
├──────────────────────────────────────────────────────────┤
│  provider.py          │ 抽象工厂层（TraceProvider 接口）    │
├──────────────────────────────────────────────────────────┤
│  processor_interface  │ 处理链层（TracingProcessor ABC）   │
├──────────────────────────────────────────────────────────┤
│  processors.py        │ 导出层（BatchProcessor, Exporter） │
├──────────────────────────────────────────────────────────┤
│  setup.py             │ 全局单例 + 延迟初始化              │
└──────────────────────────────────────────────────────────┘
```

### 3.2 全局入口与延迟初始化（`setup.py`）

```python
GLOBAL_TRACE_PROVIDER: TraceProvider | None = None
_GLOBAL_TRACE_PROVIDER_LOCK = threading.Lock()
```

`set_trace_provider()` 和 `get_trace_provider()` 使用 **Double-Checked Locking** 模式：

```python
def get_trace_provider() -> TraceProvider:
    provider = GLOBAL_TRACE_PROVIDER
    if provider is not None:
        return provider  # fast path，无锁

    with _GLOBAL_TRACE_PROVIDER_LOCK:
        provider = GLOBAL_TRACE_PROVIDER
        if provider is None:
            # 延迟创建 DefaultTraceProvider + 自动注册 BatchTraceProcessor
            provider = DefaultTraceProvider()
            provider.register_processor(default_processor())
            GLOBAL_TRACE_PROVIDER = provider
    return provider
```

延迟初始化的设计动机（源码注释）：**避免在 `import agents` 时创建网络客户端和线程**，这对 fork-based 多进程模型（如 gunicorn preload）至关重要。

`atexit.register(_shutdown_global_trace_provider)` 注册清理回调，确保进程退出时 flush 所有 buffered spans。

### 3.3 TraceProvider —— 追踪工厂

`TraceProvider` 是一个 ABC（`provider.py:130-205`），定义了创建 Trace/Span 的完整接口和 ID 生成规范：

```python
class TraceProvider(ABC):
    @abstractmethod
    def create_trace(self, name, trace_id, group_id, metadata, disabled, tracing) -> Trace: ...
    @abstractmethod
    def create_span(self, span_data, span_id, parent, disabled) -> Span: ...
    @abstractmethod
    def get_current_trace(self) -> Trace | None: ...
    @abstractmethod
    def get_current_span(self) -> Span | None: ...
    @abstractmethod
    def set_disabled(self, disabled: bool) -> None: ...
    @abstractmethod
    def gen_trace_id(self) -> str: ...  # trace_{uuid32}
    @abstractmethod
    def gen_span_id(self) -> str: ...   # span_{uuid24}
```

`DefaultTraceProvider`（`provider.py:208-401`）的实现要点：

**1. 双层禁用机制**

```python
# 环境变量（一次性读取，启动后不变）
OPENAI_AGENTS_DISABLE_TRACING = "true" | "1" | "false"

# 编程接口（优先于环境变量）
set_tracing_disabled(True)
```

`_refresh_disabled_flag()` 在每次 `create_trace` / `create_span` 时被调用，但环境变量**只在首次使用后不变化**（缓存），仅 `_manual_disabled` 允许动态切换。

**2. Span 创建时的父级推断**

```python
def create_span(self, span_data, span_id=None, parent=None, disabled=False):
    if not parent:
        current_span = Scope.get_current_span()  # ContextVar 获取
        current_trace = Scope.get_current_trace()
        if current_trace is None:
            return NoOpSpan(span_data)  # 没有活跃 trace，静默降级
        parent_id = current_span.span_id if current_span else None
```

当 `parent=None` 时，自动从 `Scope`（ContextVar）获取当前活跃的 Trace/Span 作为父级。这实现了 **隐式上下文传播**，用户无需手动传递 parent。

### 3.4 Scope —— ContextVar 上下文管理（`scope.py`）

```python
_current_span: ContextVar["Span[Any] | None"] = ContextVar("current_span", default=None)
_current_trace: ContextVar["Trace | None"] = ContextVar("current_trace", default=None)

class Scope:
    @classmethod
    def get_current_span(cls): return _current_span.get()
    @classmethod
    def set_current_span(cls, span): return _current_span.set(span)
    @classmethod
    def reset_current_span(cls, token): _current_span.reset(token)
    # ... 同样的 get/set/reset 用于 trace
```

使用 Python 的 `contextvars.ContextVar` 而非 `threading.local`，天然支持 **asyncio 协程隔离**——每个协程看到的是它自己上下文的 Trace/Span。

`set_current_span` 返回 `Token`，`reset_current_span(token)` 恢复到上一个值，支持 Span 的**嵌套栈语义**。

### 3.5 Trace 生命周期

#### Trace 接口（`traces.py:18-152`）

核心方法：
- `start(mark_as_current)` → `finish(reset_current)`
- 支持 `with trace(...) as t:` 上下文管理器
- `export()` 返回可序列化字典 `{"object": "trace", "id": ..., "workflow_name": ...}`

#### TraceImpl（`traces.py:447-534`）

```python
class TraceImpl(Trace):
    def start(self, mark_as_current=False):
        self._started = True
        self._processor.on_trace_start(self)    # 通知所有 processor
        _mark_trace_id_started(self.trace_id)    # 记录到全局 started set
        if mark_as_current:
            self._prev_context_token = Scope.set_current_trace(self)

    def finish(self, reset_current=False):
        self._processor.on_trace_end(self)       # 通知 processor
        if reset_current:
            Scope.reset_current_trace(self._prev_context_token)
```

**`_mark_trace_id_started` 机制**（`traces.py:247-269`）使用一个 `OrderedDict`（最多 4096 个条目）记录所有已启动的 trace ID。这用于支持 **Trace 恢复**（见下文）。

#### NoOpTrace（`traces.py:371-444`）

当 tracing 被禁用时，所有操作返回 `NoOpTrace`。它实现了完整的 `Trace` 接口但**不记录任何数据**、不通知 processor、`export()` 返回 `None`。

#### ReattachedTrace（`traces.py:272-368`）

支持**从持久化的 `RunState` 恢复 Trace**：

```python
class ReattachedTrace(Trace):
    def start(self, mark_as_current=False):
        self._started = True
        _mark_trace_id_started(self.trace_id)  # 标记但不通知 processor
        if mark_as_current:
            self._prev_context_token = Scope.set_current_trace(self)
```

与 `TraceImpl` 的关键差异：**不调用 `processor.on_trace_start/end`**，避免重复导出已在首次 run 中处理过的事件。

恢复流程由 `create_trace_for_run`（`context.py:47-88`）协调：

```python
def create_trace_for_run(..., reattach_resumed_trace=False, trace_state=None):
    if reattach_resumed_trace and trace_state and _trace_id_was_started(trace_state.trace_id):
        return reattach_trace(trace_state, tracing_api_key=...)  # 复用 live key
    return trace(workflow_name, ...)  # 新建
```

### 3.6 Span 生命周期

#### Span 接口（`spans.py:31-186`）

提供了完整的操作生命周期：

```python
class Span(ABC, Generic[TSpanData]):
    trace_id: str
    span_id: str
    parent_id: str | None
    span_data: TSpanData
    error: SpanError | None
    started_at: str | None
    ended_at: str | None
    tracing_api_key: str | None

    def start(mark_as_current): ...
    def finish(reset_current): ...
    def set_error(error: SpanError): ...
    def export() -> dict[str, Any] | None: ...
```

#### SpanImpl（`spans.py:263-400`）

```python
class SpanImpl(Span[TSpanData]):
    def start(self, mark_as_current=False):
        self._started_at = util.time_iso()
        self._processor.on_span_start(self)
        if mark_as_current:
            self._prev_span_token = Scope.set_current_span(self)

    def finish(self, reset_current=False):
        self._ended_at = util.time_iso()
        self._processor.on_span_end(self)
        if reset_current:
            Scope.reset_current_span(self._prev_span_token)
```

`export()` 方法（`spans.py:372-399`）构建导出 payload：

```python
def export(self):
    payload = {
        "object": "trace.span",
        "id": self.span_id,
        "trace_id": self.trace_id,
        "parent_id": self._parent_id,
        "started_at": self._started_at,
        "ended_at": self._ended_at,
        "span_data": self.span_data.export(),
        "error": self._error,
    }
    # 注入 trace_metadata + span_data.metadata（特定路由 key 如 agent_harness_id）
    if metadata:
        payload["metadata"] = metadata
    return payload
```

#### NoOpSpan（`spans.py:188-261`）

禁用 tracing 时的占位符，所有属性返回空值或 no-op 值。

#### SpanError

```python
class SpanError(TypedDict):
    message: str
    data: dict[str, Any] | None
```

通过 `attach_error_to_span` / `attach_error_to_current_span` 工具函数（`util/_error_tracing.py`）附加到当前活跃 span。

### 3.7 SpanData 类型体系（`span_data.py`）

所有 SpanData 继承自 `SpanData(ABC)`，必须实现 `type` 属性和 `export()` 方法。SDK 定义了 7 种核心类型：

| SpanData | `type` | 关键字段 | 使用场景 |
|----------|--------|---------|---------|
| `AgentSpanData` | `"agent"` | name, handoffs, tools, output_type | Agent 执行生命周期 |
| `TaskSpanData` | `"task"` (custom) | name, usage | 一次 Runner.run() 调用 |
| `TurnSpanData` | `"turn"` (custom) | turn, agent_name, usage | Agent 循环中的一轮 |
| `FunctionSpanData` | `"function"` | name, input, output | 工具/函数调用 |
| `GenerationSpanData` | `"generation"` | input, output, model, usage | LLM 生成调用 |
| `ResponseSpanData` | `"response"` | response, response_id | OpenAI Responses API 响应 |
| `HandoffSpanData` | `"handoff"` | from_agent, to_agent | Agent 交接 |
| `GuardrailSpanData` | `"guardrail"` | name, triggered | 护栏执行 |
| `CustomSpanData` | `"custom"` | name, data | 用户自定义 span |
| `TranscriptionSpanData` | `"transcription"` | model, input, output | 语音转文本 |
| `SpeechSpanData` | `"speech"` | model, input, output | 文本转语音 |
| `SpeechGroupSpanData` | `"speech_group"` | input | 语音请求分组 |
| `MCPListToolsSpanData` | `"mcp_tools"` | server, result | MCP 工具列表 |

**注意 `TaskSpanData` 和 `TurnSpanData` 的导出策略**：它们向外暴露 `type: "custom"`（而非直接暴露 `"task"` / `"turn"`），并在 `data` 子字段中放置 `sdk_span_type` 来标识真实类型。这是因为 OpenAI Traces API 的白名单 span types 有限，`"task"` 和 `"turn"` 不是标准类型。

### 3.8 创建工厂函数（`create.py`）

`create.py` 提供了 12 个工厂函数，每个都是对 `get_trace_provider().create_span(...)` 的薄封装：

```python
def agent_span(name, handoffs=None, tools=None, output_type=None, ...):
    return get_trace_provider().create_span(
        span_data=AgentSpanData(name=name, handoffs=handoffs, tools=tools, output_type=output_type),
        span_id=span_id, parent=parent, disabled=disabled,
    )
```

所有工厂函数的统一模式：
1. 构造对应的 `SpanData` 实例
2. 调用 `get_trace_provider().create_span()`
3. 返回 `Span[TSpanData]`，消费者用 `with` 语句使用

### 3.9 TracingProcessor 与导出链

#### TracingProcessor 接口（`processor_interface.py`）

```python
class TracingProcessor(ABC):
    def on_trace_start(self, trace: Trace) -> None: ...
    def on_trace_end(self, trace: Trace) -> None: ...
    def on_span_start(self, span: Span) -> None: ...
    def on_span_end(self, span: Span) -> None: ...
    def shutdown(self) -> None: ...
    def force_flush(self) -> None: ...
```

#### SynchronousMultiTracingProcessor（`provider.py:44-128`）

使用 **tuple 存储 processors**（而非 list），保证迭代期间的不可变性：

```python
class SynchronousMultiTracingProcessor:
    def __init__(self):
        self._processors: tuple[TracingProcessor, ...] = ()
        self._lock = threading.Lock()

    def add_tracing_processor(self, processor):
        with self._lock:
            self._processors += (processor,)  # 新 tuple = 旧 tuple + (新元素,)
```

每个 processor 调用包裹在 `try/except` 中，确保一个 processor 失败不影响其他。

#### BatchTraceProcessor（`processors.py:464-605`）

默认 processor，采用 **生产者-消费者 + 批量导出** 模式：

```
用户代码（任意线程）
    │ on_span_end(span)
    ▼
Queue[Trace|Span] (max=8192, thread-safe)
    │
    ▼
后台 daemon thread
    │ 每 5s 或 queue 达 70% 时触发
    ▼
BatchTraceProcessor._export_batches()
    │ 每批最多 128 条
    ▼
TracingExporter.export(items)
```

核心设计：
- **`threading.Event`** 作为 shutdown 信号
- **双阈值导出**：定时（5s）+ 队列占比（70%）
- **`_export_lock`** 确保导出串行化
- **Lazy thread start**：`_ensure_thread_started()` 首次入队时才启动 worker，`Double-Checked Locking` 确保只启动一次

```python
def _run(self):
    while not self._shutdown_event.is_set():
        if current_time >= self._next_export_time or queue_size >= self._export_trigger_size:
            self._export_batches(force=False)
            self._next_export_time = time.time() + self._schedule_delay
        else:
            time.sleep(0.2)  # 200ms 轮询间隔
    self._export_batches(force=True)  # 退出前排空
```

### 3.10 BackendSpanExporter —— OpenAI Traces API 导出

默认的 TracingExporter 实现（`processors.py:32-462`），将 spans/traces 通过 HTTP POST 发送到 `https://api.openai.com/v1/traces/ingest`。

#### API Key 解析优先级

```python
@cached_property
def api_key(self):
    return self._api_key or os.environ.get("OPENAI_API_KEY")
```

#### 按 API Key 分组发送

```python
def export(self, items):
    grouped_items: dict[str | None, list[Trace | Span]] = {}
    for item in items:
        key = item.tracing_api_key
        grouped_items.setdefault(key, []).append(item)

    for item_key, grouped in grouped_items.items():
        api_key = item_key or self.api_key
        # 每组使用各自的 key 发送
```

这支持**多租户场景**：不同 trace 可能使用不同的 API key。

#### 指数退避重试（`processors.py:142-182`）

```
HTTP 2xx → 成功，退出
HTTP 4xx → 客户端错误，不重试
HTTP 5xx / 网络异常 → 指数退避 + 10% 随机 jitter
  delay = min(delay * 2, max_delay)
  最多 max_retries 次（默认 3）
```

#### Sanitization —— 适配 OpenAI Traces API 字段限制

`BackendSpanExporter._sanitize_for_openai_tracing_api()` 对 span data 进行预处理（`processors.py:187-245`）：

1. **截断超大字段**：`input` / `output` 字段超过 100KB → 递归截断，优先保留外层键
2. **清理 usage**：非 generation span 的 `usage` 被移除；generation span 只保留 `input_tokens` + `output_tokens`
3. **过滤不可序列化对象**：`float('inf')`, `NaN`, 循环引用 → 替换为 `<type truncated>` 占位符

截断算法核心（`processors.py:311-356`）是递归式的：
- Dict：找最大 value 的 key → 截断该 value → 如果无法截断则**删掉整个 key**
- List：找最大元素的 index → 截断该元素 → 如果无法截断则**删掉该元素**
- String：二分式字符截断 + `"... [truncated]"` 后缀

### 3.11 TraceContext —— 运行时的追踪生命周期管理（`context.py`）

`TraceCtxManager` 是连接 Runner 和 Tracing 系统的桥梁：

```python
class TraceCtxManager:
    def __enter__(self):
        self.trace = create_trace_for_run(
            workflow_name=self.workflow_name,
            trace_id=self.trace_id,
            group_id=self.group_id,
            metadata=self.metadata,
            tracing=self.tracing,
            disabled=self.disabled,
            trace_state=self.trace_state,        # 从 RunState 恢复的 Trace 元信息
            reattach_resumed_trace=...,          # 是否为恢复运行
        )
        if self.trace:
            self.trace.start(mark_as_current=True)
        return self

    def __exit__(self, ...):
        if self.trace:
            self.trace.finish(reset_current=True)
```

关键逻辑 `create_trace_for_run`（`context.py:47-88`）：

1. 如果已经有活跃 trace → 返回 `None`（避免嵌套创建）
2. 如果是**恢复运行**（`reattach_resumed_trace=True`）且 trace_id 在 started set 中 → 创建 `ReattachedTrace`
3. 否则 → 创建全新的 `Trace`

`_trace_state_matches_effective_settings` 验证恢复时的 metadata、group_id、API key 与原始设置一致，使用 `_hash_tracing_api_key` 的 SHA256 指纹来安全比较 API key 而不暴露密钥原文。

### 3.12 设计总结

| 维度 | 策略 |
|------|------|
| **分层解耦** | Trace → Span → SpanData 三层抽象 + ABC 接口，可替换实现 |
| **上下文传播** | Python `ContextVar` 实现跨协程的隐式 Trace/Span 传递 |
| **延迟初始化** | Double-Checked Locking + 首次使用时创建，兼容 fork 模型 |
| **NoOp 模式** | 禁用 tracing 时返回 NoOpTrace/NoOpSpan，业务代码零分支 |
| **批量导出** | 后台 daemon 线程 + 内存 Queue + 阈值触发 + 指数退避重试 |
| **多租户** | 按 `tracing_api_key` 分组，支持不同 trace 使用不同 key |
| **恢复支持** | `ReattachedTrace` 从持久化的 RunState 恢复，不重复通知 processor |
| **安全截断** | 适配 OpenAI Traces API 的 100KB 字段限制 + usage 白名单 |
| **ID 格式** | `trace_{uuid32}`, `span_{uuid24}`, `group_{uuid24}`，格式统一 |
| **元数据路由** | `agent_harness_id` 等特殊 key 从 trace_metadata 路由到 span |

---

## 附录：跨系统交互流程

以下是一次完整 `Runner.run()` 调用中，Guardrail + Handoff + Tracing 的交互时序：

```
1. TraceCtxManager.__enter__()
   └── trace.start() → on_trace_start → 创建根 Trace

2. Runner.run()
   ├── task_span("Runner.run")
   │
   ├── [并行] run_input_guardrails(...)
   │   ├── guardrail_span("check_1") { triggered: false }
   │   └── guardrail_span("check_2") { triggered: false }
   │
   ├── [Agent 循环开始]
   │   ├── turn_span(turn=1, agent="TriageAgent")
   │   │   ├── generation_span(...) → on_span_end
   │   │   └── function_span("tool_call") → on_span_end
   │   │
   │   ├── [LLM 决定 handoff]
   │   │   └── handoff_span(from="TriageAgent", to="BillingAgent")
   │   │       ├── nest_handoff_history(input_data)
   │   │       ├── input_filter(sanitized_data)
   │   │       └── on_invoke_handoff(ctx, input_json)
   │   │
   │   ├── turn_span(turn=2, agent="BillingAgent")
   │   │   ├── generation_span(...)
   │   │   ├── [Tool 输入检查] tool_input_guardrail.run()
   │   │   ├── function_span("lookup_billing")
   │   │   ├── [Tool 输出检查] tool_output_guardrail.run()
   │   │   └── generation_span(...)
   │   │
   │   └── [Agent 输出]
   │       └── run_output_guardrails(...)
   │           └── guardrail_span("validate_output")
   │
   └── result

3. TraceCtxManager.__exit__()
   └── trace.finish() → on_trace_end
       └── BatchTraceProcessor 最终 flush
```

---

*文档生成时间：2026-05-03 | 基于 openai-agents-python main 分支源码*
