# 第三章：工具系统

## 3.1 工具系统架构

Codex 的工具系统是一个精心设计的分层架构，实现了工具的注册、路由、审批、沙箱执行的完整生命周期。

```
┌─────────────────────────────────────────────────────────────┐
│                      Tool System                             │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │  ToolSpec   │  │ToolRegistry │  │ ToolRouter  │  Core    │
│  │  (规范定义)  │  │  (注册表)   │  │  (路由分发)  │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                │                │                  │
│         └────────────────┼────────────────┘                  │
│                          ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  ToolOrchestrator                      │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │  │
│  │  │   审批决策   │→│   沙箱选择   │→│   执行重试   │    │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │  │
│  └───────────────────────────────────────────────────────┘  │
│                          │                                   │
│         ┌────────────────┼────────────────┐                  │
│         ▼                ▼                ▼                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   shell     │  │  read_file  │  │ apply_patch │ Handlers │
│  │   handler   │  │   handler   │  │   handler   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

---

## 3.2 内置工具清单

Codex 提供 16 个内置工具，覆盖文件操作、命令执行、MCP 集成等场景：

| 工具名 | 文件 | 功能 | 并行支持 |
|--------|------|------|----------|
| **shell** | `handlers/shell.rs` | Bash/PowerShell 执行 | 否 |
| **unified_exec** | `handlers/unified_exec.rs` | 统一执行接口 | 否 |
| **read_file** | `handlers/read_file.rs` | 文件读取（支持图片） | 是 |
| **apply_patch** | `handlers/apply_patch.rs` | 代码补丁应用 | 否 |
| **grep_files** | `handlers/grep_files.rs` | 文件内容搜索 | 是 |
| **list_dir** | `handlers/list_dir.rs` | 目录列表 | 是 |
| **view_image** | `handlers/view_image.rs` | 图片查看 | 是 |
| **mcp** | `handlers/mcp.rs` | MCP 工具调用 | 是 |
| **mcp_resource** | `handlers/mcp_resource.rs` | MCP 资源访问 | 是 |
| **collab** | `handlers/collab.rs` | Multi-Agent 协作 | 否 |
| **plan** | `handlers/plan.rs` | 计划生成/更新 | 否 |
| **request_user_input** | `handlers/request_user_input.rs` | 用户输入请求 | 否 |
| **dynamic** | `handlers/dynamic.rs` | 动态工具 | 视情况 |

---

## 3.3 工具规范定义

### ToolSpec 结构

工具规范定义了工具的元信息，用于生成 LLM 的 function calling schema：

```rust
// spec.rs
pub enum ToolSpec {
    Function(ResponsesApiTool),
    Custom(CustomToolSpec),
}

pub struct ResponsesApiTool {
    pub name: String,
    pub description: String,
    pub strict: bool,
    pub parameters: JsonSchema,
}
```

### 动态构建工具规范

工具规范不是静态定义的，而是根据配置动态构建：

```rust
// spec.rs:1255-1447
pub(crate) fn build_specs(
    config: &ToolsConfig,
    mcp_tools: Option<HashMap<String, mcp_types::Tool>>,
    dynamic_tools: &[DynamicToolSpec],
) -> ToolRegistryBuilder {
    let mut builder = ToolRegistryBuilder::new();

    // 1. Shell 工具：根据平台能力自动选择
    match &config.shell_type {
        ConfigShellToolType::UnifiedExec => {
            builder.push_spec(create_exec_command_tool(config.request_rule_enabled));
            builder.push_spec(create_write_stdin_tool());
        }
        ConfigShellToolType::ShellCommand => {
            builder.push_spec(create_shell_command_tool(config.request_rule_enabled));
        }
        // ...
    }

    // 2. 始终注册别名以保持向后兼容
    if config.shell_type != ConfigShellToolType::Disabled {
        builder.register_handler("shell", shell_handler.clone());
        builder.register_handler("container.exec", shell_handler.clone());
        builder.register_handler("local_shell", shell_handler);
    }

    // 3. MCP 工具：动态转换为 OpenAI 格式
    if let Some(mcp_tools) = mcp_tools {
        for (name, tool) in mcp_tools {
            match mcp_tool_to_openai_tool(name.clone(), tool) {
                Ok(converted_tool) => {
                    builder.push_spec(ToolSpec::Function(converted_tool));
                    builder.register_handler(name, mcp_handler.clone());
                }
                Err(e) => tracing::error!("Failed to convert {name:?}: {e:?}"),
            }
        }
    }

    builder
}
```

**设计亮点**：
1. **配置驱动**：通过 `ToolsConfig` 控制工具启用/禁用
2. **向后兼容**：旧 prompt 使用 "shell"，新 prompt 使用 "exec_command"
3. **错误容忍**：MCP 工具转换失败不影响其他工具

---

## 3.4 工具路由机制

### ToolRouter 结构

`ToolRouter` 负责将 LLM 的工具调用分发到对应的 handler：

```rust
// router.rs
pub struct ToolRouter {
    registry: ToolRegistry,
}

impl ToolRouter {
    pub async fn dispatch_tool_call(
        &self,
        session: Arc<Session>,
        turn: Arc<TurnContext>,
        tracker: SharedTurnDiffTracker,
        call: ToolCall,
    ) -> Result<ResponseInputItem, FunctionCallError>;
}
```

### 工具调用解析

LLM 返回的工具调用有多种格式，需要统一解析：

```rust
// router.rs:61-130
pub async fn build_tool_call(
    session: &Session,
    item: ResponseItem,
) -> Result<Option<ToolCall>, FunctionCallError> {
    match item {
        // 标准函数调用
        ResponseItem::FunctionCall { name, arguments, call_id, .. } => {
            if let Some((server, tool)) = session.parse_mcp_tool_name(&name).await {
                // MCP 工具
                Ok(Some(ToolCall {
                    tool_name: name,
                    call_id,
                    payload: ToolPayload::Mcp { server, tool, raw_arguments: arguments },
                }))
            } else {
                // 普通函数
                Ok(Some(ToolCall {
                    tool_name: name,
                    call_id,
                    payload: ToolPayload::Function { arguments },
                }))
            }
        }

        // 自定义工具调用（apply_patch 使用）
        ResponseItem::CustomToolCall { name, input, call_id, .. } => {
            Ok(Some(ToolCall {
                tool_name: name,
                call_id,
                payload: ToolPayload::Custom { input },
            }))
        }

        // 本地 shell 调用（旧协议兼容）
        ResponseItem::LocalShellCall { id, call_id, action, .. } => {
            // 转换为统一格式
            Ok(Some(ToolCall {
                tool_name: "local_shell".to_string(),
                call_id: call_id.or(id).ok_or(FunctionCallError::MissingLocalShellCallId)?,
                payload: ToolPayload::LocalShell { params },
            }))
        }

        _ => Ok(None),
    }
}
```

**设计模式**：Adapter 模式。将异构的输入格式（FunctionCall、CustomToolCall、LocalShellCall、MCP）统一转换为 `ToolCall` 结构。

---

## 3.5 工具注册表

### ToolRegistry 结构

`ToolRegistry` 管理工具名称到 handler 的映射：

```rust
// registry.rs
pub struct ToolRegistry {
    handlers: HashMap<String, Arc<dyn ToolHandler>>,
    specs: Vec<ToolSpec>,
    parallel_support: HashMap<String, bool>,
}

#[async_trait]
pub trait ToolHandler: Send + Sync {
    fn kind(&self) -> ToolKind;
    async fn is_mutating(&self, invocation: &ToolInvocation) -> bool;
    async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError>;
}
```

### 工具分发逻辑

```rust
// registry.rs:67-149
pub async fn dispatch(
    &self,
    invocation: ToolInvocation,
) -> Result<ResponseInputItem, FunctionCallError> {
    let tool_name = invocation.tool_name.clone();

    // 1. 查找 handler
    let handler = match self.handler(tool_name.as_ref()) {
        Some(handler) => handler,
        None => {
            let message = unsupported_tool_call_message(&invocation.payload, tool_name.as_ref());
            return Err(FunctionCallError::RespondToModel(message));
        }
    };

    // 2. 验证 payload 类型匹配
    if !handler.matches_kind(&invocation.payload) {
        return Err(FunctionCallError::Fatal("incompatible payload".to_string()));
    }

    // 3. 等待工具门（mutating 工具串行执行）
    if handler.is_mutating(&invocation).await {
        invocation.turn.tool_call_gate.wait_ready().await;
    }

    // 4. 执行工具
    let output = handler.handle(invocation).await?;

    // 5. 转换输出格式
    Ok(output.into_response(&call_id, &payload))
}
```

**设计亮点**：
1. **遥测集成**：自动记录工具执行时长、成功率
2. **串行保护**：mutating 工具通过 `tool_call_gate` 串行化
3. **错误分类**：`Fatal` 错误中止会话，`RespondToModel` 错误返回给 LLM

---

## 3.6 审批与沙箱编排

### ToolOrchestrator 结构

`ToolOrchestrator` 是审批和沙箱的编排中心：

```rust
// orchestrator.rs
pub struct ToolOrchestrator {
    sandbox: SandboxManager,
}

impl ToolOrchestrator {
    pub async fn run<Rq, Out, T>(
        &mut self,
        tool: &mut T,
        req: &Rq,
        tool_ctx: &ToolCtx<'_>,
        turn_ctx: &TurnContext,
        approval_policy: AskForApproval,
    ) -> Result<Out, ToolError>
    where
        T: ToolRuntime<Rq, Out>;
}
```

### 三阶段执行流程

```rust
// orchestrator.rs:35-165
pub async fn run(...) -> Result<Out, ToolError> {
    // ========== 阶段 1：审批决策 ==========
    let requirement = tool.exec_approval_requirement(req).unwrap_or_else(|| {
        default_exec_approval_requirement(approval_policy, &turn_ctx.sandbox_policy)
    });

    match requirement {
        ExecApprovalRequirement::Skip { .. } => {
            // 无需审批，直接执行
        }
        ExecApprovalRequirement::Forbidden { reason } => {
            return Err(ToolError::Rejected(reason));
        }
        ExecApprovalRequirement::NeedsApproval { reason, .. } => {
            // 请求用户审批
            let decision = tool.start_approval_async(req, approval_ctx).await;
            match decision {
                ReviewDecision::Denied | ReviewDecision::Abort => {
                    return Err(ToolError::Rejected("rejected by user".to_string()));
                }
                _ => {}
            }
            already_approved = true;
        }
    }

    // ========== 阶段 2：首次尝试（可能跳过沙箱）==========
    let initial_sandbox = match tool.sandbox_mode_for_first_attempt(req) {
        SandboxOverride::BypassSandboxFirstAttempt => SandboxType::None,
        SandboxOverride::NoOverride => self.sandbox.select_initial(...),
    };

    match tool.run(req, &initial_attempt, tool_ctx).await {
        Ok(out) => Ok(out),

        // ========== 阶段 3：沙箱拒绝后的升级流程 ==========
        Err(ToolError::Codex(CodexErr::Sandbox(SandboxErr::Denied { output }))) => {
            // 3.1 检查工具是否允许升级
            if !tool.escalate_on_failure() {
                return Err(...);
            }

            // 3.2 检查策略是否允许无沙箱审批
            if !tool.wants_no_sandbox_approval(approval_policy) {
                return Err(...);
            }

            // 3.3 询问用户是否无沙箱重试
            if !tool.should_bypass_approval(approval_policy, already_approved) {
                let decision = tool.start_approval_async(req, approval_ctx).await;
                match decision {
                    ReviewDecision::Denied | ReviewDecision::Abort => {
                        return Err(ToolError::Rejected("rejected by user".to_string()));
                    }
                    _ => {}
                }
            }

            // 3.4 无沙箱重试
            tool.run(req, &escalated_attempt, tool_ctx).await
        }

        other => other,
    }
}
```

**设计哲学**：
1. **分层决策**：审批策略 → 沙箱选择 → 失败升级
2. **防御性编程**：多重检查避免未授权的无沙箱执行
3. **用户体验**：失败原因作为 `retry_reason` 传递给审批界面

---

## 3.7 审批缓存机制

### ApprovalStore 结构

审批结果会被缓存，避免重复询问用户：

```rust
// sandboxing.rs:27-50
#[derive(Clone, Default, Debug)]
pub(crate) struct ApprovalStore {
    map: HashMap<String, ReviewDecision>,  // 序列化键 → 决策
}

impl ApprovalStore {
    pub fn get<K>(&self, key: &K) -> Option<ReviewDecision>
    where
        K: Serialize,
    {
        let s = serde_json::to_string(key).ok()?;
        self.map.get(&s).cloned()
    }

    pub fn put<K>(&mut self, key: K, value: ReviewDecision)
    where
        K: Serialize,
    {
        if let Ok(s) = serde_json::to_string(&key) {
            self.map.insert(s, value);
        }
    }
}
```

### 批量审批语义

```rust
// sandboxing.rs:58-104
pub(crate) async fn with_cached_approval<K, F, Fut>(
    services: &SessionServices,
    tool_name: &str,
    keys: Vec<K>,  // 支持多个审批键（apply_patch 修改多文件）
    fetch: F,
) -> ReviewDecision {
    // 1. 检查所有键是否已批准
    let already_approved = {
        let store = services.tool_approvals.lock().await;
        keys.iter().all(|key| {
            matches!(store.get(key), Some(ReviewDecision::ApprovedForSession))
        })
    };

    if already_approved {
        return ReviewDecision::ApprovedForSession;
    }

    // 2. 请求用户审批
    let decision = fetch().await;

    // 3. 缓存会话级审批
    if matches!(decision, ReviewDecision::ApprovedForSession) {
        let mut store = services.tool_approvals.lock().await;
        for key in keys {
            store.put(key, ReviewDecision::ApprovedForSession);
        }
    }

    decision
}
```

**设计亮点**：
1. **批量语义**：`apply_patch` 修改多文件时，所有文件都已批准才跳过
2. **细粒度缓存**：按键单独存储，子集请求可复用
3. **序列化键**：通过 `serde_json` 序列化任意类型作为缓存键

---

## 3.8 并行执行控制

### 读写锁模式

```rust
// parallel.rs:79-90
let _guard = if supports_parallel {
    Either::Left(lock.read().await)   // 并行工具：读锁
} else {
    Either::Right(lock.write().await) // 串行工具：写锁
};

router.dispatch_tool_call(session, turn, tracker, call.clone()).await
```

### 并行工具标记

```rust
// spec.rs:1319-1324
builder.push_spec_with_parallel_support(create_list_mcp_resources_tool(), true);
builder.push_spec_with_parallel_support(create_read_file_tool(), true);
builder.push_spec_with_parallel_support(create_grep_files_tool(), true);
// shell、apply_patch 等默认不支持并行
```

**设计哲学**：
- **零成本抽象**：`Either<RwLockReadGuard, RwLockWriteGuard>` 避免 `Box<dyn Guard>`
- **自动互斥**：read_file 并行，shell 串行，无需手动协调
- **取消语义**：`tokio::select!` 监听 `CancellationToken`，用户中止立即返回

---

## 3.9 Shell 工具实现

Shell 是最复杂的工具，需要处理命令解析、沙箱执行、流式输出：

```rust
// handlers/shell.rs:141-174
async fn run(
    &mut self,
    req: &ShellRequest,
    attempt: &SandboxAttempt<'_>,
    ctx: &ToolCtx<'_>,
) -> Result<ExecToolCallOutput, ToolError> {
    // 1. 构建命令规范
    let command_spec = CommandSpec {
        program: req.shell.clone(),
        args: vec!["-c".to_string(), req.command.clone()],
        cwd: req.workdir.clone().unwrap_or_else(|| ctx.cwd.clone()),
        env: ctx.env.clone(),
        expiration: req.timeout.map(ExecExpiration::Timeout).unwrap_or_default(),
        sandbox_permissions: req.sandbox_permissions.clone(),
        justification: req.justification.clone(),
    };

    // 2. 转换为执行环境
    let exec_env = attempt.manager.transform(
        command_spec,
        attempt.policy,
        attempt.sandbox,
        attempt.sandbox_cwd,
        attempt.codex_linux_sandbox_exe,
        attempt.windows_sandbox_level,
    )?;

    // 3. 执行命令
    let output = execute_env(exec_env, ctx.stdout_stream.clone()).await?;

    // 4. 构建输出
    Ok(ExecToolCallOutput {
        stdout: output.stdout,
        stderr: output.stderr,
        exit_code: output.exit_code,
    })
}
```

---

## 3.10 Apply Patch 工具实现

`apply_patch` 是代码修改的核心工具，使用 unified diff 格式：

```rust
// handlers/apply_patch.rs（概念模型）
async fn handle(&self, invocation: ToolInvocation) -> Result<ToolOutput, FunctionCallError> {
    let patch = parse_unified_diff(&invocation.arguments)?;

    // 1. 验证所有文件存在
    for file in &patch.files {
        if !file.path.exists() && !file.is_new {
            return Err(FunctionCallError::RespondToModel(
                format!("File not found: {}", file.path.display())
            ));
        }
    }

    // 2. 请求审批（批量）
    let keys: Vec<_> = patch.files.iter().map(|f| f.path.clone()).collect();
    let decision = with_cached_approval(&services, "apply_patch", keys, || async {
        request_approval(&patch).await
    }).await;

    if decision != ReviewDecision::ApprovedForSession {
        return Err(FunctionCallError::RespondToModel("Patch rejected".to_string()));
    }

    // 3. 应用补丁
    for file in &patch.files {
        apply_file_patch(file)?;
    }

    Ok(ToolOutput::success("Patch applied successfully"))
}
```

---

## 章节衔接

**本章回顾**：
- 我们深入分析了工具系统的设计
- 关键收获：分层架构 + 审批编排 + 并行控制 + 缓存机制

**下一章预告**：
- 在 `04-context-management.md` 中，我们将深入上下文管理
- 为什么需要学习：上下文是 Agent 的"记忆"，管理不当会导致 Token 溢出或信息丢失
- 关键问题：如何估算 Token？如何截断长输出？如何自动压缩历史？
