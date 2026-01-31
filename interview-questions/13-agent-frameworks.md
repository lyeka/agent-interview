# 十三、Agent 框架深度

> 10 道题目，覆盖 LangChain/LangGraph、其他框架

---

## 13.1 LangChain/LangGraph

### Q176: LangChain 的核心抽象有哪些？

**答案要点：**

| 抽象 | 描述 |
|------|------|
| **Model** | LLM 封装 |
| **Prompt** | 提示模板 |
| **Chain** | 组件链接 |
| **Agent** | 自主决策 |
| **Tool** | 外部能力 |
| **Memory** | 状态管理 |
| **Retriever** | 检索接口 |

---

### Q177: LCEL (LangChain Expression Language) 的设计理念？

**答案要点：**

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

# 声明式链式调用
chain = (
    ChatPromptTemplate.from_template("翻译成英文：{text}")
    | ChatOpenAI()
    | StrOutputParser()
)

# 支持流式、批量、异步
result = chain.invoke({"text": "你好"})
async for chunk in chain.astream({"text": "你好"}):
    print(chunk)
```

---

### Q178: LangGraph 的 State Machine 如何工作？

**答案要点：**

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class State(TypedDict):
    messages: list
    next_step: str

# 定义图
graph = StateGraph(State)

# 添加节点
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

# 添加边
graph.add_edge("agent", "tools")
graph.add_conditional_edges(
    "tools",
    should_continue,
    {"continue": "agent", "end": END}
)

# 编译
app = graph.compile()
```

---

### Q179: 如何在 LangGraph 中实现 Human-in-the-Loop？

**答案要点：**

```python
from langgraph.checkpoint.memory import MemorySaver

# 添加检查点
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)

# 执行到需要人工确认的节点
config = {"configurable": {"thread_id": "1"}}
result = app.invoke(input, config)

# 人工确认后继续
app.invoke(None, config)  # 从检查点恢复
```

---

### Q180: LangSmith 的核心功能和使用场景？

**答案要点：**

| 功能 | 描述 |
|------|------|
| **Tracing** | 执行链路追踪 |
| **Evaluation** | 自动化评估 |
| **Datasets** | 测试数据管理 |
| **Monitoring** | 生产监控 |
| **Playground** | 交互式调试 |

---

## 13.2 其他框架

### Q181: Semantic Kernel 的架构设计？

**答案要点：**

```csharp
// Microsoft Semantic Kernel
var kernel = Kernel.CreateBuilder()
    .AddOpenAIChatCompletion("gpt-4", apiKey)
    .Build();

// 定义插件
public class EmailPlugin
{
    [KernelFunction]
    public async Task SendEmail(string to, string subject, string body) { }
}

kernel.ImportPluginFromType<EmailPlugin>();
```

---

### Q182: Haystack 的 Pipeline 设计理念？

**答案要点：**

```python
from haystack import Pipeline
from haystack.components.retrievers import InMemoryBM25Retriever
from haystack.components.generators import OpenAIGenerator

# 构建 Pipeline
pipe = Pipeline()
pipe.add_component("retriever", InMemoryBM25Retriever())
pipe.add_component("generator", OpenAIGenerator())
pipe.connect("retriever", "generator")

# 执行
result = pipe.run({"retriever": {"query": "问题"}})
```

---

### Q183: LlamaIndex 的核心组件有哪些？

**答案要点：**

| 组件 | 描述 |
|------|------|
| **Document** | 文档抽象 |
| **Index** | 索引结构 |
| **Retriever** | 检索器 |
| **Query Engine** | 查询引擎 |
| **Agent** | 智能代理 |

---

### Q184: Dify 的 Workflow 设计？

**答案要点：**

- **特点**：
  1. 可视化编排
  2. 低代码/无代码
  3. 内置 RAG
  4. 多模型支持

---

### Q185: 如何评估和选择 Agent 框架？

**答案要点：**

| 评估维度 | 考量点 |
|----------|--------|
| 功能 | 是否满足需求 |
| 性能 | 延迟、吞吐 |
| 可扩展性 | 自定义能力 |
| 社区 | 文档、支持 |
| 成熟度 | 稳定性 |

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
