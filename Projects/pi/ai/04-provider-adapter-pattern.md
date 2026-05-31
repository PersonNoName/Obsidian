# 教程 04：provider 适配模式

这一章正式进入实现层。重点不是逐个厂商背接口，而是找出所有 provider 共享的适配套路。

## 推荐阅读顺序

建议先精读一个最熟悉的 provider，再横向比较其他实现。

推荐顺序：

1. `packages/ai/src/providers/openai-responses.ts`
2. `packages/ai/src/providers/anthropic.ts`
3. `packages/ai/src/providers/google.ts`
4. `packages/ai/src/providers/google-vertex.ts`
5. `packages/ai/src/providers/openai-completions.ts`
6. `packages/ai/src/providers/mistral.ts`
7. `packages/ai/src/providers/amazon-bedrock.ts`
8. `packages/ai/src/providers/openai-codex-responses.ts`

补充读这些共享模块：

1. `packages/ai/src/providers/simple-options.ts`
2. `packages/ai/src/providers/transform-messages.ts`
3. `packages/ai/src/providers/google-shared.ts`
4. `packages/ai/src/providers/openai-responses-shared.ts`
5. `packages/ai/src/providers/openai-prompt-cache.ts`
6. `packages/ai/src/providers/github-copilot-headers.ts`

## 读 provider 时要统一带着的问题

每读一个 provider，都回答同样 6 个问题：

1. 上游 SDK 或 HTTP 协议是什么。
2. 输入 `Context` 是如何被转换为该 provider 需要的消息结构的。
3. 上游流式输出是如何被转换成统一事件流的。
4. 工具调用、thinking、usage、stop reason 是如何映射的。
5. 错误、超时、重试、hook 在哪里处理。
6. 哪些逻辑属于这个 provider 专有，哪些已经被共享模块抽掉了。

如果你每次都带着这 6 个问题读，横向比较就会很快。

## provider 实现的共性骨架

大多数 provider 文件都可以概括成下面这套骨架：

1. 接收统一 `model`、`context`、`options`。
2. 把统一消息协议转换成上游 API 需要的 payload。
3. 发起 SDK 调用或 HTTP 请求。
4. 读取上游流事件或结果。
5. 产出统一 `AssistantMessageEvent` 序列。
6. 在结束时形成统一的最终 assistant message。

这说明 provider 层真正的工作不是“直接调接口”，而是“做协议翻译”。

## 为什么共享辅助模块很重要

第一次读 provider 时很容易只盯着主文件，但复杂性其实大量藏在共享辅助模块里。

### `transform-messages.ts`

这是最值得重点看的辅助文件之一。它通常承载跨 provider 的消息转换规则，比如：

1. 如何把统一 `Context` 转成厂商消息格式。
2. 工具结果、图片结果、system prompt 如何落位。
3. 特定 provider 的边界条件如何在转换层被消化。

### `simple-options.ts`

它体现的是统一化选项如何被压到不同 provider 上。重点不是函数写法，而是“统一选项到 provider 选项”的映射策略。

### `google-shared.ts` 与 `openai-responses-shared.ts`

这类文件通常说明：某些 provider 内部也存在可复用的子协议，或者某种厂商协议族本身就有多处复用。

## 读 provider 时最容易踩的误区

### 误区 1：把重点放在 SDK 调用参数细节

这些细节当然重要，但学习顺序不该先从那里开始。更关键的是先看：

1. 输入协议如何转换。
2. 输出事件如何统一。
3. 差异点如何被吸收。

### 误区 2：把每个 provider 当独立系统来读

这样很难建立横向理解。正确做法是带着统一问题清单去读，把每个 provider 当成“同一个接口下的不同翻译器”。

### 误区 3：忽略测试中的行为定义

有些 provider 细节单看实现不容易理解，这时应该回到测试，看库作者真正想保证的行为是什么。

## 推荐的精读顺序

### 第一轮

只挑一个 provider，建议 OpenAI 或 Anthropic。目标不是看完所有分支，而是识别：

1. payload 组装入口
2. 流事件解析入口
3. 统一事件输出入口

### 第二轮

再读一个差异更大的 provider，比如 Google 或 Bedrock。目标是看统一抽象在异构协议上如何成立。

### 第三轮

回头读共享模块，看看哪些复杂性本来以为是 provider 专有，实际上已经被抽成共性逻辑。

## 这一章读完后你应该回答的问题

1. 一个 provider 最少要实现哪些能力才能接入这套框架。
2. 为什么 provider 层的本质工作是协议转换，而不是单纯发请求。
3. 工具调用、thinking、usage 和错误处理在 provider 中通常落在哪一段代码。
4. 为什么共享模块能显著降低 provider 文件本体的复杂度。

## 建议的动手练习

选两个 provider，自己做一张对照表，列出：

1. provider 文件名
2. `api`
3. 上游协议类型
4. 工具调用支持情况
5. thinking 支持情况
6. 是否依赖共享 helper
7. 最明显的专有逻辑是什么

这会让你很快从“会读一个实现”升级到“会比较多个实现”。