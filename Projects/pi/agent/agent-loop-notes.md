# `agent-loop.ts` 脑图笔记

> 目标：**抓住作者的设计意图**——这个文件不是"一个 LLM 调用的循环"，而是把"对话/工具/事件/可扩展性"四种关注点分离后重新缝合的容器。

---

## 0. 一句话心智模型

```
agent-loop.ts = 一个"事件源" + 一个"可注入的循环" + 一道"类型边界"
```

- **事件源** — 所有状态变化都通过 `emit(event)` 流出去，UI / 日志 / 测试订阅同一份事件。
- **可注入的循环** — 业务策略（上下文压缩、工具权限、是否停、转 model）全部走 `AgentLoopConfig` 里的 hook 函数，循环本身没有策略。
- **类型边界** — 内部全程用 `AgentMessage[]`（可扩展），只在调 LLM 的那一行转成 `Message[]`（provider 认识），转完即丢，不污染主循环。

这个三角支撑了后面所有的设计选择。

---

## 1. 文件整体包含关系（Hierarchy）

```mermaid
graph TD
    A["agent-loop.ts<br/>= 引擎核心"]
    A --> B1["公共 API 层<br/>agentLoop / agentLoopContinue"]
    A --> B2["会话运行层<br/>runAgentLoop / runAgentLoopContinue"]
    A --> B3["主循环<br/>runLoop"]
    A --> B4["LLM 流式层<br/>streamAssistantResponse"]
    A --> B5["工具执行层<br/>executeToolCalls → sequential/parallel"]
    A --> B6["工具生命周期<br/>prepare → execute → finalize"]
    A --> B7["事件构造器<br/>createAgentStream / emitTool*"]

    B1 -.调用.-> B2
    B2 -.调用.-> B3
    B3 -.每 turn 调用.-> B4
    B3 -.有 toolCall 时调用.-> B5
    B5 -.单工具调用.-> B6
    B6 -. 旁路.-> B7
```

**为什么这样分层？**
- API 层只管"建流 + 包事件"，不碰循环细节
- 运行层只管"塞 prompt + 启动循环 + 收尾"，不碰 turn 细节
- 主循环 `runLoop` 是唯一能**多轮迭代**的地方，所以它最复杂
- LLM 流式层和工具执行层都是**单 turn 内**的事，被主循环反复调用
- 工具生命周期是**单工具**的事，被 sequential/parallel 共用

> 设计原则：**越外层越策略，越内层越机械**。外层问"还要继续吗"，内层只问"这一步怎么走"。

---

## 2. 外部函数关系（Public Surface）

```mermaid
graph LR
    Caller["调用方<br/>CLI / TUI / Tests"]
    Caller -->|新对话| AL["agentLoop"]
    Caller -->|续跑 / 重试| AC["agentLoopContinue"]

    AL --> R1["runAgentLoop"]
    AC --> R2["runAgentLoopContinue"]

    R1 --> RL["runLoop<br/>(核心循环)"]
    R2 --> RL

    AL -.返回.-> S1["EventStream&lt;AgentEvent, AgentMessage[]&gt;"]
    AC -.返回.-> S1
```

**为什么有两条入口而不是一条？**
- `agentLoop(prompts, ctx, ...)`：**追加新消息**到 context（"用户又发了条消息"）
- `agentLoopContinue(ctx, ...)`：**复用现有 context**（"重试上次失败 / 上一轮工具结果已就位"）
- `continue` 故意不做"清空 / 拼接"，把责任完全交给调用方——这样上层能自己决定"重试到底什么"
- 但两者**都校验最后一条消息的 role ≠ assistant**（line 74 / line 131）—— provider 协议上 assistant 后面跟 assistant 不合法

---

## 3. 内部数据流（最关键的一张图）

```mermaid
flowchart TB
    subgraph 内部世界["内部：AgentMessage[]（应用自有，可扩展）"]
        M1[用户消息]
        M2[自定义消息<br/>artifact / notification]
        M3[工具结果]
        M4[AssistantMessage]
    end

    subgraph 边界["类型边界（streamAssistantResponse 内）"]
        TC["transformContext?<br/>上下文压缩/裁剪"]
        CV["convertToLlm<br/>必填：类型转换"]
    end

    subgraph LLM世界["外部：Message[]（provider 协议）"]
        L1[UserMessage]
        L2[AssistantMessage]
        L3[ToolResultMessage]
    end

    subgraph 工具世界["工具世界（executeToolCalls 内）"]
        T1[prepareToolCall<br/>查找/校验/拦截]
        T2[executePreparedToolCall<br/>真跑工具]
        T3[finalizeExecutedToolCall<br/>after 钩子]
    end

    M1 --> TC
    M2 --> TC
    M3 --> TC
    M4 --> TC
    TC --> CV
    CV --> L1
    CV --> L2
    CV --> L3
    L1 --> LLM[("streamSimple<br/>provider")]
    L2 --> LLM
    L3 --> LLM
    LLM -.AssistantMessageEvent 流.-> M4

    M4 -.toolCall.-> T1
    T1 --> T2
    T2 --> T3
    T3 -.ToolResultMessage.-> M3
```

**为什么要把这道边界放在 LLM 调用前？**
- 让所有 hook（`beforeToolCall` / `afterToolCall` / `transformContext`）都操作**内部类型**——它们不需要懂 provider 的怪癖
- 内部类型可以是 `CustomAgentMessages`（UI 通知、artifact 引用等），这些**永远不会发给 LLM**
- 转换责任**收敛在一个地方**（`convertToLlm`），换 provider 时只改这一处
- "自定义消息不能转 LLM" 的过滤也在转换函数里，**不污染主循环的判断逻辑**

---

## 4. 主循环 `runLoop` 控制流

```mermaid
flowchart TB
    Start([runLoop 开始]) --> Init[初始化<br/>firstTurn = true<br/>pendingMessages = 拉一次]
    Init --> Outer{"outer while (true)"}

    Outer --> Inner["inner while<br/>(有 toolCall 或 pending 消息)"]
    Inner --> TurnCheck{firstTurn?}
    TurnCheck -->|是| SetFirst[firstTurn = false]
    TurnCheck -->|否| EmitTurn[emit turn_start]
    SetFirst --> InjectPending
    EmitTurn --> InjectPending{pending 长度?}
    InjectPending -->|>0| PushPending[把 pending 塞进 context<br/>发 message_start/end]
    InjectPending -->|0| StreamLLM
    PushPending --> StreamLLM[streamAssistantResponse]

    StreamLLM --> CheckStop{stopReason?<br/>error / aborted}
    CheckStop -->|是| EmitEnd[emit turn_end / agent_end]
    EmitEnd --> Return([返回])
    CheckStop -->|否| FindTools{有 toolCall?}

    FindTools -->|否| NoTools[hasMoreToolCalls = false]
    FindTools -->|是| ExecTools[executeToolCalls]

    ExecTools --> PushResult[结果 push 到 context]
    NoTools --> EmitTurnEnd
    PushResult --> EmitTurnEnd[emit turn_end]

    EmitTurnEnd --> PrepNext{config.prepareNextTurn?}
    PrepNext -->|有返回值| ApplyNext[替换 context / model / thinking]
    PrepNext -->|无| ShouldStop
    ApplyNext --> ShouldStop{config.shouldStopAfterTurn?}

    ShouldStop -->|true| EmitAgentEnd[emit agent_end]
    EmitAgentEnd --> Return
    ShouldStop -->|false| Refill[拉 steering 消息]

    Refill --> InnerCheck{hasMoreToolCalls<br/>或 pending?}
    InnerCheck -->|是| Inner
    InnerCheck -->|否| Outer

    Outer --> CheckFollow{getFollowUpMessages?}
    CheckFollow -->|有| SetPending[pending = followUps<br/>continue]
    CheckFollow -->|无| Break([break → 退出])
    SetPending --> Inner
```

**外层 vs 内层的语义差别（重要）：**

| 维度 | inner loop（一个 turn 内） | outer loop（多 turn 间） |
|---|---|---|
| 触发再循环的原因 | 还有 toolCall 没跑完 / 有 steering 消息 | 没人想停，但有 follow-up 排队 |
| 消费的消息类型 | steering（**打断式**：插在 assistant 之后） | follow-up（**追加式**：等当前会话自然停） |
| 模型/工具可能换 | 否（一个 turn 内固定） | 是（每个 turn 独立 `prepareNextTurn`） |
| 退出条件 | 跑完所有 toolCall 且没人想插话 | 没有 follow-up |

**为什么 steering 和 follow-up 拆开？**
- **steering** = 用户在 agent 工作时打字，要"插队"，**不能跳过当前工具**
- **follow-up** = 排在队尾的消息，要"等 agent 自己讲完话"
- 一个是 **synchronous interruption**，一个是 **asynchronous queue**——混淆会破坏语义
- 看 `runLoop` line 167、253（steering 拉取点）和 line 257（follow-up 拉取点）位置就懂

---

## 5. LLM 流式层 `streamAssistantResponse` 状态机

```mermaid
stateDiagram-v2
    [*] --> Transform: context.messages
    Transform --> LLMCall: convertToLlm → llmMessages
    LLMCall --> Iterating: for await event of response

    state Iterating {
        [*] --> Start: event.type=start
        Start --> Streaming: 标记 addedPartial=true
        Streaming --> Streaming: text/thinking/toolcall delta
        Streaming --> Final: event=done/error
        Streaming --> Final: 流自然结束
    }

    state Final <<choice>>
    Final --> Push: addedPartial?<br/>替换 / push
    Push --> Emit: emit message_end
    Emit --> Return: 返回 AssistantMessage
```

**为什么有 `addedPartial` 标志位？**
- `start` 事件携带**最初的 partial 消息**——循环要把它先 push 到 context，让"当前正在流的消息"对其它 hook 可见
- `done`/`error` 之后流可能不会再产生 `start`——但 context 里**必须**有这条消息
- `addedPartial` 区分两种情况：
  - `true`：`context.messages[length-1]` 已经是 partial 了，**替换**即可
  - `false`：流里没收到 `start`（很少见但合法），**push** 一条
- 同样的逻辑在 line 345-365 出现两次（done/error 一次 + 流自然结束一次）—— 因为**正常完成和异常完成都需要落库**

**为什么 `transformContext` 放在 `convertToLlm` 之前？**
- `transformContext` 操作 `AgentMessage[]`（应用层抽象）
- `convertToLlm` 操作内部类型→provider 类型
- 顺序：**先剪裁，再转译**——剪裁后的消息少，转译也快
- 而且"上下文压缩"这类操作**不该看到 LLM 消息**（它管的是应用层的对话历史）

**为什么 `getApiKey` 每次 LLM 调用都跑一次？**
- OAuth token 短效（GitHub Copilot 等），**一个长跑 agent 可能跨多个 token 生命周期**
- 抽到 config 里、每次拉新，是"宁可重复取，也不能过期"的策略

---

## 6. 工具执行层

### 6.1 整体包含关系

```mermaid
graph TD
    ETC["executeToolCalls<br/>(分发器)"]
    ETC -->|toolExecution=sequential<br/>或工具有 sequential| SEQ["executeToolCallsSequential"]
    ETC -->|否则| PAR["executeToolCallsParallel"]

    SEQ --> PTC["prepareToolCall<br/>(每个 toolCall)"]
    PAR --> PTC

    PTC -->|kind=prepared| EXEC["executePreparedToolCall<br/>(真跑)"]
    PTC -->|kind=immediate| SKIP["直接当错误返回<br/>(工具没找到/校验失败)"]

    EXEC --> FIN["finalizeExecutedToolCall<br/>(after 钩子)"]
    FIN --> EMIT1["emitToolExecutionEnd"]
    FIN --> MAKE["createToolResultMessage"]
    MAKE --> EMIT2["emitToolResultMessage<br/>(message_start + end)"]
```

### 6.2 工具生命周期（单工具视角）

```mermaid
flowchart LR
    A[toolCall 块] --> B[prepareToolCallArguments<br/>工具自己的 pre-process]
    B --> C[validateToolArguments<br/>schema 校验]
    C --> D{config.beforeToolCall?}
    D -->|是| E[before 钩子]
    D -->|否| F[signal 检查]
    E -->|block=true| ERR1[立刻返回 error 工具结果]
    E -->|通过| F
    F -->|aborted| ERR2[返回 aborted 错误]
    F -->|正常| G[executePreparedToolCall]
    G -->|抛错| ERR3[捕获成 error 工具结果]
    G -->|正常| H[finalizeExecutedToolCall]
    H --> I{config.afterToolCall?}
    I -->|是| J[after 钩子<br/>按字段覆盖]
    I -->|否| K[直接打包]
    J --> K
    K --> L[emit tool_execution_end]
    L --> M[emit message_start/end]
```

**为什么拆 `prepare` / `execute` / `finalize` 三段？**
- `prepare`：**纯函数式**（不做 I/O）——参数预处理、schema 校验、`beforeToolCall` 拦截
- `execute`：**真跑工具**，会 await，可能抛错，可能发 `tool_execution_update`
- `finalize`：**结果改写**——`afterToolCall` 可以审核、改写、决定是否 terminate

**为什么 `prepare` 要在并行模式下串行做？**
- 校验 / 拦截是轻量的，但**能 fail-fast**——一个参数不合法的工具，后面并行执行就是浪费
- 看 `executeToolCallsParallel` line 461-500：先**同步跑完所有 prepare**，再把"活的"塞进 thunk 数组
- 这种"**预检串行、执行并行**"是控制并发安全 + 最大化并行的经典模式

### 6.3 串行 vs 并行的"事件顺序"差异

```mermaid
sequenceDiagram
    participant L as runLoop
    participant S as executeToolCallsSequential
    participant P as executeToolCallsParallel
    participant T as prepareToolCall

    Note over L: === 串行模式 ===
    L->>S: executeToolCalls
    loop 每个 toolCall 顺序
        S->>T: prepareToolCall
        S-->>L: tool_execution_start
        S-->>L: tool_execution_end
        S-->>L: message_start
        S-->>L: message_end
    end

    Note over L: === 并行模式 ===
    L->>P: executeToolCalls
    loop 每个 toolCall 预检
        P->>T: prepareToolCall
        P-->>L: tool_execution_start
    end
    par 并行执行
        P-->>L: tool_execution_end (完成顺序)
        P-->>L: tool_execution_end (完成顺序)
    end
    Note over P: 此时再按 assistant 原始顺序发 message_start/end
    P-->>L: message_start (按原顺序)
    P-->>L: message_end (按原顺序)
```

**为什么事件顺序这样安排？**
- `tool_execution_start/end` —— 反映**真实完成顺序**，UI 要实时显示进度
- `message_start/end` —— 反映 **LLM 看到的顺序**（assistant 源顺序），否则下一轮 LLM 拿到错位的工具结果
- 串行模式天然两者一致；并行模式必须**把 message 事件延后**到 `Promise.all` 之后
- 这是 `executeToolCallsParallel` line 502-510 的微妙之处

### 6.4 `FinalizedToolCallEntry` 联合类型设计

```typescript
type FinalizedToolCallEntry =
  | FinalizedToolCallOutcome       // immediate：已经完成
  | (() => Promise<FinalizedToolCallOutcome>);  // prepared：待执行
```

**为什么是这个形态？**
- 并行模式下，`prepare` 时**无法预知**结果是 immediate（错误）还是需要真正执行
- 立即错误 → 直接把 `outcome` 塞进数组（line 471-477）
- 需要执行 → 塞一个 thunk（line 484-496）
- 最后用 `typeof entry === "function" ? entry() : Promise.resolve(entry)`（line 503）统一收敛
- **统一收口 + 保留类型差异**——比"全部包成 thunk"少一次 microtask

---

## 7. 关键设计哲学（"为什么"汇总）

| 决策 | 表层原因 | 深层原因 |
|---|---|---|
| `AgentMessage[]` 内部 + `Message[]` 边界 | 协议兼容 | 让自定义消息（UI 通知/artifact）不污染 LLM 协议 |
| Hook 函数（`transformContext`/`beforeToolCall`/...） | 配置驱动 | 业务策略和执行机制解耦，循环本身可被任何上层复用 |
| `convertToLlm` **必填**而非可选 | 类型安全 | 强制调用方声明"如何映射"，而不是循环瞎猜 |
| `convertToLlm` / `transformContext` 契约"**不要抛错**" | 健壮性 | 这俩是循环的"必要环节"，抛错会让 agent 死得很难看 |
| 双层 while（inner/outer） | 控制 turn 边界 | 一个 turn 内换 model 没必要；多 turn 间才需要换 |
| Steering 在 turn_end 后拉 | 不打断工具 | 用户在工具执行中打字，等当前工具跑完再注入新指令 |
| Follow-up 在"无 toolCall & 无 steering"时拉 | 自然排队 | "我现在没活了，下一条是谁的" |
| 工具 prepare 串行、execute 可并行 | fail-fast + 最大并发 | 校验便宜无副作用；执行慢可能重资源 |
| `getApiKey` 每次调用 | 短效 token | OAuth 时代 token 会过期，跨长 agent run 必须重拉 |
| `addedPartial` 状态标志 | 状态机补全 | 防御性编程：流协议不保证一定有 `start` 事件 |
| `streamFn` 可注入 | 测试 + 代理 | 不让循环和具体 provider SDK 耦合，方便 mock / 代理 |
| `PartialOverride` after 字段合并 | 不打破既有契约 | 想改 `terminate` 不需要重发 `content/details` |
| `terminate` 必须**所有工具**都同意 | 防止误停 | 一个工具认为完成 ≠ 应该停；要达成共识 |

---

## 8. 状态变化全景（一次典型 run 的事件序列）

```mermaid
sequenceDiagram
    participant U as User
    participant API as agentLoop
    participant RL as runLoop
    participant S as streamAssistantResponse
    participant T as executeToolCalls
    participant LLM as streamSimple

    U->>API: prompts + context
    API-->>U: EventStream (返回)
    API->>RL: runAgentLoop

    Note over RL: outer iteration 1
    RL-->>U: event: agent_start
    RL-->>U: event: turn_start
    loop 每个 prompt
        RL-->>U: event: message_start
        RL-->>U: event: message_end
    end

    Note over RL,S: inner iteration - 调 LLM
    RL->>S: streamAssistantResponse
    S->>LLM: convertToLlm + streamSimple
    LLM-->>S: event: start (partial)
    S-->>U: event: message_start
    LLM-->>S: event: text_delta
    S-->>U: event: message_update
    LLM-->>S: event: toolcall_end
    LLM-->>S: event: done
    S-->>U: event: message_end
    S-->>RL: AssistantMessage

    Note over RL,T: inner iteration - 执行工具
    RL->>T: executeToolCalls
    T-->>U: event: tool_execution_start
    T-->>U: event: tool_execution_update (若干)
    T-->>U: event: tool_execution_end
    T-->>U: event: message_start
    T-->>U: event: message_end
    T-->>RL: ToolResultMessage[]

    RL-->>U: event: turn_end
    RL->>RL: prepareNextTurn? shouldStopAfterTurn?
    RL->>RL: 拉 steering 消息

    Note over RL: inner 退出后，进入 outer
    RL->>RL: 拉 follow-up 消息
    alt 有 follow-up
        Note over RL: outer iteration 2<br/>(重新进入 inner)
    else 没有
        RL-->>U: event: agent_end
    end
```

---

## 9. 一图带走：作者脑子里在画什么

```mermaid
mindmap
  root((agent-loop<br/>的脑子里))
    事件优先
      所有变化都 emit 出去
      UI/测试/日志订阅同一流
      没有"内部状态"和"外部观察"之分
    内部类型 vs 外部类型
      AgentMessage 包含自定义
      Message 只能 LLM 认识
      边界就一道：convertToLlm
    注入而非继承
      8 个 hook 都是 config 函数
      循环本身无策略
      上层自由组合
    分层 while
      内层：一个 turn 内
      外层：多 turn 之间
      steering vs follow-up 不混淆
    工具安全
      prepare 必串行
      execute 可并行
      每步都有 abort 检查
      before/after 双层 hook
    防御式状态机
      addedPartial 防 start 缺失
      PartialOverride 字段独立
      terminate 需全员同意
    可测性
      streamFn 可注入
      契约明确不抛错
      每个阶段事件可断言
```

---

## 10. 跟读建议（按上面这个心智模型走）

1. **先读 §3 数据流图** —— 抓住"内/外类型边界"这个最核心的隐喻
2. **再读 §4 主循环图** —— 理解"双层 while + steering/follow-up 拆开"是这套架构的骨架
3. **再读 §6 工具层** —— 体会"prepare/execute/finalize 三段"和"preflight 串行 + execute 并行"
4. **最后对照源码** —— 看完图再读 line 155-269 的 `runLoop`，会发现每一行都在回应上面某张图
5. **随手翻 §7 哲学表** —— 任何"为什么这样写"的疑问，去那张表查

> 一旦你接受了"事件是真相、循环是容器、类型有边界"这三件事，整个文件读起来会顺畅很多。
