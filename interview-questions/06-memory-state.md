# 六、Memory 与状态管理

> 10 道题目，覆盖 Memory 类型、状态持久化

---

## 6.1 Memory 类型

### Q76: Agent Memory 的分类：Short-term vs Long-term vs Episodic vs Semantic

**答案要点：**

- **Short-term Memory（短期记忆）**：
  - 当前对话上下文
  - 临时工作状态
  - 生命周期：单次会话

- **Long-term Memory（长期记忆）**：
  - 跨会话持久化
  - 用户偏好、历史交互
  - 生命周期：永久

- **Episodic Memory（情景记忆）**：
  - 具体事件和经历
  - "上次用户问过 X"
  - 时间序列组织

- **Semantic Memory（语义记忆）**：
  - 事实和知识
  - "用户喜欢 Python"
  - 结构化存储

---

### Q77: 如何实现 Conversation Memory？常用策略有哪些？

**答案要点：**

- **Buffer Memory**：保存完整对话
  ```python
  class BufferMemory:
      def __init__(self):
          self.messages = []

      def add(self, message):
          self.messages.append(message)

      def get_context(self):
          return self.messages
  ```

- **Window Memory**：保留最近 N 轮
  ```python
  class WindowMemory:
      def __init__(self, k=10):
          self.k = k
          self.messages = []

      def get_context(self):
          return self.messages[-self.k*2:]  # 每轮2条消息
  ```

- **Summary Memory**：摘要历史对话
  ```python
  class SummaryMemory:
      def __init__(self):
          self.summary = ""
          self.recent = []

      def update(self, messages):
          if len(self.recent) > 10:
              self.summary = llm.summarize(self.summary + str(self.recent[:-5]))
              self.recent = self.recent[-5:]
  ```

---

### Q78: 什么是 Memory Summarization？如何实现？

**答案要点：**

- **定义**：将长对话历史压缩为摘要，保留关键信息

- **实现**：
  ```python
  def summarize_conversation(messages, existing_summary=""):
      prompt = f"""
      现有摘要：{existing_summary}

      新对话：
      {format_messages(messages)}

      请更新摘要，保留：
      1. 用户的主要需求
      2. 已完成的任务
      3. 重要的决策和结论
      4. 待处理的事项
      """
      return llm.generate(prompt)
  ```

- **增量更新**：
  - 每 N 轮更新一次摘要
  - 保留最近对话 + 摘要

---

### Q79: 如何设计 Agent 的 Working Memory？

**答案要点：**

- **Working Memory 定义**：Agent 当前任务的临时工作区

- **设计要素**：
  ```python
  class WorkingMemory:
      def __init__(self):
          self.current_goal = None      # 当前目标
          self.sub_tasks = []           # 子任务列表
          self.completed_steps = []     # 已完成步骤
          self.intermediate_results = {} # 中间结果
          self.context_variables = {}   # 上下文变量

      def update_progress(self, step, result):
          self.completed_steps.append(step)
          self.intermediate_results[step] = result

      def get_state_summary(self):
          return f"""
          目标：{self.current_goal}
          进度：{len(self.completed_steps)}/{len(self.sub_tasks)}
          最近结果：{self.intermediate_results}
          """
  ```

---

### Q80: 什么是 Reflection Memory？如何帮助 Agent 学习？

**答案要点：**

- **Reflection Memory 定义**：存储 Agent 的反思和学习经验

- **实现**：
  ```python
  class ReflectionMemory:
      def __init__(self):
          self.reflections = []

      def add_reflection(self, task, outcome, lesson):
          self.reflections.append({
              "task": task,
              "outcome": outcome,
              "lesson": lesson,
              "timestamp": datetime.now()
          })

      def get_relevant_lessons(self, current_task):
          # 检索相关经验
          relevant = []
          for r in self.reflections:
              if is_similar(r["task"], current_task):
                  relevant.append(r["lesson"])
          return relevant
  ```

- **应用**：
  - 避免重复错误
  - 优化决策策略
  - 个性化适应

---

## 6.2 状态持久化

### Q81: 如何实现 Agent 状态的 Checkpoint 和恢复？

**答案要点：**

- **Checkpoint 设计**：
  ```python
  class AgentCheckpoint:
      def save(self, agent_state, checkpoint_id):
          data = {
              "memory": agent_state.memory.serialize(),
              "task_state": agent_state.task_state,
              "tool_results": agent_state.tool_results,
              "timestamp": datetime.now().isoformat()
          }
          storage.save(checkpoint_id, json.dumps(data))

      def restore(self, checkpoint_id):
          data = json.loads(storage.load(checkpoint_id))
          agent_state = AgentState()
          agent_state.memory.deserialize(data["memory"])
          agent_state.task_state = data["task_state"]
          return agent_state
  ```

- **触发时机**：
  - 定期自动保存
  - 关键步骤完成后
  - 用户请求暂停时

---

### Q82: 长时间运行的 Agent 如何处理状态持久化？

**答案要点：**

- **挑战**：
  - 内存增长
  - 进程中断风险
  - 状态一致性

- **解决方案**：
  ```python
  class DurableAgent:
      def __init__(self, state_store):
          self.state_store = state_store

      async def run_step(self, step_id):
          # 1. 加载状态
          state = await self.state_store.load()

          # 2. 执行步骤
          result = await self.execute(state)

          # 3. 持久化状态
          state.update(result)
          await self.state_store.save(state)

          # 4. 返回结果
          return result
  ```

- **存储选择**：
  - Redis：快速，适合临时状态
  - PostgreSQL：持久，适合重要状态
  - S3：大文件，适合检查点

---

### Q83: 什么是 Durable Execution？如何实现？

**答案要点：**

- **Durable Execution 定义**：即使进程崩溃也能恢复执行的能力

- **实现原理**：
  ```python
  class DurableWorkflow:
      def __init__(self, workflow_id):
          self.workflow_id = workflow_id
          self.event_log = EventLog(workflow_id)

      async def execute_step(self, step_name, func, *args):
          # 检查是否已执行
          cached = await self.event_log.get_result(step_name)
          if cached:
              return cached

          # 执行并记录
          result = await func(*args)
          await self.event_log.record(step_name, result)
          return result
  ```

- **框架支持**：
  - Temporal
  - Inngest
  - Durable Functions (Azure)

---

### Q84: 如何设计 Agent 的 Session 管理？

**答案要点：**

- **Session 生命周期**：
  ```
  创建 → 活跃 → 空闲 → 过期/关闭
  ```

- **设计实现**：
  ```python
  class SessionManager:
      def __init__(self, ttl=3600):
          self.sessions = {}
          self.ttl = ttl

      def create_session(self, user_id):
          session_id = generate_id()
          self.sessions[session_id] = {
              "user_id": user_id,
              "state": AgentState(),
              "created_at": time.time(),
              "last_active": time.time()
          }
          return session_id

      def get_session(self, session_id):
          session = self.sessions.get(session_id)
          if session and time.time() - session["last_active"] < self.ttl:
              session["last_active"] = time.time()
              return session
          return None
  ```

---

### Q85: Multi-turn 对话的状态如何高效存储？

**答案要点：**

- **存储策略**：
  ```python
  class ConversationStore:
      def __init__(self):
          self.redis = Redis()
          self.postgres = PostgreSQL()

      def save_turn(self, session_id, turn):
          # 热数据存 Redis
          self.redis.lpush(f"conv:{session_id}", json.dumps(turn))
          self.redis.expire(f"conv:{session_id}", 3600)

          # 冷数据存 PostgreSQL
          self.postgres.insert("conversations", {
              "session_id": session_id,
              "turn": turn,
              "timestamp": datetime.now()
          })

      def get_recent_turns(self, session_id, n=10):
          # 优先从 Redis 读取
          turns = self.redis.lrange(f"conv:{session_id}", 0, n-1)
          if turns:
              return [json.loads(t) for t in turns]
          # 回退到 PostgreSQL
          return self.postgres.query(
              "SELECT turn FROM conversations WHERE session_id = %s ORDER BY timestamp DESC LIMIT %s",
              (session_id, n)
          )
  ```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
