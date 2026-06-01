# 教程 02：状态、消息与事件协议

上一章解决了“调用链怎么走”。这一章解决“这个包真正稳定的公共边界是什么”。答案主要在 `src/types.ts`。

## 为什么这一章重要

如果你只读 `agent.ts` 和 `agent-loop.ts`，会知道代码怎么运行；但你不一定知道哪些东西是设计意图，哪些只是实现细节。

在 `packages/agent` 里，真正稳定的边界主要由这些类型定义：

1. `AgentMessage`
2. `AgentState`
3. `AgentEvent`
4. `AgentLoopConfig`
5. `AgentTool` 与工具 hook 上下文

## `AgentMessage` 是什么

这里最值得注意的是：`AgentMessage` 不是简单等于 `@earendil-works/pi-ai` 的 `Message`。

它是：

1. 标准 LLM message 的并集。
2. 外部应用可通过 declaration merging 扩展的自定义消息类型。

这说明作者明确希望 agent 内部上下文承载“比 LLM 原生协议更多的信息”。

这也解释了为什么 `convertToLlm()` 是核心边界：它负责从“应用内部消息世界”过渡到“模型可理解消息世界”。

## 源码定位

优先看这几处：

1. [packages/agent/src/types.ts](../../packages/agent/src/types.ts)
	先看 `CustomAgentMessages` 和 `AgentMessage` 附近的类型定义，再跳到 [AgentLoopConfig](../../packages/agent/src/types.ts#L135)。
2. [packages/agent/src/agent.ts](../../packages/agent/src/agent.ts)
	再看 `agent.ts` 顶部的默认 `convertToLlm()`。
3. [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts)
	最后看自定义 notification message 被过滤的测试。

## `AgentState` 要怎么读

读 `AgentState` 时，不要只看字段列表，要先把它分成两类：

1. 长期配置或累积状态。
2. 运行中的瞬时状态。

例如：

- `systemPrompt`、`model`、`thinkingLevel`、`tools`、`messages` 更接近长期状态。
- `isStreaming`、`streamingMessage`、`pendingToolCalls`、`errorMessage` 更接近 run-time 状态。

这个拆分很重要，因为它决定你看代码时应该问的是：

1. 这个字段会不会跨 run 保留。
2. 这个字段是配置，还是某轮运行的中间态。
3. 它是给外部读的，还是给内部控制流程用的。

## 源码定位

建议重点看：

1. [packages/agent/src/types.ts](../../packages/agent/src/types.ts)
	先看 [AgentState](../../packages/agent/src/types.ts#L317)。
2. [packages/agent/src/agent.ts](../../packages/agent/src/agent.ts)
	再看 [createMutableAgentState()](../../packages/agent/src/agent.ts#L66) 和 `MutableAgentState`。
3. [packages/agent/test/agent.test.ts](../../packages/agent/test/agent.test.ts)
	最后看默认 state 与自定义初始 state 的测试。

## 为什么 `tools` 和 `messages` 用 accessor

`agent.ts` 里一个很值得注意但容易忽略的细节是：`tools` 和 `messages` 通过 getter/setter 包装，并在赋值时复制顶层数组。

这不是多余操作，而是在表达一种最小防线：

1. 赋值时复制，避免外部把同一个数组引用直接塞进内部状态。
2. 读取后仍允许你继续改当前状态里的数组内容，因此它不是完全不可变模型。

这说明这里追求的不是全量 immutable store，而是“防明显误用”的实用型状态边界。

## `AgentEvent` 为什么要拆这么细

第一次看会觉得事件很多：

1. `agent_start` / `agent_end`
2. `turn_start` / `turn_end`
3. `message_start` / `message_update` / `message_end`
4. `tool_execution_start` / `tool_execution_update` / `tool_execution_end`

这种细拆不是为了“事件系统完整”，而是为了明确 3 个不同层级：

1. 整次 run 的生命周期。
2. 单个 turn 的生命周期。
3. 消息与工具执行的细粒度流式反馈。

如果不这么拆，你就很难同时支持：

1. UI 层的流式渲染。
2. 持久化层的顺序保证。
3. 外部 hook 对 tool execution 的观察和控制。

## 源码定位

这里可以一边看定义一边看事件发出点：

1. [packages/agent/src/types.ts](../../packages/agent/src/types.ts)
	先看 [AgentEvent](../../packages/agent/src/types.ts#L403) 联合类型。
2. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	再看 [runLoop()](../../packages/agent/src/agent-loop.ts#L155) 中 `emit({ type: ... })` 的各个位置。
3. [packages/agent/test/agent.test.ts](../../packages/agent/test/agent.test.ts)
	最后看 lifecycle event sequence 的测试。

## `AgentLoopConfig` 是真正的扩展面

如果你把 `Agent` 当成主要扩展入口，很容易误判。更真实的扩展面其实是 `AgentLoopConfig`。

因为它决定了：

1. 用哪个 `model`。
2. 如何 `convertToLlm()`。
3. 是否 `transformContext()`。
4. 是否注入 steering / follow-up message。
5. 是否在 turn 结束后 `shouldStopAfterTurn()` 或 `prepareNextTurn()`。
6. 工具怎么执行，以及工具前后 hook 怎样工作。

你可以把它理解成“低层 loop 的策略对象”。

## 源码定位

建议顺着配置消费点去看：

1. [packages/agent/src/types.ts](../../packages/agent/src/types.ts)
	先看 [AgentLoopConfig](../../packages/agent/src/types.ts#L135)。
2. [packages/agent/src/agent.ts](../../packages/agent/src/agent.ts)
	再看 [Agent](../../packages/agent/src/agent.ts#L202) 构造函数如何把 options 映射到实例字段。
3. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	最后看 [runLoop()](../../packages/agent/src/agent-loop.ts#L155) 里 `transformContext`、`getSteeringMessages`、`prepareNextTurn`、`shouldStopAfterTurn` 的调用点。

## `ThinkingLevel` 和 provider `reasoning` 不是同一层

类型里有 `ThinkingLevel`，而在运行时最终可能落成 provider 的 `reasoning` 参数。这里体现的是一层抽象转换：

1. agent 层用统一的 thinking level 表达策略。
2. 到 provider 边界时，再转成具体上游协议需要的字段。

这说明 `packages/agent` 并不想让上层直接依赖 provider 方言。

## 工具协议为什么也放在这里

工具相关类型包括：

1. `ToolExecutionMode`
2. `BeforeToolCallContext`
3. `AfterToolCallContext`
4. `BeforeToolCallResult`
5. `AfterToolCallResult`

把这些放在协议层而不是直接散在 `agent-loop.ts`，是在表达：工具并不是 loop 的内部偶然实现，而是公开设计的一部分。

## 用测试验证这一章

`packages/agent/test/agent.test.ts` 和 `packages/agent/test/agent-loop.test.ts` 已经体现出这些协议的意图，例如：

1. 自定义消息如何通过 `convertToLlm()` 过滤。
2. `transformContext()` 一定先于 `convertToLlm()`。
3. 事件顺序是对外行为保证的一部分。

如果你读完类型后回看这些测试，会明显更容易判断哪些行为是被显式保护的。

## 这一章读完后你应该回答的问题

1. 为什么 `AgentMessage` 要允许扩展。
2. `AgentState` 里哪些字段是长期状态，哪些是 run-time 状态。
3. 为什么事件需要分成 run、turn、message、tool execution 四层。
4. 为什么 `AgentLoopConfig` 才是低层 loop 的核心扩展入口。

## 阅读题

1. `AgentMessage` 和 `Message` 的边界如果消失，会直接破坏本包里的哪些能力。
2. `AgentState` 为什么把 `pendingToolCalls` 和 `streamingMessage` 暴露出来，而不是完全藏在内部。
3. `message_update` 为什么只适用于 assistant 流，而不适合用户消息或 toolResult 消息。
4. `AgentLoopConfig` 里哪些字段更像静态配置，哪些字段更像运行中的策略回调。