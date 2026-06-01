# 教程 06：测试阅读、源码笔记与学习计划

前 5 章解决的是“代码怎么组织”。这一章解决的是“怎么把这部分代码读成体系”，也就是如何用测试确认行为、如何做阅读笔记，以及应该按什么节奏推进。

## 为什么先看测试很划算

实现文件告诉你代码怎么写，测试文件告诉你作者认为什么行为不能退化。

对 `packages/agent` 来说，这一点尤其重要，因为这里既有核心 loop，也有 harness 生命周期。测试能帮你快速分清：

1. 哪些是公共契约。
2. 哪些只是内部实现方式。
3. 哪些边界条件是作者明确在保护的。

## 建议优先看的测试

先读这些：

1. `packages/agent/test/agent.test.ts`
2. `packages/agent/test/agent-loop.test.ts`
3. `packages/agent/test/harness/agent-harness.test.ts`

这三个文件基本对应三层：

1. `Agent` 高层状态与生命周期保证。
2. 低层 loop 的行为定义。
3. 完整 harness 的运行环境语义。

## 源码定位

建议按这个顺序点开测试：

1. [packages/agent/test/agent.test.ts](../../packages/agent/test/agent.test.ts)
2. [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts)
3. [packages/agent/test/harness/agent-harness.test.ts](../../packages/agent/test/harness/agent-harness.test.ts)
4. [packages/agent/test/harness/agent-harness-stream.test.ts](../../packages/agent/test/harness/agent-harness-stream.test.ts)

每点开一个测试文件，都同时开着对应实现文件：

1. [packages/agent/src/agent.ts](../../packages/agent/src/agent.ts)
	先开 [prompt()](../../packages/agent/src/agent.ts#L327)、[continue()](../../packages/agent/src/agent.ts#L338)、[createLoopConfig()](../../packages/agent/src/agent.ts#L422)、[processEvents()](../../packages/agent/src/agent.ts#L509)。
2. [packages/agent/src/agent-loop.ts](../../packages/agent/src/agent-loop.ts)
	再开 [runAgentLoop()](../../packages/agent/src/agent-loop.ts#L95)、[runLoop()](../../packages/agent/src/agent-loop.ts#L155)、[executeToolCalls()](../../packages/agent/src/agent-loop.ts#L373)。
3. [packages/agent/src/harness/agent-harness.ts](../../packages/agent/src/harness/agent-harness.ts)
	最后开 [emitHook()](../../packages/agent/src/harness/agent-harness.ts#L231)、[createTurnState()](../../packages/agent/src/harness/agent-harness.ts#L313)、[abort()](../../packages/agent/src/harness/agent-harness.ts#L936)。

## 从 `agent.test.ts` 里重点看什么

这个文件适合验证：

1. 默认 state 长什么样。
2. listener 是如何被 await 的。
3. run failure 时事件序列是否完整。
4. `waitForIdle()` 和 settle 语义如何配合。

这里最重要的收获通常不是某个小函数，而是你会发现：这个包很在意“事件何时算真正结束”。

## 源码定位

这里建议重点看：

1. [packages/agent/test/agent.test.ts](../../packages/agent/test/agent.test.ts)
	先看默认 state 与自定义初始 state 的测试。
2. [packages/agent/test/agent.test.ts](../../packages/agent/test/agent.test.ts)
	再看 async subscriber 被 await 的测试。
3. [packages/agent/test/agent.test.ts](../../packages/agent/test/agent.test.ts)
	再看 thrown run failure 仍产出完整 lifecycle event 的测试。
4. [packages/agent/test/agent.test.ts](../../packages/agent/test/agent.test.ts)
	最后看 `waitForIdle()` 与 listener settlement 的测试。

## 从 `agent-loop.test.ts` 里重点看什么

这个文件适合验证：

1. 自定义消息怎样通过 `convertToLlm()` 进入或被过滤出 provider 边界。
2. `transformContext()` 是否先于 `convertToLlm()`。
3. tool call 如何生成 result 并回灌上下文。
4. event sequence 最少保证了什么。

换句话说，这个文件帮你确认“低层 loop 的行为契约”。

## 源码定位

建议重点看：

1. [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts)
	先看 custom message 通过 `convertToLlm()` 被过滤的测试。
2. [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts)
	再看 `transformContext()` 先于 `convertToLlm()` 的测试。
3. [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts)
	再看 tool call 执行与 tool result 生成的测试。
4. [packages/agent/test/agent-loop.test.ts](../../packages/agent/test/agent-loop.test.ts)
	最后看 continue 相关边界测试。

## 从 harness 测试里重点看什么

`agent-harness.test.ts` 最值得看的是运行语义，而不是某个 API 名字。

例如：

1. steering queue 如何按 mode drain。
2. `before_agent_start` 注入消息如何同时影响真正请求和持久化。
3. `abort()` 为什么清理 steer / follow-up，却保留 nextTurn。
4. queue update 事件在什么时候发。

这些测试能帮你快速理解 harness 为什么是“运行容器”，而不是“Agent 的工具箱”。

## 源码定位

推荐直接看：

1. [packages/agent/test/harness/agent-harness.test.ts](../../packages/agent/test/harness/agent-harness.test.ts)
	先看队列 mode drain 的测试。
2. [packages/agent/test/harness/agent-harness.test.ts](../../packages/agent/test/harness/agent-harness.test.ts)
	再看 `before_agent_start` 注入消息并持久化的测试。
3. [packages/agent/test/harness/agent-harness.test.ts](../../packages/agent/test/harness/agent-harness.test.ts)
	再看 `abort` 清队列但保留 nextTurn 的测试。
4. [packages/agent/test/harness/agent-harness-stream.test.ts](../../packages/agent/test/harness/agent-harness-stream.test.ts)
	最后看 before-provider-request 与 stream option 相关测试。

## 推荐的源码笔记方式

建议自己维护 3 张表。

### 表 1：分层表

每读一个文件，先写它更接近哪一层：

1. 核心状态层
2. 核心循环层
3. 协议层
4. harness 运行环境层

### 表 2：生命周期表

记录你确认过的关键时序：

1. `agent_start`
2. `turn_start`
3. `message_start/update/end`
4. `tool_execution_start/update/end`
5. `turn_end`
6. `agent_end`
7. harness save point

### 表 3：队列与状态变更表

记录这些行为差异：

1. steer
2. followUp
3. nextTurn
4. abort
5. prepareNextTurn
6. shouldStopAfterTurn

只要你把这三张表写出来，这部分代码就不会再只是“看过”。

## 一个 4 天学习计划

### 第 1 天：主链路

读：

1. `src/index.ts`
2. `src/agent.ts`
3. `src/agent-loop.ts`

输出：

1. 一张 `prompt()` 到 `agent_end` 的调用链图。
2. 一份 `Agent` 和 `agent-loop` 的分层说明。

### 第 2 天：协议层

读：

1. `src/types.ts`
2. `test/agent.test.ts`
3. `test/agent-loop.test.ts`

输出：

1. 一份 `AgentMessage` / `AgentState` / `AgentEvent` 关系说明。
2. 一份事件顺序和工具 hook 的行为笔记。

### 第 3 天：Harness

读：

1. `src/harness/agent-harness.ts`
2. `docs/agent-harness.md`
3. `test/harness/agent-harness.test.ts`

输出：

1. 一份 phase、save point、queue mode 说明。
2. 一份裸 `Agent` vs `AgentHarness` 的职责对照表。

### 第 4 天：Session 与扩展能力

读：

1. `src/harness/session/session.ts`
2. `src/harness/compaction/compaction.ts`
3. `docs/hooks.md`

输出：

1. 一张 session tree 与 context rebuild 图。
2. 一份 compaction 与 hooks 为什么放在 harness 层的说明。

## 最后的阅读建议

第一次读不要一开始就铺开整个 `harness/` 目录。最稳的顺序仍然是：

1. 先抓 `Agent` 和 `agent-loop`。
2. 再用 `types.ts` 固化协议边界。
3. 再进入 harness 生命周期。
4. 最后回头用测试确认你的理解。

如果你跳过前两步直接读 harness，会看到很多正确细节，但很难判断哪些是核心机制，哪些是运行环境策略。

## 阅读题

1. 哪三个测试最能帮助你区分 `Agent`、`agent-loop`、`AgentHarness` 的职责边界。
2. 哪些测试保护的是事件顺序，哪些测试保护的是上下文或队列语义。
3. 如果某个重构让 `transformContext()` 晚于 `convertToLlm()`，你预期会先坏掉哪类测试。
4. 如果你要新增一条教程章节，最值得先补哪个测试文件的阅读说明，为什么。