# 第六层：协议层

> 本章介绍 kimi-cli 的外部协议集成，包括 ACP（Agent Client Protocol）和 Wire 事件总线。

---

## 6.1 Wire 事件总线

### 设计意图

Agent 内部有多个模块需要通信（如 UI、Soul、MCP、Logger）。如何解耦这些模块？

**核心问题**：
1. **模块解耦**：模块间不应该直接依赖
2. **异步通信**：某些操作是异步的（如 LLM 调用）
3. **多订阅者**：一个事件可能被多个模块关注

### 核心实现

**源码位置**: `src/kimi_cli/wire/types.py`

```python
@dataclass
class Event:
    type: str  # 事件类型
    data: dict  # 事件数据

class Wire:
    """
    SPMC (Single Producer, Multiple Consumer) 事件总线
    """

    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = {}

    def subscribe(self, event_type: str, handler: Callable) -> None:
        """订阅事件"""
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(handler)

    async def publish(self, event: Event) -> None:
        """发布事件"""
        handlers = self._subscribers.get(event.type, [])
        for handler in handlers:
            await handler(event)
```

### 使用示例

**源码位置**: `src/kimi_cli/soul/run_soul.py:45-89`

```python
# 创建事件总线
wire = Wire()

# Soul 订阅用户输入事件
wire.subscribe("user.input", soul.handle_input)

# Logger 订阅所有事件
wire.subscribe("*", logger.log_all)

# MCP Manager 订阅工具调用事件
wire.subscribe("tool.before_call", mcp_manager.on_before_call)
wire.subscribe("tool.after_call", mcp_manager.on_after_call)

# 发布事件
await wire.publish(Event(type="user.input", data={"text": "列出文件"}))
```

### 设计分析

**Q: Wire 使用 SPMC 模型。相比传统回调或观察者模式，优势和潜在风险？**

A: 对比：

| 模式 | 优势 | 劣势 | 适用场景 |
|------|------|------|----------|
| 回调 | 简单，性能好 | 紧耦合，难以扩展 | 简单场景 |
| 观察者 | 解耦，支持多订阅者 | 同步调用，可能阻塞 | 单机应用 |
| **Wire SPMC** | 异步，完全解耦 | 复杂度高，调试困难 | 分布式系统 |

**优势**：
1. **完全解耦**：发布者不需要知道谁在订阅
2. **异步**：订阅者可以异步处理，不阻塞发布者
3. **可扩展**：新增订阅者无需修改发布者代码

**风险**：
1. **调试困难**：事件流不直观，难以追踪
2. **错误处理**：订阅者报错不影响其他订阅者，但可能丢失错误
3. **内存泄漏**：订阅者忘记取消订阅会导致内存泄漏

**Q: 如何保证事件的顺序？**

A: 当前实现不保证顺序。如果需要顺序保证：

```python
class OrderedWire(Wire):
    """保证订阅者按订阅顺序执行"""
    async def publish(self, event: Event) -> None:
        handlers = self._subscribers.get(event.type, [])
        for handler in handlers:
            await handler(event)  # 串行执行，保证顺序
```

但会牺牲性能（串行 vs 并发）。

---

## 6.2 ACP 协议

### 设计意图

ACP (Agent Client Protocol) 是 IDE 与 Agent 通信的标准协议。

**核心问题**：
1. **IDE 集成**：如何在 VS Code 中调用 Agent？
2. **流式输出**：Agent 的输出需要实时显示在 IDE 中
3. **双向通信**：IDE 可以主动发送消息，Agent 也可以推送更新

### 协议格式

```json
// IDE → Agent：请求
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "agent/chat",
  "params": {
    "message": "帮我解释这段代码",
    "context": {...}
  }
}

// Agent → IDE：响应（流式）
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "message": "这段代码的作用是..."
  }
}

// Agent → IDE：推送通知
{
  "jsonrpc": "2.0",
  "method": "agent/status",
  "params": {
    "status": "thinking",
    "message": "正在分析代码..."
  }
}
```

### 核心实现

**源码位置**: `src/kimi_cli/acp/`

```python
class ACPServer:
    """ACP 协议服务器"""

    def __init__(self, soul: KimiSoul):
        self._soul = soul
        self._handlers = {
            "agent/chat": self._handle_chat,
            "agent/status": self._handle_status,
        }

    async def handle_request(self, request: dict) -> dict:
        """处理 IDE 请求"""
        method = request["method"]
        handler = self._handlers.get(method)

        if not handler:
            return {"error": f"Unknown method: {method}"}

        return await handler(request["params"])

    async def _handle_chat(self, params: dict) -> dict:
        """处理聊天请求"""
        message = params["message"]

        # 调用 Agent
        result = await self._soul.chat(message)

        return {"result": result.message}

    async def push_status(self, status: str, message: str) -> None:
        """推送状态更新到 IDE"""
        notification = {
            "jsonrpc": "2.0",
            "method": "agent/status",
            "params": {"status": status, "message": message},
        }
        await self._send_to_ide(notification)
```

### 使用场景

**VS Code 扩展集成**：

```typescript
// VS Code 扩展
const client = new LanguageClient(
  "kimi-cli",
  { command: "kimi-cli", args: ["--acp"] }
);

// 发送聊天请求
const response = await client.sendRequest("agent/chat", {
  message: "帮我解释这段代码",
  context: { file: vscode.window.activeTextEditor.document.uri }
});

// 监听状态更新
client.onNotification("agent/status", (params) => {
  vscode.window.setStatusBarMessage(params.message);
});
```

### 设计分析

**Q: ACP 与 LSP (Language Server Protocol) 的区别？**

A: 对比：

| 协议 | 目标 | 通信模式 | 典型场景 |
|------|------|----------|----------|
| LSP | 代码分析 | 请求-响应 | 代码补全、跳转定义 |
| ACP | Agent 交互 | 双向通信 | 聊天、工具调用、状态推送 |

**示例**：

```
LSP: "给我这个函数的定义" → 返回定义位置
ACP: "帮我重构这段代码" → 多轮对话 + 工具调用 + 状态更新
```

**Q: 如何处理流式输出？**

A: 使用 Server-Sent Events (SSE)：

```python
async def _handle_chat_stream(self, params: dict):
    """流式聊天"""
    message = params["message"]

    # 流式生成
    async for chunk in self._soul.chat_stream(message):
        await self._send_sse(chunk)
```

```typescript
// VS Code 接收流式输出
const eventSource = new EventSource("http://localhost:8080/stream");
eventSource.onmessage = (event) => {
  vscode.window.showInformationMessage(event.data);
};
```

---

## 本章小结

协议层的设计：

1. **Wire 事件总线**：SPMC 模型，实现模块解耦
2. **ACP 协议**：IDE 与 Agent 通信的标准协议

这些协议让 kimi-cli 能够与外部系统无缝集成。
