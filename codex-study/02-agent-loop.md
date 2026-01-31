# 第二章：Agent 主循环

## 2.1 循环架构概览

Codex 的 Agent 循环不是经典的 ReAct（Reason-Act-Observe）模式，而是自定义的 **Sampling-Tool-Execution Loop**。

```
┌─────────────────────────────────────────────────────────────┐
│                    submission_loop()                         │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  while let Ok(sub) = rx_sub.recv().await {          │    │
│  │      match sub.op {                                  │    │
│  │          Op::UserInput => run_turn(...)             │    │
│  │          Op::Interrupt => interrupt(...)            │    │
│  │          Op::Shutdown => break                      │    │
│  │      }                                              │    │
│  │  }                                                  │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       run_turn()                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  loop {                                              │    │
│  │      1. 检查自动压缩                                  │    │
│  │      2. 注入 Skill 指令                              │    │
│  │      3. 记录用户输入                                  │    │
│  │      4. run_sampling_request()                       │    │
│  │      5. 处理结果：                                    │    │
│  │         - needs_follow_up=true → continue            │    │
│  │         - needs_follow_up=false → break              │    │
│  │         - error → handle & break                     │    │
│  │  }                                                   │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  run_sampling_request()                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 构建 Prompt                                      │    │
│  │  2. 发起流式请求                                      │    │
│  │  3. 处理流式事件：                                    │    │
│  │     - OutputTextDelta → 推送文本                      │    │
│  │     - FunctionCall → 执行工具                         │    │
│  │     - Done → 返回结果                                 │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2.2 Submission Loop：事件调度中心

`submission_loop` 是 Agent 的事件调度中心，负责接收用户提交并分发处理。

### 核心实现

```rust
// codex.rs:2386-2509
async fn submission_loop(
    sess: Arc<Session>,
    config: Arc<Config>,
    rx_sub: Receiver<Submission>
) {
    let mut previous_context: Option<Arc<TurnContext>> = Some(sess.new_default_turn().await);

    // 核心事件循环 - 阻塞等待用户提交
    while let Ok(sub) = rx_sub.recv().await {
        match sub.op.clone() {
            Op::Interrupt => {
                handlers::interrupt(&sess).await;
            }

            Op::UserInput { .. } | Op::UserTurn { .. } => {
                handlers::user_input_or_turn(
                    &sess,
                    sub.id.clone(),
                    sub.op,
                    &mut previous_context
                ).await;
            }

            Op::ExecApproval { id, decision } => {
                handlers::exec_approval(&sess, id, decision).await;
            }

            Op::Shutdown => {
                if handlers::shutdown(&sess, sub.id.clone()).await {
                    break; // 唯一退出点
                }
            }

            // ... 其他 Op 处理
        }
    }
}
```

### 设计决策分析

**为什么用 `previous_context`？**

`previous_context` 在循环外维护，支持配置继承。当用户连续发送多个请求时，后续请求可以复用前一个 Turn 的配置（如沙箱策略、工作目录）。

**为什么 Shutdown 是唯一退出点？**

这是防御性设计。即使发生错误，循环也不会退出，而是记录错误并继续等待下一个提交。只有显式的 Shutdown 操作才能终止循环，确保 Agent 的稳定性。

**代码坏味道**：这个 match 有 15+ 个分支，应该用 Command Pattern 重构。每个 Op 应该是一个实现 `Handler` trait 的结构体，而不是内联的 match 分支。

---

## 2.3 run_turn：Turn 执行核心

`run_turn` 是单次 Turn 的执行核心，协调 LLM 调用、工具执行、上下文管理。

### 核心实现

```rust
// codex.rs:3308-3517
pub(crate) async fn run_turn(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    input: Vec<UserInput>,
    cancellation_token: CancellationToken,
) -> Option<String> {
    // 1. 发送 TurnStarted 事件
    sess.send_event(&turn_context, EventMsg::TurnStarted(...)).await;

    // 2. 自动压缩检查
    let auto_compact_limit = model_info.auto_compact_token_limit().unwrap_or(i64::MAX);
    let total_usage_tokens = sess.get_total_token_usage().await;

    if total_usage_tokens >= auto_compact_limit {
        run_auto_compact(&sess, &turn_context).await;
    }

    // 3. Skill 注入和依赖解析
    let mentioned_skills = collect_explicit_skill_mentions(&input, &config);
    let skill_items = build_skill_injections(&mentioned_skills, &sess, &turn_context).await;

    // 4. 记录用户输入到历史
    sess.record_user_prompt_and_emit_turn_item(&turn_context, &input, &skill_items).await;

    // 5. 主采样循环
    let mut last_agent_message: Option<String> = None;

    loop {
        // 支持中途注入新输入
        let pending_input = sess.get_pending_input().await;

        // 构建采样请求输入
        let sampling_request_input = sess.clone_history().await.for_prompt();

        match run_sampling_request(
            &sess,
            &turn_context,
            sampling_request_input,
            cancellation_token.clone(),
        ).await {
            Ok(SamplingRequestResult { needs_follow_up, last_agent_message: msg }) => {
                last_agent_message = msg;

                // Token 超限时自动压缩后重试
                if token_limit_reached && needs_follow_up {
                    run_auto_compact(&sess, &turn_context).await;
                    continue;
                }

                if !needs_follow_up {
                    break; // 正常完成
                }
            }

            Err(CodexErr::TurnAborted) => break,

            Err(CodexErr::InvalidImageRequest()) => {
                // 清理无效图片后重试
                state.history.replace_last_turn_images("Invalid image");
                continue;
            }

            Err(e) => {
                sess.send_event(&turn_context, EventMsg::Error(...)).await;
                break;
            }
        }
    }

    last_agent_message
}
```

### 状态机视角

这段代码是一个**隐式状态机**。每次 loop 迭代代表一个状态转换：

| 状态 | 触发条件 | 转换目标 |
|------|----------|----------|
| **工具调用** | `needs_follow_up=true` | 继续循环 |
| **压缩** | `token_limit_reached` | 压缩后继续 |
| **清理** | `InvalidImageRequest` | 清理后继续 |
| **完成** | `needs_follow_up=false` | 退出循环 |
| **中止** | `TurnAborted` | 退出循环 |
| **错误** | 其他错误 | 发送错误事件后退出 |

**设计缺陷**：状态转换逻辑分散在 match 分支中，缺乏显式状态枚举。如果重构，应该定义：

```rust
enum TurnState {
    Sampling,
    ToolExecution,
    Compacting,
    Cleaning,
    Completed,
    Aborted,
    Error(CodexErr),
}
```

---

## 2.4 run_sampling_request：采样请求处理

`run_sampling_request` 负责与 LLM 通信，处理流式响应和工具调用。

### 核心实现

```rust
// codex.rs:4175-4454
async fn try_run_sampling_request(
    router: Arc<ToolRouter>,
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    client_session: &mut ModelClientSession,
    prompt: &Prompt,
    cancellation_token: CancellationToken,
) -> CodexResult<SamplingRequestResult> {
    // 1. 发起流式请求
    let mut stream = client_session.stream(prompt)
        .or_cancel(&cancellation_token).await??;

    // 2. 初始化工具调用运行时
    let tool_runtime = ToolCallRuntime::new(router, sess, turn_context);
    let mut in_flight: FuturesOrdered<BoxFuture<'static, CodexResult<ResponseInputItem>>> =
        FuturesOrdered::new();

    // 3. 流式事件处理循环
    let mut needs_follow_up = false;

    loop {
        tokio::select! {
            // 分支 1：处理流式事件
            event = stream.next() => {
                match event {
                    Some(ResponseEvent::OutputTextDelta(delta)) => {
                        // 推送文本增量给用户
                        sess.emit_text_delta(&turn_context, &delta).await;
                    }

                    Some(ResponseEvent::FunctionCall { name, arguments, call_id }) => {
                        // 启动工具调用（异步）
                        let future = tool_runtime.handle_tool_call(
                            ToolCall { tool_name: name, call_id, arguments },
                            cancellation_token.clone(),
                        );
                        in_flight.push_back(Box::pin(future));
                        needs_follow_up = true;
                    }

                    Some(ResponseEvent::Done) => {
                        // 等待所有工具调用完成
                        while let Some(result) = in_flight.next().await {
                            let output = result?;
                            sess.record_tool_output(&turn_context, output).await;
                        }
                        break;
                    }

                    None => break,
                }
            }

            // 分支 2：处理已完成的工具调用
            Some(result) = in_flight.next(), if !in_flight.is_empty() => {
                let output = result?;
                sess.record_tool_output(&turn_context, output).await;
            }

            // 分支 3：用户取消
            _ = cancellation_token.cancelled() => {
                return Err(CodexErr::TurnAborted);
            }
        }
    }

    Ok(SamplingRequestResult { needs_follow_up, last_agent_message })
}
```

### 并发模型分析

这段代码展示了 Rust 异步编程的精髓：

1. **tokio::select!**：同时监听多个异步事件，哪个先完成就处理哪个
2. **FuturesOrdered**：保持工具调用的顺序，但允许并行执行
3. **CancellationToken**：支持用户随时取消，优雅中止

**为什么用 FuturesOrdered 而不是 FuturesUnordered？**

工具调用的结果需要按顺序记录到历史，以保持对话的逻辑一致性。如果用 FuturesUnordered，结果顺序不确定，可能导致 LLM 混淆。

---

## 2.5 工具调用的触发与执行

当 LLM 返回 `FunctionCall` 事件时，Agent 需要执行对应的工具。

### 工具调用流程

```
LLM Response: FunctionCall { name: "shell", arguments: "{\"command\": \"ls -la\"}" }
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  ToolCallRuntime.handle_tool_call()                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 解析参数                                         │    │
│  │  2. 获取读锁/写锁（并行控制）                         │    │
│  │  3. 调用 ToolRouter.dispatch_tool_call()             │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  ToolRouter.dispatch_tool_call()                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 查找 handler                                     │    │
│  │  2. 验证 payload 类型                                │    │
│  │  3. 等待工具门（mutating 工具串行）                   │    │
│  │  4. 调用 handler.handle()                            │    │
│  │  5. 转换输出格式                                     │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  ToolOrchestrator.run()                                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  1. 审批决策                                         │    │
│  │  2. 沙箱选择                                         │    │
│  │  3. 首次尝试执行                                     │    │
│  │  4. 失败时升级（无沙箱重试）                          │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 并行执行的读写锁模式

```rust
// parallel.rs:79-90
let _guard = if supports_parallel {
    Either::Left(lock.read().await)   // 并行工具：读锁
} else {
    Either::Right(lock.write().await) // 串行工具：写锁
};

router.dispatch_tool_call(session, turn, tracker, call.clone()).await
```

**设计哲学**：
- **read_file、grep_files**：只读操作，可以并行执行
- **shell、apply_patch**：可能修改文件系统，必须串行执行

这是一个经典的读写锁模式，用最小的同步开销实现最大的并发度。

---

## 2.6 错误处理与重试

Agent 循环需要处理多种错误场景：

### 错误分类

| 错误类型 | 处理策略 | 示例 |
|----------|----------|------|
| **TurnAborted** | 静默退出 | 用户按 Ctrl+C |
| **InvalidImageRequest** | 清理后重试 | 图片格式不支持 |
| **TokenLimitReached** | 压缩后重试 | 上下文超限 |
| **FunctionCallError::Fatal** | 中止会话 | 工具不存在 |
| **FunctionCallError::RespondToModel** | 返回给 LLM | 参数解析失败 |
| **NetworkError** | 重试或报错 | API 超时 |

### 错误恢复示例

```rust
// 图片错误恢复
Err(CodexErr::InvalidImageRequest()) => {
    // 不是直接失败，而是清理无效图片后重试
    state.history.replace_last_turn_images("Invalid image");
    continue;
}

// Token 超限恢复
if token_limit_reached && needs_follow_up {
    // 不是直接失败，而是压缩历史后重试
    run_auto_compact(&sess, &turn_context).await;
    continue;
}
```

**设计哲学**：Agent 应该尽可能自我恢复，而不是把错误抛给用户。只有真正无法恢复的错误才应该中止会话。

---

## 2.7 中断与取消

用户可以随时中断正在执行的操作：

### 取消机制

```rust
// 创建取消令牌
let cancellation_token = CancellationToken::new();

// 在 select! 中监听取消
tokio::select! {
    event = stream.next() => { /* 处理事件 */ }
    _ = cancellation_token.cancelled() => {
        return Err(CodexErr::TurnAborted);
    }
}

// 用户按 Ctrl+C 时触发取消
Op::Interrupt => {
    cancellation_token.cancel();
}
```

### 优雅关闭

```rust
// 工具调用的取消处理
let handle: AbortOnDropHandle<...> = AbortOnDropHandle::new(tokio::spawn(async move {
    tokio::select! {
        _ = cancellation_token.cancelled() => {
            // 返回 "aborted" 结果，而不是 panic
            Ok(Self::aborted_response(&call, elapsed_secs))
        },
        res = actual_execution => res,
    }
}));
```

**设计哲学**：取消不是错误，而是正常的控制流。被取消的工具调用应该返回 "aborted" 结果，让 LLM 知道操作被中断，而不是静默失败。

---

## 2.8 与 ReAct 模式的对比

| 维度 | ReAct | Codex Loop |
|------|-------|------------|
| **循环结构** | Reason → Act → Observe → Reason... | Sample → Tool → Record → Sample... |
| **推理显式化** | 显式 Thought 步骤 | 隐式（在 LLM 内部） |
| **工具调用** | 单次调用 | 支持并行调用 |
| **状态管理** | 简单（当前观察） | 复杂（历史、Token、沙箱） |
| **错误恢复** | 通常重新开始 | 多种恢复策略 |

**为什么 Codex 不用 ReAct？**

1. **性能**：ReAct 的显式推理步骤增加了 Token 消耗和延迟
2. **并行**：ReAct 是串行的，Codex 需要并行工具调用
3. **复杂度**：Codex 的状态管理（Token、沙箱、审批）超出了 ReAct 的简单模型

---

## 章节衔接

**本章回顾**：
- 我们深入分析了 Agent 主循环的实现
- 关键收获：Sampling-Tool-Execution Loop + 队列对模式 + 并发工具执行

**下一章预告**：
- 在 `03-tools-system.md` 中，我们将深入工具系统的设计
- 为什么需要学习：工具是 Agent 与外部世界交互的桥梁
- 关键问题：16 个内置工具如何注册和分发？审批和沙箱如何协作？
