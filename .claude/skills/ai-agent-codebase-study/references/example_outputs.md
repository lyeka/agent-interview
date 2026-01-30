# AI Agent Project Study - Example Outputs

> 完整示例，作为生成文档的质量标杆

---

## 示例 1: Single Agent 应用

假设项目：**customer-service-agent**（客服 Agent）

```markdown
# customer-service-agent AI Agent 工程方向知识梳理

> 面试准备材料 - 假设我是该项目的核心开发者

## 文档定位

- **目标读者**：想要深入了解该客服 Agent 的开发者/面试者
- **文档性质**：项目分析 + 面试准备材料

---

## 一、项目概述

### 1.1 核心定义

**项目定位**：Single Agent 应用 - 智能客服系统

**核心功能**：
- 自动回答用户常见问题
- 处理退换货请求
- 升级复杂问题给人工客服

**与同类项目对比**：

| 维度 | 本项目 | LangChain Customer Service | Custom GPT |
|------|--------|---------------------------|------------|
| 架构模式 | ReAct + Tool Calling | ReAct | 纯 LLM |
| 知识库 | RAG 向量检索 | RAG | 文件上传 |
| 工具集成 | 订单系统、CRM | 有限 | 无 |
| 成本控制 | Token 缓存 + 小模型 | 依赖 GPT-4 | 高成本 |

### 1.2 Agent 架构设计

**架构模式**：ReAct（推理-行动循环）

**架构图**：
```
┌─────────────────────────────────────────────────────────────┐
│                  customer-service-agent                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   用户问题                                                     │
│      ↓                                                       │
│   ┌─────────────┐                                           │
│   │ Agent Core  │ ← ReAct Loop (Thought → Action → Obs)    │
│   └─────────────┘                                           │
│      ↓           ↓           ↓                              │
│   System Prompt  Tools    Memory                            │
│      ↓           ↓           ↓                              │
│   [LLM: GPT-4o-mini]                                   │
│      ↓                                                       │
│   响应 + 下一步行动                                           │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**技术栈分层**：

| 层级 | 技术选型 | 说明 |
|------|----------|------|
| LLM 层 | GPT-4o-mini (主要) + GPT-4o (复杂) | 成本优化策略 |
| 框架层 | 自研 ReAct 框架 | 轻量级、可控 |
| 数据层 | ChromaDB (向量) + PostgreSQL (订单) | RAG + 业务数据 |
| 工具层 | 订单查询、退款处理、人工升级 | Function Calling |

---

## 二、Context 管理

### 2.1 Context Window 策略

- **窗口限制**：GPT-4o-mini 128K tokens
- **压缩策略**：
  - 对话超过 20 轮触发 Compaction
  - 保留最近 5 轮完整对话
  - 历史对话总结为 3-5 个关键信息点

### 2.2 Context Engineering

**上下文组件结构**：
1. System prompt (~200 tokens)
2. 用户最近对话 (~1000 tokens)
3. RAG 检索的 FAQ (~500 tokens)
4. 订单信息（如有，~300 tokens）

**信息策展**：
- FAQ 检索使用语义相似度，取 Top-3
- 订单信息按需加载（检测到订单号时）

**Context Rot 对抗**：
- 关键信息持久化到 Memory（用户偏好、历史问题）

---

## 三、Prompt 系统

### 3.1 Prompt 架构

**System prompt** (~200 tokens)：
```
你是智能客服助手，职责是：
1. 回答常见问题（优先使用 RAG 检索的知识库）
2. 处理退换货（调用 refund_tool）
3. 查询订单（调用 order_query_tool）
4. 复杂问题升级人工（escalate_to_human）

规则：
- 友好、专业、简洁
- 不确定时说不知道，不要编造
- 退款需确认订单状态
```

**Few-shot 模板**：
```
示例 1:
用户：我想退货
Agent：[查询订单] 订单 #12345 已发货，可以退货。请确认退货原因。

示例 2:
用户：物流太慢了
Agent：[查询物流] 您的订单 #12345 当前在运输中，预计 2 天送达。
```

**Chain-of-Thought**：
- 复杂问题（如退换货）启用 CoT
- 简单问题（如 FAQ）直接回答

### 3.2 Prompt 优化

**幻觉抑制**：
- 明确要求"不确定时说不知道"
- FAQ 检索结果作为参考来源

**Token 优化**：
- System prompt 控制在 200 tokens
- Few-shot 仅在复杂场景加载

---

## 四、Tool & MCP 集成

### 5.1 Tool 定义

**Function Calling 实现**：
```python
tools = [
    {
        "name": "order_query",
        "description": "查询订单状态和物流信息",
        "parameters": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string", "description": "订单号"}
            },
            "required": ["order_id"]
        }
    },
    {
        "name": "refund_request",
        "description": "处理退款请求",
        "parameters": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string"},
                "reason": {"type": "string", "enum": ["quality", "wrong_item", "other"]}
            },
            "required": ["order_id", "reason"]
        }
    },
    {
        "name": "escalate_to_human",
        "description": "升级给人工客服",
        "parameters": {
            "type": "object",
            "properties": {
                "reason": {"type": "string", "description": "升级原因"}
            },
            "required": ["reason"]
        }
    }
]
```

**Tool 描述规范**：
- name：动词_名词格式（order_query）
- description：清晰说明功能 + 使用场景
- parameters：类型 + 描述 + required 标记

### 5.2 MCP 支持

本项目未使用 MCP，直接 Function Calling。

---

## 五、生产实践

### 7.1 成本优化

**Token 优化**：
- 简单问答用 GPT-4o-mini（便宜 10x）
- 复杂决策（退款判断）用 GPT-4o

**模型选择逻辑**：
```python
def choose_model(query):
    if is_simple_faq(query):
        return "gpt-4o-mini"
    elif requires_refund_or_escalation(query):
        return "gpt-4o"
    else:
        return "gpt-4o-mini"
```

**缓存方案**：
- FAQ 问答缓存 24 小时
- 相似问题（embedding 相似度 >0.9）命中缓存

### 7.2 性能优化

**并行调用**：不适用（Single Agent）

**流式响应**：启用，提升用户体验

### 7.3 可观测性

**追踪工具**：自研简易追踪
```python
log = {
    "query": user_input,
    "model": model_used,
    "tools_called": [...],
    "tokens": input_tokens + output_tokens,
    "latency_ms": ...
}
```

**监控指标**：
- 平均响应时间
- 人工升级率（目标 <20%）
- 用户满意度

### 7.4 安全防护

**Guardrails**：
- 禁止输出敏感信息（其他用户订单）
- 退款需二次确认

**Prompt 注入防护**：
- System prompt 与用户输入分离
- 检测恶意模式（如"忽略之前指令"）

---

## 八、面试要点

### 8.1 概念理解

该项目体现的 AI Agent 核心概念：
- **ReAct 模式**：推理-行动交替循环
- **Function Calling**：工具定义与参数设计
- **RAG vs Memory**：FAQ 是 RAG（静态知识），用户偏好是 Memory（动态学习）

与传统方案的本质区别：
| 传统客服机器人 | 本 Agent |
|----------------|----------|
| 关键词匹配 | 语义理解 + 推理 |
| 固定流程 | 动态工具调用 |
| 无状态 | 有 Memory |

### 8.2 源码路径

| 功能 | 文件路径 | 关键行号 | 说明 |
|------|----------|----------|------|
| ReAct 循环 | src/agent/react_loop.py | 45-120 | Thought-Action-Obs 实现 |
| Tool 调用 | src/tools/executor.py | 23-67 | Function Calling 包装 |
| Prompt 模板 | src/prompts/system.py | 10-45 | System prompt + Few-shot |
| RAG 检索 | src/rag/retriever.py | 78-134 | ChromaDB 语义检索 |
| 模型路由 | src/llm/router.py | 15-42 | 成本优化策略 |

### 8.3 设计亮点

1. **动态模型选择**：简单任务用小模型，节省 70% 成本
2. **Compaction 策略**：对话过长时保留关键信息，避免 Context Rot
3. **Tool 描述设计**：清晰的 description 使 LLM 工具调用准确率达 92%

---

## 九、学习路径

### 推荐阅读顺序

1. **第一步**：`src/agent/react_loop.py` - 理解 ReAct 核心循环
2. **第二步**：`src/prompts/system.py` - 理解 Prompt 设计
3. **第三步**：`src/tools/executor.py` - 理解 Function Calling 实现
4. **第四步**：`src/rag/retriever.py` - 理解 RAG 集成
5. **第五步**：`src/llm/router.py` - 理解成本优化策略

### 重点文件列表（按优先级）

| 优先级 | 文件路径 | 理由 |
|--------|----------|------|
| P0 | src/agent/react_loop.py | Agent 核心逻辑，面试必问 |
| P1 | src/prompts/system.py | Prompt 工程实践 |
| P2 | src/tools/executor.py | Function Calling 实现 |
| P3 | src/llm/router.py | 成本优化策略 |
| P4 | src/rag/retriever.py | RAG 检索 |

---

## 十、参考资源

### 官方文档
- [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling) - 工具定义规范
- [ChromaDB Docs](https://docs.trychroma.com) - 向量数据库使用

### 技术博客
- [Anthropic - Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) - Agent 设计原则

### 关键论文
- [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629) - ReAct 模式原论文
```

---

## 示例 2: Multi-Agent 框架

假设项目：**crewai**（Multi-Agent 框架）

```markdown
# crewai AI Agent 工程方向知识梳理

> 面试准备材料 - 假设我是 CrewAI 的核心贡献者

## 文档定位

- **目标读者**：想要深入了解 CrewAI 框架的开发者/面试者
- **文档性质**：框架分析 + 面试准备材料

---

## 一、项目概述

### 1.1 核心定义

**项目定位**：Multi-Agent 框架 - 角色驱动的 Agent 协作框架

**核心功能**：
- 定义 Agent 角色、目标、背景故事
- Agent 间任务分配与协作
- 顺序/并行执行模式

**与同类框架对比**：

| 维度 | CrewAI | LangGraph | AutoGen |
|------|--------|-----------|---------|
| 编程范式 | 角色驱动（声明式） | 图驱动（DAG） | 对话驱动（命令式） |
| 学习曲线 | 低 | 中高 | 中 |
| 灵活性 | 中 | 高 | 高 |
| 适用场景 | 快速原型 | 复杂工作流 | 自定义实现 |

### 1.2 Agent 架构设计

**架构模式**：Hierarchical + Team-Based（监督器 + 角色团队）

**架构图**：
```
┌─────────────────────────────────────────────────────────────┐
│                        CrewAI 框架                           │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│   Crew (团队)                                                 │
│      ↓                                                       │
│   ┌─────────────┐                                           │
│   │   Tasks     │ ← 任务列表（顺序依赖/并行）                 │
│   └─────────────┘                                           │
│      ↓                                                       │
│   ┌─────────────────────────────────────┐                   │
│   │            Agents (角色团队)          │                   │
│   │  ┌──────┐ ┌──────┐ ┌──────┐        │                   │
│   │  │Research│ │Writer│ │Editor│        │                   │
│   │  │Agent  │ │Agent │ │Agent │        │                   │
│   │  └──────┘ └──────┘ └──────┘        │                   │
│   └─────────────────────────────────────┘                   │
│      ↓                    ↓                                 │
│   LLM (OpenAI/Anthropic)  Tools (RAG/搜索)                   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**技术栈分层**：

| 层级 | 技术选型 | 说明 |
|------|----------|------|
| LLM 层 | OpenAI / Anthropic / Local | 多模型支持 |
| 编排层 | Crew（类）+ Agent（类）+ Task（类） | Python OOP |
| 工具层 | SerperDev / RAG | 自定义工具 |
| 通信层 | 顺序执行 / 共享状态 | 非消息传递 |

---

## 六、Multi-Agent

### 6.1 协作模式

**CrewAI 采用 Sequential Pipeline 模式**：
- Tasks 按顺序执行
- 每个 Task 可指定不同 Agent
- 前一个 Task 的输出是下一个的输入

**代码示例**：
```python
crew = Crew(
    agents=[researcher, writer, editor],
    tasks=[research_task, write_task, edit_task],
    process=Process.sequential  # 顺序执行
)
```

**通信协议**：
- 非直接消息传递
- 通过 Task 上下文共享状态
- Task.output → Task.context

### 6.2 调度机制

**顺序调度**：
```python
def sequential_process(crew):
    context = {}
    for task in crew.tasks:
        agent = crew.assign_agent(task)
        result = agent.execute(task, context)
        context[task.id] = result  # 传递给下一个 Task
    return context
```

**并行调度**（v0.1+ 支持）：
```python
crew = Crew(
    agents=[...],
    tasks=[task1, task2, task3],
    process=Process.hierarchical  # Hierarchical 模式
)
```

---

## 五、Tool & MCP 集成

### 5.1 Tool 定义

**CrewAI 的 @tool 装饰器**：
```python
from crewai_tools import tool

@tool("Search the internet")
def search_internet(query: str) -> str:
    """搜索互联网，返回结果摘要"""
    return SerperDev().run(query)
```

**Tool 描述规范**：
- `@tool("描述")` - 清晰的功能说明
- 函数 docstring - 详细用法
- 类型提示 - 参数约束

### 5.2 MCP 支持

CrewAI v0.1+ 支持 MCP Server：
```python
from crewai import MCPServer

server = MCPServer("my-mcp-server")
server.add_tool(search_tool)
```

---

## 七、生产实践

### 7.1 成本优化

**Token 优化**：
- 共享上下文，避免重复传递
- Task 间传递摘要而非完整历史

### 7.2 可观测性

**CrewKickoffLogs**：
```python
result = crew.kickoff()
print(result.logs["token_usage"])  # 每个 Agent 的 Token 消耗
```

---

## 八、面试要点

### 8.1 概念理解

CrewAI 体现的 Multi-Agent 核心概念：
- **角色驱动**：Agent 有 role、goal、backstory
- **任务协作**：Tasks 按 process 组织
- **顺序执行**：默认 Sequential Pipeline

与 AutoGen 的区别：
| CrewAI | AutoGen |
|--------|---------|
| 角色驱动（声明式） | 对话驱动（命令式） |
| Tasks 列表 | Agent 间对话 |
| 顺序执行为主 | 支持复杂对话模式 |

### 8.2 源码路径

| 功能 | 文件路径 | 关键行号 | 说明 |
|------|----------|----------|------|
| Crew 类 | crewai/crew.py | 45-180 | 团队编排核心 |
| Agent 类 | crewai/agent.py | 30-120 | Agent 角色定义 |
| Task 执行 | crewai/task.py | 60-150 | Task 执行逻辑 |
| Sequential Process | crewai/process/sequential.py | 20-90 | 顺序调度 |

### 8.3 设计亮点

1. **角色驱动设计**：降低入门门槛，非程序员也能定义 Agent
2. **声明式 Tasks**：清晰的任务依赖，易于理解
3. **共享上下文**：避免复杂的消息传递协议

---

## 九、学习路径

### 推荐阅读顺序

1. **第一步**：`crewai/agent.py` - 理解 Agent 角色定义
2. **第二步**：`crewai/task.py` - 理解 Task 执行
3. **第三步**：`crewai/crew.py` - 理解团队编排
4. **第四步**：`crewai/process/sequential.py` - 理解调度机制

### 重点文件列表（按优先级）

| 优先级 | 文件路径 | 理由 |
|--------|----------|------|
| P0 | crewai/agent.py | Agent 核心类 |
| P1 | crewai/crew.py | 团队编排逻辑 |
| P2 | crewai/task.py | Task 执行流程 |
| P3 | crewai/process/sequential.py | 调度算法 |

---

## 十、参考资源

### 官方文档
- [CrewAI Docs](https://docs.crewai.com) - 官方文档

### 技术博客
- [CrewAI vs LangGraph vs AutoGen](https://blog.crewai.com/comparison) - 框架对比

### 关键论文
- [Multi-Agent Systems: A Survey](https://arxiv.org/abs/2308.10832) - Multi-Agent 综述
```
