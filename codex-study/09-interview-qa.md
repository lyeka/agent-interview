# 第九章：面试 QA

## 9.1 架构设计类（5 题）

### Q1: Codex 为什么选择 Rust 而不是 Python/TypeScript？

**参考答案**：

Codex 选择 Rust 是深思熟虑的架构决策，主要基于三个考量：

1. **内存安全**：Agent 需要长时间运行，Rust 的所有权系统从编译期避免内存泄漏和数据竞争。Python 的 GC 可能导致不可预测的暂停，TypeScript 的 V8 内存管理在长时间运行时可能出现问题。

2. **跨平台沙箱**：Codex 实现了原生沙箱（macOS Seatbelt、Linux Landlock、Windows AppContainer），需要系统级编程能力。Rust 的 FFI 和条件编译（`#[cfg(target_os = "...")]`）完美适配这个需求。

3. **并发性能**：工具调用需要并行执行，Rust 的 async/await + Tokio 提供零成本抽象。`tokio::select!` 可以同时监听流式响应、工具完成、用户取消，这在 Python 的 asyncio 中实现会更复杂。

**源码引用**：`codex-rs/core/src/codex.rs` 的 `try_run_sampling_request` 函数展示了 Rust 并发模型的优势。

---

### Q2: 解释 Codex 的队列对模式（Submission/Event Queue Pair）

**参考答案**：

队列对模式是 Codex 的核心通信架构：

```
User Interface ──[Submission]──► Codex Engine
User Interface ◄──[Event]────── Codex Engine
```

**为什么不用传统的请求-响应模式？**

1. **长时间运行**：一次 Turn 可能执行多个工具调用，持续数分钟。请求-响应模式会阻塞调用方。

2. **流式输出**：LLM 响应是流式的（SSE），需要实时推送给用户。队列对模式天然支持流式事件。

3. **中断支持**：用户可以随时发送 `Op::Interrupt`，Agent 通过 `CancellationToken` 优雅中止。

4. **事件驱动**：工具执行、审批请求、进度更新都是异步事件，队列对模式统一处理。

**源码引用**：`codex.rs:2386-2509` 的 `submission_loop` 函数实现了这个模式。

---

### Q3: Codex 的 Agent 循环与 ReAct 有什么区别？

**参考答案**：

| 维度 | ReAct | Codex Loop |
|------|-------|------------|
| **循环结构** | Reason → Act → Observe → Reason... | Sample → Tool → Record → Sample... |
| **推理显式化** | 显式 Thought 步骤 | 隐式（在 LLM 内部） |
| **工具调用** | 单次调用 | 支持并行调用 |
| **状态管理** | 简单（当前观察） | 复杂（历史、Token、沙箱） |

**为什么 Codex 不用 ReAct？**

1. **性能**：ReAct 的显式推理步骤增加了 Token 消耗和延迟。Codex 让 LLM 内部完成推理，只输出行动。

2. **并行**：ReAct 是串行的，Codex 使用 `FuturesOrdered` 支持并行工具调用。

3. **复杂度**：Codex 的状态管理（Token 估算、自动压缩、沙箱审批）超出了 ReAct 的简单模型。

**源码引用**：`codex.rs:3308-3517` 的 `run_turn` 函数展示了 Codex 的循环实现。

---

### Q4: 如何理解 Codex 的分层 Prompt 架构？

**参考答案**：

Codex 的 Prompt 是分层组装的：

```
1. Base Instructions      → Agent 能力定义
2. Model-Specific         → 模型专用指令 + {{ personality }}
3. Personality            → Friendly / Pragmatic
4. Collaboration Mode     → Plan / Execute / Pair Programming / Code
5. User Instructions      → 用户自定义
6. AGENTS.md              → 项目规范（分层覆盖）
7. Skills                 → 可用 Skills 清单
```

**设计哲学**：

1. **可组合性**：不是一个 10000 行的超级 Prompt，而是 10 个模块化 Prompt，运行时组装。

2. **关注点分离**：Base 定义能力，Personality 定义风格，Collaboration Mode 定义工作流。

3. **优先级链**：User 直接指令 > AGENTS.md（深层 > 浅层）> Collaboration Mode > Personality > Base。

**源码引用**：`core/templates/` 目录包含所有 Prompt 模板，`project_doc.rs` 实现 AGENTS.md 注入。

---

### Q5: Codex 如何实现跨平台沙箱？

**参考答案**：

Codex 实现了三层沙箱防御：

| 平台 | 技术 | 实现文件 |
|------|------|----------|
| macOS | Seatbelt (TrustedBSD MAC) | `seatbelt.rs` |
| Linux | Landlock + Seccomp | `linux-sandbox/` |
| Windows | AppContainer / RestrictedToken | `windows_sandbox.rs` |

**macOS Seatbelt 示例**：

```scheme
(version 1)
(deny default)  ; 默认拒绝
(allow file-read*)  ; 允许读
(allow file-write* (subpath "/workspace"))  ; 限制写
(require-not (subpath "/workspace/.git"))  ; 排除 .git
```

**设计亮点**：

1. **路径规范化**：使用 `canonicalize()` 避免符号链接绕过
2. **保护关键目录**：禁止写入 `.git/hooks` 和 `.codex/config.toml`
3. **沙箱升级**：失败后可请求用户批准无沙箱重试

**源码引用**：`seatbelt.rs:46-135` 的 `create_seatbelt_command_args` 函数。

---

## 9.2 工具系统类（4 题）

### Q6: Codex 的工具路由是如何实现的？

**参考答案**：

工具路由分三步：

1. **解析**：`build_tool_call()` 将 LLM 响应解析为统一的 `ToolCall` 结构
2. **查找**：`ToolRegistry.handler()` 根据工具名查找 handler
3. **分发**：`dispatch_tool_call()` 执行工具并返回结果

**关键设计**：

```rust
#[async_trait]
pub trait ToolHandler: Send + Sync {
    fn kind(&self) -> ToolKind;
    async fn is_mutating(&self, invocation: &ToolInvocation) -> bool;
    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError>;
}
```

通过 trait 抽象消除工具类型的特殊处理，新增工具只需实现接口。

**源码引用**：`router.rs:61-130` 和 `registry.rs:67-149`。

---

### Q7: 工具的并行执行是如何控制的？

**参考答案**：

Codex 使用读写锁模式控制并行：

```rust
let _guard = if supports_parallel {
    Either::Left(lock.read().await)   // 并行工具：读锁
} else {
    Either::Right(lock.write().await) // 串行工具：写锁
};
```

**并行工具**：`read_file`、`grep_files`、`list_dir`（只读操作）
**串行工具**：`shell`、`apply_patch`（可能修改文件系统）

**设计哲学**：用最小的同步开销实现最大的并发度。读锁允许多个并行工具同时执行，写锁确保串行工具独占执行。

**源码引用**：`parallel.rs:79-90`。

---

### Q8: 审批缓存机制是如何工作的？

**参考答案**：

审批结果会被缓存，避免重复询问用户：

```rust
pub(crate) async fn with_cached_approval<K, F, Fut>(
    services: &SessionServices,
    tool_name: &str,
    keys: Vec<K>,  // 支持多个审批键
    fetch: F,
) -> ReviewDecision {
    // 1. 检查所有键是否已批准
    let already_approved = keys.iter().all(|key| {
        matches!(store.get(key), Some(ReviewDecision::ApprovedForSession))
    });

    if already_approved {
        return ReviewDecision::ApprovedForSession;
    }

    // 2. 请求用户审批
    let decision = fetch().await;

    // 3. 缓存会话级审批
    if matches!(decision, ReviewDecision::ApprovedForSession) {
        for key in keys {
            store.put(key, ReviewDecision::ApprovedForSession);
        }
    }

    decision
}
```

**批量语义**：`apply_patch` 修改多文件时，所有文件都已批准才跳过询问。

**源码引用**：`sandboxing.rs:58-104`。

---

### Q9: MCP 工具是如何转换为 OpenAI 格式的？

**参考答案**：

MCP 工具需要转换为 OpenAI 的 function calling 格式：

```rust
pub(crate) fn mcp_tool_to_openai_tool(
    fully_qualified_name: String,
    tool: mcp_types::Tool,
) -> Result<ResponsesApiTool, serde_json::Error> {
    // 1. 强制添加 properties 字段
    if input_schema.properties.is_none() {
        input_schema.properties = Some(json!({}));
    }

    // 2. 清洗 JSON Schema
    sanitize_json_schema(&mut input_schema);

    // 3. 类型推断
    if !map.contains_key("type") {
        let inferred_type = if map.contains_key("properties") {
            "object"
        } else if map.contains_key("items") {
            "array"
        } else {
            "string"
        };
        map.insert("type", inferred_type);
    }

    Ok(ResponsesApiTool { name, description, parameters: input_schema, strict: false })
}
```

**工具名称限定**：`mcp__<server>__<tool>` 格式避免冲突。

**源码引用**：`spec.rs:1088-1179`。

---

## 9.3 上下文管理类（3 题）

### Q10: Codex 如何估算 Token 数量？

**参考答案**：

Codex 使用字节近似估算：

```rust
// 经验公式：4 bytes ≈ 1 token
pub fn approx_tokens_from_byte_count(bytes: usize) -> usize {
    bytes / 4
}

pub fn estimate_token_count(&self) -> Option<i64> {
    let items_tokens = self.items.iter().fold(0i64, |acc, item| {
        acc + match item {
            ResponseItem::GhostSnapshot { .. } => 0,  // 快照不计入
            item => {
                let serialized = serde_json::to_string(item).unwrap_or_default();
                approx_token_count(&serialized)
            }
        }
    });
    Some(base_tokens + items_tokens)
}
```

**Trade-off**：精确 tokenization 需要调用 tokenizer，成本高。字节估算牺牲精度换取性能，误差在 10-20% 范围内。

**源码引用**：`history.rs:87-117`。

---

### Q11: 自动压缩（Auto Compact）是如何工作的？

**参考答案**：

当 Token 超限时，Codex 自动压缩历史：

1. **触发条件**：`total_usage_tokens >= auto_compact_limit`

2. **压缩流程**：
   - 向 LLM 发送 `SUMMARIZATION_PROMPT`，生成 handoff summary
   - 提取最近 20,000 tokens 的用户消息
   - 构建新历史：initial_context + user_messages + summary

3. **摘要前缀**：
   ```markdown
   Another language model started to solve this problem and produced a summary...
   Use this to build on the work that has already been done.
   ```

**设计哲学**：压缩是有损的，但通过精心设计的 prompt，保留关键信息让"下一个 LLM"能无缝接续。

**源码引用**：`compact.rs:43-206`。

---

### Q12: 历史规范化（Normalize）解决什么问题？

**参考答案**：

历史规范化维护 **Call-Output 配对不变式**：

```rust
pub(crate) fn ensure_call_outputs_present(items: &mut Vec<ResponseItem>) {
    for (idx, item) in items.iter().enumerate() {
        if let ResponseItem::FunctionCall { call_id, .. } = item {
            if !has_output {
                // 生成占位符 Output
                missing_outputs.push((idx, ResponseItem::FunctionCallOutput {
                    call_id: call_id.clone(),
                    output: FunctionCallOutputPayload { content: "aborted".to_string(), .. },
                }));
            }
        }
    }

    // 逆序插入，避免索引偏移
    for (idx, output) in missing_outputs.into_iter().rev() {
        items.insert(idx + 1, output);
    }
}
```

**问题场景**：工具调用被中断时，可能留下没有 Output 的 Call，导致 LLM 混淆。

**源码引用**：`normalize.rs:9-95`。

---

## 9.4 安全与协议类（3 题）

### Q13: 执行策略引擎（Exec Policy）如何工作？

**参考答案**：

执行策略引擎是沙箱系统的"大脑"：

```rust
pub(crate) async fn create_exec_approval_requirement_for_command(
    &self,
    req: ExecApprovalRequest<'_>,
) -> ExecApprovalRequirement {
    // 1. 解析复合命令
    let commands = parse_shell_lc_plain_commands(command);

    // 2. 策略匹配
    let evaluation = exec_policy.check_multiple(commands.iter(), &fallback);

    // 3. 决策
    match evaluation.decision {
        Decision::Forbidden => ExecApprovalRequirement::Forbidden { reason },
        Decision::Prompt => ExecApprovalRequirement::NeedsApproval { reason, proposed_amendment },
        Decision::Allow => ExecApprovalRequirement::Skip { bypass_sandbox },
    }
}
```

**策略文件格式**：
```
allow git *
prompt rm -rf *
forbid curl * | bash
```

**热更新**：用户审批后立即追加规则到文件并更新内存策略。

**源码引用**：`exec_policy.rs:122-189`。

---

### Q14: Codex 如何同时作为 MCP 客户端和服务器？

**参考答案**：

**作为客户端**：
- `McpConnectionManager` 管理多个 MCP 服务器连接
- 工具名称限定：`mcp__<server>__<tool>`
- 支持 OAuth 认证和 Token 自动刷新

**作为服务器**：
- `mcp-server/` 实现 JSON-RPC over stdio
- 暴露 `codex` 和 `codex-reply` 两个工具
- 支持被其他 MCP 客户端调用

**架构意义**：Codex 可以作为"Agent 中的 Agent"被编排，实现 Multi-Agent 协作。

**源码引用**：`mcp_connection_manager.rs` 和 `mcp-server/src/lib.rs`。

---

### Q15: 如果让你重构 Codex，你会改进什么？

**参考答案**：

1. **拆分 codex.rs**：6094 行的"上帝类"应该拆分为 `session.rs`、`turn.rs`、`sampling.rs`、`handlers.rs`。

2. **显式状态机**：`run_turn` 的隐式状态机应该用显式枚举表达：
   ```rust
   enum TurnState {
       Sampling,
       ToolExecution,
       Compacting,
       Completed,
       Aborted,
       Error(CodexErr),
   }
   ```

3. **Command Pattern**：`submission_loop` 的 15+ 个 match 分支应该用 Command Pattern 重构。

4. **Token 估算优化**：可以引入轻量级 tokenizer（如 tiktoken-rs）提高估算精度。

5. **测试覆盖**：增加集成测试，特别是沙箱和审批流程的端到端测试。

---

## 章节衔接

**本章回顾**：
- 我们整理了 15 道核心面试题
- 关键收获：架构设计 + 工具系统 + 上下文管理 + 安全协议

**下一章预告**：
- 在 `10-summary.md` 中，我们将构建完整的知识图谱
- 为什么需要学习：知识图谱帮助建立系统性认知
- 关键问题：如何将零散知识点串联成完整体系？
