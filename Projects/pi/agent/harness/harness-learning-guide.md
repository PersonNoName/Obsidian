
> 本指南基于 `/Users/robot/Documents/Projects/pi/packages/agent/src/harness/` 下的源码编写,目的是帮助读者快速建立对该模块的整体认知。源码总计约 5834 行,被拆分为 5 个子系统 + 共享类型层。

---

## 0. 目录速览

```
harness/
├── agent-harness.ts          # 顶层编排器:AgentHarness 类 (995 行)
├── types.ts                  # 共享类型 + 错误码 + Result 工具 (815 行)
├── messages.ts               # 消息转换:AgentMessage <-> LLM Message (164 行)
├── skills.ts                 # 技能文件加载与解析 (375 行)
├── prompt-templates.ts       # 提示模板加载与参数替换 (267 行)
├── system-prompt.ts          # 把技能注入系统提示 (34 行)
│
├── env/                      # 执行环境(FileSystem + Shell)实现
│   └── nodejs.ts             # 基于 Node.js fs/child_process 的实现 (528 行)
│
├── session/                  # 会话持久化层(树形结构)
│   ├── session.ts            # Session 业务逻辑 (252 行)
│   ├── jsonl-storage.ts      # JSONL 文件存储后端 (293 行)
│   ├── jsonl-repo.ts         # JSONL 仓库(create/open/list/fork) (177 行)
│   ├── memory-storage.ts     # 内存版存储(测试/嵌入式) (131 行)
│   ├── memory-repo.ts        # 内存版仓库 (50 行)
│   ├── repo-utils.ts         # 仓库共享工具 (51 行)
│   └── uuid.ts               # UUID v7 生成 (54 行)
│
├── compaction/               # 上下文压缩
│   ├── compaction.ts         # 主压缩流程 (755 行)
│   ├── branch-summarization.ts # 分支摘要(树导航) (262 行)
│   └── utils.ts              # 文件操作/序列化工具 (144 行)
│
└── utils/                    # 通用工具
    ├── truncate.ts           # 文本截断(head/tail) (344 行)
    └── shell-output.ts       # Shell 输出捕获与溢出落盘 (143 行)
```

---

## 1. 整体架构:分层视角

harness 是对底层 `agent-loop`(LLM 调用 + 工具循环)的"包装层"。它把单轮 LLM 循环提升为"会话化、可扩展、可持久化、可中断"的运行时。

```
┌────────────────────────────────────────────────────────────────────┐
│  Application / CLI / TUI                                           │
└────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  AgentHarness  (agent-harness.ts)                                  │
│  ── 阶段机 (idle / turn / compaction / branch_summary / retry)    │
│  ── 队列: steer / followUp / nextTurn                              │
│  ── 钩子系统: context / tool_call / before_provider_* / ...        │
│  ── 订阅系统: subscribe(listener) / on(type, handler)               │
│  ── pendingSessionWrites: 运行中产生的会话写入暂存                  │
└────────────────────────────────────────────────────────────────────┘
        │                │                  │              │
        ▼                ▼                  ▼              ▼
┌──────────────┐ ┌──────────────┐  ┌────────────────┐ ┌──────────────┐
│ runAgentLoop │ │  Session     │  │ Compaction /   │ │ ExecutionEnv │
│ (../agent-   │ │  (session.ts)│  │ BranchSummary  │ │ (env/)       │
│  loop.ts)    │ │              │  │ (compaction/)  │ │              │
└──────────────┘ └──────────────┘  └────────────────┘ └──────────────┘
        │                │                  │              │
        ▼                ▼                  ▼              ▼
┌──────────────┐ ┌──────────────┐  ┌────────────────┐ ┌──────────────┐
│ Provider SDK │ │ SessionRepo  │  │ LLM (summary)  │ │ node:fs /    │
│ (pi-ai)      │ │ (jsonl/mem)  │  │                │ │ child_process│
└──────────────┘ └──────────────┘  └────────────────┘ └──────────────┘
```

**关键设计点**

1. **层与层通过接口解耦**:`SessionStorage` / `SessionRepo` / `ExecutionEnv` 都是接口,harness 只依赖接口,不关心 JSONL 还是内存、Node 还是其他运行时。
2. **状态类型化**:`Result<T,E>` 表达底层能力的"预期失败"(`{ ok, value } | { ok, error }`),高阶 API(`Session` / `AgentHarness`)则用抛错表达"异常",形成"底层不抛 / 上层抛"的分层。
3. **副作用都集中到 Session/Env**:LLM 调用以外的副作用(写文件、改会话、跑命令)都封装在 `Session` 与 `ExecutionEnv`,便于替换/测试。

---

## 2. 顶层:AgentHarness(`agent-harness.ts`)

### 2.1 角色

`AgentHarness` 是面向应用层的**唯一入口**。它把"用户输入一句话 → 拿到 assistant 回应"包装成一个**事务**:包含阶段切换、状态机保护、消息队列、钩子链、订阅通知、失败回滚。

应用代码不直接调用 `runAgentLoop`,而是:

```ts
const harness = new AgentHarness({
  env, session, model, tools, resources, systemPrompt, getApiKeyAndHeaders, ...
});
const finalMessage = await harness.prompt("帮我重构这段代码");
// 也可显式触发技能或提示模板
await harness.skill("code-review");
await harness.promptFromTemplate("release-notes", ["v1.2.0"]);
// 也可在运行中注入新输入
await harness.steer("顺便把测试也改了");
await harness.followUp("改完提交");
```

### 2.2 内部状态机(phase)

```ts
type AgentHarnessPhase = "idle" | "turn" | "compaction" | "branch_summary" | "retry";
```

每个结构性操作(turn / compact / navigateTree)在执行前都会做 `phase !== "idle" → 抛 busy`,并同步把 phase 置为新值,直到 `finally` 中复位。`steer / followUp` 在 turn 运行中允许入队,`nextTurn` 任何时候都可入队。

```
                ┌───────────────┐
                │     idle      │◀────────────────┐
                └──────┬────────┘                 │
            prompt()  │  │ compact()              │
                       ▼  │                       │
                ┌───────────────┐                 │
                │     turn      │── agent_end ───▶│
                └───────────────┘                 │
                       │                          │
                       │  abort() / error         │
                       └──────────────────────────┘
                ┌───────────────┐
                │  compaction   │── done ─────────▶│
                └───────────────┘
                ┌───────────────────┐
                │  branch_summary   │── done ──────▶│
                └───────────────────┘
```

### 2.3 Turn 快照(`createTurnState`)

每一轮开始时,harness 调用 `createTurnState()` 拍下"这次 LLM 调用所需的全部不变输入":

```
AgentHarnessTurnState {
  messages         // Session 当前分支线性化的消息
  resources        // 技能 + 提示模板(浅拷贝)
  streamOptions    // 供应商请求选项(浅拷贝)
  sessionId        // 用于 provider session id
  systemPrompt     // 字符串或回调求值结果
  model            // 当前模型
  thinkingLevel    // 思维链强度
  tools            // 全部工具
  activeTools      // 本次启用的工具子集
}
```

**重要语义**:turn 中途的 `setModel / setThinkingLevel / setTools / setResources / setStreamOptions` 不会影响**当前** turn,只会在 `prepareNextTurn` 时生成新快照。

### 2.4 消息队列(steer / followUp / nextTurn)

| 队列 | 何时入队 | 何时消费 | 模式 |
| --- | --- | --- | --- |
| `steerQueue` | turn 运行中 | "下一条 user 输入"出现时插入当前对话 | `one-at-a-time` / `all` |
| `followUpQueue` | turn 运行中 | turn 真正结束后作为下一轮首个输入 | `one-at-a-time` / `all` |
| `nextTurnQueue` | 任何时候 | 当前 turn 即将发起下一次时合并到 messages | 总是全部取出 |

`drainQueuedMessages()` 在 `queue_update` 事件之前 splice 出队,若订阅者抛错会把消息 `unshift` 回去,保证不丢。

### 2.5 钩子与订阅

`AgentHarness` 暴露两类扩展点:

```
┌──────────────────────────┬────────────────────┬─────────────────────────────────────┐
│ Hook (on)                │ 用途               │ 关键返回                            │
├──────────────────────────┼────────────────────┼─────────────────────────────────────┤
│ before_agent_start       │ prompt 之前改 msgs │ { messages?, systemPrompt? }        │
│ context                  │ 改 LLM 上下文      │ { messages }                        │
│ tool_call                │ 拦截/阻断工具调用  │ { block?, reason? }                 │
│ tool_result              │ 改写工具结果       │ { content?, details?, isError?,     │
│                          │                    │   terminate? }                      │
│ before_provider_request  │ 改 streamOptions   │ { streamOptions }                   │
│ before_provider_payload  │ 改真实 payload     │ { payload }                         │
│ session_before_compact   │ 自定义压缩         │ { cancel?, compaction? }            │
│ session_before_tree      │ 自定义分支摘要     │ { cancel?, summary?,                │
│                          │                    │   customInstructions?, label? }     │
└──────────────────────────┴────────────────────┴─────────────────────────────────────┘
```

**自有事件**(`AgentHarnessOwnEvent`)通过 `subscribe(listener)` 暴露,典型事件:

```
queue_update | save_point | abort | settled
model_select | thinking_level_select | resources_update
session_compact | session_tree
after_provider_response
```

底层 `runAgentLoop` 产出的 `AgentEvent` 也通过同一总线透传(`message_start / message_end / turn_end / agent_end` 等)。

**钩子 ↔ 订阅的区别**:
- `on(type, handler)` 是**有返回值的处理器**,按注册顺序串行调用,最后非 undefined 的返回值被采纳;
- `subscribe(listener)` 是**观察者**,所有订阅者都会被通知,无返回值。

### 2.6 运行流程(顺序图)

```
应用                AgentHarness              runAgentLoop            Session            Provider
 │                       │                          │                   │                  │
 │  prompt(text)         │                          │                   │                  │
 │ ────────────────────▶ │  createTurnState()       │                   │                  │
 │                       │ ──────────────────────▶  │                   │ buildContext()   │
 │                       │                          │ ────────────────▶ │                  │
 │                       │                          │                   │                  │
 │                       │  runAgentLoop(messages,  │                   │                  │
 │                       │    config, event, ... )  │                   │                  │
 │                       │ ──────────────────────▶  │                   │                  │
 │                       │                          │  streamSimple()   │                  │
 │                       │                          │ ────────────────▶ │                  │
 │                       │                          │                   │     LLM HTTP     │
 │                       │                          │                   │ ────────────────▶│
 │                       │                          │                   │     ◀ SSE chunk  │
 │                       │                          │   AgentEvent      │ ◀────────────────│
 │                       │ ◀────────────────────    │                   │                  │
 │  handleAgentEvent()   │                          │                   │                  │
 │  appendMessage()      │                          │                   │                  │
 │ ─────────────────────────────────────────────────────────────────▶  │                  │
 │                       │  emit(queue_update,...)   │                   │                  │
 │                       │  emit(after_provider_...) │                   │                  │
 │  AgentHarnessEvent    │                          │                   │                  │
 │ ◀──────────────────── │                          │                   │                  │
 │                       │                          │                   │                  │
 │                       │  agent_end               │                   │                  │
 │                       │  flushPendingSessionWrites()                 │                  │
 │                       │ ───────────────────────────────────────────▶  │                  │
 │                       │                          │                   │                  │
 │  AssistantMessage     │                          │                   │                  │
 │ ◀──────────────────── │                          │                   │                  │
```

### 2.7 关键工具函数

- `applyStreamOptionsPatch` / `mergeHeaders`:对 `headers/metadata` 做"按 key patch"(`undefined` 删 key)。
- `flushPendingSessionWrites`:running 期间累积的 7 种 `PendingSessionWrite`(message / model_change / thinking_level_change / custom / custom_message / label / leaf)在 turn 边界、save_point、失败时统一落地。
- `emitRunFailure`:捕获 runAgentLoop 抛错后,生成"以 assistant error 终止的伪消息",并按 `message_start/end → turn_end → agent_end` 的事件流上报,保证订阅者看到完整的失败轨迹。
- `abort()`:清空 steer/followUp 队列、触发 `runAbortController.abort()`、等待 idle、发出 `abort` 事件。

### 2.8 架构图

```
                 ┌─────────────────────────────────────────────────────┐
                 │                   AgentHarness                       │
                 │                                                     │
   prompt() ───▶ │   phase  ───▶ createTurnState() ───▶ snapshot       │
   skill()       │                                                     │
   promptFromT() │   ┌─────────────────────────────────────────────┐   │
                 │   │  Hooks  (on)                                 │   │
   steer()       │   │   ├ before_agent_start  → 改 msgs/system     │   │
   followUp()    │   │   ├ context             → 改 LLM context     │   │
   nextTurn()    │   │   ├ tool_call           → block/allow         │   │
                 │   │   ├ tool_result         → 改写/终止           │   │
   abort()       │   │   ├ before_provider_*   → 改 streamOptions    │   │
                 │   │   └ session_before_*    → 自定义压缩/分支     │   │
                 │   └─────────────────────────────────────────────┘   │
   setModel()    │                                                     │
   setThink()    │   ┌─────────────────────────────────────────────┐   │
   setTools()    │   │  Subscribers (subscribe)                     │   │
   setRes()      │   │   queue_update / save_point / abort / settled│   │
   setStream()   │   │   model_select / thinking_level_select      │   │
                 │   │   session_compact / session_tree            │   │
   getRes()      │   │   after_provider_response                   │   │
   getStream()   │   │   (透传 AgentEvent)                         │   │
                 │   └─────────────────────────────────────────────┘   │
                 │                                                     │
                 │   pendingSessionWrites  (running 期暂存)            │
                 │     ├ message                                          │
                 │     ├ model_change                                    │
                 │     ├ thinking_level_change                           │
                 │     ├ custom / custom_message                         │
                 │     ├ label                                           │
                 │     ├ session_info                                    │
                 │     └ leaf                                            │
                 │                                                     │
                 │   Queues:                                            │
                 │     steerQueue      (one-at-a-time | all)            │
                 │     followUpQueue   (one-at-a-time | all)            │
                 │     nextTurnQueue   (always all, splice 0)           │
                 └─────────────────────────────────────────────────────┘
```

---

## 3. 类型与错误层(`types.ts`)

### 3.1 角色

`types.ts` 是 harness 的"宪法":

- 所有的**契约**(接口、错误码、消息枚举、事件 union)集中在这里。
- 实现了 `Result<T,E>` + `ok / err / getOrThrow / getOrUndefined / toError` 工具集。
- 定义了**错误码字典**与对应 `*Error` 类,所有子系统共用。

### 3.2 主要接口族

#### 3.2.1 执行环境

```ts
interface FileSystem { cwd; absolutePath; joinPath; readTextFile; ...; cleanup(); }
interface Shell { exec(command, options?); cleanup(); }
interface ExecutionEnv extends FileSystem, Shell {}
```

设计要点:

- **不抛异常**:所有方法返回 `Result<T, FileError|ExecutionError>`,失败必须编码在 `Result` 中。
- **`abortSignal` 贯穿所有 I/O**:取消语义统一。
- **路径寻址 ≠ 真实路径**:`absolutePath` 不解析 symlink,需要 `canonicalPath` 显式解析。

#### 3.2.2 会话存储与仓库

```ts
interface SessionStorage<TMetadata> {
  getMetadata / getLeafId / setLeafId
  createEntryId / appendEntry / getEntry / findEntries
  getLabel / getPathToRoot / getEntries
}
interface SessionRepo<TMetadata, TCreateOptions, TListOptions> {
  create / open / list / delete / fork
}
```

- `SessionStorage` 是"已有会话的增删查改"接口,后端实现有 `JsonlSessionStorage` 与 `InMemorySessionStorage`。
- `SessionRepo` 是"会话的目录管理"接口,负责创建、列举、fork、删除。后端有 `JsonlSessionRepo` 与 `InMemorySessionRepo`。

#### 3.2.3 资源与选项

```ts
interface Skill                  { name; description; content; filePath; disableModelInvocation? }
interface PromptTemplate         { name; description?; content }
interface AgentHarnessResources  { promptTemplates?; skills? }
interface AgentHarnessStreamOptions { transport?; timeoutMs?; maxRetries?; headers?; metadata?; cacheRetention? }
```

### 3.3 错误字典

| 错误类 | 码 | 来源 |
| --- | --- | --- |
| `FileError` | `aborted / not_found / permission_denied / not_directory / is_directory / invalid / not_supported / unknown` | FileSystem |
| `ExecutionError` | `aborted / timeout / shell_unavailable / spawn_error / callback_error / unknown` | Shell |
| `SessionError` | `not_found / invalid_session / invalid_entry / invalid_fork_target / storage / unknown` | Session 全部组件 |
| `CompactionError` | `aborted / summarization_failed / invalid_session / unknown` | compaction |
| `BranchSummaryError` | `aborted / summarization_failed / invalid_session` | branch-summarization |
| `AgentHarnessError` | `busy / invalid_state / invalid_argument / session / hook / auth / compaction / branch_summary / unknown` | AgentHarness 顶层 |

`AgentHarness` 在 `normalizeHarnessError` 中按 cause 链把子系统错误归一到顶层码(便于订阅者只处理少量顶层错误)。

### 3.4 事件字典(节选)

```
// 订阅者侧
AgentHarnessOwnEvent =
  | QueueUpdateEvent | SavePointEvent | AbortEvent | SettledEvent
  | BeforeAgentStartEvent | ContextEvent
  | BeforeProviderRequestEvent | BeforeProviderPayloadEvent | AfterProviderResponseEvent
  | ToolCallEvent | ToolResultEvent
  | SessionBeforeCompactEvent | SessionCompactEvent
  | SessionBeforeTreeEvent | SessionTreeEvent
  | ModelSelectEvent | ThinkingLevelSelectEvent
  | ResourcesUpdateEvent
AgentHarnessEvent = AgentEvent | AgentHarnessOwnEvent

// 钩子返回类型
AgentHarnessEventResultMap = {
  before_agent_start: BeforeAgentStartResult | undefined
  context: ContextResult | undefined
  before_provider_request: BeforeProviderRequestResult | undefined
  before_provider_payload: BeforeProviderPayloadResult | undefined
  after_provider_response: undefined
  tool_call: ToolCallResult | undefined
  tool_result: ToolResultPatch | undefined
  session_before_compact: SessionBeforeCompactResult | undefined
  session_compact: undefined
  session_before_tree: SessionBeforeTreeResult | undefined
  session_tree: undefined
  model_select: undefined
  thinking_level_select: undefined
  resources_update: undefined
  queue_update: undefined
  save_point: undefined
  abort: undefined
  settled: undefined
}
```

### 3.5 架构图

```
                          ┌──────────────────────┐
                          │       types.ts       │
                          │  (单一事实来源)       │
                          └──────────┬───────────┘
                                     │
        ┌───────────────────┬────────┼──────────┬────────────────┐
        │                   │        │          │                │
        ▼                   ▼        ▼          ▼                ▼
   Error Dictionary   FileSystem   Session   Skill/Template   Event / Hook
   (6 classes)       Shell         Storage   Resources         Contracts
        │             ExecutionEnv   Repo     StreamOptions
        │                   │          │             │
        │                   │          │             │
        │             env/      session/      agent-harness.ts
        │             nodejs.ts session.ts    (顶层)
        │             (Node实现)  (业务)    (编排)
        │
        └─→ 全部通过 normalizeHarnessError 在 AgentHarness 归一
```

---

## 4. 消息层(`messages.ts`)

### 4.1 角色

harness 用一种**扩展的 AgentMessage** 表达"会话中流转的消息",其中包含 pi-ai 原生 `user/assistant/toolResult` 以及 4 种**自定义角色**(`CustomAgentMessages` 模块声明):

```ts
interface CustomAgentMessages {
  bashExecution:    BashExecutionMessage      // 命令 + 输出 + exitCode + truncated
  custom:           CustomMessage             // 应用自定义,display flag
  branchSummary:    BranchSummaryMessage      // 导航回旧分支的摘要
  compactionSummary: CompactionSummaryMessage  // 压缩产生的摘要
}
```

`convertToLlm()` 是唯一把"Agent 消息 → provider LLM Message"的转换:

| Agent 角色 | 转换 |
| --- | --- |
| `user / assistant / toolResult` | 透传 |
| `bashExecution` | 渲染为 user 文本(可 `excludeFromContext` 跳过) |
| `custom` | 视为 user;字符串或 `TextContent[]` |
| `branchSummary` | 套上 `<summary>` 标签,role=user |
| `compactionSummary` | 同上,但前缀文案不同 |

### 4.2 关键常量

```ts
COMPACTION_SUMMARY_PREFIX  // "The conversation history before this point was compacted into the following summary: ..."
BRANCH_SUMMARY_PREFIX      // "The following is a summary of a branch that this conversation came back from: ..."
```

这两个 prefix 让模型看到摘要时知道"这是回顾而非用户当下发言",避免误以为要继续追问。

### 4.3 工厂函数

- `createBranchSummaryMessage / createCompactionSummaryMessage / createCustomMessage`:从 `SessionTreeEntry` 还原为 `AgentMessage`,**不在消息内容里携带业务时间戳以外的信息**。
- `bashExecutionToText`:把 bash 执行结果渲染为 markdown 代码块;在 `cancelled` / 非 0 退出 / 截断时附加说明。

### 4.4 架构图

```
SessionTreeEntry                       AgentMessage                     LLM Message
─────────────────                      ──────────────                   ───────────
message { role, ... }       ──1:1──▶   user | assistant | toolResult ───▶ 透传
custom_message              ──工厂──▶  custom                     ──▶  user(text)
branch_summary              ──工厂──▶  branchSummary              ──▶  user(<summary>)
compaction                  ──工厂──▶  compactionSummary          ──▶  user(<summary>)

bashExecution? (来源于工具执行,非 Session 树)
                            ──工厂──▶  bashExecution              ──▶  user(text) or skip
```

---

## 5. 会话子系统(`session/`)

### 5.1 角色与模型

会话是**有根、面向叶、有标签的 DAG/树**。节点类型(`SessionTreeEntry`):

```
MessageEntry | ThinkingLevelChangeEntry | ModelChangeEntry
CompactionEntry | BranchSummaryEntry
CustomEntry | CustomMessageEntry
LabelEntry | SessionInfoEntry | LeafEntry
```

每个 entry 都有 `id / parentId / timestamp`,**叶节点由 LeafEntry 显式指定**(`setLeafId` 持久化写入一个 `leaf` entry,而不是仅修改内存指针)。重启时从最后一个 leaf-affecting entry 还原叶 id。

### 5.2 `Session`(业务封装)

```
                  ┌──────────────────────────────────┐
                  │            Session                │
                  │  - storage: SessionStorage        │
                  │                                   │
                  │  getMetadata()  getLeafId()       │
                  │  getEntry()     getEntries()      │
                  │  getBranch()    buildContext()    │
                  │  getLabel()     getSessionName()  │
                  │                                   │
                  │  appendMessage(message)           │
                  │  appendModelChange(provider,id)   │
                  │  appendThinkingLevelChange(level) │
                  │  appendCompaction(summary,...)    │
                  │  appendCustomEntry(type, data)    │
                  │  appendCustomMessageEntry(...)    │
                  │  appendLabel(targetId, label)     │
                  │  appendSessionName(name)          │
                  │  moveTo(entryId, summary?)  ← fork/导航 │
                  └──────────────────────────────────┘
                                │
                                ▼
                  ┌──────────────────────────────────┐
                  │        SessionStorage             │
                  │   (接口,appendEntry/createEntryId)│
                  └────────────┬──────────────┬───────┘
                               │              │
                       JsonlSessionStorage  InMemorySessionStorage
                       (jsonl-storage.ts)   (memory-storage.ts)
```

`buildContext()` 是核心:把"从根到叶的 path"按 entry 类型线性化为 `SessionContext { messages, thinkingLevel, model }`,**遇到 compaction 时**会把 compaction entry 之前的"firstKeptEntryId 之后到 compaction 之前"的消息保留,并在开头插入一条 `compactionSummary` 消息。

`moveTo(entryId, summary?)` 是 fork/导航的统一入口:把 leaf 指向新 entry,可选地写入一条 `branch_summary`,作为"我从这个点切走了"的标记。

### 5.3 JSONL 后端(`jsonl-storage.ts`)

文件格式:

```
{"type":"session","version":3,"id":"<uuid>","timestamp":"...","cwd":"...","parentSession":"..."}
{...entry 1...}
{...entry 2...}
...
{...leaf entry...}    ← 末尾,记录 active leaf
```

- **Header 在第 1 行**,`version: 3`,有 `id/timestamp/cwd/parentSession`。
- **追加写**:每次 `appendEntry` 直接 `appendFile` 一行 JSON,内存里同步更新 `entries[] / byId / labelsById`,把 `currentLeafId` 后移。
- **不维护索引文件**:启动时全量读一次构建内存索引;label 缓存仅做"最后写入胜出"的合并。
- **ID 冲突避免**:`generateEntryId` 优先用 `uuidv7().slice(0, 8)`(短 id 便于人读),冲突时回退到完整 uuid。
- **错误定位**:`invalidEntry(filePath, lineNumber, ...)` 把错误精确到行号。

### 5.4 JSONL 仓库(`jsonl-repo.ts`)

```
   sessionsRoot/  (传入,如 ~/.pi/sessions)
   ├── --home-user-projects-foo--/      ← encodeCwd() 把 cwd 编码成目录名
   │   ├── 2024-05-01T12-00-00-000Z_<id>.jsonl
   │   └── 2024-05-02T09-15-11-022Z_<id>.jsonl
   └── --home-user-projects-bar--/
       └── ...
```

- `encodeCwd`:把 `/` `\` `:` 都替换为 `-`,前后加 `--`,得到一个稳定的目录名(避免路径分隔符冲突)。
- 文件名:`<ISO timestamp>_<sessionId>.jsonl`,`list()` 按 `createdAt` 倒序。
- `fork()`:把"源会话中目标 entry 之前的 path"复制到一个新会话(支持 `position: "before" | "at"`,详见 `repo-utils.ts` 的 `getEntriesToFork`)。

### 5.5 内存版后端

`InMemorySessionStorage` / `InMemorySessionRepo` 行为与 JSONL 版**完全一致**,但全在内存。用途:

- 单元测试(无需文件系统)
- 嵌入式场景(浏览器/无 fs 的 runtime)
- 与 `JsonlSessionStorage` 做交叉一致性测试

### 5.6 共享工具

`session/repo-utils.ts`:

```ts
createSessionId()   → uuidv7()
createTimestamp()   → new Date().toISOString()
toSession(storage)  → new Session(storage)
getFileSystemResultOrThrow(result, message) → 把 FileError 升级为 SessionError
getEntriesToFork(storage, { entryId, position }) → 取要复制的 entries
```

`getEntriesToFork` 的逻辑:
- 没有 `entryId` → 复制全部。
- `position === "at"` → 取 `target.id` 到根的 path(包含 target 本身)。
- `position === "before"`(默认)→ target 必须是 user message,取 target.parentId 到根的 path(不包含 target 自身)。

### 5.7 UUID v7(`uuid.ts`)

自实现的 `uuidv7()`:48-bit 毫秒时间戳 + 12-bit 序列(同毫秒内自增)+ 62-bit 随机;支持 `crypto.getRandomValues` 和 `Math.random` 回退。**id 单调递增**,便于在 JSONL 中按时间浏览。

### 5.8 架构图

```
                ┌──────────────────────────────────────────────────────┐
                │                    Session (业务)                      │
                │  appendMessage / appendCompaction / moveTo / ...       │
                └───────────────────────────┬──────────────────────────┘
                                            │ 依赖
                                            ▼
                ┌──────────────────────────────────────────────────────┐
                │             SessionStorage (接口)                      │
                │   appendEntry / getEntry / getPathToRoot / setLeafId  │
                └──────────┬──────────────────────────────┬────────────┘
                           │                              │
                           ▼                              ▼
                ┌────────────────────────┐  ┌────────────────────────────┐
                │   JsonlSessionStorage  │  │   InMemorySessionStorage   │
                │   ├ file: *.jsonl      │  │   ├ entries[] / byId       │
                │   ├ header (v3)        │  │   ├ labelsById cache       │
                │   ├ appendFile 一行    │  │   └ leafId (内存变量)       │
                │   └ 全量 load 建索引   │  │                            │
                └──────────┬─────────────┘  └─────────────┬──────────────┘
                           │                              │
                           └──────────┬───────────────────┘
                                      ▼
                ┌──────────────────────────────────────────────────────┐
                │                    SessionRepo (接口)                  │
                │  create / open / list / delete / fork                │
                └────────────┬──────────────────────────┬──────────────┘
                             │                          │
                             ▼                          ▼
                ┌────────────────────────┐  ┌────────────────────────────┐
                │    JsonlSessionRepo    │  │   InMemorySessionRepo      │
                │  sessionsRoot/         │  │   sessions: Map<id,Session>│
                │  encodeCwd 子目录       │  │                            │
                │  <timestamp>_<id>.jsonl│  │                            │
                └────────────────────────┘  └────────────────────────────┘
```

---

## 6. 压缩子系统(`compaction/`)

### 6.1 角色

上下文压缩是 harness 的"续命机制":随着对话增长,把超出窗口的早期消息**让 LLM 总结成结构化摘要**,持久化为 `compaction` entry,下次构建上下文时只保留"摘要 + 压缩点之后的消息"。

压缩分两类:

| 操作 | 入口 | 目标 |
| --- | --- | --- |
| **Compaction** | `harness.compact(customInstructions?)` | 当前 leaf 之前的全部 → 摘要 + firstKeptEntryId |
| **Branch summary** | `harness.navigateTree(targetId, { summarize: true })` | old leaf → target 的非共同祖先 → 摘要 |

### 6.2 `compaction.ts` 主流程

```
                            prepareCompaction(pathEntries, settings)
                                       │
                       ┌───────────────┼───────────────┐
                       ▼               ▼               ▼
                prevCompaction?    estimateTokens    findCutPoint
                (续写 vs 新建)     (tokensBefore)    (firstKeptEntryId)
                       │                               │
                       └──────────┬────────────────────┘
                                  ▼
                          CompactionPreparation
                          { firstKeptEntryId, messagesToSummarize,
                            turnPrefixMessages?, isSplitTurn,
                            tokensBefore, previousSummary?, fileOps,
                            settings }
                                  │
                                  ▼
                                 compact()
                                  │
        ┌─────────────────────────┴─────────────────────────┐
        ▼                                                   ▼
  isSplitTurn && turnPrefix                               否则:
  generateTurnPrefixSummary()                    generateSummary(messagesToSummarize,
  generateSummary(messagesToSummarize,                     model, reserveTokens, ...,
                  model, reserveTokens, ...                customInstructions,
                  customInstructions,                      previousSummary,
                  previousSummary, ...                     thinkingLevel)
                  thinkingLevel)                                 │
        │                                                   │
        └────────────────┬──────────────────────────────────┘
                         ▼
                  summary = history + "\n\n---\n\n" + turnPrefix
                  + formatFileOperations(readFiles, modifiedFiles)
                         │
                         ▼
                  CompactionResult { summary, firstKeptEntryId,
                                     tokensBefore, details }
                         │
                         ▼
              session.appendCompaction(...)
```

#### 6.2.1 切点选择(`findCutPoint`)

- 在 `[boundaryStart, boundaryEnd)` 内枚举**合法切点**:user/bashExecution/custom/branchSummary/compactionSummary 等可独立成轮的 entry。
- 从尾部向前累计 `estimateTokens`,达到 `keepRecentTokens` 阈值时,就近对齐到合法切点。
- **跨 turn 切**(`isSplitTurn`):如果切点不是 user 消息,会回溯到该 turn 的起点(`user` 或 `bashExecution`),并把"切点到 turn 起点"之间作为 `turnPrefixMessages`,**单独再做一次摘要**("turn 前缀摘要")。

#### 6.2.2 上下文 token 估算

- 优先用**最近一次 assistant usage**(provider 真实报告的 `totalTokens/input+output+cacheRead+cacheWrite`)+ 之后手算的增量。
- 没有 usage 时退化为字符数/4 的启发式估算(`estimateTokens`)。

#### 6.2.3 文件操作追踪

```
extractFileOpsFromMessage(msg, fileOps)
  - 扫 assistant content 里的 toolCall block
  - name==="read"   → fileOps.read.add(path)
  - name==="write"  → fileOps.written.add(path)
  - name==="edit"   → fileOps.edited.add(path)
computeFileLists(fileOps) → { readFiles, modifiedFiles }
formatFileOperations(...) → "<read-files>...</read-files><modified-files>...</modified-files>"
```

- **复用历史**:如果之前已有 compaction(且非 fromHook),把它记录过的 read/modified 文件带入本轮 fileOps。
- 这样在多轮压缩迭代中,被读/改过的文件**始终**出现在最终摘要里,模型能持续看到"这次会话动过哪些文件"。

#### 6.2.4 续写 vs 新建

`previousSummary` 决定走哪条 prompt:

- 新建 → `SUMMARIZATION_PROMPT`(完整结构)
- 续写 → `UPDATE_SUMMARIZATION_PROMPT`("保留旧条目 + 加入新条目")

`SUMMARIZATION_SYSTEM_PROMPT` 统一为:"你是上下文摘要助手,只输出结构化摘要,不要继续对话"。

### 6.3 `branch-summarization.ts` 分支摘要

当用户从某 leaf 跳到历史中更早的一个 entry(`navigateTree`),需要把"被绕过的分支"总结成 `branch_summary` 注入上下文,避免失去对旧分支工作内容的记忆。

```
collectEntriesForBranchSummary(session, oldLeafId, targetId)
  ├ 旧 path 集合 = getPathToRoot(oldLeafId) 转为 Set<id>
  ├ 新 path     = getPathToRoot(targetId)
  └ commonAncestorId = 第一次在新 path 中出现于旧 path 集合的 id
  └ entries = 从 oldLeafId 沿 parentId 上溯到 commonAncestorId(不含)
             → 反转 → 时间正序

prepareBranchEntries(entries, tokenBudget)
  └ 按 token 预算从尾部挑选消息(最后追加的优先)
  └ 累加 fileOps

generateBranchSummary(entries, options)
  └ LLM 提示:"用户在分支里尝试了X,完成了Y,还剩Z"
  └ 输出 BranchSummaryResult { summary, readFiles, modifiedFiles }
```

返回结果通过 `session.moveTo(targetId, { summary, details, fromHook })` 持久化为 `branch_summary` entry。

### 6.4 `compaction/utils.ts` 共享工具

- `serializeConversation(messages)`:把 LLM 消息数组转为可读文本,作为摘要 prompt 的输入。
  - user: `[User]: <text>`
  - assistant: 分别累加 thinking / text / toolCalls,产生 `[Assistant thinking]` / `[Assistant]` / `[Assistant tool calls]` 行
  - toolResult: `[Tool result]: <text>`(> 2000 字符会被截断并标注)
- `truncateForSummary`:辅助函数,在序列化时对超长工具结果做尾部截断。

### 6.5 架构图

```
┌────────────────────────────────────────────────────────────────────────┐
│                            compaction                                  │
│                                                                        │
│  prepareCompaction ───▶ findCutPoint ───▶ estimateContextTokens         │
│        │                   │                  │                         │
│        │                   │                  └─ fileOps(累积)            │
│        │                   │                                            │
│        ▼                   ▼                                            │
│  CompactionPreparation { firstKeptEntryId, messagesToSummarize,        │
│                          turnPrefixMessages?, isSplitTurn,             │
│                          tokensBefore, previousSummary?,               │
│                          fileOps, settings }                            │
│        │                                                                │
│        ▼                                                                │
│  compact()                                                              │
│    ├ isSplitTurn → Promise.all([                                        │
│    │     generateSummary(messagesToSummarize, ...),                     │
│    │     generateTurnPrefixSummary(turnPrefixMessages, ...)             │
│    │  ])                                                                │
│    └ else      → generateSummary(messagesToSummarize, ...)              │
│        │                                                                │
│        ▼                                                                │
│  summary = history + turnPrefix + formatFileOperations(read,modified)  │
│        │                                                                │
│        ▼                                                                │
│  CompactionResult { summary, firstKeptEntryId,                          │
│                     tokensBefore, details: {readFiles, modifiedFiles}}  │
│                                                                        │
│  utils.ts:                                                              │
│    extractFileOpsFromMessage (read/write/edit)                          │
│    computeFileLists / formatFileOperations                              │
│    serializeConversation / truncateForSummary                           │
│                                                                        │
│  branch-summarization.ts:                                               │
│    collectEntriesForBranchSummary  ←→  diff two paths                   │
│    prepareBranchEntries             ←→  token budget aware              │
│    generateBranchSummary            ←→  LLM 摘要                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 执行环境(`env/nodejs.ts`)

### 7.1 角色

为 `FileSystem` + `Shell` 接口提供 **Node.js 实现**,把文件系统 I/O、进程 spawn、错误码、取消信号全部归一化为 `Result<T, FileError|ExecutionError>`。

### 7.2 文件系统映射

| 接口方法 | Node 实现 | 错误码映射 |
| --- | --- | --- |
| `absolutePath` | `isAbsolute ? path : resolve(cwd, path)` | 几乎不报错 |
| `readTextFile` | `readFile(path, 'utf8')` | ENOENT→not_found, EACCES→permission_denied, ABORT_ERR→aborted |
| `readBinaryFile` | `readFile(path)` | 同上 |
| `readTextLines` | `createReadStream` + `readline`,`maxLines` 提前 break | 同上 |
| `writeFile` | `mkdir(parent, recursive)` + `writeFile` | 同上 |
| `appendFile` | `mkdir(parent, recursive)` + `appendFile` | 同上 |
| `fileInfo` | `lstat`(不跟随 symlink) | 失败 → not_found 或其他 |
| `listDir` | `readdir(withFileTypes)` | 失败映射 |
| `canonicalPath` | `realpath` | 失败映射 |
| `exists` | `fileInfo` 后看 `code === "not_found"` | 不存在 → `ok(false)` |
| `createDir` | `mkdir({ recursive })` | 失败映射 |
| `remove` | `rm({ recursive, force })` | 失败映射 |
| `createTempDir` | `mkdtemp(join(tmpdir(), prefix))` | 失败映射 |
| `createTempFile` | `createTempDir` + `writeFile(<uuid>)` | 失败映射 |

`isNodeError` 通过 `code` 字段把 Node 的 `ErrnoException` 翻译成稳定错误码,**应用层永远拿不到 ENOENT 这种字符串**。

### 7.3 Shell

```
exec(command, options) {
  shell = getShellConfig(shellPath) {
    win32 → ProgramFiles/Git/bin/bash.exe → where bash.exe → sh
    其他 → /bin/bash → which bash → sh
  }
  child = spawn(shell, [args..., command], { cwd, env, detached: !win32 })
  on('data')  → stdout/stderr 累加 + onStdout/onStderr 回调
  on('abort') → killProcessTree(child.pid)
  on('timeout') → killProcessTree
  on('error' / 'close') → settle(Result)
}
```

要点:
- **detached = 非 win32**:使 `process.kill(-pid, SIGKILL)` 能杀整个进程组(包含管道子进程)。
- **win32** 用 `taskkill /F /T /PID <pid>`。
- **callback_error**:用户提供的 `onStdout/onStderr` 抛错会被翻译成 `ExecutionError("callback_error")` 并立即 `killProcessTree`。
- **超时单位是秒**(应用层接口),内部 `* 1000` 转为 ms。

### 7.4 架构图

```
                ┌──────────────────────────────────────────────────────┐
                │                  ExecutionEnv (接口)                    │
                │  FileSystem { read/write/list/exists/temp/... }       │
                │  Shell      { exec(command, options) }                │
                └──────────────────────────┬───────────────────────────┘
                                           │ implements
                                           ▼
                ┌──────────────────────────────────────────────────────┐
                │                NodeExecutionEnv                        │
                │  cwd, shellPath?, shellEnv?                            │
                │                                                          │
                │  fs 部分:                                               │
                │    access / readFile / writeFile / appendFile /        │
                │    mkdir / readdir / lstat / realpath / mkdtemp        │
                │  → toFileError(errno) → FileError 稳定码                │
                │                                                          │
                │  shell 部分:                                            │
                │    getShellConfig() → win32 vs posix                    │
                │    spawn() + data/abort/timeout/close 事件              │
                │    killProcessTree(pid) → process.kill / taskkill       │
                │    统一通过 Result<{ stdout, stderr, exitCode },         │
                │                     ExecutionError>                     │
                └──────────────────────────────────────────────────────┘
```

---

## 8. 工具集(`utils/`)

### 8.1 `truncate.ts` — 文本截断

为长输出提供 head / tail 两种截断,**双上限**:`maxLines` 与 `maxBytes`,谁先到截谁。

| 函数 | 用途 | 关键特性 |
| --- | --- | --- |
| `truncateHead` | 文件读取等"看开头"场景 | 永不输出残行;首行 > maxBytes 时返回空 + `firstLineExceedsLimit: true` |
| `truncateTail` | Bash 输出等"看结尾"场景 | 仅"还没装满任何行 + 第一行就 > maxBytes"时才返回残行(并标 `lastLinePartial`) |
| `truncateLine` | grep 命中行过长 | 截到 N 字符 + `... [truncated]` |
| `formatSize` | B / KB / MB 自适应 | - |

`utf8ByteLength`:在 Node 上用 `Buffer.byteLength`,浏览器回退时手算 UTF-8(规避大字符串 `TextEncoder` 的开销)。**正确处理代理对**(high+low),`replaceUnpairedSurrogates` 把孤立代理替换为 U+FFFD。

### 8.2 `shell-output.ts` — Bash 输出捕获与溢出落盘

为"长时间/可能大体积"的 shell 命令提供安全捕获:

```
executeShellWithCapture(env, command, options):
  onChunk(chunk):
    totalBytes += chunk.bytes
    text = sanitizeBinaryOutput(chunk).replace(/\r/g, "")
    if totalBytes > DEFAULT_MAX_BYTES && !fullOutputPath:
      ensureFullOutputFile(<so-far output>)        # 创建 temp file
    else:
      appendFullOutput(text)                       # 串行追加到 temp file
    outputChunks.push(text) (环形累计 <= 2x DEFAULT_MAX_BYTES)
    options.onChunk?.(text)

  完成时:
    tail = outputChunks.join("")
    truncation = truncateTail(tail)
    if truncated && !fullOutputPath:
      ensureFullOutputFile(tail)                   # 补一份完整落盘
    await writeChain                                # 等所有 append 完成
    return ok({ output, exitCode, cancelled, truncated, fullOutputPath })
```

设计要点:
- **内存上限**:`outputChunks` 用 `maxOutputBytes = DEFAULT_MAX_BYTES * 2` 当环形缓冲,丢弃最早 chunk。
- **磁盘兜底**:超过 `DEFAULT_MAX_BYTES` 就开 temp file 把全部内容写下去(`<tmpdir>/bash-<uuid>.log`),最终告诉调用者"完整内容在 fullOutputPath,这里只给你看 tail"。
- **二进制安全**:`sanitizeBinaryOutput` 过滤控制字符(保留 `\t \n \r`)和 Unicode 干扰字符(`U+FFF9..U+FFFB`)。
- **回调容错**:`onChunk` 抛错会被记为 `captureError` 并通过 `ExecutionError("callback_error")` 返回,但**不会**中断已经进行的写文件链。
- **中止语义**:abort → `cancelled: true, exitCode: undefined`;非 0 exit → 原样返回。

### 8.3 架构图

```
truncate.ts                                    shell-output.ts
──────────                                     ───────────────
truncateHead(content)                          executeShellWithCapture(env, command)
   ├ 计算总行/总字节                                 ├ onChunk: 累计 / sanitizeBinaryOutput
   ├ 首行 > maxBytes ? 空+标记                        ├ 超阈值 ?  落盘 (createTempFile + appendFile)
   └ 按行/字节装入, 永不残行                          └ 环形缓冲 outputChunks(2x maxBytes)
                                                     └ truncateTail(tail) + 最终落盘补全
truncateTail(content)                              → { output, exitCode, cancelled,
   ├ 从尾部按行/字节装入                                  truncated, fullOutputPath }
   └ 唯一允许残行的场景:首行 > maxBytes
truncateLine(line, maxChars)                    sanitizeBinaryOutput(text)
   └ 截到 maxChars 字符 + "... [truncated]"         └ 过滤控制字符 + U+FFF9..FFFB
```

---

## 9. 资源加载(`skills.ts` / `prompt-templates.ts` / `system-prompt.ts`)

### 9.1 `skills.ts` — 技能加载

技能遵循 [agentskills.io](https://agentskills.io) 风格:`SKILL.md` 文件 + YAML frontmatter(`name / description / disable-model-invocation?`)。

```
loadSkills(env, dirs | [dirs])
  └ for each dir:
       ├ fileInfo → not_found 跳过 / 不是 directory 跳过
       └ loadSkillsFromDirInternal(env, dir, includeRootFiles=true, ignoreMatcher, rootDir)
            ├ 读取 ignore 文件(.gitignore / .ignore / .fdignore),prefix 后加入 ig
            ├ 优先找 SKILL.md
            │   └ loadSkillFromFile → frontmatter(name/description/disable-model-invocation) + body
            └ 其它 .md 文件(in 根) / 子目录(递归)
                 └ 子目录中递归调用(includeRootFiles=false)
```

校验规则:
- `name` 必须等于父目录名、≤ 64 字符、只含 `a-z0-9-`、不以 `-` 开头/结尾、不含连续 `--`。
- `description` 必填且 ≤ 1024 字符。
- 校验失败 → 收集 `SkillDiagnostic`,不抛错,**不**返回该技能。

`loadSourcedSkills`:把每条输入绑上一个"source"(应用层定义来源含义,例如 user / project / builtin),返回 `Array<{ skill, source }>`,应用层可以按 source 决定可见性。

`formatSkillInvocation(skill, additionalInstructions?)`:把技能体渲染为 `<skill name="..." location="..."> ... </skill>`,作为 `harness.skill(name)` 的 prompt 前缀。

### 9.2 `prompt-templates.ts` — 提示模板加载

模板 = 一个 `.md` 文件(可带 frontmatter `description / argument-hint`),**不递归子目录**,文件名(去掉 `.md`)作为 `name`。首行非空内容会作为默认 `description`(超过 60 字符加 `...`)。

参数替换(`substituteArgs`):

| 占位符 | 含义 |
| --- | --- |
| `$1` `$2` ... | 第 N 个参数 |
| `$@` / `$ARGUMENTS` | 全部参数,空格连接 |
| `${@:N}` | 第 N 个开始的全部参数 |
| `${@:N:L}` | 第 N 个开始的 L 个参数 |

`parseCommandArgs`:支持 `"..."` 和 `'...'` 引号、单空格/Tab 分隔,**不做转义解析**。这与 Claude Code 的 `/command arg1 "arg 2"` 行为一致。

`formatPromptTemplateInvocation(template, args)` 组合以上两步,作为 `harness.promptFromTemplate(name, args)` 的 prompt。

### 9.3 `system-prompt.ts` — 把技能拼进系统提示

```
formatSkillsForSystemPrompt(skills):
  1. 过滤 disableModelInvocation
  2. 拼出 <available_skills> XML 块(name / description / location)
  3. 头部说明:技能提供专门指令;相对路径以技能目录为锚点
```

应用层在构造 `systemPrompt` 时(可以是字符串或回调)可选择性地把这段 XML 追加到系统提示末尾,让模型在合适的时机读取并使用技能文件。

### 9.4 架构图

```
                     ┌────────────────────────────────────────┐
                     │           Application Code            │
                     └────────────────┬───────────────────────┘
                                      │ loadSkills(env, [...])
                                      │ loadPromptTemplates(env, [...])
                                      ▼
        ┌─────────────────────────────────────────────────────┐
        │                 Resources Layer                       │
        │                                                      │
        │  skills.ts                prompt-templates.ts        │
        │  ─────────                ──────────────────         │
        │  递归扫描 + 忽略文件       非递归目录                   │
        │  SKILL.md 优先             .md 顶层文件                │
        │  YAML frontmatter         YAML frontmatter           │
        │  name==parentDir 校验     name = basename(file)      │
        │  ↓                        ↓                           │
        │  Skill[]                  PromptTemplate[]            │
        │  + diagnostics            + diagnostics               │
        └────────────────────┬─────────────────────────────────┘
                             │
                             ▼
        ┌─────────────────────────────────────────────────────┐
        │                  AgentHarnessResources                │
        │   { skills, promptTemplates }   ← 由 setResources() │
        └────────────────────┬─────────────────────────────────┘
                             │
                ┌────────────┴────────────┐
                ▼                         ▼
    formatSkillsForSystemPrompt    formatSkillInvocation
    formatPromptTemplateInvocation
                │                         │
                └────────────┬────────────┘
                             ▼
                  systemPrompt / prompt()
```

---

## 10. 端到端示例(综合串联)

下面用一段伪代码把 harness 各部分串起来,方便建立完整心智模型:

```ts
// 1. 准备执行环境
const env = new NodeExecutionEnv({ cwd: process.cwd() });

// 2. 加载资源
const { skills } = await loadSkills(env, ["./.pi/skills", "~/.pi/skills"]);
const { promptTemplates } = await loadPromptTemplates(env, "./.pi/prompts");

// 3. 打开/创建会话
const repo = new JsonlSessionRepo({ fs: env, sessionsRoot: "~/.pi/sessions" });
const sessions = await repo.list();
const session = await repo.open(sessions[0]!);

// 4. 构造 harness
const harness = new AgentHarness({
  env,
  session,
  model,
  tools: [readTool, writeTool, editTool, bashTool],
  resources: { skills, promptTemplates },
  systemPrompt: async ({ activeTools, resources }) => [
    "You are an expert coding assistant.",
    formatSkillsForSystemPrompt(resources.skills ?? []),
  ].join("\n\n"),
  getApiKeyAndHeaders: async (model) => ({ apiKey: "...", headers: {} }),
  streamOptions: { maxRetries: 3, cacheRetention: "short" },
  steeringMode: "all",
  followUpMode: "all",
});

// 5. 订阅事件
harness.subscribe((event) => console.log("[harness]", event.type));

// 6. 注入提示
const reply = await harness.prompt("看看 src/agent.ts 有没有内存泄漏");

// 7. 后续操作
await harness.steer("顺便给我加个测试");
await harness.compact();           // 当上下文快满时
await harness.navigateTree(entryId, { summarize: true });  // 跳到旧 entry
```

---

## 11. 设计哲学总结

1. **接口优先**:SessionStorage / SessionRepo / ExecutionEnv 都用接口抽象,JSONL 与内存、Node 与其他 runtime 可替换。
2. **Result 表达"预期失败",异常表达"未预期"**:低层(capability)用 Result,高层(orchestration)用抛错。
3. **快照隔离**:Turn 状态在开始时拍下快照,运行中改 config/queue/pending write 不污染当前 turn。
4. **持久化优先**:`setLeafId` 写 `leaf` entry 而不是只动内存指针;运行中的写暂存到 `pendingSessionWrites`,在 turn 边界/失败/save_point 落盘。
5. **钩子串行+订阅广播**:`on(type)` 单返回值(后注册覆盖前注册),`subscribe` 广播(无返回)。
6. **可压缩可回溯**:压缩用 `firstKeptEntryId` 精准定位保留起点;分支摘要把"被绕过的分支"沉淀为可回看的记忆。
7. **安全降级**:`sanitizeBinaryOutput` 过滤控制字符、`truncateTail` 避免内存爆炸、`shell-output` 在超阈值时落盘 temp file。

---

## 12. 推荐阅读路径

1. `agent-harness.ts` — 顶层 API + 状态机
2. `types.ts` — 接口/错误/事件字典
3. `session/session.ts` + `session/jsonl-storage.ts` — 树形会话模型
4. `messages.ts` — Agent 消息 ↔ LLM 消息
5. `compaction/compaction.ts` — 压缩算法
6. `env/nodejs.ts` — 真实环境适配
7. `skills.ts` / `prompt-templates.ts` — 资源加载
8. `utils/shell-output.ts` / `utils/truncate.ts` — 工程细节
