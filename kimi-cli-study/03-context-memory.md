# 第三层：Context 与 Memory

> 本章介绍 kimi-cli 如何管理上下文、压缩历史、实现"时间旅行"。

---

## 3.1 Context 压缩策略

### 设计意图

LLM 的上下文窗口有限（如 128K tokens），但 Agent 可能运行很长时间。如何保证关键信息不被丢弃？

**核心问题**：
1. **Context Rot**：上下文过长时，LLM 可能"遗忘"早期信息
2. **Token 成本**：每次请求都携带全量历史，成本线性增长
3. **信息密度**：历史消息中有很多冗余（寒暄、错误尝试）

### SimpleCompaction 算法

**源码位置**: `src/kimi_cli/soul/compaction.py:42-117`

```python
class SimpleCompaction:
    def __init__(
        self,
        keep_recent: int = 20,  # 保留最近 N 条消息
        compact_threshold: int = 60,  # 超过 N 条时触发压缩
    ):
        self._keep_recent = keep_recent
        self._compact_threshold = compact_threshold

    async def compact(self, messages: list[Message]) -> list[Message]:
        if len(messages) < self._compact_threshold:
            return messages

        # 1. 保留 system 消息
        system_msgs = [m for m in messages if m.role == "system"]

        # 2. 保留最近的消息
        recent_msgs = messages[-self._keep_recent:]

        # 3. 中间的消息用 LLM 压缩成摘要
        middle_msgs = messages[len(system_msgs):-self._keep_recent]
        summary = await self._summarize(middle_msgs)

        # 4. 构建新的消息列表
        return [
            *system_msgs,
            Message(role="user", content=f"[历史对话摘要]\n{summary}"),
            *recent_msgs
        ]
```

### 设计分析

**Q: 为什么保留最近 20 条，而不是 10 条或 50 条？**

A: 这是经验值，基于以下权衡：

| 数量 | 优势 | 劣势 |
|------|------|------|
| 10 条 | 节省 token | 可能丢失关键上下文 |
| 20 条 | 平衡点 | - |
| 50 条 | 保留更多上下文 | 压缩效果有限 |

**Q: LLM 压缩会丢失信息吗？如何检测和恢复？**

A: 会丢失信息，但这是不可避免的权衡。检测和恢复策略：

1. **检测**：如果压缩后的 LLM 输出质量下降（如重复问同一问题），说明压缩丢失了关键信息
2. **恢复**：增加 `keep_recent` 数量，或者使用向量检索保留关键历史片段

**Q: 相比滑动窗口、向量检索、CRDT 摘要等方案，LLM 压缩的优势和劣势？**

A: 对比分析：

| 方案 | 优势 | 劣势 |
|------|------|------|
| 滑动窗口 | 简单，无成本 | 硬性丢弃，可能丢失关键信息 |
| 向量检索 | 保留语义相关片段 | 需要额外存储和检索，增加延迟 |
| CRDT 摘要 | 结构化，可恢复 | 实现复杂，通用性差 |
| **LLM 压缩** | 保留语义，通用性强 | 有成本，可能丢失细节 |

kimi-cli 选择 LLM 压缩是因为：
1. 实现简单
2. 适用于任意对话场景
3. 成本可接受（摘要通常只需要几百 tokens）

**Q: 这种压缩策略是否适用于长期记忆（Agent 运行数月）？**

A: 不适用。长期记忆需要：

1. **持久化存储**：将重要信息写入数据库
2. **语义检索**：根据当前问题检索相关历史
3. **层次化记忆**：区分工作记忆（当前会话）、短期记忆（最近几天）、长期记忆（所有历史）

kimi-cli 的压缩策略只解决了"工作记忆"的问题。

---

## 3.2 Checkpoint 机制

### 设计意图

Agent 执行过程中可能会出错（如工具调用失败、用户中断）。如何"回滚"到之前的状态？

### 核心实现

**源码位置**: `src/kimi_cli/soul/context.py`

```python
class Context:
    def __init__(self):
        self._messages: list[Message] = []
        self._checkpoints: dict[str, list[Message]] = {}

    def save_checkpoint(self, name: str) -> None:
        """保存当前状态快照"""
        self._checkpoints[name] = self._messages.copy()

    def restore_checkpoint(self, name: str) -> None:
        """恢复到之前的状态"""
        if name not in self._checkpoints:
            raise ValueError(f"Checkpoint {name} not found")
        self._messages = self._checkpoints[name].copy()
```

### 使用场景

```python
# 在执行危险操作前保存 checkpoint
context.save_checkpoint("before_deletion")

# 执行危险操作
result = await tool.delete_file(path)

# 如果失败，回滚
if result.is_error:
    context.restore_checkpoint("before_deletion")
```

### 设计分析

**Q: 为什么用深拷贝而非指针？**

A: 防止意外修改：

```python
# 错误：浅拷贝会导致原数据被修改
checkpoint = context._messages  # 只是引用
checkpoint.append(...)  # 会修改原数据！

# 正确：深拷贝
checkpoint = context._messages.copy()
```

**Q: Checkpoint 是否会消耗大量内存？**

A: 是的，每个 checkpoint 都保存完整的历史副本。优化策略：

1. **增量快照**：只保存新增的消息
2. **快照过期**：自动删除旧的 checkpoint
3. **压缩存储**：用 LLM 压缩后再保存

---

## 3.3 D-Mail 时间旅行

### 设计意图

"D-Mail"来自动画《命运石之门》，指"发送到过去的信息"。在 Agent 场景中，指"修改历史消息并重新生成"。

### 核心实现

**源码位置**: `src/kimi_cli/soul/dmail.py`

```python
async def dmail(
    context: Context,
    target_index: int,
    new_message: Message,
    model: str,
    tools: Toolset,
) -> Message:
    """
    修改历史消息，然后重新生成之后的所有内容

    Args:
        context: 当前上下文
        target_index: 要修改的消息索引
        new_message: 新的消息内容
        model: LLM 模型
        tools: 工具集

    Returns:
        新生成的最终响应
    """
    # 1. 删除 target_index 之后的所有消息
    context._messages = context._messages[:target_index + 1]

    # 2. 替换目标消息
    context._messages[target_index] = new_message

    # 3. 重新运行 ReAct 循环
    return await rerun_from(context, model, tools)
```

### 使用场景

**场景 1：修正用户输入**

```
用户："列出当前目录的文件"
Agent：[调用工具，返回文件列表]
用户："不对，我说的是列出 Python 文件"
```

使用 D-Mail 修正：

```python
# 将"列出当前目录的文件"改为"列出 Python 文件"
await dmail(
    context=context,
    target_index=0,  # 第一条用户消息
    new_message=Message(role="user", content="列出 Python 文件"),
    model=model,
    tools=tools,
)
# Agent 会重新调用工具，返回正确的文件列表
```

**场景 2：修正工具输出**

```
用户："帮我查一下天气"
Agent：[调用 weather 工具，但返回错误数据]
```

使用 D-Mail 修正：

```python
# 手动修正工具输出
await dmail(
    context=context,
    target_index=2,  # 工具结果消息
    new_message=Message(
        role="tool",
        content=ToolResult(tool_use_id="...", content="今天晴，25°C")
    ),
    model=model,
    tools=tools,
)
# Agent 会基于正确的数据重新生成响应
```

### 设计分析

**Q: D-Mail 和 Checkpoint 的区别？**

A: 对比：

| 特性 | Checkpoint | D-Mail |
|------|------------|--------|
| 用途 | 回滚到之前的状态 | 修改历史并重新生成 |
| 方向 | 向后（回退） | 向前（从修改点继续） |
| 数据 | 完整快照 | 增量修改 |
| 成本 | 低（只是复制） | 高（需要重新调用 LLM） |

**Q: D-Mail 的成本是什么？**

A: 修改第 i 条消息后，需要重新生成 i+1 到 N 的所有消息，成本是 O(N)。

优化策略：
1. **选择性重新生成**：只重新生成受影响的消息
2. **缓存机制**：如果某条消息的输入没变，复用之前的输出
3. **增量修改**：用 LLM 修正而非完全重新生成

---

## 本章小结

Context 与 Memory 的设计：

1. **SimpleCompaction**：用 LLM 压缩历史，平衡信息保留和 token 成本
2. **Checkpoint**：保存状态快照，支持回滚
3. **D-Mail**：修改历史并重新生成，实现"时间旅行"

这些机制让 Agent 能够处理长期运行的场景，同时保持上下文的有效性。
