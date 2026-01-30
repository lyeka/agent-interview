# AI Agent 工程方向知识梳理

> AI Agent 工程方向的学习资料、面试准备、技术栈梳理

## 文档定位

- **目标读者**：想要了解/转型 AI Agent 工程方向的开发者
- **文档性质**：知识梳理 + 学习指南 + 面试准备

---

## 一、什么是 AI Agent

### 1.1 定义

**AI Agent** 是能够**感知环境、推理决策、执行行动、拥有记忆**的智能体。

| 特性 | Chatbot | AI Agent |
|------|---------|----------|
| **感知** | 被动接收输入 | 主动获取信息 |
| **推理** | 单次响应 | 多步推理、规划 |
| **行动** | 仅返回文本 | 调用工具、执行操作 |
| **记忆** | 无状态/简单历史 | 长期记忆、学习积累 |

**核心能力**：
- **感知（Perception）**：理解用户输入和环境状态
- **推理（Reasoning）**：规划任务步骤、做出决策
- **行动（Action）**：调用工具、执行操作
- **记忆（Memory）**：存储和检索信息、跨会话学习

### 1.2 Agent 工作原理：ReAct 模式

```
┌─────────────────────────────────────────────────────────────┐
│                    ReAct 循环（推理-行动）                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   Thought (思考)   →   Action (行动)   →   Observation (观察) │
│        │                  │                    │             │
│        └──────────────────┴────────────────────┘             │
│                           │                                  │
│                           ↓                                  │
│                    下一轮思考...                              │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**与传统程序的区别**：
- **传统程序**：确定性逻辑，相同输入必然产生相同输出
- **AI Agent**：概率性推理，每次执行可能产生不同路径

---

## 二、AI Agent 技术栈全景

```
┌─────────────────────────────────────────────────────────────┐
│                      AI Agent 技术栈                          │
├─────────────────────────────────────────────────────────────┤
│  LLM 层          │  OpenAI, Anthropic, Llama, DeepSeek       │
│  框架层          │  LangChain, LangGraph, CrewAI, AutoGen     │
│  数据层          │  向量数据库, RAG 架构, Memory 系统         │
│  编排层          │  Single/Multi-Agent, 工作流, 通信协议      │
│  工程层          │  Prompt, 优化, 监控, 安全, 测试            │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、核心知识模块

### 3.1 LLM 基础

#### Token 管理
- **计算规则**：英文约 1 token = 4 字符，中文约 1 token = 1-2 字符
- **限制**：每个模型有 Context Window 上限（如 GPT-4o: 128K）
- **优化策略**：精简 Prompt、清理历史、使用压缩

#### Context Window 管理
- **Context Rot（上下文腐烂）**：上下文过长时，模型准确回忆信息能力下降
- **管理策略**：动态检索、压缩清理、分块处理

#### 采样参数
| 参数 | 作用 | 典型值 |
|------|------|--------|
| Temperature | 控制随机性 | 0-1，0=确定性，1=创造性 |
| Top-p | 核采样阈值 | 0.1-1.0 |
| Top-k | 保留前 k 个候选 | 1-50 |
| Max Tokens | 最大输出长度 | 根据需求设置 |

#### Embedding
- **定义**：将文本转换为高维向量表示
- **用途**：语义搜索、相似度计算、聚类
- **主流模型**：OpenAI text-embedding-3, Cohere embed-v3

### 3.2 Context Engineering（上下文工程）⭐

> **Context Engineering 是 2025 年 AI 工程最重要的新兴概念**

#### 核心定义

**Context Engineering** 是在 LLM 推理期间**策展和维护最优 token 集（信息）**的策略集合。

```
Prompt Engineering      →  如何编写有效指令
Context Engineering     →  如何管理整个上下文状态
```

#### 与 Prompt Engineering 的区别

| 维度 | Prompt Engineering | Context Engineering |
|------|-------------------|---------------------|
| 焦点 | 指令编写 | 信息策展 |
| 范围 | System prompt | 所有上下文组件 |
| 时间特性 | 离散任务 | 迭代过程 |
| 适用场景 | 单次任务 | 多轮推理、长时间跨度 |

#### 核心策略

1. **最小高信号原则**
   - 目标：找到最大化期望结果的最小高信号 token 集
   - 实践：每个上下文组件都应"信息丰富但紧凑"

2. **动态上下文检索（Just-in-Time Context）**
   - 运行时动态加载数据，而非预处理所有相关数据
   - Claude Code 使用文件路径标识符，按需加载

3. **Compaction（压缩）**
   - 对话接近上下文窗口限制时，总结并重新初始化
   - 保留架构决策、未解决问题，丢弃冗余输出

4. **结构化笔记**
   - Agent 定期将笔记持久化到上下文窗口外的记忆中
   - 示例：NOTES.md、TODO 列表

### 3.3 Prompt Engineering

#### Few-shot Learning
```
示例 1: 输入 -> 输出
示例 2: 输入 -> 输出
示例 3: 输入 -> 输出
---
问题: 输入 -> ?
```

#### Chain-of-Thought
```
问题: ...
思考: 让我一步步分析...
  第一步: ...
  第二步: ...
  第三步: ...
答案: ...
```

#### ReAct Prompting
```
Thought: 我需要先查询...
Action: 查询工具(...)
Observation: 工具返回结果...
Thought: 基于结果，我需要...
Action: 下一个工具(...)
```

#### 幻觉抑制技巧
- 要求模型"不确定时说不知道"
- 提供具体参考资料
- 使用 RAG 增强事实准确性

### 3.4 Single Agent 开发

#### Agent 模式

| 模式 | 描述 | 适用场景 |
|------|------|----------|
| ReAct | 推理-行动交替循环 | 通用任务解决 |
| Plan-and-Execute | 先规划再执行 | 复杂多步骤任务 |
| Router | 根据输入路由到不同处理 | 分类、分发 |

#### Function Calling / Tool Use

```python
# 工具定义
tools = [
    {
        "name": "get_weather",
        "description": "获取指定城市的天气",
        "parameters": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"}
            },
            "required": ["city"]
        }
    }
]

# LLM 决定调用工具
response = llm.call_with_tools(messages, tools)
tool_call = response.tool_calls[0]

# 执行工具
result = get_weather(tool_call["city"])

# 将结果返回给 LLM
final_response = llm.call_with_tool_result(messages, result)
```

#### 工具集成最佳实践
- 工具描述要清晰：说明功能、参数、返回值
- 错误处理：捕获异常，返回友好错误信息
- Token 高效：避免返回冗余数据

### 3.5 Model Context Protocol (MCP) ⭐

> **MCP 是 2025 年最重要的 AI Agent 标准之一**

#### 什么是 MCP

**Model Context Protocol** 是连接 AI Agent 与外部系统的**开放标准**。

```
┌─────────────────────────────────────────────────────────────┐
│                      MCP 架构                                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   AI Agent   ←→   MCP Client   ←→   MCP Server   ←→   外部系统 │
│                    (本地)          (标准协议)      (API/数据库) │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

#### MCP 解决的问题

- **标准化**：统一的工具连接协议，无需为每个系统写适配器
- **解耦**：Agent 与外部系统松耦合
- **可扩展**：社区贡献 MCP 服务器，共享工具生态

#### 2025-2026 发展

- 捐赠给 Linux Foundation，成立 Agentic AI Foundation
- MCP Registry：社区驱动的服务器发现平台
- Gartner 预测：2026 年 75% API 网关将支持 MCP

### 3.6 Multi-Agent 系统

#### 8 种核心设计模式

| 模式 | 描述 | 适用场景 |
|------|------|----------|
| **Sequential Pipeline** | 按顺序处理，每个输出是下一个输入 | 步骤依赖的工作流 |
| **Hierarchical/Supervisor** | 主 Agent 协调子 Agent | 复杂企业任务 |
| **Parallel/Map-Reduce** | 并行工作，结果聚合 | 独立任务批量处理 |
| **Central Coordinator** | 单一协调节点管理通信 | 简单协调场景 |
| **Team-Based** | Agent 组队协作，共享目标 | 复杂问题解决 |
| **Flat** | 所有 Agent 平等，点对点通信 | 去中心化场景 |
| **Hybrid** | 结合多种模式元素 | 大型企业系统 |
| **Agent-to-Agent (A2A)** | Agent 直接通信，无需中央协调 | 去中心化消息传递 |

#### 架构选型

```
简单任务          →  Sequential / Flat
需要专业分工      →  Hierarchical / Team-Based
独立任务批量处理  →  Parallel
大型企业系统      →  Hybrid
```

#### 主流框架对比

| 框架 | 优势 | 适用场景 | 学习难度 |
|------|------|----------|----------|
| **LangGraph** | 生产级控制、DAG 支持 | 复杂工作流、状态管理 | 中高 |
| **Google ADK** | 8 种核心模式、Gemini 深度集成 | Multi-Agent 系统 | 中 |
| **CrewAI** | 角色驱动、快速协作 | 快速原型 | 低 |
| **AutoGen** | 程序化、完整代码控制 | 自定义实现 | 中 |

### 3.7 Memory 系统

#### RAG ≠ Agent Memory

| RAG | Agent Memory |
|-----|--------------|
| 从外部知识库检索静态信息 | 支持学习和自主性的内部基础 |
| 无状态 | 有状态、可学习 |
| 检索预存储的知识 | 积累交互经验 |

#### Memory 类型

| 类型 | 定义 | 存储方式 | 用途 |
|------|------|----------|------|
| **Episodic Memory** | 过去交互的事件记录 | Vector Database | 个性化、上下文连续性 |
| **Semantic Memory** | 抽象知识和事实 | 知识图谱、结构化数据库 | 推理、知识查询 |
| **Working Memory** | 当前任务的活动上下文 | 上下文窗口内 | 即时推理、任务追踪 |
| **Long-term Memory** | 跨会话持久信息 | NOTES.md、文件系统 | 长时间跨度策略、学习积累 |

#### 实现方式

```python
# Episodic Memory 示例
episodic_memory.add({
    "content": "用户昨天问了关于 Python 装饰器的问题",
    "timestamp": "2025-01-29",
    "embed": embed("用户昨天问了关于 Python 装饰器的问题")
})

# Long-term Memory 示例（NOTES.md）
# Agent 定期将重要信息写入 NOTES.md
agent.write_note("用户偏好使用 TypeScript 进行前端开发")
```

### 3.8 RAG（检索增强生成）

#### RAG 架构类型

| 架构 | 描述 | 适用场景 |
|------|------|----------|
| **Vanilla RAG** | 基础检索+生成 | 简单问答 |
| **Self-RAG** | 自我反思与修正 | 高精度要求 |
| **Graph-RAG** | 知识图谱增强 | 复杂关系推理 |
| **Hybrid RAG** | 多种检索策略融合 | 综合性能 |
| **Agentic RAG** | Agent 自主决定检索 | 动态信息获取 |

#### RAG 流程

```
问题 → Embedding → 向量检索 → Top-K 文档 → LLM 生成 → 答案
```

#### 向量数据库选型

| 数据库 | 最佳场景 | 成本 |
|--------|----------|------|
| **ChromaDB** | 快速原型、本地开发 | 开源免费 |
| **Pinecone** | 生产 RAG、实时搜索 | 付费托管 |
| **Milvus** | 大规模企业部署 | 开源+云 |
| **Weaviate** | 混合搜索需求 | 开源+云 |

### 3.9 生产工程

#### 成本优化
- **Token 优化**：精简 Prompt、清理冗余历史
- **模型选择**：简单任务用小模型（gpt-4o-mini）
- **缓存策略**：语义缓存、避免重复调用

#### 性能优化
- **并行调用**：多个工具同时调用
- **流式响应**：逐步返回结果
- **Context 管理**：动态加载，避免过长上下文

#### 可观测性
- **追踪调试**：LangSmith、Braintrust
- **监控告警**：记录 Agent 执行轨迹
- **日志分析**：输入、思考、工具调用、错误

#### 安全防护
- **Guardrails**：输出过滤、边界限制
- **Prompt 注入防护**：检测恶意输入
- **MCP 安全**：服务器认证、权限控制

---

## 四、面试核心考察点

### 4.1 概念理解

- **Agent vs Chatbot**：核心能力差异（感知、推理、行动、记忆）
- **ReAct 模式**：Thought → Action → Observation 循环
- **Context Engineering**：上下文策展 vs Prompt 优化
- **RAG 原理**：检索增强生成，优化策略
- **MCP 协议**：Model Context Protocol 作用和价值
- **Memory 架构**：RAG ≠ Agent Memory

### 4.2 工程实现

- **Function Calling / Tool Use**：工具定义、参数设计、错误处理
- **Context 管理**：动态检索、压缩策略、笔记系统
- **Memory 实现**：Episodic、Semantic、Working、Long-term
- **Prompt/Context 优化**：减少幻觉、提高信号质量

### 4.3 系统设计

- **Single Agent**：ReAct 模式、工具集成
- **Multi-Agent 架构**：8 种模式选型（Hierarchical/Flat/Parallel/A2A 等）
- **RAG 系统**：向量数据库选型、检索策略
- **生产部署**：可扩展性、可靠性、安全性

### 4.4 生产实践

- **成本优化**：Token 优化、模型选择、缓存策略
- **性能优化**：延迟优化、并行调用、Context 管理
- **可观测性**：追踪调试、监控告警（LangSmith、Braintrust）
- **安全防护**：Guardrails、Prompt 注入、MCP 安全
- **评估测试**：质量指标、A/B 测试、自动化基准

---

## 五、常见面试题

### 概念类

1. **什么是 AI Agent？与 Chatbot 的核心区别是什么？**
   - Agent 具备感知、推理、行动、记忆四大核心能力
   - Chatbot 仅被动响应，Agent 可主动执行操作

2. **解释 Context Engineering vs Prompt Engineering**
   - Prompt：如何编写有效指令
   - Context：如何管理整个上下文状态（包括指令、工具、数据、历史）

3. **什么是 ReAct 模式？**
   - Thought（思考）→ Action（行动）→ Observation（观察）循环
   - 推理与行动交替进行

4. **RAG 是什么？RAG 与 Agent Memory 的区别？**
   - RAG：从外部知识库检索静态信息
   - Agent Memory：支持学习和自主性的内部基础，可积累经验

5. **MCP（Model Context Protocol）是什么？**
   - 连接 AI Agent 与外部系统的开放标准
   - 解决工具连接的标准化问题

6. **Context Rot 现象是什么？**
   - 上下文过长时，模型准确回忆信息能力下降
   - 对策：压缩、笔记、子 Agent

### 实现类

1. **如何实现 Function Calling？**
   - 定义工具（name, description, parameters）
   - LLM 决定是否调用及参数
   - 执行工具并将结果返回给 LLM

2. **如何设计一个文档问答系统？**
   - 文档加载 → 分块 → Embedding → 向量存储 → 检索 → LLM 生成

3. **如何优化 Context 以减少幻觉？**
   - 最小高信号原则：只保留必要信息
   - 提供可靠参考资料
   - 使用 RAG 增强事实准确性

4. **如何实现 Agent 的长期记忆？**
   - NOTES.md：结构化笔记持久化
   - Episodic Memory：向量数据库存储历史事件
   - 文件系统：关键信息持久化

5. **如何处理长上下文场景？**
   - Compaction：压缩总结历史对话
   - Note-taking：将重要信息持久化到外部
   - Sub-Agent：专门化子 Agent 处理深层内容

### 系统设计类

1. **设计一个 Multi-Agent 客服系统**
   - 架构：Hierarchical（监督器模式）
   - Agent：分类 Agent、查询 Agent、退款 Agent、人工接管 Agent
   - 通信：监督器协调，子 Agent 专业分工

2. **设计一个代码审查 Agent**
   - 工具：代码读取、静态分析、安全扫描
   - Memory：历史审查记录、代码规范知识库
   - Context：当前代码 + 相关历史 + 规范文档

3. **设计一个百万级用户的 RAG 系统**
   - 向量数据库：Pinecone/Milvus（分布式部署）
   - 缓存：Redis 语义缓存
   - 优化：分片、负载均衡、模型选择

4. **如何监控和调试 Agent？**
   - 追踪：LangSmith 记录完整执行轨迹
   - 监控：记录输入、思考、工具调用、错误
   - 日志：结构化日志，便于分析

5. **MCP 服务器如何设计？**
   - 标准接口：实现 MCP 协议
   - 安全：认证、权限控制
   - 可扩展：支持动态工具注册

### 场景类

1. **Agent 频繁调用同一工具导致成本高，如何优化？**
   - 缓存：相同输入直接返回缓存结果
   - 批处理：合并多个请求
   - Prompt 优化：明确告诉模型优先使用缓存

2. **如何处理 Agent 循环调用问题？**
   - 最大迭代次数限制
   - 状态检测：检测重复状态
   - 中断机制：人工介入

3. **生产环境部署需要注意哪些安全问题？**
   - Guardrails：输出过滤
   - Prompt 注入防护：检测恶意输入
   - MCP 安全：服务器认证、权限控制
   - 数据隔离：敏感数据不进入上下文

4. **如何评估一个 Agent 系统的质量？**
   - 任务成功率
   - 响应准确率
   - 成本效率
   - 用户满意度
   - A/B 测试对比

5. **Context Window 不足时如何设计系统？**
   - 动态检索：按需加载信息
   - 压缩清理：定期总结历史
   - 子 Agent：将深层探索交给专门 Agent
   - 长期 Memory：重要信息持久化到外部

---

## 六、推荐学习资源

### 官方文档
- [Model Context Protocol](https://modelcontextprotocol.io)
- [OpenAI API Docs](https://platform.openai.com/docs) - o1/o3 推理模型
- [Anthropic API Docs](https://docs.anthropic.com) - Claude、Computer Use
- [LangChain Docs](https://python.langchain.com)
- [LangGraph Docs](https://langchain-ai.github.io/langgraph)
- [Google ADK Docs](https://google.github.io/adk-docs)

### 开源项目
- [LangChain](https://github.com/langchain-ai/langchain) - 122K+ stars
- [MetaGPT](https://github.com/geekan/MetaGPT) - 62K+ stars
- [AutoGen](https://github.com/microsoft/autogen) - 53K+ stars
- [CrewAI](https://github.com/joaomdmoura/crewAI) - 10K+ stars

### 学习路径
- [Roadmap.sh - AI Agents](https://roadmap.sh/ai-agents)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
- [Google ADK Multi-Agent Guide](https://developers.googleblog.com/developers-guide-to-multi-agent-patterns-in-adk/)

### 技术博客（必读）
- [Anthropic - Effective Context Engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Anthropic - Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [Google Cloud Blog - Multi-Agent Systems with ADK](https://cloud.google.com/blog/topics/developers-practitioners/building-collaborative-ai-a-developers-guide-to-multi-agent-systems-with-adk)

---

## 七、实战项目建议

| 难度 | 项目 | 涉及技术 | 学习目标 |
|------|------|----------|----------|
| Beginner | 文档问答 Agent | RAG、向量数据库、LangChain | 掌握基础 RAG 流程 |
| Beginner | 工具调用 Agent | Function Calling、API 封装 | 掌握工具集成 |
| Beginner | 天气查询助手 | Prompt、API 调用 | 理解 Agent 基础 |
| Intermediate | 研究助手 Agent | Multi-Agent、Web 搜索 | 掌握 Agent 协作 |
| Intermediate | 代码审查 Agent | Context Engineering、MCP | 理解上下文管理 |
| Intermediate | 数据分析师 Agent | 工具链、数据处理 | 掌握复杂任务编排 |
| Advanced | 软件开发团队 | Multi-Agent 协作 | 理解生产级系统 |
| Advanced | 自主研究员 | 长期 Memory、反思 | 掌握高级 Agent |

---

## 八、学习建议

1. **先理解概念，再动手实践**
   - 理解 ReAct 模式 → 跑通第一个 Agent
   - 理解 RAG → 实现文档问答
   - 理解 Multi-Agent → 构建协作系统

2. **从简单到复杂**
   - 先用 LangChain 快速原型
   - 再用 LangGraph 掌握状态管理
   - 最后考虑生产部署问题

3. **关注核心能力**
   - Context Engineering：学会管理上下文
   - Prompt 优化：减少幻觉、提高准确率
   - 成本控制：Token 优化、模型选择
   - 调试技巧：追踪、监控、日志

4. **边学边做**
   - 每学一个概念，动手实现一个小项目
   - 把项目放到 GitHub，积累作品集
   - 记录学习笔记和踩坑经验

---

## 附录

### 主流框架对比（2025-2026）

| 框架 | 特点 | 适用场景 | 学习难度 |
|------|------|----------|----------|
| **LangGraph** | 生产级控制、DAG 支持 | 复杂工作流、状态管理 | 中高 |
| **Google ADK** | 8 种核心模式、Gemini 深度集成 | Multi-Agent 系统 | 中 |
| **LangChain** | 生态最全 | 通用开发 | 中 |
| **CrewAI** | 角色驱动、快速协作 | 快速原型 | 低 |
| **AutoGen** | 程序化、完整代码控制 | 自定义实现 | 中 |
| **MetaGPT** | SOP 驱动 | 企业级 | 高 |
| **Dify/Temporal** | 生产就绪、工作流编排 | 生产部署 | 中 |

### 编排框架对比

| 框架 | 优势 | 适用场景 |
|------|------|----------|
| LangGraph | Multi-Agent 编排领导者 | 复杂工作流 |
| AutoGen | 高灵活性 | 自定义实现 |
| CrewAI | 易于设置 | 快速原型 |
