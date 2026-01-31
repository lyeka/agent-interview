# 三、RAG (Retrieval Augmented Generation)

> 15 道题目，覆盖 RAG 基础架构、检索优化、高级 RAG 技术

---

## 3.1 基础架构

### Q31: 解释 RAG 的基本工作流程：Indexing → Retrieval → Generation

**答案要点：**

- **三阶段流程**：
  ```
  [Indexing 阶段]
  文档 → 分块(Chunking) → 向量化(Embedding) → 存储(Vector DB)

  [Retrieval 阶段]
  用户查询 → 向量化 → 相似度搜索 → Top-K 文档

  [Generation 阶段]
  查询 + 检索文档 → LLM → 生成回答
  ```

- **各阶段关键点**：
  | 阶段 | 关键技术 | 常见问题 |
  |------|----------|----------|
  | Indexing | Chunking 策略、Embedding 模型 | 信息丢失、语义断裂 |
  | Retrieval | 相似度算法、Re-ranking | 召回不准、噪声多 |
  | Generation | Prompt 设计、上下文整合 | 幻觉、引用错误 |

- **Prompt 模板示例**：
  ```
  基于以下参考文档回答用户问题。如果文档中没有相关信息，请明确说明。

  参考文档：
  {retrieved_documents}

  用户问题：{query}
  ```

---

### Q32: 什么是 Naive RAG vs Advanced RAG vs Modular RAG？

**答案要点：**

- **Naive RAG**：
  - 基础的检索-生成流程
  - 简单的 Top-K 检索
  - 问题：检索质量差、上下文利用不足

- **Advanced RAG**：
  - 优化检索：Query Rewriting、HyDE
  - 优化索引：Hierarchical Index、Metadata
  - 优化生成：Re-ranking、Compression
  ```
  Query → [Query Optimization] → [Retrieval] → [Re-ranking] → [Generation]
  ```

- **Modular RAG**：
  - 模块化设计，灵活组合
  - 可插拔组件：Retriever、Reranker、Generator
  - 支持复杂流程：迭代检索、自适应检索
  ```
  [Router] → [Retriever A/B/C] → [Reranker] → [Generator]
      ↑                                           ↓
      └──────────── [Feedback Loop] ←────────────┘
  ```

---

### Q33: Chunking 策略有哪些？如何选择合适的 chunk size？

**答案要点：**

- **Chunking 策略**：
  | 策略 | 描述 | 适用场景 |
  |------|------|----------|
  | 固定大小 | 按字符/Token 数切分 | 通用场景 |
  | 句子分割 | 按句子边界切分 | 短文本 |
  | 段落分割 | 按段落切分 | 结构化文档 |
  | 语义分割 | 按语义相似度切分 | 长文本 |
  | 递归分割 | 多级分隔符递归切分 | 代码、Markdown |

- **Chunk Size 选择**：
  ```
  太小：语义不完整，检索噪声大
  太大：检索精度低，上下文浪费

  推荐范围：
  - 通用文本：500-1000 tokens
  - 技术文档：300-500 tokens
  - 代码：按函数/类切分
  ```

- **Overlap 设置**：
  - 通常 10-20% 重叠
  - 保持上下文连贯性

---

### Q34: 什么是 Semantic Chunking？与固定大小分块的区别？

**答案要点：**

- **Semantic Chunking 定义**：基于语义相似度进行分块，确保每个 chunk 语义完整

- **实现原理**：
  ```python
  def semantic_chunking(text, threshold=0.5):
      sentences = split_sentences(text)
      embeddings = embed(sentences)
      chunks = []
      current_chunk = [sentences[0]]

      for i in range(1, len(sentences)):
          similarity = cosine_similarity(
              embeddings[i], embeddings[i-1]
          )
          if similarity < threshold:
              # 语义断裂，开始新 chunk
              chunks.append(' '.join(current_chunk))
              current_chunk = []
          current_chunk.append(sentences[i])

      return chunks
  ```

- **对比**：
  | 维度 | 固定大小 | Semantic Chunking |
  |------|----------|-------------------|
  | 语义完整性 | 可能断裂 | 保持完整 |
  | 计算成本 | 低 | 高（需要 Embedding） |
  | Chunk 大小 | 均匀 | 不均匀 |
  | 适用场景 | 通用 | 高质量要求 |

---

### Q35: 如何处理跨 chunk 的上下文丢失问题？

**答案要点：**

- **问题描述**：重要信息可能跨越多个 chunk，单独检索时丢失上下文

- **解决方案**：

  1. **Chunk Overlap**：
     ```
     Chunk 1: [A B C D E]
     Chunk 2:     [D E F G H]
     Chunk 3:         [G H I J K]
     ```

  2. **Parent-Child 索引**：
     ```
     Parent Chunk (大块，用于上下文)
         ├── Child Chunk 1 (小块，用于检索)
         ├── Child Chunk 2
         └── Child Chunk 3
     检索时：匹配 Child，返回 Parent
     ```

  3. **Sentence Window**：
     - 检索时匹配句子
     - 返回时扩展到周围句子

  4. **Document Summary**：
     - 为每个文档生成摘要
     - 检索时同时考虑摘要和详细内容

---

## 3.2 检索优化

### Q36: Dense Retrieval vs Sparse Retrieval vs Hybrid Search 的区别？

**答案要点：**

- **Sparse Retrieval（稀疏检索）**：
  - 基于关键词匹配（BM25、TF-IDF）
  - 优点：精确匹配、可解释
  - 缺点：无法理解语义

- **Dense Retrieval（稠密检索）**：
  - 基于向量相似度
  - 优点：语义理解、泛化能力
  - 缺点：精确匹配差、需要训练

- **Hybrid Search（混合检索）**：
  ```python
  def hybrid_search(query, alpha=0.5):
      sparse_scores = bm25_search(query)
      dense_scores = vector_search(query)
      # 分数融合
      final_scores = alpha * sparse_scores + (1-alpha) * dense_scores
      return rank_by_score(final_scores)
  ```

- **选择建议**：
  | 场景 | 推荐方案 |
  |------|----------|
  | 精确术语查询 | Sparse |
  | 语义相似查询 | Dense |
  | 通用场景 | Hybrid |

---

### Q37: 什么是 Re-ranking？为什么需要两阶段检索？

**答案要点：**

- **两阶段检索**：
  ```
  Stage 1: 召回（Retrieval）
  - 快速、粗粒度
  - 从百万文档中召回 Top-100

  Stage 2: 精排（Re-ranking）
  - 慢速、细粒度
  - 从 Top-100 中精选 Top-10
  ```

- **为什么需要**：
  - 召回阶段追求速度，牺牲精度
  - 精排阶段追求精度，可以慢一些
  - 平衡效率和效果

- **Re-ranking 方法**：
  | 方法 | 描述 | 效果 |
  |------|------|------|
  | Cross-Encoder | Query-Doc 联合编码 | 最好 |
  | ColBERT | Late Interaction | 较好 |
  | LLM Re-ranking | 用 LLM 打分 | 好但慢 |

- **实现示例**：
  ```python
  from sentence_transformers import CrossEncoder

  reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')
  pairs = [(query, doc) for doc in retrieved_docs]
  scores = reranker.predict(pairs)
  reranked = sorted(zip(retrieved_docs, scores), key=lambda x: -x[1])
  ```

---

### Q38: 如何评估检索质量？常用的指标有哪些（Recall@K, MRR, NDCG）？

**答案要点：**

- **Recall@K**：
  - 定义：Top-K 结果中包含相关文档的比例
  - 公式：`Recall@K = |相关文档 ∩ Top-K| / |相关文档|`
  - 用途：评估召回能力

- **MRR (Mean Reciprocal Rank)**：
  - 定义：第一个相关结果排名的倒数的平均值
  - 公式：`MRR = (1/N) * Σ(1/rank_i)`
  - 用途：评估首个相关结果的位置

- **NDCG (Normalized Discounted Cumulative Gain)**：
  - 定义：考虑位置权重的相关性得分
  - 公式：`DCG = Σ(rel_i / log2(i+1))`
  - 用途：评估排序质量

- **实际应用**：
  ```python
  def evaluate_retrieval(queries, ground_truth, retriever):
      metrics = {"recall@5": [], "mrr": [], "ndcg@10": []}
      for query, relevant_docs in zip(queries, ground_truth):
          retrieved = retriever.search(query, k=10)
          metrics["recall@5"].append(recall_at_k(retrieved[:5], relevant_docs))
          metrics["mrr"].append(mrr(retrieved, relevant_docs))
          metrics["ndcg@10"].append(ndcg(retrieved, relevant_docs))
      return {k: np.mean(v) for k, v in metrics.items()}
  ```

---

### Q39: 什么是 Query Expansion/Rewriting？如何实现？

**答案要点：**

- **Query Expansion（查询扩展）**：
  - 添加同义词、相关词
  - 示例：`"Python 教程"` → `"Python 教程 入门 学习 编程"`

- **Query Rewriting（查询重写）**：
  - 用 LLM 重写查询，使其更适合检索
  ```python
  def rewrite_query(query):
      prompt = f"""
      将以下用户查询重写为更适合搜索的形式。
      保持原意，但使用更精确的术语。

      原始查询：{query}
      重写查询：
      """
      return llm.generate(prompt)
  ```

- **Multi-Query**：
  - 生成多个查询变体
  - 合并检索结果
  ```python
  def multi_query_retrieval(query):
      queries = generate_query_variants(query, n=3)
      all_docs = []
      for q in queries:
          all_docs.extend(retriever.search(q))
      return deduplicate_and_rank(all_docs)
  ```

---

### Q40: HyDE (Hypothetical Document Embeddings) 是什么？原理是什么？

**答案要点：**

- **HyDE 定义**：用 LLM 生成假设性答案文档，用其 Embedding 进行检索

- **原理**：
  ```
  传统：Query Embedding → 搜索 → 文档
  HyDE：Query → LLM 生成假设答案 → 答案 Embedding → 搜索 → 文档
  ```

- **为什么有效**：
  - Query 和 Document 在语义空间中可能不对齐
  - 假设答案更接近真实文档的表达方式
  - 弥补 Query-Document 的语义鸿沟

- **实现**：
  ```python
  def hyde_retrieval(query):
      # 1. 生成假设答案
      prompt = f"请回答以下问题（即使不确定也请尝试）：{query}"
      hypothetical_doc = llm.generate(prompt)

      # 2. 用假设答案的 Embedding 检索
      embedding = embed(hypothetical_doc)
      results = vector_db.search(embedding, k=10)

      return results
  ```

- **注意事项**：
  - 增加了 LLM 调用成本
  - 假设答案质量影响检索效果

---

## 3.3 高级 RAG 技术

### Q41: 什么是 Self-RAG？如何实现检索的自适应？

**答案要点：**

- **Self-RAG 定义**：让模型自己决定是否需要检索、检索什么、如何使用检索结果

- **核心机制**：
  ```
  1. [Retrieve] 判断是否需要检索
  2. [IsRel] 判断检索结果是否相关
  3. [IsSup] 判断生成内容是否有检索支持
  4. [IsUse] 判断生成内容是否有用
  ```

- **工作流程**：
  ```python
  def self_rag(query):
      # 判断是否需要检索
      need_retrieval = llm.predict_retrieval_need(query)

      if need_retrieval:
          docs = retriever.search(query)
          # 过滤不相关文档
          relevant_docs = [d for d in docs if llm.is_relevant(query, d)]
          response = llm.generate_with_docs(query, relevant_docs)
          # 验证生成内容
          if not llm.is_supported(response, relevant_docs):
              response = llm.regenerate(query, relevant_docs)
      else:
          response = llm.generate(query)

      return response
  ```

---

### Q42: 解释 CRAG (Corrective RAG) 的工作原理

**答案要点：**

- **CRAG 定义**：在检索后增加纠错机制，提高检索质量

- **核心流程**：
  ```
  Query → Retrieval → [Evaluator] →
      ├── Correct: 直接使用
      ├── Incorrect: 重新检索/Web Search
      └── Ambiguous: 知识精炼后使用
  ```

- **关键组件**：
  1. **Retrieval Evaluator**：评估检索结果质量
  2. **Knowledge Refinement**：提取关键信息，过滤噪声
  3. **Web Search**：检索失败时的后备方案

- **实现示例**：
  ```python
  def crag(query):
      docs = retriever.search(query)
      evaluation = evaluate_retrieval(query, docs)

      if evaluation == "correct":
          context = docs
      elif evaluation == "incorrect":
          context = web_search(query)
      else:  # ambiguous
          context = refine_knowledge(docs)

      return llm.generate(query, context)
  ```

---

### Q43: 什么是 Graph RAG？与传统 RAG 的区别？

**答案要点：**

- **Graph RAG 定义**：结合知识图谱的 RAG，利用实体关系增强检索和生成

- **核心区别**：
  | 维度 | 传统 RAG | Graph RAG |
  |------|----------|-----------|
  | 索引结构 | 向量索引 | 向量 + 图索引 |
  | 检索方式 | 相似度匹配 | 相似度 + 图遍历 |
  | 上下文 | 独立文档 | 关联实体和关系 |
  | 推理能力 | 弱 | 强（多跳推理） |

- **工作流程**：
  ```
  1. 构建知识图谱（实体、关系提取）
  2. 检索相关实体和子图
  3. 结合文档和图谱信息生成回答
  ```

- **适用场景**：
  - 需要多跳推理的问答
  - 实体关系密集的领域
  - 需要全局视角的总结任务

---

### Q44: 如何处理多模态 RAG（图片、表格、PDF）？

**答案要点：**

- **挑战**：
  - 图片：需要视觉理解
  - 表格：结构化信息提取
  - PDF：混合内容解析

- **处理策略**：

  1. **图片处理**：
     ```python
     # 方案1：图片描述
     description = vision_model.describe(image)
     embedding = embed(description)

     # 方案2：多模态 Embedding
     embedding = clip_model.encode_image(image)
     ```

  2. **表格处理**：
     ```python
     # 转换为结构化文本
     table_text = f"表格标题：{title}\n列：{columns}\n数据：{rows}"
     # 或保持 JSON 格式
     table_json = {"headers": [...], "rows": [...]}
     ```

  3. **PDF 处理**：
     ```python
     # 使用专门的 PDF 解析器
     from unstructured.partition.pdf import partition_pdf
     elements = partition_pdf(pdf_path)
     # 分别处理文本、表格、图片
     ```

---

### Q45: 什么是 Agentic RAG？Agent 如何增强 RAG 能力？

**答案要点：**

- **Agentic RAG 定义**：将 Agent 能力引入 RAG，实现更智能的检索和生成

- **增强方式**：
  1. **智能路由**：根据查询类型选择检索策略
  2. **迭代检索**：根据初步结果决定是否继续检索
  3. **多源整合**：协调多个数据源
  4. **自我纠错**：验证和修正检索结果

- **架构示例**：
  ```
  Query → [Router Agent]
              ├── 简单查询 → 直接检索
              ├── 复杂查询 → 分解 + 多轮检索
              └── 实时查询 → Web Search

  [Retrieval Agent] → 检索 → 评估 → 需要更多信息？
                                        ├── 是 → 继续检索
                                        └── 否 → 生成回答
  ```

- **优势**：
  - 更高的检索准确率
  - 更好的复杂问题处理
  - 自适应的检索策略

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
