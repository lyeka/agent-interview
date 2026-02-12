# Gateway WebSocket 控制平面

Gateway 是 OpenClaw 的中枢神经系统。它以单进程运行在用户的本地设备上（`ws://127.0.0.1:18789`），同时扮演 WebSocket 服务器、HTTP 服务器和事件总线三重角色。所有通道、Agent、CLI、Web UI 和移动节点都通过 Gateway 交互。

## Gateway 启动流程

Gateway 的启动序列（`src/gateway/server.impl.ts:152-456`）是一个精心编排的初始化链：

```
1. loadConfig()         — 加载 ~/.openclaw/openclaw.json
2. loadPlugins()        — 发现、校验、激活所有插件
3. startHttpServer()    — Express HTTP 服务
4. startWsServer()      — WebSocket 升级处理
5. startChannels()      — 启动所有已配置的通道（Telegram, Discord...）
6. startNodeRegistry()  — 初始化设备节点注册表
7. startCron()          — 初始化定时任务
8. broadcastPresence()  — 通知所有客户端 Gateway 已就绪
```

**关键设计决策**：通道启动是异步且容错的。一个通道（比如 Telegram）启动失败不会阻塞其他通道。Gateway 会记录错误并继续。这保证了部分配置错误不会让整个系统瘫痪。

## WebSocket 协议详解

### 连接握手

WebSocket 连接建立后，Gateway 发起一个挑战-应答握手：

```
Client                              Gateway
  │                                   │
  │←── connect.challenge ─────────────│  { nonce, ts }
  │                                   │
  │──── req: connect ────────────────>│  { minProtocol, maxProtocol,
  │       { auth, role, scopes,       │    client, device, commands }
  │         client, device }          │
  │                                   │
  │←── res: hello-ok ─────────────────│  { protocol, server, features,
  │       { snapshot, policy,         │    snapshot, canvasHostUrl,
  │         features, auth }          │    auth, policy }
  │                                   │
```

ConnectParams 定义了客户端的身份声明（`src/gateway/protocol/schema/frames.ts:21-68`）：

```typescript
// 连接参数——客户端自我声明
ConnectParams = {
  minProtocol: number,     // 最低兼容协议版本
  maxProtocol: number,     // 最高兼容协议版本
  client: {
    id: string,            // 客户端标识
    version: string,       // 客户端版本
    platform: string,      // macOS / iOS / Android / CLI
    mode: string,          // operator / node
  },
  role?: "operator" | "node",
  scopes?: string[],       // 权限范围
  auth?: { token?, password? },
  device?: { id, publicKey, signature },
};
```

HelloOk 响应包含了客户端正常工作所需的一切：

```typescript
// 握手成功响应
HelloOk = {
  protocol: number,
  server: { version, commit, host, connId },
  features: {
    methods: string[],    // 可用的 RPC 方法列表
    events: string[],     // 可订阅的事件列表
  },
  snapshot: Snapshot,     // 当前系统状态快照
  policy: {
    maxPayload: number,
    maxBufferedBytes: number,
    tickIntervalMs: number,
  },
};
```

### RPC 方法体系

Gateway 暴露了约 40 个 RPC 方法（`src/gateway/server-methods-list.ts`），按领域分组：

| 领域 | 方法 | 说明 |
|------|------|------|
| 系统 | `health`, `logs.tail` | 健康检查、日志流 |
| 配置 | `config.get`, `config.set`, `config.patch` | 运行时配置管理 |
| Agent | `agent`, `agent.wait` | 执行 Agent、等待完成 |
| 聊天 | `chat.send`, `chat.history`, `chat.abort` | WebChat 交互 |
| 会话 | `sessions.list`, `sessions.patch`, `sessions.reset` | 会话 CRUD |
| 通道 | `channels.status`, `channels.login` | 通道状态管理 |
| 发送 | `send` | 通过通道发送消息 |
| 节点 | `node.list`, `node.invoke`, `node.event` | 设备节点交互 |
| 定时 | `cron.list`, `cron.set`, `cron.delete` | 定时任务管理 |

方法分发流程：

```typescript
// src/gateway/server/ws-connection/message-handler.ts
handleGatewayRequest(method, params) {
  // 1. 授权检查
  authorizeGatewayMethod(client, method);
  // 2. 路由到处理器
  const handler = coreGatewayHandlers[method]
    ?? extraHandlers[method];
  // 3. 执行并响应
  const result = await handler(params, context);
  respond(true, result);
}
```

### 事件推送体系

Gateway 的事件系统采用发布-订阅模式。核心事件类型：

```
agent          — Agent 执行生命周期事件
chat           — 聊天消息流（delta + final）
presence       — 系统存在状态
tick           — 周期性心跳
health         — 健康状态变更
shutdown       — 服务器关闭
talk.mode      — 语音模式切换
node.pair.*    — 设备配对事件
exec.approval.* — 执行审批事件
```

事件广播支持作用域过滤——不是所有客户端都能接收所有事件：

```typescript
// 广播时可以指定作用域
broadcast("exec.approval.requested", payload, {
  scopes: ["operator.approvals"],
});
// 只有拥有 approvals 权限的客户端会收到
```

## 状态模型

Gateway 维护的运行时状态是纯内存的：

| 状态 | 类型 | 用途 |
|------|------|------|
| `clients` | `Set<GatewayWsClient>` | 已连接的 WebSocket 客户端 |
| `agentRunSeq` | `Map<string, number>` | 每个 Agent 执行的序列号 |
| `chatRunState` | `ChatRunState` | 聊天会话的运行状态 |
| `dedupe` | `Map<string, DedupeEntry>` | 幂等性去重缓存 |
| `healthCache` | `HealthSummary` | 健康状态缓存 |

**为什么不持久化这些状态？** 因为 Gateway 设计为常驻进程（daemon），这些状态在进程生命周期内有效。重启后，客户端会重新连接，通道会重新初始化，状态自然重建。真正需要持久化的（配置、会话历史、设备配对）存储在文件系统中。

## 通道管理

ChannelManager（`src/gateway/server-channels.ts`）管理所有通道的生命周期：

```typescript
// 通道生命周期
createChannelManager() {
  startChannel(channelId, accountId?) {
    const plugin = getChannelPlugin(channelId);
    await plugin.gateway.startAccount(accountId);
    // 更新运行时快照
  }
  stopChannel(channelId, accountId?) {
    // 中止并清理
  }
  getRuntimeSnapshot() {
    // 返回所有通道的当前状态
  }
}
```

通道启停是热操作——不需要重启 Gateway 就可以连接或断开一个通道。这使得用户可以在运行时动态管理通道。

## HTTP 端点

Gateway 同时运行一个 Express HTTP 服务器（`src/gateway/server-http.ts`），提供：

| 端点 | 用途 |
|------|------|
| `POST /hooks/wake` | Webhook 唤醒 |
| `POST /hooks/agent` | 通过 HTTP 触发 Agent |
| `POST /v1/chat/completions` | OpenAI 兼容 API |
| `POST /v1/responses` | OpenResponses API |
| `GET /` | Control UI 静态页面 |
| `GET /webchat` | WebChat 界面 |
| Canvas host | A2UI 画布 HTTP 服务 |

OpenAI 兼容 API 使得 OpenClaw 可以作为其他工具（如 Cursor、Continue）的模型后端使用。

---

## 章节衔接

**本章回顾**：
- 我们学习了 Gateway 的完整内部实现：WebSocket 协议、RPC 方法体系、事件广播、状态模型
- 关键收获：三帧协议的极简设计和热通道管理的运维友好性

**下一章预告**：
- 在 `03-agent-runtime.md` 中，我们将学习 Pi Agent 运行时和会话模型
- 为什么需要学习：Gateway 是调度器，Agent 是执行者，两者如何协作是理解整个系统的关键
- 关键问题：Agent 如何处理消息、工具如何注册和调用、会话如何管理
