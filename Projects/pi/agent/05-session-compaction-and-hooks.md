# 教程 05：Session、Compaction 与 Hooks

这一章看的是 `AgentHarness` 背后的三套支撑结构：session、compaction 和 hooks。

这三块放在一起读，会更容易理解作者为什么没有把所有能力都塞进 `Agent` 本体。

## 先看 session：它不是简单 transcript

从 `src/harness/session/session.ts` 看，session 并不是“顺序数组存消息”这么简单。

它更接近一棵带 leaf 游标的会话树：

1. 每条 entry 都有 `id`、`parentId`、`timestamp`。
2. 当前上下文来自 active leaf 到 root 的路径。
3. `moveTo()` 可以切换 leaf，并可附带 branch summary。

这意味着 session 设计目标不是只保存历史，而是支持分支、切换、摘要和重建上下文。

## 源码定位

建议先扫 entry 形状，再看重建逻辑：

1. [packages/agent/src/harness/types.ts](../../packages/agent/src/harness/types.ts)
	先看 session entry 相关类型。
2. [packages/agent/src/harness/session/session.ts](../../packages/agent/src/harness/session/session.ts)
	再看 [buildSessionContext()](../../packages/agent/src/harness/session/session.ts#L21) 和 `Session` 方法。
3. [packages/agent/docs/agent-harness.md](../../packages/agent/docs/agent-harness.md)
	最后看 Session 一节。

## `buildSessionContext()` 在做什么

这个函数值得认真读，因为它告诉你“持久化记录”是怎么被还原成“下一轮 agent 上下文”的。

它会沿路径重建：

1. `messages`
2. `thinkingLevel`
3. `model`
4. compaction 后的可见上下文

也就是说，session 不是被动存档，而是下一轮执行输入的一部分。

## 源码定位

这里建议边读边画图：

1. [packages/agent/src/harness/session/session.ts](../../packages/agent/src/harness/session/session.ts)
	先精读 [buildSessionContext()](../../packages/agent/src/harness/session/session.ts#L21)。
2. [packages/agent/src/harness/messages.ts](../../packages/agent/src/harness/messages.ts)
	再看 compaction / branch summary / custom message 的构造函数。
3. [packages/agent/src/harness/session](../../packages/agent/src/harness/session)
	最后看 storage 实现里叶子游标和 entry 持久化。

## compaction 的重点不只是压缩 token

很多人会把 compaction 理解成“上下文太长了，做个 summary”。这不算错，但太浅。

从 `src/harness/compaction/compaction.ts` 看，compaction 至少还承担了：

1. 估算上下文 token 使用。
2. 决定是否需要 compact。
3. 保留最近上下文片段。
4. 生成可持久化的 summary entry。
5. 记录 compact 前历史里涉及过的文件操作信息。

这里已经能看出它是为 coding agent 场景服务的，不只是普通聊天摘要。

## 源码定位

重点看这些函数：

1. [packages/agent/src/harness/compaction/compaction.ts](../../packages/agent/src/harness/compaction/compaction.ts)
	先看 [estimateContextTokens()](../../packages/agent/src/harness/compaction/compaction.ts#L165)、[shouldCompact()](../../packages/agent/src/harness/compaction/compaction.ts#L196)、[estimateTokens()](../../packages/agent/src/harness/compaction/compaction.ts#L202)。
2. [packages/agent/src/harness/compaction/compaction.ts](../../packages/agent/src/harness/compaction/compaction.ts)
	再看 [prepareCompaction()](../../packages/agent/src/harness/compaction/compaction.ts#L541) 和 [compact()](../../packages/agent/src/harness/compaction/compaction.ts#L626)。
3. [packages/agent/src/harness/compaction](../../packages/agent/src/harness/compaction)
	最后看 utils 和 branch summary 相关实现。

## 为什么 compaction 放在 harness 而不放在 `Agent`

因为 compaction 本质上依赖的是：

1. session tree
2. 持久化 entry
3. save point
4. 对运行上下文的再构造

这些都超出了裸 `Agent` 的责任范围。

如果把 compaction 强塞进 `Agent`，就会让核心循环同时承担持久化策略和历史裁剪策略，抽象边界会明显变差。

## hooks 解决的是什么问题

`docs/hooks.md` 展示的不是一个“方便扩展的监听器 API”，而是一套有结果语义的事件系统。

它区分了：

1. `observe()` 这种只观察、不参与结果的处理器。
2. `on(type, handler)` 这种真的参与该事件语义的处理器。

更关键的是，某些事件不是简单广播，而是有 reducer 或顺序变换逻辑，例如：

1. context transform
2. before provider request / payload patch
3. tool call block
4. tool result patch

这说明 hooks 的目标不是“多订阅几个事件”，而是“在严格受控的语义点上允许扩展”。

## 源码定位

建议直接按文档的三个层次看：

1. [packages/agent/docs/hooks.md](../../packages/agent/docs/hooks.md)
	先看 core model。
2. [packages/agent/docs/hooks.md](../../packages/agent/docs/hooks.md)
	再看 mutation semantics。
3. [packages/agent/src/harness/agent-harness.ts](../../packages/agent/src/harness/agent-harness.ts)
	最后看 [emitHook()](../../packages/agent/src/harness/agent-harness.ts#L231)、[emitBeforeProviderRequest()](../../packages/agent/src/harness/agent-harness.ts#L250) 等调用点。

## 为什么 hooks 文档反复强调 context facade

文档里反复强调不要把 raw internals 直接暴露给 hook，而要给 plain object facade。原因很现实：

1. hook 很容易在运行中重入调用 harness API。
2. 如果暴露的是内部对象，就容易破坏 snapshot、pending writes 和 settle 语义。

所以这套设计非常关注“扩展能力”和“生命周期安全”之间的平衡。

## 把这三块合起来看

你可以把它们理解成：

1. session 负责“历史如何被持久化和重建”。
2. compaction 负责“历史太大时如何收缩，同时保留有用工作记忆”。
3. hooks 负责“如何在不破坏生命周期的前提下扩展运行语义”。

这三块都是产品运行语义，不是最小 agent 循环的一部分，所以它们被放进 harness，而不是塞进 `Agent` 本体。

## 这一章读完后你应该回答的问题

1. 为什么 session 被设计成树，而不是简单 transcript list。
2. compaction 为什么依赖 session 与 save point。
3. hooks 为什么不仅是观察事件，还能参与语义变换。
4. 为什么这些能力必须待在 harness 层，而不是核心 loop 层。

## 阅读题

1. 为什么 session 必须记录 model change、thinking level change 这类 entry，而不只是消息。
2. compaction 为什么要把文件读写痕迹带进 details，而不是只保留自然语言摘要。
3. hooks 系统里为什么要区分 `observe()` 和真正参与结果语义的 handler。
4. 如果把 session、compaction、hooks 直接做进 `Agent`，最先变差的抽象边界会是什么。