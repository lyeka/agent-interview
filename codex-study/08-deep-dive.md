# 第八章：核心源码深度解析

## 8.1 关键文件索引

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `codex.rs` | 6094 | Agent 主循环、Turn 执行、采样请求 |
| `spec.rs` | 2638 | 工具规范定义、MCP 转换 |
| `exec_policy.rs` | 1390 | 执行策略引擎 |
| `seatbelt.rs` | 624 | macOS 沙箱 |
| `history.rs` | ~400 | 上下文管理 |
| `router.rs` | 191 | 工具路由 |
| `orchestrator.rs` | 173 | 审批+沙箱编排 |

---

## 8.2 codex.rs 深度解析

### 文件结构

```rust
// codex.rs 结构概览（6094 行）

// ========== 数据结构定义 ==========
pub struct Codex { ... }                    // 行 100-150
pub struct Session { ... }                  // 行 200-300
pub struct TurnContext { ... }              // 行 350-450

// ========== 公共 API ==========
impl Codex {
    pub fn new(...) -> Self { ... }         // 行 500-600
    pub async fn submit(...) { ... }        // 行 650-700
}

// ========== 核心循环 ==========
async fn submission_loop(...) { ... }       // 行 2386-2509
async fn run_turn(...) { ... }              // 行 3308-3517
async fn run_sampling_request(...) { ... }  // 行 4175-4454

// ========== 辅助函数 ==========
mod handlers { ... }                        // 行 2512-3300
mod helpers { ... }                         // 行 4500-6000
```

### 关键代码段 1：Session 初始化

```rust
// codex.rs:500-600（概念模型）
impl Session {
    pub async fn new(config: Config) -> Result<Self, SessionError> {
        // 1. 加载执行策略
        let exec_policy = ExecPolicyManager::load(&config.config_stack).await?;

        // 2. 初始化 MCP 连接
        let mcp_manager = McpConnectionManager::new(&config.mcp_servers).await?;

        // 3. 构建工具路由
        let mcp_tools = mcp_manager.list_all_tools().await;
        let tool_router = ToolRouter::new(
            build_specs(&config.tools, Some(mcp_tools), &config.dynamic_tools)
        );

        // 4. 初始化上下文管理器
        let history = ContextManager::new();

        // 5. 创建事件通道
        let (event_tx, event_rx) = mpsc::channel(256);

        Ok(Self {
            config: Arc::new(config),
            exec_policy,
            mcp_manager,
            tool_router: Arc::new(tool_router),
            history: RwLock::new(history),
            event_sender: event_tx,
            // ...
        })
    }
}
```

### 关键代码段 2：Turn 状态机

```rust
// codex.rs:3308-3517
pub(crate) async fn run_turn(
    sess: Arc<Session>,
    turn_context: Arc<TurnContext>,
    input: Vec<UserInput>,
    cancellation_token: CancellationToken,
) -> Option<String> {
    // 状态：初始化
    sess.send_event(&turn_context, EventMsg::TurnStarted(...)).await;

    // 状态：自动压缩检查
    let total_tokens = sess.get_total_token_usage().await;
    if total_tokens >= auto_compact_limit {
        // 状态转换：压缩
        run_auto_compact(&sess, &turn_context).await;
    }

    // 状态：Skill 注入
    let mentioned_skills = collect_explicit_skill_mentions(&input, &config);
    let skill_items = build_skill_injections(&mentioned_skills, &sess, &turn_context).await;

    // 状态：记录输入
    sess.record_user_prompt_and_emit_turn_item(&turn_context, &input, &skill_items).await;

    // 主循环：采样-工具-执行
    let mut last_agent_message: Option<String> = None;

    loop {
        let sampling_request_input = sess.clone_history().await.for_prompt();

        match run_sampling_request(&sess, &turn_context, sampling_request_input, cancellation_token.clone()).await {
            Ok(SamplingRequestResult { needs_follow_up, last_agent_message: msg }) => {
                last_agent_message = msg;

                // 状态转换：Token 超限 → 压缩后重试
                if token_limit_reached && needs_follow_up {
                    run_auto_compact(&sess, &turn_context).await;
                    continue;
                }

                // 状态转换：完成
                if !needs_follow_up {
                    break;
                }

                // 状态转换：继续（工具调用后）
                continue;
            }

            // 状态转换：中止
            Err(CodexErr::TurnAborted) => break,

            // 状态转换：清理后重试
            Err(CodexErr::InvalidImageRequest()) => {
                state.history.replace_last_turn_images("Invalid image");
                continue;
            }

            // 状态转换：错误
            Err(e) => {
                sess.send_event(&turn_context, EventMsg::Error(...)).await;
                break;
            }
        }
    }

    last_agent_message
}
```

### 关键代码段 3：流式响应处理

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

    // 2. 初始化并发工具执行
    let tool_runtime = ToolCallRuntime::new(router, sess.clone(), turn_context.clone());
    let mut in_flight: FuturesOrdered<BoxFuture<'static, CodexResult<ResponseInputItem>>> =
        FuturesOrdered::new();

    let mut needs_follow_up = false;

    // 3. 事件处理循环
    loop {
        tokio::select! {
            // 分支 1：流式事件
            event = stream.next() => {
                match event {
                    Some(ResponseEvent::OutputTextDelta(delta)) => {
                        // 推送文本增量
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

            // 分支 2：已完成的工具调用
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

---

## 8.3 spec.rs 深度解析

### 工具规范构建

```rust
// spec.rs:1255-1447
pub(crate) fn build_specs(
    config: &ToolsConfig,
    mcp_tools: Option<HashMap<String, mcp_types::Tool>>,
    dynamic_tools: &[DynamicToolSpec],
) -> ToolRegistryBuilder {
    let mut builder = ToolRegistryBuilder::new();

    // ========== Shell 工具 ==========
    // 根据平台能力选择实现
    match &config.shell_type {
        ConfigShellToolType::UnifiedExec => {
            // 新架构：exec_command + write_stdin
            builder.push_spec(create_exec_command_tool(config.request_rule_enabled));
            builder.push_spec(create_write_stdin_tool());
            builder.register_handler("exec_command", unified_exec_handler.clone());
            builder.register_handler("write_stdin", unified_exec_handler);
        }
        ConfigShellToolType::ShellCommand => {
            // 旧架构：shell_command
            builder.push_spec(create_shell_command_tool(config.request_rule_enabled));
        }
        ConfigShellToolType::Disabled => {}
    }

    // 向后兼容：注册别名
    if config.shell_type != ConfigShellToolType::Disabled {
        builder.register_handler("shell", shell_handler.clone());
        builder.register_handler("container.exec", shell_handler.clone());
        builder.register_handler("local_shell", shell_handler);
    }

    // ========== 文件工具 ==========
    builder.push_spec_with_parallel_support(create_read_file_tool(), true);
    builder.push_spec_with_parallel_support(create_grep_files_tool(), true);
    builder.push_spec_with_parallel_support(create_list_dir_tool(), true);
    builder.push_spec(create_apply_patch_tool());

    // ========== MCP 工具 ==========
    if let Some(mcp_tools) = mcp_tools {
        for (name, tool) in mcp_tools {
            match mcp_tool_to_openai_tool(name.clone(), tool) {
                Ok(converted_tool) => {
                    builder.push_spec(ToolSpec::Function(converted_tool));
                    builder.register_handler(name, mcp_handler.clone());
                }
                Err(e) => {
                    tracing::error!("Failed to convert MCP tool {name:?}: {e:?}");
                    // 错误容忍：继续处理其他工具
                }
            }
        }
    }

    // ========== 动态工具 ==========
    for tool in dynamic_tools {
        match dynamic_tool_to_openai_tool(tool) {
            Ok(converted_tool) => {
                builder.push_spec(ToolSpec::Function(converted_tool));
                builder.register_handler(tool.name.clone(), dynamic_tool_handler.clone());
            }
            Err(e) => {
                tracing::error!("Failed to convert dynamic tool {:?}: {e:?}", tool.name);
            }
        }
    }

    builder
}
```

---

## 8.4 exec_policy.rs 深度解析

### 策略匹配算法

```rust
// exec_policy.rs:122-189
pub(crate) async fn create_exec_approval_requirement_for_command(
    &self,
    req: ExecApprovalRequest<'_>,
) -> ExecApprovalRequirement {
    let ExecApprovalRequest {
        features,
        command,
        approval_policy,
        sandbox_policy,
        sandbox_permissions,
        prefix_rule,
    } = req;

    // 1. 解析复合命令
    // "bash -lc 'git status && npm install'" → ["git status", "npm install"]
    let commands = parse_shell_lc_plain_commands(command)
        .unwrap_or_else(|| vec![command.to_vec()]);

    // 2. 获取当前策略
    let exec_policy = self.policy.load();

    // 3. 构建回退策略
    let exec_policy_fallback = ExecPolicyFallback {
        approval_policy,
        sandbox_policy,
        sandbox_permissions: &sandbox_permissions,
        prefix_rule: prefix_rule.as_deref(),
    };

    // 4. 批量检查所有子命令
    let evaluation = exec_policy.check_multiple(
        commands.iter().map(|c| c.as_slice()),
        &exec_policy_fallback,
    );

    // 5. 生成决策
    let reason = evaluation.matched_rules
        .iter()
        .map(|m| m.reason())
        .collect::<Vec<_>>()
        .join("; ");

    match evaluation.decision {
        Decision::Forbidden => {
            ExecApprovalRequirement::Forbidden { reason }
        }

        Decision::Prompt => {
            // 检查冲突：策略要求审批，但用户禁用审批
            if matches!(approval_policy, AskForApproval::Never) {
                return ExecApprovalRequirement::Forbidden {
                    reason: "Command requires approval but approval is disabled".to_string(),
                };
            }

            // 生成自动化规则提议
            let proposed_amendment = generate_amendment_proposal(&commands, &evaluation);

            ExecApprovalRequirement::NeedsApproval {
                reason,
                proposed_execpolicy_amendment: proposed_amendment,
            }
        }

        Decision::Allow => {
            // 检查是否可以绕过沙箱
            let bypass_sandbox = evaluation.matched_rules.iter().any(|rule_match| {
                is_policy_match(rule_match) && rule_match.decision() == Decision::Allow
            });

            ExecApprovalRequirement::Skip { bypass_sandbox }
        }
    }
}
```

---

## 8.5 seatbelt.rs 深度解析

### 策略生成核心

```rust
// seatbelt.rs:46-135
pub(crate) fn create_seatbelt_command_args(
    command: Vec<String>,
    sandbox_policy: &SandboxPolicy,
    sandbox_policy_cwd: &Path,
) -> Vec<String> {
    // ========== 写权限策略 ==========
    let (file_write_policy, file_write_dir_params) = {
        if sandbox_policy.has_full_disk_write_access() {
            // 完全写权限
            (r#"(allow file-write* (regex #"^/"))"#.to_string(), Vec::new())
        } else {
            let writable_roots = sandbox_policy.get_writable_roots_with_cwd(sandbox_policy_cwd);
            let mut policies: Vec<String> = Vec::new();
            let mut params: Vec<(String, PathBuf)> = Vec::new();

            for (index, wr) in writable_roots.iter().enumerate() {
                // 路径规范化（避免符号链接绕过）
                let canonical_root = wr.root.as_path().canonicalize()
                    .unwrap_or_else(|_| wr.root.to_path_buf());
                let root_param = format!("WRITABLE_ROOT_{index}");
                params.push((root_param.clone(), canonical_root));

                if wr.read_only_subpaths.is_empty() {
                    // 简单情况：整个目录可写
                    policies.push(format!("(subpath (param \"{root_param}\"))"));
                } else {
                    // 复杂情况：排除 .git 和 .codex
                    let mut require_parts = vec![
                        format!("(subpath (param \"{root_param}\"))")
                    ];

                    for (subpath_index, ro) in wr.read_only_subpaths.iter().enumerate() {
                        let canonical_ro = ro.as_path().canonicalize()
                            .unwrap_or_else(|_| ro.to_path_buf());
                        let ro_param = format!("WRITABLE_ROOT_{index}_RO_{subpath_index}");
                        require_parts.push(format!("(require-not (subpath (param \"{ro_param}\")))"));
                        params.push((ro_param, canonical_ro));
                    }

                    // 生成 (require-all ...) 表达式
                    policies.push(format!("(require-all {} )", require_parts.join(" ")));
                }
            }

            if policies.is_empty() {
                (String::new(), Vec::new())
            } else {
                (format!("(allow file-write*\n{})", policies.join("\n")), params)
            }
        }
    };

    // ========== 组装完整策略 ==========
    let full_policy = format!(
        "{MACOS_SEATBELT_BASE_POLICY}\n\
         ; allow read-only file operations\n\
         (allow file-read*)\n\
         {file_write_policy}\n\
         {network_policy}"
    );

    // ========== 构建命令行 ==========
    let mut args = vec!["-p".to_string(), full_policy];

    // 添加参数定义
    for (key, value) in file_write_dir_params {
        args.push(format!("-D{key}={}", value.to_string_lossy()));
    }

    args.push("--".to_string());
    args.extend(command);

    args
}
```

---

## 8.6 设计决策 Trade-off 分析

### 1. 单文件 vs 模块化

**现状**：`codex.rs` 达到 6094 行

**Trade-off**：
- **优点**：所有核心逻辑在一个文件，便于理解整体流程
- **缺点**：违反单一职责原则，难以测试和维护

**建议重构**：
```
codex/
├── mod.rs           # 公共 API
├── session.rs       # Session 结构和方法
├── turn.rs          # Turn 执行逻辑
├── sampling.rs      # 采样请求处理
├── handlers.rs      # Op 处理器
└── helpers.rs       # 辅助函数
```

### 2. 字节估算 vs 精确 Tokenization

**现状**：使用 `4 bytes ≈ 1 token` 估算

**Trade-off**：
- **优点**：零依赖、高性能
- **缺点**：估算误差可能导致 Token 超限

**实际影响**：误差在 10-20% 范围内，通过预留 buffer 缓解

### 3. 原生沙箱 vs Docker

**现状**：每平台独立实现原生沙箱

**Trade-off**：
- **优点**：零依赖、低开销、深度集成
- **缺点**：维护成本高、平台差异大

**设计哲学**：追求"零依赖"的本地体验

### 4. 队列对模式 vs 请求-响应

**现状**：使用 Submission/Event 队列对

**Trade-off**：
- **优点**：支持长时间运行、流式输出、中断
- **缺点**：复杂度高、调试困难

**设计哲学**：Agent 的异步本质决定了队列对模式是必要的

---

## 章节衔接

**本章回顾**：
- 我们深入分析了核心源码的实现细节
- 关键收获：状态机设计 + 并发模型 + 策略生成

**下一章预告**：
- 在 `09-interview-qa.md` 中，我们将整理面试 QA
- 为什么需要学习：面试是检验学习成果的最佳方式
- 关键问题：如何回答架构设计问题？如何展示源码理解？
