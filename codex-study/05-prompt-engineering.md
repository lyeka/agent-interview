# 第五章：Prompt 工程

## 5.1 Prompt 架构概览

Codex 的 Prompt 系统是一个精心设计的分层架构，支持模块化组装、人格切换、协作模式、项目规范注入。

```
┌─────────────────────────────────────────────────────────────┐
│                    Final Prompt                              │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Base Instructions                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Agent 身份、能力定义、工具使用指南、输出规范        │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Model-Specific Instructions                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  模型专用指令 + {{ personality }} 占位符             │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 3: Personality                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Friendly / Pragmatic 人格模板                       │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 4: Collaboration Mode                                 │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Plan / Execute / Pair Programming / Code 模式       │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 5: User Instructions                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  用户配置的自定义指令                                │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 6: AGENTS.md                                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  项目特定规范（分层覆盖，深层优先）                   │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│  Layer 7: Skills                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  可用 Skills 清单 + 使用规则                         │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 5.2 Base Instructions

Base Instructions 定义了 Agent 的核心能力和工作流程。

### 核心内容

```markdown
# Base Instructions (default.md)

You are Codex, a coding agent in the Codex CLI.

## Core Capabilities
- Read and write files
- Execute shell commands
- Search and navigate codebases
- Apply code patches

## AGENTS.md Convention
- AGENTS.md files provide project-specific instructions
- They can appear at any level in the directory hierarchy
- Deeper files override shallower ones
- Always follow AGENTS.md instructions when modifying files

## Tool Usage Guidelines
- Use `apply_patch` for code modifications (unified diff format)
- Use `rg` (ripgrep) for searching, not `grep`
- Use `update_plan` to track progress on multi-step tasks

## Output Format
- Keep responses concise and scannable
- Use fenced code blocks with language identifiers
- Reference files with relative paths
```

### 设计哲学

Base Instructions 是"员工手册"——定义了 Agent 的职责、流程、规范。它应该是稳定的，不随用户或项目变化。

---

## 5.3 人格系统

### Personality 枚举

```rust
pub enum Personality {
    Friendly,
    Pragmatic,
}
```

### Friendly 人格

```markdown
# Personality (gpt-5.2-codex_friendly.md)

优化团队士气和支持性队友，温暖沟通，频繁检查，无自负解释概念。

## Values
- Empathy: 在人们所在的地方与他们相遇
- Collaboration: 主动技能，邀请输入，综合观点
- Ownership: 不仅对代码负责，还对队友是否解除阻塞负责

## Tone & User Experience
- 使用 "we" 和 "let's" 的团队导向语言
- 肯定进步，用好奇心替代判断
- 轻度热情和幽默，维持能量和专注
- 用户应感到安全提问，得到支持，真正合作
```

### Pragmatic 人格

```markdown
# Personality (gpt-5.2-codex_pragmatic.md)

深度务实、高效的软件工程师。认真对待工程质量，协作是一种安静的喜悦。

## Values
- Clarity: 明确具体地传达推理，决策和权衡易于评估
- Pragmatism: 关注最终目标和动力，专注于实际可行的方案
- Rigor: 期望技术论证连贯且可辩护，礼貌地指出差距

## Interaction Style
- 简洁尊重地沟通，专注于手头任务
- 优先提供可操作的指导，明确说明假设、环境前提和后续步骤
- 避免过度冗长的解释
```

### 人格注入机制

```rust
// openai_models.rs:225-244
impl ModelInfo {
    pub fn get_model_instructions(&self, personality: Option<Personality>) -> String {
        if let Some(template) = &self.model_messages.instructions_template {
            // 有模板：替换 {{ personality }} 占位符
            let personality_message = self.model_messages
                .get_personality_message(personality)
                .unwrap_or_default();
            template.replace("{{ personality }}", personality_message.as_str())
        } else {
            // 无模板：直接返回 base_instructions
            self.base_instructions.clone()
        }
    }
}
```

**设计哲学**：人格是"皮肤"，不是"骨架"。Friendly 和 Pragmatic 不改变 Agent 的能力，只改变交互风格。底层推理逻辑保持一致。

---

## 5.4 协作模式

### 四种模式

| 模式 | 文件 | 特点 |
|------|------|------|
| **Plan** | `plan.md` | 只读探索，生成 `<proposed_plan>` |
| **Execute** | `execute.md` | 独立执行，做假设，不协作 |
| **Pair Programming** | `pair_programming.md` | 逐步解释，邀请决策 |
| **Code** | `code.md` | 纯编码，无额外开销 |

### Plan Mode

```markdown
# Plan Mode (plan.md)

## 工作流
3 阶段对话式规划：
1. Phase 1 - Ground in environment: 先探索后提问，消除未知
2. Phase 2 - Intent chat: 明确目标、成功标准、范围、约束
3. Phase 3 - Implementation chat: 决策完整的规范

## 严格规则
- 只能执行非变更操作（读取、搜索、静态分析）
- 禁止编辑文件、运行格式化工具、应用补丁
- 每轮必须是：
  A) `request_user_input` 工具调用 OR
  B) 最终 `<proposed_plan>` 输出

## 输出格式
<proposed_plan>
# 标题
## tldr
## 重要签名/结构/类型变更
## 测试用例和场景
## 明确的假设和默认选择
</proposed_plan>
```

### Execute Mode

```markdown
# Execute Mode (execute.md)

## 哲学
独立执行，不协作决策。

## 核心原则
- Assumptions-first: 缺失信息时做合理假设，明确说明，继续执行
- Think out loud: 分享推理，帮助用户评估权衡
- Think ahead: 提前考虑用户需求（如测试模式、调试功能）
- Be mindful of time: 用户在等待，最小化延迟

## 进度报告
使用 plan 工具更新进度，映射实际工作。
```

### Pair Programming Mode

```markdown
# Pair Programming Mode (pair_programming.md)

## 风格
默认配对，用户在终端旁边。

## 行为
- 避免过大步骤或长时间操作（除非被要求）
- 逐步解释推理，动态调整深度
- 边构建边检查对齐和舒适度
- 多路径时提供清晰选项，友好框架，邀请用户决策
- 复杂工作时自由使用 planning 工具保持用户知情

## 调试
假设是团队，可以要求用户提供开发者工具错误消息或截图。
```

---

## 5.5 AGENTS.md 注入机制

### 发现逻辑

```rust
// project_doc.rs
pub fn discover_project_doc_paths(config: &Config) -> Vec<PathBuf> {
    // 1. 检测 Git 根目录（.git 标记）
    let git_root = find_git_root(&config.cwd);

    // 2. 从根目录到 cwd 构建路径链
    let path_chain = build_path_chain(git_root, &config.cwd);

    // 3. 优先级：AGENTS.override.md > AGENTS.md > fallback_filenames
    let mut paths = Vec::new();
    for dir in path_chain {
        if let Some(path) = find_agents_md(&dir) {
            paths.push(path);
        }
    }

    paths
}
```

### 组装逻辑

```rust
// project_doc.rs:39-84
pub async fn get_user_instructions(
    config: &Config,
    skills: Option<&[SkillMetadata]>,
) -> Option<String> {
    let mut output = String::new();

    // 1. Config 中的 user_instructions
    if let Some(instructions) = config.user_instructions.clone() {
        output.push_str(&instructions);
    }

    // 2. 拼接所有 AGENTS.md（用 "\n\n" 分隔）
    if let Ok(Some(docs)) = read_project_docs(config).await {
        if !output.is_empty() {
            output.push_str("\n\n--- project-doc ---\n\n");
        }
        output.push_str(&docs);
    }

    // 3. 拼接 Skills 清单
    if let Some(skills_section) = render_skills_section(skills) {
        output.push_str("\n\n");
        output.push_str(&skills_section);
    }

    Some(output)
}
```

### 分层覆盖规则

```markdown
# Hierarchical AGENTS.md Rules

- AGENTS.md 可出现在容器任何位置（/、~、Git 仓库深处）
- 每个 AGENTS.md 管辖其所在目录及所有子目录
- 修改文件时，必须遵守覆盖该文件的所有 AGENTS.md
- 冲突时：更深层的 AGENTS.md 覆盖上层
- 优先级：system/developer/user 直接指令 > AGENTS.md
```

**设计哲学**：AGENTS.md 的分层覆盖类似文件系统权限——更深层的规则覆盖上层，用户直接指令最高优先级。

---

## 5.6 Skill 注入

### Skill 清单渲染

```rust
// skills/render.rs
pub fn render_skills_section(skills: &[SkillMetadata]) -> Option<String> {
    // 输出格式：
    // ## Skills
    // ### Available skills
    // - {name}: {description} (file: {path})
    // ### How to use skills
    // - Discovery: 列表是本会话可用的 skills
    // - Trigger rules: 用户提及 skill 名称或任务匹配描述时必须使用
    // - Progressive disclosure: 先读 SKILL.md，按需加载 references/
    // - Context hygiene: 保持上下文小，总结长段落，避免深度引用追踪
}
```

### Skill 注入格式

```xml
<skill>
<name>{skill_name}</name>
<path>{skill_path}</path>
{skill_contents}
</skill>
```

---

## 5.7 权限与沙箱 Prompt

### Sandbox Mode

| 模式 | 文件 | 权限 |
|------|------|------|
| **read_only** | `read_only.md` | 只读文件，网络可配置 |
| **workspace_write** | `workspace_write.md` | 写入工作区 |
| **danger_full_access** | `danger_full_access.md` | 完全访问 |

### Approval Policy

| 策略 | 文件 | 行为 |
|------|------|------|
| **on_request** | `on_request.md` | 默认沙箱，可请求升级 |
| **on_failure** | `on_failure.md` | 失败后请求批准 |
| **unless_trusted** | `unless_trusted.md` | 非信任命令需批准 |
| **never** | `never.md` | 从不请求批准 |

### on_request 模式示例

```markdown
# Approval Policy: on_request

需要升级权限的场景：
- 写入受保护目录
- GUI 应用
- 网络访问
- 潜在破坏性操作

请求方式：
sandbox_permissions: "require_escalated"
justification: "需要访问 /etc/hosts 修改 DNS 配置"
```

---

## 5.8 Prompt 组装完整流程

### 组装顺序

```
1. Model-specific base_instructions (from models.json or hardcoded)
   ↓
2. Personality injection ({{ personality }} → friendly/pragmatic content)
   ↓
3. Collaboration mode instructions (plan/execute/pair_programming/code)
   ↓
4. User instructions from config
   ↓
5. AGENTS.md (hierarchical, from repo root to cwd)
   ↓
6. Skills section (available skills + usage rules)
   ↓
7. Hierarchical AGENTS.md explanation
   ↓
8. Sandbox/approval policy instructions
   ↓
9. Developer message (user's actual query)
```

### 代码实现

```rust
// 概念模型
pub fn build_final_prompt(
    model_info: &ModelInfo,
    personality: Option<Personality>,
    collaboration_mode: CollaborationMode,
    user_instructions: Option<&str>,
    agents_md: &[String],
    skills: &[SkillMetadata],
    sandbox_policy: &SandboxPolicy,
    approval_policy: AskForApproval,
) -> String {
    let mut prompt = String::new();

    // 1. Base + Personality
    prompt.push_str(&model_info.get_model_instructions(personality));

    // 2. Collaboration Mode
    prompt.push_str("\n\n");
    prompt.push_str(&get_collaboration_mode_instructions(collaboration_mode));

    // 3. User Instructions
    if let Some(instructions) = user_instructions {
        prompt.push_str("\n\n");
        prompt.push_str(instructions);
    }

    // 4. AGENTS.md
    for doc in agents_md {
        prompt.push_str("\n\n--- project-doc ---\n\n");
        prompt.push_str(doc);
    }

    // 5. Skills
    if let Some(skills_section) = render_skills_section(skills) {
        prompt.push_str("\n\n");
        prompt.push_str(&skills_section);
    }

    // 6. Sandbox/Approval
    prompt.push_str("\n\n");
    prompt.push_str(&get_sandbox_instructions(sandbox_policy));
    prompt.push_str("\n\n");
    prompt.push_str(&get_approval_instructions(approval_policy));

    prompt
}
```

---

## 5.9 设计哲学总结

### 可组合性是复杂度的解药

不是写一个 10000 行的超级 Prompt，而是 10 个 1000 行的模块化 Prompt，运行时组装。每个模块可独立演化、测试、替换。

### 上下文即状态，状态需管理

LLM 的上下文窗口是稀缺资源。Compaction 机制是对"时间使状态产生歧义"的对抗。Progressive disclosure 是对"不可变性带来确定性"的实践。

### 人格是接口，不是实现

Friendly/Pragmatic 不是改变 Agent 能力，而是改变用户体验的"皮肤"。底层推理逻辑保持一致，表达方式可切换。

### 分层覆盖是权限模型

AGENTS.md 的分层覆盖类似文件系统权限。更深层的规则覆盖上层 = 更具体的规则优先。用户直接指令最高优先级 = root 权限。

---

## 章节衔接

**本章回顾**：
- 我们深入分析了 Prompt 工程的设计
- 关键收获：分层架构 + 人格系统 + 协作模式 + AGENTS.md 注入

**下一章预告**：
- 在 `06-sandbox-security.md` 中，我们将深入沙箱安全机制
- 为什么需要学习：沙箱是 Agent 安全的最后防线
- 关键问题：如何实现跨平台沙箱？如何平衡安全与便利？
