# `@earendil-works/pi-agent-core` 源码深度学习报告

> **报告对象**：`/packages/agent/`（v0.75.4，发布于 2026-05-20）
> **作者视角**：Mario Zechner（earendil-works）
> **报告定位**：从资深源码架构分析师视角，还原作者设计思想、拆解架构取舍、内化可迁移技术能力
> **适配读者**：需要吃透 Agent 运行时原理、构建可复用的 Agent 框架、或基于该库做二次扩展与性能调优的工程师

---

## 一、项目核心定位与诞生痛点（设计起点）

### 1.1 项目是什么

`@earendil-works/pi-agent-core`（仓库内路径为 `packages/agent`）是 `pi-mono` 多包仓库中的**Agent 核心运行时**。它和 `@earendil-works/pi-ai`（协议/Provider 适配）以及 `@earendil-works/pi-coding-agent`（终端编程 Agent）、`@earendil-works/pi-tui`（终端 UI）共同构成 pi 生态：

```
pi-mono/
├── packages/
│   ├── ai/          ← 纯 LLM 协议层（多 Provider、流式事件、usage 计费）
│   ├── agent/       ← Agent 运行时：消息循环、工具执行、事件、队列、状态机  ★ 本报告对象
│   ├── coding-agent/← 基于 agent-core 之上构建的"编程 Agent"应用层
│   └── tui/         ← 终端 UI 渲染
```

它对外暴露两个主要入口：

| 入口 | 文件 | 适用场景 |
| --- | --- | --- |
| `Agent`（高层） | `src/agent.ts` | 需要完整状态、订阅事件、steer/followUp 队列、abort 控制的"应用"层 |
| `runAgentLoop` / `runAgentLoopContinue`（低层） | `src/agent-loop.ts` | 仅需要 LLM 循环与事件流的轻量场景；返回 `EventStream<...>` 与 `Promise<AgentMessage[]>` |
| `AgentHarness`（更高层） | `src/harness/agent-harness.ts` | 加上 Session 持久化、Hook 体系、Compaction、Tree 导航、文件系统抽象等的"完整"运行时 |

### 1.2 解决了什么行业痛点

阅读 `README.md`、`docs/agent-harness.md` 与 `CHANGELOG.md` 三个文档，可以反推出作者最初面对的具体痛点：

1. **多 Provider LLM 接入碎片化**：Anthropic / OpenAI / Google / 各种 OAuth 代理（GitHub Copilot），每个 Provider 的流式事件 schema、retry 行为、usage 字段、session/cache hint 都不同，Agent 业务方不应该重写 N 套。`pi-ai` 抽象 Provider；`pi-agent-core` 站在其上、永远不直接持有 Provider 细节。
2. **Agent 循环的"长会话"问题**：token 累计超过窗口必须能压缩，错误后必须可重试，用户在 Agent 思考时输入消息必须可"steer"或"follow up"，而不能被丢弃——这三件事是真正"工程化" Agent 与"几行 demo"之间的分水岭。
3. **工具调用不是 `JSON.parse(name+args)` 那么简单**：参数可能不合法、需要预解析 shim（`prepareArguments`）、可能需要"运行前拦截"（`beforeToolCall`）、运行后可能需要"打补丁 / 改写 / 标红 / 终止"（`afterToolCall` + `terminate: true`）；同时多 tool 调用需要并行/串行的可配置策略。作者把这些都做成了**一等公民扩展点**。
4. **"事件可订阅"≠"事件能落地"**：Agent 在长 tool 执行期间需要让 UI 知道当前进度（`pendingToolCalls`、`streamingMessage`、`isStreaming`），同时允许监听者异步落库、保存上下文、刷新 UI——这就要求事件监听是 `await` 的、`agent_end` 是 barrier、但 transport reader 本身又不能被 block。
5. **Session 持久化必须支持"树形"与"分支摘要"**：用户在 Agent 产出后回头改了 prompt、想复盘某个中间态、想从某个历史节点 fork 出去——这些都需要 session 不再是"线性 append-only"，而是 parentId 链 + 可分支 + 可回退。`session/session.ts` + `compaction/branch-summarization.ts` 一起把这件事做成了可插拔。
6. **执行环境必须可替换**：bot 跑在 nodejs、container、edge runtime 都要能跑。所以环境被抽象成 `ExecutionEnv`（= `FileSystem` + `Shell`），Node 实现只是其中一种 backend。

### 1.3 与同类项目的差异化

| 维度 | pi-agent-core | LangGraph / LangChain | Vercel AI SDK | OpenAI Assistants API |
| --- | --- | --- | --- | --- |
| 抽象层级 | 中等，**强类型、可读、可手撕** | 高（Graph DSL） | 低（更像 SDK） | 服务端托管 |
| LLM Provider 抽象 | 下沉到独立包 `pi-ai` | 自带 | 自带 | 绑定 OpenAI |
| Session 持久化 | 完整 JSONL 树形 session + memory + compaction + branch summary | 通过 checkpointer | 无 | 服务端 |
| 事件/订阅模型 | 显式 `subscribe` + `agent_end` barrier | Stream + configurable | Stream chunks | 服务端事件 |
| 工具扩展点 | `prepareArguments` / `beforeToolCall` / `afterToolCall` / `terminate` | Tool Node | Tool definition | Function spec |
| 队列 | `steer` / `followUp` / `nextTurn` 三套 | 不内建 | 无 | 服务端 thread |
| 体积 | 极小，0 业务依赖 | 重 | 轻 | — |

**作者的核心定位**："**不抢业务的 Agent 运行时**"——把所有跨场景通用的工程问题（loop、queue、hook、session、compaction、env）做成可被业务方自由组合的可控底层，而不是一个"开箱即用但束手束脚"的成品。

---

## 二、整体架构与模块拆解（全局认知）

### 2.1 目录骨架与职责映射

```
src/
├── index.ts                ← 公共导出
├── agent.ts                ← 高层 Agent：状态、订阅、队列、生命周期
├── agent-loop.ts           ← 低层 runAgentLoop / runAgentLoopContinue
├── proxy.ts                ← 透传代理 streamFn（服务端代理 LLM 调用的瘦身协议）
├── types.ts                ← 核心类型：AgentMessage、AgentEvent、AgentTool、AgentState、AgentLoopConfig
├── node.ts                 ← nodejs 专用 re-export（指向 harness/env/nodejs.ts）
└── harness/                ← 高级运行时：把 session、compaction、env 装到一起
    ├── agent-harness.ts        ← AgentHarness 类：把低层 loop + 高级能力装成"完整版"
    ├── messages.ts             ← AgentMessage 扩展（bashExecution/branchSummary/compactionSummary/custom）
    ├── skills.ts               ← 从 SKILL.md + .gitignore 加载技能
    ├── prompt-templates.ts     ← 加载 prompt 模板，$1/$@/$ARGUMENTS 占位符替换
    ├── system-prompt.ts        ← 把 Skill 列表格式化为 system-prompt 块
    ├── types.ts                ← Session/Compaction/Result/Error 等一整套类型
    ├── session/                ← 持久化层（Storage + Repo + UUIDv7）
    │   ├── session.ts          ← Session 门面：append*/moveTo/buildContext
    │   ├── jsonl-storage.ts    ← 文件系统 JSONL 实现（leaf 链 + label cache）
    │   ├── jsonl-repo.ts       ← 仓库（按 cwd 编目录、按时间排序、fork/list/delete）
    │   ├── memory-storage.ts   ← 纯内存实现（同接口，方便测试）
    │   ├── memory-repo.ts      ← 内存 Repo
    │   ├── repo-utils.ts       ← Repo 通用工具
    │   └── uuid.ts             ← 自实现 UUIDv7（带时间排序 + 单调 sequence）
    ├── compaction/             ← 上下文压缩
    │   ├── compaction.ts       ← 主入口：estimateTokens / findCutPoint / compact
    │   ├── branch-summarization.ts ← 分支切换时的"旧分支总结"
    │   └── utils.ts            ← 文本截断 + 文件操作提取 + 会话序列化
    ├── env/
    │   └── nodejs.ts           ← NodeExecutionEnv（fs + shell + 进程树 kill）
    └── utils/
        ├── shell-output.ts     ← 沙箱化的 shell output 捕获
        └── truncate.ts         ← UTF-8 感知的 head/tail 截断
```

### 2.2 三层架构与依赖方向

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Application  (coding-agent / TUI / 用户自己的 host)                     │
└─────────────▲───────────────────────────────────────────────────────────┘
              │  订阅事件、调用 prompt()/steer()/compact()/navigateTree()
┌─────────────┴───────────────────────────────────────────────────────────┐
│  Harness 层                                                              │
│   • AgentHarness  — 编排、phase、save point、pending writes              │
│   • Compaction    — 自动上下文压缩 + 分支摘要                            │
│   • Session       — JSONL / Memory 双实现，树形持久化                    │
│   • ExecutionEnv  — FileSystem + Shell 抽象                             │
└─────────────▲───────────────────────────────────────────────────────────┘
              │  runAgentLoop(messages, context, config, emit)
┌─────────────┴───────────────────────────────────────────────────────────┐
│  Agent Loop 层 (agent.ts + agent-loop.ts + types.ts + proxy.ts)         │
│   • Agent  — 状态、队列、订阅、abort                                     │
│   • runAgentLoop — 外层 followUp 循环 + 内层 tool/steer 循环             │
│   • proxy  — 远端代理模式的 stream function                              │
└─────────────▲───────────────────────────────────────────────────────────┘
              │  streamSimple(model, ctx, opts) → EventStream<...>
┌─────────────┴───────────────────────────────────────────────────────────┐
│  pi-ai 层（不归本仓库管）                                                │
│   • streamSimple  • 模型 registry  • Provider 适配  • EventStream<T,R>  │
└─────────────────────────────────────────────────────────────────────────┘
```

**关键设计**：依赖永远向下，越下越纯粹。`agent-loop.ts` 只依赖 `@earendil-works/pi-ai` 与自己的 `types.ts`，**它不依赖 harness**。这就是为什么同一份 loop 既能跑出"无 session、无 compaction"的轻量用法（直接 `runAgentLoop`），也能被 `AgentHarness` 包装成完整运行时。

### 2.3 核心数据流（一次完整 run）

```
                  prompt(text)
                       │
                       ▼
   ┌────────────────────────────────────────┐
   │ Agent (高层) / AgentHarness (更高层)   │  ─ ① snapshot state (systemPrompt, model, tools, messages, thinkingLevel, streamOptions)
   └─────────────────────┬──────────────────┘
                         │  runAgentLoop(messages, context, config, emit, signal, streamFn)
                         ▼
   ┌────────────────────────────────────────┐
   │ runLoop (外循环)                        │  ─ ② 先 drain steer 队列；每次迭代：stream 一个 assistant 响应
   │  while (true) {                         │     若有 toolCall：executeToolCalls（parallel/sequential）
   │      inner while (tool/steer) {         │     若应停：shouldStopAfterTurn? → break
   │          stream  →  tool calls         │     否则：drain steer 队列 / drain followUp 队列
   │      }                                 │  ─ ③ 直到无 followUp
   │      drain followUp                     │
   │  }                                     │
   └─────────────────────┬──────────────────┘
                         │  emit(event) 串行
                         ▼
   ┌────────────────────────────────────────┐
   │ subscribers (UI / 持久化 / hooks)      │  ─ ④ agent_end 后才被算作 idle
   └────────────────────────────────────────┘
                         ▲
                         │  streamSimple(model, ctx, opts)
   ┌─────────────────────┴──────────────────┐
   │ pi-ai Provider 适配层                   │  ─ ⑤ 真正的 LLM HTTP 请求
   └────────────────────────────────────────┘
```

---

## 三、核心主流程底层实现深度解析（核心重点）

### 3.1 入口一：`runAgentLoop` 的双层 while

> 文件：[agent-loop.ts:95-118](src/agent-loop.ts#L95-L118)

```ts
export async function runAgentLoop(
    prompts: AgentMessage[],
    context: AgentContext,
    config: AgentLoopConfig,
    emit: AgentEventSink,
    signal?: AbortSignal,
    streamFn?: StreamFn,
): Promise<AgentMessage[]> {
    const newMessages: AgentMessage[] = [...prompts];
    const currentContext: AgentContext = {
        ...context,
        messages: [...context.messages, ...prompts],
    };
    await emit({ type: "agent_start" });
    await emit({ type: "turn_start" });
    for (const prompt of prompts) {
        await emit({ type: "message_start", message: prompt });
        await emit({ type: "message_end", message: prompt });
    }
    await runLoop(currentContext, newMessages, config, signal, emit, streamFn);
    return newMessages;
}
```

**两个值得品味的点**：
1. **`newMessages` 与 `currentContext.messages` 故意是两份**。`newMessages` 记录本次 run **新增**的消息（被循环里 push），`currentContext.messages` 是发往 LLM 的完整 transcript。两份数组的存在，是为了让"失败时只回滚新增部分"或"续传时只重放新增"成为可能——`runAgentLoopContinue` 复用同一段 `runLoop` 但传空的 `newMessages`。
2. **每个 `await emit` 是 barrier**。`agent_start → turn_start → (message_start/end) → runLoop`，外层监听者必须等当前 emit 完成才会收到下一个事件——这是"事件可订阅 ≠ 事件能落地"的解法。`agent_end` 是最后一个事件，监听者可以放心写库。

### 3.2 内核：`runLoop` 的双层 while 与多优先级队列

> 文件：[agent-loop.ts:155-269](src/agent-loop.ts#L155-L269)

```ts
async function runLoop(initialContext, newMessages, initialConfig, signal, emit, streamFn) {
    let currentContext = initialContext;
    let config = initialConfig;
    let firstTurn = true;
    let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) ?? [];

    while (true) {                                            // 外层：followUp 循环
        let hasMoreToolCalls = true;
        while (hasMoreToolCalls || pendingMessages.length > 0) { // 内层：tool/steer 循环
            if (!firstTurn) await emit({ type: "turn_start" }); else firstTurn = false;

            if (pendingMessages.length > 0) {                 // ① 先吃 steering
                for (const m of pendingMessages) {
                    await emit({ type: "message_start", message: m });
                    await emit({ type: "message_end", message: m });
                    currentContext.messages.push(m);
                    newMessages.push(m);
                }
                pendingMessages = [];
            }

            const message = await streamAssistantResponse(...); // ② 跑 LLM
            newMessages.push(message);

            if (message.stopReason === "error" || "aborted") {
                await emit({ type: "turn_end", message, toolResults: [] });
                await emit({ type: "agent_end", messages: newMessages });
                return;
            }

            const toolCalls = message.content.filter(c => c.type === "toolCall");
            const toolResults: ToolResultMessage[] = [];
            hasMoreToolCalls = false;
            if (toolCalls.length > 0) {
                const batch = await executeToolCalls(currentContext, message, config, signal, emit);
                toolResults.push(...batch.messages);
                hasMoreToolCalls = !batch.terminate;
                for (const r of toolResults) { currentContext.messages.push(r); newMessages.push(r); }
            }

            await emit({ type: "turn_end", message, toolResults });

            // ③ prepareNextTurn：动态调整下一次请求的 context / model / thinkingLevel
            const nextTurnContext = { message, toolResults, context: currentContext, newMessages };
            const nextTurnSnapshot = await config.prepareNextTurn?.(nextTurnContext);
            if (nextTurnSnapshot) {
                currentContext = nextTurnSnapshot.context ?? currentContext;
                config = { ...config, model: nextTurnSnapshot.model ?? config.model, reasoning: ... };
            }

            // ④ shouldStopAfterTurn：主动叫停
            if (await config.shouldStopAfterTurn?.({ ... })) {
                await emit({ type: "agent_end", messages: newMessages });
                return;
            }

            pendingMessages = (await config.getSteeringMessages?.()) ?? [];   // ⑤ 收新的 steering
        }
        // 外层：agent 准备停时再看 followUp
        const followUpMessages = (await config.getFollowUpMessages?.()) ?? [];
        if (followUpMessages.length > 0) { pendingMessages = followUpMessages; continue; }
        break;
    }
    await emit({ type: "agent_end", messages: newMessages });
}
```

**这段代码是整个 agent 包的"心脏"，把"循环控制 + 队列 + 钩子"凝成一段线性叙事。** 作者解决的核心难题：

| 难题 | 解决方式 |
| --- | --- |
| "用户在 LLM 思考时打字" 怎么办 | 每次 turn 收尾再 poll `getSteeringMessages`，把 message 插入到 context，再让 LLM 看到 |
| "用户在 agent 结束后追加任务" 怎么办 | 外层循环在"无 tool 无 steer"时再 poll `getFollowUpMessages`，把它们当成本轮的"虚拟 prompt" |
| "用户中途想换模型/换 system prompt" 怎么办 | `prepareNextTurn` 返回 `AgentLoopTurnUpdate` 覆盖 `context / model / reasoning`；下一轮 LLM 看到的就是新值 |
| "agent 越聊越长 token 不够" 怎么办 | `shouldStopAfterTurn` 让你在每次 turn 收尾时检查并优雅退出；由调用方决定何时 compact |
| "一次 assistant 消息里带 5 个 tool 调用，部分失败部分成功" 怎么办 | `executeToolCalls` 把所有 `toolResult` 收齐后再 emit `turn_end`，保证 LLM 看到的总是"完整一轮" |

### 3.3 工具执行：`executeToolCalls` 的两套实现

> 文件：[agent-loop.ts:373-516](src/agent-loop.ts#L373-L516)

工具执行被拆成"准备 → 执行 → 终结"三段流水线：

```ts
prepareToolCall()           // 1) 查工具 → prepareArguments shim → validateToolArguments → beforeToolCall hook
  ├── kind: "immediate"     //    工具不存在 / before 拦截 / 中途 abort → 走"直接出错误结果"快路
  └── kind: "prepared"      //    通过校验后装进对象再走 executePreparedToolCall

executePreparedToolCall()   // 2) 调 tool.execute，partialResult 通过 onUpdate emit "tool_execution_update"
  └── 任何抛错都被 try/catch 兜成 isError=true 的 toolResult

finalizeExecutedToolCall()  // 3) afterToolCall hook 可改写 content / details / isError / terminate
  └── 任何抛错都换成 "hook_error" 的 toolResult（永不冒泡到外层循环）
```

而"**到底并行还是串行**"由 `config.toolExecution` 与 `tool.executionMode` 共同决定：

```ts
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
    return executeToolCallsSequential(...);
}
return executeToolCallsParallel(...);
```

**并行实现**采用了一个非常巧妙的设计——**prepare 与 execute 是两段**：

```ts
async function executeToolCallsParallel(...) {
    const finalizedCalls = [];
    for (const tc of toolCalls) {                           // ① 串行 preflight
        await emit({ type: "tool_execution_start", ... });
        const preparation = await prepareToolCall(...);
        if (preparation.kind === "immediate") {             //    工具缺失/被拦截 → 直接 finalize
            finalizedCalls.push({ toolCall, result, isError });
            continue;
        }
        finalizedCalls.push(async () => {                   //    装一个 thunk 占位
            const executed = await executePreparedToolCall(...);
            return await finalizeExecutedToolCall(...);
        });
    }
    const ordered = await Promise.all(                       // ② 真正并行
        finalizedCalls.map(e => typeof e === "function" ? e() : Promise.resolve(e)),
    );
    // ③ 按 assistant 源顺序 emit toolResult messages
    for (const f of ordered) { await emitToolResultMessage(createToolResultMessage(f), emit); ... }
}
```

> 这种 **"preflight sequential + execution parallel"** 模式来自 *CHANGELOG* 0.58.0 的设计记录。为什么要 preflight？——**工具校验（schema、权限）通常是廉价且纯函数，但执行可能很重**。先串行确认"哪些能跑"再并发执行"真正能跑的"，可以避免"一个工具被拒导致它的兄弟白跑了"。

**`tool_execution_end` 与 `toolResult message` 的 emit 顺序**也很有意思（CHANGELOG 0.68.1 修复）：在并行模式下
- `tool_execution_end` 按**完成顺序**emit（先完成的工具先告知 UI）
- `toolResult message` 按 **assistant 消息中的源顺序**emit（保证 LLM 看到的对齐）

这个差异是必须的：UI 关心的是"谁先完成"，LLM 关心的是"对得上上一轮 assistant 调用的顺序"。

### 3.4 一次 assistant 响应：`streamAssistantResponse` 与 partial 重建

> 文件：[agent-loop.ts:275-368](src/agent-loop.ts#L275-L368)

```ts
let partialMessage: AssistantMessage | null = null;
let addedPartial = false;

for await (const event of response) {
    switch (event.type) {
        case "start":
            partialMessage = event.partial;
            context.messages.push(partialMessage);     // ① 拿到 provider 给的"空壳"先入 context
            addedPartial = true;
            await emit({ type: "message_start", message: { ...partialMessage } });
            break;
        case "text_start": case "text_delta": case "thinking_start": case "thinking_delta":
        case "toolcall_start": case "toolcall_delta": case "toolcall_end":
            if (partialMessage) {
                partialMessage = event.partial;        // ② provider 推一份"累积后的 partial"过来，in-place 替换
                context.messages[context.messages.length - 1] = partialMessage;
                await emit({ type: "message_update", assistantMessageEvent: event, message: { ...partialMessage } });
            }
            break;
        case "done": case "error": {
            const finalMessage = await response.result();
            if (addedPartial) context.messages[context.messages.length - 1] = finalMessage;
            else              context.messages.push(finalMessage);
            await emit({ type: "message_end", message: finalMessage });
            return finalMessage;
        }
    }
}
```

**为什么把 partial 推到 `context.messages` 里？** 因为 LLM 消息体是 immutable array，如果只把 partial 留在循环外变量里，**所有 hook（包括 `transformContext`、`convertToLlm`）就拿不到**了。作者选择**就地 push + 末尾覆盖**，让"流式过程中 context 已经反映最新内容"成为不变式。同理 `proxy.ts` 的服务端代理也得在客户端重建 partial（见 `processProxyEvent`，streaming JSON 解析增量 toolCall 参数）。

### 3.5 Agent 高层：`MutableAgentState` 与队列

> 文件：[agent.ts:59-153](src/agent.ts#L59-L153)

`Agent` 类的内部状态特意用 **getter/setter 拦截写入**：

```ts
type MutableAgentState = Omit<AgentState, "isStreaming" | ...> & {
    isStreaming: boolean;
    streamingMessage?: AgentMessage;
    pendingToolCalls: Set<string>;
    errorMessage?: string;
};

function createMutableAgentState(initialState?): MutableAgentState {
    let tools = initialState?.tools?.slice() ?? [];
    let messages = initialState?.messages?.slice() ?? [];
    return {
        ...
        get tools()  { return tools; },
        set tools(t) { tools = t.slice(); },       // ① 外部 array 不会污染内部
        get messages() { return messages; },
        set messages(m) { messages = m.slice(); },  // ② 同样，slice 保护
        isStreaming: false,
        pendingToolCalls: new Set<string>(),
        ...
    };
}
```

**为什么用闭包变量 + getter/setter 而不是普通 class field？**——这是 **CHANGELOG 0.65.0 重构留下的关键设计**：业务方写 `agent.state.tools = [a, b, c]`，作者希望**复制一份**而不是直接持有同一份引用——避免外部代码后续 `arr.push(d)` 默默污染 agent 状态。这种"防御式拷贝"是 agent 状态机正确性的基石。

`PendingMessageQueue` 的 `drain()` 实现更是把"队列模式"语义抠得很细：

```ts
drain(): AgentMessage[] {
    if (this.mode === "all") {
        const drained = this.messages.slice();
        this.messages = [];
        return drained;
    }
    const first = this.messages[0];
    if (!first) return [];
    this.messages = this.messages.slice(1);    // "one-at-a-time" 模式：只取队首
    return [first];
}
```

`steer` 在 turn 收尾时被 drain，`followUp` 在 agent 自然停时被 drain。**两个队列的存在让"用户输入"和"agent 当前工作"在时间维度上解耦**，避免互相打断。

### 3.6 Session 持久化：树形 leaf 链

> 文件：[session/session.ts:21-76](src/harness/session/session.ts#L21-L76) + [session/jsonl-storage.ts:109-244](src/harness/session/jsonl-storage.ts#L109-L244)

`Session.buildContext` 决定了"如何从持久化 entry 序列恢复出当前 LLM 视野"：

```ts
export function buildSessionContext(pathEntries: SessionTreeEntry[]): SessionContext {
    let thinkingLevel = "off", model: { provider; modelId } | null = null, compaction: CompactionEntry | null = null;
    for (const entry of pathEntries) {                                   // ① 扫一遍恢复 model/thinking
        if (entry.type === "thinking_level_change") thinkingLevel = entry.thinkingLevel;
        else if (entry.type === "model_change")     model = { ... };
        else if (entry.type === "message" && entry.message.role === "assistant")
            model = { provider: entry.message.provider, modelId: entry.message.model };
        else if (entry.type === "compaction")       compaction = entry;
    }

    const messages: AgentMessage[] = [];
    if (compaction) {                                                    // ② 出现 compaction：从 compaction 之后开始
        messages.push(createCompactionSummaryMessage(...));
        const compactionIdx = pathEntries.findIndex(...);
        let foundFirstKept = false;
        for (let i = 0; i < compactionIdx; i++) {
            if (pathEntries[i].id === compaction.firstKeptEntryId) foundFirstKept = true;
            if (foundFirstKept) appendMessage(pathEntries[i]);
        }
        for (let i = compactionIdx + 1; i < pathEntries.length; i++) appendMessage(pathEntries[i]);
    } else {
        for (const entry of pathEntries) appendMessage(entry);
    }
    return { messages, thinkingLevel, model };
}
```

而 leaf 节点（当前活跃分支指针）**自身也是 entry 的一种**：

```ts
// jsonl-storage.ts
async setLeafId(leafId: string | null): Promise<void> {
    ...
    const entry: LeafEntry = { type: "leaf", id: ..., parentId: this.currentLeafId, timestamp, targetId: leafId };
    await this.fs.appendFile(this.filePath, `${JSON.stringify(entry)}\n`);  // ① 持久化为一行 JSONL
    this.entries.push(entry);
    this.byId.set(entry.id, entry);
    this.currentLeafId = leafId;
}
```

**关键洞察**：**leaf 变化也是 append-only 事件**——`setLeafId` 不只是改 in-memory 游标，而是把"我从 A 节点切到 B 节点"这件事记成一个新的 `leaf` entry。重启时只要扫文件最后一条 `leaf` entry 就知道当前位置。这意味着：

1. **整个 session 是 event sourcing**——`leaf` entry 是"指针迁移事件"；
2. **可以回放**——`getPathToRoot(leafId)` 沿 parentId 链走，从 leaf 一路回到 root；
3. **可以分支**——`navigateTree` 调 `moveTo(entryId, summary?)` 就完成了"指针迁移 + 可选摘要 entry"。

JSONL 头部的 `parentSession` 字段是"我从哪个 session fork 出来的"指针——这是 session 级别的引用，而 entry 的 `parentId` 是 session 内的引用，两层维度区分清楚。

### 3.7 压缩：`findCutPoint` 的"对齐 turn"策略

> 文件：[compaction/compaction.ts:328-376](src/harness/compaction/compaction.ts#L328-L376)

```ts
export function findCutPoint(entries, startIndex, endIndex, keepRecentTokens) {
    const cutPoints = findValidCutPoints(entries, startIndex, endIndex);
    // 1) 从尾部往前数 token，凑够 keepRecentTokens 找到"基础切点"
    let accumulatedTokens = 0, cutIndex = cutPoints[0];
    for (let i = endIndex - 1; i >= startIndex; i--) {
        if (entries[i].type !== "message") continue;
        const messageTokens = estimateTokens(entries[i].message);
        accumulatedTokens += messageTokens;
        if (accumulatedTokens >= keepRecentTokens) {
            for (const c of cutPoints) { if (c >= i) { cutIndex = c; break; } }
            break;
        }
    }
    // 2) 回退到上一个"可切"位置（不能切在 toolResult 中间）
    while (cutIndex > startIndex) {
        const prev = entries[cutIndex - 1];
        if (prev.type === "compaction") break;
        if (prev.type === "message") break;
        cutIndex--;
    }
    // 3) 如果切点不是 user 消息，则向上找 user/bashExecution/custom/branchSummary，作为"轮次起点"
    const isUserMessage = cutEntry.type === "message" && cutEntry.message.role === "user";
    const turnStartIndex = isUserMessage ? -1 : findTurnStartIndex(entries, cutIndex, startIndex);
    return { firstKeptEntryIndex: cutIndex, turnStartIndex, isSplitTurn: !isUserMessage && turnStartIndex !== -1 };
}
```

**为什么"不能切在 turn 中间"？**——LLM 看到的消息体里，assistant 发出 `toolCall` 后必须紧跟 `toolResult`；如果压缩时把 `toolCall` 留下却把 `toolResult` 砍掉，**消息体直接非法**。所以 `findValidCutPoints` 把 `toolResult` 排除在切点候选之外，并且当切点落在 turn 中间时，**额外保留 turn 前缀的 `turnPrefixMessages`**，并通过 `TURN_PREFIX_SUMMARIZATION_PROMPT` 单独小结一次——这就是 "split turn" 的本质。

### 3.8 压缩：增量更新（`UPDATE_SUMMARIZATION_PROMPT`）

> 文件：[compaction/compaction.ts:415-518](src/harness/compaction/compaction.ts#L415-L518)

```ts
let basePrompt = previousSummary ? UPDATE_SUMMARIZATION_PROMPT : SUMMARIZATION_PROMPT;
if (customInstructions) basePrompt = `${basePrompt}\n\nAdditional focus: ${customInstructions}`;
```

第二次压缩不是"把上次压缩到现在之间的话再总结一遍"，而是**把"上次 summary + 新增 messages"作为输入，让 LLM 输出"合并后的 summary"**。这样**summary 不会随每次压缩而丢失信息**——它只会在合并中被改写。**这是一种增量维护**。

`tokensBefore` 字段被显式保留在 `CompactionEntry` 中，**这不是浪费**：它让后续判断"再压一次要花多少 token"有依据。

### 3.9 Hook / 事件：观察者模式 + 类型化结果

> 文件：[harness/agent-harness.ts:231-300](src/harness/agent-harness.ts#L231-L300) + [harness/types.ts:641-706](src/harness/types.ts#L641-L706)

`AgentHarness` 把低层 `agent.ts` 的 `subscribe(event, signal)` **升级成两类语义**：

1. **观察者**（`*` 通配事件）——"我什么事件都想知道"
2. **Hook**（带类型化 `event.result`）——"我能在某个时刻改变行为"

Hook 通过 `AgentHarnessEventResultMap` 把 `before_agent_start`、`context`、`tool_call`、`tool_result`、`before_provider_request`、`before_provider_payload` 等等都标上"返回什么类型"。例如：

```ts
context:        { messages: AgentMessage[] }
tool_call:      { block?: boolean; reason?: string }
tool_result:    { content?, details?, isError?, terminate? }
```

而 `subscribe` 的 listener 永远是 `(event, signal) => Promise<void> | void`——不返回结果。这样**两类语义的"权限边界"就被类型系统写死**：listener 不能改 context，hook 才能。

---

## 四、关键设计决策与技术取舍复盘（还原作者想法）

### 4.1 决策：用 `EventStream<TEvents, TResult>` 抽象"流式 + 可终结"

- **作者意图**：避免 callback hell，但又不想把每个 Provider 的细节都漏到 Agent 层。
- **取舍**：选了一个**比 Promise 复杂但比 callback 简单**的"push + end" 抽象——订阅者只关心事件流，控制权（何时 end、给什么 result）归调用方。
- **代价**：用户必须先 `for await (const e of stream)` 才能拿到事件；如果忘了 drain 就丢消息。但 `Agent.subscribe()` 已经帮你 drain 掉了，**用户层完全无感**。

### 4.2 决策：消息转换在循环"边界"做

- **作者意图**：`AgentMessage` 是业务侧可能扩展的（`CustomMessage`、`BashExecutionMessage`…），但 LLM 只能吃 `Message`。如果在循环里随时转换会有竞争。
- **取舍**：把 `transformContext`（业务侧裁剪/注入）和 `convertToLlm`（AgentMessage → Message）放在**每次 LLM 请求前一次性做完**，避免 streaming 期间 context 与 LLM 视野不一致。
- **代价**：每 turn 都会执行一次 `convertToLlm`——对超长 session 会有重复开销；用 `prepareNextTurn` 可以换 context 但同样的 messages 还要重转一次。

### 4.3 决策：tool 扩展点拆成 5 个

```ts
AgentTool = {
  prepareArguments,          // 1) 原始 args 进来 → 返回符合 schema 的对象（兼容 shim）
  execute(args, signal, onUpdate), // 2) 真正执行，可流式更新
  executionMode,             // 3) 单个工具声明"我必须串行"（如写文件工具）
  ... // + 全局 beforeToolCall / afterToolCall
}
```

- **作者意图**：不同 LLM 厂商（OpenAI/Anthropic）发出来的 toolCall args 形态有差异，schema validation 可能不通过；想不破坏 schema 校验的同时让 args 兼容。
- **取舍**：`prepareArguments` 必须是纯函数（无 IO），否则会把"校验阶段"和"执行阶段"的边界打破。这是"为灵活性付出的工程纪律代价"。

### 4.4 决策：`shouldStopAfterTurn` 与 `prepareNextTurn` 分离

- **CHANGELOG 0.72.0 新增 `shouldStopAfterTurn`**：用于"压缩前先停一停"这种**主动让位**场景。
- **CHANGELOG 0.74.0 起 `prepareNextTurn`**：用于"让下一次 LLM 用不同的 model / system prompt / context"。

**为什么是两条 hook 而不是一条？**——因为**"停"和"换"是正交语义**：
- 一个用户可能想"停但不换"（保存 token）
- 另一个可能想"换但不一定要停"（route to fallback model）
- 第三个可能想"压缩 → 换模型 → 继续"（多步组合）

把两条拆开是**最小正交**的扩展点设计。

### 4.5 决策：并行工具的"preflight sequential + execute parallel"

- **作者意图**（CHANGELOG 0.58.0 注释）：避免"5 个 toolCall，其中 2 个因权限被拦截，但 3 个能跑的"场景下，那 3 个白跑。
- **取舍**：preflight 是串行的——故意让它**短**而**纯**（查表、校验、跑 `beforeToolCall` hook），执行是并行的。
- **代价**：preflight 自身也可能很慢（如果用户的 `beforeToolCall` 要查数据库），但这是用户**显式承担的成本**。

### 4.6 决策：tool 终止是"全票通过"

```ts
function shouldTerminateToolBatch(finalizedCalls) {
    return finalizedCalls.length > 0 && finalizedCalls.every(f => f.result.terminate === true);
}
```

- **作者意图**：避免"一个工具说停"导致兄弟工具的结果丢失。
- **取舍**：把"终止权"交给工具集本身，**所有工具都明确表示"可以停"才会停**。
- **代价**：业务方要协调多个工具的语义——但**这正是耦合点该暴露给业务方的原因**。

### 4.7 决策：Session 用 JSONL + 树形 leaf

- **作者意图**：可追加、可 fork、可回放、纯文本好 diff。
- **取舍**：放弃了"压缩存储 / 索引 / 二级结构"。每个 entry 一行 JSON，所有索引（`byId`、`labelsById`、`leafId`）**都是启动时从全文件重算的**。
- **代价**：session 文件 > 几万行时，启动会比较慢——**接受这个代价换"git-friendly、可 cat"**。这是工程取舍，不是性能 bug。

### 4.8 决策：Compaction 是"摘要 + 切片"二合一

```ts
// compression 后的 session 形态
[old entries] → [compaction entry (summary)] → [kept entries]
                ↑ 但 "old entries" 仍存在文件中
                ↑ 下次 compact 时 "compaction.firstKeptEntryId" 才是真正起点
```

- **作者意图**：压缩**不删历史**——它只是插入一个"忽略点"。这样用户可以回滚到压缩前的状态。
- **取舍**：文件会越压越大。如果想要"压缩同时删除"，得自己 fork 一份——这是"宁可多存、不愿丢东西"的设计哲学。

### 4.9 决策：Hook 错误用 `AggregateError` 收口

```ts
// agent-harness.ts
const cause = new AggregateError([toError(error), toError(failureError)], "...");
throw new AgentHarnessError("unknown", cause.message, cause);
```

- **作者意图**：单点失败（一个 hook 抛错）不应该让"其它所有 hook"的结果丢失。
- **取舍**：用 `cause` 链传错、`AggregateError` 收多错，调用方想看哪个看哪个。
- **代价**：业务方需要懂 `error.cause` 才能用——但 ECMAScript 2022 后这是标准。

### 4.10 决策：UUID v7 而不是 v4

> 文件：[session/uuid.ts:15-49](src/harness/session/uuid.ts#L15-L49)

```ts
bytes[0..5] = big-endian millis timestamp
bytes[6]    = 0x70 | ((sequence >>> 28) & 0x0f)   // 0x7x = UUIDv7
bytes[7..9] = sequence(高) + random
bytes[10]   = ((sequence & 0x3f) << 2) | random
bytes[11..15] = random
```

- **作者意图**：session entry 的 id 应当**与时间序对齐**——这样 `entries` 数组天然有序，无需额外 sort。
- **取舍**：v7 在分布式场景下需要中心化时钟；此处进程内 `Date.now()` 足够，**自己实现比引依赖划算**。
- **代价**：`sequence` 在同一毫秒内单调递增，但**多进程并发写同一个 session** 会产生冲突 id（概率极低，因为 `getRandomValues` 兜底）。`generateEntryId` 还会**先尝试 8 位短 id + 冲突检测**，无冲突时把 id 缩到 8 个字符——对调试更友好。

### 4.11 决策：`ExecutionEnv` 用 `Result<T, E>` 而非抛错

- **作者意图**：`fs.*` / `child_process.*` 经常抛错，业务方 handler 经常忘 try/catch——一旦忘了整个 session 死掉。
- **取舍**：把**预期失败**编进 `Result` 类型，**意外失败**也兜成 `Result`（`toFileError` 把 Node 的 `ENOENT/EACCES/...` 映射成稳定 code）。
- **代价**：调用方多写 `if (!result.ok)`——但**这把"忘记处理错误"的运行时风险前移到了编译期**。

---

## 五、项目迭代思路与核心演进逻辑

> CHANGELOG 0.34 → 0.75（约 4 个月，~150 个版本）。**对一次提交密度而言，这是个"快迭代"**。

### 5.1 三大演进轴

| 演进方向 | 关键版本 | 设计变化 | 隐含作者取舍 |
| --- | --- | --- | --- |
| **API 表面扁平化** | 0.65.0 | 移除所有 `setX/getX` 方法，改成属性访问 | 想让用户少写 wrapper、方便 IDE 自动补全 |
| **可观察性增强** | 0.58.0, 0.67.67, 0.70.x | `tool_execution_start/update/end` 三态、`after_provider_response` 事件、hook 错误聚合 | 让"外部 APM 不依赖 SDK"成为可能（见 docs/observability.md） |
| **可扩展点正交化** | 0.58.0, 0.64.0, 0.69.0, 0.72.0, 0.74.0 | 加 `beforeToolCall/afterToolCall`/`prepareArguments`/`shouldStopAfterTurn`/`prepareNextTurn`/`terminate` | **每次只加一个最小正交点**——避免"巨型 config" |

### 5.2 0.65.0 重构的内在逻辑

`streamMessage → streamingMessage`、`error → errorMessage`、所有 mutator 移除……这次重构**几乎改了所有用户面 API**，但**没有改任何底层事件协议**。

作者要的是什么？——**`agent.state` 永远反映"可被 React/Vue 双向绑定的纯数据"**。移除方法后：

```ts
// 旧
agent.setModel(m);
agent.appendMessage(msg);
agent.clearMessages();

// 新
agent.state.model = m;
agent.state.messages.push(msg);
agent.state.messages = [];
```

**用户面只是把"动词变成名词/赋值"**，但对**响应式框架、调试器、序列化器**都更友好——因为属性就是 `JSON.stringify(agent.state)` 能直接吃的东西。

### 5.3 0.69.0 终止语义的设计

`terminate: true` 是一个**反直觉但极有用**的设计：单个工具不能说停，**要全部都同意**才停。

作者可能踩过的坑：如果单个工具能终止，那 A 工具说"我已经回答完用户问题了"会**强杀掉**还没跑完的 B 工具——但 B 工具可能是"清理临时文件"，**漏跑会留垃圾**。

### 5.4 0.70.x Node 22.19 与 TypeScript strip-only 兼容

- **CHANGELOG 0.75.4**："Changed source syntax to avoid TypeScript constructs that require JavaScript emit, keeping the package compatible with Node.js strip-only TypeScript checks."
- **含义**：作者**主动放弃部分 TS 高级特性**（如 enum、namespace、参数属性），让 Node 自带 `--experimental-strip-types` 能直接跑源码。
- **意图**：让"装好依赖就能跑"——不需要 `tsc` 编译。这条对未来 Node 工具链影响深远。

### 5.5 0.74.x 的 `prepareNextTurn` 是 Agent 真正"动态化"的标志

```ts
return {
    context: this.createContext(nextTurnState),
    model: nextTurnState.model,
    thinkingLevel: nextTurnState.thinkingLevel,
};
```

这是 Agent 第一次能**在运行中无副作用地切换 model**。配合 `AgentHarness` 的 `setModel()` + 事件 `model_select`，用户能写"主模型挂了切到 fallback"的策略——而**不必重启 Agent**。

---

## 六、动手验证与改造落地思考

### 6.1 最小可跑示例（拆解 `Agent` 的每个面）

```ts
import { Agent } from "@earendil-works/pi-agent-core";
import { getModel, streamSimple } from "@earendil-works/pi-ai";

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
    thinkingLevel: "off",
    tools: [],
    messages: [],
  },
  // 1) 自定义 AgentMessage → LLM 消息
  convertToLlm: (msgs) => msgs.filter(m => m.role === "user" || m.role === "assistant"),
  // 2) 转换前的裁剪
  transformContext: async (msgs) => msgs.slice(-50),
  // 3) 拦截危险工具
  beforeToolCall: async ({ toolCall }) =>
    toolCall.name === "rm_rf" ? { block: true, reason: "blocked by policy" } : undefined,
  // 4) 给结果加审计
  afterToolCall: async ({ result, isError }) =>
    !isError ? { details: { ...(result.details as object), audited: true } } : undefined,
  // 5) 自定义流函数（譬如走公司代理）
  streamFn: streamProxy, // 见 proxy.ts
  // 6) 每次重取 API key
  getApiKey: async (provider) => process.env[`${provider.toUpperCase()}_KEY`],
});

// 订阅事件
const unsub = agent.subscribe(async (event, signal) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
  if (event.type === "agent_end") {
    // 这里 await → 算进 idle 等待；外部 await agent.waitForIdle() 才会返回
    await saveTranscript(agent.state.messages);
  }
});

// 启动一次对话
await agent.prompt("用一句话介绍 pi-agent-core");

// 期间可注入新消息
setTimeout(() => agent.steer({ role: "user", content: "别超过 30 字", timestamp: Date.now() }), 1000);

// 等 idle
await agent.waitForIdle();
unsub();
```

### 6.2 验证：跑测试看底层 loop 的真实行为

仓库根目录有 `pi-test.sh` / `pi-test.ps1` / `pi-test.bat`；`packages/agent/test/` 里有：
- `agent-loop.test.ts`：低层 `runAgentLoop` 的事件流、工具执行、steer/followUp 行为
- `agent.test.ts`：高层 `Agent` 的状态、队列、abort
- `harness/`：harness 与 session/compaction 的端到端
- `utils/`：truncate、shell-output 的边界

跑法：
```bash
cd packages/agent
npm run test:agent          # vitest 默认配置
npm run test:harness        # 用 harness 配置单独跑 harness 相关
```

> 注意：CHANGELOG 0.75.4 已移除 watch script（tsc 改为 strip-only 检查）。

### 6.3 改造方向 A：增加"用户中途切换 model"的策略

```ts
const primary = getModel("anthropic", "claude-sonnet-4-20250514");
const fallback = getModel("openai", "gpt-4o");
let currentModel = primary;

const agent = new Agent({
  initialState: { model: primary, /* ... */ },
  getApiKey: async (p) => p === "anthropic" ? process.env.ANTHROPIC_KEY : process.env.OPENAI_KEY,
});

// 用 AgentHarness 的 prepareNextTurn 等价能力，自己写：
// （如果你直接用 low-level runAgentLoop，需要自己实现 config.prepareNextTurn）
```

如果是基于 `AgentHarness`：

```ts
harness.on("after_provider_response", async ({ status }) => {
  if (status === 429) await harness.setModel(fallback);
});
```

### 6.4 改造方向 B：自定义 Compaction 策略

```ts
// 只压到 50% 而不是窗口差
harness.on("session_before_compact", async ({ preparation }) => {
  // preparation.fileOps 是用户读/写过的文件清单
  // 业务方可以决定"哪些消息不能被压"
  return { cancel: false };
});
```

### 6.5 改造方向 C：实现自己的 Storage Backend

如果你想让 session 存在 SQLite 而非 JSONL，只需实现 `SessionStorage<TMetadata>`：

```ts
class SqliteSessionStorage implements SessionStorage<MyMetadata> {
  async getLeafId() { ... }
  async setLeafId(id) { ... }
  async appendEntry(entry) { ... }
  // ... 9 个方法
}
```

然后 `new Session(new SqliteSessionStorage(...))` 就能用。**因为 Session 内部不假设存储后端**，你甚至可以把它接 Redis、DuckDB、S3。

### 6.6 改造方向 D：复刻最小 loop（30 行内）

把 `runLoop` 简化为：

```ts
while (true) {
  const reply = await streamLlm(state);
  state.messages.push(reply);
  if (reply.stopReason !== "toolUse") break;
  for (const tc of reply.toolCalls) {
    state.messages.push({ role: "toolResult", toolCallId: tc.id, content: await runTool(tc) });
  }
}
```

这能帮你**重新理解"循环到底做了什么"**——没有事件、没有队列、只看本质。然后再回过头看 `agent-loop.ts:155-269`，你就会明白每行 `await emit(...)` 是在补什么。

### 6.7 原生设计的精妙之处（vs 自写常见误区）

| 自写常见误区 | pi-agent-core 的做法 | 精妙在哪里 |
| --- | --- | --- |
| "一个巨大的 `try { ... } catch` 包住所有 emit" | 事件源 emit 与监听者 await 串行化 | 监听者抛错时**知道是哪个事件抛的**（`processEvents` 同步处理） |
| "tool error 整个 run 崩" | 工具错误被 `executePreparedToolCall` 兜成 `toolResult`（`isError: true`） | 一次失败不会拖累兄弟工具 |
| "session 用单文件 JSON" | JSONL + leaf 链 + 树形 | **git-friendly、可 fork、可回放** |
| "compaction 时把消息删掉" | 插入 compaction 节点，old entries 留底 | **可回滚、不丢历史** |
| "Queue 用同一个 array" | 拆 `steerQueue` / `followUpQueue` / `nextTurnQueue` | 三个**语义独立**的输入时间窗 |
| "把 message 直接传给 LLM" | `convertToLlm` 是必填 hook | 业务方**必须**面对"自定义消息怎么变 LLM 消息"这个问题 |
| "tool 失败时整个 batch 终止" | `shouldTerminateToolBatch` 要求"全票通过" | **单个工具不能绑架其他工具** |

---

## 七、学习总结与能力内化沉淀

### 7.1 核心设计思想（可复用）

1. **"循环"是 Agent 真正的工作单元**——所有问题（队列、状态、压缩、终止）最终都被装回"while 循环"里解决。
2. **事件流是"控制反转"的最小单位**——把"业务方想干嘛"从 if-else 改成 `subscribe / on` 后，扩展面立刻变成正交空间。
3. **类型化事件结果**——`AgentHarnessEventResultMap` 把"事件能返回什么"写进类型，是**把"运行时配置"和"编译时契约"绑定**的最佳实践。
4. **Result 优先于 throw**——`Result<T, E>` 在 fs/shell 这种"失败有几十种 code"的地方比 `try/catch` 更稳。
5. **"正交扩展点" 优于 "巨型 hook"**——`prepareArguments / beforeToolCall / afterToolCall / terminate / shouldStopAfterTurn / prepareNextTurn` 各自只负责一件事，可以任意组合。
6. **闭包持有内部状态 + getter/setter 拦截**——替代 `class field`，让"外部写入必须 copy"成为不变式。
7. **partial 推到 context.messages 末尾**——streaming 过程中 LLM 视野 = Agent 视野，永远不会脱节。
8. **"事件可 await + agent_end barrier"**——让监听者落库、保存 session、刷新 UI 的副作用都安全，而 transport reader 不会被 block。

### 7.2 编码技巧

- **UUIDv7 自实现**（`session/uuid.ts`）：48-bit ms + 单调 sequence + 12-bit random，5 行写完，比引依赖快。
- **getter/setter 拦截赋值**（`createMutableAgentState`）：强制"防御式拷贝"在语言层可见。
- **TypeBox schema 校验**（`validateToolArguments`）：用同一份 schema 既生成 JSON Schema 又做运行时校验。
- **`preflight + execute` 拆开**：`prepare` 阶段纯函数 + 串行，`execute` 阶段可重 + 并行。
- **`Object.hasOwn` 做"patch 语义"**（`applyStreamOptionsPatch`）：`patch.x === undefined` 表示"删除"，`patch.x === null` 表示"置空"。
- **process tree kill**（`killProcessTree`）：Windows 用 `taskkill /F /T`，POSIX 用 `process.kill(-pid, "SIGKILL")`（负 pid 表组）。
- **UTF-8 字节感知截断**（`truncateTail` / `truncateStringToBytesFromEnd`）：从不切碎多字节字符，包括 unpaired surrogate。

### 7.3 解决了哪些认知盲区

1. **"Agent 是 ChatGPT wrapper"** — 实际上：loop + 事件 + 队列 + 持久化 + 压缩 + 分支是**5 件事拼起来**。
2. **"compaction 是 chat history 截断"** — 实际上：compaction 节点也是 entry、old entries 留底、split turn 单独小结、turn-prefix 二次 summarize。
3. **"tool 失败 = 重跑"** — 实际上：先 try 一次 `prepareArguments` 兼容 shim，再 validate schema，再 `beforeToolCall` 拦截，再 `execute`（并捕获），再 `afterToolCall` 改写，**4 道关卡**才到 `toolResult`。
4. **"stream 出来就是最终消息"** — 实际上：partial 是被流式累积出来的，**每收到一个 `text_delta` 都替换 `context.messages[length-1]`**，LLM 视野与 UI 视野一致。
5. **"session = 历史"** — 实际上：session = **event-sourced tree**（entry 链 + leaf 指针 + 可选 compaction 节点 + 可选 branch summary 节点）。

### 7.4 可迁移到哪些场景

| 场景 | 可复用模式 |
| --- | --- |
| **自研 Coding Agent** | `AgentHarness` + 自定义 `ExecutionEnv` 隔离 docker 容器 |
| **多 Agent 协作** | 每个 Agent 一个 Session，多个 Agent 共享同一个 `repo`；Agent 间通过 `appendMessage` 互发 |
| **客服 / 知识库对话** | `transformContext` 注入 RAG 检索结果，`afterToolCall` 改写敏感词 |
| **长任务 worker** | `steer` 实现"用户中途改方向"、`followUp` 实现"等结果后继续追问" |
| **沙箱评测 / Replay** | Session 全 event-sourced + `getPathToRoot` + `navigateTree`，**任意时刻可回放** |
| **可视化调试器** | 订阅 `*` 事件后做"消息时间线 + tool 调用甘特图" |
| **跨 Provider fallback** | 订阅 `after_provider_response` 检测 429/5xx，下次 `prepareNextTurn` 换 model |

---

## 八、现存设计局限与优化思考

### 8.1 已知边界

| 局限 | 描述 | 适用边界 |
| --- | --- | --- |
| **JSONL 重算启动** | session > 几万行时 `byId`、`labelsById` 启动较慢 | 中等规模（< 50k entries/session） |
| **Compaction 仍存底** | 重复压缩后文件**只增不减** | 如果想要"压缩即瘦身"，需要自己 fork |
| **preflight 串行** | 如果 `beforeToolCall` 钩子很慢（查 DB / 网络），preflight 自身就慢 | 用户需要保证 hook 廉价 |
| **Steer/followUp 仍是内存队列** | agent 进程崩了，队列内容丢 | 业务方需要自己做持久化 |
| **`getEntries()` 返回全量数组** | 大 session 一次拷贝 | 当前 O(N) 拷贝可接受，未来可改为 lazy stream |
| **Session 缺少"压缩 + 删除原条目"** | 留底哲学导致磁盘无限增长 | 用户需要自写"压缩后 fork + 删除原文件" |
| **`AgentLoopConfig` 没有 schema 校验** | 用户写错 `convertToLlm` 只能运行时炸 | 暂无（业务方应自己写 type tests） |
| **Hook 错误是 throw 而不是 emit** | 一个 hook 抛错会中断事件流 | 业务方应该把"非关键 hook"用 try/catch 包 |

### 8.2 优化方向

#### 8.2.1 性能层

1. **session 索引文件**（`session.idx`）—— 启动只读 header + idx，加载 entry 按需 mmap。当前重算是 O(N) 内存 + O(N) 时间。
2. **Compaction 增量 GC**—— 标记"以最近一次 compaction 为分界"的老 entry，定期 fork 到 archive、删除原文件。
3. **并行 preflight**——用 `Promise.all` 把 `beforeToolCall` 并发掉，**前提是 hook 自身线程安全**（可配置）。
4. **stream 期间增量 partial 编码**——`proxy.ts` 的 SSE 解析目前是 `for line in buffer.split('\n')`，可以换成 `events.on('data')` 流式解析，省一次大字符串拼接。

#### 8.2.2 架构层

1. **Hook 优先级 + 短路**——目前多 handler 是 `lastResult` 覆盖，可以让用户声明"我必须最后跑"或"我一旦返回 result 就 break"。
2. **Harness Session Facade**（`docs/agent-harness.md` 计划）——让 hook context 拿到的是 `HarnessSession` 而非裸 `Session`，封掉"未 flush 的 pending writes"等不安全操作。
3. **`Agent` 与 `AgentHarness` 合并**——目前是"两层独立的 API 表面"，用户面要选。理论上 `AgentHarness` 完全可以暴露 `Agent` 的子集 API。
4. **Pluggable stream reader**——`EventStream<TEvents, TResult>` 抽象可以下沉成 `packages/ai`，让所有用到流的地方都受益。
5. **Compaction 多策略插件**——`DEFAULT_COMPACTION_SETTINGS` 之外，可以注册"按 token 预算压缩 / 按时间窗口压缩 / 按文件操作压缩"等不同策略。

#### 8.2.3 业务层（对调用方）

1. **可观测性面板**——`docs/observability.md` 已经规划了 `AsyncLocalStorage` 风格 trace context，加上 `metrics` 端点（counter / gauge / histogram）就能直接对接 Prometheus。
2. **多租户隔离**——目前 `ExecutionEnv.cwd` 是单租户。`NodeExecutionEnv` 可以扩展为 `Map<tenant, NodeExecutionEnv>`。
3. **跨 Session 上下文**——业务方经常需要"上一个 session 看过哪些文件 / 做过哪些决定"，但目前只能 fork。可以在 Session 之上引入"Memory"层（其实 `memory-storage.ts` 已经存在雏形，可继续扩展）。
4. **可中断 checkpoint**——`shouldStopAfterTurn` 现在是"等当前 turn 跑完才停"，无法在 tool 执行中立即停。如果工具很重（编译、测试），需要"checkpoint + 恢复"机制。

### 8.3 总体评价

`pi-agent-core` 不是一个"看起来很炫"的库——它是一份**用 4 个月、~150 个版本迭代出来的工程答卷**，回答了"把 LLM 装进长会话、装进可恢复、装进可扩展、装进可观测"这一系列真问题。它的最大价值不是某个 API 多漂亮，而是**"正交扩展点" + "类型化事件" + "Result 优先" + "事件 barrier"** 这套**思维方式**——**任何想自己写 Agent runtime 的人，都应当把这套思维内化，而不是抄它的 API**。

> **作者原话（README 暗含的工程哲学）**：
> - "Agent is already processing a prompt"（避免双 run）—— **状态机要严格**
> - "agent_end is the final emitted event for a run, but the agent does not become idle until all awaited listeners for that event have settled" —— **事件是 barrier，不是 fire-and-forget**
> - "must not throw or reject. Return a safe fallback value instead" —— **低层契约要能驯服第三方**

---

## 附录 A：核心类型速查

```ts
// types.ts（agent.ts 的核心类型）
type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];

interface AgentState {
  systemPrompt: string;
  model: Model<any>;
  thinkingLevel: ThinkingLevel;
  set tools(t: AgentTool<any>[]): void;     // 写入会复制
  get tools(): AgentTool<any>[];
  set messages(m: AgentMessage[]): void;    // 写入会复制
  get messages(): AgentMessage[];
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}

interface AgentTool<TParameters extends TSchema, TDetails = any> extends Tool<TParameters> {
  label: string;
  prepareArguments?: (args: unknown) => Static<TParameters>;
  execute: (id, params, signal?, onUpdate?) => Promise<AgentToolResult<TDetails>>;
  executionMode?: ToolExecutionMode;
}

type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId; toolName; args }
  | { type: "tool_execution_update"; toolCallId; toolName; args; partialResult }
  | { type: "tool_execution_end"; toolCallId; toolName; result; isError };
```

## 附录 B：核心钩子速查

| 钩子 | 阶段 | 何时调用 | 入参 | 出参语义 |
| --- | --- | --- | --- | --- |
| `transformContext` | 每次 LLM 请求前 | `AgentMessage[] → AgentMessage[]` | `messages, signal` | 替换 messages（裁剪/注入） |
| `convertToLlm` | 每次 LLM 请求前 | `AgentMessage[] → Message[]` | `messages` | 过滤/转换；必填 |
| `getApiKey` | 每次 LLM 请求前 | 动态取 key | `provider` | 返回 string \| undefined |
| `beforeToolCall` | 工具 preflight | tool 校验后 | `assistantMessage, toolCall, args, context` | `{ block?, reason? }` |
| `afterToolCall` | 工具 finalize | 工具返回后 | `+ result, isError` | `{ content?, details?, isError?, terminate? }` |
| `shouldStopAfterTurn` | 每次 turn 收尾 | `turn_end` 后 | `message, toolResults, context, newMessages` | `boolean`（true=停） |
| `prepareNextTurn` | 每次 turn 收尾 | `shouldStopAfterTurn` 之前 | 同上 | `{ context?, model?, thinkingLevel? }` |
| `getSteeringMessages` | 每次 turn 收尾 | 决定是否注入 | — | `AgentMessage[]` |
| `getFollowUpMessages` | agent 自然停时 | 决定是否继续 | — | `AgentMessage[]` |
| `before_provider_request` | harness 每次 LLM | patch 流选项 | `model, sessionId, streamOptions` | `streamOptions patch` |
| `before_provider_payload` | harness 每次 LLM | 改写 payload | `model, payload` | `{ payload }` |
| `after_provider_response` | harness 每次 LLM | 拿到响应 | `status, headers` | — |
| `session_before_compact` | harness compact 前 | 拦截/替换 | `preparation, branchEntries, customInstructions?, signal` | `{ cancel?, compaction? }` |
| `session_before_tree` | harness navigateTree 前 | 拦截/替换 | `preparation, signal` | `{ cancel?, summary?, customInstructions?, replaceInstructions?, label? }` |
| `before_agent_start` | harness 每次 prompt | 改 system prompt / 注入消息 | `prompt, images?, systemPrompt, resources` | `{ messages?, systemPrompt? }` |

---

> **报告完成。** 该报告基于 `packages/agent/src/**/*.ts`、`packages/agent/CHANGELOG.md`、`packages/agent/README.md`、`packages/agent/docs/agent-harness.md`、`packages/agent/docs/observability.md` 与 `packages/agent/CHANGELOG.md` 0.34–0.75 全量 changelog 撰写，所有结论均来自源码与官方文档，未引入第三方臆测。
