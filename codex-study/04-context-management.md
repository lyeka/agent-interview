# 第四章：上下文管理

## 4.1 上下文管理的核心挑战

LLM 的上下文窗口是稀缺资源。Codex 需要在有限的 Token 预算内维持对话的连续性，同时保证关键信息不丢失。

**核心挑战**：
1. **无界增长**：对话历史会无限累积
2. **Token 估算**：需要实时追踪使用量
3. **信息保留**：截断时保留最有价值的信息
4. **状态一致性**：工具调用的 Call-Output 必须配对

---

## 4.2 ContextManager 结构

`ContextManager` 是上下文管理的核心，负责历史消息的存储、截断、规范化：

```rust
// history.rs:22-26
pub(crate) struct ContextManager {
    /// The oldest items are at the beginning of the vector.
    items: Vec<ResponseItem>,
    token_info: Option<TokenUsageInfo>,
}
```

**设计哲学**：
- **时序有序性**：oldest → newest，便于从头部删除旧消息
- **Token 感知**：内置 token 统计，实时追踪上下文窗口使用情况
- **不可变输出**：`for_prompt()` 消费 self，确保输出后不可修改

---

## 4.3 Token 估算机制

### 估算算法

```rust
// history.rs:87-117
pub(crate) fn estimate_token_count(&self, turn_context: &TurnContext) -> Option<i64> {
    // 1. 基础指令的 Token
    let base_tokens = i64::try_from(approx_token_count(&base_instructions)).unwrap_or(i64::MAX);

    // 2. 历史消息的 Token
    let items_tokens = self.items.iter().fold(0i64, |acc, item| {
        acc + match item {
            // Ghost Snapshot 不计入 Token（不发送给模型）
            ResponseItem::GhostSnapshot { .. } => 0,

            // Reasoning 内容需要特殊估算（加密格式）
            ResponseItem::Reasoning { encrypted_content: Some(content), .. } => {
                let reasoning_bytes = estimate_reasoning_length(content.len());
                i64::try_from(approx_tokens_from_byte_count(reasoning_bytes)).unwrap_or(i64::MAX)
            }

            // 其他消息：序列化为 JSON 后计算
            item => {
                let serialized = serde_json::to_string(item).unwrap_or_default();
                i64::try_from(approx_token_count(&serialized)).unwrap_or(i64::MAX)
            }
        }
    });

    Some(base_tokens.saturating_add(items_tokens))
}
```

### 字节到 Token 的转换

```rust
// 经验公式：4 bytes ≈ 1 token
pub fn approx_tokens_from_byte_count(bytes: usize) -> usize {
    bytes / 4
}

// Reasoning 内容的特殊估算（Base64 解码 + 固定开销）
fn estimate_reasoning_length(encoded_len: usize) -> usize {
    encoded_len.saturating_mul(3).checked_div(4).unwrap_or(0).saturating_sub(650)
}
```

**设计权衡**：精确的 tokenization 需要调用 tokenizer，成本高。字节估算是实用主义的胜利——牺牲精度换取性能。

---

## 4.4 截断策略

### TruncationPolicy 枚举

```rust
// truncate.rs:12-16
pub enum TruncationPolicy {
    Bytes(usize),
    Tokens(usize),
}

impl TruncationPolicy {
    pub fn token_budget(&self) -> usize {
        match self {
            TruncationPolicy::Bytes(bytes) => approx_tokens_from_byte_count(*bytes),
            TruncationPolicy::Tokens(tokens) => *tokens,
        }
    }
}
```

### 智能截断算法

Codex 的截断策略是**保留头尾，省略中间**：

```rust
// truncate.rs:186-219
fn truncate_with_byte_estimate(s: &str, policy: TruncationPolicy) -> String {
    let max_bytes = policy.byte_budget();

    // 50/50 分配给头部和尾部
    let (left_budget, right_budget) = split_budget(max_bytes);

    // 在字符边界截断
    let (removed_chars, left, right) = split_string(s, left_budget, right_budget);

    // 生成截断标记
    let marker = format_truncation_marker(policy, removed_units_for_source(...));

    // 组装输出
    assemble_truncated_output(left, right, &marker)
}

fn split_budget(budget: usize) -> (usize, usize) {
    let left = budget / 2;
    (left, budget - left)  // 处理奇数，右侧多 1
}
```

**示例输出**：
```
this is an example o…11 tokens truncated…also some other line
```

**设计哲学**：保留开头和结尾，让用户能看到问题的起始和结束，便于理解上下文。中间的细节可以通过重新查询获取。

---

## 4.5 历史规范化

### Call-Output 配对不变式

每个工具调用（FunctionCall/CustomToolCall）必须有对应的输出（FunctionCallOutput）。这是 LLM 理解对话的基础。

```rust
// normalize.rs:9-95
pub(crate) fn ensure_call_outputs_present(items: &mut Vec<ResponseItem>) {
    let mut missing_outputs_to_insert: Vec<(usize, ResponseItem)> = Vec::new();

    for (idx, item) in items.iter().enumerate() {
        match item {
            ResponseItem::FunctionCall { call_id, .. } => {
                // 检查后续是否有对应的 Output
                let has_output = items[idx+1..].iter().any(|i| {
                    matches!(i, ResponseItem::FunctionCallOutput { call_id: cid, .. } if cid == call_id)
                });

                if !has_output {
                    // 生成占位符 Output
                    missing_outputs_to_insert.push((
                        idx,
                        ResponseItem::FunctionCallOutput {
                            call_id: call_id.clone(),
                            output: FunctionCallOutputPayload {
                                content: "aborted".to_string(),
                                ..Default::default()
                            },
                        },
                    ));
                }
            }
            _ => {}
        }
    }

    // 逆序插入，避免索引偏移
    for (idx, output_item) in missing_outputs_to_insert.into_iter().rev() {
        items.insert(idx + 1, output_item);
    }
}
```

### 孤儿清理

```rust
// normalize.rs
pub(crate) fn remove_orphan_outputs(items: &mut Vec<ResponseItem>) {
    // 收集所有 Call 的 ID
    let call_ids: HashSet<_> = items.iter()
        .filter_map(|item| match item {
            ResponseItem::FunctionCall { call_id, .. } => Some(call_id.clone()),
            _ => None,
        })
        .collect();

    // 删除没有对应 Call 的 Output
    items.retain(|item| {
        match item {
            ResponseItem::FunctionCallOutput { call_id, .. } => call_ids.contains(call_id),
            _ => true,
        }
    });
}
```

**设计亮点**：
- **逆序插入**：避免索引重新计算
- **合成 Output**：缺失时自动生成 `"aborted"` 占位符
- **Debug 模式 panic**：开发时强制修复，生产环境容错

---

## 4.6 自动压缩机制

### 触发条件

```rust
// codex.rs:3319-3327
let auto_compact_limit = model_info.auto_compact_token_limit().unwrap_or(i64::MAX);
let total_usage_tokens = sess.get_total_token_usage().await;

if total_usage_tokens >= auto_compact_limit {
    run_auto_compact(&sess, &turn_context).await;
}
```

### 压缩流程

```rust
// compact.rs:43-206
pub async fn run_inline_auto_compact_task(
    sess: &Session,
    turn_context: &TurnContext,
) -> Result<(), CompactError> {
    // 1. 生成摘要
    let summary_prompt = format!(
        "{}\n\n{}",
        SUMMARIZATION_PROMPT,
        sess.clone_history().await.to_string()
    );

    let summary_response = sess.client.complete(&summary_prompt).await?;
    let summary_text = extract_summary(&summary_response)?;

    // 2. 提取用户消息（保留最近 20,000 tokens）
    let user_messages = extract_recent_user_messages(
        &sess.clone_history().await,
        COMPACT_USER_MESSAGE_MAX_TOKENS,
    );

    // 3. 构建新历史
    let new_history = build_compacted_history(
        sess.get_initial_context().await,
        &user_messages,
        &summary_text,
    );

    // 4. 替换历史
    sess.replace_history(new_history).await;

    Ok(())
}
```

### 压缩模板

**Summarization Prompt** (`templates/compact/prompt.md`)：
```markdown
You are performing a CONTEXT CHECKPOINT COMPACTION.
Create a handoff summary for another LLM that will resume the task.

Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed to continue

Be concise, structured, and focused on helping the next LLM seamlessly continue the work.
```

**Summary Prefix** (`templates/compact/summary_prefix.md`)：
```markdown
Another language model started to solve this problem and produced a summary of its thinking process.
You also have access to the state of the tools that were used by that language model.
Use this to build on the work that has already been done and avoid duplicating work.
```

**设计哲学**：压缩是有损的，但通过精心设计的 prompt，可以保留关键信息。摘要的目标是让"下一个 LLM"能够无缝接续工作。

---

## 4.7 Ghost Snapshot 机制

### 用途

Ghost Snapshot 用于记录 Git 仓库状态，支持 `undo` 功能：

```rust
ResponseItem::GhostSnapshot { ghost_commit: GhostCommit }
```

### 特殊处理

1. **不计入 Token**：`estimate_token_count` 中返回 0
2. **不发送给模型**：`for_prompt()` 中被过滤
3. **压缩时保留**：`compact.rs` 中重新添加到新历史

```rust
// history.rs:73-78
pub(crate) fn for_prompt(mut self) -> Vec<ResponseItem> {
    self.normalize_history();  // 1. 规范化
    self.items.retain(|item| !matches!(item, ResponseItem::GhostSnapshot { .. }));  // 2. 过滤快照
    self.items  // 3. 返回清洁历史
}
```

### 超时警告

```rust
// tasks/ghost_snapshot.rs
const SNAPSHOT_WARNING_THRESHOLD: Duration = Duration::from_secs(240);

// 240 秒后提示用户检查大文件
if elapsed > SNAPSHOT_WARNING_THRESHOLD {
    warn!("Ghost snapshot taking too long. Check for large files.");
}
```

---

## 4.8 Prompt 构建流程

### 完整流程

```rust
// 概念模型
pub async fn build_prompt(sess: &Session, turn_context: &TurnContext) -> Prompt {
    // 1. 获取历史（规范化 + 过滤快照）
    let history = sess.clone_history().await.for_prompt();

    // 2. 获取工具规范
    let tools = sess.tool_router.get_specs();

    // 3. 构建基础指令
    let base_instructions = build_base_instructions(
        &turn_context.model_info,
        &turn_context.personality,
        &turn_context.collaboration_mode,
        &sess.user_instructions,
        &sess.agents_md,
        &sess.skills,
    );

    Prompt {
        input: history,
        tools,
        parallel_tool_calls: true,
        base_instructions,
        personality: turn_context.personality.clone(),
        output_schema: None,
    }
}
```

### 消费式设计

`for_prompt()` 消费 `self`，确保历史只被使用一次：

```rust
pub(crate) fn for_prompt(mut self) -> Vec<ResponseItem> {
    // 消费 self，返回 Vec
}
```

调用前必须 `clone_history()`，保护原始状态：

```rust
let sampling_request_input = sess.clone_history().await.for_prompt();
```

---

## 4.9 删除最旧消息

当需要释放空间时，从头部删除最旧的消息：

```rust
// history.rs:119-129
pub(crate) fn remove_first_item(&mut self) {
    if self.items.is_empty() {
        return;
    }

    let removed = self.items.remove(0);

    // 如果删除的是 Call，同时删除对应的 Output
    if let ResponseItem::FunctionCall { call_id, .. } = &removed {
        self.items.retain(|item| {
            !matches!(item, ResponseItem::FunctionCallOutput { call_id: cid, .. } if cid == call_id)
        });
    }
}
```

**设计亮点**：删除 Call 时自动删除 Output，维护不变式，无需全量规范化。

---

## 4.10 与 Claude Code 的对比

| 维度 | Codex | Claude Code |
|------|-------|-------------|
| **Token 估算** | 字节近似（4 bytes/token） | 类似 |
| **截断策略** | 保留头尾，省略中间 | 类似 |
| **自动压缩** | LLM 生成 handoff summary | 自动摘要 |
| **快照机制** | Ghost Snapshot（Git 状态） | 无 |
| **规范化** | Call-Output 配对检查 | 类似 |

**关键差异**：Codex 的 Ghost Snapshot 机制是独特的，它将 Git 状态与对话历史解耦，支持 undo 而不污染上下文。

---

## 章节衔接

**本章回顾**：
- 我们深入分析了上下文管理的设计
- 关键收获：Token 估算 + 智能截断 + 自动压缩 + 规范化

**下一章预告**：
- 在 `05-prompt-engineering.md` 中，我们将深入 Prompt 工程
- 为什么需要学习：Prompt 是 Agent 的"灵魂"，决定了 Agent 的行为和风格
- 关键问题：如何组织分层 Prompt？如何实现人格切换？如何注入项目规范？
