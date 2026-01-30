# 第五层：Multi-Agent 协作

> 本章介绍 kimi-cli 的 Multi-Agent 架构，包括 LaborMarket、子 Agent 创建、上下文隔离。

---

## 5.1 LaborMarket 架构

### 设计意图

单一 Agent 能力有限，如何组织多个 Agent 协作完成任务？

**核心问题**：
1. **职责划分**：如何决定由哪个 Agent 处理任务？
2. **上下文共享**：子 Agent 能否访问父 Agent 的历史？
3. **结果合并**：多个子 Agent 的结果如何合并？

### 核心实现

**源码位置**: `src/kimi_cli/soul/agent.py:200-280`

```python
class LaborMarket:
    """
    Agent 劳动力市场，负责任务分配和子 Agent 管理
    """

    def __init__(self, runtime: Runtime):
        self._runtime = runtime
        self._fixed_agents: dict[str, Agent] = {}
        self._dynamic_agents: list[Agent] = []

    def register_fixed(self, name: str, agent: Agent) -> None:
        """注册固定的子 Agent"""
        self._fixed_agents[name] = agent

    async def hire(self, task: str, context: Context) -> Agent:
        """
        根据任务雇佣合适的 Agent

        Args:
            task: 任务描述
            context: 当前上下文

        Returns:
            选择的 Agent
        """
        # 1. 尝试匹配固定的子 Agent
        for name, agent in self._fixed_agents.items():
            if agent.can_handle(task):
                return agent

        # 2. 如果没有匹配，动态创建子 Agent
        dynamic_agent = await self._create_dynamic_agent(task, context)
        self._dynamic_agents.append(dynamic_agent)
        return dynamic_agent
```

### Fixed vs Dynamic 子 Agent

**源码位置**: `src/kimi_cli/agents/`

```python
# Fixed Agent: 预定义的专用 Agent
class CodeReviewAgent(Agent):
    def can_handle(self, task: str) -> bool:
        return "代码审查" in task or "review" in task

    async def run(self, task: str, context: Context) -> str:
        # 专门的代码审查逻辑
        ...

# Dynamic Agent: 根据任务动态创建
async def _create_dynamic_agent(task: str, context: Context) -> Agent:
    """根据任务描述动态创建 Agent"""
    # 使用 LLM 分析任务，生成 Agent 规格
    spec = await analyze_task(task)

    # 创建 Agent
    agent = Agent(
        name=spec.name,
        system_prompt=spec.system_prompt,
        tools=spec.tools,
    )
    return agent
```

### 设计分析

**Q: Fixed vs Dynamic 的选择标准？**

A: 使用决策树：

```
任务是否可预见？
├─ 是 → 使用 Fixed Agent
│   └─ 原因：性能更好，可复用，可优化
└─ 否 → 使用 Dynamic Agent
    └─ 原因：灵活性高，适应性强
```

**示例**：

| 任务类型 | Agent 类型 | 原因 |
|----------|-----------|------|
| 代码审查 | Fixed | 任务明确，可预定义审查规则 |
| 研究"量子计算" | Dynamic | 任务不确定，需要动态生成研究策略 |
| 部署应用 | Fixed | 流程固定，可预定义部署步骤 |
| 解释"这段代码" | Dynamic | 代码内容未知，需要动态分析 |

**Q: 与 Microsoft AutoGen 的对话驱动多 Agent 相比，优劣？**

A: 对比：

| 维度 | kimi-cli 分形架构 | AutoGen 对话模型 |
|------|------------------|------------------|
| 协作模式 | 层级（父 Agent 分配任务） | 对话（Agent 互相协商） |
| 优势 | 清晰的责任边界，易于调试 | 灵活性高，自组织 |
| 劣势 | 父 Agent 成为瓶颈 | 可能陷入无限对话 |
| 适用场景 | 任务明确的场景 | 探索性场景 |

**示例对比**：

**kimi-cli（分形）**：
```
用户："帮我审查代码"
父 Agent：雇佣 CodeReviewAgent
CodeReviewAgent：执行审查，返回结果
父 Agent：合并结果，返回给用户
```

**AutoGen（对话）**：
```
UserAgent："帮我审查代码"
CoderAgent："我需要先看到代码"
UserAgent："代码在 /path/to/file"
CoderAgent："让我读取..."
SecurityAgent："等等，我先检查安全问题"
CoderAgent："好的，你检查完我再看"
...（可能持续很多轮）
```

---

## 5.2 上下文隔离机制

### 设计意图

子 Agent 是否应该访问父 Agent 的历史？

**核心问题**：
1. **隐私**：子 Agent 不应该看到无关的历史
2. **上下文传递**：子 Agent 需要足够的上下文才能完成任务
3. **安全性**：防止子 Agent 滥用父 Agent 的权限

### 核心实现

**源码位置**: `src/kimi_cli/tools/multiagent/task.py:52-150`

```python
class TaskTool(Tool):
    """
    将任务委托给子 Agent 的工具
    """

    async def execute(self, task: str, context: Context) -> ToolReturnValue:
        # 1. 创建隔离的上下文
        sub_context = Context()

        # 2. 传递必要的上下文（而非全部历史）
        relevant_messages = await self._extract_relevant_context(
            task=task,
            parent_context=context,
        )
        sub_context.add_messages(relevant_messages)

        # 3. 创建子 Agent
        agent = await self._labor_market.hire(task, sub_context)

        # 4. 运行子 Agent
        result = await agent.run(task, sub_context)

        # 5. 将结果添加到父 Agent 的上下文
        context.add_message(
            Message(role="assistant", content=f"子 Agent 完成：{result}")
        )

        return ToolReturnValue(
            output=result,
            message=f"委托任务给子 Agent：{task}",
        )
```

### 设计分析

**Q: 子 Agent 能否访问父 Agent 的历史？**

A: 不能直接访问，但可以传递"相关上下文"：

```python
async def _extract_relevant_context(
    self,
    task: str,
    parent_context: Context,
) -> list[Message]:
    """
    从父 Agent 的历史中提取与任务相关的上下文
    """
    # 策略 1: 只传递最近 N 条消息
    return parent_context.messages[-10:]

    # 策略 2: 使用向量检索找到相关历史
    # relevant = vector_store.search(task)
    # return [parent_context.messages[i] for i in relevant]

    # 策略 3: 用 LLM 判断哪些历史相关
    # relevant = await llm.ask(f"这些历史中哪些与 '{task}' 相关？")
    # return relevant
```

**Q: 子 Agent 的工具权限是继承还是独立的？**

A: 独立的。每个 Agent 有自己的工具集：

```python
# 父 Agent：有所有工具
parent_agent = Agent(tools=[file_tool, web_tool, database_tool])

# 子 Agent：只有部分工具
sub_agent = Agent(tools=[file_tool])  # 只能操作文件，不能访问数据库
```

安全边界：
1. **最小权限原则**：子 Agent 只拥有完成任务必需的工具
2. **审批机制**：子 Agent 使用危险工具时需要父 Agent 审批
3. **审计日志**：记录子 Agent 的所有操作

**Q: 如果要实现子 Agent 之间的直接通信（非通过父 Agent），需要什么机制？**

A: 需要实现消息总线：

```python
class AgentMessageBus:
    """Agent 间通信的消息总线"""

    def __init__(self):
        self._channels: dict[str, list[Agent]] = {}

    def subscribe(self, agent: Agent, channel: str) -> None:
        """Agent 订阅频道"""
        if channel not in self._channels:
            self._channels[channel] = []
        self._channels[channel].append(agent)

    async def publish(self, channel: str, message: Message) -> None:
        """发布消息到频道"""
        agents = self._channels.get(channel, [])
        for agent in agents:
            await agent.receive(message)
```

**场景设计：代码审查 Agent 系统**

```python
# 三个子 Agent
analyzer_agent = Agent(name="Analyzer", role="分析代码")
tester_agent = Agent(name="Tester", role="运行测试")
reporter_agent = Agent(name="Reporter", role="生成报告")

# 协作流程
analyzer_agent.publish("code.analyzed", result)
tester_agent.subscribe("code.analyzed")  # 收到分析结果，开始测试
tester_agent.publish("tests.completed", results)
reporter_agent.subscribe("tests.completed")  # 收到测试结果，生成报告
```

---

## 5.3 场景设计：GitHub Issue 自动修复

### 需求

一家开源项目希望接入 AI Agent，实现"自动修复 Issue"的功能：

1. 监听 GitHub Issues
2. 分析 Issue 描述和相关代码
3. 生成修复代码
4. 创建 Pull Request
5. 如果测试失败，自动迭代

### Agent 架构设计

```
┌─────────────────────────────────────────────────────┐
│                    OrchestratorAgent                 │
│  负责任务分配、进度跟踪、最终决策                     │
└─────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ AnalyzerAgent│    │ CoderAgent   │    │ TesterAgent  │
│ 分析 Issue   │───→│ 生成修复代码 │───→│ 运行测试     │
└──────────────┘    └──────────────┘    └──────────────┘
         │                                      │
         └──────────┬───────────────────────────┘
                    ▼
            ┌──────────────┐
            │ReporterAgent │
            │ 生成 PR 描述 │
            └──────────────┘
```

### 工具选择

| Agent | 工具 |
|-------|------|
| OrchestratorAgent | github.list_issues, github.create_pr |
| AnalyzerAgent | github.read_issue, file.read, code.analyze |
| CoderAgent | file.write, code.generate |
| TesterAgent | test.run, git.create_branch |
| ReporterAgent | github.create_pr, markdown.format |

### 安全边界

1. **沙箱执行**：所有代码修改在临时分支进行
2. **人工审批**：创建 PR 前需要人工确认
3. **测试门禁**：必须通过所有测试才能创建 PR
4. **回滚机制**：如果 PR 被拒绝，自动回滚分支

### 评估体系

如何判断 Agent 生成的 PR 是"好"的？

1. **测试通过率**：所有测试必须通过
2. **代码审查**：自动运行 linter 和 type checker
3. **人工反馈**：PR 被合并的频率
4. **修复时间**：从 Issue 创建到 PR 合并的时间

### 失败恢复

如果 Agent 生成的代码导致测试失败：

1. **分析失败原因**：用 LLM 分析测试输出
2. **迭代修复**：CoderAgent 重新生成代码
3. **回滚策略**：连续 3 次失败后放弃，通知人工介入

---

## 本章小结

Multi-Agent 的设计：

1. **LaborMarket 架构**：支持 Fixed 和 Dynamic 两种子 Agent
2. **上下文隔离**：子 Agent 只接收必要的上下文，保证隐私和安全
3. **协作模式**：层级分配（kimi-cli）vs 对话驱动（AutoGen）

相比复杂的 P2P 协作，kimi-cli 选择了清晰的层级架构。
