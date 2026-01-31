# 十、生产部署与运维

> 15 道题目，覆盖部署架构、性能优化、成本控制

---

## 10.1 部署架构

### Q131: Agent 的生产部署架构有哪些模式？

**答案要点：**

| 模式 | 描述 | 适用场景 |
|------|------|----------|
| **单体部署** | 所有组件在一个服务 | 小规模、简单场景 |
| **微服务** | Agent、Tool、Memory 分离 | 中大规模 |
| **Serverless** | 按需执行 | 低频、突发流量 |
| **混合** | 核心服务 + Serverless 扩展 | 复杂场景 |

---

### Q132: 如何实现 Agent 的水平扩展？

**答案要点：**

```python
# 无状态 Agent 设计
class StatelessAgent:
    def __init__(self, state_store, tool_registry):
        self.state_store = state_store  # 外部状态存储
        self.tools = tool_registry

    async def execute(self, session_id, input):
        # 1. 加载状态
        state = await self.state_store.load(session_id)

        # 2. 执行逻辑
        result = await self.process(input, state)

        # 3. 保存状态
        await self.state_store.save(session_id, state)

        return result

# 部署多个实例，通过负载均衡分发请求
```

---

### Q133: 什么是 Agent Gateway？如何设计？

**答案要点：**

```python
class AgentGateway:
    def __init__(self):
        self.rate_limiter = RateLimiter()
        self.auth = AuthService()
        self.router = AgentRouter()

    async def handle_request(self, request):
        # 1. 认证
        user = await self.auth.authenticate(request)

        # 2. 限流
        await self.rate_limiter.check(user.id)

        # 3. 路由
        agent = self.router.select_agent(request)

        # 4. 执行
        response = await agent.execute(request)

        # 5. 日志
        self.log_request(request, response)

        return response
```

---

### Q134: 如何处理 Agent 的高可用和故障转移？

**答案要点：**

- **高可用策略**：
  1. 多实例部署
  2. 健康检查
  3. 自动重启
  4. 故障转移

```python
class HighAvailabilityAgent:
    def __init__(self, primary, fallback):
        self.primary = primary
        self.fallback = fallback

    async def execute(self, request):
        try:
            return await asyncio.wait_for(
                self.primary.execute(request),
                timeout=30
            )
        except (TimeoutError, ServiceUnavailable):
            return await self.fallback.execute(request)
```

---

### Q135: Serverless vs Container 部署 Agent 的优缺点？

**答案要点：**

| 维度 | Serverless | Container |
|------|------------|-----------|
| 冷启动 | 有延迟 | 无 |
| 成本 | 按调用计费 | 按资源计费 |
| 扩展 | 自动 | 需配置 |
| 状态 | 无状态 | 可有状态 |
| 适用 | 低频、突发 | 高频、稳定 |

---

## 10.2 性能优化

### Q136: 如何优化 Agent 的响应延迟？

**答案要点：**

```python
class LatencyOptimizer:
    # 1. 并行执行
    async def parallel_tools(self, tool_calls):
        return await asyncio.gather(*[
            self.execute_tool(call) for call in tool_calls
        ])

    # 2. 流式响应
    async def stream_response(self, prompt):
        async for chunk in self.llm.stream(prompt):
            yield chunk

    # 3. 预热
    def warmup(self):
        self.llm.generate("warmup")  # 预热模型连接
```

---

### Q137: 什么是 Streaming Response？如何实现？

**答案要点：**

```python
async def stream_agent_response(request):
    async def generate():
        async for chunk in agent.stream_execute(request):
            yield f"data: {json.dumps(chunk)}\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream"
    )
```

---

### Q138: Token 使用量如何优化？

**答案要点：**

- **优化策略**：
  1. Prompt 压缩
  2. 上下文裁剪
  3. 小模型预处理
  4. 缓存复用

```python
class TokenOptimizer:
    def optimize_context(self, messages, max_tokens):
        # 1. 移除冗余
        messages = self.remove_redundant(messages)

        # 2. 摘要历史
        if self.count_tokens(messages) > max_tokens:
            messages = self.summarize_old(messages)

        return messages
```

---

### Q139: 什么是 Model Routing？如何根据任务选择模型？

**答案要点：**

```python
class ModelRouter:
    def __init__(self):
        self.models = {
            "simple": "gpt-3.5-turbo",
            "complex": "gpt-4",
            "code": "claude-3-5-sonnet"
        }

    def route(self, task):
        complexity = self.assess_complexity(task)

        if complexity < 0.3:
            return self.models["simple"]
        elif "code" in task.type:
            return self.models["code"]
        else:
            return self.models["complex"]
```

---

### Q140: 如何实现 Agent 的 Caching 策略？

**答案要点：**

```python
class AgentCache:
    def __init__(self):
        self.cache = Redis()

    async def get_or_compute(self, key, compute_fn, ttl=3600):
        # 1. 检查缓存
        cached = await self.cache.get(key)
        if cached:
            return json.loads(cached)

        # 2. 计算结果
        result = await compute_fn()

        # 3. 存入缓存
        await self.cache.setex(key, ttl, json.dumps(result))

        return result
```

---

## 10.3 成本控制

### Q141: Agent 的成本构成有哪些？如何优化？

**答案要点：**

| 成本项 | 占比 | 优化方法 |
|--------|------|----------|
| LLM API | 60-80% | 模型路由、缓存 |
| 向量数据库 | 10-20% | 索引优化 |
| 计算资源 | 5-15% | 自动扩缩容 |
| 存储 | 5-10% | 数据清理 |

---

### Q142: 什么是 Prompt Caching？如何利用它降低成本？

**答案要点：**

```python
# Anthropic Prompt Caching
response = client.messages.create(
    model="claude-3-5-sonnet",
    system=[{
        "type": "text",
        "text": long_system_prompt,
        "cache_control": {"type": "ephemeral"}
    }],
    messages=[...]
)

# 缓存命中时，输入 Token 成本降低 90%
```

---

### Q143: 如何实现 Token Budget 管理？

**答案要点：**

```python
class TokenBudgetManager:
    def __init__(self, daily_budget):
        self.daily_budget = daily_budget
        self.usage = {}

    async def check_budget(self, user_id, estimated_tokens):
        today = date.today().isoformat()
        used = self.usage.get(f"{user_id}:{today}", 0)

        if used + estimated_tokens > self.daily_budget:
            raise BudgetExceeded("今日配额已用完")

    async def record_usage(self, user_id, tokens):
        today = date.today().isoformat()
        key = f"{user_id}:{today}"
        self.usage[key] = self.usage.get(key, 0) + tokens
```

---

### Q144: 小模型 vs 大模型的成本效益分析？

**答案要点：**

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| 简单分类 | 小模型 | 成本低，速度快 |
| 复杂推理 | 大模型 | 准确率高 |
| 代码生成 | 中等模型 | 平衡成本和质量 |

---

### Q145: 如何监控和预警 Agent 的成本异常？

**答案要点：**

```python
class CostMonitor:
    def __init__(self, alert_threshold):
        self.threshold = alert_threshold

    async def check_anomaly(self, current_cost, historical_avg):
        if current_cost > historical_avg * self.threshold:
            await self.send_alert({
                "type": "cost_anomaly",
                "current": current_cost,
                "expected": historical_avg,
                "ratio": current_cost / historical_avg
            })
```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
