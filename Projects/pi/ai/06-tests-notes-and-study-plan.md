# 教程 06：测试阅读、源码笔记与 5 天学习计划

前面 5 章解决的是“源码怎么组织”。这一章解决的是“怎么学得更快”，也就是如何用测试反推实现、如何做自己的源码笔记，以及如何在几天内把这块代码读出体系感。

## 为什么测试文件很值得读

实现文件告诉你代码怎么写，测试文件告诉你作者想保证什么行为。两者关注点不同。

对于 `packages/ai` 这种适配层很多、provider 差异很大的包来说，测试尤其重要，因为它能帮你分清：

1. 哪些行为是设计意图。
2. 哪些只是某个实现的局部写法。
3. 哪些边界条件是作者特别在意的。

## 建议优先阅读的测试

先从这些测试开始：

1. `packages/ai/test/stream.test.ts`
2. `packages/ai/test/abort.test.ts`
3. `packages/ai/test/cross-provider-handoff.test.ts`
4. `packages/ai/test/env-api-keys.test.ts`
5. `packages/ai/test/images.test.ts`
6. `packages/ai/test/openrouter-images.test.ts`
7. `packages/ai/test/lazy-module-load.test.ts`
8. `packages/ai/test/validation.test.ts`

这几类测试分别帮助你理解：

1. 统一流行为
2. 中断与错误边界
3. 跨 provider 上下文连续性
4. 认证与环境探测
5. 图片支线
6. lazy load 的运行时保证
7. 工具参数与协议校验

## 按测试反推实现的方法

建议每读一个测试，就做这 4 步：

1. 先用一句话写下它验证的行为。
2. 再找它命中的主入口函数。
3. 再找落到哪个 provider 或 helper。
4. 最后判断它在保护的是“统一协议”还是“某个 provider 特性”。

如果你能坚持这么读，理解速度会比直接硬啃 provider 文件快很多。

## 推荐的源码笔记方式

建议自己维护 3 张表。

### 表 1：入口表

记录这些文件分别负责什么：

1. `index.ts`
2. `stream.ts`
3. `api-registry.ts`
4. `models.ts`
5. `types.ts`

### 表 2：provider 对照表

每读完一个 provider，补这些列：

1. provider 文件名
2. 对应 `api`
3. 上游 SDK 或 HTTP 协议
4. 是否支持 thinking
5. 是否支持 tool calling
6. 是否有 OAuth 特殊逻辑
7. 是否依赖共享 helper

### 表 3：测试对照表

把测试名和行为标签对应起来，例如：

1. abort
2. lazy load
3. prompt cache
4. tool result images
5. reasoning replay
6. cross-provider handoff

这三张表会把你的阅读从“看过代码”提升到“能定位职责和行为”。

## 一个 5 天学习安排

### 第 1 天：主链路

读：

1. `index.ts`
2. `stream.ts`
3. `api-registry.ts`
4. `providers/register-builtins.ts`

输出：

1. 一张主调用链图
2. 一份“为什么按 `api` 分发”的说明

### 第 2 天：统一协议与模型元数据

读：

1. `types.ts`
2. `utils/event-stream.ts`
3. `models.ts`
4. `models.generated.ts`

输出：

1. 一张消息模型图
2. 一份 thinking level / cost 计算说明

### 第 3 天：provider 适配

读：

1. 一个最熟悉的 provider
2. 一个差异较大的 provider
3. 对应共享 helper

输出：

1. 一张 provider 对照表
2. 一份“统一协议如何落到厂商协议”的总结

### 第 4 天：支线能力

读：

1. 图片生成链路
2. `env-api-keys.ts`
3. `oauth.ts`
4. `session-resources.ts`
5. `cli.ts`

输出：

1. 一份支线模块职责图
2. 一份运行时兼容要点清单

### 第 5 天：测试回查

读：

1. 前面列出的重点测试
2. 与它们对应的实现文件

输出：

1. 一份行为到实现的映射表
2. 一份自己的最终总结

## 最后的阅读建议

第一次读不要试图覆盖所有 provider 细节。最稳的顺序仍然是：

1. 先抓统一抽象。
2. 再抓注册与分发。
3. 再深挖 1 到 2 个 provider。
4. 最后用测试把理解闭环。

如果你一开始就钻进某个 provider 的 SDK 调用细节，很容易只看到厂商差异，而看不到这个包真正的设计价值。