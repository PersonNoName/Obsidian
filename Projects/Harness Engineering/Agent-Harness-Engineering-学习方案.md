# Agent 领域 Harness Engineering 系统学习方案

> **版本**: v1.0 | **日期**: 2026-04-13 | **定位**: 面向有一定编程基础、希望深入掌握 AI Agent 运行时控制与编排能力的工程师

---

## 目录

1. [概念定义与边界厘清](#1-概念定义与边界厘清)
2. [分阶段学习路径](#2-分阶段学习路径)
3. [知识体系全景](#3-知识体系全景)
4. [推荐参考项目](#4-推荐参考项目)
5. [实战项目设计](#5-实战项目设计)
6. [前置知识要求](#6-前置知识要求)
7. [推荐资料](#7-推荐资料)
8. [8~12 周学习计划表](#8-812-周学习计划表)
9. [架构图集](#9-架构图集)

---

## 1. 概念定义与边界厘清

### 1.1 什么是 Agent Harness Engineering

**Agent Harness** 是 AI Agent 的**运行时控制层（Runtime Control Layer）**，它不是 Agent 的"大脑"（LLM），也不是 Agent 的"四肢"（Tools），而是连接大脑与四肢、控制 Agent 如何思考-行动-反馈-恢复的**中枢神经系统与骨骼框架**。

**Harness Engineering** = 设计、实现、调优、治理这个控制层的工程实践。

具体而言，Agent Harness 负责：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent Harness (控制层)                        │
│                                                                   │
│  ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────┐  │
│  │Execution │  │    Tool       │  │  Memory /   │  │Planning /│  │
│  │  Loop    │──│Orchestration │──│  Context    │──│  Task    │  │
│  │Control   │  │  & Routing   │  │ Management  │  │Decompose │  │
│  └──────────┘  └──────────────┘  └────────────┘  └──────────┘  │
│                                                                   │
│  ┌──────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────┐  │
│  │Sandbox / │  │Observability │  │ Evaluation  │  │Guardrails│  │
│  │Env Ctrl  │  │Trace/Logging │  │ & Benchmark │  │& Policy  │  │
│  └──────────┘  └──────────────┘  └────────────┘  └──────────┘  │
│                                                                   │
│  ┌──────────┐  ┌──────────────┐  ┌────────────┐                 │
│  │Reliability│  │Human-in-the │  │Multi-Agent │                 │
│  │Engineering│  │   -Loop      │  │Coordination│                 │
│  └──────────┘  └──────────────┘  └────────────┘                 │
└─────────────────────────────────────────────────────────────────┘
```

**一句话定义**：Agent Harness Engineering 是让 LLM Agent 从"能跑"变为"能用、可控、可观测、可治理、可扩展"的系统工程学科。

### 1.2 与相关概念的区别

| 维度 | Prompt Engineering | Context Engineering | Agent Framework | Workflow Engine | **Harness Engineering** |
|------|-------------------|-------------------|-----------------|-----------------|------------------------|
| **关注层** | 单次 LLM 调用的输入 | LLM 调用的完整上下文窗口 | Agent 的搭建脚手架 | 确定性流程编排 | Agent 运行时控制全生命周期 |
| **核心问题** | 如何写好 prompt | 如何组装最优 context | 如何快速搭建 agent | 如何编排确定性步骤 | **如何让 agent 可靠、可控、可观测地运行** |
| **典型产物** | system prompt、few-shot examples | context window 管理策略 | Agent 类、Chain 类 | DAG/状态机定义 | **execution loop、tool router、retry policy、trace pipeline、eval harness、sandbox** |
| **技术深度** | 浅（文本层） | 中（信息管理层） | 中（抽象层） | 中（编排层） | **深（系统工程层）** |
| **关注时间线** | 单次请求 | 单次请求 | 开发时 | 开发时 | **开发时 + 运行时 + 运维时** |
| **类比** | 写剧本台词 | 准备演员资料 | 搭建舞台骨架 | 编排走位顺序 | **导演 + 舞台监督 + 安全官 + 质检员** |
| **代表** | OpenAI 提示词指南 | Anthropic context engineering | LangChain, CrewAI | Temporal, Prefect | **SWE-agent harness, OpenHands runtime, Claude Code agent loop, Inspect AI eval harness** |

### 1.3 Agent Harness Engineering 解决的核心问题

```
┌─────────────────────────────────────────────────────┐
│            Agent 从"Demo"到"Production"的鸿沟         │
├─────────────────────────────────────────────────────┤
│                                                       │
│   Demo Agent              Production Agent            │
│   ─────────              ────────────────             │
│   happy path only   →    error recovery               │
│   无限 token 预算   →    token budget control          │
│   单工具调用        →    tool routing + fallback       │
│   无状态            →    memory + context mgmt         │
│   无监控            →    full observability            │
│   无评估            →    eval + regression testing     │
│   无安全边界        →    sandbox + guardrails          │
│   单 agent          →    multi-agent coordination      │
│   人工重启          →    auto-recovery + HITL          │
│                                                       │
│   这个鸿沟 = Harness Engineering 要填的坑              │
└─────────────────────────────────────────────────────┘
```

**核心问题清单**：

| # | 问题 | Harness 的回答 |
|---|------|---------------|
| 1 | Agent 卡死在循环里怎么办？ | Execution loop 超时 + 最大步数限制 + 死循环检测 |
| 2 | Tool 调用失败怎么办？ | Retry with backoff + fallback tool + graceful degradation |
| 3 | Agent 做了危险操作怎么办？ | Sandbox 隔离 + permission policy + human approval gate |
| 4 | 上下文窗口爆了怎么办？ | Context window management + summarization + memory tier |
| 5 | 怎么知道 Agent 在做什么？ | Trace + structured logging + step-level observability |
| 6 | 怎么知道 Agent 做得好不好？ | Eval harness + benchmark suite + regression testing |
| 7 | 如何扩展 Agent 能力？ | MCP/Skills/Tools 插件体系 + tool registry |
| 8 | 多个 Agent 如何协作？ | Multi-agent protocol + message passing + shared state |
| 9 | 如何让人介入 Agent 决策？ | Human-in-the-loop gate + approval workflow |
| 10 | 如何保证幂等和一致性？ | Idempotency key + checkpoint + state recovery |

---

## 2. 分阶段学习路径

### 总览

```
Phase 1: 入门         Phase 2: 进阶          Phase 3: 提升          Phase 4: 架构实战
(Week 1-3)           (Week 4-6)             (Week 7-9)            (Week 10-12)
                                                                    
┌──────────┐        ┌──────────────┐       ┌──────────────┐      ┌──────────────┐
│ 理解Agent│        │ 构建完整的   │       │ 可靠性 +     │      │ 生产级Agent  │
│ 基本循环 │───────→│ Tool + Memory│──────→│ 可观测 +     │─────→│ 架构设计与   │
│ 与控制流 │        │ + Planning   │       │ 评估体系     │      │ 多Agent编排  │
└──────────┘        └──────────────┘       └──────────────┘      └──────────────┘
     │                     │                      │                      │
  输出: 最小               输出: 带工具+          输出: 带 trace+       输出: 生产级
  agent loop              记忆的 agent            eval 的 agent         multi-agent
```

### Phase 1: 入门 — 理解 Agent 执行循环

| 维度        | 内容                                                                                                                                                                                                                                                                                                                  |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **学习周期**  | 3 周（每周 15~20 小时）                                                                                                                                                                                                                                                                                                    |
| **阶段目标**  | 能手写一个最小 agent execution loop；理解 ReAct/tool-use 模式的控制流；能解释 harness 各组件的职责                                                                                                                                                                                                                                            |
| **核心知识点** | ① LLM API 基础（chat completion、function calling、streaming）<br>② ReAct 循环：think → act → observe → repeat<br>③ Execution loop 的实现模式（while loop + state machine）<br>④ Tool calling 协议（OpenAI function calling / Anthropic tool_use）<br>⑤ 基本错误处理：max steps、timeout、parse error recovery<br>⑥ Agent 与 Chain/Pipeline 的本质区别 |
| **推荐顺序**  | LLM API → function calling → 手写 ReAct loop → 加入 max_steps/timeout → 阅读 SWE-agent 源码入门                                                                                                                                                                                                                               |
| **阶段输出物** | ✅ 一个从零手写的 minimal agent loop（无框架依赖）<br>✅ execution loop 状态机图<br>✅ 学习笔记：harness 各组件关系图                                                                                                                                                                                                                               |

### Phase 2: 进阶 — Tool 编排 + Memory + Planning

| 维度 | 内容 |
|------|------|
| **学习周期** | 3 周（每周 15~20 小时） |
| **阶段目标** | 实现 tool registry + routing；实现 short-term / long-term memory；实现基本的 planning + task decomposition；理解 MCP 协议 |
| **核心知识点** | ① Tool registry 设计（注册、发现、schema validation）<br>② Tool routing 策略（LLM 自选 vs 规则路由 vs 混合）<br>③ MCP (Model Context Protocol) 协议与实现<br>④ Memory 分层：working memory / episodic / semantic / procedural<br>⑤ Context window management：压缩、摘要、滑动窗口<br>⑥ Planning 模式：single-shot plan / iterative plan / plan-then-execute<br>⑦ Task decomposition 策略 |
| **推荐顺序** | Tool registry → MCP 基础 → Memory 实现 → Context management → Planning 模式 → 整合到 execution loop |
| **阶段输出物** | ✅ 带 tool orchestration 的 agent<br>✅ 带 memory（至少 2 层）的 agent<br>✅ MCP server/client 的最小实现<br>✅ 一份 tool/memory/planning 技术选型笔记 |

### Phase 3: 提升 — 可靠性 + 可观测性 + 评估

| 维度        | 内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **学习周期**  | 3 周（每周 15~20 小时）                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **阶段目标**  | 实现 retry/fallback/recovery 机制；构建 trace + logging pipeline；搭建 eval harness + benchmark；理解 guardrails 与 sandbox                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **核心知识点** | ① Reliability patterns：retry with exponential backoff、circuit breaker、fallback、timeout、idempotency<br>② Checkpoint + state recovery<br>③ Structured logging for agents（step-level、tool-level、LLM-call-level）<br>④ Distributed tracing for agent chains（OpenTelemetry 集成）<br>⑤ Eval harness 设计：task suite、scorer、reporter<br>⑥ Benchmark 方法论：SWE-bench、HumanEval、GAIA<br>⑦ Sandbox：Docker-based、E2B、gVisor<br>⑧ Guardrails：input validation、output filtering、permission scoping<br>⑨ Human-in-the-loop：approval gate、escalation |
| **推荐顺序**  | Retry/fallback → Checkpoint → Structured logging → Trace → Eval harness → Sandbox → Guardrails → HITL                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **阶段输出物** | ✅ 带完整可靠性机制的 agent<br>✅ Trace dashboard（可基于 Langfuse/Braintrust）<br>✅ 一套 eval harness（至少 10 个 test case）<br>✅ 一份 agent 可靠性 checklist                                                                                                                                                                                                                                                                                                                                                                                            |

### Phase 4: 架构实战 — 生产级 Agent 系统

| 维度 | 内容 |
|------|------|
| **学习周期** | 3 周（每周 15~20 小时） |
| **阶段目标** | 设计生产级 agent 架构；实现 multi-agent 协调；完成端到端 agent 系统包含 CI/eval/deploy |
| **核心知识点** | ① Multi-agent patterns：supervisor、peer-to-peer、hierarchy、blackboard<br>② Agent communication protocol（A2A / 自定义 message bus）<br>③ Shared state management<br>④ Agent-as-a-Service 架构设计<br>⑤ CI for agents：eval-in-CI、regression gate<br>⑥ Agent versioning + A/B testing<br>⑦ Cost control：token budget、caching strategy<br>⑧ Production monitoring + alerting<br>⑨ Scaling patterns：horizontal agent scaling、queue-based dispatch |
| **推荐顺序** | Multi-agent patterns → Communication protocol → Production architecture → CI/CD for agents → Monitoring + Cost control |
| **阶段输出物** | ✅ 一个生产级 multi-agent 系统原型<br>✅ 架构设计文档（含 ADR）<br>✅ CI pipeline 含 eval gate<br>✅ 一份完整的 Agent Harness Engineering 知识地图 |

---

## 3. 知识体系全景

### 3.0 知识体系架构图

```
                    ┌─────────────────────────────────┐
                    │    Production-Ready Agent        │
                    │       Architecture               │
                    └──────────┬──────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
    ┌─────▼─────┐      ┌──────▼──────┐     ┌──────▼──────┐
    │Multi-Agent │      │ Reliability │     │   Cost &    │
    │Coordination│      │ Engineering │     │  Scaling    │
    └─────┬─────┘      └──────┬──────┘     └──────┬──────┘
          │                    │                    │
    ┌─────┴────────────────────┴────────────────────┴─────┐
    │                GOVERNANCE LAYER                       │
    │  ┌───────────┐ ┌──────────┐ ┌───────────────────┐   │
    │  │ Guardrails│ │   HITL   │ │ Sandbox/Env Ctrl  │   │
    │  └───────────┘ └──────────┘ └───────────────────┘   │
    └──────────────────────┬──────────────────────────────┘
                           │
    ┌──────────────────────┴──────────────────────────────┐
    │              OBSERVABILITY LAYER                      │
    │  ┌──────┐  ┌────────────┐  ┌────────────────────┐   │
    │  │Trace │  │  Eval/     │  │ Structured Logging │   │
    │  │      │  │  Benchmark │  │                    │   │
    │  └──────┘  └────────────┘  └────────────────────┘   │
    └──────────────────────┬──────────────────────────────┘
                           │
    ┌──────────────────────┴──────────────────────────────┐
    │               RUNTIME CORE                           │
    │  ┌────────────┐ ┌──────────┐ ┌───────────────────┐  │
    │  │ Execution  │ │  Tool    │ │  Memory/Context   │  │
    │  │   Loop     │ │Orchestr. │ │   Management      │  │
    │  └────────────┘ └──────────┘ └───────────────────┘  │
    │  ┌────────────┐                                      │
    │  │ Planning / │                                      │
    │  │Task Decomp │                                      │
    │  └────────────┘                                      │
    └──────────────────────┬──────────────────────────────┘
                           │
    ┌──────────────────────┴──────────────────────────────┐
    │              FOUNDATION LAYER                        │
    │  ┌──────────────┐  ┌─────────────────────────────┐  │
    │  │ LLM API /    │  │  Agent 基础概念             │  │
    │  │ Function Call │  │  (ReAct, tool-use, loops)  │  │
    │  └──────────────┘  └─────────────────────────────┘  │
    └─────────────────────────────────────────────────────┘
```

### 3.1 Agent 基础

| 维度                | 内容                                                                                                                                                        |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **学什么**           | Agent 定义（Perception → Reasoning → Action 循环）、ReAct 模式、Tool-augmented LLM、Agent vs Chain vs Pipeline 的区别、主流 agent 架构分类（单循环/树搜索/图状态机）                       |
| **学到什么深度**        | 能不依赖框架，纯 Python 手写一个 ReAct agent；能画出任意 agent 系统的 execution flow                                                                                           |
| **常见误区**          | ❌ "Agent = LLM + Prompt" → Agent 的核心是循环控制，不是 prompt<br>❌ "用了 LangChain 就是 agent" → 框架是脚手架，不是 harness<br>❌ "Agent 天然不确定所以不用管控制 " → harness 的意义就是在不确定中建立确定性 |
| **与 Harness 的关系** | Harness 的操作对象就是 agent；不理解 agent 基础就无法设计 harness                                                                                                           |

### 3.2 Tool Calling / Skills / MCP

| 维度                | 内容                                                                                                                                                                                                                      |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **学什么**           | OpenAI function calling 协议、Anthropic tool_use 协议、MCP (Model Context Protocol) 规范、Tool schema 设计（JSON Schema）、Tool registry/discovery、Tool routing（LLM 自选 vs 硬编码 vs 混合路由）、Tool 权限控制、Tool 结果解析与错误处理、Skills 抽象（高阶 tool 组合） |
| **学到什么深度**        | 能实现一个支持动态注册/发现的 tool registry；能实现 MCP server + client；能设计 tool permission model                                                                                                                                         |
| **常见误区**          | ❌ "Tool 越多越好" → 工具过多会严重降低 LLM 选择准确率，需要 tool routing 策略<br>❌ "MCP 只是调用 API" → MCP 是标准化的 tool/resource/prompt 发现与交互协议<br>❌ "Tool 失败就重试" → 需要区分 transient/permanent failure，设计 fallback                                    |
| **与 Harness 的关系** | Tool orchestration 是 harness 的核心子系统之一；harness 负责 tool 的注册、路由、权限、retry、结果处理                                                                                                                                              |

**Tool Orchestration 架构图**：

```
                        ┌──────────────┐
                        │   LLM 决策   │
                        │ "use tool X" │
                        └──────┬───────┘
                               │
                    ┌──────────▼──────────┐
                    │   Tool Router       │
                    │ ┌────────────────┐  │
                    │ │ Permission     │  │
                    │ │ Check          │  │
                    │ └────────────────┘  │
                    │ ┌────────────────┐  │
                    │ │ Rate Limiter   │  │
                    │ └────────────────┘  │
                    │ ┌────────────────┐  │
                    │ │ Schema Valid.  │  │
                    │ └────────────────┘  │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
        ┌─────▼────┐   ┌──────▼─────┐   ┌─────▼──────┐
        │Local Tool │   │ MCP Server │   │ REST API   │
        │(in-proc)  │   │ (stdio/sse)│   │ (http)     │
        └─────┬────┘   └──────┬─────┘   └─────┬──────┘
              │                │                │
              └────────────────┼────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Result Handler    │
                    │ ┌────────────────┐  │
                    │ │ Retry/Fallback │  │
                    │ └────────────────┘  │
                    │ ┌────────────────┐  │
                    │ │ Result Parse   │  │
                    │ └────────────────┘  │
                    │ ┌────────────────┐  │
                    │ │ Trace Emit     │  │
                    │ └────────────────┘  │
                    └──────────┬──────────┘
                               │
                        ┌──────▼───────┐
                        │ 返回 Agent   │
                        │   Loop       │
                        └──────────────┘
```

### 3.3 Runtime / Orchestration

| 维度 | 内容 |
|------|------|
| **学什么** | Execution loop 设计模式（while-loop / state-machine / event-driven）、Step 抽象（LLM call step / tool call step / human step）、Loop termination 条件（max steps / goal reached / error budget exhausted / human stop）、Async execution（并行 tool call）、Streaming 处理、State management（FSM / event sourcing）、Orchestration patterns（sequential / parallel / conditional / loop） |
| **学到什么深度** | 能设计并实现一个生产级 execution loop，支持：step-level timeout、graceful shutdown、checkpoint/resume、parallel tool execution |
| **常见误区** | ❌ "while True + try/except 就够了" → 生产级 loop 需要 state machine、circuit breaker、graceful degradation<br>❌ "用 async 就是并行" → 并行 tool 调用需要依赖分析 + 结果合并策略<br>❌ "orchestration = workflow engine" → Agent orchestration 是非确定性的，每一步都由 LLM 决定 |
| **与 Harness 的关系** | Execution loop 是 harness 的心脏；所有其他组件（tool、memory、eval）都挂载在 loop 上 |

**Execution Loop 状态机**：

```
                    ┌─────────┐
                    │  INIT   │
                    └────┬────┘
                         │ initialize context
                         ▼
              ┌──────────────────────┐
              │     PLANNING         │◄────────────────┐
              │  (optional phase)    │                  │
              └──────────┬───────────┘                  │
                         │                              │
                         ▼                              │
              ┌──────────────────────┐         ┌───────┴───────┐
         ┌───→│    LLM_THINKING      │         │   REPLANNING  │
         │    │  (send to LLM)       │         │  (on failure/ │
         │    └──────────┬───────────┘         │   new info)   │
         │               │                     └───────────────┘
         │               ▼                              ▲
         │    ┌──────────────────────┐                  │
         │    │   PARSE_RESPONSE     │                  │
         │    │ (extract action/tool)│                  │
         │    └──────────┬───────────┘                  │
         │               │                              │
         │         ┌─────┴──────┐                       │
         │         │            │                       │
         │         ▼            ▼                       │
         │  ┌────────────┐ ┌──────────┐                │
         │  │ TOOL_CALL  │ │  FINAL   │                │
         │  │ (execute)  │ │ ANSWER   │                │
         │  └─────┬──────┘ └────┬─────┘                │
         │        │              │                      │
         │        ▼              ▼                      │
         │  ┌────────────┐ ┌──────────┐                │
         │  │  OBSERVE   │ │ COMPLETE │                │
         │  │ (get result)│ └──────────┘                │
         │  └─────┬──────┘                              │
         │        │                                     │
         │        ├──── success ────→ loop back ────┘   │
         │        │                     (上方)          │
         │        ├──── retriable err → RETRY ──────┘   │
         │        │                                     │
         │        └──── fatal err ──→ ERROR ────────────┘
         │                            │
         │                       ┌────▼────┐
         │                       │  HITL   │
         │                       │  GATE   │
         │                       └────┬────┘
         │                            │
         │              human approves│
         └────────────────────────────┘
         
    终止条件:
    ├── max_steps reached → TERMINATED (budget)
    ├── max_time reached  → TERMINATED (timeout)  
    ├── goal achieved     → COMPLETE
    ├── error budget exhausted → FAILED
    └── human cancellation → CANCELLED
```

### 3.4 Memory / Context Management

| 维度 | 内容 |
|------|------|
| **学什么** | Memory 分层模型（Working / Episodic / Semantic / Procedural）、Context window management（滑动窗口、摘要压缩、重要性评分）、Short-term memory（conversation buffer、tool results cache）、Long-term memory（vector store、knowledge graph）、Memory retrieval 策略（recency / relevance / importance）、Token budget 管理 |
| **学到什么深度** | 能实现至少 3 层 memory 系统；能设计 context window 压缩策略使 agent 在长对话中不丢关键信息 |
| **常见误区** | ❌ "把所有历史塞进 context" → 会突破窗口限制且降低质量<br>❌ "RAG = memory" → RAG 只是 memory 的一种检索方式<br>❌ "向量检索就够了" → 需要结合 recency + relevance + importance 三维排序 |
| **与 Harness 的关系** | Harness 负责管理 agent 的所有记忆的生命周期：写入时机、检索策略、清理策略、上下文组装 |

**Memory 分层架构**：

```
┌──────────────────────────────────────────────────────┐
│                  Context Window (有限)                │
│  ┌────────────────────────────────────────────────┐  │
│  │  System Prompt + Instructions                  │  │
│  ├────────────────────────────────────────────────┤  │
│  │  Working Memory (当前任务状态/变量)             │  │
│  ├────────────────────────────────────────────────┤  │
│  │  Retrieved Episodic Memory (相关历史交互)       │  │
│  ├────────────────────────────────────────────────┤  │
│  │  Retrieved Semantic Memory (相关知识)           │  │
│  ├────────────────────────────────────────────────┤  │
│  │  Recent Conversation (滑动窗口)                 │  │
│  ├────────────────────────────────────────────────┤  │
│  │  Tool Results (最近 N 次)                       │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
                         ▲  检索
                         │
          ┌──────────────┼──────────────┐
          │              │              │
   ┌──────▼─────┐ ┌─────▼──────┐ ┌────▼───────┐
   │  Episodic  │ │  Semantic  │ │ Procedural │
   │  Memory    │ │  Memory    │ │  Memory    │
   │ (历史交互) │ │ (知识库)   │ │ (技能/流程)│
   │ Vector DB  │ │ Vector DB  │ │ Code/Config│
   └────────────┘ └────────────┘ └────────────┘
```

### 3.5 Planning / Task Decomposition

| 维度                | 内容                                                                                                                                                                                                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **学什么**           | Planning 范式（Plan-then-Execute / Interleaved Planning / Hierarchical Planning）、Task decomposition 策略（top-down / bottom-up / iterative refinement）、Plan representation（自然语言 / JSON / DAG / tree）、Plan validation + re-planning on failure、Goal tracking + progress estimation |
| **学到什么深度**        | 能实现 plan-then-execute 模式；能实现动态 re-planning；能设计 plan validation 机制                                                                                                                                                                                                           |
| **常见误区**          | ❌ "让 LLM 一次性给出完整计划" → 长计划质量差，需要迭代细化<br>❌ "计划是一成不变的" → 必须支持 re-planning，因为执行中会有意外<br>❌ "不需要 planning" → 复杂任务无 planning 会导致 agent 在低层循环浪费 token                                                                                                                               |
| **与 Harness 的关系** | Harness 控制 planning 的时机、深度、验证、以及 plan 与 execution loop 的交互                                                                                                                                                                                                                  |

### 3.6 Sandbox / Environment Control

| 维度 | 内容 |
|------|------|
| **学什么** | Sandbox 类型（Docker container / microVM / gVisor / WASM / E2B）、文件系统隔离、网络隔离、进程隔离、Resource limits（CPU/memory/disk/network）、Sandbox lifecycle（create / snapshot / restore / destroy）、Environment provisioning（base image / dependencies / state setup） |
| **学到什么深度** | 能用 Docker 实现一个 agent sandbox；理解 gVisor/microVM 的隔离原理；能设计 sandbox snapshot/restore 机制 |
| **常见误区** | ❌ "在宿主机跑 agent 也行" → 代码执行类 agent 必须隔离，否则安全风险极高<br>❌ "Docker 就是万能的 sandbox" → Docker 隔离不够强，需根据安全等级选择方案<br>❌ "每次新建 sandbox" → 应复用 sandbox + snapshot，降低启动延迟 |
| **与 Harness 的关系** | Sandbox 是 harness 的安全执行环境；harness 管理 sandbox 的生命周期、资源分配、快照恢复 |

### 3.7 Observability / Trace / Logging

| 维度                | 内容                                                                                                                                                                                                                                                                                                                                                    |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **学什么**           | Agent-specific observability 三支柱（Trace / Metrics / Logs）、Trace 模型：Span 层次（Agent Run → Step → LLM Call → Tool Call）、Structured logging 设计（step_id、tool_name、token_count、latency、cost）、OpenTelemetry 集成、Trace 存储与可视化（Langfuse / Braintrust / Phoenix）、Agent-specific metrics（steps_per_task, tool_success_rate, token_efficiency, task_completion_rate） |
| **学到什么深度**        | 能为 agent 构建完整的 trace pipeline；能设计 agent metrics dashboard；能用 trace 数据定位 agent 行为问题                                                                                                                                                                                                                                                                    |
| **常见误区**          | ❌ "print 就是 logging" → Agent 需要结构化、层次化、可查询的日志<br>❌ "只看最终结果" → 必须看中间步骤才能诊断问题<br>❌ "复用 web 服务的 observability" → Agent 的 trace 模型与 web 请求完全不同（非确定性、多步、嵌套）                                                                                                                                                                                                |
| **与 Harness 的关系** | Observability 是 harness 的"眼睛"；没有它，harness 就是盲飞                                                                                                                                                                                                                                                                                                        |

**Trace Span 层次架构**：

```
Agent Run Span (trace_id: abc123)
├── duration: 45s
├── total_tokens: 12,340
├── total_cost: $0.15
├── status: completed
│
├─── Step 1 Span (Planning)
│    ├── LLM Call Span
│    │   ├── model: claude-sonnet-4-20250514
│    │   ├── input_tokens: 1200
│    │   ├── output_tokens: 350
│    │   └── latency: 2.1s
│    └── result: plan with 3 tasks
│
├─── Step 2 Span (Tool Execution)
│    ├── LLM Call Span (decide tool)
│    │   └── decision: use "search_code"
│    ├── Tool Call Span
│    │   ├── tool: search_code
│    │   ├── args: {"query": "auth handler"}
│    │   ├── latency: 0.3s
│    │   ├── status: success
│    │   └── result_size: 2.4KB
│    └── LLM Call Span (process result)
│
├─── Step 3 Span (Tool Execution)
│    ├── Tool Call Span
│    │   ├── tool: edit_file
│    │   ├── retry_count: 1    ← 第一次失败后重试
│    │   ├── status: success
│    │   └── latency: 0.8s
│    └── ...
│
└─── Step 4 Span (Final Answer)
     └── LLM Call Span
         └── output: "Done. Fixed the auth bug."
```

### 3.8 Evaluation / Benchmark / Regression Testing

| 维度 | 内容 |
|------|------|
| **学什么** | Eval harness 架构（Task Suite → Agent Runner → Scorer → Reporter）、Task 设计（input / expected output / grading rubric）、Scoring 方法（exact match / fuzzy / LLM-as-judge / human eval）、Benchmark 方法论（SWE-bench / GAIA / τ-bench / AgentBench）、Regression testing for agents（行为回归而非代码回归）、Statistical significance in eval（多次运行 + 置信区间）、Eval-in-CI pipeline |
| **学到什么深度** | 能从零搭建 eval harness；能设计 task suite 覆盖 agent 核心能力；能实现 eval-in-CI |
| **常见误区** | ❌ "跑几个 case 看看就行" → 需要统计显著性，单次运行不可靠<br>❌ "用单元测试测 agent" → Agent 是非确定性的，需要 eval 而非 test<br>❌ "Benchmark 分数 = 实际能力" → Benchmark 有 overfitting 风险，需要自定义 eval |
| **与 Harness 的关系** | Eval harness 是 harness engineering 中"质量保证"的核心；没有 eval，任何 harness 改动都是盲改 |

**Eval Harness 架构**：

```
┌─────────────────────────────────────────────────────┐
│                   Eval Harness                       │
│                                                       │
│  ┌──────────┐    ┌──────────────┐    ┌────────────┐ │
│  │  Task    │    │    Agent     │    │   Scorer   │ │
│  │  Suite   │───→│    Runner    │───→│            │ │
│  │          │    │              │    │            │ │
│  │ - input  │    │ - sandbox    │    │ - exact    │ │
│  │ - expect │    │ - timeout    │    │ - fuzzy    │ │
│  │ - rubric │    │ - max_steps  │    │ - LLM judge│ │
│  └──────────┘    └──────────────┘    └─────┬──────┘ │
│                                            │        │
│                                     ┌──────▼──────┐ │
│                                     │  Reporter   │ │
│                                     │ - score     │ │
│                                     │ - breakdown │ │
│                                     │ - trace     │ │
│                                     │ - CI gate   │ │
│                                     └─────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 3.9 Guardrails / Policy / Permission

| 维度 | 内容 |
|------|------|
| **学什么** | Input guardrails（prompt injection 检测、topic boundary）、Output guardrails（PII filtering、harmful content detection、格式校验）、Tool permission model（allow-list / deny-list / capability-based）、Action approval gate（自动 / 人工审批条件）、Policy engine 设计（rule-based / LLM-based / hybrid）、Rate limiting & cost guardrails |
| **学到什么深度** | 能实现 input/output guardrail pipeline；能设计分级 permission model；能实现 human approval gate |
| **常见误区** | ❌ "LLM 不会做坏事" → LLM 可以被 prompt inject、可以 hallucinate 危险操作<br>❌ "给 agent 全部权限" → 最小权限原则，agent 只应有完成任务所需的最小权限<br>❌ "guardrails 只在上线前做" → 运行时 guardrails 是持续生效的 |
| **与 Harness 的关系** | Guardrails 是 harness 的"安全阀"；harness 在每个 step 的 before/after 挂载 guardrail checks |

### 3.10 Reliability Engineering

| 维度 | 内容 |
|------|------|
| **学什么** | Retry 策略（exponential backoff + jitter）、Timeout 分层（step-level / tool-level / total）、Fallback 模式（降级 tool / 降级 model / 降级策略）、Circuit breaker（连续失败后短路）、Idempotency（tool 调用的幂等性保证）、Checkpoint + Recovery（从断点恢复）、Error classification（transient / permanent / partial）、Graceful degradation（部分功能不可用时的降级方案） |
| **学到什么深度** | 能为 agent 实现完整的 retry/fallback/circuit-breaker 体系；能实现 checkpoint-based recovery；能设计 error budget 策略 |
| **常见误区** | ❌ "catch Exception 就是错误处理" → 需要分类错误、区分策略<br>❌ "重试 3 次就够了" → 需要 backoff + jitter 避免 thundering herd<br>❌ "Agent 失败就重头开始" → checkpoint 可以从最近成功步恢复 |
| **与 Harness 的关系** | 可靠性是 harness 的核心品质指标；一个没有可靠性机制的 harness 无法投入生产 |

**Reliability 机制全景**：

```
┌────────────────────────────────────────────────────────┐
│              Reliability Layer                          │
│                                                          │
│   Per Tool Call:                                         │
│   ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌────────┐  │
│   │ Timeout │→ │ Retry   │→ │ Fallback │→ │Circuit │  │
│   │ (5-30s) │  │ (exp    │  │ (alt tool│  │Breaker │  │
│   │         │  │ backoff)│  │ /alt API)│  │        │  │
│   └─────────┘  └─────────┘  └──────────┘  └────────┘  │
│                                                          │
│   Per Step:                                              │
│   ┌──────────────┐  ┌──────────────┐                    │
│   │ Checkpoint   │  │ Step Timeout │                    │
│   │ (save state) │  │ (60-300s)    │                    │
│   └──────────────┘  └──────────────┘                    │
│                                                          │
│   Per Agent Run:                                         │
│   ┌─────────────┐ ┌──────────┐ ┌───────────────┐       │
│   │Max Steps    │ │ Total    │ │ Error Budget  │       │
│   │ Limit       │ │ Timeout  │ │ (max N errors │       │
│   │ (e.g. 50)   │ │ (e.g.10m)│ │  then abort)  │       │
│   └─────────────┘ └──────────┘ └───────────────┘       │
│                                                          │
│   Recovery:                                              │
│   ┌──────────────────┐  ┌────────────────────────────┐  │
│   │ Checkpoint-based │  │ Idempotency Key            │  │
│   │ Resume           │  │ (prevent duplicate actions) │  │
│   └──────────────────┘  └────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### 3.11 Multi-Agent Coordination

| 维度 | 内容 |
|------|------|
| **学什么** | Multi-agent 模式（Supervisor / Peer-to-peer / Hierarchical / Debate / Blackboard）、Agent communication（消息传递 / 共享状态 / A2A 协议）、Task allocation + routing、Shared state management（锁 / 事件/ CRDT）、Agent lifecycle management（spawn / monitor / terminate）、Conflict resolution（优先级 / 投票 / 仲裁 agent） |
| **学到什么深度** | 能实现 supervisor pattern 的 multi-agent 系统；理解 A2A 协议；能处理 shared state 并发问题 |
| **常见误区** | ❌ "多 agent 总比单 agent 好" → 协调开销可能 > 收益，需要衡量<br>❌ "Agent 间用自然语言通信就行" → 需要结构化消息 + schema 保证可靠通信<br>❌ "不需要管 agent 生命周期" → 僵尸 agent、失控 agent 是真实问题 |
| **与 Harness 的关系** | Multi-agent harness 是 single-agent harness 的扩展；增加了协调、通信、生命周期管理的复杂性 |

### 3.12 Production-Ready Agent Architecture

| 维度 | 内容 |
|------|------|
| **学什么** | Agent-as-a-Service 架构（API / Queue / Event-driven）、Deployment patterns（serverless / long-running / hybrid）、Scaling 策略（horizontal / queue-based / priority-based）、Cost management（token budget / caching / model routing）、Versioning + A/B testing、CI/CD for agents（eval-in-CI / canary deploy）、Monitoring + alerting + on-call、Security hardening（secrets management、audit log） |
| **学到什么深度** | 能设计一个可部署、可扩展、有完整 CI/CD 的 agent 系统架构 |
| **常见误区** | ❌ "Agent 不需要 CI/CD" → Agent 行为会因 prompt/model 变化而 regress，必须 eval-in-CI<br>❌ "直接用最好的模型" → 需要 model routing（简单任务用小模型，复杂任务用大模型）<br>❌ "成本不重要" → 生产级 agent 的 token 成本可以轻松失控 |
| **与 Harness 的关系** | Production readiness 是 harness engineering 的终极目标；所有前面的模块（tool、memory、reliability、eval、guardrail）必须在这里整合 |

---

## 4. 推荐参考项目

### 4.0 项目定位分类

```
                    偏 Framework          偏 Runtime/Harness         偏 Eval/Observability
                    ──────────            ──────────────────         ────────────────────
                    LangGraph             SWE-agent                  Inspect AI
                    CrewAI                OpenHands                  Braintrust
                    AutoGen               Claude Code (闭源参考)      Langfuse
                    Semantic Kernel       Goose                      SWE-bench
                                          OpenAI Agents SDK
                                          Pydantic AI
```

### 4.1 项目详细分析

#### 项目 1：OpenAI Agents SDK

| 维度         | 内容                                                                                                                                             |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| **项目名称**   | [openai/openai-agents-python](https://github.com/openai/openai-agents-python)                                                                  |
| **项目定位**   | 轻量级 agent framework with harness primitives                                                                                                    |
| **技术栈**    | Python, OpenAI API, Pydantic                                                                                                                   |
| **体量与复杂度** | ⭐⭐ 小（~5k LOC），结构清晰，入门友好                                                                                                                        |
| **为什么适合**  | 官方实现了最小化的 agent loop + tool calling + guardrails + handoff（多 agent 交接）+ tracing，是学习 harness 核心概念的最佳起点                                          |
| **重点阅读模块** | `agent_loop.py`（execution loop）、`tool.py`（tool abstraction）、`guardrail.py`（guardrails）、`handoff.py`（multi-agent handoff）、`tracing/` （trace 实现） |
| **建议阶段**   | **Phase 1**                                                                                                                                    |
| **学完能力**   | 理解最小 agent harness 的核心结构；能实现基础 agent loop + tool + guardrail + trace                                                                           |
| **主要难点**   | 理解 handoff 模式与 agent 委托机制                                                                                                                      |
| **分类**     | 🟡 Framework / Harness 混合（偏轻量 harness）                                                                                                         |

#### 项目 2：SWE-agent

| 维度         | 内容                                                                                                                                                                        |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **项目名称**   | [SWE-agent/SWE-agent](https://github.com/SWE-agent/SWE-agent)                                                                                                             |
| **项目定位**   | 面向软件工程任务的 agent，harness 设计是核心亮点                                                                                                                                           |
| **技术栈**    | Python, Docker, LLM API（multi-provider）                                                                                                                                   |
| **体量与复杂度** | ⭐⭐⭐ 中（~15k LOC），harness 层设计精巧                                                                                                                                             |
| **为什么适合**  | 项目名字本身就叫"agent"，但其核心创新在 **harness**：如何设计 agent-computer interface (ACI)、sandbox 环境、tool 定义、execution loop 控制策略。是 harness engineering 的经典教材                                |
| **重点阅读模块** | `sweagent/agent/`（agent loop + planning）、`sweagent/environment/`（sandbox / Docker）、`sweagent/tools/`（tool 定义 + ACI）、`config/`（harness 配置）、`run/`（execution orchestration） |
| **建议阶段**   | **Phase 2**                                                                                                                                                               |
| **学完能力**   | 理解 ACI 设计哲学；能实现 Docker-based sandbox；能设计面向特定任务的 tool suite                                                                                                                |
| **主要难点**   | Docker sandbox 管理；理解 ACI 设计对 agent 性能的影响                                                                                                                                  |
| **分类**     | 🟢 **偏 Runtime / Harness**                                                                                                                                                |

#### 项目 3：OpenHands (formerly OpenDevin)

| 维度         | 内容                                                                                                                                                                                                                                               |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **项目名称**   | [All-Hands-AI/OpenHands](https://github.com/All-Hands-AI/OpenHands)                                                                                                                                                                              |
| **项目定位**   | 通用 AI 软件工程 agent 平台，完整生产级 harness                                                                                                                                                                                                                |
| **技术栈**    | Python, Docker/E2B, FastAPI, React, LLM multi-provider                                                                                                                                                                                           |
| **体量与复杂度** | ⭐⭐⭐⭐ 大（~50k+ LOC），功能全面，架构分层清晰                                                                                                                                                                                                                    |
| **为什么适合**  | 业界最完整的开源 agent harness 实现之一：完整的 runtime/sandbox 抽象、event-driven architecture、multi-provider LLM 支持、plugin 体系、eval harness。从中可以学到"生产级 agent harness 长什么样"                                                                                         |
| **重点阅读模块** | `openhands/runtime/`（sandbox runtime 抽象）、`openhands/controller/`（agent controller / execution loop）、`openhands/events/`（event system）、`openhands/memory/`（memory management）、`openhands/server/`（agent-as-a-service）、`evaluation/`（eval harness） |
| **建议阶段**   | **Phase 3 ~ Phase 4**                                                                                                                                                                                                                            |
| **学完能力**   | 理解生产级 agent 平台架构；掌握 event-driven agent 设计；能实现 agent-as-a-service                                                                                                                                                                                 |
| **主要难点**   | 代码量大，需要先理解事件驱动架构；runtime 抽象层次多                                                                                                                                                                                                                   |
| **分类**     | 🟢 **偏 Runtime / Harness**（最佳参考）                                                                                                                                                                                                                 |

#### 项目 4：LangGraph

| 维度         | 内容                                                                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **项目名称**   | [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)                                                                                              |
| **项目定位**   | 基于图的 agent/workflow 编排框架                                                                                                                                         |
| **技术栈**    | Python, LangChain 生态                                                                                                                                             |
| **体量与复杂度** | ⭐⭐⭐ 中（~20k LOC），概念丰富                                                                                                                                             |
| **为什么适合**  | 用 State Graph 表达 agent execution flow，提供 checkpoint/persistence、human-in-the-loop、streaming、子图等机制。是学习"如何用图/状态机建模 agent 控制流"的最佳参考                                 |
| **重点阅读模块** | `langgraph/graph/`（状态图定义）、`langgraph/checkpoint/`（checkpoint/persistence）、`langgraph/pregel/`（执行引擎：Pregel 计算模型）、`langgraph/prebuilt/`（预制 agent 模式，如 react agent） |
| **建议阶段**   | **Phase 2**                                                                                                                                                      |
| **学完能力**   | 掌握图/状态机建模 agent 控制流；理解 checkpoint-based recovery；理解 streaming + HITL 集成                                                                                          |
| **主要难点**   | Pregel 执行模型较抽象；LangChain 生态依赖重                                                                                                                                   |
| **分类**     | 🟡 Framework / Harness 混合（偏 orchestration）                                                                                                                       |

#### 项目 5：Pydantic AI

| 维度 | 内容 |
|------|------|
| **项目名称** | [pydantic/pydantic-ai](https://github.com/pydantic/pydantic-ai) |
| **项目定位** | 类型安全的 agent framework，强调 runtime 正确性 |
| **技术栈** | Python, Pydantic, multi-provider LLM |
| **体量与复杂度** | ⭐⭐ 小~中（~8k LOC），设计精良 |
| **为什么适合** | 用 Pydantic 的类型系统保证 tool input/output 的正确性；内建 Logfire 集成做 observability；agent 设计简洁但 harness primitives 齐全（retry、validation、structured output、dependency injection） |
| **重点阅读模块** | `pydantic_ai/agent.py`（agent loop）、`pydantic_ai/tools.py`（typed tool system）、`pydantic_ai/models/`（multi-provider abstraction）、`pydantic_ai/result.py`（structured output validation） |
| **建议阶段** | **Phase 1 ~ Phase 2** |
| **学完能力** | 理解类型安全如何提升 harness 可靠性；掌握 structured output validation |
| **主要难点** | 需要理解 Pydantic v2 + Python type system |
| **分类** | 🟡 Framework / Harness 混合（偏类型安全 harness） |

#### 项目 6：Inspect AI

| 维度 | 内容 |
|------|------|
| **项目名称** | [UKGovernmentBEIS/inspect_ai](https://github.com/UKGovernmentBEIS/inspect_ai) |
| **项目定位** | Agent/LLM 评估框架（eval harness 专项） |
| **技术栈** | Python, multi-provider LLM |
| **体量与复杂度** | ⭐⭐⭐ 中（~25k LOC），eval 体系完整 |
| **为什么适合** | 业界最专业的 agent eval harness 开源实现。从 task 定义、solver（agent 策略）、scorer（评分器）、sandbox（Docker/E2B）到 log viewer 全覆盖。是学习"如何系统化评估 agent"的标准参考 |
| **重点阅读模块** | `src/inspect_ai/solver/`（agent 策略/solver）、`src/inspect_ai/scorer/`（评分系统）、`src/inspect_ai/tool/`（tool sandbox）、`src/inspect_ai/log/`（structured trace log）、`src/inspect_ai/dataset/`（task suite 管理） |
| **建议阶段** | **Phase 3** |
| **学完能力** | 能从零搭建完整的 eval harness；理解 eval 方法论；能设计 + 运行 agent benchmark |
| **主要难点** | eval 方法论理解（scoring 策略选择、统计显著性） |
| **分类** | 🔵 **偏 Eval / Observability** |

#### 项目 7：Langfuse

| 维度 | 内容 |
|------|------|
| **项目名称** | [langfuse/langfuse](https://github.com/langfuse/langfuse) |
| **项目定位** | LLM/Agent 可观测性平台 |
| **技术栈** | TypeScript (server), Python (SDK), PostgreSQL, ClickHouse |
| **体量与复杂度** | ⭐⭐⭐⭐ 大（server + SDK），可专注 SDK |
| **为什么适合** | 完整的 agent trace 数据模型（trace → generation → span → event）、集成 eval scoring、cost tracking、prompt management。是学习"agent observability 系统怎么建"的最佳参考 |
| **重点阅读模块** | `python-sdk/langfuse/`（trace SDK 实现）、数据模型设计文档、`web/src/`（trace viewer UI 逻辑，可选） |
| **建议阶段** | **Phase 3** |
| **学完能力** | 理解 agent trace 数据模型；能集成 observability 到自己的 agent；能设计 trace storage |
| **主要难点** | 全栈项目，需选择性阅读；理解 trace 数据模型需要先有 agent 开发经验 |
| **分类** | 🔵 **偏 Eval / Observability** |

#### 项目 8：AutoGen

| 维度 | 内容 |
|------|------|
| **项目名称** | [microsoft/autogen](https://github.com/microsoft/autogen) |
| **项目定位** | Multi-agent 对话与协调框架 |
| **技术栈** | Python, .NET (AutoGen.NET) |
| **体量与复杂度** | ⭐⭐⭐⭐ 大（v0.4 重构后模块化），multi-agent 功能最全 |
| **为什么适合** | Multi-agent coordination 的最全面实现：agent runtime、message passing、team patterns（RoundRobin / Selector / Swarm）、tool use、code execution sandbox。v0.4 架构分层清晰 |
| **重点阅读模块** | `python/packages/autogen-core/`（agent runtime 核心）、`python/packages/autogen-agentchat/`（agent chat / multi-agent patterns）、`python/packages/autogen-ext/`（extensions: tools, code execution） |
| **建议阶段** | **Phase 4** |
| **学完能力** | 掌握 multi-agent runtime 设计；理解 team patterns；能实现 agent-to-agent communication |
| **主要难点** | 代码量大；v0.4 架构变化大，需读最新版 |
| **分类** | 🟡 Framework 偏重，但 runtime 层值得学习 |

#### 项目 9：Goose (Block)

| 维度 | 内容 |
|------|------|
| **项目名称** | [block/goose](https://github.com/block/goose) |
| **项目定位** | 开发者 AI agent，强调 harness 工程（MCP-native） |
| **技术栈** | Rust（核心）, MCP 原生集成 |
| **体量与复杂度** | ⭐⭐⭐ 中，Rust 增加阅读门槛 |
| **为什么适合** | MCP-native 架构的最佳参考：所有 tool 通过 MCP 提供、extension 体系基于 MCP server。Rust 实现提供了不同视角的 harness 设计：类型安全、性能、并发 |
| **重点阅读模块** | `crates/goose/`（核心 agent loop）、`crates/goose-mcp/`（MCP 集成层）、`crates/goose-server/`（agent server） |
| **建议阶段** | **Phase 3 ~ Phase 4**（如果会 Rust；否则只看架构文档） |
| **学完能力** | 理解 MCP-native agent 架构；理解 Rust 视角的 harness 设计 |
| **主要难点** | Rust 语言门槛 |
| **分类** | 🟢 **偏 Runtime / Harness** |

#### 项目 10：SWE-bench

| 维度 | 内容 |
|------|------|
| **项目名称** | [princeton-nlp/SWE-bench](https://github.com/princeton-nlp/SWE-bench) |
| **项目定位** | 软件工程 agent benchmark |
| **技术栈** | Python, Docker |
| **体量与复杂度** | ⭐⭐ 小~中 |
| **为什么适合** | 理解"如何设计 agent benchmark"的标准参考：task 采集、sandbox 设置、evaluation criteria、运行框架。配合 SWE-agent 理解 eval harness 的完整链路 |
| **重点阅读模块** | `swebench/harness/`（评测 harness 核心）、`swebench/collect/`（task 采集）、`swebench/metrics/`（评分） |
| **建议阶段** | **Phase 3** |
| **学完能力** | 理解 agent benchmark 设计方法论；能设计自己的 eval task suite |
| **主要难点** | 需要理解软件工程任务（git patch、test case）的领域知识 |
| **分类** | 🔵 **偏 Eval** |

### 4.2 项目按阶段推荐汇总

| 阶段      | 推荐项目                                   | 阅读深度   | 重点                           |
| ------- | -------------------------------------- | ------ | ---------------------------- |
| Phase 1 | OpenAI Agents SDK, Pydantic AI         | 精读全部源码 | agent loop、tool、guardrail 基础 |
| Phase 2 | SWE-agent, LangGraph                   | 精读核心模块 | sandbox、ACI、状态图、checkpoint   |
| Phase 3 | Inspect AI, Langfuse, SWE-bench, Goose | 精读目标模块 | eval harness、trace、MCP       |
| Phase 4 | OpenHands, AutoGen                     | 架构级阅读  | 生产级架构、multi-agent            |

---

## 5. 实战项目设计

### 项目路线总览

```
Project 1          Project 2          Project 3          Project 4
最小Agent Loop     Tool+Memory        可靠+可观测        Eval Harness
───────────→      ───────────→      ───────────→       ───────────→
  (Phase 1)         (Phase 2)         (Phase 3)          (Phase 3)

                                    Project 5          Project 6
                                    Sandbox+Guard      Multi-Agent
                                    ───────────→      ───────────→
                                      (Phase 3-4)       (Phase 4)
```

### Project 1：Minimal Agent Loop（最小可运行 Agent 循环）

| 维度 | 内容 |
|------|------|
| **目标** | 从零手写一个 ReAct 模式的 agent execution loop，不依赖任何 agent 框架 |
| **功能范围** | LLM 调用 → 解析 action → 执行 tool → 观察结果 → 循环 / 终止 |
| **技术栈** | Python 3.12+, OpenAI/Anthropic API（选一），无框架依赖 |
| **必做模块** | ① `agent_loop.py`：while 循环 + 状态管理<br>② `llm_client.py`：LLM API 封装（支持 function calling）<br>③ `tools.py`：3~5 个简单 tool（calculator, web_search_mock, file_reader, shell_exec_mock）<br>④ `parser.py`：LLM 输出解析（提取 tool name + args）<br>⑤ 终止条件：max_steps（如 20）、max_time（如 300s）、goal_reached |
| **增强项** | streaming 输出、多 provider 支持、配置化（YAML）、step 日志打印 |
| **交付物** | ✅ 可运行的 agent（至少解决 3 个 demo 任务）<br>✅ Execution loop 状态图文档<br>✅ README：agent 控制流说明 |
| **学完能力** | 理解 agent loop 的本质是"LLM-in-the-loop"；能解释 agent vs chain 的区别；掌握 function calling 协议 |

### Project 2：Tool Orchestration + Memory Agent

| 维度 | 内容 |
|------|------|
| **目标** | 在 Project 1 基础上加入 tool registry/routing、分层 memory、基础 planning |
| **功能范围** | 动态 tool 管理 + context window 管理 + memory 读写 + 基础 plan-then-execute |
| **技术栈** | Python 3.12+, LLM API, chromadb/lancedb（向量库）, MCP SDK |
| **必做模块** | ① `tool_registry.py`：tool 注册 / 发现 / schema 验证<br>② `tool_router.py`：tool 选择策略（LLM 默认 + 规则 override）<br>③ `mcp_client.py`：实现 MCP client，连接至少一个 MCP server<br>④ `memory/working.py`：当前任务状态内存<br>⑤ `memory/episodic.py`：历史交互记忆（向量存储）<br>⑥ `memory/manager.py`：memory retrieval + context assembly<br>⑦ `context_manager.py`：token budget 管理 + 滑动窗口 + 摘要压缩<br>⑧ `planner.py`：plan-then-execute 模式（LLM 生成计划 → 逐步执行 → re-plan on failure） |
| **增强项** | MCP server 自己写一个（如 local filesystem MCP）、semantic memory（知识库检索）、plan visualization |
| **交付物** | ✅ 带完整 tool + memory + planning 的 agent<br>✅ 一个可工作的 MCP server<br>✅ Context window usage 可视化<br>✅ 技术选型文档：memory 方案对比 |
| **学完能力** | 能设计 tool 扩展体系；能实现分层 memory；理解 context window 管理的关键性；掌握 MCP 协议 |

### Project 3：Reliable + Observable Agent

| 维度 | 内容 |
|------|------|
| **目标** | 为 agent 加入完整的可靠性机制和 observability pipeline |
| **功能范围** | retry/fallback/timeout + checkpoint/recovery + structured trace + metrics dashboard |
| **技术栈** | Python 3.12+, OpenTelemetry, Langfuse（或自建）, SQLite（checkpoint 存储） |
| **必做模块** | ① `reliability/retry.py`：exponential backoff + jitter<br>② `reliability/timeout.py`：step-level + tool-level + total timeout<br>③ `reliability/fallback.py`：fallback tool + fallback model<br>④ `reliability/circuit_breaker.py`：连续 N 次失败后短路<br>⑤ `reliability/checkpoint.py`：state 序列化 + 恢复<br>⑥ `trace/tracer.py`：span 生成（agent_run → step → llm_call → tool_call）<br>⑦ `trace/exporter.py`：发送到 Langfuse / OTLP<br>⑧ `trace/logger.py`：structured JSON logging<br>⑨ `metrics/collector.py`：steps_per_task, token_usage, tool_success_rate, latency |
| **增强项** | circuit breaker dashboard、自动 retry 策略调优、cost tracking、anomaly detection |
| **交付物** | ✅ 可靠性增强后的 agent（能从断点恢复）<br>✅ Trace dashboard（可用 Langfuse）<br>✅ 可靠性 checklist 文档<br>✅ 一次"注入故障并观察 agent 恢复"的演练报告 |
| **学完能力** | 能为 agent 构建生产级可靠性机制；能通过 trace 诊断 agent 问题；理解 agent 运维 |

### Project 4：Agent Eval Harness

| 维度 | 内容 |
|------|------|
| **目标** | 构建一套 eval harness，系统化评估 agent 表现 |
| **功能范围** | task suite 定义 + agent runner + scorer + reporter + CI 集成 |
| **技术栈** | Python 3.12+, pytest, Docker（sandbox），CI（GitHub Actions） |
| **必做模块** | ① `eval/task.py`：Task 定义（input, expected_output, grading_rubric, timeout, sandbox_config）<br>② `eval/suite.py`：Task suite 管理（YAML/JSON 定义，分类标签）<br>③ `eval/runner.py`：Agent runner（隔离运行，超时控制）<br>④ `eval/scorer.py`：多种打分器（exact_match, contains, llm_judge, custom_func）<br>⑤ `eval/reporter.py`：生成评估报告（pass rate, score distribution, failure analysis）<br>⑥ `eval/regression.py`：对比两次运行结果，检测 regression<br>⑦ `.github/workflows/eval.yml`：CI 中运行 eval，regression 时 block merge |
| **增强项** | LLM-as-judge 评分、多次运行取均值 + 置信区间、eval 结果可视化 web UI |
| **交付物** | ✅ 一套可复用的 eval harness<br>✅ 至少 20 个 eval task（覆盖 tool use / planning / error recovery）<br>✅ CI pipeline with eval gate<br>✅ 一份 eval 方法论文档 |
| **学完能力** | 能设计 agent eval 体系；能实现 eval-in-CI；理解 agent 质量度量 |

### Project 5：Sandboxed + Guardrailed Agent

| 维度 | 内容 |
|------|------|
| **目标** | 为 agent 加入 sandbox 执行环境和安全 guardrails |
| **功能范围** | Docker sandbox + permission model + input/output guardrails + human-in-the-loop |
| **技术栈** | Python 3.12+, Docker SDK, 可选 E2B |
| **必做模块** | ① `sandbox/docker_sandbox.py`：容器生命周期管理（create/exec/snapshot/restore/destroy）<br>② `sandbox/resource_limits.py`：CPU/memory/network/disk 限制<br>③ `guardrails/input_guard.py`：prompt injection 检测、topic boundary 检查<br>④ `guardrails/output_guard.py`：PII 过滤、harmful content 检测<br>⑤ `guardrails/permission.py`：tool permission model（allow-list + capability scoping）<br>⑥ `guardrails/approval_gate.py`：高风险操作人工审批<br>⑦ `hitl/interface.py`：human-in-the-loop 交互界面（CLI / 简易 web） |
| **增强项** | sandbox filesystem snapshot + diff、network policy（白名单 URL）、audit log |
| **交付物** | ✅ Sandbox 化的 agent（代码执行在容器中）<br>✅ Guardrail pipeline（至少 3 种 guard）<br>✅ HITL 审批界面<br>✅ 安全 threat model 文档 |
| **学完能力** | 能设计 agent 安全体系；理解 sandbox 隔离级别；能实现 permission model |

### Project 6：Multi-Agent Orchestrated System

| 维度 | 内容 |
|------|------|
| **目标** | 构建一个多 agent 协调系统，整合前 5 个项目的所有能力 |
| **功能范围** | supervisor agent + worker agents + shared state + communication protocol + 端到端 eval + monitoring |
| **技术栈** | Python 3.12+, 前面所有技术栈 + Redis/PostgreSQL（shared state）+ FastAPI（agent API） |
| **必做模块** | ① `agents/supervisor.py`：supervisor agent（任务分配 + 进度监控 + 结果整合）<br>② `agents/workers/`：至少 3 种 worker agent（researcher / coder / reviewer）<br>③ `coordination/message_bus.py`：agent 间消息传递<br>④ `coordination/shared_state.py`：共享状态管理（task board + artifacts）<br>⑤ `coordination/lifecycle.py`：agent 生命周期（spawn / monitor / timeout / terminate）<br>⑥ `api/server.py`：FastAPI agent-as-a-service<br>⑦ `deploy/docker-compose.yml`：完整部署栈<br>⑧ 整合前面所有能力：tool + memory + reliability + trace + eval + sandbox + guardrail |
| **增强项** | A2A 协议集成、agent A/B testing 框架、cost dashboard、alerting |
| **交付物** | ✅ 可运行的 multi-agent 系统<br>✅ 架构设计文档（含 ADR + sequence diagrams）<br>✅ 完整 CI/CD pipeline（含 eval gate + canary deploy）<br>✅ Monitoring dashboard<br>✅ 性能报告（latency / cost / success rate） |
| **学完能力** | 能设计并实现生产级 multi-agent 系统；具备完整的 harness engineering 能力 |

---

## 6. 前置知识要求

### 分级要求

| 级别 | 知识领域 | 具体项 | 说明 |
|------|---------|--------|------|
| **必须掌握** | Python | Python 3.10+ 高级特性（async/await、type hints、dataclass、context manager） | agent 开发的主力语言 |
| **必须掌握** | HTTP & API | REST API 设计、HTTP 协议、JSON | LLM API 和 tool 调用的基础 |
| **必须掌握** | LLM 基础 | Chat completion API、token 概念、temperature、prompt 基础、function calling | 不理解 LLM API 就无法写 agent |
| **必须掌握** | Git | 基本 Git 操作、branching | 代码管理和 CI 的基础 |
| **必须掌握** | CLI 操作 | 终端基本操作 | 运行和调试 agent |
| **边学边补** | Docker | 容器基础、Dockerfile、docker-compose | Phase 2 sandbox 需要 |
| **边学边补** | 异步编程 | Python asyncio、并发模式 | 并行 tool 调用需要 |
| **边学边补** | 状态机 / 事件驱动 | FSM 概念、event loop | execution loop 设计需要 |
| **边学边补** | 向量数据库 | embedding 概念、向量检索原理 | memory 模块需要 |
| **边学边补** | OpenTelemetry | 分布式追踪基础概念 | observability 需要 |
| **边学边补** | CI/CD | GitHub Actions 基础 | eval-in-CI 需要 |
| **可后置学习** | Rust | Rust 基础（如果要读 Goose 源码） | 仅 Phase 3-4 可选 |
| **可后置学习** | Kubernetes | 容器编排 | 生产部署可选 |
| **可后置学习** | 分布式系统 | 一致性、分布式锁、消息队列 | multi-agent 高级场景 |
| **可后置学习** | 前端 | React/Web 基础 | 如果要做 trace viewer UI |
| **可后置学习** | 论文阅读 | NLP/AI 论文阅读能力 | 持续跟进前沿 |

---

## 7. 推荐资料

### 7.1 必读文章

| # | 文章 | 作者/来源 | 适合阶段 | 阅读重点 |
|---|------|----------|---------|---------|
| 1 | [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) | Anthropic | Phase 1 | Agent 设计模式分类（workflows vs agents）；augmented LLM 概念；何时用 agent |
| 2 | [The Model Context Protocol](https://modelcontextprotocol.io/introduction) | Anthropic | Phase 1-2 | MCP 协议规范：resource / tool / prompt / sampling；server/client 架构 |
| 3 | [What is Context Engineering?](https://www.oreilly.com/radar/what-is-context-engineering/) | O'Reilly | Phase 1 | 理解 context engineering 与 prompt engineering 的区别；与 harness 的关系 |
| 4 | [SWE-agent: Agent-Computer Interfaces](https://arxiv.org/abs/2405.15793) | Princeton | Phase 2 | ACI 设计原则；tool/env 设计如何影响 agent 性能 |
| 5 | [OpenHands Architecture](https://docs.all-hands.dev/modules/usage/architecture) | All-Hands-AI | Phase 3-4 | 生产级 agent 平台架构：event system、runtime 抽象、controller |
| 6 | [Practices for Governing Agentic AI Systems](https://openai.com/index/practices-for-governing-agentic-ai-systems/) | OpenAI | Phase 3 | Agent 治理框架：safety、oversight、auditability |
| 7 | [The Shift from Models to Compound AI Systems](https://bair.berkeley.edu/blog/2024/02/18/compound-ai-systems/) | Berkeley AI Research | Phase 1 | 理解"复合 AI 系统"的概念，agent 是其中一种形态 |

### 7.2 官方文档

| # | 文档 | 适合阶段 | 阅读重点 |
|---|------|---------|---------|
| 1 | [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling) | Phase 1 | tool calling 协议、parallel function calling |
| 2 | [Anthropic Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview) | Phase 1 | tool_use 协议、tool_result、错误处理 |
| 3 | [MCP Specification](https://spec.modelcontextprotocol.io/) | Phase 2 | 完整协议 spec：transport、lifecycle、capabilities |
| 4 | [LangGraph Docs](https://langchain-ai.github.io/langgraph/) | Phase 2 | 状态图、checkpoint、human-in-the-loop |
| 5 | [OpenTelemetry Python Docs](https://opentelemetry.io/docs/languages/python/) | Phase 3 | trace/span/exporter 概念和用法 |
| 6 | [Inspect AI Docs](https://inspect.ai-safety-institute.org.uk/) | Phase 3 | eval 方法论、task/solver/scorer 模式 |
| 7 | [Docker SDK for Python](https://docker-py.readthedocs.io/) | Phase 2-3 | 容器生命周期管理 API |

### 7.3 开源仓库（补充第 4 节未覆盖的）

| # | 仓库 | 适合阶段 | 用途 |
|---|------|---------|------|
| 1 | [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) | Phase 2 | 官方 MCP server 参考实现集合 |
| 2 | [braintrustdata/braintrust](https://github.com/braintrustdata/braintrust-sdk) | Phase 3 | 另一种 eval + observability 方案 |
| 3 | [e2b-dev/e2b](https://github.com/e2b-dev/e2b) | Phase 3 | 云端 sandbox 方案 |
| 4 | [guardrails-ai/guardrails](https://github.com/guardrails-ai/guardrails) | Phase 3 | Agent / LLM output validation 框架 |
| 5 | [google/A2A](https://github.com/google/A2A) | Phase 4 | Agent-to-Agent 协议参考 |
| 6 | [openai/swarm](https://github.com/openai/swarm) | Phase 2 | 轻量级 multi-agent handoff 参考（教育用途） |

### 7.4 视频 / 教程

| # | 资源 | 适合阶段 | 阅读重点 |
|---|------|---------|---------|
| 1 | [Anthropic Prompt Engineering Interactive Tutorial](https://github.com/anthropics/courses/tree/master/prompt_engineering_interactive_tutorial) | Phase 1 | Prompt 基础（harness 工程师也需要） |
| 2 | [LangChain Academy - LangGraph Course](https://academy.langchain.com/courses/intro-to-langgraph) | Phase 2 | 状态图编排 agent 实操 |
| 3 | [DeepLearning.AI - AI Agents in LangGraph](https://www.deeplearning.ai/short-courses/ai-agents-in-langgraph/) | Phase 2 | Agent 模式入门短课 |
| 4 | [Andrej Karpathy - Let's build GPT](https://www.youtube.com/watch?v=kCc8FmEb1nY) | 前置 | 理解 Transformer 原理（可选但推荐） |

### 7.5 论文 / 综述

| # | 论文 | 适合阶段 | 阅读重点 |
|---|------|---------|---------|
| 1 | [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) | Phase 1 | ReAct 模式的原始论文；理解 think-act-observe 循环 |
| 2 | [Toolformer: Language Models Can Teach Themselves to Use Tools](https://arxiv.org/abs/2302.04761) | Phase 1 | 理解 LLM tool use 的原理 |
| 3 | [SWE-bench: Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770) | Phase 3 | Agent benchmark 设计方法论 |
| 4 | [A Survey on Large Language Model based Autonomous Agents](https://arxiv.org/abs/2308.11432) | Phase 1 | Agent 全景综述：architecture、planning、memory、tool |
| 5 | [Agent-as-a-Judge: Evaluate Agents with Agents](https://arxiv.org/abs/2410.10934) | Phase 3 | 用 agent 评估 agent 的方法论 |
| 6 | [τ-bench: A Benchmark for Tool-Agent-User Interaction](https://arxiv.org/abs/2406.12045) | Phase 3 | 多轮 tool-agent-user 交互的 benchmark |
| 7 | [Voyager: An Open-Ended Embodied Agent](https://arxiv.org/abs/2305.16291) | Phase 2 | 自动 skill 库构建、curriculum learning（与 procedural memory 相关） |

---

## 8. 8~12 周学习计划表

### Week 1：Agent 基础 + LLM API

| 维度 | 内容 |
|------|------|
| **学习目标** | 理解 Agent 定义；掌握 LLM chat completion + function calling API |
| **重点资料** | Anthropic "Building effective agents"、OpenAI function calling 文档、ReAct 论文 |
| **代码实践** | 用 raw API 实现 3 个 function calling demo（不用框架） |
| **周产出物** | ✅ function calling 代码 demo × 3<br>✅ 笔记：agent vs chain vs pipeline 区别 |

### Week 2：手写最小 Agent Loop

| 维度 | 内容 |
|------|------|
| **学习目标** | 实现完整的 ReAct execution loop；理解 loop 终止条件 |
| **重点资料** | OpenAI Agents SDK 源码（精读 agent_loop + tool） |
| **代码实践** | 完成 **Project 1**（Minimal Agent Loop） |
| **周产出物** | ✅ Project 1 完成<br>✅ execution loop 状态图 |

### Week 3：Tool System + MCP 入门

| 维度 | 内容 |
|------|------|
| **学习目标** | 实现 tool registry；理解 MCP 协议基础；阅读 Pydantic AI 源码 |
| **重点资料** | MCP 官方文档（Introduction + Architecture）、Pydantic AI 源码 |
| **代码实践** | 实现 Project 2 的 tool_registry + tool_router + 一个 MCP client |
| **周产出物** | ✅ Tool registry 实现<br>✅ MCP client 连接到官方 reference server |

### Week 4：Memory + Context Management

| 维度 | 内容 |
|------|------|
| **学习目标** | 实现分层 memory；掌握 context window management |
| **重点资料** | Voyager 论文（skill library / procedural memory）、chromadb 文档 |
| **代码实践** | 实现 Project 2 的 memory 模块（working + episodic）+ context manager |
| **周产出物** | ✅ Memory 系统（至少 2 层）<br>✅ Context window usage 可视化 |

### Week 5：Planning + SWE-agent 精读

| 维度 | 内容 |
|------|------|
| **学习目标** | 实现 plan-then-execute 模式；精读 SWE-agent 的 harness 设计 |
| **重点资料** | SWE-agent 源码（agent/ + environment/ + tools/）、SWE-agent 论文 |
| **代码实践** | 完成 **Project 2** 全部；精读 SWE-agent 并写架构分析笔记 |
| **周产出物** | ✅ Project 2 完成<br>✅ SWE-agent harness 架构分析笔记 |

### Week 6：Reliability Engineering

| 维度 | 内容 |
|------|------|
| **学习目标** | 实现 retry / fallback / timeout / circuit breaker / checkpoint |
| **重点资料** | LangGraph checkpoint 源码、微服务可靠性 pattern 资料 |
| **代码实践** | 完成 **Project 3** 的 reliability 模块 |
| **周产出物** | ✅ Reliability 模块全部实现<br>✅ 故障注入 + 恢复演练报告 |

### Week 7：Observability + Trace

| 维度 | 内容 |
|------|------|
| **学习目标** | 构建 agent trace pipeline；集成 Langfuse 或自建 trace 存储 |
| **重点资料** | Langfuse Python SDK 源码、OpenTelemetry Python 文档 |
| **代码实践** | 完成 **Project 3** 的 trace + logging + metrics 模块 |
| **周产出物** | ✅ Project 3 完成<br>✅ Trace dashboard 可访问<br>✅ 可靠性 checklist |

### Week 8：Eval Harness

| 维度 | 内容 |
|------|------|
| **学习目标** | 构建完整 eval harness；理解 agent eval 方法论 |
| **重点资料** | Inspect AI 源码（solver/ + scorer/ + log/）、SWE-bench 源码、τ-bench 论文 |
| **代码实践** | 完成 **Project 4**（Eval Harness） |
| **周产出物** | ✅ Project 4 完成（20+ test cases）<br>✅ CI pipeline with eval gate<br>✅ Eval 方法论文档 |

### Week 9：Sandbox + Guardrails + HITL

| 维度 | 内容 |
|------|------|
| **学习目标** | 实现 Docker sandbox；实现 input/output guardrails；实现 human-in-the-loop |
| **重点资料** | Docker SDK for Python、E2B 文档、OpenAI agentic AI governance 文章 |
| **代码实践** | 完成 **Project 5**（Sandbox + Guardrails） |
| **周产出物** | ✅ Project 5 完成<br>✅ 安全 threat model 文档 |

### Week 10：Multi-Agent Patterns

| 维度 | 内容 |
|------|------|
| **学习目标** | 理解 multi-agent 模式；实现 supervisor + worker 协调 |
| **重点资料** | AutoGen v0.4 源码（agentchat/ + core/）、Google A2A 协议、OpenAI Swarm |
| **代码实践** | 开始 **Project 6**：实现 supervisor + 2 worker agents + message bus |
| **周产出物** | ✅ Multi-agent 基础协调运行<br>✅ Agent communication protocol 文档 |

### Week 11：Production Architecture

| 维度 | 内容 |
|------|------|
| **学习目标** | 精读 OpenHands 架构；实现 agent-as-a-service；完善 Project 6 |
| **重点资料** | OpenHands 源码（runtime/ + controller/ + server/）、架构文档 |
| **代码实践** | 完成 Project 6：FastAPI server + docker-compose 部署 + 整合所有能力 |
| **周产出物** | ✅ Project 6 完成<br>✅ 完整架构设计文档<br>✅ Docker-compose 一键部署 |

### Week 12：整合 + 回顾 + 知识地图

| 维度 | 内容 |
|------|------|
| **学习目标** | 整合所有项目；完善文档；输出完整知识地图 |
| **重点资料** | 回顾全部笔记 + 查漏补缺 |
| **代码实践** | 端到端测试 + 性能优化 + CI/CD 完善 |
| **周产出物** | ✅ 完整 Agent Harness Engineering 知识地图<br>✅ 所有项目代码仓库整理（README + 文档）<br>✅ 一篇总结博文：《我如何学习 Agent Harness Engineering》 |

### 周计划速览表

| 周 | 阶段 | 主题 | 关键项目 | 核心产出 |
|----|------|------|---------|---------|
| W1 | Phase 1 | Agent 基础 + LLM API | - | function calling demo × 3 |
| W2 | Phase 1 | 手写 Agent Loop | Project 1 ✅ | 最小 agent loop |
| W3 | Phase 1→2 | Tool + MCP | Project 2 开始 | tool registry + MCP client |
| W4 | Phase 2 | Memory + Context | Project 2 继续 | 分层 memory 系统 |
| W5 | Phase 2 | Planning + 源码研读 | Project 2 ✅ | plan-then-execute + SWE-agent 笔记 |
| W6 | Phase 3 | Reliability | Project 3 开始 | retry/fallback/checkpoint |
| W7 | Phase 3 | Observability | Project 3 ✅ | trace dashboard |
| W8 | Phase 3 | Eval Harness | Project 4 ✅ | eval harness + CI |
| W9 | Phase 3 | Sandbox + Guardrails | Project 5 ✅ | sandbox + security |
| W10 | Phase 4 | Multi-Agent | Project 6 开始 | supervisor + workers |
| W11 | Phase 4 | Production Arch | Project 6 ✅ | agent-as-a-service |
| W12 | Phase 4 | 整合 + 总结 | 全部 | 知识地图 + 博文 |

---

## 9. 架构图集

### 9.1 Agent Harness 总体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        USER / EXTERNAL SYSTEM                           │
│                     (HTTP API / CLI / Web UI / CI)                      │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                    ┌───────────▼───────────────┐
                    │      API Gateway /        │
                    │     Agent Dispatcher      │
                    │  (routing, auth, rate-lim) │
                    └───────────┬───────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────┐
│                          AGENT HARNESS LAYER                            │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │                    EXECUTION ENGINE                             │    │
│   │                                                                 │    │
│   │  ┌──────────────┐    ┌──────────────┐    ┌────────────────┐   │    │
│   │  │  Execution   │    │   Planner    │    │   Context      │   │    │
│   │  │    Loop      │◄──→│ (plan/replan)│◄──→│   Assembler    │   │    │
│   │  │ (state mach) │    └──────────────┘    │ (window mgmt)  │   │    │
│   │  └──────┬───────┘                        └───────┬────────┘   │    │
│   │         │                                        │             │    │
│   │         ▼                                        ▼             │    │
│   │  ┌──────────────┐                       ┌────────────────┐    │    │
│   │  │    Step      │                       │    Memory      │    │    │
│   │  │  Executor    │                       │   Manager      │    │    │
│   │  │ (LLM/tool/   │                       │ (W/E/S/P)     │    │    │
│   │  │  human step) │                       └────────────────┘    │    │
│   │  └──────────────┘                                             │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │                TOOL ORCHESTRATION LAYER                        │    │
│   │                                                                 │    │
│   │  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌─────────────┐  │    │
│   │  │  Tool    │  │   Tool    │  │   MCP    │  │   Tool      │  │    │
│   │  │ Registry │  │  Router   │  │  Client  │  │  Executor   │  │    │
│   │  └──────────┘  └───────────┘  └──────────┘  └─────────────┘  │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │                 GOVERNANCE LAYER                                │    │
│   │                                                                 │    │
│   │  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌─────────────┐  │    │
│   │  │Guardrails│  │   HITL    │  │Permission│  │  Cost       │  │    │
│   │  │ Pipeline │  │   Gate    │  │  Engine  │  │  Controller │  │    │
│   │  └──────────┘  └───────────┘  └──────────┘  └─────────────┘  │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │                RELIABILITY LAYER                               │    │
│   │                                                                 │    │
│   │  ┌──────┐  ┌────────┐  ┌──────────┐  ┌──────────┐  ┌──────┐ │    │
│   │  │Retry │  │Timeout │  │Fallback  │  │Circuit   │  │Check │ │    │
│   │  │      │  │        │  │          │  │Breaker   │  │point │ │    │
│   │  └──────┘  └────────┘  └──────────┘  └──────────┘  └──────┘ │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │              OBSERVABILITY LAYER                               │    │
│   │                                                                 │    │
│   │  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌─────────────┐  │    │
│   │  │  Tracer  │  │ Structured│  │ Metrics  │  │  Eval       │  │    │
│   │  │(span/log)│  │  Logger   │  │Collector │  │  Harness    │  │    │
│   │  └──────────┘  └───────────┘  └──────────┘  └─────────────┘  │    │
│   └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└───────────────────────┬──────────────────────────┬──────────────────────┘
                        │                          │
          ┌─────────────▼──────────┐   ┌──────────▼──────────────┐
          │      SANDBOX LAYER     │   │    EXTERNAL SERVICES    │
          │                        │   │                          │
          │  ┌──────────────────┐  │   │  ┌────────┐ ┌────────┐ │
          │  │  Docker / microVM│  │   │  │  LLM   │ │  MCP   │ │
          │  │  / E2B / gVisor  │  │   │  │  APIs  │ │Servers │ │
          │  └──────────────────┘  │   │  └────────┘ └────────┘ │
          │  ┌──────────────────┐  │   │  ┌────────┐ ┌────────┐ │
          │  │  Resource Limits │  │   │  │ Vector │ │  REST  │ │
          │  │  (CPU/mem/net)   │  │   │  │   DB   │ │  APIs  │ │
          │  └──────────────────┘  │   │  └────────┘ └────────┘ │
          └────────────────────────┘   └──────────────────────────┘
```

### 9.2 Agent Step 生命周期（单步详细流程）

```
                    ┌─────────────────────┐
                    │    New Step Start   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Pre-Step Hooks     │
                    │  ├─ Input guard     │
                    │  ├─ Budget check    │
                    │  ├─ Step count check│
                    │  └─ Timeout check   │
                    └──────────┬──────────┘
                               │
                         Pass? │ ──No──→ TERMINATE / HITL
                               │Yes
                    ┌──────────▼──────────┐
                    │  Context Assembly   │
                    │  ├─ System prompt   │
                    │  ├─ Memory retrieve │
                    │  ├─ Tool schemas    │
                    │  ├─ Recent history  │
                    │  └─ Token budget    │
                    │      enforcement    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  LLM Call           │
                    │  ├─ timeout: 60s    │
                    │  ├─ retry: 3x       │
                    │  ├─ trace: span     │
                    │  └─ cost: track     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Response Parse     │
                    │  ├─ Extract action  │
                    │  ├─ Validate schema │
                    │  └─ Handle parse err│
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Action Dispatch    │
                    │  ├─ tool_call?      │──→ Tool Execution Pipeline
                    │  ├─ final_answer?   │──→ Return Result
                    │  └─ invalid?        │──→ Error Recovery
                    └──────────┬──────────┘
                               │ (tool_call)
                    ┌──────────▼──────────┐
                    │  Tool Execution     │
                    │  Pipeline           │
                    │  ├─ Permission check│
                    │  ├─ HITL gate       │
                    │  │   (if high-risk) │
                    │  ├─ Sandbox exec    │
                    │  ├─ Timeout: 30s    │
                    │  ├─ Retry: 2x       │
                    │  ├─ Fallback tool   │
                    │  └─ Trace: span     │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Post-Step Hooks    │
                    │  ├─ Output guard    │
                    │  ├─ Result validate │
                    │  ├─ Memory write    │
                    │  ├─ Checkpoint save │
                    │  ├─ Trace complete  │
                    │  └─ Metrics emit    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Loop Decision     │
                    │  ├─ Goal reached?   │──→ COMPLETE
                    │  ├─ Error budget?   │──→ FAILED
                    │  ├─ Max steps?      │──→ TERMINATED
                    │  └─ Continue        │──→ Next Step
                    └─────────────────────┘
```

### 9.3 Multi-Agent Supervisor 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      USER REQUEST                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
              ┌───────────▼───────────┐
              │   SUPERVISOR AGENT    │
              │                       │
              │  ┌─────────────────┐  │
              │  │ Task Planner    │  │  - 分解任务
              │  └────────┬────────┘  │  - 分配给 worker
              │  ┌────────▼────────┐  │  - 监控进度
              │  │ Task Allocator  │  │  - 整合结果
              │  └────────┬────────┘  │  - 处理冲突
              │  ┌────────▼────────┐  │
              │  │ Result Merger   │  │
              │  └─────────────────┘  │
              └───┬────────┬────────┬─┘
                  │        │        │
        ┌─────────▼──┐ ┌──▼──────┐ ┌▼───────────┐
        │  Worker A  │ │Worker B │ │  Worker C  │
        │ (Researcher)│ │ (Coder) │ │ (Reviewer) │
        │            │ │         │ │            │
        │  Own loop  │ │Own loop │ │  Own loop  │
        │  Own tools │ │Own tools│ │  Own tools │
        │  Own memory│ │Own mem  │ │  Own memory│
        └─────┬──────┘ └───┬─────┘ └─────┬──────┘
              │            │              │
              └────────────┼──────────────┘
                           │
              ┌────────────▼────────────┐
              │    SHARED STATE         │
              │  ┌───────────────────┐  │
              │  │  Task Board       │  │
              │  │  (status, deps)   │  │
              │  ├───────────────────┤  │
              │  │  Artifact Store   │  │
              │  │  (code, docs)     │  │
              │  ├───────────────────┤  │
              │  │  Message Log      │  │
              │  │  (agent comms)    │  │
              │  └───────────────────┘  │
              └─────────────────────────┘
```

### 9.4 从 Demo 到 Production 的能力层次图

```
Level 5 ┌─────────────────────────────────────────────┐
        │          PRODUCTION EXCELLENCE               │
        │  Multi-agent · CI/CD · A/B testing ·        │
        │  Auto-scaling · Cost optimization ·          │
        │  Full monitoring + alerting                  │
Level 4 ├─────────────────────────────────────────────┤
        │           GOVERNANCE                         │
        │  Sandbox · Guardrails · Permission ·         │
        │  HITL · Audit log · Policy engine            │
Level 3 ├─────────────────────────────────────────────┤
        │       QUALITY + OBSERVABILITY                │
        │  Trace · Metrics · Eval harness ·            │
        │  Regression testing · Structured logging     │
Level 2 ├─────────────────────────────────────────────┤
        │          RELIABILITY                          │
        │  Retry · Fallback · Timeout · Checkpoint ·   │
        │  Circuit breaker · Error recovery            │
Level 1 ├─────────────────────────────────────────────┤
        │        FUNCTIONAL CORE                        │
        │  Execution loop · Tool calling · Memory ·    │
        │  Context management · Planning · MCP         │
Level 0 ├─────────────────────────────────────────────┤
        │           DEMO / POC                          │
        │  LLM API call · Basic prompt · No control    │
        └─────────────────────────────────────────────┘

  Phase 1 ─→ Level 0 → 1
  Phase 2 ─→ Level 1 (完善)
  Phase 3 ─→ Level 2 → 3 → 4
  Phase 4 ─→ Level 5
```

---

## 附录 A：术语表

| 术语 | 英文 | 定义 |
|------|------|------|
| Agent Harness | Agent Harness | AI Agent 的运行时控制层，负责执行循环、工具编排、记忆管理、可靠性、可观测性等 |
| Execution Loop | Execution Loop | Agent 的核心循环：think → act → observe → repeat |
| ACI | Agent-Computer Interface | SWE-agent 提出的概念，Agent 与环境交互的接口设计 |
| MCP | Model Context Protocol | Anthropic 提出的标准协议，用于 LLM/Agent 与外部工具/资源的交互 |
| A2A | Agent-to-Agent Protocol | Google 提出的 agent 间通信协议 |
| HITL | Human-in-the-Loop | 在 agent 决策流程中引入人类审批/干预 |
| Eval Harness | Evaluation Harness | 系统化评估 agent 表现的框架 |
| Circuit Breaker | Circuit Breaker | 连续失败后短路，避免继续浪费资源的机制 |
| Checkpoint | Checkpoint | 保存 agent 运行中间状态，支持断点恢复 |
| Guardrails | Guardrails | Agent 输入/输出的安全检查机制 |

## 附录 B：能力自测清单

完成本方案后，应能回答以下问题：

- [ ] 能不依赖框架，手写一个完整的 agent execution loop
- [ ] 能解释 harness engineering 与 prompt engineering / context engineering 的区别
- [ ] 能设计一个支持动态注册和 MCP 的 tool orchestration 系统
- [ ] 能实现分层 memory，并解释何时用哪种 memory
- [ ] 能实现 checkpoint-based recovery
- [ ] 能为 agent 构建完整的 trace pipeline
- [ ] 能从零搭建 eval harness 并运行 benchmark
- [ ] 能实现 Docker-based sandbox
- [ ] 能设计 agent permission model + guardrail pipeline
- [ ] 能实现 supervisor-worker multi-agent 协调
- [ ] 能设计 agent-as-a-service API + CI/CD pipeline
- [ ] 能画出完整的 agent harness 架构图并解释每个组件

---

> **本方案是起点，不是终点。** Agent Harness Engineering 是一个快速演进的工程领域，保持对前沿项目（SWE-agent、OpenHands、Claude Code、Devin 等）的持续跟踪是长期成长的关键。
