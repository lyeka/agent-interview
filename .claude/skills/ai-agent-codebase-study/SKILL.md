---
name: ai-agent-codebase-study
description: |
  深度分析 AI Agent 代码仓库，生成结构化学习文档用于面试准备。

  **适用项目**：包含 Agent/LLM 相关代码的项目
  - Agent 框架（LangChain、CrewAI、AutoGen 等）
  - Agent 应用（客服 Agent、研究助手、代码审查等）
  - Agent 工具（MCP Server、Tool Library 等）

  **触发场景**：
  1. 用户说"研究这个 Agent 项目"、"分析这个 Agent 代码库"
  2. 用户说"生成项目文档"、"创建学习笔记"（且项目是 AI Agent）
  3. 用户说"准备面试"、"梳理技术栈"（且项目是 AI Agent）
  4. 项目代码包含 agent/llm/tool calling/mcp 等关键词

  **输出**：{repo-name}-study/ 文件夹（包含多个章节文档）
  **结构**：动态制定，根据源码实际情况调整
---

# AI Agent Codebase Study

## Overview

深度分析 AI Agent 项目，生成面试准备材料。

**核心原则**：
- 根据源码实际情况动态制定章节组织（非固定套路）
- 讲解 70% + 代码 30%（黄金比例）
- 代码片段精炼（关键逻辑 + 行号 + "为什么"）
- 无 emoji，严肃文学风格

用户作为该项目"开发者"参加面试时，这份材料是回答的核心依据。

## 输出格式要求

**学习资料格式**：
- 不需要 GEB 分形文档标记（无 [INPUT]/[OUTPUT]/[POS]/[PROTOCOL]）
- 不需要在 study/ 文件夹下创建 CLAUDE.md
- 纯学习资料格式，面向阅读优先
- 每个文件开头无需 L3 头部契约

**版本声明**（00-index.md 开头必须包含）：
```markdown
> **分析版本**: `{commit_hash}` ({branch_name})
> **分析日期**: {date}
> **仓库地址**: {repo_url}
```

## 代码展示原则

**黄金比例**：讲解 70% + 代码 30%

**代码展示规则**：
1. 每个代码块不超过 20 行
2. 只展示核心逻辑，省略样板代码
3. 必须有前置讲解（为什么需要这段代码）
4. 必须有后置分析（这段代码的设计意图）
5. 用伪代码或流程图替代冗长实现

**反模式**：
- 大段复制源码（> 30 行）
- 连续多个代码块无讲解
- 代码块内无注释说明

## Analysis Workflow

### Phase 1: 侦察与特性识别

使用 Explore agent 快速侦察：
- 项目类型（框架/应用/工具）
- 核心 AI Agent 特性（ReAct/Multi-Agent/Memory/Tool/MCP）
- 技术栈和依赖关系
- 目录结构和模块划分

**获取版本信息**（必须执行）：
```bash
git rev-parse HEAD                    # commit hash
git rev-parse --abbrev-ref HEAD       # branch name
git remote get-url origin             # repo url
date +%Y-%m-%d                        # 分析日期
```

### Phase 2: 动态章节规划

根据 Phase 1 的发现，动态规划章节组织。

**通用章节**（所有项目）：
- `00-index.md` - 总目录 + 学习路径 + 核心术语速查
- `0X-deep-dive.md` - 核心源码深度解析
- `0Y-interview-qa.md` - 面试 QA（15 题）
- `XX-summary.md` - 知识图谱总结（最后一章）

**动态章节**（根据项目实际）：
- 基础抽象层（如果有自研框架）
- Agent 核心循环（如果是 Agent 应用）
- Context 管理（如果有复杂 Context 策略）
- Prompt 系统（如果有 Prompt 模板）
- Memory 系统（如果实现了 Memory）
- Tool 系统（如果定义了工具）
- Multi-Agent（如果是 Multi-Agent 项目）
- MCP 集成（如果支持 MCP）
- 协议层（如果有 ACP/Wire 等协议）

### Phase 3: 并行深度探索

根据 Phase 2 的章节规划，启动 3-5 个 Explore agents 并行探索：

**每个 agent 任务**：
- 读取完整文件（不是摘要）
- 分析代码逻辑和设计模式
- 提取关键代码片段（带行号，不超过 20 行）
- 识别可面试的技术点
- 分析设计决策（为什么这样设计、trade-off）

**thoroughness 设置**：
- 核心模块：very thorough
- 扩展模块：medium
- 协议层：medium

### Phase 4: 文档生成

生成章节文档，每个章节包含：

1. **概念解释**（是什么）
2. **代码实现**（怎么做 - 带行号，精炼）
3. **设计分析**（为什么这样设计）
4. **对比分析**（与其他框架的区别）
5. **章节衔接**（本章回顾 + 下一章预告）

**章节衔接格式**（每章结尾必须包含）：
```markdown
---

## 章节衔接

**本章回顾**：
- 我们学习了 {本章核心内容}
- 关键收获：{1-2 个核心知识点}

**下一章预告**：
- 在 `{下一章文件名}` 中，我们将学习 {下一章主题}
- 为什么需要学习：{本章内容如何引出下一章}
- 关键问题：{下一章要解决的核心问题}
```

### Phase 5: 名词解释生成

在 `00-index.md` 末尾生成「核心术语速查」章节。

**费曼学习法**：用最简单的语言解释复杂概念

**讲解模板**：
1. 一句话定义
2. 生活类比
3. 为什么重要
4. 在本项目中

参考 `references/glossary_guide.md` 获取详细指南。

### Phase 6: 面试 QA 生成

生成 15 道核心题，参考 `references/interview_qa_guide.md`：

**动态 QA 策略**：
- 以面试官视角设计问题
- 根据项目核心方向动态分配比例
- 项目核心方向占 60%+
- 从源码反推问题

**QA 格式**：
- 类别标签（架构决策/框架选择/Agent 核心/工具系统/...）
- 深度问题（为什么、trade-off、对比、如果重构）
- 源码引用（{文件}:{行号}）

### Phase 7: 总结章节生成

生成 `XX-summary.md`（编号根据实际章节数动态调整）：

**内容结构**：
1. 核心概念关系图（ASCII 图）
2. 学习检查清单（基础/深度/实战三级）
3. 知识点速查表
4. 推荐学习路径（新手/进阶/速成）

参考 `references/chapter_templates.md` 获取详细模板。

## Document Style

### 严肃文学风格

- 无 emoji
- 专业术语准确
- 技术密度高（每段都有实质信息）
- 拒绝"废话文学"
- 代码片段精炼（关键逻辑 + 行号）

### 可视化辅助

- 架构图（组件关系）
- 流程图（执行路径）
- 时序图（交互过程）
- 对比表格（框架对比、技术选型）

### 讲解与代码平衡

- 讲解 70%（设计意图、架构决策、trade-off 分析、概念解释）
- 代码 30%（源码片段、实战案例、常见陷阱）

## Resources

- `references/interview_qa_guide.md` - 面试 QA 生成指南（动态策略）
- `references/chapter_templates.md` - 章节模板（动态选择）
- `references/code_analysis_guide.md` - 代码分析指南
- `references/glossary_guide.md` - 名词解释生成指南（费曼学习法）
