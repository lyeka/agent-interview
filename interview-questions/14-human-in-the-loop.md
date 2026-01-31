# 十四、Human-in-the-Loop

> 10 道题目，覆盖设计模式、实现细节

---

## 14.1 设计模式

### Q186: 什么是 Human-in-the-Loop？为什么重要？

**答案要点：**

- **定义**：在 Agent 执行过程中引入人工审核和干预

- **重要性**：
  1. 安全保障：防止错误操作
  2. 质量控制：确保输出质量
  3. 合规要求：满足监管需求
  4. 信任建立：增强用户信心

---

### Q187: Approval Workflow 如何设计？

**答案要点：**

```python
class ApprovalWorkflow:
    async def execute_with_approval(self, action):
        # 1. 检查是否需要审批
        if self.requires_approval(action):
            # 2. 发送审批请求
            approval_request = await self.create_approval_request(action)

            # 3. 等待审批结果
            result = await self.wait_for_approval(approval_request)

            if result.approved:
                return await self.execute(action)
            else:
                return ApprovalRejected(result.reason)

        return await self.execute(action)
```

---

### Q188: 如何实现 Agent 的可中断性？

**答案要点：**

```python
class InterruptibleAgent:
    def __init__(self):
        self.interrupt_flag = False

    async def execute(self, task):
        for step in self.plan(task):
            # 检查中断标志
            if self.interrupt_flag:
                return InterruptedResult(completed_steps=self.completed)

            await self.execute_step(step)
            self.completed.append(step)

    def interrupt(self):
        self.interrupt_flag = True
```

---

### Q189: 什么是 Escalation Pattern？何时使用？

**答案要点：**

- **定义**：当 Agent 无法处理时，升级到人工

- **触发条件**：
  1. 置信度低
  2. 敏感操作
  3. 异常情况
  4. 用户请求

```python
class EscalationHandler:
    def should_escalate(self, context):
        if context.confidence < 0.7:
            return True
        if context.action in self.sensitive_actions:
            return True
        return False
```

---

### Q190: 如何平衡自动化和人工干预？

**答案要点：**

| 场景 | 自动化程度 | 人工干预 |
|------|------------|----------|
| 低风险查询 | 100% | 无 |
| 中风险操作 | 80% | 抽检 |
| 高风险操作 | 50% | 审批 |
| 关键决策 | 0% | 必须 |

---

## 14.2 实现细节

### Q191: 如何设计 Human Review 的 UI/UX？

**答案要点：**

- **关键要素**：
  1. 清晰展示 Agent 决策
  2. 提供上下文信息
  3. 简化审批操作
  4. 支持批量处理

---

### Q192: Async Approval 如何实现？

**答案要点：**

```python
class AsyncApproval:
    async def request_approval(self, action):
        # 1. 创建审批任务
        task_id = await self.create_task(action)

        # 2. 发送通知
        await self.notify_approvers(task_id)

        # 3. 返回任务 ID，不阻塞
        return task_id

    async def check_approval_status(self, task_id):
        return await self.get_task_status(task_id)

    async def on_approval_complete(self, task_id, result):
        # 回调处理
        if result.approved:
            await self.resume_execution(task_id)
```

---

### Q193: 如何处理 Human 响应的超时？

**答案要点：**

```python
class ApprovalTimeout:
    async def wait_with_timeout(self, approval_id, timeout=3600):
        try:
            result = await asyncio.wait_for(
                self.wait_for_approval(approval_id),
                timeout=timeout
            )
            return result
        except asyncio.TimeoutError:
            # 超时处理策略
            return await self.handle_timeout(approval_id)

    async def handle_timeout(self, approval_id):
        # 选项：自动拒绝、升级、提醒
        await self.send_reminder(approval_id)
        return TimeoutResult()
```

---

### Q194: Feedback Loop 如何设计？

**答案要点：**

```python
class FeedbackLoop:
    def collect_feedback(self, session_id, feedback):
        self.feedback_store.save({
            "session_id": session_id,
            "rating": feedback.rating,
            "comments": feedback.comments,
            "corrections": feedback.corrections
        })

    def apply_feedback(self, feedback):
        # 1. 更新 Prompt
        if feedback.prompt_improvement:
            self.update_prompt(feedback.prompt_improvement)

        # 2. 添加到训练数据
        if feedback.is_correction:
            self.add_training_example(feedback)
```

---

### Q195: 如何利用 Human Feedback 改进 Agent？

**答案要点：**

- **改进方式**：
  1. **Prompt 优化**：根据反馈调整提示
  2. **Few-shot 示例**：添加正确示例
  3. **规则更新**：添加业务规则
  4. **模型微调**：收集数据微调

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
