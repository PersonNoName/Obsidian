# 教程 05：图片生成、OAuth 与运行时兼容

这一章看的是几条相对独立的支线。它们不是主文本链路的一部分，但又是 `packages/ai` 能在真实环境里工作的重要基础设施。

## 推荐阅读文件

### 图片生成支线

1. `packages/ai/src/images.ts`
2. `packages/ai/src/images-api-registry.ts`
3. `packages/ai/src/image-models.ts`
4. `packages/ai/src/providers/images/register-builtins.ts`
5. `packages/ai/src/providers/images/openrouter.ts`

### 认证与运行时支线

1. `packages/ai/src/env-api-keys.ts`
2. `packages/ai/src/oauth.ts`
3. `packages/ai/src/utils/oauth/`
4. `packages/ai/src/session-resources.ts`
5. `packages/ai/src/cli.ts`

## 为什么图片生成单独一条链路

图片生成没有复用文本生成主链路，这是一个很合理的架构选择。

原因主要有 3 个：

1. 输入输出结构不同，文本生成是消息流，图片生成更接近图像结果集合。
2. stop reason、usage、内容块结构虽然相似，但不是同一层抽象。
3. 图片生成通常不需要复用文本的流式工具调用协议。

所以这里单独拆出了：

1. `ImagesApi`
2. `ImagesModel`
3. `ImagesContext`
4. `AssistantImages`
5. `images-api-registry.ts`

这说明作者不是为了“抽象统一”而强行统一，而是在统一和清晰边界之间做了取舍。

## 图片链路怎么读

读图片链路时建议沿着这个顺序看：

1. `generateImages()` 如何成为统一入口。
2. registry 如何按 `model.api` 找到图片 provider。
3. 内置图片 provider 如何注册。
4. 具体 provider 如何把上游结果转成 `AssistantImages`。

这里和文本主链路的结构是相似的，但协议更窄、更独立。

## `env-api-keys.ts` 为什么值得重点读

这个文件是运行时兼容层的关键节点之一。它不只是“读环境变量”，而是在做多环境、多 provider 的认证探测。

重点看这几类逻辑：

1. 不同 provider 映射到哪些环境变量。
2. 某些 provider 是否支持多种认证源。
3. Vertex ADC 和 Bedrock 这类非单一 API key 的探测如何处理。
4. Node、Bun、浏览器环境差异如何被兼容。

尤其值得注意的是，这里并没有把所有认证逻辑塞进 provider 本身，而是抽成公共能力。这让 provider 文件可以更聚焦在协议翻译上。

## `oauth.ts` 与 `utils/oauth/` 的职责

这里是另一层认证抽象。读这部分时要关注：

1. 哪些 provider 需要 OAuth，而不是静态 API key。
2. 设备码登录或交互式登录的流程如何被统一。
3. prompt、select、callback 等交互能力如何抽象。

这套设计说明库不仅关心“能发请求”，还关心“认证过程如何被宿主应用接住”。

## `session-resources.ts` 的意义

这个文件很短，但概念上很重要。它定义的是会话级资源清理机制。

当某些 provider 或认证能力会持有 session 相关状态时，需要有一个统一位置负责清理。否则主链路虽然能跑，资源生命周期却会变乱。

这类文件通常是架构理解中的盲点，因为逻辑简单，但边界价值很高。

## `cli.ts` 怎么看

它不是整个包的核心设计来源，但很适合作为“对外能力投影”来读。

你可以用它回答两个问题：

1. 这个包最想把哪些能力暴露给终端用户。
2. 从包 API 到可执行命令，中间哪些抽象被真正复用了。

## 这一章读完后你应该回答的问题

1. 为什么图片生成单独走 `ImagesApi` 和 `images-api-registry`。
2. 认证逻辑为什么没有全部塞进 provider 文件内部。
3. `env-api-keys.ts` 为什么同时也是运行时兼容层的一部分。
4. session 资源清理为什么需要独立的统一出口。

## 建议的动手练习

你可以挑一个需要 OAuth 的 provider，再挑一个只依赖 API key 的 provider，对照回答：

1. 认证入口在哪里。
2. 凭据如何被解析。
3. provider 最终拿到的认证形态是什么。
4. 哪些部分属于公共能力，哪些部分属于 provider 特例。

这个练习会帮助你理解为什么这部分代码一定要放在主调用链之外。