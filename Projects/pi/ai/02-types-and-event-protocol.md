# 教程 02：类型系统与统一事件协议

这一章的目标是看清楚：这个包到底统一了什么。答案不是“统一文本返回”，而是“统一消息结构和事件流结构”。

## 推荐阅读文件

1. `packages/ai/src/types.ts`
2. `packages/ai/src/utils/event-stream.ts`
3. `packages/ai/src/utils/diagnostics.ts`
4. `packages/ai/src/utils/validation.ts`
5. `packages/ai/src/utils/overflow.ts`

## 为什么 `types.ts` 是核心文件

如果只能选一个文件作为 `packages/ai` 的核心，我会优先选 `types.ts`。因为这个库真正稳定的契约就在这里。

provider 实现可以替换，模型清单可以生成，OAuth 和图片能力可以扩展，但整个系统想要保持一致，靠的是这里定义的公共协议。

## 先抓 3 类最关键类型

### 1. 标识体系

重点看：

- `Api`
- `Provider`
- `KnownApi`
- `KnownProvider`

这一组类型描述“模型来自谁”和“要走哪套实现协议”。第一次读要牢牢记住：`provider` 是业务标签，`api` 是实现分发键。

### 2. 会话与消息体系

重点看：

- `Context`
- `Message`
- `UserMessage`
- `AssistantMessage`
- `ToolResultMessage`

这里最关键的设计点是：assistant 不是简单返回一个字符串，而是返回一组 content block。这样才能同时承载：

1. 文本
2. thinking
3. tool call
4. 图片输入输出

这也是为什么不同厂商的复杂返回格式可以被统一掉。

### 3. 流式协议体系

重点看：

- `AssistantMessageEvent`
- `AssistantMessageEventStream`
- `StreamOptions`
- `SimpleStreamOptions`

这一层定义的是“事件过程”，不是“最终结果”。provider 先产出事件，再由事件流收敛成最终 `AssistantMessage`。

## 为什么 assistant 内容是 block 列表

如果设计成单字符串，很多能力都会变成补丁式设计。

block 列表的好处是：

1. 工具调用可以成为一等公民，而不是塞进文本里解析。
2. thinking 内容可以独立承载与屏蔽。
3. 图片和文本可以走同一套内容容器。
4. 不同 provider 的中间状态更容易映射到统一结构。

这说明这个包从一开始就不是“聊天文本库”，而是“多种内容块的统一协议层”。

## `StreamOptions` 和 `SimpleStreamOptions` 的边界

这两个类型看起来很像，但层次不一样：

1. `StreamOptions` 更接近底层 provider 请求能力，比如超时、headers、metadata、transport。
2. `SimpleStreamOptions` 在此之上增加了统一化推理入口，比如 `reasoning` 和 thinking budget。

你可以把它理解为：

- `stream` 面向 provider 能力。
- `streamSimple` 面向跨 provider 的一致体验。

## `event-stream.ts` 要看什么

读这个文件时不要纠结实现细节，重点看它扮演的角色：

1. 它是 provider 输出事件的统一容器。
2. 它负责在异步事件过程中累积最终结果。
3. 它把“边流边读”和“等待最终结果”统一成同一条协议线。

这就是为什么 `complete()` 能直接依赖 `stream()`，而不是再写一套并行逻辑。

## `diagnostics.ts` 和 `validation.ts` 的意义

这两个文件看起来像辅助模块，但其实非常重要，因为它们体现了这个库对“错误”和“工具协议”的处理方式。

### `diagnostics.ts`

你要关注的是：错误不只是抛异常，还可能被编码进 `AssistantMessage` 的诊断字段里。这说明库在设计上允许“失败也作为结果的一部分返回”。

### `validation.ts`

你要关注的是：工具参数不是 provider 自己随便解析，而是被纳入统一校验模型。这样才能做到跨 provider 的稳定工具调用语义。

## `overflow.ts` 要从什么角度看

这里不要只把它当成“超长输入处理”。它体现的是统一上下文管理的边界：

1. 什么情况被认为是上下文溢出。
2. 这种异常如何被上层感知。
3. 它应该在 provider 层处理，还是在统一层处理。

这个问题会直接影响你以后理解不同 provider 的错误映射方式。

## 这一章读完后你应该回答的问题

1. 为什么 assistant 内容是 block 列表，而不是字符串。
2. 事件流和最终 `AssistantMessage` 是什么关系。
3. `stream` 与 `streamSimple` 各自服务哪一层抽象。
4. 工具校验和错误诊断为什么要进入统一协议，而不是散落在各 provider 里。

## 建议的动手练习

自己画一张最小消息模型图，至少包含：

1. `Context`
2. `UserMessage`
3. `AssistantMessage`
4. `ToolResultMessage`
5. `TextContent`
6. `ThinkingContent`
7. `ToolCall`

当你能把这些关系讲顺时，再去看 provider 文件，你会更容易识别哪些逻辑是“协议映射”，哪些是“厂商特性”。