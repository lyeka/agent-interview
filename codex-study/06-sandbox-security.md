# 第六章：沙箱安全机制

## 6.1 安全架构概览

Codex 实现了**跨平台多层沙箱防御体系**，核心设计哲学是"纵深防御 + 最小权限原则"。

```
┌─────────────────────────────────────────────────────────────┐
│                    Security Architecture                     │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Exec Policy (命令白名单/黑名单)                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  .rules 文件定义允许/禁止的命令模式                   │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Approval Flow (用户审批)                           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  危险操作需用户确认，审批结果可缓存                   │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Platform Sandbox (操作系统级隔离)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   macOS     │  │   Linux     │  │  Windows    │          │
│  │  Seatbelt   │  │  Landlock   │  │ AppContainer│          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

---

## 6.2 执行策略引擎

### ExecPolicyManager 结构

```rust
// exec_policy.rs:86-88
pub(crate) struct ExecPolicyManager {
    policy: ArcSwap<Policy>,  // 原子指针交换，零停机更新规则
}
```

### 决策流程

```rust
// exec_policy.rs:122-189
pub(crate) async fn create_exec_approval_requirement_for_command(
    &self,
    req: ExecApprovalRequest<'_>,
) -> ExecApprovalRequirement {
    // 1. 解析 shell 命令（支持 bash -lc "cmd1 && cmd2"）
    let commands = parse_shell_lc_plain_commands(command)
        .unwrap_or_else(|| vec![command.to_vec()]);

    // 2. 策略匹配（前缀规则 + 启发式规则）
    let evaluation = exec_policy.check_multiple(commands.iter(), &exec_policy_fallback);

    // 3. 决策分支
    match evaluation.decision {
        Decision::Forbidden => ExecApprovalRequirement::Forbidden { reason },

        Decision::Prompt => {
            if matches!(approval_policy, AskForApproval::Never) {
                // 冲突：策略要求审批，但用户禁用审批 → 拒绝
                ExecApprovalRequirement::Forbidden { reason: PROMPT_CONFLICT_REASON }
            } else {
                ExecApprovalRequirement::NeedsApproval {
                    reason,
                    proposed_execpolicy_amendment  // 提议自动化规则
                }
            }
        }

        Decision::Allow => ExecApprovalRequirement::Skip {
            bypass_sandbox: evaluation.matched_rules.iter().any(|rule_match| {
                is_policy_match(rule_match) && rule_match.decision() == Decision::Allow
            }),
        }
    }
}
```

### 策略文件格式

```
# ~/.codex/rules/default.rules

# 允许的命令
allow git *
allow npm install
allow cargo build

# 需要审批的命令
prompt rm -rf *
prompt sudo *

# 禁止的命令
forbid curl * | bash
forbid wget * | sh
```

### 热更新机制

```rust
// exec_policy.rs:191-214
pub async fn append_rule_and_reload(&self, rule: &str, config_stack: &ConfigLayerStack) {
    // 1. 追加规则到文件
    let rules_file = config_stack.get_user_rules_file();
    fs::append(&rules_file, format!("\n{}\n", rule)).await?;

    // 2. 重新加载策略
    let new_policy = load_exec_policy(config_stack).await?;

    // 3. 原子替换（无锁）
    self.policy.store(Arc::new(new_policy));
}
```

**设计亮点**：用户审批后立即追加规则到文件并更新内存策略，实现"学习"效果。

---

## 6.3 macOS Seatbelt 沙箱

### 技术背景

Seatbelt 是 macOS 的 TrustedBSD MAC（Mandatory Access Control）框架，通过 Scheme 语言描述策略。

### 策略生成

```rust
// seatbelt.rs:46-135
pub(crate) fn create_seatbelt_command_args(
    command: Vec<String>,
    sandbox_policy: &SandboxPolicy,
    sandbox_policy_cwd: &Path,
) -> Vec<String> {
    // 1. 写权限策略
    let file_write_policy = if sandbox_policy.has_full_disk_write_access() {
        // 完全写权限
        r#"(allow file-write* (regex #"^/"))"#.to_string()
    } else {
        // 限制写权限到特定目录
        let writable_roots = sandbox_policy.get_writable_roots_with_cwd(sandbox_policy_cwd);
        build_write_policy(&writable_roots)
    };

    // 2. 读权限策略
    let file_read_policy = if sandbox_policy.has_full_disk_read_access() {
        "; allow read-only file operations\n(allow file-read*)"
    } else {
        ""
    };

    // 3. 网络策略
    let network_policy = if sandbox_policy.has_full_network_access() {
        MACOS_SEATBELT_NETWORK_POLICY
    } else {
        ""
    };

    // 4. 组装完整策略
    let full_policy = format!(
        "{MACOS_SEATBELT_BASE_POLICY}\n{file_read_policy}\n{file_write_policy}\n{network_policy}"
    );

    // 5. 构建 sandbox-exec 命令行
    vec!["-p".to_string(), full_policy, "--".to_string()]
        .into_iter()
        .chain(command)
        .collect()
}
```

### 基础策略

```scheme
; seatbelt_base_policy.sbpl
(version 1)

; 默认拒绝所有操作
(deny default)

; 允许进程管理
(allow process-exec)
(allow process-fork)
(allow signal (target same-sandbox))

; 允许写入 /dev/null
(allow file-write-data
  (require-all
    (path "/dev/null")
    (vnode-type CHARACTER-DEVICE)))

; 允许读取特定 sysctl
(allow sysctl-read
  (sysctl-name "hw.activecpu")
  (sysctl-name "hw.memsize")
  (sysctl-name "kern.osversion")
  ; ... 约 50 个白名单项
)

; 允许 PTY 操作
(allow pseudo-tty)
(allow file-read* file-write* file-ioctl (literal "/dev/ptmx"))
```

### 保护 .git 和 .codex

```rust
// seatbelt.rs（概念模型）
fn build_write_policy(writable_roots: &[WritableRoot]) -> String {
    let mut policies = Vec::new();

    for wr in writable_roots {
        if wr.read_only_subpaths.is_empty() {
            // 简单情况：整个目录可写
            policies.push(format!("(subpath (param \"WRITABLE_ROOT\"))"));
        } else {
            // 复杂情况：排除 .git 和 .codex
            let mut parts = vec![format!("(subpath (param \"WRITABLE_ROOT\"))")];
            for ro in &wr.read_only_subpaths {
                parts.push(format!("(require-not (subpath (param \"RO_PATH\")))"));
            }
            policies.push(format!("(require-all {})", parts.join(" ")));
        }
    }

    format!("(allow file-write*\n{})", policies.join("\n"))
}
```

**安全设计**：
1. **防止权限提升**：禁止写入 `.git/hooks/pre-commit`
2. **防止配置篡改**：禁止写入 `.codex/config.toml`
3. **路径规范化**：使用 `canonicalize()` 避免符号链接绕过

---

## 6.4 Linux Landlock 沙箱

### 技术背景

Landlock 是 Linux 5.13+ 的 LSM（Linux Security Module），提供非特权进程的文件系统隔离。

### 实现方式

Codex 使用外部辅助进程 `codex-linux-sandbox` 实现沙箱：

```rust
// landlock.rs
pub async fn spawn_command_under_linux_sandbox<P>(
    codex_linux_sandbox_exe: P,
    command: Vec<String>,
    command_cwd: PathBuf,
    sandbox_policy: &SandboxPolicy,
    sandbox_policy_cwd: &Path,
    stdio_policy: StdioPolicy,
    env: HashMap<String, String>,
) -> std::io::Result<Child> {
    // 将策略序列化为 JSON 传递给辅助进程
    let sandbox_policy_json = serde_json::to_string(sandbox_policy)?;

    let args = vec![
        "--sandbox-policy-cwd".to_string(),
        sandbox_policy_cwd.to_string_lossy().to_string(),
        "--sandbox-policy".to_string(),
        sandbox_policy_json,
        "--".to_string(),
    ];

    spawn_child_async(
        codex_linux_sandbox_exe.as_ref().to_path_buf(),
        args.into_iter().chain(command).collect(),
        Some("codex-linux-sandbox"),
        command_cwd,
        sandbox_policy,
        stdio_policy,
        env,
    ).await
}
```

**设计特点**：
1. **进程隔离**：沙箱逻辑在独立进程中，避免污染主进程
2. **JSON 协议**：策略通过 JSON 传递，易于调试和扩展
3. **arg0 伪装**：设置 `arg0 = "codex-linux-sandbox"` 让进程列表显示友好名称

---

## 6.5 Windows 沙箱

### 两级实现

```rust
pub enum WindowsSandboxLevel {
    Disabled,
    RestrictedToken,  // 遗留架构：受限令牌
    Elevated,         // 新架构：AppContainer
}

impl WindowsSandboxLevelExt for WindowsSandboxLevel {
    fn from_features(features: &Features) -> WindowsSandboxLevel {
        if features.enabled(Feature::WindowsSandboxElevated) {
            return WindowsSandboxLevel::Elevated;
        }
        if features.enabled(Feature::WindowsSandbox) {
            WindowsSandboxLevel::RestrictedToken
        } else {
            WindowsSandboxLevel::Disabled
        }
    }
}
```

### Windows 特殊性

```rust
// exec_policy.rs:308-311
// Windows 的 ReadOnly 沙箱不是真沙箱，因此危险命令必须审批
if cfg!(target_os = "windows") && matches!(sandbox_policy, SandboxPolicy::ReadOnly) {
    // 强制审批
}
```

---

## 6.6 沙箱管理器

### SandboxManager 结构

```rust
// sandboxing/mod.rs
pub struct SandboxManager {
    // 平台检测结果缓存
}

impl SandboxManager {
    pub fn select_initial(
        &self,
        policy: &SandboxPolicy,
        pref: SandboxablePreference,
        windows_sandbox_level: WindowsSandboxLevel,
    ) -> SandboxType {
        match pref {
            SandboxablePreference::Forbid => SandboxType::None,
            SandboxablePreference::Require => {
                get_platform_sandbox(windows_sandbox_level != WindowsSandboxLevel::Disabled)
                    .unwrap_or(SandboxType::None)
            }
            SandboxablePreference::Auto => match policy {
                SandboxPolicy::DangerFullAccess | SandboxPolicy::ExternalSandbox { .. } => {
                    SandboxType::None
                }
                _ => get_platform_sandbox(windows_sandbox_level != WindowsSandboxLevel::Disabled)
                    .unwrap_or(SandboxType::None),
            },
        }
    }
}
```

### 命令转换

```rust
pub fn transform(
    &self,
    spec: CommandSpec,
    policy: &SandboxPolicy,
    sandbox: SandboxType,
    sandbox_policy_cwd: &Path,
    codex_linux_sandbox_exe: Option<&PathBuf>,
    windows_sandbox_level: WindowsSandboxLevel,
) -> Result<ExecEnv, SandboxTransformError> {
    let (command, sandbox_env, arg0_override) = match sandbox {
        SandboxType::None => (command, HashMap::new(), None),

        #[cfg(target_os = "macos")]
        SandboxType::MacosSeatbelt => {
            let args = create_seatbelt_command_args(command, policy, sandbox_policy_cwd);
            let full_command = vec![MACOS_PATH_TO_SEATBELT_EXECUTABLE.to_string()]
                .into_iter()
                .chain(args)
                .collect();
            (full_command, HashMap::new(), None)
        }

        SandboxType::LinuxSeccomp => {
            let exe = codex_linux_sandbox_exe
                .ok_or(SandboxTransformError::MissingLinuxSandboxExecutable)?;
            let args = create_linux_sandbox_command_args(command, policy, sandbox_policy_cwd);
            let full_command = vec![exe.to_string_lossy().to_string()]
                .into_iter()
                .chain(args)
                .collect();
            (full_command, HashMap::new(), Some("codex-linux-sandbox".to_string()))
        }

        // ...
    };

    Ok(ExecEnv { command, sandbox, /* ... */ })
}
```

---

## 6.7 审批流程

### 审批决策枚举

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum ReviewDecision {
    Approved,              // 单次批准
    ApprovedForSession,    // 会话级批准（缓存）
    Denied,                // 拒绝
    Abort,                 // 中止整个操作
}
```

### 审批策略枚举

```rust
pub enum AskForApproval {
    Never,          // 从不请求审批
    OnFailure,      // 失败后请求
    OnRequest,      // 按需请求
    UnlessTrusted,  // 非信任命令请求
}
```

### 审批缓存

```rust
// sandboxing.rs:58-104
pub(crate) async fn with_cached_approval<K, F, Fut>(
    services: &SessionServices,
    tool_name: &str,
    keys: Vec<K>,
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

---

## 6.8 安全测试

### 测试用例示例

```rust
#[test]
fn create_seatbelt_args_with_read_only_git_and_codex_subpaths() {
    // 创建测试环境
    let tmp = tempdir().unwrap();
    let vulnerable_root = tmp.path().join("project");
    fs::create_dir_all(&vulnerable_root.join(".git")).unwrap();
    fs::create_dir_all(&vulnerable_root.join(".codex")).unwrap();
    fs::write(
        vulnerable_root.join(".codex/config.toml"),
        "sandbox_mode = \"read-only\"\n"
    ).unwrap();

    // 构建策略
    let policy = SandboxPolicy::WorkspaceWrite {
        writable_roots: vec![vulnerable_root.clone().try_into().unwrap()],
        network_access: false,
        exclude_tmpdir_env_var: true,
        exclude_slash_tmp: true,
    };

    // 尝试写入 .codex/config.toml
    let shell_command = vec![
        "bash", "-c",
        "echo 'sandbox_mode = \"danger-full-access\"' > .codex/config.toml",
    ];

    let args = create_seatbelt_command_args(shell_command, &policy, &vulnerable_root);

    // 执行并验证失败
    let output = Command::new("/usr/bin/sandbox-exec")
        .args(&args)
        .current_dir(&vulnerable_root)
        .output()
        .unwrap();

    assert!(!output.status.success(), "写入 .codex/config.toml 应失败");

    // 验证原文件未被修改
    let content = fs::read_to_string(vulnerable_root.join(".codex/config.toml")).unwrap();
    assert_eq!(content, "sandbox_mode = \"read-only\"\n");
}
```

---

## 6.9 与 Claude Code 的对比

| 维度 | Codex | Claude Code |
|------|-------|-------------|
| **沙箱技术** | 原生（Seatbelt/Landlock/AppContainer） | Docker |
| **依赖** | 无外部依赖 | 需要 Docker |
| **性能** | 低开销 | 容器启动开销 |
| **隔离级别** | 文件系统 + 网络 | 完整容器隔离 |
| **跨平台** | 每平台独立实现 | Docker 统一 |

**设计权衡**：Codex 选择原生沙箱追求"零依赖"体验，Claude Code 选择 Docker 追求"一致性"保证。

---

## 章节衔接

**本章回顾**：
- 我们深入分析了沙箱安全机制
- 关键收获：三层防御 + 跨平台实现 + 审批缓存

**下一章预告**：
- 在 `07-mcp-integration.md` 中，我们将深入 MCP 协议集成
- 为什么需要学习：MCP 是 Agent 扩展能力的标准协议
- 关键问题：如何实现 MCP 客户端和服务器？如何转换工具格式？
