# 04 - Context 管理与 Checkpoint 系统

## 本章目标

理解 kimi-cli 的 Context 管理机制，掌握 Checkpoint 的创建和回退，学习 Compaction 策略。

## Context 概述

Context 是 Agent 的"记忆"，存储所有对话历史。kimi-cli 的 Context 设计有三个核心特性：

1. **文件持久化**：所有消息写入 JSONL 文件，支持断点续传
2. **Checkpoint 机制**：每个 step 前创建快照，支持时间旅行
3. **自动压缩**：当 context 接近上限时，用 LLM 总结历史

## Context 核心实现

### 数据结构

```python
# src/kimi_cli/soul/context.py:20-45
class Context:
    def __init__(self, file_backend: Path | str):
        self._file_backend = Path(file_backend)
        self._history: list[Message] = []
        self._token_count: int = 0
        self._next_checkpoint_id: int = 0

    @property
    def history(self) -> Sequence[Message]:
        """Get the conversation history."""
        return self._history

    @property
    def token_count(self) -> int:
        """Get the current token count."""
        return self._token_count

    @property
    def n_checkpoints(self) -> int:
        """Get the number of checkpoints."""
        return self._next_checkpoint_id
```

**设计决策**：
- `_history`：内存中的消息列表，用于快速访问
- `_file_backend`：JSONL 文件路径，用于持久化
- `_token_count`：累计 token 数，用于触发压缩
- `_next_checkpoint_id`：下一个 checkpoint ID，递增分配

### 消息追加

```python
# src/kimi_cli/soul/context.py:47-66
async def append_message(self, message: Message) -> None:
    """Append a message to the context."""
    self._history.append(message)

    # 同步写入文件
    async with aiofiles.open(self._file_backend, "a", encoding="utf-8") as f:
        await f.write(message.model_dump_json() + "\n")
```

**设计亮点**：
- **双写策略**：同时更新内存和文件，保证一致性
- **JSONL 格式**：每行一个 JSON 对象，便于追加和解析
- **异步 I/O**：使用 aiofiles 避免阻塞

### 文件格式示例

```jsonl
{"role": "user", "content": [{"type": "text", "text": "Hello"}]}
{"role": "_checkpoint", "id": 0}
{"role": "assistant", "content": [{"type": "text", "text": "Hi!"}], "tool_calls": null}
{"role": "_usage", "token_count": 150}
{"role": "_checkpoint", "id": 1}
{"role": "user", "content": [{"type": "text", "text": "Write a function"}]}
```

**特殊行类型**：
- `_checkpoint`：checkpoint 标记，包含 ID
- `_usage`：token 计数更新

## Checkpoint 机制

### 创建 Checkpoint

```python
# src/kimi_cli/soul/context.py:68-78
async def checkpoint(self, add_user_message: bool):
    """Create a checkpoint in the context."""
    checkpoint_id = self._next_checkpoint_id
    self._next_checkpoint_id += 1
    logger.debug("Checkpointing, ID: {id}", id=checkpoint_id)

    # 写入 checkpoint 标记
    async with aiofiles.open(self._file_backend, "a", encoding="utf-8") as f:
        await f.write(json.dumps({"role": "_checkpoint", "id": checkpoint_id}) + "\n")

    # 可选：在 context 中插入 CHECKPOINT 消息（供 Agent 感知）
    if add_user_message:
        await self.append_message(
            Message(role="user", content=[system(f"CHECKPOINT {checkpoint_id}")])
        )
```

**设计决策**：
- **文件持久化**：checkpoint 写入文件，重启后可恢复
- **可选用户消息**：当启用 D-Mail 工具时，在 context 中插入 `CHECKPOINT N`，让 Agent 知道可以回退到哪里

### 回退到 Checkpoint

```python
# src/kimi_cli/soul/context.py:80-133
async def revert_to(self, checkpoint_id: int):
    """Revert the context to the specified checkpoint."""
    logger.debug("Reverting checkpoint, ID: {id}", id=checkpoint_id)

    if checkpoint_id >= self._next_checkpoint_id:
        raise ValueError(f"Checkpoint {checkpoint_id} does not exist")

    # 1. 轮转旧文件（备份）
    rotated_file_path = await next_available_rotation(self._file_backend)
    await aiofiles.os.replace(self._file_backend, rotated_file_path)
    logger.debug("Rotated context file: {path}", path=rotated_file_path)

    # 2. 重建 context
    self._history.clear()
    self._token_count = 0
    self._next_checkpoint_id = 0

    async with (
        aiofiles.open(rotated_file_path, encoding="utf-8") as old_file,
        aiofiles.open(self._file_backend, "w", encoding="utf-8") as new_file,
    ):
        async for line in old_file:
            if not line.strip():
                continue

            line_json = json.loads(line)

            # 遇到目标 checkpoint 就停止
            if line_json["role"] == "_checkpoint" and line_json["id"] == checkpoint_id:
                break

            # 写入新文件
            await new_file.write(line)

            # 更新内存状态
            if line_json["role"] == "_usage":
                self._token_count = line_json["token_count"]
            elif line_json["role"] == "_checkpoint":
                self._next_checkpoint_id = line_json["id"] + 1
            else:
                message = Message.model_validate(line_json)
                self._history.append(message)
```

**设计亮点**：

1. **文件轮转**：回退时先将旧文件重命名为 `.1`, `.2` 等备份，避免数据丢失
2. **流式重建**：逐行读取旧文件，遇到目标 checkpoint 就停止
3. **状态一致性**：同时更新 `_history`, `_token_count`, `_next_checkpoint_id`

### 文件轮转示例

```
回退前：
  context.jsonl (当前文件)

回退后：
  context.jsonl (重建的文件，只包含 checkpoint 之前的内容)
  context.jsonl.1 (备份，包含完整历史)

再次回退：
  context.jsonl (重建)
  context.jsonl.1 (新备份)
  context.jsonl.2 (旧备份)
```

## Compaction 策略

当 context 接近 max_context_size 时，需要压缩历史对话。

### 触发条件

```python
# src/kimi_cli/soul/kimisoul.py:345-348
reserved = self._loop_control.reserved_context_size  # 默认 50000
if self._context.token_count + reserved >= self._runtime.llm.max_context_size:
    logger.info("Context too long, compacting...")
    await self.compact_context()
```

**计算公式**：`token_count + reserved >= max_context_size`

例如：
- max_context_size = 200,000
- reserved = 50,000
- 当 token_count >= 150,000 时触发压缩

### SimpleCompaction 实现

```python
# src/kimi_cli/soul/compaction.py:42-76
class SimpleCompaction:
    def __init__(self, max_preserved_messages: int = 2):
        self.max_preserved_messages = max_preserved_messages

    async def compact(self, messages: Sequence[Message], llm: LLM) -> Sequence[Message]:
        compact_message, to_preserve = self.prepare(messages)
        if compact_message is None:
            return to_preserve

        # 调用 LLM 进行压缩
        result = await kosong.step(
            chat_provider=llm.chat_provider,
            system_prompt="You are a helpful assistant that compacts conversation context.",
            toolset=EmptyToolset(),
            history=[compact_message],
        )

        # 构建压缩后的消息
        content: list[ContentPart] = [
            system("Previous context has been compacted. Here is the compaction output:")
        ]
        compacted_msg = result.message

        # 过滤掉 ThinkPart
        content.extend(part for part in compacted_msg.content if not isinstance(part, ThinkPart))

        compacted_messages: list[Message] = [Message(role="user", content=content)]
        compacted_messages.extend(to_preserve)
        return compacted_messages
```

### prepare 策略

```python
# src/kimi_cli/soul/compaction.py:82-116
def prepare(self, messages: Sequence[Message]) -> PrepareResult:
    """Prepare messages for compaction."""
    if not messages or self.max_preserved_messages <= 0:
        return PrepareResult(compact_message=None, to_preserve=messages)

    history = list(messages)

    # 从后往前找，保留最近 N 条 user/assistant 消息
    preserve_start_index = len(history)
    n_preserved = 0
    for index in range(len(history) - 1, -1, -1):
        if history[index].role in {"user", "assistant"}:
            n_preserved += 1
            if n_preserved == self.max_preserved_messages:
                preserve_start_index = index
                break

    # 如果消息不够，不压缩
    if n_preserved < self.max_preserved_messages:
        return PrepareResult(compact_message=None, to_preserve=messages)

    to_compact = history[:preserve_start_index]
    to_preserve = history[preserve_start_index:]

    # 构建压缩输入
    compact_message = Message(role="user", content=[])
    for i, msg in enumerate(to_compact):
        compact_message.content.append(
            TextPart(text=f"## Message {i + 1}\nRole: {msg.role}\nContent:\n")
        )
        # 过滤 ThinkPart
        compact_message.content.extend(
            part for part in msg.content if not isinstance(part, ThinkPart)
        )
    compact_message.content.append(TextPart(text="\n" + prompts.COMPACT))

    return PrepareResult(compact_message=compact_message, to_preserve=to_preserve)
```

**压缩策略**：
1. **保留最近消息**：默认保留最近 2 轮对话（`max_preserved_messages=2`）
2. **LLM 驱动压缩**：用 LLM 自己总结历史对话
3. **去除 ThinkPart**：压缩时过滤掉思考内容，减少噪音

### 压缩流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Compaction 流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始 Context:                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ [msg1] [msg2] [msg3] [msg4] [msg5] [msg6] [msg7] [msg8] │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  prepare() 分割:                                                 │
│  ┌─────────────────────────────┐ ┌─────────────────────────┐   │
│  │      to_compact             │ │     to_preserve         │   │
│  │ [msg1] [msg2] ... [msg6]    │ │ [msg7] [msg8]           │   │
│  └─────────────────────────────┘ └─────────────────────────┘   │
│                                                                 │
│  LLM 压缩:                                                       │
│  ┌─────────────────────────────┐                               │
│  │ "Summary of msg1-msg6..."   │                               │
│  └─────────────────────────────┘                               │
│                                                                 │
│  压缩后 Context:                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ [compacted_summary] [msg7] [msg8]                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────���───┘
```

## Token 计数更新

```python
# src/kimi_cli/soul/context.py:135-145
async def update_token_count(self, token_count: int) -> None:
    """Update the token count."""
    self._token_count = token_count

    # 写入 _usage 标记
    async with aiofiles.open(self._file_backend, "a", encoding="utf-8") as f:
        await f.write(json.dumps({"role": "_usage", "token_count": token_count}) + "\n")
```

**设计决策**：
- Token 计数来自 LLM 响应的 `usage.input`
- 持久化到文件，重启后可恢复

## Context 加载

```python
# src/kimi_cli/soul/context.py:147-176
@classmethod
async def load(cls, file_backend: Path | str) -> "Context":
    """Load context from file."""
    context = cls(file_backend)

    if not context._file_backend.exists():
        return context

    async with aiofiles.open(context._file_backend, encoding="utf-8") as f:
        async for line in f:
            if not line.strip():
                continue

            line_json = json.loads(line)

            if line_json["role"] == "_usage":
                context._token_count = line_json["token_count"]
            elif line_json["role"] == "_checkpoint":
                context._next_checkpoint_id = line_json["id"] + 1
            else:
                message = Message.model_validate(line_json)
                context._history.append(message)

    return context
```

**加载流程**：
1. 逐行读取 JSONL 文件
2. 根据 `role` 字段分类处理
3. 重建内存状态

## 设计决策分析

### 为什么用 JSONL 而非 SQLite？

**现象层**：Context 使用 JSONL 文件存储。

**本质层**：
1. **追加友好**：JSONL 只需追加，不需要事务
2. **人类可读**：可以直接用文本编辑器查看
3. **流式处理**：可以逐行读取，内存友好
4. **简单可靠**：没有数据库锁、损坏等问题

**哲学层**：这是"简单即美"的体现。对于 Agent 的对话历史，JSONL 的简单性远比 SQLite 的功能性更重要。

### 为什么 Checkpoint 写入文件？

**现象层**：Checkpoint 是文件中的一行 `{"role": "_checkpoint", "id": N}`。

**本质层**：
1. **持久化**：重启后可恢复 checkpoint 状态
2. **原子性**：与消息在同一个文件，保证一致性
3. **可追溯**：可以看到历史上创建过哪些 checkpoint

### 为什么 Compaction 保留最近 2 轮？

**现象层**：`max_preserved_messages=2`。

**本质层**：
1. **上下文连贯**：最近的对话最重要，不能丢失
2. **工具结果**：最近的工具调用结果可能还在被引用
3. **平衡点**：2 轮足够保持连贯，又不会占用太多空间

---

## 章节衔接

**本章回顾**：
- 我们学习了 Context 的文件持久化、Checkpoint 机制、Compaction 策略
- 关键收获：JSONL 格式、文件轮转、LLM 驱动压缩

**下一章预告**：
- 在 `05-tools.md` 中，我们将学习 Tool 系统与 MCP 集成
- 为什么需要学习：工具是 Agent 的"手脚"，理解工具系统才能理解 Agent 如何与外界交互
- 关键问题：依赖注入如何实现？MCP 工具如何加载？Approval 如何集成？
