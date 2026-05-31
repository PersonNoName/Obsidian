# 教程 03：模型目录、计费与生成文件

这一章回答的是另一个基础问题：调用方是怎么从 provider/model id 拿到统一模型描述的，以及 generated 文件在系统里到底承担什么角色。

## 推荐阅读文件

1. `packages/ai/src/models.ts`
2. `packages/ai/src/models.generated.ts`
3. `packages/ai/scripts/generate-models.ts`
4. `packages/ai/src/image-models.ts`
5. `packages/ai/src/image-models.generated.ts`
6. `packages/ai/scripts/generate-image-models.ts`

## `models.ts` 在系统里的位置

`stream.ts` 负责分发，`types.ts` 负责定义协议，`models.ts` 负责提供“这个模型是什么、能力如何、价格如何”的统一元数据。

它最核心的任务有 4 个：

1. 从 generated 常量初始化模型注册表。
2. 提供 `getModel()`、`getProviders()`、`getModels()` 等查询能力。
3. 提供 usage 到 cost 的统一计算。
4. 处理 thinking level 的能力映射和降级。

## 为什么模型清单是 generated 文件

不要把 `models.generated.ts` 当成普通源码。它更像一份构建产物或数据快照。

这样设计的好处是：

1. 运行时查模型更快，不需要动态组装。
2. 手写逻辑和模型数据分离，降低维护成本。
3. 可以通过脚本统一生成、审查和更新模型能力。

因此学习时不要逐行精读 `models.generated.ts`，而要重点理解：

1. 它的数据结构长什么样。
2. `models.ts` 如何消费它。
3. `generate-models.ts` 用什么规则生成它。

## `getModel()` 为什么重要

`getModel()` 表面上只是查表，实际上它把调用方和底层模型目录隔离开了。

调用方不用关心：

1. 模型条目从哪来。
2. 结构是手写还是生成。
3. reasoning、cost、api、provider 这些字段怎么组织。

调用方只拿到一个统一 `Model`，后续所有主链路都围绕这个对象展开。

## 计费为什么也放在这里

`calculateCost()` 放在 `models.ts` 很合理，因为 cost 本质上是“usage 和模型元数据”的函数。

它不是 provider 网络层逻辑，也不是业务层逻辑。把它放在模型目录层，能让计费规则和模型参数保持一致。

你读这里时要注意：

1. usage 本身是运行时产生的。
2. 单价来自模型元数据。
3. 这两者结合后才得到统一的 cost 结果。

## thinking level 为什么需要 clamp

这部分设计很值得细看，因为它体现了“统一抽象”和“provider 差异”之间的折衷。

调用方想表达的是统一级别，比如：

- `off`
- `minimal`
- `low`
- `medium`
- `high`
- `xhigh`

但每个模型真正支持的等级并不一样，所以系统需要：

1. 描述模型支持哪些 level。
2. 在请求超出支持范围时做合理降级。
3. 保证跨 provider 的调用体验尽量一致。

`clampThinkingLevel()` 本质上就是这层兼容逻辑的入口。

## 图片模型为什么单独一套

文本模型和图片模型都属于“模型目录”，但它们不是一个东西。

图片模型单独走：

1. `image-models.ts`
2. `image-models.generated.ts`
3. `generate-image-models.ts`

原因很简单：它们服务的是另一条能力线，输入输出协议、调用方式、stop reason、返回内容结构都不同。把它们硬塞进同一套目录，反而会污染抽象。

## 学习这一章时建议回答的问题

1. 为什么模型数据更适合生成为常量，而不是手写分散在 provider 文件中。
2. `getModel()` 返回的统一 `Model` 如何成为后续调用链的输入。
3. usage 和 cost 为什么在模型层汇合。
4. thinking level 的统一抽象为什么必须包含降级逻辑。
5. 为什么图片模型目录不能直接复用文本模型目录。

## 建议的动手练习

自己挑一个模型做追踪：

1. 在 `models.generated.ts` 找到它的条目。
2. 看 `models.ts` 如何把它注册进 registry。
3. 看这个模型的 `api` 最终会把请求带到哪个 provider 实现。

这个练习能把“模型目录层”和“注册分发层”真正串起来。