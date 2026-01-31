# 九、评估与测试

> 15 道题目，覆盖评估指标、Hallucination 检测、测试策略

---

## 9.1 评估指标

### Q116: 如何评估 Agent 的任务完成率？

**答案要点：**

```python
class TaskCompletionEvaluator:
    def evaluate(self, agent, test_cases):
        results = {"success": 0, "partial": 0, "failed": 0}

        for case in test_cases:
            output = agent.execute(case.input)
            score = self.score_completion(output, case.expected)

            if score >= 0.9:
                results["success"] += 1
            elif score >= 0.5:
                results["partial"] += 1
            else:
                results["failed"] += 1

        return {
            "completion_rate": results["success"] / len(test_cases),
            "partial_rate": results["partial"] / len(test_cases),
            "details": results
        }
```

---

### Q117: 什么是 Agent Benchmark？常用的有哪些（SWE-bench, GAIA, AgentBench）？

**答案要点：**

| Benchmark | 领域 | 评估内容 |
|-----------|------|----------|
| **SWE-bench** | 代码 | 解决 GitHub Issue |
| **GAIA** | 通用 | 多步骤推理任务 |
| **AgentBench** | 通用 | 多环境交互能力 |
| **WebArena** | Web | 网页操作任务 |
| **ToolBench** | 工具 | API 调用能力 |

---

### Q118: 如何评估 Agent 的推理能力？

**答案要点：**

- **评估维度**：
  1. 逻辑正确性
  2. 推理步骤完整性
  3. 结论准确性

```python
def evaluate_reasoning(agent_output, ground_truth):
    # 1. 步骤评估
    steps_score = evaluate_steps(agent_output.reasoning_steps)

    # 2. 结论评估
    conclusion_score = evaluate_conclusion(agent_output.conclusion, ground_truth)

    # 3. 一致性评估
    consistency_score = check_consistency(agent_output.reasoning_steps, agent_output.conclusion)

    return {
        "steps": steps_score,
        "conclusion": conclusion_score,
        "consistency": consistency_score
    }
```

---

### Q119: Tool Calling 的准确率如何衡量？

**答案要点：**

```python
class ToolCallingMetrics:
    def evaluate(self, predictions, ground_truth):
        return {
            # 工具选择准确率
            "tool_selection_accuracy": self.tool_accuracy(predictions, ground_truth),
            # 参数准确率
            "parameter_accuracy": self.param_accuracy(predictions, ground_truth),
            # 调用顺序准确率
            "sequence_accuracy": self.sequence_accuracy(predictions, ground_truth),
            # 端到端成功率
            "e2e_success_rate": self.e2e_success(predictions, ground_truth)
        }
```

---

### Q120: 如何评估 Multi-Agent 系统的协作效率？

**答案要点：**

- **评估指标**：
  1. 任务完成时间
  2. Agent 利用率
  3. 通信开销
  4. 冲突解决效率

---

## 9.2 Hallucination 检测

### Q121: 什么是 Hallucination？在 Agent 场景下有什么特殊挑战？

**答案要点：**

- **Hallucination 定义**：模型生成与事实不符或无依据的内容

- **Agent 特殊挑战**：
  1. 工具调用结果的误解
  2. 多步推理中的错误累积
  3. 外部数据的错误引用

---

### Q122: 如何检测 RAG 系统中的 Hallucination？

**答案要点：**

```python
class RAGHallucinationDetector:
    def detect(self, query, retrieved_docs, response):
        # 1. 事实一致性检查
        facts_in_response = extract_facts(response)
        facts_in_docs = extract_facts(retrieved_docs)
        unsupported = facts_in_response - facts_in_docs

        # 2. 引用验证
        citations = extract_citations(response)
        invalid_citations = self.verify_citations(citations, retrieved_docs)

        return {
            "unsupported_facts": unsupported,
            "invalid_citations": invalid_citations,
            "hallucination_score": len(unsupported) / len(facts_in_response)
        }
```

---

### Q123: Faithfulness vs Relevance vs Coherence 如何评估？

**答案要点：**

| 指标 | 定义 | 评估方法 |
|------|------|----------|
| **Faithfulness** | 输出与源文档一致 | NLI 模型判断 |
| **Relevance** | 输出与问题相关 | 语义相似度 |
| **Coherence** | 输出逻辑连贯 | 语言模型评分 |

---

### Q124: 什么是 Grounding？如何提高 Agent 输出的可靠性？

**答案要点：**

- **Grounding 定义**：将输出锚定到可验证的信息源

- **实现方式**：
  1. 强制引用来源
  2. 事实核查
  3. 置信度标注

---

### Q125: 常用的 Hallucination 检测工具有哪些？

**答案要点：**

| 工具 | 特点 |
|------|------|
| **Cleanlab TLM** | 自动检测不可信输出 |
| **Galileo** | 端到端评估平台 |
| **Ragas** | RAG 专用评估 |
| **DeepEval** | 开源评估框架 |

---

## 9.3 测试策略

### Q126: 如何设计 Agent 的端到端测试？

**答案要点：**

```python
class AgentE2ETest:
    def test_scenario(self, scenario):
        # 1. 设置环境
        env = self.setup_environment(scenario.config)

        # 2. 执行 Agent
        result = self.agent.execute(scenario.input, env)

        # 3. 验证结果
        assertions = [
            self.assert_output(result, scenario.expected_output),
            self.assert_tools_called(result, scenario.expected_tools),
            self.assert_no_errors(result)
        ]

        return all(assertions)
```

---

### Q127: 什么是 Agent Simulation Testing？

**答案要点：**

- **定义**：在模拟环境中测试 Agent 行为

- **优势**：
  1. 可重复
  2. 可控制
  3. 低成本
  4. 安全

---

### Q128: 如何进行 Agent 的回归测试？

**答案要点：**

```python
class RegressionTestSuite:
    def __init__(self):
        self.baseline_results = load_baseline()

    def run_regression(self, agent):
        current_results = self.run_all_tests(agent)

        regressions = []
        for test_id, result in current_results.items():
            baseline = self.baseline_results.get(test_id)
            if baseline and result.score < baseline.score * 0.95:
                regressions.append({
                    "test": test_id,
                    "baseline": baseline.score,
                    "current": result.score
                })

        return regressions
```

---

### Q129: Adversarial Testing 在 Agent 中如何应用？

**答案要点：**

- **测试类型**：
  1. Prompt Injection 测试
  2. 边界条件测试
  3. 错误输入测试
  4. 资源耗尽测试

---

### Q130: 如何建立 Agent 的 CI/CD Pipeline？

**答案要点：**

```yaml
# .github/workflows/agent-ci.yml
name: Agent CI/CD

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Unit Tests
        run: pytest tests/unit

      - name: Integration Tests
        run: pytest tests/integration

      - name: E2E Tests
        run: pytest tests/e2e

      - name: Evaluation
        run: python scripts/evaluate.py

      - name: Check Regression
        run: python scripts/regression_check.py
```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
