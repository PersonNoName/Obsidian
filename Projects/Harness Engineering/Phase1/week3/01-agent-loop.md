# Agent Loop 精读笔记

> 对应源码：`src/agents/run.py`、`src/agents/run_internal/run_loop.py`、`src/agents/run_internal/turn_resolution.py`、`src/agents/run_internal/run_steps.py`

---

## 1. 整体架构

```
用户调用 Runner.run() / Runner.run_sync() / Runner.run_streamed()
        ↓
  AgentRunner (实验性内部类，全局单例 DEFAULT_AGENT_RUNNER)
        ↓
  外层循环: while current_turn < max_turns
        ↓
    run_single_turn()  ──► get_new_response() (调用模型)
        ↓
    process_model_response() → ProcessedResponse
        ↓
    execute_tools_and_side_effects()
        ↓
    get_single_step_result_from_response() → SingleStepResult
        ↓
  根据 next_step 类型决定是否继续循环
```

---

## 2. 入口：Runner 与 AgentRunner

`Runner` 是公开的静态门面，三个类方法都委托给全局 `DEFAULT_AGENT_RUNNER`（`AgentRunner` 实例）：

| 方法 | 返回 | 说明 |
|------|------|------|
| `Runner.run()` | `RunResult` | 异步，等待完成 |
| `Runner.run_sync()` | `RunResult` | 同步包装，内部用 `asyncio.run()` |
| `Runner.run_streamed()` | `RunResultStreaming` | 异步，流式事件 |

`AgentRunner` 是真正的执行体（`run.py` 中 `AgentRunner.run()`），逻辑如下：

```python
# 简化伪代码
async def run(self, starting_agent, input, **kwargs):
    # 1. 初始化上下文 (RunContextWrapper)
    # 2. 处理 RunState 恢复
    # 3. 准备 Session / Conversation 跟踪
    # 4. 创建 Trace
    # 5. 运行输入 Guardrails
    # 6. 进入主循环 while current_turn < max_turns:
    #      a. run_single_turn(agent, input, ...)
    #      b. 检查 next_step 类型
    #      c. NextStepHandoff  → 切换 agent，继续
    #      d. NextStepRunAgain → 继续同一 agent
    #      e. NextStepFinalOutput → 运行输出 Guardrails，返回
    #      f. NextStepInterruption → 保存 RunState，返回中断结果
    # 7. 超过 max_turns → 抛出 MaxTurnsExceeded（或调用 error_handler）
```

---

## 3. 单次 Turn：`run_single_turn()`

文件：`src/agents/run_internal/run_loop.py`（实际实现在 `turn_resolution.py`）

### 3.1 Turn 的完整流程

```
run_single_turn()
  ├─ turn_span 开始（tracing）
  ├─ turn_preparation: get_all_tools / get_handoffs / get_output_schema / get_model
  ├─ maybe_filter_model_input()  （CallModelInputFilter，可选）
  ├─ get_new_response()          ← 调用 LLM，返回 ModelResponse
  ├─ process_model_response()    ← 解析模型输出 → ProcessedResponse
  ├─ execute_tools_and_side_effects()
  │    ├─ execute_function_tool_calls()   （FunctionTool）
  │    ├─ execute_computer_actions()      （ComputerTool）
  │    ├─ execute_handoffs()              （Handoff）
  │    ├─ execute_shell_calls()           （ShellTool）
  │    ├─ execute_apply_patch_calls()     （ApplyPatchTool）
  │    └─ execute_mcp_approval_requests() （HostedMCPTool）
  └─ get_single_step_result_from_response() → SingleStepResult (next_step)
```

### 3.2 `ProcessedResponse`（`run_steps.py`）

模型返回的原始 Response 经 `process_model_response()` 解析后变成结构化的 `ProcessedResponse`：

```python
@dataclass
class ProcessedResponse:
    new_items: list[RunItem]             # 新产生的 RunItem
    handoffs: list[ToolRunHandoff]       # 准备执行的 handoff
    functions: list[ToolRunFunction]     # 准备执行的函数工具
    computer_actions: list[ToolRunComputerAction]
    local_shell_calls: list[ToolRunLocalShellCall]
    shell_calls: list[ToolRunShellCall]
    apply_patch_calls: list[ToolRunApplyPatchCall]
    mcp_approval_requests: list[ToolRunMCPApprovalRequest]
    interruptions: list[ToolApprovalItem]  # 需要人工审批的工具调用
    tools_used: list[str]
    custom_tool_calls: list[ToolRunCustom]
```

### 3.3 `NextStep` 类型

执行工具后，由 `get_single_step_result_from_response()` 决定下一步：

| 类型 | 含义 | 外层循环行为 |
|------|------|------------|
| `NextStepRunAgain` | 有工具调用结果，需要再次调用模型 | 继续循环，`current_turn += 1` |
| `NextStepHandoff` | 模型请求切换 agent | 更新 `current_agent`，继续循环 |
| `NextStepFinalOutput` | 产生了最终输出 | 运行输出 Guardrails，退出循环 |
| `NextStepInterruption` | 工具等待人工审批 (HITL) | 保存 RunState，返回中断结果 |

---

## 4. 流式模式：`run_single_turn_streamed()` & `start_streaming()`

流式模式通过 `asyncio.Queue` 在协程间传递事件：

```
start_streaming()
  ├─ 创建 RunResultStreaming（含内部 _event_queue）
  ├─ asyncio.create_task(run_single_turn_streamed(...))  ← 后台任务
  └─ 返回 RunResultStreaming 给调用方

run_single_turn_streamed()
  ├─ stream_response_with_retry()     ← 获取流式 Response
  ├─ 逐 chunk 处理: stream_step_items_to_queue()
  │    └─ put_nowait(RunItemStreamEvent / RawResponsesStreamEvent)
  ├─ 处理完整 turn 结果: stream_step_result_to_queue()
  └─ put_nowait(QueueCompleteSentinel)  ← 结束信号
```

调用方通过 `async for event in result.stream_events()` 消费事件。

---

## 5. RunState 与中断恢复（HITL）

当某个 Turn 需要人工审批工具时，流程会产生 `NextStepInterruption`：

1. 当前 Turn 的所有中间状态序列化进 `RunState`。
2. `Runner.run()` 返回包含 `RunState` 的中断结果，不是最终 `RunResult`。
3. 用户在外部处理审批后，以 `RunState` 作为 `input` 重新调用 `Runner.run()`。
4. SDK 检测到 `isinstance(input, RunState)` → 走恢复路径 `resolve_interrupted_turn()`。

关键字段：
- `RunState._original_input` — 用户最初的输入
- `RunState._max_turns` — 保持原始 max_turns
- `RunState._current_turn` — 恢复后继续计数

---

## 6. Turn 计数规则

- **只有真正调用模型**才会使 `current_turn += 1`。
- 从 `RunState` 恢复不增加 turn 计数。
- Handoff 切换 agent 后，继续使用**同一个 turn 计数器**。
- 输入 Guardrail 只在**第一个 Turn**、**起始 agent** 上运行。

---

## 7. Session 与 Conversation 跟踪

两种服务端会话管理（互斥）：

| 机制 | 参数 | 说明 |
|------|------|------|
| **Session** | `session=Session(...)` | SDK 自动管理历史，写入 OpenAI Conversation 对象 |
| **Conversation ID** | `conversation_id="..."` | 服务端管理，仅发 delta；推荐只用于 OpenAI 模型 |
| **Previous Response ID** | `previous_response_id="..."` | 手动指定上一次 Response，跳过传入历史 |

---

## 8. 关键文件速查

| 文件 | 职责 |
|------|------|
| `src/agents/run.py` | 公开入口 `Runner`、实验性 `AgentRunner`、主循环逻辑 |
| `src/agents/run_internal/run_loop.py` | re-export 聚合模块，`run_single_turn`、`start_streaming` 在此定义 |
| `src/agents/run_internal/turn_resolution.py` | 模型响应解析、工具/handoff 执行、final output 确认 |
| `src/agents/run_internal/run_steps.py` | 内部数据结构：`ProcessedResponse`、`NextStep*`、`ToolRun*` |
| `src/agents/run_internal/turn_preparation.py` | `get_all_tools`、`get_handoffs`、`get_model`、输入过滤 |
| `src/agents/run_internal/tool_planning.py` | 工具执行计划、并行/串行调度 |
| `src/agents/run_state.py` | `RunState` 序列化/反序列化，Schema 版本管理 |
| `src/agents/result.py` | `RunResult`、`RunResultStreaming` |
| `src/agents/run_config.py` | `RunConfig`、`DEFAULT_MAX_TURNS`（默认 10）|

---

## 9. 典型代码流程追踪

```
Runner.run(agent, "hello")
  → AgentRunner.run()
      → ensure_context_wrapper()
      → create_trace_for_run()
      → run_input_guardrails()         # 仅首 Turn
      → while turn < max_turns:
          → run_single_turn(agent, prepared_input)
              → get_new_response()     # 调用 OpenAI API
              → process_model_response()
              → execute_tools_and_side_effects()
              → SingleStepResult(next_step=NextStepFinalOutput(...))
          → run_output_guardrails()
          → save_result_to_session()
          → return RunResult(output=...)
```
