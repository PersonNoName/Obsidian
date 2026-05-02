# Tracing 精读笔记

> 对应源码：`src/agents/tracing/`（全目录）、`src/agents/run_internal/run_loop.py` 中 tracing 集成部分

---

## 1. 整体层次结构

```
TraceProvider（全局单例，setup.py）
  └─ 管理 TracingProcessor 列表
       └─ 每个 Processor 处理 Trace/Span 事件

Trace（一次完整 workflow 的容器）
  └─ Span（单个操作的记录单元）
       ├─ SpanData（携带具体业务数据）
       └─ 支持嵌套（parent/child span）
```

---

## 2. Trace：workflow 级容器

### 2.1 创建与使用

```python
from agents.tracing import trace

# 作为 context manager（推荐）
with trace("Customer Service Query") as t:
    result = await Runner.run(agent, user_input)

# 带分组和元数据
with trace(
    "Order Processing",
    group_id="session_abc123",         # 关联同一对话的多次 trace
    metadata={"user_id": "u_456"},
) as t:
    ...
```

### 2.2 Trace 属性

```python
t.trace_id   # "trace_<32位随机字母数字>" 全局唯一
t.name       # workflow 名称
t.export()   # → dict，用于传输到后端
```

### 2.3 与 Agent Loop 集成

SDK 会在 `AgentRunner.run()` 入口处**自动**创建 Trace，无需手动调用：

```python
# run_internal/run_loop.py (简化)
with task_span(name=task_name) as task_sp:
    # 整个 run 期间所有 spans 都挂在这个 task span 下
    async with create_trace_for_run(run_config, ...) as run_trace:
        while current_turn < max_turns:
            with turn_span(turn=current_turn, agent_name=agent.name) as turn_sp:
                with agent_span(name=agent.name, tools=[...]) as agent_sp:
                    # 调用模型（生成 generation/response span）
                    # 执行工具（生成 function span）
                    # handoff（生成 handoff span）
                    # guardrail（生成 guardrail span）
```

---

## 3. Span 类型与 SpanData

SDK 内置以下 Span 类型，通过工厂函数创建：

| Span 类型 | 工厂函数 | SpanData 类 | 包含信息 |
|-----------|---------|------------|---------|
| **Task Span** | `task_span()` | `TaskSpanData` | 整个 Runner.run() 调用，含 usage 汇总 |
| **Turn Span** | `turn_span()` | `TurnSpanData` | 单次 agent loop turn，含 turn 编号 |
| **Agent Span** | `agent_span()` | `AgentSpanData` | 单个 agent 执行，含工具列表、handoff 列表、output_type |
| **Generation Span** | `generation_span()` | `GenerationSpanData` | LLM 调用，含输入/输出/model/usage |
| **Response Span** | `response_span()` | `ResponseSpanData` | OpenAI Responses API 调用，含 response_id |
| **Function Span** | `function_span()` | `FunctionSpanData` | 工具函数调用，含名称/输入/输出 |
| **Handoff Span** | `handoff_span()` | `HandoffSpanData` | Agent 切换，含 from/to agent 名称 |
| **Guardrail Span** | `guardrail_span()` | `GuardrailSpanData` | 护栏执行，含名称和 `triggered` 字段 |
| **Custom Span** | `custom_span()` | `CustomSpanData` | 用户自定义，含 name 和任意 data dict |
| **MCP List Tools Span** | `mcp_tools_span()` | `MCPListToolsSpanData` | MCP 工具列表获取 |

### 3.1 SpanData 示例：AgentSpanData

```python
class AgentSpanData(SpanData):
    __slots__ = ("name", "handoffs", "tools", "output_type", "metadata")
    
    def export(self) -> dict:
        return {
            "type": "agent",
            "name": self.name,
            "handoffs": self.handoffs,   # 可用 handoff 的名称列表
            "tools": self.tools,         # 可用工具的名称列表
            "output_type": self.output_type,
        }
```

### 3.2 Span 嵌套层次示例

```
task_span ("Runner.run")
  └─ turn_span (turn=0, agent="Support")
       ├─ agent_span ("Support", tools=["search", "respond"])
       │    ├─ response_span (response_id="resp_xxx")
       │    ├─ function_span ("search", input="...", output="...")
       │    └─ function_span ("respond", input="...", output="...")
       └─ guardrail_span ("topic_check", triggered=False)
  └─ turn_span (turn=1, agent="Support")
       └─ agent_span (...)
            └─ response_span (...)
```

---

## 4. TracingProcessor：插件接口

```python
from agents.tracing import TracingProcessor

class MyProcessor(TracingProcessor):
    def on_trace_start(self, trace: Trace) -> None:
        print(f"Trace 开始: {trace.name}")
    
    def on_trace_end(self, trace: Trace) -> None:
        # 在这里导出完整 trace 到自己的后端
        data = trace.export()
        my_backend.send(data)
    
    def on_span_start(self, span: Span) -> None:
        pass
    
    def on_span_end(self, span: Span) -> None:
        # 细粒度：每个 span 结束时单独处理
        if span.span_data.type == "generation":
            metrics.record_llm_call(span.span_data.usage)
    
    def shutdown(self) -> None:
        # 清理资源，flush 队列
        pass
    
    def force_flush(self) -> None:
        pass

# 注册自定义 Processor
from agents.tracing import add_trace_processor
add_trace_processor(MyProcessor())
```

---

## 5. 内置 Processor 与 Exporter

### 5.1 BatchTraceProcessor（默认）

异步批量发送，生产环境默认使用：

```
on_span_end() → 放入内存队列
后台线程定时批量 → TracingExporter.export()
```

### 5.2 BackendSpanExporter（默认 Exporter）

将 trace/span 数据 POST 到 `https://api.openai.com/v1/traces/ingest`，支持：
- 自动读取 `OPENAI_API_KEY` 环境变量
- 指数退避重试（最多 3 次）
- 字段超长截断（100,000 bytes）
- 连接池复用（httpx.Client）

### 5.3 ConsoleSpanExporter（调试用）

```python
from agents.tracing.processors import ConsoleSpanExporter, BatchTraceProcessor
from agents.tracing import add_trace_processor

add_trace_processor(BatchTraceProcessor(ConsoleSpanExporter()))
```

---

## 6. 自定义 Span（用户代码中埋点）

```python
from agents.tracing import custom_span

async def my_workflow():
    with custom_span("data_validation", {"records": 100}) as span:
        result = validate_data(records)
        span.span_data.data["valid_count"] = result.valid_count
        if result.has_errors:
            span.set_error({"message": "验证失败", "data": {"errors": result.errors}})
```

---

## 7. 禁用 Tracing

```python
# 方法 1：全局禁用
from agents import set_tracing_disabled
set_tracing_disabled(True)

# 方法 2：单次 run 禁用（通过 RunConfig）
from agents import RunConfig
result = await Runner.run(agent, input, run_config=RunConfig(tracing_disabled=True))

# 方法 3：创建禁用的 trace
with trace("my_workflow", disabled=True):
    ...
```

---

## 8. TraceProvider 与初始化

```python
# setup.py: get_trace_provider() 惰性初始化
GLOBAL_TRACE_PROVIDER: TraceProvider | None = None  # 全局单例

def get_trace_provider() -> TraceProvider:
    if GLOBAL_TRACE_PROVIDER is not None:
        return GLOBAL_TRACE_PROVIDER
    # 首次访问时创建 DefaultTraceProvider
    # 并注册 default_processor()（BatchTraceProcessor + BackendSpanExporter）
    # 同时注册 atexit handler 确保程序退出时 flush
```

替换 Provider（高级用法）：

```python
from agents.tracing import set_trace_provider
from agents.tracing.provider import DefaultTraceProvider

custom_provider = DefaultTraceProvider()
custom_provider.register_processor(MyProcessor())
set_trace_provider(custom_provider)
```

---

## 9. Span 的线程安全与 contextvars

Span 的当前激活状态通过 `contextvars.ContextVar` 存储，天然支持 asyncio 协程隔离：

```python
# spans.py 内部简化
_current_span: contextvars.ContextVar[Span | None] = contextvars.ContextVar(
    "current_span", default=None
)

def start(self, mark_as_current: bool = False):
    if mark_as_current:
        self._prev_span_token = _current_span.set(self)
```

这意味着：
- 同一 event loop 中不同协程可以并行持有不同的 "current span"。
- `with function_span(...)` 会自动成为当前 agent_span 的子 span。

---

## 10. 关键文件速查

| 文件 | 职责 |
|------|------|
| `src/agents/tracing/create.py` | 所有 Span/Trace 工厂函数（`trace()`、`agent_span()`、`function_span()` 等）|
| `src/agents/tracing/traces.py` | `Trace` 抽象类、`TraceState`（序列化）|
| `src/agents/tracing/spans.py` | `Span` 抽象类、`NoOpSpan`（tracing 禁用时用）|
| `src/agents/tracing/span_data.py` | 所有 `SpanData` 子类（数据结构定义）|
| `src/agents/tracing/processor_interface.py` | `TracingProcessor`、`TracingExporter` 接口 |
| `src/agents/tracing/processors.py` | `BatchTraceProcessor`、`BackendSpanExporter`、`ConsoleSpanExporter` |
| `src/agents/tracing/provider.py` | `TraceProvider`、`DefaultTraceProvider` |
| `src/agents/tracing/setup.py` | 全局 Provider 单例管理、惰性初始化 |
| `src/agents/tracing/scope.py` | `contextvars` 管理当前 trace/span |
| `src/agents/tracing/context.py` | `TraceCtxManager`、`create_trace_for_run()` |
| `src/agents/tracing/model_tracing.py` | 模型调用的 tracing 实现（generation/response span）|
