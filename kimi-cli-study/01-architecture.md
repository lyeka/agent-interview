# 01 - 整体架构与分层设计

## 本章目标

理解 kimi-cli 的整体架构设计，掌握各层职责和模块间的通信机制。这是深入源码的前提——先有全局视野，再看细节。

## 架构哲学

kimi-cli 的架构设计体现了三个核心哲学：

1. **分层解耦**：Kosong (LLM) + Wire (事件流) + UI (交互)，每层独立演化
2. **协议优先**：通过 Protocol 定义契约，而非继承实现多态
3. **消除特殊情况**：Wire 协议统一所有 UI，Kosong 统一所有 LLM

## 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        表现层 (UI Layer)                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  Shell UI   │    │  Print UI   │    │   ACP UI    │         │
│  │  (交互式)   │    │  (简单输出) │    │  (IDE集成)  │         │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘         │
├─────────┼──────────────────┼──────────────────┼─────────────────┤
│         └──────────────────┼──────────────────┘                 │
│                            │                                    │
│                     ┌──────▼──────┐                             │
│                     │    Wire     │  ◄── 协议层 (Protocol)      │
│                     │  事件流抽象  │                             │
│                     └──────┬──────┘                             │
├────────────────────────────┼────────────────────────────────────┤
│                            │                                    │
│  ┌─────────────────────────▼─────────────────────────┐         │
│  │                   业务层 (Soul)                    │         │
│  │                                                   │         │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────────┐  │         │
│  │  │ KimiSoul  │  │  Context  │  │   Approval    │  │         │
│  │  │ Agent循环 │  │ 上下文管理 │  │   批准机制    │  │         │
│  │  └─────┬─────┘  └─────┬─────┘  └───────┬───────┘  │         │
│  │        │              │                │          │         │
│  │  ┌─────▼─────┐  ┌─────▼─────┐  ┌───────▼───────┐  │         │
│  │  │ Toolset   │  │Compaction │  │  DenwaRenji   │  │         │
│  │  │ 工具集    │  │ 上下文压缩 │  │  时间旅行    │  │         │
│  │  └───────────┘  └───────────┘  └───────────────┘  │         │
│  │                                                   │         │
│  │  ┌───────────────────────────────────────────┐    │         │
│  │  │              LaborMarket                  │    │         │
│  │  │  ┌─────────────┐    ┌─────────────────┐   │    │         │
│  │  │  │Fixed Subagent│   │Dynamic Subagent │   │    │         │
│  │  │  └─────────────┘    └─────────────────┘   │    │         │
│  │  └───────────────────────────────────────────┘    │         │
│  └───────────────────────────────────────────────────┘         │
├─────────────────────────────────────────────────────────────────┤
│                       框架层 (Framework)                         │
│  ┌─────────────────────────────────────────────────────┐       │
│  │                      Kosong                          │       │
│  │  ┌─────────┐  ┌───────────┐  ┌─────────────────┐    │       │
│  │  │  Kimi   │  │ Anthropic │  │  Google Gemini  │    │       │
│  │  │ Provider│  │  Provider │  │    Provider     │    │       │
│  │  └─────────┘  └───────────┘  └─────────────────┘    │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────┐       │
│  │                      PyKAOS                          │       │
│  │  ┌─────────────┐    ┌─────────────────────────┐     │       │
│  │  │Local Backend│    │     SSH Backend         │     │       │
│  │  └─────────────┘    └─────────────────────────┘     │       │
│  └─────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

## 目录结构

```
kimi-cli/
├── src/kimi_cli/                 # 主应用代码
│   ├── cli.py                    # CLI 入口 (Typer)
│   ├── app.py                    # KimiCLI 主类
│   ├── config.py                 # 配置管理 (TOML)
│   ├── llm.py                    # LLM 创建逻辑
│   ├── session.py                # Session 管理
│   ├── agentspec.py              # Agent YAML 加载
│   │
│   ├── soul/                     # Agent 核心逻辑 ★
│   │   ├── kimisoul.py           # 主 Agent 循环
│   │   ├── agent.py              # Runtime + LaborMarket
│   │   ├── toolset.py            # 工具集管理
│   │   ├── context.py            # Context + Checkpoint
│   │   ├── compaction.py         # Context 压缩
│   │   ├── approval.py           # 工具批准机制
│   │   ├── denwarenji.py         # D-Mail 时间旅行
│   │   └── message.py            # 消息工具函数
│   │
│   ├── tools/                    # 内置工具
│   │   ├── file/                 # 文件操作 (Read/Write/Grep/Glob)
│   │   ├── shell/                # Shell 命令
│   │   ├── web/                  # Web 搜索/抓取
│   │   ├── multiagent/           # Multi-Agent (Task/CreateSubagent)
│   │   ├── dmail/                # D-Mail 工具
│   │   └── think/                # 思考工具
│   │
│   ├── wire/                     # Wire 协议 ★
│   │   ├── types.py              # 事件类型定义
│   │   ├── __init__.py           # Wire/WireSoulSide/WireUISide
│   │   ├── server.py             # WireServer (JSON-RPC)
│   │   └── file.py               # Wire 持久化
│   │
│   ├── ui/                       # UI 实现
│   │   ├── shell/                # 交互式 Shell UI
│   │   ├── print/                # 简单打印 UI
│   │   └── acp/                  # ACP 服务器 UI
│   │
│   ├── agents/                   # 内置 Agent 配置
│   │   ├── default/              # 默认 Agent
│   │   │   ├── agent.yaml        # Agent 配置
│   │   │   └── system.md         # System Prompt
│   │   └── okabe/                # Okabe Agent (D-Mail)
│   │
│   └── skill/                    # Skill 系统
│       ├── __init__.py           # Skill 发现 + 加载
│       └── flow/                 # Flow Skill 解析
│
├── packages/                     # 独立包
│   ├── kosong/                   # LLM 抽象框架 ★
│   │   └── src/kosong/
│   │       ├── chat_provider/    # ChatProvider 协议
│   │       ├── message.py        # 统一消息结构
│   │       └── tooling/          # 工具抽象
│   │
│   ├── kaos/                     # OS 抽象层
│   │   └── src/kaos/
│   │       ├── path.py           # KaosPath
│   │       └── backends/         # local/ssh
│   │
│   └── kimi-code/                # Kimi Code SDK
│
├── docs/                         # 文档
├── tests/                        # 单元测试
└── tests_e2e/                    # E2E 测试
```

## 核心模块职责

### 1. CLI 入口层

**cli.py** - Typer CLI 框架入口

```python
# src/kimi_cli/cli.py:45-67
@app.command()
def main(
    command: Annotated[str | None, typer.Argument()] = None,
    *,
    model: Annotated[str | None, typer.Option("--model", "-m")] = None,
    yolo: Annotated[bool, typer.Option("--yolo", "-y")] = False,
    agent: Annotated[str | None, typer.Option("--agent", "-a")] = None,
    # ... 更多参数
):
    """Kimi CLI - Your AI-powered terminal assistant."""
    asyncio.run(_main(...))
```

职责：解析命令行参数，创建 KimiCLI 实例，启动主循环。

### 2. 业务层 (Soul)

**KimiSoul** - Agent 核心循环

```python
# src/kimi_cli/soul/kimisoul.py:89-120
class KimiSoul:
    def __init__(
        self,
        agent: Agent,
        *,
        context: Context | None = None,
        loop_control: LoopControl | None = None,
    ):
        self._agent = agent
        self._runtime = agent.runtime
        self._context = context or Context(...)
        self._loop_control = loop_control or LoopControl()
        self._denwa_renji = self._runtime.denwa_renji
        self._approval = self._runtime.approval
```

职责：执行 Agent 循环，管理 Context，处理工具调用，协调 D-Mail。

**Runtime** - 运行时依赖容器

```python
# src/kimi_cli/soul/agent.py:78-110
@dataclass
class Runtime:
    config: Config
    oauth: OAuthManager
    llm: LLM | None
    session: Session
    builtin_args: BuiltinSystemPromptArgs
    denwa_renji: DenwaRenji
    approval: Approval
    labor_market: LaborMarket
    environment: Environment
    skills: SkillRegistry
```

职责：聚合所有运行时依赖，支持 subagent 的 Runtime 克隆。

### 3. 协议层 (Wire)

**Wire** - 事件流抽象

```python
# src/kimi_cli/wire/__init__.py:18-35
class Wire:
    def __init__(self, *, file_backend: WireFile | None = None):
        self._raw_queue = WireMessageQueue()      # 原始消息
        self._merged_queue = WireMessageQueue()   # 合并后消息
        self._soul_side = WireSoulSide(...)
        self._recorder = _WireRecorder(...)
```

职责：解耦 Soul 和 UI，支持事件持久化和回放。

### 4. 框架层 (Kosong)

**ChatProvider** - LLM 统一接口

```python
# packages/kosong/src/kosong/chat_provider/__init__.py:10-56
@runtime_checkable
class ChatProvider(Protocol):
    name: str

    @property
    def model_name(self) -> str: ...

    async def generate(
        self,
        system_prompt: str,
        tools: Sequence[Tool],
        history: Sequence[Message],
    ) -> "StreamedMessage": ...
```

职责：定义 LLM 调用契约，隐藏厂商差异。

## 数据流分析

### 用户输入到 Agent 响应

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  User   │────►│  Shell  │────►│  Wire   │────►│  Soul   │
│  Input  │     │   UI    │     │ TurnBegin│    │ _step() │
└─────────┘     └─────────┘     └─────────┘     └────┬────┘
                                                     │
                     ┌───────────────────────────────┘
                     │
              ┌──────▼──────┐
              │   Kosong    │
              │  generate() │
              └──────┬──────┘
                     │
              ┌──────▼──────┐
              │ ChatProvider│
              │  (Kimi/...)│
              └──────┬──────┘
                     │
              ┌──────▼──────┐     ┌─────────┐
              │  ToolCall   │────►│ Toolset │
              │  (if any)   │     │ handle()│
              └──────┬──────┘     └────┬────┘
                     │                 │
                     │◄────────────────┘
                     │
              ┌──────▼──────┐
              │   Context   │
              │ grow_context│
              └──────┬──────┘
                     │
              ┌──────▼──────┐     ┌─────────┐     ┌─────────┐
              │    Wire     │────►│  Shell  │────►│  User   │
              │  TextPart   │     │   UI    │     │  Output │
              └─────────────┘     └─────────┘     └─────────┘
```

### 工具调用流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Tool Call Flow                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. LLM 返回 ToolCall                                           │
│     ┌─────────────┐                                             │
│     │ kosong.step │                                             │
│     │ on_tool_call│                                             │
│     └──────┬──────┘                                             │
│            │                                                    │
│  2. Toolset 分发                                                │
│     ┌──────▼──────┐                                             │
│     │ KimiToolset │                                             │
│     │   handle()  │                                             │
│     └──────┬──────┘                                             │
│            │                                                    │
│  3. 依赖注入 + 实例化                                            │
│     ┌──────▼──────┐                                             │
│     │  _load_tool │                                             │
│     │ inspect.sig │                                             │
│     └──────┬──────┘                                             │
│            │                                                    │
│  4. Approval 检查                                               │
│     ┌──────▼──────┐     ┌─────────────┐                         │
│     │   Tool      │────►│  Approval   │                         │
│     │  __call__   │     │  request()  │                         │
│     └──────┬──────┘     └──────┬──────┘                         │
│            │                   │                                │
│            │◄──────────────────┘ (approved/rejected)            │
│            │                                                    │
│  5. 执行 + 返回结果                                              │
│     ┌──────▼──────┐                                             │
│     │ ToolResult  │                                             │
│     │ ToolOk/Err  │                                             │
│     └─────────────┘                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 设计决策分析

### 为什么用 Protocol 而非 ABC？

**现象层**：ChatProvider 是 Protocol，不是 ABC。

**本质层**：Protocol 实现结构化子类型（structural subtyping），只要实现了相同方法签名就满足协议，无需显式继承。

**哲学层**：这是 Python 的"鸭子类型"哲学——"如果它走起来像鸭子，叫起来像鸭子，那它就是鸭子"。Protocol 让这种哲学有了类型检查的支持。

```python
# 不需要继承，只要实现相同方法
class MyProvider:  # 没有继承 ChatProvider
    name = "my"

    @property
    def model_name(self) -> str:
        return "my-model"

    async def generate(self, ...): ...

# 类型检查通过
provider: ChatProvider = MyProvider()  # OK
```

### 为什么 Wire 需要两个队列？

**现象层**：Wire 有 `_raw_queue` 和 `_merged_queue` 两个队列。

**本质层**：解决流式输出的性能与完整性矛盾。
- `raw_queue`：保证每个消息片段都能被实时消费（Shell UI 需要逐字渲染）
- `merged_queue`：减少消息数量，降低持久化开销（wire.jsonl 文件大小）

**哲学层**：这是"时间换空间"的经典权衡。Shell UI 需要低延迟（merge=False），而 WireRecorder 需要低存储（merge=True）。

### 为什么 Runtime 需要克隆？

**现象层**：`copy_for_fixed_subagent()` 和 `copy_for_dynamic_subagent()` 两个方法。

**本质层**：隔离 subagent 的状态，避免污染主 Agent。

```python
# src/kimi_cli/soul/agent.py:125-153
def copy_for_fixed_subagent(self) -> Runtime:
    return Runtime(
        # 共享的
        config=self.config,
        llm=self.llm,
        session=self.session,
        # 独立的
        denwa_renji=DenwaRenji(),      # 独立时间线
        labor_market=LaborMarket(),    # 独立人才市场
        approval=self.approval.share(), # 共享批准状态
    )
```

**哲学层**：这是"组合优于继承"的体现。通过克隆 + 选择性共享，实现灵活的隔离策略。

## 与其他 Agent 框架对比

| 维度 | kimi-cli | LangChain | AutoGen |
|------|----------|-----------|---------|
| **LLM 抽象** | Kosong (Protocol) | BaseLLM (ABC) | OpenAI SDK |
| **工具系统** | 依赖注入 | 装饰器 | 函数签名 |
| **Multi-Agent** | LaborMarket | AgentExecutor | GroupChat |
| **事件流** | Wire 协议 | Callbacks | 无 |
| **状态管理** | Context + Checkpoint | Memory | 无 |
| **时间旅行** | D-Mail | 无 | 无 |

**kimi-cli 的独特之处**：
1. **Protocol-based 设计**：更灵活，更易测试
2. **Wire 协议**：彻底解耦，支持多种 UI
3. **D-Mail**：业界唯一的时间旅行机制

---

## 章节衔接

**本章回顾**：
- 我们学习了 kimi-cli 的整体架构设计
- 关键收获：分层解耦（UI/Wire/Soul/Kosong）、Protocol 优于 ABC、Wire 双队列设计

**下一章预告**：
- 在 `02-kosong.md` 中，我们将学习 Kosong LLM 框架
- 为什么需要学习：Kosong 是 kimi-cli 的基础设施，理解它才能理解 Agent 如何调用 LLM
- 关键问题：如何设计 vendor-agnostic 的 LLM 接口？如何处理不同厂商的协议差异？
