# 四、向量数据库与 Embedding

> 15 道题目，覆盖 Embedding 基础、向量数据库、相似度搜索

---

## 4.1 Embedding 基础

### Q46: 什么是 Text Embedding？常用的 Embedding 模型有哪些？

**答案要点：**

- **Text Embedding 定义**：将文本转换为固定维度的稠密向量，捕捉语义信息

- **常用模型**：
  | 模型 | 维度 | 特点 |
  |------|------|------|
  | OpenAI text-embedding-3-large | 3072 | 高质量，闭源 |
  | OpenAI text-embedding-3-small | 1536 | 性价比高 |
  | Cohere embed-v3 | 1024 | 多语言支持 |
  | BGE-large | 1024 | 开源，中文优秀 |
  | E5-large-v2 | 1024 | 开源，通用性强 |
  | GTE-large | 1024 | 阿里开源 |
  | Jina-embeddings-v2 | 768 | 长文本支持 |

- **选择考量**：
  - 语言支持
  - 维度和性能平衡
  - 开源 vs 闭源
  - 特定领域适配

---

### Q47: 如何选择 Embedding 模型？评估标准是什么？

**答案要点：**

- **评估标准**：
  1. **MTEB Benchmark**：多任务评估基准
  2. **检索任务**：Recall@K, MRR, NDCG
  3. **语义相似度**：Spearman 相关系数
  4. **分类任务**：准确率

- **选择流程**：
  ```python
  def evaluate_embedding_model(model, test_data):
      # 1. 检索评估
      retrieval_score = evaluate_retrieval(model, test_data.retrieval)

      # 2. 语义相似度评估
      similarity_score = evaluate_similarity(model, test_data.sts)

      # 3. 领域适配评估
      domain_score = evaluate_domain(model, test_data.domain)

      return {
          "retrieval": retrieval_score,
          "similarity": similarity_score,
          "domain": domain_score
      }
  ```

- **实际考量**：
  - 延迟要求
  - 成本预算
  - 部署环境（云/本地）

---

### Q48: Embedding 的维度如何影响性能和效果？

**答案要点：**

- **维度影响**：
  | 维度 | 存储 | 计算 | 表达能力 |
  |------|------|------|----------|
  | 低（256-512） | 小 | 快 | 有限 |
  | 中（768-1024） | 中 | 中 | 良好 |
  | 高（1536-3072） | 大 | 慢 | 强 |

- **权衡考量**：
  ```
  高维度：
  + 更丰富的语义表达
  + 更好的区分能力
  - 存储成本高
  - 计算���迟大
  - 可能过拟合

  低维度：
  + 存储效率高
  + 计算速度快
  - 信息损失
  - 区分能力弱
  ```

- **降维技术**：
  - PCA 降维
  - Matryoshka Representation Learning（MRL）
  - 量化压缩

---

### Q49: 什么是 Late Interaction（如 ColBERT）？与 Dense Embedding 的区别？

**答案要点：**

- **Dense Embedding**：
  ```
  Query → [Encoder] → 单一向量 q
  Doc → [Encoder] → 单一向量 d
  Score = similarity(q, d)
  ```

- **Late Interaction (ColBERT)**：
  ```
  Query → [Encoder] → 多个 Token 向量 [q1, q2, ..., qn]
  Doc → [Encoder] → 多个 Token 向量 [d1, d2, ..., dm]
  Score = Σ max_j(similarity(qi, dj))  # MaxSim
  ```

- **对比**：
  | 维度 | Dense | Late Interaction |
  |------|-------|------------------|
  | 表达能力 | 压缩为单向量 | 保留 Token 级信息 |
  | 存储 | 低 | 高 |
  | 计算 | 快 | 较慢 |
  | 效果 | 好 | 更好 |

- **适用场景**：
  - Dense：大规模检索、实时场景
  - Late Interaction：高精度要求、Re-ranking

---

### Q50: 如何处理 Embedding 的 Domain Adaptation？

**答案要点：**

- **问题**：通用 Embedding 模型在特定领域效果不佳

- **解决方案**：

  1. **Fine-tuning**：
     ```python
     from sentence_transformers import SentenceTransformer, losses

     model = SentenceTransformer('base-model')
     train_loss = losses.MultipleNegativesRankingLoss(model)
     model.fit(train_dataloader, loss=train_loss, epochs=3)
     ```

  2. **Contrastive Learning**：
     - 构建领域内的正负样本对
     - 拉近相似样本，推远不相似样本

  3. **Adapter**：
     - 冻结基础模型
     - 训练轻量级适配层

  4. **Prompt Tuning**：
     - 为特定领域设计查询前缀
     ```
     "医学问答：{query}" vs "法律咨询：{query}"
     ```

---

## 4.2 向量数据库

### Q51: 常见向量数据库对比：Pinecone vs Milvus vs Weaviate vs Qdrant vs Chroma

**答案要点：**

| 特性 | Pinecone | Milvus | Weaviate | Qdrant | Chroma |
|------|----------|--------|----------|--------|--------|
| 部署 | 云托管 | 自托管/云 | 自托管/云 | 自托管/云 | 本地/云 |
| 开源 | 否 | 是 | 是 | 是 | 是 |
| 扩展性 | 高 | 高 | 中 | 中 | 低 |
| 易用性 | 高 | 中 | 高 | 高 | 高 |
| 混合搜索 | 是 | 是 | 是 | 是 | 有限 |
| 适用场景 | 生产 | 大规模 | 语义搜索 | 中小规模 | 原型 |

- **选择建议**：
  - 快速原型：Chroma
  - 生产环境（托管）：Pinecone
  - 大规模自托管：Milvus
  - 中小规模自托管：Qdrant/Weaviate

---

### Q52: 什么是 ANN (Approximate Nearest Neighbor)？常用算法有哪些（HNSW, IVF, PQ）？

**答案要点：**

- **ANN 定义**：近似最近邻搜索，牺牲少量精度换取大幅速度提升

- **常用算法**：

  1. **HNSW (Hierarchical Navigable Small World)**：
     - 构建多层图结构
     - 从顶层快速定位，逐层精确
     - 优点：查询快，精度高
     - 缺点：内存占用大

  2. **IVF (Inverted File Index)**：
     - 聚类后建立倒排索引
     - 查询时只搜索相关聚类
     - 优点：内存效率高
     - 缺点：需要训练聚类

  3. **PQ (Product Quantization)**：
     - 向量压缩技术
     - 将向量分段量化
     - 优点：极大压缩存储
     - 缺点：精度损失

- **组合使用**：
  ```
  IVF + PQ：大规模场景
  HNSW + PQ：平衡速度和存储
  ```

---

### Q53: 如何在向量搜索中实现 Metadata Filtering？

**答案要点：**

- **Metadata Filtering 定义**：在向量搜索时同时应用属性过滤

- **实现方式**：

  1. **Pre-filtering**：
     ```
     先过滤 → 再向量搜索
     优点：精确
     缺点：可能过滤掉太多，结果不足
     ```

  2. **Post-filtering**：
     ```
     先向量搜索 → 再过滤
     优点：保证向量相似度
     缺点：可能需要多次搜索
     ```

  3. **Hybrid**：
     ```python
     results = vector_db.search(
         vector=query_embedding,
         filter={"category": "tech", "date": {"$gte": "2024-01-01"}},
         limit=10
     )
     ```

- **索引设计**：
  - 为常用过滤字段建立索引
  - 考虑过滤字段的基数

---

### Q54: 向量数据库的 Sharding 和 Replication 策略？

**答案要点：**

- **Sharding（分片）**：
  ```
  数据量大 → 水平分片 → 多节点存储

  策略：
  1. Hash Sharding：按 ID 哈希分片
  2. Range Sharding：按属性范围分片
  3. Cluster Sharding：按向量聚类分片
  ```

- **Replication（复制）**：
  ```
  高可用 → 数据复制 → 多副本

  策略：
  1. 主从复制：写主读从
  2. 多主复制：多点写入
  ```

- **查询路由**：
  ```python
  def distributed_search(query, k):
      # 并行查询所有分片
      results = parallel_query_all_shards(query, k)
      # 合并结果
      merged = merge_and_rank(results)
      return merged[:k]
  ```

---

### Q55: 如何处理向量数据库的增量更新和版本管理？

**答案要点：**

- **增量更新**：
  ```python
  # 1. Upsert 操作
  vector_db.upsert(
      ids=["doc_1", "doc_2"],
      embeddings=[emb1, emb2],
      metadatas=[meta1, meta2]
  )

  # 2. 增量索引
  # 新数据先写入缓冲区，定期合并到主索引
  ```

- **版本管理**：
  ```python
  # 1. 时间戳版本
  vector_db.upsert(
      id="doc_1",
      embedding=emb,
      metadata={"version": "2024-01-15", "content_hash": "abc123"}
  )

  # 2. 多版本存储
  # 保留历史版本，支持回滚
  ```

- **一致性保证**：
  - 写入确认机制
  - 事务支持（部分数据库）
  - 最终一致性 vs 强一致性

---

## 4.3 相似度搜索

### Q56: Cosine Similarity vs Euclidean Distance vs Dot Product 的区别和适用场景？

**答案要点：**

- **公式**：
  ```
  Cosine Similarity: cos(θ) = (A·B) / (||A|| * ||B||)
  Euclidean Distance: d = √Σ(ai - bi)²
  Dot Product: A·B = Σ(ai * bi)
  ```

- **对比**：
  | 指标 | 范围 | 归一化 | 适用场景 |
  |------|------|--------|----------|
  | Cosine | [-1, 1] | 是 | 文本相似度 |
  | Euclidean | [0, ∞) | 否 | 聚类、异常检测 |
  | Dot Product | (-∞, ∞) | 否 | 推荐系统 |

- **选择建议**：
  - 文本语义相似：Cosine（对长度不敏感）
  - 需要考虑向量模长：Dot Product
  - 几何距离：Euclidean

---

### Q57: 什么是 Hybrid Search？如何结合关键词搜索和向量搜索？

**答案要点：**

- **Hybrid Search 定义**：结合稀疏检索（关键词）和稠密检索（向量）

- **融合策略**：

  1. **分数融合**：
     ```python
     def hybrid_search(query, alpha=0.5):
         bm25_scores = bm25_search(query)  # 归一化到 [0,1]
         vector_scores = vector_search(query)  # 归一化到 [0,1]

         final_scores = {}
         for doc_id in set(bm25_scores) | set(vector_scores):
             s1 = bm25_scores.get(doc_id, 0)
             s2 = vector_scores.get(doc_id, 0)
             final_scores[doc_id] = alpha * s1 + (1-alpha) * s2

         return sorted(final_scores.items(), key=lambda x: -x[1])
     ```

  2. **RRF (Reciprocal Rank Fusion)**：
     ```python
     def rrf_fusion(rankings, k=60):
         scores = {}
         for ranking in rankings:
             for rank, doc_id in enumerate(ranking):
                 scores[doc_id] = scores.get(doc_id, 0) + 1/(k + rank + 1)
         return sorted(scores.items(), key=lambda x: -x[1])
     ```

---

### Q58: 如何处理 Out-of-Distribution 查询？

**答案要点：**

- **问题**：查询与索引数据分布差异大，检索效果差

- **检测方法**：
  ```python
  def detect_ood(query_embedding, index_stats):
      # 计算与索引中心的距离
      distance = cosine_distance(query_embedding, index_stats.centroid)
      # 与阈值比较
      return distance > index_stats.ood_threshold
  ```

- **处理策略**：
  1. **降级处理**：返回"无相关结果"
  2. **扩展检索**：使用 Web Search
  3. **Query 改写**：尝试不同表述
  4. **混合策略**：结合关键词搜索

---

### Q59: 什么是 MMR (Maximal Marginal Relevance)？如何实现结果多样性？

**答案要点：**

- **MMR 定义**：在保证相关性的同时，增加结果多样性

- **公式**：
  ```
  MMR = argmax[λ * Sim(di, q) - (1-λ) * max(Sim(di, dj))]

  其中：
  - Sim(di, q)：文档与查询的相似度
  - Sim(di, dj)：文档与已选文档的相似度
  - λ：相关性与多样性的权衡参数
  ```

- **实现**：
  ```python
  def mmr_selection(query_emb, doc_embs, k, lambda_param=0.5):
      selected = []
      candidates = list(range(len(doc_embs)))

      while len(selected) < k and candidates:
          best_score = -float('inf')
          best_idx = None

          for idx in candidates:
              relevance = cosine_sim(query_emb, doc_embs[idx])
              diversity = max([cosine_sim(doc_embs[idx], doc_embs[s])
                              for s in selected] or [0])
              score = lambda_param * relevance - (1-lambda_param) * diversity

              if score > best_score:
                  best_score = score
                  best_idx = idx

          selected.append(best_idx)
          candidates.remove(best_idx)

      return selected
  ```

---

### Q60: 如何优化大规模向量搜索的延迟？

**答案要点：**

- **索引优化**：
  - 选择合适的 ANN 算法（HNSW 查询快）
  - 调整索引参数（ef_construction, M）
  - 使用量化压缩（PQ, SQ）

- **查询优化**：
  ```python
  # 1. 批量查询
  results = vector_db.batch_search(queries, k=10)

  # 2. 预过滤减少搜索空间
  results = vector_db.search(
      vector=query,
      filter={"category": "tech"},  # 先过滤
      k=10
  )

  # 3. 调整搜索参数
  results = vector_db.search(
      vector=query,
      ef=50,  # 搜索时的候选数量
      k=10
  )
  ```

- **架构优化**：
  - 分片并行查询
  - 缓存热点查询
  - GPU 加速

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
