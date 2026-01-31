# 07 - Wire 协议与 UI 解耦

## 本章目标

理解 Wire 协议的设计，掌握 Soul 和 UI 的解耦机制，学习事件流的处理方式。

## Wire 协议概述

Wire 是 kimi-cli 的事件流抽象层，实现了 Soul（Agent 核心）和 UI（交互界面）的完全解耦。

**设计目标**：
1. **解耦**：Soul 只负责发送事件，UI 只负责消费事件
2. **多 UI 支持**：同一个 Wire 可以被多个 UI 消费
3. **持久化**：事件可以写入文件，支持回放

```
┌─────────────────────────────────────────────────────────────────┐
│                        Wire 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐                      ┌─────────────┐          │
│  │   KimiSoul  │                      │  Shell UI   │          │
│  │  (soul_side)│                      │  (ui_side)  │          │
│  └──────┬──────┘                      └──────▲──────┘          │
│         │                                    │                  │
│         │ send()                             │ receive()        │
│         ▼                                    │                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                         Wire                             │   │
│  │                                                         │   │
│  │  ┌─────────────┐              ┌─────────────┐          │   │
│  │  │  raw_queue  │──────────────│merged_queue │          │   │
│  │  │ (原始消息)  │   merge      │ (合并消息)  │          │   │
│  │  └─────────────┘              └─────────────┘          │   │
│  │                                                         │   │
│  │  ┌─────────────┐                                       │   │
│  │  │ WireRecorder│ ◄── 持久化到 wire.jsonl               │   │
│  │  └─────────────┘                                       │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 事件类型定义

### 控制流事件

```python
# src/kimi_cli/wire/types.py:36-93
class TurnBegin(BaseModel):
    """New conversation turn begins."""
    user_input: str | list[ContentPart]

class StepBegin(BaseModel):
    """Agent step begins."""
    n: int

class StepInterrupted(BaseModel):
    """Step was interrupted."""
    pass

class CompactionBegin(BaseModel):
    """Context compaction begins."""
    pass

class CompactionEnd(BaseModel):
    """Context compaction ends."""
    pass

class StatusUpdate(BaseModel):
    """Status update."""
    context_usage: float | None = None
    token_usage: TokenUsage | None = None
    message_id: str | None = None
```

### 内容事件

内容事件直接复用 Kosong 的 ContentPart 类型：
- `TextPart`：文本内容
- `ThinkPart`：思考内容
- `ToolCall`：工具调用
- `ToolCallPart`：工具调用参数片段
- `ToolResult`：工具执行结果

### 请求-响应事件

```python
# src/kimi_cli/wire/types.py:145-246
class ApprovalRequest(BaseModel):
    """Request user approval for an action."""
    id: str
    tool_call_id: str
    action: str
    description: str
    display: list[DisplayBlock]

    _future: asyncio.Future[ApprovalResponse.Kind] | None = PrivateAttr(default=None)

    async def wait(self) -> ApprovalResponse.Kind:
        """Wait for user response."""
        if self._future is None:
            self._future = asyncio.Future()
        return await self._future

    def resolve(self, response: ApprovalResponse.Kind) -> None:
        """Resolve the request with user response."""
        if self._future is not None:
            self._future.set_result(response)

class ApprovalResponse(BaseModel):
    """Response to approval request."""
    id: str
    response: Literal["approve", "approve_for_session", "reject"]
```

**设计亮点**：
- **Future 内嵌**：ApprovalRequest 内嵌 asyncio.Future，支持 `await request.wait()`
- **双向通信**：UI 通过 `resolve()` 响应请求

### SubagentEvent 嵌套

```python
# src/kimi_cli/wire/types.py:95-130
class SubagentEvent(BaseModel):
    """Event from a subagent."""
    task_tool_call_id: str  # 关联父级 tool call
    event: Event            # 嵌套的子事件

    @field_serializer("event", when_used="json")
    def _serialize_event(self, event: Event) -> dict[str, Any]:
        envelope = WireMessageEnvelope.from_wire_message(event)
        return envelope.model_dump(mode="json")
```

**设计决策**：
- **层级保留**：通过嵌套而非扁平化，保留事件的语义结构
- **序列化处理**：使用 Envelope 包装，解决 Union 类型的序列化问题

## Wire 核心实现

### 双队列架构

```python
# src/kimi_cli/wire/__init__.py:18-35
class Wire:
    def __init__(self, *, file_backend: WireFile | None = None):
        self._raw_queue = WireMessageQueue()      # 原始消息
        self._merged_queue = WireMessageQueue()   # 合并后消息
        self._soul_side = WireSoulSide(self._raw_queue, self._merged_queue)
        self._recorder = _WireRecorder(file_backend, self._merged_queue)

    def ui_side(self, *, merge: bool) -> WireUISide:
        """Create a UI side consumer."""
        if merge:
            return WireUISide(self._merged_queue.subscribe())
        else:
            return WireUISide(self._raw_queue.subscribe())
```

**为什么需要两个队列？**

1. **raw_queue**：保证每个消息片段都能被实时消费
   - Shell UI 需要逐字渲染，使用 `merge=False`
2. **merged_queue**：减少消息数量，降低持久化开销
   - WireRecorder 使用合并后的消息

### Soul 侧发送逻辑

```python
# src/kimi_cli/wire/__init__.py:66-113
class WireSoulSide:
    def __init__(self, raw_queue: WireMessageQueue, merged_queue: WireMessageQueue):
        self._raw_queue = raw_queue
        self._merged_queue = merged_queue
        self._merge_buffer: StreamedMessagePart | None = None

    def send(self, msg: WireMessage) -> None:
        # 1. 发送原始消息
        self._raw_queue.publish_nowait(msg)

        # 2. 合并逻辑
        match msg:
            case MergeableMixin():
                if self._merge_buffer is None:
                    self._merge_buffer = copy.deepcopy(msg)
                elif self._merge_buffer.merge_in_place(msg):
                    pass  # 合并成功
                else:
                    self.flush()  # 合并失败，刷新缓冲区
                    self._merge_buffer = copy.deepcopy(msg)
            case _:
                self.flush()  # 非可合并消息
                self._send_merged(msg)

    def flush(self) -> None:
        """Flush the merge buffer."""
        if self._merge_buffer is not None:
            self._send_merged(self._merge_buffer)
            self._merge_buffer = None
```

**合并逻辑**：
- 可合并消息（TextPart、ToolCallPart）累积到 `_merge_buffer`
- 不可合并消息触发 flush，将缓冲区内容发送到 merged_queue

### UI 侧接收逻辑

```python
# src/kimi_cli/wire/__init__.py:115-128
class WireUISide:
    def __init__(self, queue: Queue[WireMessage]):
        self._queue = queue

    async def receive(self) -> WireMessage:
        """Receive next message."""
        return await self._queue.get()
```

### BroadcastQueue 实现

```python
# src/kimi_cli/utils/broadcast.py:6-37
class BroadcastQueue[T]:
    """Single-producer, multiple-consumer queue."""

    def __init__(self) -> None:
        self._queues: set[Queue[T]] = set()

    def subscribe(self) -> Queue[T]:
        """Create a new consumer queue."""
        queue: Queue[T] = Queue()
        self._queues.add(queue)
        return queue

    def publish_nowait(self, item: T) -> None:
        """Publish to all consumers."""
        for queue in self._queues:
            queue.put_nowait(item)
```

**设计亮点**：
- **SPMC 模式**：单生产者多消费者
- **动态订阅**：每次 `subscribe()` 创建新队列

## Shell UI 实现

### 可视化引擎

```python
# src/kimi_cli/ui/shell/visualize.py:446-520
class _LiveView:
    def __init__(self, initial_status: StatusUpdate, cancel_event: asyncio.Event | None):
        self._mooning_spinner: Spinner | None = None
        self._compacting_spinner: Spinner | None = None
        self._current_content_block: _ContentBlock | None = None
        self._tool_call_blocks: dict[str, _ToolCallBlock] = {}
        self._approval_request_queue = deque[ApprovalRequest]()

    async def visualize_loop(self, wire: WireUISide):
        with Live(self.compose(), console=console, refresh_per_second=10) as live:
            async with _keyboard_listener(keyboard_handler):
                while True:
                    msg = await wire.receive()
                    self.dispatch_wire_message(msg)
                    if self._need_recompose:
                        live.update(self.compose(), refresh=True)
```

### 消息分发

```python
# src/kimi_cli/ui/shell/visualize.py:545-593
def dispatch_wire_message(self, msg: WireMessage) -> None:
    match msg:
        case TurnBegin():
            self.flush_content()
            console.print(Panel(Text(message_stringify(...))))
        case CompactionBegin():
            self._compacting_spinner = Spinner("balloon", "Compacting...")
        case StatusUpdate():
            self._status_block.update(msg)
        case ContentPart():
            self.append_content(msg)
        case ToolCall():
            self.append_tool_call(msg)
        case ToolResult():
            self.append_tool_result(msg)
        case SubagentEvent():
            self.handle_subagent_event(msg)
        case ApprovalRequest():
            self.request_approval(msg)
```

### 批准请求交互

```python
# src/kimi_cli/ui/shell/visualize.py:595-632
def dispatch_keyboard_event(self, event: KeyEvent) -> None:
    if event == KeyEvent.ESCAPE and self._cancel_event is not None:
        self._cancel_event.set()
        return

    match event:
        case KeyEvent.UP:
            self._current_approval_request_panel.move_up()
        case KeyEvent.DOWN:
            self._current_approval_request_panel.move_down()
        case KeyEvent.ENTER:
            resp = self._current_approval_request_panel.get_selected_response()
            self._current_approval_request_panel.request.resolve(resp)

            if resp == "approve_for_session":
                # 批量批准相同 action 的请求
                for request in self._approval_request_queue:
                    if request.action == current_action:
                        request.resolve("approve_for_session")
            elif resp == "reject":
                # 拒绝所有队列中的请求
                while self._approval_request_queue:
                    self._approval_request_queue.popleft().resolve("reject")
```

## Wire 持久化

### WireFile 格式

```python
# src/kimi_cli/wire/file.py:20-60
class WireFile:
    PROTOCOL_VERSION = 1

    def __init__(self, path: Path):
        self._path = path
        self._file: aiofiles.threadpool.AsyncTextIOWrapper | None = None

    async def write_metadata(self, metadata: dict[str, Any]) -> None:
        """Write metadata as first line."""
        await self._file.write(json.dumps({
            "protocol_version": self.PROTOCOL_VERSION,
            **metadata,
        }) + "\n")

    async def write_message(self, msg: WireMessage) -> None:
        """Write a message."""
        envelope = WireMessageEnvelope.from_wire_message(msg)
        await self._file.write(envelope.model_dump_json() + "\n")
```

### 文件格式示例

```jsonl
{"protocol_version": 1, "session_id": "abc123", "created_at": "2024-01-01T00:00:00Z"}
{"type": "TurnBegin", "payload": {"user_input": "Hello"}}
{"type": "TextPart", "payload": {"type": "text", "text": "Hi there!"}}
{"type": "ToolCall", "payload": {"id": "tc1", "function": {"name": "Shell", "arguments": "..."}}}
{"type": "ToolResult", "payload": {"tool_call_id": "tc1", "return_value": {...}}}
```

## JSON-RPC 协议层

Wire 还支持通过 JSON-RPC 进行远程通信（用于 IDE 集成）。

```python
# src/kimi_cli/wire/jsonrpc.py:25-80
# 入站消息（Client → Server）
type JSONRPCInMessage = (
    JSONRPCSuccessResponse
    | JSONRPCErrorResponse
    | JSONRPCInitializeMessage
    | JSONRPCPromptMessage
    | JSONRPCCancelMessage
)

# 出站消息（Server → Client）
type JSONRPCOutMessage = (
    JSONRPCSuccessResponse
    | JSONRPCErrorResponse
    | JSONRPCEventMessage      # 事件推送
    | JSONRPCRequestMessage    # 请求（需要响应）
)
```

## 设计决策分析

### 为什么用双队列？

**现象层**：Wire 有 raw_queue 和 merged_queue。

**本质层**：解决流式输出的性能与完整性矛盾。
- Shell UI 需要低延迟（逐字渲染），使用 raw_queue
- WireRecorder 需要低存储（减少文件大小），使用 merged_queue

**哲学层**：这是"时间换空间"的经典权衡。

### 为什么用 BroadcastQueue？

**现象层**：每次 `subscribe()` 创建新队列。

**本质层**：支持多个 UI 同时消费同一个 Wire。
- 例如：同时运行 Shell UI 和 Debug UI
- 每个 UI 有独立的消费进度

### 为什么 ApprovalRequest 内嵌 Future？

**现象层**：`await request.wait()` 可以阻塞等待响应。

**本质层**：简化请求-响应的编程模型。
- Soul 发送请求后可以直接 await
- UI 通过 `resolve()` 设置结果
- 无需额外的回调或事件监听

---

## 章节衔接

**本章回顾**：
- 我们学习了 Wire 协议的双队列架构、事件类型、持久化机制
- 关键收获：SPMC 模式、消息合并、Future 内嵌

**下一章预告**：
- 在 `08-dmail.md` 中，我们将学习 D-Mail 时间旅行机制
- 为什么需要学习：D-Mail 是 kimi-cli 的独创设计，理解它才能理解"后悔药"功能
- 关键问题：D-Mail 如何触发？异常机制如何实现时间旅行？
