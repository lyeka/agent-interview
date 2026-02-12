# OpenClaw 整体架构设计

OpenClaw 是一个本地优先（local-first）的多通道个人 AI 助手网关平台。它不是一个 SaaS 产品，而是运行在用户自己设备上的控制平面，将 AI 能力统一投射到用户已有的消息通道中：WhatsApp、Telegram、Slack、Discord、Signal、iMessage、WebChat 等。

这个架构选择本身就值得深思——它将"助手"从单一聊天界面解放出来，变成了一个无处不在的代理。

## 架构总览

```
WhatsApp / Telegram / Slack / Discord / Signal / iMessage / WebChat / ...
                 │
                 ▼
  ┌──────────────────────────────┐
  │     Gateway (控制平面)        │
  │   ws://127.0.0.1:18789       │
  │                              │
  │  ┌─────────┐  ┌───────────┐  │
  │  │ WebSocket│  │  HTTP API │  │
  │  │ Server  │  │ (Express) │  │
  │  └────┬────┘  └─────┬─────┘  │
  │       │              │        │
  │  ┌────┴──────────────┴────┐   │
  │  │    RPC Method Layer    │   │
  │  │   (agent/send/config/  │   │
  │  │    sessions/channels)  │   │
  │  └────────────┬───────────┘   │
  │               │               │
  │  ┌────────────┴───────────┐   │
  │  │  Event Broadcast Layer │   │
  │  └────────────────────────┘   │
  └──────────┬───────────────────┘
             │
    ┌────────┼────────────┐
    ▼        ▼            ▼
┌────────┐ ┌──────┐  ┌────────┐
│ Agent  │ │ CLI  │  │ Nodes  │
│Runtime │ │Tools │  │iOS/Mac │
│(Pi SDK)│ │      │  │Android │
└────────┘ └──────┘  └────────┘
```

## 三层架构模型

OpenClaw 的架构可以拆解为三个清晰的层次：

### 第一层：通道层（Channel Layer）

通道层负责与外部消息平台对接。每个通道都是一个独立的插件，通过统一的 `ChannelPlugin` 接口接入 Gateway。

通道层的核心抽象是 Channel Dock（`src/channels/dock.ts`），它定义了每个通道的能力元数据：

```typescript
// src/channels/dock.ts — 通道能力声明
capabilities: {
  chatTypes: ["dm", "group"],   // 支持的聊天类型
  media: ["image", "audio"],    // 支持的媒体类型
  polls: false,                 // 是否支持投票
  threads: true,                // 是否支持线程
}
```

这个设计的关键在于：Gateway 不需要理解每个通道的具体协议，它只需要知道通道"能做什么"。Telegram 和 WhatsApp 的 SDK 完全不同，但它们在 Gateway 眼中是同构的。

**为什么这样设计？** 想象你要加一个新通道（比如 Zalo），你不需要修改 Gateway 的任何核心代码，只需要实现 `ChannelPlugin` 接口并注册即可。这是经典的开放-封闭原则。

### 第二层：控制层（Gateway / Control Plane）

Gateway 是整个系统的心脏。它运行在本地（`ws://127.0.0.1:18789`），通过 WebSocket 提供实时控制平面，通过 HTTP 提供 REST API。

Gateway 的核心职责：
1. **连接管理** — 认证、握手、心跳、角色分配
2. **消息路由** — 从通道到 Agent 的路由决策
3. **会话管理** — 创建、恢复、压缩、删除会话
4. **事件广播** — 将 Agent 事件推送给所有订阅的客户端
5. **状态协调** — 在 CLI、Web UI、移动端之间同步状态

Gateway 的 WebSocket 协议只有三种帧类型，极其简洁：

```typescript
// src/gateway/protocol/schema/frames.ts
type RequestFrame  = { type: "req",   id: string, method: string, params?: unknown };
type ResponseFrame = { type: "res",   id: string, ok: boolean, payload?: unknown };
type EventFrame    = { type: "event", event: string, payload?: unknown, seq?: number };
```

这三种帧覆盖了所有交互模式：请求-响应（RPC）和发布-订阅（事件）。没有第四种。

**设计意图**：三帧协议是对复杂性的极限压缩。Gateway 的协议不需要像 gRPC 那样复杂，因为它只服务于单用户场景。简单意味着可靠，可靠意味着它能在后台默默运行而不出问题。

### 第三层：智能层（Agent Runtime）

Agent 层使用 Pi SDK（`@mariozechner/pi-agent-core`）作为运行时。Pi Agent 以嵌入模式运行在 Gateway 进程中，通过事件订阅与 Gateway 通信。

Agent 的执行模型是经典的 ReAct 循环：

```
用户消息 → Agent 思考 → 工具调用 → 观察结果 → 再思考 → ... → 最终回复
```

工具系统包含两类：
- **内置工具**：bash、browser、canvas、cron 等
- **插件工具**：通过 Plugin SDK 注册的扩展工具（memory_search、memory_store 等）

## 消息路由：从通道到 Agent 的完整流程

路由是 OpenClaw 架构中最精巧的部分。当一条消息从 WhatsApp 进入系统，会经历以下路径：

```
1. WhatsApp 通道收到消息
   │
2. resolveAgentRoute() — 路由决策
   │  匹配顺序：peer → guild → team → account → channel → default
   │  输出：agentId + sessionKey
   │
3. buildAgentPeerSessionKey() — 构造会话键
   │  格式：agent:{agentId}:{channel}:{accountId}:{peerKind}:{peerId}
   │
4. agentCommand() — 启动 Agent 处理
   │
5. emitAgentEvent() → createAgentEventHandler()
   │  事件类型：message_start, tool_execution, message_end
   │
6. broadcast("agent") + broadcast("chat")
   │  推送给所有订阅的 WebSocket 客户端
   │
7. resolveOutboundTarget() → deliverOutboundPayloads()
      回复发送回原始通道
```

路由决策（`src/routing/resolve-route.ts`）的匹配优先级设计值得关注：

```typescript
// src/routing/resolve-route.ts — 路由匹配优先级
// 1. binding.peer       — 精确匹配对话方
// 2. binding.peer.parent — 线程父级回退
// 3. binding.guild      — 服务器/组织级
// 4. binding.team       — 团队级
// 5. binding.account    — 账号级
// 6. binding.channel    — 通道级（accountId: "*"）
// 7. default            — 默认 Agent
```

这种层级回退设计使得同一个 Gateway 可以同时服务多个 Agent：工作相关的 Slack 消息路由到"工作助手"，个人 WhatsApp 消息路由到"生活助手"，而 Discord 群组消息路由到"社区助手"。

**设计哲学**：路由系统的本质是一个从具体到抽象的匹配链。越具体的规则越优先，最终的 default 兜底保证不会有消息掉进黑洞。

## 会话键（Session Key）设计

会话键是路由系统的关键数据结构，它唯一标识了一个对话上下文：

```typescript
// src/routing/session-key.ts
// 主会话键格式
function buildAgentMainSessionKey({ agentId, mainKey }) {
  return `agent:${agentId}:${mainKey}`;
}
// 对等会话键格式
function buildAgentPeerSessionKey({
  agentId, channel, accountId, peerKind, peerId, dmScope
}) {
  // → agent:default:telegram:bot123:dm:user456
}
```

会话键的设计确保了：
- **隔离性**：不同通道、不同用户的会话完全隔离
- **可路由性**：从键名即可反推通道、Agent、对话方
- **持久性**：键值稳定，重启后可恢复会话

## 认证与安全模型

Gateway 的认证体系分为四层：

| 认证方式 | 场景 | 实现 |
|---------|------|------|
| Token | CLI、远程客户端 | `gateway.auth.token` 配置 |
| Password | Web UI、远程访问 | `gateway.auth.password` 配置 |
| Tailscale | 安全网络内访问 | Tailscale whois 身份头 |
| Device Pairing | 移动设备配对 | 公钥签名 + 配对码 |

WebSocket 连接建立的握手流程：

```
Client                          Gateway
  │                               │
  │──── connect ─────────────────>│
  │                               │
  │<──── connect.challenge ───────│  (nonce + timestamp)
  │                               │
  │──── connect { auth, role } ──>│  (token/password/device)
  │                               │
  │<──── hello-ok { snapshot } ───│  (features, policy, state)
  │                               │
```

DM 安全策略默认为 `pairing` 模式：未知发送者会收到一个配对码，需要通过 `openclaw pairing approve` 手动批准。这是对"AI 助手连接到真实消息表面"这一安全风险的务实回应。

## 事件广播机制

Gateway 的广播系统（`src/gateway/server-broadcast.ts`）是连接所有客户端的神经网络。Agent 的每一个动作——开始思考、调用工具、生成回复——都会通过事件系统实时推送给所有订阅的客户端。

关键设计决策：**慢消费者保护**。

```typescript
// src/gateway/server-broadcast.ts — 慢消费者处理
if (socket.bufferedAmount > MAX_BUFFERED_BYTES && dropIfSlow) {
  // 丢弃事件，保护服务器
} else if (socket.bufferedAmount > MAX_BUFFERED_BYTES) {
  // 关闭连接，带 1008 状态码
}
```

如果某个客户端处理不够快（比如一个网络状况不好的手机），Gateway 不会等待它，而是选择丢弃事件或断开连接。这确保了一个慢客户端不会拖垮整个系统。

## 对比其他方案

| 维度 | OpenClaw | LangChain Serve | AutoGen |
|------|---------|-----------------|---------|
| 部署模型 | 本地单用户 | 云端多用户 | 库/框架 |
| 通道集成 | 原生多通道 | 无（需自建） | 无（纯 Agent） |
| 协议 | 自研 WS + HTTP | REST API | 无标准协议 |
| 状态管理 | Gateway 集中式 | 外部存储 | 内存/自管理 |
| 插件系统 | 运行时 TypeScript | Python 包 | Python 包 |

OpenClaw 的独特之处在于它不仅仅是一个 Agent 框架，而是一个完整的"AI 助手基础设施"。Gateway 控制平面 + 多通道路由 + 设备节点网络的组合，在开源领域几乎没有直接竞品。

---

## 章节衔接

**本章回顾**：
- 我们学习了 OpenClaw 的三层架构模型（通道层 → 控制层 → 智能层）
- 关键收获：Gateway 三帧协议和层级路由匹配是整个系统的骨架

**下一章预告**：
- 在 `02-gateway.md` 中，我们将深入 Gateway 的内部实现
- 为什么需要学习：架构总览给出了骨架，Gateway 是骨架的核心支柱
- 关键问题：WebSocket 连接管理、方法分发、状态模型的具体实现
