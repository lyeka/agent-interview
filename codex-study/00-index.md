# OpenAI Codex CLI 深度学习指南

> **分析版本**: `8660ad6c64895fa0a0aea05c3935d9d3828763d7` (main)
> **分析日期**: 2026-01-31
> **仓库地址**: https://github.com/openai/codex

## 项目定位

Codex CLI 是 OpenAI 官方出品的**本地运行 AI Coding Agent**，通过命令行界面提供代码生成、执行、调试等能力。它不是简单的 LLM 包装器，而是一个完整的 Agent 系统，具备工具调用、沙箱隔离、多 Agent 协作、上下文管理等核心能力。

**技术栈**：
- **核心引擎**: Rust（codex-rs/，6000+ 行核心逻辑）
- **CLI 包装**: Node.js/TypeScript（npm 分发）
- **构建系统**: Bazel + Cargo
- **TUI 框架**: Ratatui 0.29.0
- **异步运行时**: Tokio
- **协议标准**: MCP (Model Context Protocol)

---

## 章节目录

| 章节 | 文件 | 核心内容 |
|------|------|----------|
| 01 | [01-architecture.md](./01-architecture.md) | 整体架构、模块划分、技术选型 |
| 02 | [02-agent-loop.md](./02-agent-loop.md) | Agent 主循环、Turn 执行、采样请求处理 |
| 03 | [03-tools-system.md](./03-tools-system.md) | 工具系统、路由分发、并行执行 |
| 04 | [04-context-management.md](./04-context-management.md) | 上下文管理、Token 估算、自动压缩 |
| 05 | [05-prompt-engineering.md](./05-prompt-engineering.md) | Prompt 模板、人格系统、协作模式 |
| 06 | [06-sandbox-security.md](./06-sandbox-security.md) | 跨平台沙箱、执行策略、审批流程 |
| 07 | [07-mcp-integration.md](./07-mcp-integration.md) | MCP 客户端/服务器、工具转换、资源访问 |
| 08 | [08-deep-dive.md](./08-deep-dive.md) | 核心源码深度解析 |
| 09 | [09-interview-qa.md](./09-interview-qa.md) | 面试 QA（15 题） |
| 10 | [10-summary.md](./10-summary.md) | 知识图谱总结 |

---

## 学习路径

### 新手路径（2-3 小时）

1. **01-architecture.md** - 建立整体认知
2. **02-agent-loop.md** - 理解 Agent 核心循环
3. **09-interview-qa.md** - 快速掌握面试要点

### 进阶路径（4-6 小时）

1. 完成新手路径
2. **03-tools-system.md** - 深入工具系统设计
3. **04-context-management.md** - 理解上下文管理策略
4. **05-prompt-engineering.md** - 学习 Prompt 工程实践
5. **06-sandbox-security.md** - 掌握安全机制

### 深度路径（8+ 小时）

1. 完成进阶路径
2. **07-mcp-integration.md** - 理解 MCP 协议集成
3. **08-deep-dive.md** - 源码级深度分析
4. **10-summary.md** - 构建完整知识图谱

---

## 核心术语速查

### Agent 核心概念

| 术语 | 一句话定义 | 生活类比 | 在本项目中 |
|------|-----------|----------|-----------|
| **Agent Loop** | Agent 持续接收输入、调用工具、返回结果的循环过程 | 餐厅服务员：接单→下单→上菜→等待下一桌 | `codex.rs:run_turn()` 实现的 Sampling-Tool-Execution 循环 |
| **Turn** | Agent 处理一次用户输入的完整过程 | 一局棋：从落子到对方响应 | `TurnContext` 封装单次交互的所有状态 |
| **Sampling Request** | 向 LLM 发送的推理请求 | 向专家咨询问题 | `run_sampling_request()` 构建并发送请求 |
| **Tool Call** | Agent 调用外部工具执行操作 | 厨师使用厨具 | `ToolRouter.dispatch_tool_call()` 分发执行 |

### 上下文管理

| 术语 | 一句话定义 | 生活类比 | 在本项目中 |
|------|-----------|----------|-----------|
| **Context Window** | LLM 单次能处理的最大 Token 数 | 工作台面积：放不下就得清理 | `estimate_token_count()` 实时追踪使用量 |
| **Auto Compact** | Token 超限时自动压缩历史 | 整理书桌：保留重要文件，归档旧资料 | `run_auto_compact()` 生成 handoff summary |
| **Truncation** | 截断过长的输出 | 摘要：保留开头结尾，省略中间 | `TruncationPolicy` 定义截断策略 |
| **Ghost Snapshot** | Git 仓库状态快照 | 游戏存档：支持回退 | `GhostCommit` 记录 undo 点 |

### 安全机制

| 术语 | 一句话定义 | 生活类比 | 在本项目中 |
|------|-----------|----------|-----------|
| **Sandbox** | 隔离执行环境，限制程序权限 | 儿童游乐场围栏：玩耍但不能跑出去 | macOS Seatbelt / Linux Landlock / Windows AppContainer |
| **Approval Policy** | 危险操作需用户确认的策略 | 银行转账需短信验证 | `AskForApproval` 枚举定义审批时机 |
| **Exec Policy** | 命令执行的白名单/黑名单规则 | 门禁系统：谁能进、谁不能进 | `ExecPolicyManager` 管理 `.rules` 文件 |

### Prompt 工程

| 术语 | 一句话定义 | 生活类比 | 在本项目中 |
|------|-----------|----------|-----------|
| **Base Instructions** | Agent 的核心能力定义 | 员工手册：职责、流程、规范 | `default.md` 定义 Agent 身份和工作流 |
| **Personality** | Agent 的交互风格 | 客服态度：热情 vs 专业 | `friendly.md` / `pragmatic.md` 两种人格 |
| **Collaboration Mode** | Agent 的工作模式 | 工作方式：独立完成 vs 结对编程 | `plan/execute/pair_programming/code` 四种模式 |
| **AGENTS.md** | 项目特定的 Agent 指令 | 项目规范文档 | 分层覆盖，深层优先 |

### MCP 协议

| 术语 | 一句话定义 | 生活类比 | 在本项目中 |
|------|-----------|----------|-----------|
| **MCP** | Model Context Protocol，工具调用标准协议 | USB 接口：统一不同设备的连接方式 | `McpConnectionManager` 管理多个 MCP 服务器 |
| **Tool Qualification** | 工具名称加前缀避免冲突 | 姓名+公司：区分同名的人 | `mcp__<server>__<tool>` 格式 |
| **Resource** | MCP 服务器提供的数据资源 | 图书馆藏书：可查询、可借阅 | `list_all_resources()` 聚合所有服务器资源 |

---

## 架构速览

```
┌─────────────────────────────────────────────────��───────────────┐
│                         Codex CLI                                │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   TUI       │  │   CLI       │  │ MCP Server  │   Interfaces │
│  │  (Ratatui)  │  │  (clap)     │  │  (stdio)    │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
│         └────────────────┼────────────────┘                      │
│                          ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      Core Engine                           │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │  │
│  │  │   Codex     │  │   Session   │  │ TurnContext │        │  │
│  │  │ (队列对模式) │  │ (会话状态)  │  │ (Turn 上下文)│        │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │  │
│  │         │                │                │                │  │
│  │         └────────────────┼────────────────┘                │  │
│  │                          ▼                                 │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │                   Agent Loop                         │  │  │
│  │  │  submission_loop() → run_turn() → run_sampling_request│  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          │                                       │
│         ┌────────────────┼────────────────┐                      │
│         ▼                ▼                ▼                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Tools     │  │   Context   │  │   Sandbox   │   Subsystems │
│  │  System     │  │  Manager    │  │   System    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│         │                │                │                      │
│         ▼                ▼                ▼                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ ToolRouter  │  │ History     │  │ Seatbelt    │              │
│  │ ToolRegistry│  │ Truncate    │  │ Landlock    │              │
│  │ Orchestrator│  │ Compact     │  │ AppContainer│              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Backend     │  │    MCP      │  │   Exec      │   External   │
│  │ Client      │  │  Servers    │  │  Server     │   Services   │
│  │ (OpenAI API)│  │ (stdio/SSE) │  │ (隔离执行)  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 章节衔接

**本章回顾**：
- 我们建立了对 Codex CLI 的整体认知
- 关键收获：这是一个 Rust 实现的本地 AI Coding Agent，具备完整的工具系统、沙箱隔离、上下文管理能力

**下一章预告**：
- 在 `01-architecture.md` 中，我们将深入分析项目的整体架构
- 为什么需要学习：理解架构是深入源码的前提
- 关键问题：Codex 如何组织 6000+ 行 Rust 代码？模块间如何协作？
