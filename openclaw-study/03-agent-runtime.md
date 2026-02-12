# Pi Agent 运行时与会话模型

OpenClaw 的智能层构建在 Pi SDK（`@mariozechner/pi-agent-core`, `@mariozechner/pi-coding-agent`）之上。Pi Agent 以嵌入模式（embedded mode）运行在 Gateway 进程内部，通过事件订阅与 Gateway 通信。这不是一个 HTTP RPC 调用——Agent 和 Gateway 共享同一个进程空间。

## Agent 执行模型

### 嵌入模式启动

Agent 的启动入口在 `src/agents/pi-embedded-runner/run/attempt.ts`：

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:479-491
const { session } = await createAgentSession({
  cwd: resolvedWorkspace,
  agentDir,
  authStorage: params.authStorage,
  modelRegistry: params.modelRegistry,
  model: params.model,
  thinkingLevel: mapThinkingLevel(params.thinkLevel),
  tools: builtInTools,
  customTools: allCustomTools,
  sessionManager,
  settingsManager,
});
```

`createAgentSession` 是 Pi SDK 提供的核心函数，它创建一个 Agent 会话，配置模型、工具、工作空间等参数。

### 事件订阅

创建 session 后，Gateway 通过订阅模式监听 Agent 的所有事件：

```typescript
// src/agents/pi-embedded-subscribe.ts:602
const unsubscribe = params.session.subscribe(
  createEmbeddedPiSessionEventHandler(ctx)
);
```

事件处理器（`src/agents/pi-embedded-subscribe.handlers.ts`）处理以下事件类型：

| 事件 | 触发时机 | Gateway 动作 |
|------|---------|-------------|
| `agent_start` | Agent 开始处理 | 广播开始事件 |
| `message_start` | 开始生成回复 | 广播流式起始 |
| `message_update` | 生成中（delta） | 广播流式增量 |
| `message_end` | 回复完成 | 广播最终消息 |
| `tool_execution_start` | 开始调用工具 | 广播工具启动 |
| `tool_execution_update` | 工具执行中 | 广播工具进度 |
| `tool_execution_end` | 工具执行完成 | 广播工具结果 |
| `auto_compaction_start` | 开始上下文压缩 | 日志记录 |
| `auto_compaction_end` | 压缩完成 | 日志记录 |

### 消息处理完整流程

一条用户消息从接收到回复的完整路径：

```
1. 通道收到消息（如 WhatsApp）
   │
2. resolveAgentRoute() → 确定 agentId + sessionKey
   │
3. agentCommand() — 构建 Agent 执行参数
   │  - 加载 workspace 配置
   │  - 解析模型偏好
   │  - 收集工具集
   │  - 加载会话历史
   │
4. createAgentSession() — Pi SDK 创建会话
   │
5. session.subscribe() — 订阅事件流
   │
6. streamSimple() — 发送用户消息并启动 Agent 循环
   │
7. Pi Agent ReAct 循环：
   │  a. 思考（LLM 推理）
   │  b. 决策（调用工具 or 回复用户）
   │  c. 如果调用工具：
   │     - tool_execution_start → 执行 → tool_execution_end
   │     - 将结果加入上下文
   │     - 回到 (a)
   │  d. 如果回复用户：
   │     - message_start → delta... → message_end
   │
8. emitAgentEvent() → createAgentEventHandler()
   │
9. broadcast("agent") + broadcast("chat")
   │  → 推送给 WebSocket 客户端
   │
10. resolveOutboundTarget() → deliverOutboundPayloads()
    → 发送回原始通道（WhatsApp）
```

## 会话模型

### 会话键（Session Key）

每个对话上下文由一个唯一的会话键标识。键的格式编码了完整的路由信息：

```
主会话：  agent:{agentId}:{mainKey}
DM 会话：agent:{agentId}:{channel}:{accountId}:dm:{peerId}
群组会话：agent:{agentId}:{channel}:{accountId}:group:{groupId}
```

主会话（main session）是用户与 Agent 的直接对话。群组会话和 DM 会话分别对应群组消息和私聊消息。这种分离确保了上下文不会混淆——你在 Telegram 群里讨论的内容不会污染 WhatsApp 私聊的上下文。

### 会话生命周期

```
创建 → 活跃 → [压缩] → [重置]
 │                │          │
 │   消息积累     │  /compact │  /reset
 │   工具调用     │  命令触发  │  命令触发
 │                │          │
 │                ▼          ▼
 │         上下文缩减   上下文清空
 │         保留摘要     全新开始
```

会话的核心操作：
- **创建**：首条消息到达时自动创建
- **恢复**：后续消息到达时加载历史上下文
- **压缩**（compact）：当上下文接近 Token 上限时，生成摘要替代完整历史
- **重置**（reset）：清空上下文，重新开始

### 会话补丁（Session Patch）

会话支持运行时修改属性，无需重置：

```typescript
// 可通过 sessions.patch 方法修改的属性
{
  thinkingLevel: "off" | "minimal" | "low" | "medium" | "high",
  verboseLevel: "on" | "off",
  model: "anthropic/claude-opus-4-6",
  sendPolicy: "auto" | "manual",
  groupActivation: "mention" | "always",
}
```

这些修改通过 Gateway 的 `sessions.patch` RPC 方法即时生效，不影响已有的对话历史。

## 工具系统

### 工具注册

工具分为两类：

**内置工具**（Pi SDK 提供）：
- `bash` — 执行 Shell 命令
- `read` / `write` / `edit` — 文件操作
- `browser` — Chrome/Chromium 控制
- `process` — 进程管理

**插件工具**（通过 Plugin SDK 注册）：
- `memory_search` / `memory_get` — 记忆搜索
- `memory_recall` / `memory_store` / `memory_forget` — LanceDB 记忆
- `llm_task` — LLM 子任务
- 以及各通道特有的工具（discord_reply, slack_reply 等）

### 工具执行流程

当 Agent 决定调用工具时：

```
Agent 输出 tool_call → Gateway 拦截
    │
    ├── before_tool_call 钩子（插件可修改参数）
    │
    ├── 执行工具
    │   ├── 内置工具：Pi SDK 直接执行
    │   └── 插件工具：调用注册的 execute 函数
    │
    ├── after_tool_call 钩子（插件可修改结果）
    │
    └── tool_result_persist 钩子（同步，用于持久化）
```

插件钩子的执行顺序由 `priority` 控制，数字越大越先执行。`before_tool_call` 是串行执行的（modifying hook），因为后一个钩子可能依赖前一个钩子的修改结果。`agent_end` 是并行执行的（void hook），因为它们之间没有依赖。

## Agent 工作空间

Agent 的工作空间（workspace）是一个文件系统目录，默认位于 `~/.openclaw/workspace`。它包含：

```
~/.openclaw/workspace/
├── AGENTS.md      — Agent 的系统级指令
├── SOUL.md        — Agent 的人格定义
├── TOOLS.md       — 可用工具的说明
├── MEMORY.md      — 主动记忆文件（memory-core 的数据源）
├── memory/        — 记忆子目录
├── skills/        — 本地 skills
└── ...            — 用户自定义文件
```

`AGENTS.md`、`SOUL.md`、`TOOLS.md` 会被自动注入到 Agent 的 System Prompt 中。这是一种"文件即配置"的设计——用户通过编辑文件来定制 Agent 的行为，而不是通过复杂的配置 UI。

## 多 Agent 路由

OpenClaw 支持在同一个 Gateway 上运行多个 Agent 实例。每个 Agent 有独立的：
- 工作空间目录
- 会话集合
- 模型配置
- 工具权限

通过 `config.bindings` 配置将不同的通道/账号/对话方路由到不同的 Agent：

```json5
{
  bindings: [
    { agentId: "work", match: { channel: "slack" } },
    { agentId: "personal", match: { channel: "whatsapp" } },
    { agentId: "community", match: { channel: "discord", guildId: "123" } },
  ]
}
```

这种架构使得"工作助手"和"生活助手"可以完全隔离运行，同时共享 Gateway 的基础设施。

## Agent 间通信

OpenClaw 提供了 `sessions_*` 工具族实现 Agent 间的消息传递：

- `sessions_list` — 发现活跃的会话（Agent）
- `sessions_history` — 获取另一个会话的对话记录
- `sessions_send` — 向另一个会话发送消息

这不是一个完整的 Multi-Agent 框架，但它提供了最基础的协调能力。一个 Agent 可以让另一个 Agent 帮忙完成特定任务，然后接收结果。

## 流式输出

Agent 的回复通过流式方式推送，包含两种流：

**Block 流**（用于文本回复）：
```
message_start → message_update (delta) → ... → message_end
```

**Tool 流**（用于工具调用）：
```
tool_execution_start → tool_execution_update → tool_execution_end
```

`EmbeddedBlockChunker` 负责将 Pi SDK 的原始事件聚合成适合传输的块。小的 delta 会被合并，避免产生过多的小包。

---

## 章节衔接

**本章回顾**：
- 我们学习了 Pi Agent 的嵌入执行模型、会话生命周期和工具系统
- 关键收获：Agent 与 Gateway 共进程运行的事件驱动架构，以及"文件即配置"的工作空间设计

**下一章预告**：
- 在 `04-memory-system.md` 中，我们将深入 Memory 系统设计
- 为什么需要学习：Memory 是 Agent 的"长期记忆"，决定了 Agent 能否跨会话保持连贯性
- 关键问题：混合搜索算法、增量索引策略、双层记忆架构的设计权衡
