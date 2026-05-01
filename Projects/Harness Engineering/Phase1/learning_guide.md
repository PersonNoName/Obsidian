# Week 2 学习指南：从零构建最小 AI Agent

> 本文档系统讲解 `Phase 1/week2` 项目的全部设计思路、代码实现和核心概念，适合从头开始学习 AI Agent 的开发者。

---

## 目录

1. [项目定位](#1-项目定位)
2. [核心理念：什么是 Agent Harness](#2-核心理念什么是-agent-harness)
3. [总架构图](#3-总架构图)
4. [逐模块精讲](#4-逐模块精讲)
   - [4.1 程序入口 `main.py`](#41-程序入口-mainpy)
   - [4.2 配置管理 `config.py`](#42-配置管理-configpy)
   - [4.3 状态管理 `agent/state.py`](#43-状态管理-agentstatepy)
   - [4.4 执行引擎 `agent/loop.py`](#44-执行引擎-agentlooppy--核心)
   - [4.5 LLM 客户端 `llm/client.py`](#45-llm-客户端-llmclientpy)
   - [4.6 响应解析 `llm/parser.py`](#46-响应解析-llmparserpy)
   - [4.7 工具抽象 `tools/base.py`](#47-工具抽象-toolsbasepy)
   - [4.8 五个具体工具](#48-五个具体工具)
5. [完整数据流：一次执行的全过程](#5-完整数据流一次执行的全过程)
6. [错误处理全景](#6-错误处理全景)
7. [关键设计模式](#7-关键设计模式)
8. [动手实践建议](#8-动手实践建议)

---

## 1. 项目定位

Week 2 的核心任务是：**从零手写一个最小可运行的 Agent 框架**。

关键约束：
- **不依赖** LangChain、CrewAI、Agents SDK 等任何 agent 框架
- 仅使用 `openai`（LLM 调用）+ `pydantic`（数据校验）+ `pydantic-settings`（配置加载）
- 代码量控制在 ~500 行以内，保证可读性
- 目标是**理解 agent 的本质**，而不是构建生产系统

学习价值：当你写完这段代码，你会清楚地知道一个 AI Agent 不过是 **"LLM + 工具调用 + 循环控制"**，没有任何魔法。

---

## 2. 核心理念：什么是 Agent Harness？

### Harness 的含义

**Harness** = 马具 / 安全带，在 AI 工程中引申为**控制框架**。

一个 Agent Harness 负责：
```
┌─────────────────────────────────────────┐
│            Agent Harness                │
│                                         │
│  ① 管理消息历史（上下文窗口）            │
│  ② 决定何时调用 LLM                     │
│  ③ 解析 LLM 的响应（工具调用 or 最终答案）│
│  ④ 执行工具并收集结果（observation）      │
│  ⑤ 将结果反馈给 LLM，驱动下一轮思考       │
│  ⑥ 判断何时停止（完成/超时/步数耗尽）     │
│  ⑦ 错误恢复（失败重试、parse error 修正） │
│                                         │
└─────────────────────────────────────────┘
```

### ReAct 模式

这个项目实现了经典的 **ReAct (Reasoning + Acting)** 循环：

```
User Question
    ↓
┌─→ LLM 推理 → 输出 action（工具调用）
│       ↓
│   执行工具 → 得到 observation（观察结果）
│       ↓
│   将 observation 追加到消息历史
│       ↓
└─── LLM 再次推理（基于新信息）→ ...
         ↓
    最终输出 final_answer
```

每轮都是：**Think → Act → Observe → Think again → ... → Answer**。

---

## 3. 总架构图

### 目录结构

```text
week2/
├── main.py                    ← 入口：装配配置、工具、启动运行
├── README.md
├── data/
│   └── sample_note.txt        ← file demo 的测试数据
├── demos/                     ← 三个薄包装脚本
│   ├── demo_math.py
│   ├── demo_weather.py
│   └── demo_file.py
├── docs/                      ← 文档
│   ├── architecture.md
│   └── execution_loop.md
│   └── learning_guide.md      ← 本文档
└── minimal_agent/             ← 核心包
    ├── config.py              ← 配置加载
    ├── agent/
    │   ├── state.py           ← AgentState + AgentRunResult
    │   └── loop.py           ← 执行引擎（核心！！！）
    ├── llm/
    │   ├── client.py          ← OpenAI 兼容接口封装
    │   └── parser.py          ← 响应 → TOOL_CALLS / FINAL_ANSWER / PARSE_ERROR
    └── tools/
        ├── base.py            ← BaseTool 抽象基类
        ├── calculator.py      ← 四则运算
        ├── weather.py         ← 模拟天气
        ├── search.py          ← 模拟搜索
        ├── file_reader.py     ← 读文件
        └── shell.py           ← 模拟 shell
```

### 依赖关系图

```
main.py
  ├─→ AgentConfig.from_env()        ──→ config.py ──→ Phase 1/.env
  ├─→ build_default_tools()         ──→ tools/__init__.py
  └─→ AgentLoop(config, tools)      ──→ agent/loop.py
        ├─→ AgentState              ──→ agent/state.py
        ├─→ LLMClient               ──→ llm/client.py ──→ OpenAI API
        ├─→ parse_response()        ──→ llm/parser.py
        ├─→ tool.run(raw_args)      ──→ tools/base.py ──→ 各工具 execute()
        └─→ AgentRunResult          ──→ agent/state.py
```

---

## 4. 逐模块精讲

### 4.1 程序入口 `main.py`

**文件**：`minimal_agent/../main.py`

```python
def main() -> None:
    args = parse_args()                              # ① 解析 CLI 参数（math/weather/file）
    config = AgentConfig.from_env()                  # ② 从 .env 加载模型配置
    agent = AgentLoop(config=config, tools=build_default_tools())  # ③ 创建 Agent
    prompt = build_demo_prompt(args.demo)            # ④ 构造 prompt
    result = agent.run(prompt)                       # ⑤ 运行 Agent
    print(result.output_text)                        # ⑥ 输出结果
```

**关键设计点**：
- `main.py` 只负责**装配**，不包含任何业务逻辑
- `build_demo_prompt()` 为每个 demo 硬编码了中文 prompt
- `file` demo 中使用了绝对路径引用 `sample_note.txt`，因为 LLM 需要真实的文件路径来调用 `read_file` 工具

**三个 Demo 的 prompt**：

| Demo | Prompt | 预期行为 |
|---|---|---|
| `math` | "请帮我计算 (25 * 8) - 14，然后告诉我最终结果。" | LLM 调用 `calculator` 2 次（先乘后减） |
| `weather` | "请查询 Shanghai 的天气，并用一句话总结给我。" | LLM 调用 `get_weather_mock` 1 次 |
| `file` | "请读取文件 `{path}`，总结其中的 3 个关键信息点..." | LLM 调用 `read_file` 1 次 |

---

### 4.2 配置管理 `config.py`

**文件**：`minimal_agent/config.py`

```python
class EnvSettings(BaseSettings):
    api_key: str = Field(..., alias="API_KEY")
    base_url: str = Field(..., alias="BASE_URL")
    model: str = Field(..., alias="MODEL")
    # 自动读取 Phase 1/.env

@dataclass(slots=True)
class AgentConfig:
    api_key: str
    base_url: str
    model: str
    system_prompt: str          # "You are a helpful agent..."
    max_steps: int = 20         # 最多执行 20 步
    max_time_seconds: int = 300 # 最长运行 5 分钟
    llm_timeout_seconds: int = 60  # 单次 LLM 调用超时
    llm_max_retries: int = 2       # LLM 失败最大重试次数
```

**学习要点**：

1. **`pydantic-settings` 的 `BaseSettings`**：自动从 `.env` 文件加载环境变量，`alias` 定义了变量名映射。例如环境变量 `API_KEY` 映射到 Python 属性 `api_key`。

2. **`SettingsConfigDict`**：
   ```python
   model_config = SettingsConfigDict(
       env_file=ENV_FILE,           # 指定 .env 文件路径（Phase 1/.env）
       env_file_encoding="utf-8",
       extra="ignore",              # 忽略 .env 中未定义的字段
       populate_by_name=True,       # 允许用 Python 属性名或 alias 名访问
   )
   ```

3. **`EnvSettings` 与 `AgentConfig` 的分层**：
   - `EnvSettings`：只管从环境中读取原始值（API key、URL、model）
   - `AgentConfig`：把环境值和**业务默认值**（max_steps、system_prompt 等）组装成一个完整的运行时配置
   - 这种分层让后续改配置文件格式（如换成 YAML）时，只需修改 `AgentConfig.from_env()`，不影响其他代码

4. **`slots=True`**：使用 `__slots__` 减少内存占用，提升属性访问速度（这是一个性能优化细节）。

---

### 4.3 状态管理 `agent/state.py`

**文件**：`minimal_agent/agent/state.py`

```python
class AgentStatus(str, Enum):
    RUNNING = "running"           # 正在执行
    COMPLETED = "completed"       # 正常完成（收到了 final_answer）
    MAX_STEPS_REACHED = "max_steps_reached"  # 步数耗尽
    TIMEOUT = "timeout"           # 超时
    FAILED = "failed"             # LLM 调用彻底失败
```

#### AgentState — 运行时可变状态

```python
@dataclass(slots=True)
class AgentState:
    messages: list[dict[str, Any]]   # 完整的消息历史（含 system、user、assistant、tool）
    max_steps: int                   # 步数上限
    max_time_seconds: int            # 时间上限
    status: AgentStatus              # 当前状态，初始为 RUNNING
    step_count: int = 0              # 已执行步数
    start_time: float = time.time()  # 开始时间戳
    final_answer: str | None = None  # 最终答案（仅 COMPLETED 时有值）
    errors: list[str] = []           # 错误日志
```

**关键设计**：
- `start_time` 使用 `field(default_factory=time.time)` —— 在对象创建时记录时间，而非类定义时
- `elapsed_seconds` 是计算属性（`@property`），每次访问都实时计算已过时间
- `record_error()` 方法集中管理错误记录
- `messages` 是字典列表，兼容 OpenAI 的消息格式：`{"role": "system/user/assistant/tool", "content": "..."}`

#### AgentRunResult — 最终不可变输出

```python
@dataclass(slots=True)
class AgentRunResult:
    status: AgentStatus          # 终止状态
    output_text: str             # 用户可读的输出文本
    step_count: int              # 共执行了多少步
    elapsed_seconds: float       # 总耗时
    errors: list[str]            # 错误汇总
```

**为什么分离 State 和 Result**：
- `AgentState` 是引擎内部使用的可变对象，会被反复修改
- `AgentRunResult` 是不可变的最终产物，暴露给调用方
- 这避免了调用方直接依赖内部状态，便于后续扩展（如添加 trace、导出日志）

---

### 4.4 执行引擎 `agent/loop.py` — 核心

**文件**：`minimal_agent/agent/loop.py`

这是整个项目最重要的文件，全部精华都在 `AgentLoop.run()` 方法中。让我们逐段剖析。

#### 类初始化

```python
class AgentLoop:
    def __init__(self, config: AgentConfig, tools: list[BaseTool]) -> None:
        self._config = config
        self._llm_client = LLMClient(config)
        self._tool_registry = {tool.name: tool for tool in tools}
```

- `_tool_registry` 是一个 `{name: tool_instance}` 的字典，用于 O(1) 查找工具
- 每个工具的名称（如 `"calculator"`）必须和 LLM 返回的 `function.name` 完全匹配

#### run() 方法总览

```python
def run(self, user_input: str) -> AgentRunResult:
    state = AgentState(messages=[...])   # ① 初始化状态
    
    while state.status is AgentStatus.RUNNING:   # ② 主循环
        termination = self._check_termination(state)   # ③ 终止检查
        if termination:
            break
        
        message = self._call_llm_with_retry(state)     # ④ 调用 LLM
        if message is None:
            state.status = AgentStatus.FAILED
            break
        
        state.step_count += 1
        state.messages.append(message)
        
        parsed = parse_response(message)    # ⑤ 解析响应
        
        if parsed is FINAL_ANSWER:          # ⑥ 最终答案 → 完成
            state.final_answer = ...
            break
        elif parsed is PARSE_ERROR:         # ⑦ 解析错误 → 提示重试
            state.messages.append(hint)
            continue
        else:                                # ⑧ 工具调用 → 执行
            self._handle_tool_calls(state, parsed)
    
    return self._build_result(state)         # ⑨ 构建结果
```

#### 各子方法详解

**`_check_termination()` — 终止条件检查**

```python
def _check_termination(self, state: AgentState) -> AgentStatus | None:
    if state.step_count >= state.max_steps:   # 条件 1：步数耗尽
        return AgentStatus.MAX_STEPS_REACHED
    if state.elapsed_seconds > state.max_time_seconds:  # 条件 2：超时
        return AgentStatus.TIMEOUT
    return None   # 继续运行
```

终止条件有且仅有这两条硬限制。注意检查顺序：**先检查步数、再检查时间**，因为步数检查开销更小。

**`_call_llm_with_retry()` — 带重试的 LLM 调用**

```python
def _call_llm_with_retry(self, state: AgentState):
    for attempt in range(1, llm_max_retries + 2):   # 总共尝试 max_retries+1 次
        try:
            return self._llm_client.chat(state.messages, tools)
        except Exception:
            state.record_error(f"LLM call failed on attempt {attempt}")
            time.sleep(0.5 * attempt)   # 指数退避的简化版
    return None   # 全部失败
```

关键细节：
- `range(1, llm_max_retries + 2)`：默认 `llm_max_retries=2`，所以遍历 `range(1, 4)` → 尝试 3 次
- `time.sleep(0.5 * attempt)`：简化版退避（0.5s → 1.0s → 1.5s）
- 如果全部失败返回 `None`，调用方会设置状态为 `FAILED`

**`_handle_tool_calls()` — 工具分发**

```python
def _handle_tool_calls(self, state, parsed):
    for tool_call in parsed.tool_calls:
        tool = self._tool_registry.get(tool_call.tool_name)
        if tool is None:
            tool_output = {"ok": False, "error": "Unknown tool"}
        else:
            try:
                result_text = tool.run(tool_call.arguments)
                tool_output = {"ok": True, "result": result_text}
            except Exception:
                tool_output = {"ok": False, "error": f"Tool failed: ..."}
        
        state.messages.append({
            "role": "tool",
            "tool_call_id": tool_call.tool_call_id,
            "content": json.dumps(tool_output)
        })
```

关键设计：
- **工具错误不中断循环**：即使工具执行失败，错误信息也会作为 observation 写入消息历史，LLM 看到后可能自我纠正
- **`tool_call_id` 必须正确传递**：OpenAI 函数调用协议要求 tool 消息关联对应的 `tool_call_id`
- **observation 格式统一为 JSON**：`{"ok": true/false, "result/error": "..."}`，保持一致性

**`_sanitize_final_answer()` — 清理思考块**

```python
def _sanitize_final_answer(self, text: str) -> str:
    cleaned = re.sub(r"<think>.*?</think>", "", text, flags=re.DOTALL).strip()
    return cleaned or text.strip()
```

某些模型（如 MiniMax、DeepSeek）会在最终答案中嵌入 `{think}` 标签，包含模型的内部推理过程。这个函数将它们移除，只保留用户可读的答案。

**`_log_step()` — 步骤日志**

```python
print(f"[Step {state.step_count}] action={action} | detail={detail} | elapsed={elapsed}s")
```

简单但有效：每步打印一条日志，让运行过程可见。

---

### 4.5 LLM 客户端 `llm/client.py`

**文件**：`minimal_agent/llm/client.py`

```python
class LLMClient:
    def __init__(self, config: AgentConfig):
        self._client = OpenAI(
            api_key=config.api_key,
            base_url=config.base_url,
            timeout=config.llm_timeout_seconds,
        )
    
    def chat(self, messages, tools) -> Any:
        response = self._client.chat.completions.create(
            model=self._config.model,
            messages=messages,
            tools=[tool.to_openai_tool() for tool in tools],
            tool_choice="auto",   # 让模型自己决定是否调用工具
        )
        return response.choices[0].message
```

**学习要点**：
- `tool_choice="auto"`：告诉 OpenAI API 让模型自己决定是否调用工具、调用哪个工具
- `[tool.to_openai_tool() for tool in tools]`：将每个 `BaseTool` 实例转换为 OpenAI 要求的函数 schema 格式
- 返回的是原始 `ChatCompletionMessage` 对象，由 parser 进一步处理

---

### 4.6 响应解析 `llm/parser.py`

**文件**：`minimal_agent/llm/parser.py`

这是将 LLM 的原始响应**归一化**为三种确定动作的关键模块。

```python
def parse_response(message: Any) -> ParsedResponse:
    # 情况 1：模型要求调用工具
    if message.tool_calls:
        parsed_calls = []
        for tc in message.tool_calls:
            try:
                arguments = json.loads(tc.function.arguments)
            except JSONDecodeError:
                return ParsedResponse(PARSE_ERROR, error_message="Invalid JSON...")
            parsed_calls.append(ParsedToolCall(...))
        return ParsedResponse(TOOL_CALLS, tool_calls=parsed_calls)
    
    # 情况 2：模型返回了文本（视为最终答案）
    if message.content and message.content.strip():
        return ParsedResponse(FINAL_ANSWER, final_answer=message.content.strip())
    
    # 情况 3：既无工具调用也无文本 → 错误
    return ParsedResponse(PARSE_ERROR, error_message="...")
```

**三种 ActionType**：

| ActionType | 触发条件 | Loop 行为 |
|---|---|---|
| `TOOL_CALLS` | 响应中有 `tool_calls` 且 JSON 合法 | 执行工具，追加 observation |
| `FINAL_ANSWER` | 响应中有非空 `content` 文本 | 设置最终答案，标记 COMPLETED |
| `PARSE_ERROR` | JSON 解析失败 或 响应完全为空 | 追加修正提示，让 LLM 重试 |

**关键细节**：`json.loads()` 可能失败。如果 LLM 生成了不合法的 JSON 参数（例如引号不匹配），parser 会捕获 `JSONDecodeError` 并返回 `PARSE_ERROR`，而不是让整个程序崩溃。这就是 agent **自愈能力**的基础。

---

### 4.7 工具抽象 `tools/base.py`

**文件**：`minimal_agent/tools/base.py`

```python
class BaseTool(ABC):
    name: str           # 工具名，如 "calculator"
    description: str    # 工具描述，会发送给 LLM
    args_model: type[BaseModel]   # Pydantic 参数模型

    def to_openai_tool(self) -> dict:
        schema = self.args_model.model_json_schema()
        schema["additionalProperties"] = False
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": schema,
            },
        }
    
    def run(self, raw_args: dict) -> str:
        validated_args = self.args_model.model_validate(raw_args)
        return self.execute(validated_args)
    
    @abstractmethod
    def execute(self, args: BaseModel) -> str:
        ...
```

**设计模式：模板方法（Template Method）**

`run()` 是模板方法，它：
1. 先用 Pydantic 校验参数（类型检查、必填项检查）
2. 再调用子类的 `execute()` 执行具体逻辑

这样每个工具只需关注 `execute()` 的业务逻辑，参数校验由基类统一处理。

**`to_openai_tool()` 的工作原理**：
- `model_json_schema()` 将 Pydantic 模型自动转换为 JSON Schema
- 例如 `CalculatorArgs(a: float, b: float, operator: Literal["add","subtract","multiply","divide"])` 会自动生成包含类型信息和约束的 schema
- `"additionalProperties": False` 确保 LLM 不会传递未定义的参数

---

### 4.8 五个具体工具

#### calculator — 计算器

```python
class CalculatorArgs(BaseModel):
    a: float
    b: float
    operator: Literal["add", "subtract", "multiply", "divide"]

class CalculatorTool(BaseTool):
    name = "calculator"
    description = "Perform basic arithmetic on two numbers."
    
    def execute(self, args):
        # 根据 operator 执行对应运算，返回 "a + b = result" 形式
        if args.operator == "divide" and args.b == 0:
            raise ValueError("Division by zero is not allowed.")
```

**要点**：
- 使用 `Literal` 类型限制 `operator` 只能为四种值之一，LLM 看到 schema 后会遵守
- 除零错误会抛出异常，被上层 `_handle_tool_calls` 捕获并作为 observation 反馈

#### get_weather_mock — 天气查询（Mock）

```python
WEATHER_DATA = {
    "beijing": "晴，24C，东南风 2 级",
    "shanghai": "多云，22C，东北风 3 级",
    "shenzhen": "小雨，28C，南风 2 级",
    "hong kong": "多云，27C，湿度较高",
}
```

- 内置了 4 个城市的硬编码天气数据
- 城市名做 `strip().lower()` 处理，提高容错性
- 未匹配的城市返回 "暂无 mock 数据"

#### search_web_mock — 搜索（Mock）

- 对 "agent harness" 和 "weather" 两个关键词返回预设结果
- 其他查询返回通用 mock 结果
- 保留了真实搜索工具的接口形态，为后续升级做准备

#### read_file — 文件读取

```python
def execute(self, args):
    file_path = Path(args.file_path).expanduser()
    if not file_path.is_absolute():
        file_path = Path.cwd() / file_path
    if not file_path.exists():
        raise FileNotFoundError(...)
    return file_path.read_text(encoding="utf-8")
```

- 支持相对路径和绝对路径
- `expanduser()` 处理 `~` 路径
- 完善的错误处理（文件不存在、路径是目录）

#### shell_exec_mock — Shell（Mock）

```python
def execute(self, args):
    return f"Mock shell execution only. Command `{args.command}` was not executed."
```

- 仅返回提示信息，**不执行任何真实命令**
- 保留工具接口形态，Phase 1 不引入 sandbox 风险

---

## 5. 完整数据流：一次执行的全过程

以 `math` demo 为例，追踪 `(25 * 8) - 14` 的完整执行：

```
Step 0: 初始化
┌────────────────────────────────────────────┐
│ messages:                                  │
│   [system] "You are a helpful agent..."   │
│   [user]   "请计算 (25 * 8) - 14"         │
│ status: RUNNING, step_count: 0             │
└────────────────────────────────────────────┘

Step 1: LLM 返回 tool_calls
┌────────────────────────────────────────────┐
│ LLM 响应:                                  │
│   tool_calls: [                            │
│     {name: "calculator",                   │
│      args: {a:25, b:8, operator:"multiply"}}│
│   ]                                        │
│                                            │
│ → parser 返回: ActionType.TOOL_CALLS       │
│ → dispatch: calculator.execute(25, 8, *)  │
│ → observation: "25 * 8 = 200"             │
│                                            │
│ messages 新增:                              │
│   [assistant] (tool_call)                  │
│   [tool]     {"ok":true,"result":"25*8=200"}│
└────────────────────────────────────────────┘

Step 2: LLM 再次返回 tool_calls
┌────────────────────────────────────────────┐
│ LLM 响应: (基于已有结果 200)               │
│   tool_calls: [                            │
│     {name: "calculator",                   │
│      args: {a:200, b:14, operator:"subtract"}}│
│   ]                                        │
│                                            │
│ → dispatch: calculator.execute(200, 14, -)│
│ → observation: "200 - 14 = 186"           │
│                                            │
│ messages 新增:                              │
│   [assistant] (tool_call)                  │
│   [tool]     {"ok":true,"result":"200-14=186"}│
└────────────────────────────────────────────┘

Step 3: LLM 返回 final_answer
┌────────────────────────────────────────────┐
│ LLM 响应:                                  │
│   content: "计算结果为 186。"              │
│                                            │
│ → parser 返回: ActionType.FINAL_ANSWER     │
│ → sanitize: "计算结果为 186。"             │
│ → state.status = COMPLETED                 │
│ → 退出循环                                 │
└────────────────────────────────────────────┘

最终输出:
  AgentRunResult(
    status=COMPLETED,
    output_text="计算结果为 186。",
    step_count=3,
    elapsed_seconds=2.34,
    errors=[]
  )
```

---

## 6. 错误处理全景

本项目实现了 **4 层错误处理**，每一层都有明确的设计意图：

```
层级          故障类型           处理策略             为什么这样设计
──────────────────────────────────────────────────────────────────────
LLM 调用层    网络超时/API故障   重试 2 次 + 退避     LLM 调用是最不可靠的环节
Parser 层     JSON 解析失败      回写错误提示重试      给 LLM 自我纠正的机会
Tool 执行层   除零/文件不存在    记录错误 observation  不让单个工具失败中断整个任务
Loop 控制层   步数/时间耗尽      优雅终止 + 状态报告   防止无限循环
```

### 错误恢复示例：Parse Error

```
LLM 返回:
  tool_calls: [{function: {name:"calculator", arguments:"{a:25 b:8}"}}]
                                                      ↑ JSON 格式错误
Parser 检测 → PARSE_ERROR

Loop 处理:
  messages.append({
    "role": "user",
    "content": "Your previous tool call arguments were invalid JSON. 
                Error: Expecting ':' delimiter. Please regenerate."
  })

LLM 重新生成 → 正确的 JSON → 继续执行
```

---

## 7. 关键设计模式

### 7.1 模板方法（Template Method）

```
BaseTool.run()           ← 模板：参数校验 → 执行 → 返回
  └─→ self.execute()     ← 子类实现：具体业务逻辑
```

### 7.2 策略模式（Strategy）

```
parse_response() 根据消息内容返回 3 种 ParsedResponse：
  - TOOL_CALLS   → _handle_tool_calls()
  - FINAL_ANSWER → 设置完成状态
  - PARSE_ERROR  → 提示重试
```

### 7.3 注册表模式（Registry）

```python
self._tool_registry = {
    "calculator": CalculatorTool(),
    "get_weather_mock": WeatherMockTool(),
    ...
}
```

O(1) 查找 + 运行时动态选择工具。

### 7.4 不可变结果模式

```
AgentState      (内部可变) → Loop 修改
AgentRunResult  (外部不可变) → 调用方读取
```

---

## 8. 动手实践建议

理解完代码后，推荐按以下顺序动手实验：

### 实验 1：Trace 一次完整执行
```powershell
python "Phase 1\week2\main.py" math
```
观察终端输出的 `[Step N]` 日志，追踪每一步发生了什么。

### 实验 2：修改 system_prompt
在 `config.py` 的 `system_prompt` 中加入新规则（如"始终用英文回答"），观察行为变化。

### 实验 3：添加一个新工具
仿照 `calculator.py` 添加一个 `string_tool`（如字符串反转、大小写转换），并在 `tools/__init__.py` 中注册。

### 实验 4：制造错误场景
- 临时修改 parser 使其总是返回 PARSE_ERROR，观察 agent 的重试行为
- 将 `max_steps` 改为 1，观察提前终止
- 在 calculator.execute 中故意抛出异常，观察错误 observation

### 实验 5：对比真实框架
运行完 Week 2 后，尝试阅读 LangChain AgentExecutor 或 OpenAI Agents SDK 的源码，你会发现它们的核心循环结构和本项目**本质相同**。

---

## 附录：与后续 Week 的关联

| 后续 Week | 新增能力 | Week 2 的基础 |
|---|---|---|
| Week 3 | 结构化日志、YAML 配置、streaming | AgentConfig 分层设计、log_step 接口 |
| Week 4 | MCP 工具集成 | BaseTool 抽象、tool_registry |
| Week 5-6 | 记忆系统、上下文压缩 | AgentState.messages 管理 |
| Week 7-9 | Trace、Evaluation、Guardrails | AgentRunResult 结构化输出 |
| Week 10-12 | Multi-Agent、Server、Monitoring | AgentLoop.run() 可编排性 |

---

> **记住一句话：Agent = LLM + Tool Calling + Loop Control。Week 2 让你亲手实现了这三样东西。**
