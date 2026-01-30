# kimi-cli 工程方向知识梳理

## 文档定位

**目标读者**：AI Agent 工程师、LLM 应用开发者、系统架构师
**文档性质**：深度源码分析 + 面试准备材料
**核心价值**：理解 Agent 系统的设计本质，掌握工程实现细节

---

## 学习路径

```
第一阶段：基础抽象（2-3 天）
├── 01-foundation.md
│   ├── Kosong 消息结构（ContentPart 多态）
│   ├── Tool 抽象演进（三层语义）
│   └── PyKAOS 系统抽象
│
第二阶段：Agent 核心（3-5 天）
├── 02-core-agent.md
│   ├── KimiSoul 主循环
│   ├── ReAct 实现
│   └── 状态管理与保护机制
│
第三阶段：Context 与 Memory（2-3 天）
├── 03-context-memory.md
│   ├── SimpleCompaction 压缩算法
│   ├── Checkpoint 机制
│   └── D-Mail 时间旅行
│
第四阶段：工具系统（2-3 天）
├── 04-tools-mcp.md
│   ├── KimiToolset 依赖注入
│   ├── MCP 集成
│   └── 工具权限控制
│
第五阶段：Multi-Agent（2-3 天）
├── 05-multi-agent.md
│   ├── LaborMarket 架构
│   ├── Fixed vs Dynamic 子 Agent
│   └── 上下文隔离机制
│
第六阶段：协议层（1-2 天）
├── 06-protocols.md
│   ├── ACP 协议
│   └── Wire 事件总线
│
第七阶段：源码深度解析（3-5 天）
├── 07-deep-dive.md
│   └── 核心文件逐行分析
│
第八阶段：面试准备（2-3 天）
├── 08-interview-qa.md
│   └── 15 道核心面试题
│
附录：参考材料
└── 09-appendix.md
    ├── 术语表
    ├── 文件索引
    └── 参考资源
```

---

## 核心技术点

| 类别 | 核心技术 | 面试重点 |
|------|----------|----------|
| 架构决策 | 自研 Kosong vs LangChain | Q1, Q2 |
| Agent 核心 | ReAct 循环、状态管理 | Q6, Q7 |
| Context | 压缩策略、时间旅行 | Q7 |
| 工具系统 | 依赖注入、MCP 集成 | Q8, Q9, Q10 |
| Multi-Agent | 分形架构、上下文隔离 | Q11, Q12 |
| 协议层 | ACP、Wire 总线 | Q1 |

---

## 源码探索顺序

```
第一优先级：理解消息抽象
packages/kosong/src/kosong/message.py         # ContentPart 多态设计
packages/kosong/src/kosong/tooling/base.py    # ToolReturnValue 三层语义

第二优先级：理解 Agent 核心
src/kimi_cli/soul/kimisoul.py:120-200        # 主循环（最重要）
src/kimi_cli/soul/context.py                  # 上下文管理
src/kimi_cli/soul/compaction.py:42-117        # 压缩算法

第三优先级：理解工具系统
src/kimi_cli/soul/toolset.py:80-150           # 依赖注入实现
src/kimi_cli/mcp.py:45-120                    # MCP 集成

第四优先级：理解 Multi-Agent
src/kimi_cli/soul/agent.py:200-280            # LaborMarket
src/kimi_cli/tools/multiagent/task.py:52-150  # Task 工具

第五优先级：理解协议层
src/kimi_cli/wire/types.py                    # 事件定义
src/kimi_cli/acp/                             # ACP 协议
```

---

## 文档风格说明

- **无 emoji**：严肃文学风格
- **代码精炼**：只展示关键逻辑 + 行号 + "为什么"
- **理论实践平衡**：理论 40% + 实践 60%
- **深度分析**：每个设计点都回答"为什么"和"trade-off"

---

## 版本信息

- **分析版本**：kimi-cli (master branch)
- **生成时间**：2026-01-31
- **分析工具**：ai-agent-codebase-study skill v2
