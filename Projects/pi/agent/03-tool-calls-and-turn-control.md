# 教程 03：工具调用与 turn 控制

`packages/agent` 里最容易低估的一点是：tool call 不是附属功能，而是 turn 控制的一部分。

如果你把工具理解成“assistant 顺手调一下函数”，你会误解很多代码。更准确的理解是：assistant 的一条消息可能会把控制权暂时转交给工具执行阶段，而 turn 是否结束、是否继续下一轮，都受它影响。

## 先抓一条完整路径

在 `src/agent-loop.ts` 中，一轮 assistant 响应大致会经过：

1. 流式生成 assistant message。
2. 检查 `message.content` 中是否含有 `toolCall`。
3. 执行 `executeToolCalls()`。
4. 生成 `toolResult` message 并追加回上下文。
5. 发出 `turn_end`。
6. 决定是否继续下一轮。

所以工具执行不是“turn 结束之后再做点别的”，而是 turn 定义的一部分。

## 源码定位

建议直接看这些位置：

1. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	先看 [runLoop()](../../packages/agent/src/agent-loop.ts#L155) 里 assistant message 完成后筛 `toolCall` block 的逻辑。
2. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	再按 [executeToolCalls()](../../packages/agent/src/agent-loop.ts#L373)、[executeToolCallsSequential()](../../packages/agent/src/agent-loop.ts#L395)、[executeToolCallsParallel()](../../packages/agent/src/agent-loop.ts#L451) 的顺序看。
3. [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts)
	最后看 tool call 会生成 result 并回灌上下文的测试。

## `beforeToolCall` 在拦什么

类型定义里已经写得很清楚：`beforeToolCall` 发生在参数校验之后、真正执行工具之前。

这意味着它适合做：

1. 权限与策略判断。
2. 安全拦截。
3. 按上下文条件禁止某次调用。

它不适合做的事是“自己重新解析一遍参数”，因为这一步之前框架已经做过 schema 校验了。

## 源码定位

可以对照看：

1. [packages/agent/src/types.ts](../../packages/agent/src/types.ts)
	先看 [BeforeToolCallContext](../../packages/agent/src/types.ts#L84) 和 `BeforeToolCallResult`。
2. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	再看 [agent-loop.ts](../../packages/agent/src/agent-loop.ts#L581) 里参数校验和 `beforeToolCall` 的调用顺序。
3. [packages/agent/README.md](../../packages/agent/README.md)
	最后看 `beforeToolCall` 的说明示例。

## `afterToolCall` 在改什么

`afterToolCall` 发生在工具执行之后、最终 tool result 事件和 message 发出之前。

它能改的是：

1. `content`
2. `details`
3. `isError`
4. `terminate`

也就是说，它不是重新执行工具，而是在最终落盘和对外发布之前，对结果做最后一次补丁。

这让它非常适合做：

1. 审计信息注入。
2. 统一错误包装。
3. 给某些工具结果附加“这一批执行完就可以停”的 hint。

## 源码定位

这里建议重点看：

1. [packages/agent/src/types.ts](../../packages/agent/src/types.ts)
	先看 [AfterToolCallContext](../../packages/agent/src/types.ts#L96) 和 `AfterToolCallResult`。
2. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	再看 [agent-loop.ts](../../packages/agent/src/agent-loop.ts#L676) 里工具执行结束后、事件发出前的 finalize 逻辑。
3. [packages/agent/README.md](../../packages/agent/README.md)
	最后看 `terminate: true` 的说明。

## `parallel` 和 `sequential` 真正差在哪里

这里最容易误解的是：并行和串行模式差异不只是性能。

根据 README 和类型说明，两者差异至少有两层：

1. 工具执行顺序不同。
2. 工具相关事件发出的顺序不同。

特别是并行模式下：

1. `tool_execution_end` 可能按完成顺序发出。
2. 但最终持久化的 `toolResult` message 仍然按 assistant 源顺序组织。

这说明作者在同时满足两个目标：

1. 保留并行执行带来的吞吐优势。
2. 保持消息 transcript 的稳定顺序，避免上下文语义漂移。

## 源码定位

这部分推荐结合文档和实现一起读：

1. [packages/agent/src/types.ts](../../packages/agent/src/types.ts)
	先看 [ToolExecutionMode](../../packages/agent/src/types.ts#L36) 注释。
2. [packages/agent/README.md](../../packages/agent/README.md)
	再看 parallel / sequential 的行为说明。
3. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	最后看 [executeToolCallsSequential()](../../packages/agent/src/agent-loop.ts#L395) 与 [executeToolCallsParallel()](../../packages/agent/src/agent-loop.ts#L451) 对 `toolResult` message 顺序的控制。

## 为什么 `terminate` 只是 hint

`afterToolCall` 可以返回 `terminate: true`，但 loop 并不是看到一个 true 就立刻停。

设计上只有“这一批里所有 finalized tool results 都设置了 terminate: true”时，才会真正提前结束自动 follow-up LLM 调用。

这背后的考虑很务实：

1. 单个工具可能希望结束。
2. 但同一批里其他工具结果可能仍需要 assistant 消化。

所以这里是批级决策，不是单调用决策。

## `prepareNextTurn()` 和 `shouldStopAfterTurn()` 怎么看

这两个 callback 都在 `turn_end` 之后参与决策，但职责不同：

1. `prepareNextTurn()` 用于修改下一轮使用的上下文或运行配置。
2. `shouldStopAfterTurn()` 用于在当前 turn 完整结束后，优雅地阻止下一轮继续发生。

顺着这个顺序读代码，你会发现 loop 在乎的不是“一个函数 call 完了没”，而是“下一轮开始前系统应该进入什么状态”。

## 用测试理解工具与队列

`packages/agent/test/agent-loop.test.ts` 里适合重点看的就是：

1. `transformContext()` 和 `convertToLlm()` 的先后。
2. tool call 结果如何回灌到上下文。
3. event sequence 如何体现 tool execution。

而 `packages/agent/test/harness/agent-harness.test.ts` 则进一步告诉你，在完整 harness 里：

1. steering queue 怎么在安全点 drain。
2. abort 清理哪些队列，不清理哪些队列。
3. `before_agent_start` 注入的消息如何落到真正请求和持久化里。

## 这一章读完后你应该回答的问题

1. 为什么 tool call 会直接影响 turn 是否继续。
2. `beforeToolCall` 和 `afterToolCall` 的截点有什么本质差别。
3. 为什么并行工具执行仍然要保持 transcript 的源顺序。
4. 为什么 `terminate` 被设计成批级 hint，而不是硬中断命令。

## 阅读题

1. 为什么 tool call 必须附着在 assistant message 上，而不是设计成另一条独立消息流。
2. `beforeToolCall` 如果放到参数校验之前，会破坏哪些保证。
3. 并行执行下，为什么事件顺序允许按完成时间发出，但 transcript 仍坚持源顺序。
4. `prepareNextTurn()` 和 `shouldStopAfterTurn()` 如果顺序对调，会导致哪些语义变化。