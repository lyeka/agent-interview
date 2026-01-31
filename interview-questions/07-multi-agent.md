# 七、Multi-Agent 系统

> 15 道题目，覆盖架构设计、协作与竞争、框架对比

---

## 7.1 架构设计

### Q86: Multi-Agent 系统的常见架构模式有哪些？

**答案要点：**

| 模式 | 描述 | 适用场景 |
|------|------|----------|
| **Supervisor** | 中央协调者分配任务 | 任务可分解 |
| **Hierarchical** | 多层级管理 | 大规模系统 |
| **Peer-to-Peer** | Agent 平等协作 | 去中心化 |
| **Pipeline** | 串行处理流程 | 顺序依赖任务 |
| **Blackboard** | 共享工作区 | 知识整合 |

---

### Q87: 如何设计 Agent 间的通信协议？

**答案要点：**

```python
# 消息格式
class AgentMessage:
    sender: str           # 发送者 ID
    receiver: str         # 接收者 ID
    type: str            # request/response/broadcast
    content: dict        # 消息内容
    correlation_id: str  # 关联 ID（用于追踪）

# 通信方式
class AgentCommunication:
    # 1. 直接消息
    async def send(self, to: str, message: AgentMessage):
        await self.message_queue.publish(to, message)

    # 2. 广播
    async def broadcast(self, message: AgentMessage):
        for agent in self.agents:
            await self.send(agent.id, message)

    # 3. 请求-响应
    async def request(self, to: str, message: AgentMessage) -> AgentMessage:
        await self.send(to, message)
        return await self.wait_response(message.correlation_id)
```

---

### Q88: 什么是 Agent Handoff？如何实现平滑的任务交接？

**答案要点：**

- **Handoff 定义**：将任务从一个 Agent 转移到另一个 Agent

- **实现**：
  ```python
  class AgentHandoff:
      def transfer(self, from_agent, to_agent, context):
          # 1. 打包上下文
          handoff_context = {
              "conversation_history": context.messages,
              "current_task": context.task,
              "user_info": context.user,
              "reason": "需要专业知识处理"
          }

          # 2. 通知用户
          notify_user("正在转接到专业客服...")

          # 3. 执行转移
          to_agent.receive_handoff(handoff_context)

          # 4. 确认交接
          return HandoffResult(success=True, new_agent=to_agent.id)
  ```

---

### Q89: 如何处理 Multi-Agent 系统中的死锁和循环依赖？

**答案要点：**

- **死锁场景**：Agent A 等待 B，B 等待 A

- **预防策略**：
  ```python
  class DeadlockPrevention:
      def __init__(self):
          self.dependency_graph = {}
          self.max_wait_time = 30

      def request_resource(self, agent_id, resource_id):
          # 1. 检测循环依赖
          if self.would_create_cycle(agent_id, resource_id):
              raise DeadlockRisk("检测到潜在死锁")

          # 2. 超时机制
          try:
              result = await asyncio.wait_for(
                  self.acquire(resource_id),
                  timeout=self.max_wait_time
              )
          except asyncio.TimeoutError:
              self.release_all(agent_id)
              raise DeadlockTimeout()

      def would_create_cycle(self, agent_id, resource_id):
          # 图遍历检测环
          visited = set()
          return self.dfs_detect_cycle(agent_id, resource_id, visited)
  ```

---

### Q90: Agent 角色分配的策略有哪些？

**答案要点：**

| 策略 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **静态分配** | 预定义角色 | 简单可控 | 不灵活 |
| **动态分配** | 根据任务分配 | 灵活 | 复杂 |
| **自组织** | Agent 自主选择 | 自适应 | 难以预测 |
| **竞标机制** | Agent 竞争任务 | 高效 | 开销大 |

```python
# 动态分配示例
class RoleAllocator:
    def allocate(self, task, available_agents):
        # 计算每个 Agent 的适合度
        scores = []
        for agent in available_agents:
            score = self.compute_fitness(agent, task)
            scores.append((agent, score))

        # 选择最适合的 Agent
        scores.sort(key=lambda x: -x[1])
        return scores[0][0]
```

---

## 7.2 协作与竞争

### Q91: 如何实现 Agent 间的协作（Collaboration）？

**答案要点：**

```python
class CollaborativeAgents:
    async def collaborative_task(self, task):
        # 1. 任务分解
        subtasks = self.decompose(task)

        # 2. 并行执行
        results = await asyncio.gather(*[
            self.assign_and_execute(subtask)
            for subtask in subtasks
        ])

        # 3. 结果整合
        final_result = self.integrate_results(results)

        # 4. 质量检查
        if not self.quality_check(final_result):
            return await self.refine_collaboratively(final_result)

        return final_result
```

---

### Q92: 什么是 Agent Debate？如何提高决策质量？

**答案要点：**

- **Agent Debate 定义**：多个 Agent 对同一问题提出不同观点，通过辩论达成更好的结论

- **实现**：
  ```python
  class AgentDebate:
      def __init__(self, agents, rounds=3):
          self.agents = agents
          self.rounds = rounds

      async def debate(self, question):
          positions = []

          # 1. 初始立场
          for agent in self.agents:
              position = await agent.generate_position(question)
              positions.append(position)

          # 2. 多轮辩论
          for round in range(self.rounds):
              new_positions = []
              for i, agent in enumerate(self.agents):
                  other_positions = positions[:i] + positions[i+1:]
                  refined = await agent.refine_position(
                      positions[i], other_positions
                  )
                  new_positions.append(refined)
              positions = new_positions

          # 3. 综合结论
          return await self.synthesize(positions)
  ```

---

### Q93: 如何处理 Agent 间的冲突和分歧？

**答案要点：**

- **冲突类型**：
  - 资源冲突：争夺同一资源
  - 目标冲突：目标不一致
  - 信息冲突：信息不一致

- **解决策略**：
  ```python
  class ConflictResolver:
      def resolve(self, conflict):
          if conflict.type == "resource":
              return self.priority_based_resolution(conflict)
          elif conflict.type == "goal":
              return self.negotiation(conflict)
          elif conflict.type == "information":
              return self.consensus_building(conflict)

      def negotiation(self, conflict):
          # 1. 识别共同利益
          common_ground = self.find_common_ground(conflict.parties)
          # 2. 提出折中方案
          compromise = self.generate_compromise(conflict, common_ground)
          # 3. 投票决定
          return self.vote(conflict.parties, compromise)
  ```

---

### Q94: 什么是 Consensus Mechanism 在 Multi-Agent 中的应用？

**答案要点：**

- **共识机制**：多个 Agent 就某个决策达成一致

- **常用方法**：
  | 方法 | 描述 | 适用场景 |
  |------|------|----------|
  | 多数投票 | 超过半数同意 | 简单决策 |
  | 加权投票 | 按能力加权 | 专业决策 |
  | 一致同意 | 所有人同意 | 关键决策 |
  | 领导者决定 | 指定决策者 | 快速决策 |

---

### Q95: 如何评估 Multi-Agent 系统的整体性能？

**答案要点：**

- **评估维度**：
  ```python
  class MultiAgentMetrics:
      def evaluate(self, system, test_tasks):
          return {
              # 效果指标
              "task_completion_rate": self.completion_rate(test_tasks),
              "quality_score": self.quality_assessment(test_tasks),

              # 效率指标
              "total_latency": self.measure_latency(test_tasks),
              "token_usage": self.count_tokens(test_tasks),
              "agent_utilization": self.utilization_rate(),

              # 协作指标
              "handoff_success_rate": self.handoff_metrics(),
              "conflict_resolution_time": self.conflict_metrics(),

              # 可靠性指标
              "error_rate": self.error_rate(test_tasks),
              "recovery_success": self.recovery_metrics()
          }
  ```

---

## 7.3 框架对比

### Q96: LangGraph vs CrewAI vs AutoGen 的架构差异和适用场景？

**答案要点：**

| 维度 | LangGraph | CrewAI | AutoGen |
|------|-----------|--------|---------|
| 核心抽象 | 状态图 | 角色+任务 | 对话 |
| 控制流 | 显式图定义 | 任务驱动 | 对话驱动 |
| 灵活性 | 高 | 中 | 中 |
| 学习曲线 | 陡 | 平缓 | 中等 |
| 适用场景 | 复杂工作流 | 团队协作 | 对话式协作 |

---

### Q97: 如何选择 Multi-Agent 框架？评估标准是什么？

**答案要点：**

- **评估标准**：
  1. **功能匹配**：是否支持所需的协作模式
  2. **可扩展性**：能否支持 Agent 数量增长
  3. **可观测性**：调试和监控能力
  4. **社区生态**：文档、示例、支持
  5. **集成能力**：与现有系统的兼容性

---

### Q98: OpenAI Swarm 的设计理念是什么？

**答案要点：**

- **核心理念**：
  - **轻量级**：最小化抽象
  - **Handoff 优先**：Agent 间简单转移
  - **无状态**：状态由调用方管理

- **示例**：
  ```python
  from swarm import Swarm, Agent

  def transfer_to_sales():
      return sales_agent

  triage_agent = Agent(
      name="Triage",
      instructions="判断用户需求，转接到合适的部门",
      functions=[transfer_to_sales, transfer_to_support]
  )

  client = Swarm()
  response = client.run(agent=triage_agent, messages=[...])
  ```

---

### Q99: Microsoft AutoGen 的 Conversation Pattern 有哪些？

**答案要点：**

| Pattern | 描述 | 示例 |
|---------|------|------|
| **Two-Agent** | 两个 Agent 对话 | 用户代理 + 助手 |
| **Group Chat** | 多 Agent 群聊 | 团队讨论 |
| **Hierarchical** | 层级对话 | 管理者 + 执行者 |
| **Sequential** | 顺序对话 | 流水线处理 |

---

### Q100: 如何在 LangGraph 中实现复杂的 Agent 工作流？

**答案要点：**

```python
from langgraph.graph import StateGraph, END

# 定义状态
class AgentState(TypedDict):
    messages: list
    current_agent: str
    task_status: str

# 构建图
workflow = StateGraph(AgentState)

# 添加节点
workflow.add_node("router", router_agent)
workflow.add_node("researcher", research_agent)
workflow.add_node("writer", writer_agent)
workflow.add_node("reviewer", review_agent)

# 添加边
workflow.add_edge("router", "researcher")
workflow.add_conditional_edges(
    "researcher",
    should_continue,
    {"continue": "writer", "end": END}
)
workflow.add_edge("writer", "reviewer")
workflow.add_conditional_edges(
    "reviewer",
    review_decision,
    {"approve": END, "revise": "writer"}
)

# 编译运行
app = workflow.compile()
result = app.invoke(initial_state)
```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
