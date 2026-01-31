# 评估与可观测性

RAG 系统的质量评估是一个复杂问题。LightRAG 集成了 RAGAS 评估框架和 Langfuse 可观测性平台，提供完整的质量保证方案。

## RAGAS 评估集成

### RAGEvaluator 类

**源码位置**: `lightrag/evaluation/eval_rag_quality.py`

LightRAG 提供了 `RAGEvaluator` 类，封装了 RAGAS 评估逻辑：

```python
class RAGEvaluator:
    def __init__(
        self,
        rag_instance: LightRAG,
        llm_model: str = "gpt-4o-mini",
        embedding_model: str = "text-embedding-3-small",
    ):
        self.rag = rag_instance
        self.llm = ChatOpenAI(model=llm_model)
        self.embeddings = OpenAIEmbeddings(model=embedding_model)

    async def evaluate(
        self,
        questions: list[str],
        ground_truths: list[str] | None = None,
        query_mode: str = "mix",
    ) -> dict:
        # 1. 执行查询
        responses = []
        contexts = []
        for question in questions:
            result = await self.rag.aquery_llm(question, QueryParam(mode=query_mode))
            responses.append(result["llm_response"]["content"])
            contexts.append(result["raw_data"])

        # 2. 构建评估数据集
        dataset = Dataset.from_dict({
            "question": questions,
            "answer": responses,
            "contexts": contexts,
            "ground_truth": ground_truths or [""] * len(questions),
        })

        # 3. 执行 RAGAS 评估
        results = evaluate(
            dataset,
            metrics=[
                faithfulness,
                answer_relevancy,
                context_precision,
                context_recall,
            ],
            llm=self.llm,
            embeddings=self.embeddings,
        )

        return results
```

### RAGAS 评估指标

| 指标 | 含义 | 计算方式 |
|------|------|----------|
| **Faithfulness** | 回答是否忠于上下文 | LLM 判断回答中的声明是否能从上下文推导 |
| **Answer Relevancy** | 回答是否相关 | 生成问题与原问题的语义相似度 |
| **Context Precision** | 上下文精度 | 相关上下文在检索��果中的排名 |
| **Context Recall** | 上下文召回 | 检索结果覆盖 ground truth 的程度 |

### 评估示例

```python
# 初始化评估器
evaluator = RAGEvaluator(rag_instance)

# 准备测试数据
questions = [
    "苹果公司的 CEO 是谁？",
    "特斯拉的市值是多少？",
]
ground_truths = [
    "Tim Cook 是苹果公司的 CEO。",
    "特斯拉的市值约为 8000 亿美元。",
]

# 执行评估
results = await evaluator.evaluate(
    questions=questions,
    ground_truths=ground_truths,
    query_mode="mix",
)

print(f"Faithfulness: {results['faithfulness']:.2f}")
print(f"Answer Relevancy: {results['answer_relevancy']:.2f}")
print(f"Context Precision: {results['context_precision']:.2f}")
print(f"Context Recall: {results['context_recall']:.2f}")
```

## Langfuse 可观测性

### 集成方式

LightRAG 支持通过环境变量配置 Langfuse：

```bash
export LANGFUSE_PUBLIC_KEY="pk-..."
export LANGFUSE_SECRET_KEY="sk-..."
export LANGFUSE_HOST="https://cloud.langfuse.com"
```

### 追踪内容

Langfuse 可以追踪以下内容：

1. **LLM 调用**：
   - 输入 Prompt
   - 输出响应
   - Token 使用量
   - 延迟时间

2. **检索过程**：
   - 查询关键词
   - 检索结果
   - Rerank 分数

3. **文档处理**：
   - 分块结果
   - 实体提取
   - 图谱更新

### 使用示例

```python
from langfuse import Langfuse

# 初始化 Langfuse
langfuse = Langfuse()

# 创建 trace
trace = langfuse.trace(name="rag_query")

# 记录检索
span = trace.span(name="retrieval")
span.update(
    input={"query": query, "mode": "mix"},
    output={"entities": entities, "relations": relations},
)

# 记录 LLM 调用
generation = trace.generation(
    name="llm_response",
    model="gpt-4o-mini",
    input=prompt,
    output=response,
    usage={"input_tokens": 1000, "output_tokens": 500},
)

# 结束 trace
trace.update(output={"response": response})
```

## 日志系统

### 日志配置

**源码位置**: `lightrag/utils.py`

```python
def setup_logger(
    name: str,
    level: str = "INFO",
    log_file_path: str | None = None,
):
    logger = logging.getLogger(name)
    logger.setLevel(getattr(logging, level.upper()))

    # 控制台输出
    console_handler = logging.StreamHandler()
    console_handler.setFormatter(
        logging.Formatter("%(asctime)s - %(name)s - %(levelname)s - %(message)s")
    )
    logger.addHandler(console_handler)

    # 文件输出（可选）
    if log_file_path:
        file_handler = RotatingFileHandler(
            log_file_path,
            maxBytes=10 * 1024 * 1024,  # 10MB
            backupCount=5,
        )
        file_handler.setFormatter(
            logging.Formatter("%(asctime)s - %(name)s - %(levelname)s - %(message)s")
        )
        logger.addHandler(file_handler)

    return logger
```

### 日志级别

| 级别 | 用途 | 示例 |
|------|------|------|
| DEBUG | 详细调试信息 | 向量检索结果、Token 计数 |
| INFO | 正常运行信息 | 文档插入完成、查询执行 |
| WARNING | 潜在问题 | 关键词为空、缓存未命中 |
| ERROR | 错误信息 | LLM 调用失败、存储异常 |

### 关键日志点

```python
# 文档插入
logger.info(f"Inserted document {doc_id} with {len(chunks)} chunks")

# 实体提取
logger.debug(f"Extracted {len(entities)} entities, {len(relations)} relations")

# 查询执行
logger.info(f"Query mode={mode}, entities={len(entities)}, relations={len(relations)}")

# 缓存命中
logger.debug(f"Cache hit for query hash {args_hash}")

# 错误处理
logger.error(f"LLM call failed: {e}", exc_info=True)
```

## 性能监控

### Token 使用追踪

LightRAG 提供 Token 使用追踪功能：

```python
class TokenTracker:
    def __init__(self):
        self.input_tokens = 0
        self.output_tokens = 0
        self.embedding_tokens = 0

    def track_llm_call(self, input_tokens: int, output_tokens: int):
        self.input_tokens += input_tokens
        self.output_tokens += output_tokens

    def track_embedding_call(self, tokens: int):
        self.embedding_tokens += tokens

    def get_summary(self) -> dict:
        return {
            "input_tokens": self.input_tokens,
            "output_tokens": self.output_tokens,
            "embedding_tokens": self.embedding_tokens,
            "total_tokens": self.input_tokens + self.output_tokens + self.embedding_tokens,
        }
```

### 延迟监控

```python
import time

class LatencyTracker:
    def __init__(self):
        self.latencies = {}

    def track(self, operation: str):
        class Timer:
            def __init__(self, tracker, op):
                self.tracker = tracker
                self.op = op

            def __enter__(self):
                self.start = time.time()
                return self

            def __exit__(self, *args):
                elapsed = time.time() - self.start
                if self.op not in self.tracker.latencies:
                    self.tracker.latencies[self.op] = []
                self.tracker.latencies[self.op].append(elapsed)

        return Timer(self, operation)

# 使用示例
tracker = LatencyTracker()

with tracker.track("retrieval"):
    results = await rag.aquery_data(query)

with tracker.track("llm_generation"):
    response = await rag.aquery(query)

print(tracker.latencies)
# {"retrieval": [0.5, 0.6], "llm_generation": [2.1, 1.8]}
```

## 质量保证最佳实践

### 1. 建立基准测试集

```python
# 创建测试数据集
test_cases = [
    {
        "question": "苹果公司的 CEO 是谁？",
        "expected_entities": ["Apple", "Tim Cook"],
        "expected_answer_contains": ["Tim Cook"],
    },
    # ... 更多测试用例
]

# 运行基准测试
async def run_benchmark(rag, test_cases):
    results = []
    for case in test_cases:
        response = await rag.aquery(case["question"])
        result = {
            "question": case["question"],
            "response": response,
            "entity_match": check_entities(response, case["expected_entities"]),
            "answer_match": check_answer(response, case["expected_answer_contains"]),
        }
        results.append(result)
    return results
```

### 2. 定期评估

```python
# 每周运行评估
async def weekly_evaluation():
    evaluator = RAGEvaluator(rag_instance)

    results = await evaluator.evaluate(
        questions=benchmark_questions,
        ground_truths=benchmark_answers,
    )

    # 检查指标是否下降
    if results["faithfulness"] < 0.8:
        alert("Faithfulness dropped below threshold!")

    if results["context_recall"] < 0.7:
        alert("Context recall dropped below threshold!")

    # 保存结果
    save_evaluation_results(results)
```

### 3. A/B 测试

```python
# 比较不同检索模式
async def ab_test_modes():
    modes = ["local", "global", "hybrid", "mix"]
    results = {}

    for mode in modes:
        evaluator = RAGEvaluator(rag_instance)
        results[mode] = await evaluator.evaluate(
            questions=test_questions,
            ground_truths=test_answers,
            query_mode=mode,
        )

    # 比较结果
    for mode, result in results.items():
        print(f"{mode}: faithfulness={result['faithfulness']:.2f}, "
              f"relevancy={result['answer_relevancy']:.2f}")
```

---

## 章节衔接

**本章回顾**：
- 我们学习了 RAGAS 评估和 Langfuse 可观测性
- 关键收获：评估指标、日志系统、性能监控

**下一章预告**：
- 在 `08-qa.md` 中，我们将进入面试 QA 环节
- 为什么需要学习：面试是检验学习成果的最佳方式
- 关键问题：15 道核心面试题，覆盖架构、检索、存储、部署
