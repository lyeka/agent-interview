# 附录

> 本章包含术语表、文件索引和参考资源。

---

## 术语表

| 术语 | 英文 | 解释 |
|------|------|------|
| ReAct | Reasoning + Acting | Agent 的核心循环：思考-行动-观察 |
| Context Rot | - | 上下文过长时，LLM "遗忘"早期信息的现象 |
| MCP | Model Context Protocol | 工具标准协议，实现工具生态化 |
| ACP | Agent Client Protocol | IDE 与 Agent 通信的标准协议 |
| SPMC | Single Producer, Multiple Consumer | 事件总线模型，单一发布者、多个订阅者 |
| D-Mail | - | "时间旅行"机制，修改历史并重新生成 |
| Checkpoint | - | 状态快照，支持回滚 |
| Compaction | - | 上下文压缩，减少 token 消耗 |
| LaborMarket | - | Agent 劳动力市场，负责任务分配 |
| Fixed Agent | - | 预定义的专用 Agent |
| Dynamic Agent | - | 根据任务动态创建的 Agent |

---

## 文件索引

### 核心模块

| 文件 | 行号 | 功能 |
|------|------|------|
| `src/kimi_cli/soul/kimisoul.py` | 120-200 | KimiSoul 主循环 |
| `src/kimi_cli/soul/context.py` | - | 上下文管理 |
| `src/kimi_cli/soul/compaction.py` | 42-117 | SimpleCompaction 压缩算法 |
| `src/kimi_cli/soul/toolset.py` | 80-150 | KimiToolset 依赖注入 |
| `src/kimi_cli/soul/agent.py` | 200-280 | LaborMarket 架构 |

### 基础抽象

| 文件 | 行号 | 功能 |
|------|------|------|
| `packages/kosong/src/kosong/message.py` | 15-50 | Kosong 消息结构 |
| `packages/kosong/src/kosong/tooling/base.py` | - | ToolReturnValue 三层语义 |
| `packages/kaos/src/kaos/path.py` | - | PyKAOS 路径抽象 |

### 工具系统

| 文件 | 行号 | 功能 |
|------|------|------|
| `src/kimi_cli/mcp.py` | 45-120 | MCP 集成 |
| `src/kimi_cli/tools/multiagent/task.py` | 52-150 | Task 工具 |

### 协议层

| 文件 | 行号 | 功能 |
|------|------|------|
| `src/kimi_cli/wire/types.py` | - | Wire 事件总线 |
| `src/kimi_cli/acp/` | - | ACP 协议实现 |

---

## 参考资源

### 官方文档

- [kimi-cli GitHub](https://github.com/MoonshotAI/kimi-cli)
- [MCP 协议规范](https://modelcontextprotocol.io/)
- [Anthropic Messages API](https://docs.anthropic.com/en/api/messages)

### 相关文章

- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)
- [AutoGen: Enabling Next-Gen LLM Applications](https://arxiv.org/abs/2308.08155)

### 类似框架

| 框架 | 特点 | 链接 |
|------|------|------|
| LangChain | 最流行的 LLM 应用框架 | [github.com/langchain-ai/langchain](https://github.com/langchain-ai/langchain) |
| LangGraph | 状态机建模 Agent 工作流 | [github.com/langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) |
| AutoGen | 对话驱动多 Agent | [github.com/microsoft/autogen](https://github.com/microsoft/autogen) |
| CrewAI | 角色分配组织 Agent | [github.com/joaomdmoura/crewai](https://github.com/joaomdmoura/crewai) |

---

## 学习建议

### 第一遍：理解架构

1. 阅读 `00-index.md`，了解整体结构
2. 阅读 `01-foundation.md`，理解基础抽象
3. 跟着代码走一遍 `src/kimi_cli/soul/kimisoul.py:120-200`

### 第二遍：深入细节

1. 逐章阅读 `02-core-agent.md` 到 `06-protocols.md`
2. 对照源码，理解每个设计决策
3. 尝试回答每章的"设计分析"问题

### 第三遍：面试准备

1. 阅读 `08-interview-qa.md`，尝试回答所有问题
2. 对于不会的问题，回到源码找答案
3. 模拟面试，用语言表达你的理解

### 第四遍：源码阅读

1. 阅读 `07-deep-dive.md`，逐行分析核心代码
2. 自己尝试实现简化版本
3. 思考：如果是你，会如何设计？

---

## 贡献

如果你发现文档有错误或遗漏，欢迎提交 PR 或 Issue。

---

## 许可

本文档基于 kimi-cli 的源码分析，遵循相同的开源协议。
