# Week 3 学习指南：Agent 工程化 — 配置、日志与流式输出

> 本文档系统讲解 `Phase 1/week3` 项目在 Week 2 基础上的三项工程化升级：YAML 配置管理、步骤日志系统、流式输出支持。适合已完成 Week 2、准备进入 Phase 2 的开发者。

---

## 目录

1. [项目定位](#1-项目定位)
2. [Week 2 → Week 3：演进总览](#2-week-2--week-3演进总览)
3. [总架构图](#3-总架构图)
4. [逐模块精讲](#4-逐模块精讲)
   - [4.1 YAML 配置管理 `config.py`](#41-yaml-配置管理-configpy--核心变更)
   - [4.2 步骤日志系统 `agent/logger.py`](#42-步骤日志系统-agentloggerpy--新增)
   - [4.3 流式输出 `llm/client.py`](#43-流式输出-llmclientpy--新增流式能力)
   - [4.4 程序入口重构 `main.py`](#44-程序入口重构-mainpy--重构)
   - [4.5 执行引擎升级 `agent/loop.py`](#45-执行引擎升级-agentlooppy--升级)
   - [4.6 响应解析 `llm/parser.py`](#46-响应解析-llmparserpy--微调)
   - [4.7 工具层保持不变](#47-工具层保持不变)
5. [关键设计决策](#5-关键设计决策)
6. [完整数据流：一次流式执行的全程](#6-完整数据流一次流式执行的全程)
7. [Week 2 vs Week 3 对比表](#7-week-2-vs-week-3-对比表)
8. [动手实践建议](#8-动手实践建议)
9. [与后续 Week 的关联](#9-与后续-week-的关联)

---

## 1. 项目定位

Week 3 是 Phase 1 的**收官周**。核心任务不是写新的 agent 能力，而是为 Week 2 的 Minimal Agent Loop 加上三项**工程化基础设施**：

| 升级项    | Week 2 状态                         | Week 3 目标                    |
| ------ | --------------------------------- | ---------------------------- |
| 配置管理   | 硬编码在 `AgentConfig.from_env()`     | YAML 文件驱动，修改配置无需改代码          |
| 运行日志   | `print(f"[Step {n}] action=...")` | 结构化 `StepLogger`，含耗时、token 数 |
| LLM 输出 | 仅 `chat()` 同步调用                   | 新增 `stream_chat()` 流式输出      |
|        |                                   |                              |

**核心理念**：Week 2 证明了 agent loop **能跑**；Week 3 让它**可观测、可配置、可演示**。这三样东西是从 demo 走向工程的第 0 步。

学习价值：理解 agent 框架不只是执行循环——**配置加载、日志追踪、流式渲染**同样是 harness 的一部分。

---

## 2. Week 2 → Week 3：演进总览

```
Week 2                           Week 3
═══════                           ═══════
AgentConfig.from_env()    ──→    AgentConfig.from_yaml("config.yaml")
print(f"[Step {n}] ...")  ──→    StepLogger.log(StepLog(...))
LLMClient.chat() only     ──→    LLMClient.chat() + LLMClient.stream_chat()
单一 main() 入口           ──→    argparse + 4 个 demo + streaming 独立路径
无文档                     ──→    architecture / execution_loop / harness_components / phase1_summary
```

**Week 3 刻意保持不变的**：
- 5 个工具实现零改动
- ReAct 循环核心逻辑不变
- 终止条件（max_steps / max_time / goal_reached）不变
- 错误处理层不变

这样设计的意图：**让 Week 3 的变化聚焦于 harness 结构本身，而非业务逻辑**。

---

## 3. 总架构图

### 目录结构

```text
week3/
├── README.md
├── config.yaml                  ← 新增：YAML 运行时配置
├── main.py                      ← 重构：argparse + streaming 入口
├── data/
│   └── sample_note.txt
├── demos/                       ← 新增：4 个独立 demo 脚本
│   ├── demo_math.py
│   ├── demo_weather.py
│   ├── demo_file.py
│   └── demo_stream.py           ← 新增：流式输出 demo
├── docs/                        ← 新增：项目文档
│   ├── architecture.md
│   ├── execution_loop.md
│   ├── harness_components.md
│   ├── phase1_summary.md
│   └── learning_guide.md        ← 本文档
└── minimal_agent/
    ├── __init__.py
    ├── config.py                ← 重构：新增 from_yaml() + 手写 YAML 解析
    ├── agent/
    │   ├── __init__.py
    │   ├── loop.py              ← 升级：StepLogger 集成 + force_first_tool
    │   ├── logger.py            ← 新增：结构化步骤日志
    │   └── state.py             ← 微调：TokenUsage 集成
    ├── llm/
    │   ├── __init__.py
    │   ├── client.py            ← 升级：新增 stream_chat() + TokenUsage 解析
    │   └── parser.py            ← 微调：更精准的 JSON decode 错误信息
    └── tools/                   ← 无变化（5 个工具与 Week 2 完全一致）
        ├── __init__.py
        ├── base.py
        ├── calculator.py
        ├── file_reader.py
        ├── search.py
        ├── shell.py
        └── weather.py
```

### 模块依赖关系图

```
main.py
  ├─→ AgentConfig.from_yaml("config.yaml")  ──→ config.py ──→ .env + config.yaml
  ├─→ build_default_tools()                  ──→ tools/__init__.py
  ├─→ AgentLoop(config, tools)               ──→ agent/loop.py
  │     ├─→ AgentState                       ──→ agent/state.py
  │     ├─→ LLMClient                        ──→ llm/client.py ──→ OpenAI API
  │     │     ├─→ chat()          (同步，含 tool calling)
  │     │     └─→ stream_chat()   (流式，纯文本)    ← 新增
  │     ├─→ parse_response()                 ──→ llm/parser.py
  │     ├─→ StepLogger                       ──→ agent/logger.py  ← 新增
  │     ├─→ tool_registry[name].run(args)    ──→ tools/base.py
  │     └─→ AgentRunResult                   ──→ agent/state.py
  └─→ run_streaming_demo(config, prompt)     ──→ LLMClient.stream_chat()  ← 新增
        └─→ _strip_streaming_think_blocks()  ──→ 流式思考块过滤器
```

---

## 4. 逐模块精讲

### 4.1 YAML 配置管理 `config.py` — 核心变更

**文件**：`minimal_agent/config.py`

Week 2 的配置全部硬编码在 `AgentConfig.from_env()` 的默认参数中。Week 3 新增了 **YAML 文件驱动 + 手写解析器**，实现"改配置不改代码"。

#### 配置加载链

```
.env (项目根 / Phase 1 目录)
  ↓ 提供：api_key, base_url, model
config.yaml (Phase 1/week3/config.yaml)
  ↓ 提供：max_steps, max_time_seconds, llm_timeout_seconds, llm_max_retries, stream_temperature, system_prompt
AgentConfig.from_yaml(path)
  ↓ 合并以上两层，生成完整配置
```

#### from_yaml 方法

```python
@classmethod
def from_yaml(cls, path: Path | str = CONFIG_FILE) -> "AgentConfig":
    env = EnvSettings()        # ① 加载 .env 中的凭证
    data = _load_simple_yaml(Path(path))  # ② 解析 YAML 文件
    return cls(
        api_key=env.api_key,           # 来自 .env（安全敏感，不入 YAML）
        base_url=env.base_url,         # 来自 .env
        model=str(data.get("model") or env.model),  # YAML 优先，回退 .env
        system_prompt=str(data.get("system_prompt", "默认 prompt...")),
        max_steps=int(data.get("max_steps", 20)),
        max_time_seconds=int(data.get("max_time_seconds", 300)),
        llm_timeout_seconds=int(data.get("llm_timeout_seconds", 60)),
        llm_max_retries=int(data.get("llm_max_retries", 2)),
        stream_temperature=float(data.get("stream_temperature", 0.2)),
    )
```

**关键设计点**：
- **凭证与配置分离**：`api_key` / `base_url` 只从 `.env` 读取，绝不写入 YAML。YAML 仅管运行时行为参数
- **YAML 优先，.env 兜底**：`model` 字段如果 YAML 为空字符串 `""`，则回退到 `.env` 的值
- **每个参数都有默认值**：YAML 文件缺失或字段缺失都不会导致崩溃

#### 手写 YAML 解析器 `_load_simple_yaml()`

Week 3 刻意**不引入 PyYAML 依赖**，而是手写了一个 ~50 行的微型 YAML 解析器。这不是"重新造轮子"——它是一个**精心设计的教学点**：

```python
def _load_simple_yaml(path: Path) -> dict[str, Any]:
    if not path.exists():
        return {}                         # 文件不存在 → 空字典，全部用默认值

    data: dict[str, Any] = {}
    lines = path.read_text(encoding="utf-8").splitlines()
    index = 0
    while index < len(lines):
        raw_line = lines[index]
        stripped = raw_line.strip()
        index += 1
        if not stripped or stripped.startswith("#"):
            continue                      # 跳过空行和注释

        if ": |" in raw_line:             # ① 多行字符串（literal block）
            key = raw_line.split(":", 1)[0].strip()
            block_lines: list[str] = []
            while index < len(lines):
                candidate = lines[index]
                if candidate and not candidate.startswith((" ", "\t")):
                    break                 # 缩进结束 → 块结束
                block_lines.append(candidate.strip())
                index += 1
            data[key] = " ".join(line for line in block_lines if line)
            continue

        if ":" not in raw_line:
            continue                      # 无效行

        key, value = raw_line.split(":", 1)
        data[key.strip()] = _parse_scalar(value.strip())

    return data
```

**解析器支持的语法子集**：

| YAML 语法 | 示例                        | 解析结果                          |
| ------- | ------------------------- | ----------------------------- |
| 简单键值    | `max_steps: 20`           | `{"max_steps": 20}`           |
| 布尔值     | `llm_max_retries: 2`      | `{"llm_max_retries": 2}`      |
| 浮点数     | `stream_temperature: 0.2` | `{"stream_temperature": 0.2}` |
| 空字符串    | `model: ""`               | `{"model": ""}`               |
| 字面量块    | `system_prompt: \|`       | 多行合并为单行                       |
| 注释      | `# 这是注释`                  | 忽略                            |

**类型推断 `_parse_scalar()`**：

```python
def _parse_scalar(value: str) -> Any:
    value = value.strip().strip("\"'")
    if value.lower() in {"true", "false"}:
        return value.lower() == "true"     # "true" → True, "false" → False
    try:
        return int(value)                  # "20" → 20
    except ValueError:
        pass
    try:
        return float(value)                # "0.2" → 0.2
    except ValueError:
        return value                       # "hello" → "hello"
```

**设计决策：为什么不直接用 PyYAML？**

| 维度 | PyYAML | 手写解析器 |
|------|--------|----------|
| 依赖 | 需要 `pip install pyyaml` | 无额外依赖 |
| 安全性 | 有 `yaml.safe_load()` | 天然安全（只解析简单结构） |
| 代码量 | 1 行 `yaml.safe_load(f)` | ~50 行 |
| 学习价值 | 低（黑盒） | 高（理解 YAML 解析原理） |
| 适用性 | 通用 YAML | 专为 Week 3 的扁平配置设计 |

在 Phase 1 这个阶段，**理解配置加载的原理**比用现成库更重要。Phase 2+ 可以替换为 `pyyaml` 或 `toml`。

---

### 4.2 步骤日志系统 `agent/logger.py` — 新增

**文件**：`minimal_agent/agent/logger.py`

Week 2 的日志是一个简单的 `print(f"[Step {n}] action=... detail=...")`。Week 3 将其升级为结构化的 `StepLogger` 系统。

#### 数据模型

```python
@dataclass(slots=True)
class StepLog:
    step_number: int              # 第几步
    action_type: ActionType       # TOOL_CALLS | FINAL_ANSWER | PARSE_ERROR
    tool_name: str                # 工具名（多个用逗号分隔）
    elapsed_seconds: float        # 从 run() 开始到当前的耗时
    token_usage: TokenUsage       # 本次 LLM 调用的 token 消耗
    detail: str                   # 附加信息（工具参数摘要 / 错误信息）
```

#### 日志输出器

```python
class StepLogger:
    def log(self, entry: StepLog) -> None:
        print(
            f"[Step {entry.step_number:02d}] "
            f"action={entry.action_type.value} | "
            f"tool={entry.tool_name or '-'} | "
            f"elapsed={entry.elapsed_seconds:.2f}s | "
            f"tokens={entry.token_usage.total_tokens} "
            f"(prompt={entry.token_usage.prompt_tokens}, "
            f"completion={entry.token_usage.completion_tokens}) | "
            f"{entry.detail}"
        )
```

**典型输出示例**：

```
[Step 01] action=tool_calls | tool=calculator | elapsed=1.23s | tokens=245 (prompt=200, completion=45) | tool_calls=calculator
[Step 02] action=tool_calls | tool=calculator | elapsed=2.67s | tokens=310 (prompt=260, completion=50) | tool_calls=calculator
[Step 03] action=final_answer | tool=- | elapsed=4.12s | tokens=350 (prompt=320, completion=30) | final_answer
```

#### Token 使用量追踪

Week 3 新增了 `TokenUsage` 数据类：

```python
@dataclass(slots=True)
class TokenUsage:
    prompt_tokens: int = 0        # 输入 token 数
    completion_tokens: int = 0    # 输出 token 数
    total_tokens: int = 0         # 总 token 数
```

在 `LLMClient.chat()` 中，`usage` 对象从 API 响应中提取：

```python
def _parse_usage(self, usage: Any | None) -> TokenUsage:
    if usage is None:
        return TokenUsage()
    return TokenUsage(
        prompt_tokens=int(getattr(usage, "prompt_tokens", 0) or 0),
        completion_tokens=int(getattr(usage, "completion_tokens", 0) or 0),
        total_tokens=int(getattr(usage, "total_tokens", 0) or 0),
    )
```

**设计意图**：每步打印 token 消耗，让开发者直观感受到 agent 的成本累积。这是从 demo 到 production 的必备意识。

---

### 4.3 流式输出 `llm/client.py` — 新增流式能力

**文件**：`minimal_agent/llm/client.py`

Week 2 只有同步 `chat()` 方法。Week 3 新增 `stream_chat()` 方法，返回一个 **Python 生成器**。

```python
def stream_chat(self, messages: list[dict[str, Any]]) -> Iterator[str]:
    """Stream plain assistant text for Week 3 streaming practice."""
    stream = self._client.chat.completions.create(
        model=self._config.model,
        messages=messages,
        temperature=self._config.stream_temperature,
        stream=True,                      # 关键：启用流式
    )
    for chunk in stream:
        delta = chunk.choices[0].delta
        content = getattr(delta, "content", None)
        if content:
            yield content                 # 逐块产出文本
```

**关键设计点**：
- **`stream=True`**：告诉 OpenAI API 使用 Server-Sent Events 模式，逐 token 返回
- **`Iterator[str]`**：生成器模式，每 yield 一次就是一小段文本（可能几个 token）
- **仅用于纯文本场景**：`stream_chat()` 不传 `tools` 参数，不涉及 tool calling
- **独立的温度参数**：`stream_temperature: 0.2`，流式输出用较低温度以获得更连贯的文本

**为什么流式调用不包含 tool calling？**

Week 3 刻意将流式输出与 tool calling 分离。流式 tool calling 涉及：
- 增量解析 JSON 参数（需要流式 JSON parser）
- tool call 结果的中途处理
- 状态一致性管理

这些是 Phase 3 的可靠性工程内容。Week 3 的 `stream_chat()` 专注于演示 provider 流式传输的基础机制。

#### 思考块过滤器 `_strip_streaming_think_blocks()`

某些 LLM 提供商会将推理过程包裹在 `<think>...</think>` 标签中。这个函数在流式渲染时**实时过滤**这些标签：

```python
def _strip_streaming_think_blocks(
    chunk: str,
    state: dict[str, object],
    flush: bool = False,
) -> str:
    state["buffer"] = str(state.get("buffer", "")) + chunk
    buffer = str(state["buffer"])
    output: list[str] = []

    while buffer:
        if state.get("in_think"):           # 当前在思考块内部
            end = buffer.find("</think>")
            if end == -1:
                buffer = buffer[-7:] if not flush else ""  # 保留尾部防截断
                break
            buffer = buffer[end + len("</think>"):]        # 跳过思考块
            state["in_think"] = False
            continue

        start = buffer.find("<think>")
        if start == -1:                     # 没有思考块标记
            keep = 6 if not flush else 0    # 保留尾部 6 字符防截断
            if len(buffer) <= keep:
                break
            output.append(buffer[:-keep] if keep else buffer)
            buffer = buffer[-keep:] if keep else ""
            break

        output.append(buffer[:start])       # 输出 `<think>` 之前的内容
        buffer = buffer[start + len("<think>"):]
        state["in_think"] = True

    state["buffer"] = buffer
    return "".join(output)
```

**为什么需要这么复杂的状态机？**

流式传输中，`<think>` 标签可能被切分成多个 chunk：

```
chunk 1: "开头文字<thi"
chunk 2: "nk>内部推理..."
chunk 3: "...</think>结尾文字"
```

过滤器维护 `in_think` 状态 + 6 字符的尾部保留缓冲区，确保不会因分片而漏判或误判。

---

### 4.4 程序入口重构 `main.py` — 重构

**文件**：`main.py`

Week 2 的 `main.py` 只有简单的 CLI 选择和 3 个硬编码 demo。Week 3 做了完整的重构：

#### argparse 命令行接口

```python
def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Run the Week 3 configurable minimal agent loop demos.",
    )
    parser.add_argument(
        "demo",
        choices=["math", "weather", "file", "stream"],  # 新增 "stream"
        help="Which demo to run.",
    )
    parser.add_argument(
        "--config",
        default=str(Path(__file__).resolve().parent / "config.yaml"),
        help="Path to the Week 3 YAML config file.",
    )
    return parser.parse_args()
```

**变更点**：
- 支持 4 个 demo（新增 `stream`）
- `--config` 参数允许指定自定义配置文件路径
- 默认配置文件路径使用 `Path(__file__).resolve().parent` 动态计算，不依赖工作目录

#### 流式 demo 独立路径

```python
def main() -> None:
    args = parse_args()
    config = AgentConfig.from_yaml(args.config)
    if args.demo == "stream":
        run_streaming_demo(config, build_demo_prompt(args.demo))
        return                          # 流式 demo 走独立路径

    # 同步 demo 走 AgentLoop 路径
    agent = AgentLoop(
        config=config,
        tools=build_default_tools(),
        force_first_tool="calculator" if args.demo == "math" else None,
    )
    ...
```

**设计意图**：流式 demo**不使用 AgentLoop**，直接调用 `LLMClient.stream_chat()`。因为 Week 3 的流式输出是纯文本生成，不涉及 tool calling 循环。

#### 四个 Demo 的 prompt

| Demo | Prompt 意图 | 预期行为 |
|------|-----------|---------|
| `math` | 强制使用 `calculator` 工具计算 `(25 * 8) - 14` | Agent 调用 calculator 两次（先乘后减） |
| `weather` | 查询上海天气并总结为一句 | Agent 调用 `get_weather_mock` 一次 |
| `file` | 读取 `sample_note.txt`，总结 3 个关键点并建议下一步 | Agent 调用 `read_file` 一次 |
| `stream` | 用 4 句话解释 Agent Harness Engineering 的学习路径 | 流式输出纯文本，不调用工具 |

#### Demo 包装脚本

`demos/` 目录下 4 个薄脚本（`demo_math.py` / `demo_weather.py` / `demo_file.py` / `demo_stream.py`）都是同一模式：

```python
def main() -> None:
    week3_dir = Path(__file__).resolve().parents[1]
    subprocess.run(
        [sys.executable, str(week3_dir / "main.py"), "math"],
        check=True,
    )
```

它们只是 `python main.py <demo_name>` 的快捷方式，便于从任意工作目录运行。

---

### 4.5 执行引擎升级 `agent/loop.py` — 升级

**文件**：`minimal_agent/agent/loop.py`

#### 新增：force_first_tool 机制

```python
class AgentLoop:
    def __init__(
        self,
        config: AgentConfig,
        tools: list[BaseTool],
        force_first_tool: str | None = None,  # 新增参数
    ) -> None:
        ...
        self._force_first_tool = force_first_tool

    def _tool_choice_for_state(self, state: AgentState) -> str | dict[str, object]:
        if state.step_count != 0 or not self._force_first_tool:
            return "auto"
        return {
            "type": "function",
            "function": {"name": self._force_first_tool},
        }
```

**设计意图**：`math` demo 中，用户要求"用 calculator 工具计算，不要心算"。通过 `tool_choice={"type": "function", "function": {"name": "calculator"}}`，强制 LLM 在第一步就调用 `calculator`，而不是直接心算回答。

这与 AgentConfig 中的 `tool_choice` 概念一致，但它是 **per-run** 的参数，而非全局配置。

#### 升级后：StepLogger 集成

```python
def _log_step(
    self,
    state: AgentState,
    parsed: ParsedResponse,
    token_usage: TokenUsage,
) -> None:
    if parsed.action_type is ActionType.TOOL_CALLS:
        tool_names = ", ".join(call.tool_name for call in parsed.tool_calls)
        detail = f"tool_calls={tool_names}"
        tool_name = tool_names
    elif parsed.action_type is ActionType.FINAL_ANSWER:
        detail = "final_answer"
        tool_name = ""
    else:
        detail = f"parse_error={parsed.error_message}"
        tool_name = ""

    self._step_logger.log(
        StepLog(
            step_number=state.step_count,
            action_type=parsed.action_type,
            tool_name=tool_name,
            elapsed_seconds=state.elapsed_seconds,
            token_usage=token_usage,
            detail=detail,
        )
    )
```

`_log_step()` 根据 `action_type` 自动区分三种日志格式：

| ActionType | tool_name | detail |
|-----------|-----------|--------|
| `TOOL_CALLS` | 逗号分隔的工具名列表 | `tool_calls=calculator,search_web_mock` |
| `FINAL_ANSWER` | 空字符串 | `final_answer` |
| `PARSE_ERROR` | 空字符串 | `parse_error=<错误信息>` |

#### 其他微调

- `_call_llm_with_retry()` 中，`self._llm_client.chat()` 调用新增了 `tool_choice=self._tool_choice_for_state(state)` 参数
- `_sanitize_final_answer()` 方法保持不变（Week 2 就已引入）

---

### 4.6 响应解析 `llm/parser.py` — 微调

**文件**：`minimal_agent/llm/parser.py`

Week 3 的 parser 与 Week 2 几乎一致，唯一的改进是错误信息更精准：

```python
# Week 2
f"Invalid JSON in tool call arguments for `{tool_call.function.name}`: {exc}"

# Week 3
f"Invalid JSON for tool `{tool_call.function.name}`: {exc}"
```

结构上新增了 `ActionType` 的 `str` 继承（`class ActionType(str, Enum)`），使得 `.value` 属性可以直接用于字符串拼接（如日志输出中的 `entry.action_type.value`）。

---

### 4.7 工具层保持不变

`minimal_agent/tools/` 中的所有文件与 Week 2 **完全一致**。这验证了 `BaseTool` 抽象的设计正确性——当 harness 层升级时，工具实现无需改动。

5 个工具回顾：

| 工具 | 类型 | 说明 |
|------|------|------|
| `calculator` | 真实执行 | 四则运算，参数校验（除零检查） |
| `get_weather_mock` | Mock | 4 个城市的硬编码天气数据 |
| `search_web_mock` | Mock | 关键词匹配返回预设结果 |
| `read_file` | 真实执行 | 读取本地 UTF-8 文本文件 |
| `shell_exec_mock` | Mock | 仅返回提示，不执行任何命令 |

---

## 5. 关键设计决策

### 决策 1：手写 YAML 解析 vs 引入 PyYAML

**选择**：手写 ~50 行微型解析器。

**理由**：
- Phase 1 的配置结构极简单（扁平键值 + 一个字面量块），不需要完整的 YAML 1.2 支持
- 手写解析器是理解"配置加载原理"的教学点
- 避免引入不必要的依赖（python 环境中不一定预装了 PyYAML）
- Phase 2+ 可以随时替换为标准库

### 决策 2：步数强制限制 vs 模型自主退出

**选择**：通过 `force_first_tool` 在第一步限制 `tool_choice`。

**理由**：
- 对于明确要求"用工具计算"的场景，只靠 system prompt 提示不够可靠
- `tool_choice={"type": "function", ...}` 是 OpenAI API 原生支持的硬约束，比 prompt engineering 更可靠
- 只在 step 0 生效，后续步数恢复 `"auto"`，不会限制 agent 的自主性

### 决策 3：流式输出不接入 AgentLoop

**选择**：`run_streaming_demo()` 直接调用 `LLMClient.stream_chat()`，完全绕过 `AgentLoop`。

**理由**：
- Week 3 的流式输出目标是**纯文本生成**，不需要 tool calling
- 流式 + tool calling 的整合需要流式 JSON parser + 状态管理，复杂度远超 Phase 1 范围
- 独立的代码路径更清晰：同步 agent → AgentLoop；流式文本 → LLMClient.stream_chat()
- Phase 3 (可靠性工程) 会重新审视流式 tool calling

### 决策 4：思考块过滤放在 main.py 而非 client.py

**选择**：`_strip_streaming_think_blocks()` 定义在 `main.py` 而非 `llm/client.py`。

**理由**：
- 不是所有 LLM provider 都使用 `<think>` 标签，这是 provider-specific 的处理
- `LLMClient` 应保持 provider-agnostic（只负责 API 调用）
- 内容后处理属于应用层的职责

---

## 6. 完整数据流：一次流式执行的全程

以 `stream` demo 为例追踪 `python main.py stream` 的完整过程：

```
Step 0: 启动
┌────────────────────────────────────────────────────┐
│ python "Phase 1\week3\main.py" stream              │
│                                                    │
│ parse_args() → args.demo = "stream"                │
│ AgentConfig.from_yaml("config.yaml") → config      │
│ build_demo_prompt("stream") → prompt:              │
│   "Explain the core learning path of Agent        │
│    Harness Engineering..."                         │
└────────────────────────────────────────────────────┘

Step 1: 进入流式路径
┌────────────────────────────────────────────────────┐
│ main() 检测 args.demo == "stream"                  │
│   → run_streaming_demo(config, prompt)             │
│                                                    │
│ LLMClient(config) 初始化                            │
│   → OpenAI(api_key, base_url, timeout=60)          │
└────────────────────────────────────────────────────┘

Step 2: 发起流式请求
┌────────────────────────────────────────────────────┐
│ client.stream_chat(messages)                       │
│   messages = [                                     │
│     {"role": "system", "content": config.system_prompt},│
│     {"role": "user", "content": prompt},           │
│   ]                                                │
│                                                    │
│ API 调用:                                           │
│   POST /v1/chat/completions                        │
│   {                                                │
│     "model": "...",                                │
│     "messages": [...],                             │
│     "temperature": 0.2,                            │
│     "stream": true                                 │
│   }                                                │
└────────────────────────────────────────────────────┘

Step 3: 逐 chunk 处理
┌────────────────────────────────────────────────────┐
│ for chunk in client.stream_chat(messages):         │
│   chunk → delta.content → "Agent Harness"          │
│   → _strip_streaming_think_blocks(chunk, state)    │
│     → 过滤 <think>...</think> 块                   │
│     → 返回用户可见文本                              │
│   → print(visible, end="", flush=True)             │
│     → 终端实时显示，不换行                          │
│                                                    │
│ 下一个 chunk: " Engineering is"                     │
│ 下一个 chunk: " a systematic approach..."           │
│ ...持续直到流结束                                   │
└────────────────────────────────────────────────────┘

Step 4: 流结束
┌────────────────────────────────────────────────────┐
│ stream 生成器耗尽                                   │
│ _strip_streaming_think_blocks("", state, flush=True)│
│   → 清空缓冲区中的残留内容                           │
│ print() → 换行                                     │
│ → 程序正常退出                                     │
└────────────────────────────────────────────────────┘
```

**与同步 AgentLoop 的流程对比**：

```
同步路径 (math/weather/file):
  main() → AgentLoop.run() → while loop → chat() → parse → tool dispatch → final_answer

流式路径 (stream):
  main() → run_streaming_demo() → stream_chat() → 逐 chunk 打印 → 结束
```

---

## 7. Week 2 vs Week 3 对比表

| 维度 | Week 2 | Week 3 |
|------|--------|--------|
| **配置来源** | `.env` + 代码硬编码默认值 | `.env` + `config.yaml` + 代码默认值（三层优先级） |
| **配置加载** | `AgentConfig.from_env()` | `AgentConfig.from_yaml(path)` |
| **日志方式** | `print(f"[Step {n}] action=...")` | `StepLogger.log(StepLog(...))` 结构化输出 |
| **日志内容** | action + detail | action + tool + elapsed + tokens (prompt/completion/total) + detail |
| **LLM 调用** | `chat()` 同步调用 | `chat()` 同步 + `stream_chat()` 流式 |
| **CLI 接口** | `sys.argv` 硬编码 | `argparse` + `--config` 选项 |
| **Demo 数量** | 3 个 (math/weather/file) | 4 个 (+ stream) |
| **Demo 脚本** | 无独立脚本 | `demos/` 目录 4 个薄包装 |
| **项目文档** | README + architecture.md + execution_loop.md | + harness_components.md + phase1_summary.md |
| **Tool 实现** | 5 个 | 5 个（无变化） |
| **错误处理** | 4 层 | 4 层（无变化） |
| **force_first_tool** | 无 | 支持（per-run 强制首步工具选择） |
| **YAML 解析** | 无 | 手写 ~50 行微型解析器 |

---

## 8. 动手实践建议

### 实验 1：修改 config.yaml 观察行为变化

```powershell
# 改 max_steps 为 2
# Phase 1\week3\config.yaml: max_steps: 2
python "Phase 1\week3\main.py" math
```

观察 agent 在 2 步后提前终止。

### 实验 2：对比同步与流式输出

```powershell
python "Phase 1\week3\main.py" stream
```

观察文字逐字出现的流式效果，理解 `flush=True` 的作用。

### 实验 3：观察 token 消耗

运行 `python main.py math`，注意终端输出的 token 数：

```
[Step 01] action=tool_calls | tool=calculator | elapsed=1.23s | tokens=245 (prompt=200, completion=45) | tool_calls=calculator
```

每一轮 LLM 调用都会累积 prompt tokens（因为 messages 历史在增长）。这是 agent 的核心成本来源，在后续 Phase 的 memory/context 管理中会重点处理。

### 实验 4：添加新工具并观察日志

仿照 `calculator.py` 添加一个新工具（如翻译工具），在 `tools/__init__.py` 中注册，运行后观察 `StepLogger` 的输出是否正确显示了新工具名。

### 实验 5：trace 流式思考块过滤

如果使用的 LLM provider 会返回 `<think>` 标签，临时注释掉 `_strip_streaming_think_blocks`，对比过滤前后的输出差异。

### 实验 6：手动触发 YAML 解析边界情况

- 删除 `config.yaml` → agent 应使用全部默认值正常运行
- 在 YAML 中添加注释 → 应被忽略
- 将 `max_steps: 20` 改为 `max_steps: not_a_number` → 观察类型转换失败的行为

---

## 9. 与后续 Week 的关联

| 后续 Week | 新增能力 | Week 3 的基础 |
|-----------|---------|-------------|
| Week 4 | MCP 工具集成、Tool Registry | `BaseTool` 抽象 + 结构化日志 |
| Week 5-6 | Memory 系统、上下文压缩 | `AgentState.messages` 管理 + TokenUsage 追踪 |
| Week 7-9 | 结构化 Trace、Guardrails、Eval | `StepLogger` → `TraceCollector` 演进 |
| Week 10-12 | Multi-Agent、Server、Monitoring | `config.yaml` 模式 → 生产配置中心 |

**核心递进关系**：

```
Week 2: Agent Loop 能跑
   ↓
Week 3: Agent Loop 可观测 + 可配置 + 可演示  ← 你在这里
   ↓
Phase 2: Tool Orchestration + Memory + Planning
   ↓
Phase 3: Reliability + Observability + Evaluation
   ↓
Phase 4: Production Architecture + Multi-Agent
```

---

## 附录 A：config.yaml 完整字段说明

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `model` | `str` | `""` | 为空时使用 `.env` 的 `MODEL` |
| `max_steps` | `int` | `20` | Agent 最多执行步数 |
| `max_time_seconds` | `int` | `300` | 单次运行最长秒数 |
| `llm_timeout_seconds` | `int` | `60` | 单次 LLM API 调用超时 |
| `llm_max_retries` | `int` | `2` | LLM 调用失败最大重试次数 |
| `stream_temperature` | `float` | `0.2` | 流式输出的温度参数 |
| `system_prompt` | `str` | 多行文本 | Agent 的 system prompt |

## 附录 B：关键文件行数统计

| 文件 | 行数 | 职责 |
|------|------|------|
| `main.py` | ~140 | 入口：参数解析 + 流式/同步分发 + 思考块过滤 |
| `config.py` | ~135 | 配置：.env 加载 + YAML 解析 + AgentConfig 组装 |
| `agent/loop.py` | ~188 | 核心：ReAct 循环 + 重试 + 工具分发 + 日志集成 |
| `agent/logger.py` | ~33 | 日志：StepLog 数据类 + StepLogger 输出 |
| `agent/state.py` | ~47 | 状态：AgentState + AgentRunResult + AgentStatus |
| `llm/client.py` | ~82 | LLM：chat() + stream_chat() + TokenUsage 解析 |
| `llm/parser.py` | ~71 | 解析：响应 → TOOL_CALLS / FINAL_ANSWER / PARSE_ERROR |
| `tools/base.py` | ~35 | 工具抽象：BaseTool + to_openai_tool + run + execute |
| **总计** | **~730** | Week 3 相比 Week 2 (~500 行) 增加了 ~230 行工程化代码 |

---

> **记住 Week 3 的核心价值：写 agent 不难，写一个可观测、可配置、可维护的 agent harness 才是工程能力的分水岭。**
