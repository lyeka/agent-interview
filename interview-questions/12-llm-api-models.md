# 十二、LLM API 与模型选择

> 15 道题目，覆盖 API 使用、模型选择、模型能力

---

## 12.1 API 使用

### Q161: OpenAI API vs Anthropic API vs Google API 的主要区别？

**答案要点：**

| 维度 | OpenAI | Anthropic | Google |
|------|--------|-----------|--------|
| 模型 | GPT-4o, o1 | Claude 3.5 | Gemini |
| Tool Calling | function_call | tool_use | function_calling |
| 流式 | stream=True | stream=True | stream=True |
| 特色 | JSON Mode | Prompt Caching | 长上下文 |

---

### Q162: 如何处理 API 的 Rate Limiting？

**答案要点：**

```python
class RateLimitHandler:
    def __init__(self):
        self.retry_delays = [1, 2, 4, 8, 16]

    async def call_with_retry(self, api_call):
        for delay in self.retry_delays:
            try:
                return await api_call()
            except RateLimitError:
                await asyncio.sleep(delay)
        raise MaxRetriesExceeded()

    async def call_with_backoff(self, api_call):
        # 指数退避
        delay = 1
        while True:
            try:
                return await api_call()
            except RateLimitError as e:
                if delay > 60:
                    raise
                await asyncio.sleep(delay + random.uniform(0, 1))
                delay *= 2
```

---

### Q163: 什么是 Batch API？适用于什么场景？

**答案要点：**

- **Batch API 定义**：批量处理请求，异步返回结果

- **适用场景**：
  1. 大规模数据处理
  2. 非实时任务
  3. 成本敏感场景（通常有折扣）

```python
# OpenAI Batch API
batch = client.batches.create(
    input_file_id="file-xxx",
    endpoint="/v1/chat/completions",
    completion_window="24h"
)
```

---

### Q164: 如何实现多模型的 Fallback 策略？

**答案要点：**

```python
class ModelFallback:
    def __init__(self):
        self.models = [
            {"name": "gpt-4o", "client": openai_client},
            {"name": "claude-3-5-sonnet", "client": anthropic_client},
            {"name": "gemini-pro", "client": google_client}
        ]

    async def call(self, prompt):
        for model in self.models:
            try:
                return await model["client"].generate(prompt)
            except (RateLimitError, ServiceUnavailable):
                continue
        raise AllModelsFailed()
```

---

### Q165: API 调用的错误处理最佳实践？

**答案要点：**

```python
class APIErrorHandler:
    async def safe_call(self, api_call):
        try:
            return await api_call()
        except RateLimitError:
            # 等待后重试
            await asyncio.sleep(60)
            return await api_call()
        except InvalidRequestError as e:
            # 记录并返回错误
            logger.error(f"Invalid request: {e}")
            raise
        except APIConnectionError:
            # 使用备用模型
            return await self.fallback_call()
        except Exception as e:
            # 通用错误处理
            logger.exception("Unexpected error")
            raise
```

---

## 12.2 模型选择

### Q166: 如何根据任务选择合适的 LLM？

**答案要点：**

| 任务类型 | 推荐模型 | 原因 |
|----------|----------|------|
| 简单对话 | GPT-3.5/Haiku | 成本低、速度快 |
| 复杂推理 | GPT-4o/Opus | 能力强 |
| 代码生成 | Claude 3.5 Sonnet | 代码能力强 |
| 长文档 | Gemini 1.5 | 长上下文 |
| 数学推理 | o1 | 专门优化 |

---

### Q167: 开源模型 vs 闭源模型的选择标准？

**答案要点：**

| 维度 | 开源 | 闭源 |
|------|------|------|
| 成本 | 自托管成本 | API 费用 |
| 隐私 | 数据不出境 | 数据上传 |
| 定制 | 可微调 | 有限 |
| 性能 | 中等 | 最强 |
| 运维 | 需要 | 无需 |

---

### Q168: 什么是 Model Distillation？如何应用于 Agent？

**答案要点：**

- **定义**：用大模型生成数据训练小模型

```python
# 1. 用大模型生成训练数据
training_data = []
for task in tasks:
    response = large_model.generate(task)
    training_data.append((task, response))

# 2. 微调小模型
small_model.fine_tune(training_data)

# 3. 部署小模型处理简单任务
```

---

### Q169: Fine-tuning vs Prompt Engineering 的选择？

**答案要点：**

| 场景 | 推荐方案 |
|------|----------|
| 快速迭代 | Prompt Engineering |
| 特定格式输出 | Fine-tuning |
| 领域知识 | RAG + Prompt |
| 大规模部署 | Fine-tuning（降低成本） |

---

### Q170: 如何评估模型在特定任务上的表现？

**答案要点：**

```python
class ModelEvaluator:
    def evaluate(self, model, test_set):
        results = []
        for input, expected in test_set:
            output = model.generate(input)
            score = self.score(output, expected)
            results.append(score)

        return {
            "accuracy": np.mean([r["correct"] for r in results]),
            "latency_p50": np.percentile([r["latency"] for r in results], 50),
            "cost_per_query": np.mean([r["cost"] for r in results])
        }
```

---

## 12.3 模型能力

### Q171: 不同模型的 Context Window 限制如何影响 Agent 设计？

**答案要点：**

| 模型 | Context Window | 设计影响 |
|------|----------------|----------|
| GPT-4o | 128K | 可处理长文档 |
| Claude 3.5 | 200K | 更长上下文 |
| Gemini 1.5 | 1M | 超长文档 |

- **设计考量**：
  1. 上下文管理策略
  2. 分块处理
  3. 摘要压缩

---

### Q172: 什么是 Vision 能力？如何在 Agent 中使用？

**答案要点：**

```python
# 多模态输入
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "描述这张图片"},
            {"type": "image_url", "image_url": {"url": image_url}}
        ]
    }]
)
```

---

### Q173: Code Interpreter 能力如何集成？

**答案要点：**

```python
# OpenAI Code Interpreter
assistant = client.beta.assistants.create(
    model="gpt-4o",
    tools=[{"type": "code_interpreter"}]
)

# 上传文件
file = client.files.create(file=open("data.csv"), purpose="assistants")

# 执行分析
run = client.beta.threads.runs.create(
    thread_id=thread.id,
    assistant_id=assistant.id
)
```

---

### Q174: 什么是 Computer Use？Anthropic 的实现原理？

**答案要点：**

- **定义**：让 AI 控制计算机界面

- **原理**：
  1. 截图 → 视觉理解
  2. 决策 → 下一步操作
  3. 执行 → 鼠标/键盘操作
  4. 循环 → 直到任务完成

```python
# Anthropic Computer Use
response = client.messages.create(
    model="claude-3-5-sonnet",
    tools=[{
        "type": "computer_20241022",
        "display_width_px": 1920,
        "display_height_px": 1080
    }],
    messages=[{"role": "user", "content": "打开浏览器搜索天气"}]
)
```

---

### Q175: 如何利用模型的 JSON Mode？

**答案要点：**

```python
# OpenAI JSON Mode
response = client.chat.completions.create(
    model="gpt-4o",
    response_format={"type": "json_object"},
    messages=[{
        "role": "user",
        "content": "提取用户信息，返回 JSON 格式"
    }]
)

# Structured Outputs
from pydantic import BaseModel

class UserInfo(BaseModel):
    name: str
    age: int

response = client.beta.chat.completions.parse(
    model="gpt-4o",
    response_format=UserInfo,
    messages=[...]
)
```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
