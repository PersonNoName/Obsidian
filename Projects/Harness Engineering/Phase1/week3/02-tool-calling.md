# Tool Calling 精读笔记

> 对应源码：`src/agents/tool.py`、`src/agents/run_internal/tool_execution.py`、`src/agents/run_internal/tool_planning.py`、`src/agents/run_internal/run_steps.py`、`src/agents/tool_guardrails.py`

---

## 1. Tool 类型全览

SDK 支持多种 Tool，所有类型都是 `Tool` 联合类型的成员：

| 类 | 说明 | 执行位置 |
|----|------|---------|
| `FunctionTool` | Python 函数工具，最常用 | 本地执行 |
| `ComputerTool` | 计算机控制（截图/点击）| 本地执行 |
| `HostedMCPTool` | OpenAI 托管的 MCP 服务 | 服务端执行 |
| `LocalShellTool` | 本地 shell 命令 | 本地执行 |
| `ShellTool` | Sandbox shell 命令 | 本地/远程执行 |
| `ApplyPatchTool` | 文件 patch 编辑器 | 本地执行 |
| `CustomTool` | 自定义执行器 | 本地执行 |
| 内置托管工具 | `WebSearchTool`、`FileSearchTool`、`CodeInterpreterTool`、`ImageGenerationTool` | 服务端执行 |

---

## 2. FunctionTool：定义与结构

### 2.1 用 `@function_tool` 装饰器定义

```python
from agents import function_tool

@function_tool
def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"{city} 今天晴，25°C"

# 等价于手动创建：
FunctionTool(
    name="get_weather",
    description="获取指定城市的天气。",
    params_json_schema={...},     # 从函数签名自动生成
    on_invoke_tool=...,           # 包装后的函数
)
```

### 2.2 FunctionTool 数据结构

```python
@dataclass
class FunctionTool:
    name: str                        # 工具名（模型调用时的名字）
    description: str                 # 工具描述（进 system prompt）
    params_json_schema: dict         # JSON Schema，约束参数
    on_invoke_tool: Callable         # 真正的执行函数
    strict_json_schema: bool = True  # 是否强制严格 JSON Schema
    failure_error_function: ...      # 自定义错误处理
    is_enabled: ...                  # 动态启用/禁用工具
    needs_approval: ...              # 审批策略（HITL）
    input_guardrails: list[ToolInputGuardrail]
    output_guardrails: list[ToolOutputGuardrail]
    namespace: str | None            # 命名空间，防止名称冲突
```

### 2.3 三种函数签名形式

装饰器支持三种参数形式，SDK 自动识别：

```python
# 1. 无 context
@function_tool
def tool_a(x: int) -> str: ...

# 2. 有 RunContextWrapper（访问 run context）
@function_tool
def tool_b(ctx: RunContextWrapper, x: int) -> str: ...

# 3. 有 ToolContext（含 run context + 工具元数据）
@function_tool
def tool_c(ctx: ToolContext, x: int) -> str: ...
```

---

## 3. 工具调用完整执行流程

### 3.1 模型响应解析阶段

```
模型返回 Response（含 tool_calls）
        ↓
process_model_response()           [turn_resolution.py]
        ↓
按输出 item 类型分类:
  ResponseFunctionToolCall → ToolRunFunction / ToolRunHandoff
  ResponseComputerToolCall → ToolRunComputerAction
  LocalShellCall           → ToolRunLocalShellCall
  McpApprovalRequest       → ToolRunMCPApprovalRequest
  ResponseOutputMessage    → MessageOutputItem（普通文本）
        ↓
组装 ProcessedResponse
```

### 3.2 工具执行阶段

```
execute_tools_and_side_effects()   [turn_resolution.py]
        ↓
调度策略（tool_planning.py）:
  _build_plan_for_fresh_turn()  ← 正常 turn
  _build_plan_for_resume_turn() ← 从中断恢复
        ↓
_execute_tool_plan()
  ├─ execute_function_tool_calls()   [tool_execution.py]
  ├─ execute_computer_actions()
  ├─ execute_handoffs()
  ├─ execute_local_shell_calls()
  ├─ execute_shell_calls()
  ├─ execute_apply_patch_calls()
  └─ execute_mcp_approval_requests() [tool_planning.py]
```

### 3.3 单个 FunctionTool 执行细节

```python
# tool_execution.py: execute_function_tool_calls()
async def execute_function_tool_calls(tool_runs, ...):
    # 1. 解析工具调用参数 (JSON)
    # 2. 检查 is_enabled (动态启用状态)
    # 3. 检查 needs_approval → 若需审批，加入 interruptions
    # 4. 运行 input guardrails (ToolInputGuardrail)
    # 5. 调用 invoke_function_tool() → 执行真正的 Python 函数
    # 6. 运行 output guardrails (ToolOutputGuardrail)
    # 7. 构造 ToolCallOutputItem，放入 new_step_items
    # 8. 若 tools_to_final_output 返回 is_final_output=True → 提前结束循环
```

---

## 4. 工具输出类型

工具函数可以返回多种类型，SDK 会自动序列化：

| 返回类型 | 模型收到的内容 |
|----------|--------------|
| `str` | 文本 |
| `ToolOutputText` | 文本 |
| `ToolOutputImage` | 图像（URL 或 base64 data URL） |
| `ToolOutputFileContent` | 文件内容（base64/URL/file_id） |
| Pydantic BaseModel | JSON 序列化后的文本 |
| `dict` / `list` | JSON 序列化后的文本 |

```python
class ToolOutputImage(BaseModel):
    type: Literal["image"] = "image"
    image_url: str | None = None   # URL 或 data:image/...;base64,...
    file_id: str | None = None
    detail: Literal["low", "high", "auto"] | None = None
```

---

## 5. 工具审批（HITL）

`needs_approval` 参数控制工具是否需要人工审批：

```python
@function_tool(needs_approval=True)   # 始终需要审批
def delete_file(path: str) -> str: ...

@function_tool(needs_approval=lambda ctx, call: call.args.get("path", "").startswith("/etc"))
def read_file(path: str) -> str: ...  # 条件审批
```

**执行流程：**

1. `function_needs_approval()` 判断是否需要审批。
2. 需要 → 创建 `ToolApprovalItem` 加入 `ProcessedResponse.interruptions`。
3. Loop 产生 `NextStepInterruption`，运行保存为 `RunState` 返回。
4. 用户在外部对 `ToolApprovalItem` 设置 `approved=True/False`。
5. 以 `RunState` 重新调用 `Runner.run()`，走 `_build_plan_for_resume_turn()` 恢复。

---

## 6. ToolOrigin：工具来源追踪

```python
class ToolOriginType(Enum):
    FUNCTION = "function"         # 普通 Python 函数
    MCP = "mcp"                   # MCP 服务器工具
    AGENT_AS_TOOL = "agent_as_tool"  # 子 agent 作为工具

@dataclass(frozen=True)
class ToolOrigin:
    type: ToolOriginType
    mcp_server_name: str | None = None
    agent_name: str | None = None
    agent_tool_name: str | None = None
```

每个 `ToolCallOutputItem` 都携带 `ToolOrigin`，用于追踪调试。

---

## 7. Tool Guardrails（工具级护栏）

独立于 Agent 级 Guardrail，作用于单个工具调用：

```python
from agents import ToolInputGuardrail, ToolOutputGuardrail

@ToolInputGuardrail
async def check_input(ctx, call_data: ToolInputGuardrailData) -> GuardrailFunctionOutput:
    # call_data.tool_name, call_data.tool_input (JSON string)
    dangerous = "rm -rf" in call_data.tool_input
    return GuardrailFunctionOutput(output_info=None, tripwire_triggered=dangerous)

@function_tool(input_guardrails=[check_input])
def run_shell(cmd: str) -> str: ...
```

触发时抛出 `ToolInputGuardrailTripwireTriggered` / `ToolOutputGuardrailTripwireTriggered`。

---

## 8. Agent as Tool

可以将一个 `Agent` 包装成工具，供父 agent 调用：

```python
from agents import Agent

sub_agent = Agent(name="Translator", instructions="将文本翻译为英文")
parent_agent = Agent(
    name="Orchestrator",
    tools=[sub_agent.as_tool(tool_name="translate", tool_description="翻译文本")]
)
```

`as_tool()` 内部创建了一个 `FunctionTool`，其执行函数会：
1. 以子 agent 作为 `starting_agent` 调用 `Runner.run()`。
2. 返回子 agent 的最终输出作为工具结果。

---

## 9. 并发执行策略

`tool_planning.py` 中，同一 Turn 中的多个函数工具调用**默认并发执行**（`asyncio.gather`）：

```python
# _execute_tool_plan() 内部
tasks = [
    asyncio.create_task(execute_single_function_tool(run))
    for run in function_tool_runs
]
results = await asyncio.gather(*tasks, return_exceptions=True)
```

注意事项：
- 工具函数之间**不应有副作用依赖**，否则并发会导致竞态。
- `ToolContext` 提供 `tool_call_id` 用于关联调用。

---

## 10. 工具错误处理

工具执行出错时，默认行为：

```python
# 默认: 将异常信息作为工具结果返回给模型（而不是让整个 run 崩溃）
DEFAULT_TOOL_ERROR = "An error occurred while running the tool: {error}"

# 自定义错误格式
@function_tool(failure_error_function=lambda ctx, exc: f"自定义错误: {exc}")
def my_tool(): ...

# 关闭自动错误捕获，让异常传播
@function_tool(failure_error_function=None)
def strict_tool(): ...
```

---

## 11. 关键文件速查

| 文件 | 职责 |
|------|------|
| `src/agents/tool.py` | 所有 Tool 类定义、`@function_tool` 装饰器、`invoke_function_tool()` |
| `src/agents/function_schema.py` | 从 Python 函数签名自动生成 JSON Schema |
| `src/agents/run_internal/tool_execution.py` | 工具执行实现、审批逻辑、错误处理 |
| `src/agents/run_internal/tool_planning.py` | 工具调用计划构建、并发调度 |
| `src/agents/run_internal/run_steps.py` | `ToolRun*` 数据结构、`ProcessedResponse` |
| `src/agents/tool_guardrails.py` | `ToolInputGuardrail`、`ToolOutputGuardrail` |
| `src/agents/tool_context.py` | `ToolContext`（函数工具运行时上下文） |
| `src/agents/run_internal/tool_actions.py` | `ComputerAction`、`ShellAction`、`ApplyPatchAction` |
