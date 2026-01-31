# 十一、可观测性与监控

> 15 道题目，覆盖 Tracing、监控告警、调试与排错

---

## 11.1 Tracing

### Q146: 什么是 LLM Tracing？为什么重要？

**答案要点：**

- **定义**：追踪 LLM 应用的完整执行链路
- **重要性**：
  1. 调试问题根因
  2. 性能分析
  3. 成本归因
  4. 质量监控

---

### Q147: 如何实现 Agent 执行链路的完整追踪？

**答案要点：**

```python
class AgentTracer:
    def __init__(self):
        self.spans = []

    @contextmanager
    def trace_span(self, name, metadata=None):
        span = {
            "name": name,
            "start": time.time(),
            "metadata": metadata or {}
        }
        try:
            yield span
        finally:
            span["end"] = time.time()
            span["duration"] = span["end"] - span["start"]
            self.spans.append(span)

    async def trace_agent_execution(self, agent, input):
        with self.trace_span("agent_execution") as root:
            with self.trace_span("llm_call") as llm_span:
                response = await agent.llm.generate(input)
                llm_span["tokens"] = response.usage

            with self.trace_span("tool_execution") as tool_span:
                result = await agent.execute_tools(response.tool_calls)

        return self.spans
```

---

### Q148: LangSmith vs Langfuse vs Phoenix 的对比？

**答案要点：**

| 特性 | LangSmith | Langfuse | Phoenix |
|------|-----------|----------|---------|
| 开源 | 否 | 是 | 是 |
| 托管 | 是 | 是 | 否 |
| LangChain 集成 | 原生 | 好 | 好 |
| 评估功能 | 强 | 中 | 强 |
| 成本 | 付费 | 免费/付费 | 免费 |

---

### Q149: 如何设计 Agent 的 Span 和 Trace 结构？

**答案要点：**

```
Trace: agent_session_123
├── Span: user_input_processing
├── Span: llm_reasoning
│   ├── Span: prompt_construction
│   └── Span: model_inference
├── Span: tool_execution
│   ├── Span: tool_1_call
│   └── Span: tool_2_call
└── Span: response_generation
```

---

### Q150: 分布式 Agent 系统的 Tracing 如何实现？

**答案要点：**

```python
class DistributedTracer:
    def __init__(self):
        self.tracer = opentelemetry.trace.get_tracer(__name__)

    def propagate_context(self, headers):
        # 注入 trace context 到请求头
        inject(headers)

    def extract_context(self, headers):
        # 从请求头提取 trace context
        return extract(headers)

    async def call_remote_agent(self, agent_url, request):
        headers = {}
        self.propagate_context(headers)
        return await http_client.post(agent_url, request, headers=headers)
```

---

## 11.2 监控告警

### Q151: Agent 需要监控哪些关键指标？

**答案要点：**

| 类别 | 指标 | 告警阈值 |
|------|------|----------|
| 性能 | 响应延迟 P99 | > 10s |
| 性能 | 吞吐量 | < 预期 50% |
| 质量 | 任务成功率 | < 90% |
| 成本 | Token 使用量 | > 预算 120% |
| 可用性 | 错误率 | > 5% |

---

### Q152: 如何检测 Agent 的性能退化？

**答案要点：**

```python
class PerformanceMonitor:
    def __init__(self, baseline):
        self.baseline = baseline

    def detect_degradation(self, current_metrics):
        alerts = []

        if current_metrics.latency_p99 > self.baseline.latency_p99 * 1.5:
            alerts.append("延迟退化")

        if current_metrics.success_rate < self.baseline.success_rate * 0.9:
            alerts.append("成功率下降")

        return alerts
```

---

### Q153: 什么是 Drift Detection？如何实现？

**答案要点：**

- **Drift 类型**：
  1. 数据漂移：输入分布变化
  2. 概念漂移：任务性质变化
  3. 性能漂移：输出质量变化

```python
class DriftDetector:
    def detect_data_drift(self, recent_inputs, historical_inputs):
        recent_embedding = self.embed(recent_inputs)
        historical_embedding = self.embed(historical_inputs)
        distance = cosine_distance(recent_embedding, historical_embedding)
        return distance > self.threshold
```

---

### Q154: 如何设置 Agent 的 SLA 和告警阈值？

**答案要点：**

```python
SLA_CONFIG = {
    "availability": {
        "target": 99.9,
        "alert_threshold": 99.5
    },
    "latency_p99": {
        "target": 5000,  # ms
        "alert_threshold": 8000
    },
    "error_rate": {
        "target": 1,  # %
        "alert_threshold": 3
    }
}
```

---

### Q155: Agent 的健康检查如何设计？

**答案要点：**

```python
class AgentHealthCheck:
    async def check(self):
        checks = {
            "llm_connection": await self.check_llm(),
            "tool_availability": await self.check_tools(),
            "memory_store": await self.check_memory(),
            "vector_db": await self.check_vector_db()
        }

        healthy = all(checks.values())
        return {
            "status": "healthy" if healthy else "unhealthy",
            "checks": checks
        }
```

---

## 11.3 调试与排错

### Q156: Agent 调试的常见挑战有哪些？

**答案要点：**

1. **非确定性**：相同输入可能产生不同输出
2. **链路复杂**：多步骤、多工具调用
3. **外部依赖**：LLM API、工具服务
4. **状态管理**：上下文和记忆

---

### Q157: 如何复现和调试 Agent 的异常行为？

**答案要点：**

```python
class AgentDebugger:
    def capture_state(self, session_id):
        return {
            "messages": self.get_messages(session_id),
            "tool_calls": self.get_tool_calls(session_id),
            "llm_responses": self.get_llm_responses(session_id),
            "state": self.get_state(session_id)
        }

    def replay(self, captured_state):
        # 使用捕获的状态重放执行
        agent = Agent(deterministic=True)
        return agent.execute(captured_state)
```

---

### Q158: 什么是 Replay Debugging？如何实现？

**答案要点：**

```python
class ReplayDebugger:
    def __init__(self, event_log):
        self.event_log = event_log

    def replay_session(self, session_id):
        events = self.event_log.get_events(session_id)

        for event in events:
            print(f"[{event.timestamp}] {event.type}")
            print(f"  Input: {event.input}")
            print(f"  Output: {event.output}")
            print(f"  Duration: {event.duration}ms")
```

---

### Q159: 如何分析 Agent 的失败模式？

**答案要点：**

```python
class FailureAnalyzer:
    def analyze(self, failed_sessions):
        patterns = {}

        for session in failed_sessions:
            error_type = self.classify_error(session.error)
            root_cause = self.identify_root_cause(session)

            key = (error_type, root_cause)
            patterns[key] = patterns.get(key, 0) + 1

        return sorted(patterns.items(), key=lambda x: -x[1])
```

---

### Q160: 日志设计的最佳实践是什么？

**答案要点：**

```python
import structlog

logger = structlog.get_logger()

# 结构化日志
logger.info(
    "agent_execution",
    session_id=session_id,
    user_id=user_id,
    input_tokens=input_tokens,
    output_tokens=output_tokens,
    latency_ms=latency,
    tools_called=tools,
    success=True
)

# 日志级别
# DEBUG: 详细调试信息
# INFO: 正常操作
# WARNING: 潜在问题
# ERROR: 错误
# CRITICAL: 严重错误
```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
