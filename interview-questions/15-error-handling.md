# 十五、错误处理与可靠性

> 10 道题目，覆盖错误处理、可靠性模式

---

## 15.1 错误处理

### Q196: Agent 的常见错误类型有哪些？

**答案要点：**

| 错误类型 | 描述 | 处理方式 |
|----------|------|----------|
| LLM 错误 | API 失败、超时 | 重试、降级 |
| Tool 错误 | 工具执行失败 | 重试、跳过 |
| 解析错误 | 输出格式错误 | 重新生成 |
| 逻辑错误 | 推理错误 | 自我纠正 |
| 资源错误 | 配额耗尽 | 限流、告警 |

---

### Q197: 如何设计 Retry 策略？Exponential Backoff 的实现？

**答案要点：**

```python
class RetryStrategy:
    def __init__(self, max_retries=3, base_delay=1):
        self.max_retries = max_retries
        self.base_delay = base_delay

    async def execute_with_retry(self, func):
        for attempt in range(self.max_retries):
            try:
                return await func()
            except RetryableError as e:
                if attempt == self.max_retries - 1:
                    raise
                # 指数退避 + 抖动
                delay = self.base_delay * (2 ** attempt)
                jitter = random.uniform(0, delay * 0.1)
                await asyncio.sleep(delay + jitter)
```

---

### Q198: 什么是 Graceful Degradation？如何实现？

**答案要点：**

```python
class GracefulDegradation:
    async def execute(self, request):
        try:
            # 尝试完整功能
            return await self.full_execution(request)
        except PrimaryServiceUnavailable:
            # 降级到基础功能
            return await self.basic_execution(request)
        except AllServicesUnavailable:
            # 返回缓存或默认响应
            return self.get_cached_response(request)
```

---

### Q199: Tool 调用失败如何处理？

**答案要点：**

```python
class ToolErrorHandler:
    async def handle_tool_error(self, tool_call, error):
        # 1. 记录错误
        self.log_error(tool_call, error)

        # 2. 判断是否可重试
        if self.is_retryable(error):
            return await self.retry_tool(tool_call)

        # 3. 尝试替代工具
        alternative = self.find_alternative(tool_call)
        if alternative:
            return await self.execute_tool(alternative)

        # 4. 返回错误信息给 LLM
        return ToolResult(
            success=False,
            error=f"工具执行失败：{error}，请尝试其他方式"
        )
```

---

### Q200: 如何实现 Agent 的 Self-Healing？

**答案要点：**

```python
class SelfHealingAgent:
    async def execute_with_healing(self, task):
        for attempt in range(3):
            result = await self.execute(task)

            # 检查结果质量
            if self.validate_result(result):
                return result

            # 自我诊断
            diagnosis = await self.diagnose_failure(result)

            # 自我修复
            task = await self.adjust_approach(task, diagnosis)

        return FailedResult("无法完成任务")
```

---

## 15.2 可靠性模式

### Q201: 什么是 Circuit Breaker Pattern？在 Agent 中如何应用？

**答案要点：**

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failures = 0
        self.state = "closed"
        self.last_failure_time = None

    async def call(self, func):
        if self.state == "open":
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = "half-open"
            else:
                raise CircuitOpenError()

        try:
            result = await func()
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise

    def on_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        if self.failures >= self.failure_threshold:
            self.state = "open"
```

---

### Q202: 如何实现 Agent 的 Idempotency？

**答案要点：**

```python
class IdempotentAgent:
    def __init__(self):
        self.executed_operations = {}

    async def execute(self, operation_id, action):
        # 检查是否已执行
        if operation_id in self.executed_operations:
            return self.executed_operations[operation_id]

        # 执行操作
        result = await self.perform_action(action)

        # 记录结果
        self.executed_operations[operation_id] = result

        return result
```

---

### Q203: 什么是 Saga Pattern？在 Agent Workflow 中的应用？

**答案要点：**

```python
class SagaWorkflow:
    def __init__(self):
        self.steps = []
        self.compensations = []

    async def execute(self):
        completed = []
        try:
            for step in self.steps:
                result = await step.execute()
                completed.append(step)
        except Exception as e:
            # 回滚已完成的步骤
            for step in reversed(completed):
                await step.compensate()
            raise

    def add_step(self, execute_fn, compensate_fn):
        self.steps.append(SagaStep(execute_fn, compensate_fn))
```

---

### Q204: 如何处理 Partial Failure？

**答案要点：**

```python
class PartialFailureHandler:
    async def execute_batch(self, tasks):
        results = []
        failures = []

        for task in tasks:
            try:
                result = await self.execute(task)
                results.append(result)
            except Exception as e:
                failures.append({"task": task, "error": str(e)})

        return {
            "successful": results,
            "failed": failures,
            "success_rate": len(results) / len(tasks)
        }
```

---

### Q205: Agent 的 Timeout 策略如何设计？

**答案要点：**

```python
class TimeoutStrategy:
    def __init__(self):
        self.timeouts = {
            "llm_call": 30,
            "tool_execution": 60,
            "total_task": 300
        }

    async def execute_with_timeout(self, task):
        try:
            return await asyncio.wait_for(
                self.execute(task),
                timeout=self.timeouts["total_task"]
            )
        except asyncio.TimeoutError:
            # 保存进度
            await self.save_progress(task)
            raise TaskTimeoutError("任务执行超时")
```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
