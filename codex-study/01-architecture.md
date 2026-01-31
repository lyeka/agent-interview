# 第一章：整体架构

## 1.1 项目结构概览

Codex CLI 采用 **Rust + TypeScript 混合架构**，核心引擎用 Rust 实现高性能和内存安全，CLI 包装用 TypeScript 实现跨平台分发。

```
codex/
├── codex-rs/                    # Rust 核心引擎（主体）
│   ├── core/                    # Agent 核心逻辑
│   ├── tui/                     # 终端 UI (Ratatui)
│   ├── cli/                     # CLI 入口
│   ├── mcp-server/              # MCP 服务器实现
│   ├── backend-client/          # OpenAI API 客户端
│   ├── exec-server/             # 隔离命令执行服务
│   ├── linux-sandbox/           # Linux Landlock 沙箱
│   ├── protocol/                # 协议定义
│   ├── state/                   # 状态管理
│   ├── skills/                  # Skill 加载器
│   └── [40+ 其他模块]
├── codex-cli/                   # TypeScript CLI 包装
└── BUILD.bazel                  # Bazel 构建配置
```

### 为什么选择 Rust？

这不是偶然的技术选型，而是深思熟虑的架构决策：

1. **内存安全**：Agent 需要长时间运行，Rust 的所有权系统避免内存泄漏
2. **并发性能**：工具调用需要并行执行，Rust 的 async/await + Tokio 提供零成本抽象
3. **跨平台沙箱**：沙箱实现需要系统级编程，Rust 的 FFI 能力和条件编译完美适配
4. **二进制分发**：单一可执行文件，无运行时依赖

---

## 1.2 核心模块职责

### core/ - Agent 核心引擎

这是整个项目的心脏，包含 83 个源文件，核心逻辑集中在 `codex.rs`（6094 行）。

```
core/src/
├── codex.rs                 # 主 Agent 循环（6094 行，需要重构）
├── agent/                   # Multi-Agent 控制
│   └── control.rs           # AgentControl 实现
├── tools/                   # 工具系统
│   ├── spec.rs              # 工具规范定义（2638 行）
│   ├── router.rs            # 工具路由（191 行）
│   ├── registry.rs          # 工具注册表（234 行）
│   ├── orchestrator.rs      # 审批+沙箱编排（173 行）
│   ├── sandboxing.rs        # 沙箱审批机制（329 行）
│   ├── parallel.rs          # 并行执行运行时（138 行）
│   └── handlers/            # 16 个工具实现
├── context_manager/         # 上下文管理
│   ├── history.rs           # 历史消息管理
│   ├── truncate.rs          # 截断策略
│   ├── compact.rs           # 自动压缩
│   └── normalize.rs         # 历史规范化
├── mcp/                     # MCP 客户端
├── skills/                  # Skill 系统
├── exec_policy.rs           # 执行策略引擎（1390 行）
├── seatbelt.rs              # macOS 沙箱（624 行）
└── windows_sandbox.rs       # Windows 沙箱（148 行）
```

**代码坏味道警告**：`codex.rs` 达到 6094 行，严重违反"每文件不超过 800 行"的原则。这是一个典型的"上帝类"反模式，应该拆分为多个职责单一的模块。

### protocol/ - 协议定义

定义了 Agent 与外部世界交互的所有数据结构：

```rust
// protocol/src/models.rs
pub struct Prompt {
    pub input: Vec<ResponseItem>,           // 对话历史
    pub tools: Vec<ToolSpec>,               // 可用工具
    pub parallel_tool_calls: bool,
    pub base_instructions: BaseInstructions, // 系统指令
    pub personality: Option<Personality>,    // 人格选项
    pub output_schema: Option<Value>,
}
```

### backend-client/ - OpenAI API 客户端

封装了与 OpenAI API 的通信，支持 SSE 流式响应：

```rust
// backend-client/src/lib.rs
pub struct ModelClientSession {
    client: reqwest::Client,
    config: ClientConfig,
}

impl ModelClientSession {
    pub fn stream(&mut self, prompt: &Prompt) -> impl Stream<Item = ResponseEvent>;
}
```

---

## 1.3 数据流架构

Codex 采用**队列对模式**（Submission/Event Queue Pair）实现异步通信：

```
┌─────────────┐     Submission      ┌─────────────┐
│   User      │ ──────────────────► │   Codex     │
│  Interface  │                     │   Engine    │
│  (TUI/CLI)  │ ◄────────────────── │             │
└─────────────┘       Event         └─────────────┘
```

### 为什么用队列对模式？

传统的请求-响应模式无法满足 Agent 的需求：

1. **长时间运行**：一次 Turn 可能执行多个工具调用，持续数分钟
2. **流式输出**：LLM 响应是流式的，需要实时推送给用户
3. **中断支持**：用户可以随时中断正在执行的操作
4. **事件驱动**：工具执行、审批请求、进度更新都是异步事件

队列对模式完美解决了这些问题：

```rust
// codex.rs:2386-2509
async fn submission_loop(sess: Arc<Session>, config: Arc<Config>, rx_sub: Receiver<Submission>) {
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op.clone() {
            Op::Interrupt => handlers::interrupt(&sess).await,
            Op::UserInput { .. } => handlers::user_input_or_turn(&sess, ...).await,
            Op::ExecApproval { id, decision } => handlers::exec_approval(&sess, id, decision).await,
            Op::Shutdown => {
                if handlers::shutdown(&sess, sub.id.clone()).await {
                    break; // 唯一退出点
                }
            }
            // ... 其他操作
        }
    }
}
```

---

## 1.4 会话状态管理

### Session 结构

`Session` 是 Agent 的核心状态容器，管理单个会话的所有状态：

```rust
// 概念模型（实际实现分散在多个文件）
pub struct Session {
    // 配置
    config: Arc<Config>,

    // 历史管理
    history: RwLock<ContextManager>,

    // 工具系统
    tool_router: Arc<ToolRouter>,
    tool_approvals: Mutex<ApprovalStore>,

    // MCP 连接
    mcp_manager: McpConnectionManager,

    // 事件发送
    event_sender: Sender<Event>,
}
```

### TurnContext 结构

`TurnContext` 封装单次 Turn 的上下文，生命周期短于 Session：

```rust
pub struct TurnContext {
    // 沙箱配置
    pub sandbox_policy: SandboxPolicy,
    pub cwd: PathBuf,

    // 工具门控
    pub tool_call_gate: ToolCallGate,

    // 取消令牌
    pub cancellation_token: CancellationToken,

    // 客户端会话
    pub client: ModelClientSession,
}
```

**设计哲学**：Session 是"长寿"对象，TurnContext 是"短寿"对象。这种分离让状态管理更清晰，也便于实现 Turn 级别的取消和重试。

---

## 1.5 模块依赖关系

```
                    ┌─────────────┐
                    │   protocol  │  ← 纯数据定义，无依赖
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│backend-client│   │    state    │   │   skills    │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         ▼
                  ┌─────────────┐
                  │    core     │  ← 核心引擎，依赖上述所有模块
                  └──────┬──────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
  │     tui     │ │     cli     │ │ mcp-server  │
  └─────────────┘ └─────────────┘ └─────────────┘
```

**依赖原则**：
1. **protocol 无依赖**：纯数据定义，可被任何模块引用
2. **core 是中心**：依赖所有基础模块，被所有接口模块依赖
3. **接口模块互不依赖**：tui、cli、mcp-server 是平行的

---

## 1.6 构建系统

Codex 使用 **Bazel + Cargo 双构建系统**：

- **Bazel**：用于 CI/CD 和跨平台构建
- **Cargo**：用于本地开发和调试

```bash
# Bazel 构建
bazel build //codex-rs/cli:codex

# Cargo 构建
cd codex-rs && cargo build --release
```

**为什么用双构建系统？**

1. **Bazel 优势**：增量构建、远程缓存、跨平台一致性
2. **Cargo 优势**：Rust 生态原生支持、IDE 集成、快速迭代
3. **权衡**：CI 用 Bazel 保证一致性，开发用 Cargo 保证效率

---

## 1.7 技术选型对比

| 组件 | Codex 选择 | 替代方案 | 选择理由 |
|------|-----------|----------|----------|
| **核心语言** | Rust | Go/Python | 内存安全 + 性能 + 跨平台沙箱 |
| **异步运行时** | Tokio | async-std | 生态成熟、性能优秀 |
| **TUI 框架** | Ratatui | tui-rs | 活跃维护、API 友好 |
| **序列化** | serde | 手写 | Rust 生态标准 |
| **HTTP 客户端** | reqwest | hyper | 高层抽象、易用 |
| **构建系统** | Bazel+Cargo | 纯 Cargo | CI 一致性 + 开发效率 |

---

## 1.8 与 Claude Code 的架构对比

| 维度 | Codex CLI | Claude Code |
|------|-----------|-------------|
| **核心语言** | Rust | TypeScript |
| **运行环境** | 本地 | 本地 |
| **LLM 后端** | OpenAI API | Anthropic API |
| **沙箱实现** | 原生（Seatbelt/Landlock/AppContainer） | 依赖 Docker |
| **MCP 支持** | 客户端 + 服务器 | 客户端 |
| **Multi-Agent** | 原生支持 | 通过 Task 工具 |
| **上下文压缩** | 自动 handoff summary | 自动摘要 |

**关键差异**：Codex 选择 Rust 实现原生沙箱，而 Claude Code 依赖 Docker。这反映了不同的设计哲学——Codex 追求"零依赖"的本地体验，Claude Code 追求"容器化"的隔离保证。

---

## 章节衔接

**本章回顾**：
- 我们分析了 Codex CLI 的整体架构
- 关键收获：Rust 核心引擎 + 队列对模式 + 分层模块设计

**下一章预告**：
- 在 `02-agent-loop.md` 中，我们将深入 Agent 主循环的实现
- 为什么需要学习：Agent Loop 是整个系统的心跳，理解它才能理解 Agent 的行为
- 关键问题：`run_turn()` 如何协调 LLM 调用和工具执行？如何处理中断和错误？
