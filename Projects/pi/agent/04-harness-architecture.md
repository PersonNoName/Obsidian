# 教程 04：AgentHarness 架构

读到这里，你已经能理解“裸 Agent 怎么跑”。接下来要回答的问题是：为什么 `packages/agent` 还要再做一层 `AgentHarness`。

最短答案是：裸 `Agent` 解决核心循环，`AgentHarness` 解决产品环境里的生命周期、持久化、资源与运行时更新语义。

## 先看 `AgentHarness` 管什么

从 `src/harness/agent-harness.ts` 和 `docs/agent-harness.md` 看，`AgentHarness` 至少多管理了这些维度：

1. session 持久化
2. runtime config
3. resources
4. tools 与 active tools
5. steering / follow-up / nextTurn 队列
6. hooks 与事件
7. operation phase
8. save point

这说明 harness 不是“给 Agent 加几个 helper”，而是在 Agent 之上定义了一套更强的运行语义。

## 源码定位

建议先用字段把它扫一遍：

1. [packages/agent/src/harness/agent-harness.ts](../../packages/agent/src/harness/agent-harness.ts)
	先看 `AgentHarness` 的私有字段和 [emitHook()](../../packages/agent/src/harness/agent-harness.ts#L231)、[emitBeforeProviderRequest()](../../packages/agent/src/harness/agent-harness.ts#L250)、[createTurnState()](../../packages/agent/src/harness/agent-harness.ts#L313)。
2. [packages/agent/docs/agent-harness.md](../../packages/agent/docs/agent-harness.md)
	再看 state model 和 operation phases 两节。
3. [packages/agent/test/harness/agent-harness.test.ts](../../packages/agent/test/harness/agent-harness.test.ts)
	最后看直接构造 harness 和队列模式相关测试。

## 核心理解：config 和 turn snapshot 分离

`docs/agent-harness.md` 里一个非常关键的设计是：

1. harness config 是“最新配置”。
2. turn snapshot 是“当前这轮真正使用的冻结视图”。

这层分离解决的是一个很现实的问题：

1. 外部代码可能在 turn 执行中修改 model、thinking level、resources、stream options。
2. 但当前 provider request 不应该被中途篡改。

所以正确语义不是“所有改动立刻影响当前请求”，而是“改动先更新 future config，等到下一轮 snapshot 再生效”。

## 源码定位

这里可以对照：

1. [packages/agent/docs/agent-harness.md](../../packages/agent/docs/agent-harness.md)
	先看 Harness config 与 Turn snapshot 两节。
2. [packages/agent/src/harness/agent-harness.ts](../../packages/agent/src/harness/agent-harness.ts)
	再看 [createTurnState()](../../packages/agent/src/harness/agent-harness.ts#L313) 和 [createTurnState() 的后续消费点](../../packages/agent/src/harness/agent-harness.ts#L441)。
3. [packages/agent/test/harness/agent-harness-stream.test.ts](../../packages/agent/test/harness/agent-harness-stream.test.ts)
	最后看运行中配置如何影响后续 request 的相关测试。

## 为什么 phase 很重要

文档里把 harness phase 明确成：

1. `idle`
2. `turn`
3. `compaction`
4. `branch_summary`
5. `retry`

这不是为了多一层状态机，而是为了精确回答：

1. 哪些操作在 busy 期间必须拒绝。
2. 哪些操作可以在 turn 中途安全接受。
3. 哪些写入需要延后到 save point 或 settle 再刷盘。

例如结构性操作只允许在 `idle` 执行，而 `steer()`、`followUp()`、`nextTurn()`、`abort()`、config setter 则可以在 turn 中作为安全操作存在。

## 源码定位

建议重点看：

1. [packages/agent/docs/agent-harness.md](../../packages/agent/docs/agent-harness.md)
	先看 Operation phases。
2. [packages/agent/src/harness/types.ts](../../packages/agent/src/harness/types.ts)
	再看 [AgentHarnessPhase](../../packages/agent/src/harness/types.ts#L485)。
3. [packages/agent/src/harness/agent-harness.ts](../../packages/agent/src/harness/agent-harness.ts)
	最后看 [steer()](../../packages/agent/src/harness/agent-harness.ts#L652)、[followUp()](../../packages/agent/src/harness/agent-harness.ts#L658)、[nextTurn()](../../packages/agent/src/harness/agent-harness.ts#L664)、[abort()](../../packages/agent/src/harness/agent-harness.ts#L936) 以及 turn 启动路径附近的 phase 控制。

## save point 才是 harness 的关键发明

如果你只记一件事，就记 save point。

文档定义 save point 出现在：一个 assistant turn 以及它的 tool result message 全部完成之后。

在这个点上，harness 会做 3 件事：

1. flush pending session writes
2. 创建新的 turn snapshot
3. 让 turn 期间发生的配置变化影响下一轮请求

这意味着 harness 的目标不是“每一步都实时同步”，而是“在安全点上有序结算”。

## 源码定位

这部分优先看文档，再回到实现：

1. [packages/agent/docs/agent-harness.md](../../packages/agent/docs/agent-harness.md)
	先看 Save points。
2. [packages/agent/src/harness/agent-harness.ts](../../packages/agent/src/harness/agent-harness.ts)
	再看 [emitBeforeProviderRequest()](../../packages/agent/src/harness/agent-harness.ts#L250)、[createTurnState()](../../packages/agent/src/harness/agent-harness.ts#L313) 和后续 save-point 刷新逻辑。
3. [packages/agent/test/harness/agent-harness.test.ts](../../packages/agent/test/harness/agent-harness.test.ts)
	最后看 `before_agent_start` 持久化和 abort queue 语义的测试。

## 队列语义为什么比裸 Agent 更复杂

`Agent` 已经有 steering 和 follow-up 队列，但 harness 又加了 `nextTurn`，并把 queue mode 作为 live config。

从测试和文档能看出这背后的意图：

1. steering 是给当前 run 插队。
2. follow-up 是在 agent 本来要结束时继续推进。
3. nextTurn 是留给下一个用户触发 turn 的延迟插入消息。

`abort()` 的测试也明确了：

1. steer / follow-up 会被清理。
2. nextTurn 不会被清理。

这不是偶然细节，而是在定义不同队列的产品语义。

## `AgentHarness` 和 `Agent` 的关系

你可以这样理解：

1. `Agent` 是状态化循环门面。
2. `AgentHarness` 是带 session 与资源语义的运行容器。

这两者不是替代关系，而是不同抽象层级。

如果你的需求只是一个内存态 agent，读 `Agent` 就够。

如果你的需求是一个真实可持续运行的 coding/session agent，那么阅读重点就应该落到 harness。

## 这一章读完后你应该回答的问题

1. 为什么 harness 要把 config 和 turn snapshot 分开。
2. 为什么 save point 是整套运行语义的核心。
3. phase 在保护哪些不变量。
4. steering、follow-up、nextTurn 三种队列分别服务什么场景。

## 阅读题

1. 为什么 `AgentHarness` 不能简单地把 `Agent` 包一层再额外加几个 getter/setter 就算完成。
2. 如果没有 turn snapshot，运行中修改 model 或 resources 会造成什么一致性问题。
3. 为什么 save point 必须落在 assistant turn 和 tool results 都结束之后，而不是更早。
4. `abort()` 为什么清 steer / follow-up，却保留 nextTurn，反映了什么产品语义。