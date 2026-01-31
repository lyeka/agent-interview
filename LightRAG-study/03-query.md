# 多模式检索系统

LightRAG 提供五种检索模式：local、global、hybrid、mix、naive。本章深入分析每种模式的实现原理和适用场景。

## 检索模式概览

| 模式 | 数据源 | 适用场景 | 推荐度 |
|------|--------|----------|--------|
| **local** | 实体 + 相关块 | 特定实体查询（"张三是谁"） | ★★★ |
| **global** | 关系 + 社区摘要 | 宏观主题查询（"公司组织架构"） | ★★★ |
| **hybrid** | local + global | 平衡查询 | ★★★★ |
| **mix** | hybrid + 向量检索 | 最全面（推荐） | ★★★★★ |
| **naive** | 纯向量检索 | 无 KG 场景 | ★★ |
| **bypass** | 空数据 | 直接 LLM 对话 | ★ |

## 关键词提取：双层级系统

在执行检索之前，LightRAG 先从查询中提取关键词。这是一个双层级系统：

- **high_level_keywords**：主题、概念、意图（用于 global 模式）
- **low_level_keywords**：实体、专有名词、技术术语（用于 local 模式）

**源码位置**: `lightrag/prompt.py:374-393`

```python
PROMPTS["keywords_extraction"] = """---Role---
You are an expert keyword extractor for RAG system.

---Goal---
Extract two types of keywords:
1. **high_level_keywords**: overarching concepts/themes (core intent, subject area)
2. **low_level_keywords**: specific entities/details (proper nouns, technical jargon)

---Instructions---
1. **Output Format**: Valid JSON only, no markdown fences
2. **Concise & Meaningful**: Multi-word phrases for single concepts
   - Example: "latest financial report of Apple Inc."
   - Extract: ["latest financial report", "Apple Inc."]
   - NOT: ["latest", "financial", "report", "Apple"]
"""
```

**示例输出**：
```json
{
  "high_level_keywords": ["International trade", "Global economic stability"],
  "low_level_keywords": ["Trade agreements", "Tariffs", "Currency exchange"]
}
```

**设计意图**：

- **多词短语**：保持语义完整性，避免拆分成单词。
- **严格 JSON**：禁止 markdown 包裹，直接 `json.loads()`。
- **边界情况**：对于模糊查询（如"hello"），返回空列表。

## Local 模式：实体聚焦检索

Local 模式从实体出发，通过向量检索找到相关实体，再通过图谱扩展找到相关关系。

**源码位置**: `lightrag/operate.py:4167-4217`

```python
async def _get_node_data(
    query: str,
    knowledge_graph_inst: BaseGraphStorage,
    entities_vdb: BaseVectorStorage,
    query_param: QueryParam,
):
    # 1. 向量检索相似实体
    results = await entities_vdb.query(query, top_k=query_param.top_k)

    if not len(results):
        return [], []

    node_ids = [r["entity_name"] for r in results]

    # 2. 批量获取实体数据 + 度数（并发）
    nodes_dict, degrees_dict = await asyncio.gather(
        knowledge_graph_inst.get_nodes_batch(node_ids),
        knowledge_graph_inst.node_degrees_batch(node_ids),
    )

    # 3. 组装实体数据（带 rank）
    node_datas = [
        {
            **n,
            "entity_name": k["entity_name"],
            "rank": d,  # 度数作为排序依据
        }
        for k, n, d in zip(results, node_datas, node_degrees)
        if n is not None
    ]

    # 4. 图谱扩展：找到实体间的关系
    use_relations = await _find_most_related_edges_from_entities(
        node_datas, query_param, knowledge_graph_inst,
    )

    return node_datas, use_relations
```

**执行流程**：

```
查询 "张三是谁"
        │
        ▼
┌───────────────────────────────────────┐
│ 1. 向量检索：entities_vdb.query()     │
│    → 找到 ["张三", "李四", "王五"]     │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│ 2. 批量获取：get_nodes_batch()        │
│    → 获取实体描述、类型、来源         │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│ 3. 图谱扩展：_find_most_related_edges │
│    → 找到 "张三-李四" 的关系          │
└───────────────────────────────────────┘
```

**适用场景**：

- 查询特定实体（"苹果公司的 CEO 是谁"）
- 查询实体属性（"北京的人口是多少"）
- 查询实体关系（"张三和李四是什么关系"）

## Global 模式：关系聚焦检索

Global 模式从关系出发，通过向量检索找到相关关系，再提取涉及的实体。

**源码位置**: `lightrag/operate.py:4440-4278`

```python
async def _get_edge_data(
    keywords,
    knowledge_graph_inst: BaseGraphStorage,
    relationships_vdb: BaseVectorStorage,
    query_param: QueryParam,
):
    # 1. 向量检索相似关系
    results = await relationships_vdb.query(keywords, top_k=query_param.top_k)

    if not len(results):
        return [], []

    # 2. 批量获取关系数据 + 度数
    edge_pairs_tuples = [(r["src_id"], r["tgt_id"]) for r in results]

    edge_data_dict, edge_degrees_dict = await asyncio.gather(
        knowledge_graph_inst.get_edges_batch(edge_pairs_dicts),
        knowledge_graph_inst.edge_degrees_batch(edge_pairs_tuples),
    )

    # 3. 组装关系数据（带 rank + weight）
    all_edges_data = []
    for pair in edge_pairs_tuples:
        edge_props = edge_data_dict.get(pair)
        if edge_props is not None:
            combined = {
                "src_tgt": pair,
                "rank": edge_degrees_dict.get(pair, 0),
                **edge_props,
            }
            all_edges_data.append(combined)

    # 4. 按 rank + weight 排序
    all_edges_data = sorted(
        all_edges_data, key=lambda x: (x["rank"], x["weight"]), reverse=True
    )

    # 5. 提取关系涉及的实体
    global_entities = await _get_entities_from_relations(
        all_edges_data, knowledge_graph_inst,
    )

    return all_edges_data, global_entities
```

**执行流程**：

```
查询 "公司的组织架构"
        │
        ▼
┌───────────────────────────────────────┐
│ 1. 向量检索：relationships_vdb.query()│
│    → 找到 ["汇报关系", "管理关系"]     │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│ 2. 批量获取：get_edges_batch()        │
│    → 获取关系描述、关键词、权重       │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│ 3. 提取实体：_get_entities_from_relations │
│    → 找到 ["CEO", "CTO", "VP"]        │
└───────────────────────────────────────┘
```

**适用场景**：

- 查询宏观主题（"公司的发展战略"）
- 查询关系网络（"团队的协作模式"）
- 查询趋势分析（"市场的变化趋势"）

## Hybrid 模式：轮询合并

Hybrid 模式同时执行 local 和 global，然后轮询合并结果。

**源码位置**: `lightrag/operate.py:3470-3520`

```python
# hybrid mode: 同时执行 local + global
if len(ll_keywords) > 0:
    local_entities, local_relations = await _get_node_data(
        ll_keywords, knowledge_graph_inst, entities_vdb, query_param,
    )
if len(hl_keywords) > 0:
    global_relations, global_entities = await _get_edge_data(
        hl_keywords, knowledge_graph_inst, relationships_vdb, query_param,
    )

# Round-robin 合并实体（交替取 local/global）
final_entities = []
seen_entities = set()
max_len = max(len(local_entities), len(global_entities))
for i in range(max_len):
    if i < len(local_entities):
        entity = local_entities[i]
        entity_name = entity.get("entity_name")
        if entity_name and entity_name not in seen_entities:
            final_entities.append(entity)
            seen_entities.add(entity_name)

    if i < len(global_entities):
        entity = global_entities[i]
        entity_name = entity.get("entity_name")
        if entity_name and entity_name not in seen_entities:
            final_entities.append(entity)
            seen_entities.add(entity_name)
```

**轮询合并的设计意图**：

- **公平性**：local 和 global 的结果交替出现，避免一方主导。
- **去重**：通过 `seen_entities` 避免重复实体。
- **保序**：保持原始排序（向量相似度）。

## Mix 模式：最全面的检索

Mix 模式在 hybrid 基础上，额外执行向量检索，获取原始文本块。这是**推荐的默认模式**。

**源码位置**: `lightrag/operate.py:3470-3520`

```python
# mix 模式额外检索向量块
if query_param.mode == "mix" and chunks_vdb:
    vector_chunks = await _get_vector_context(
        query, chunks_vdb, query_param, query_embedding,
    )
    # 追踪向量块来源
    for i, chunk in enumerate(vector_chunks):
        chunk_id = chunk.get("chunk_id") or chunk.get("id")
        if chunk_id:
            chunk_tracking[chunk_id] = {
                "source": "C",  # C = Chunk (向量检索)
                "frequency": 1,
                "order": i + 1,
            }
```

**Mix 模式的数据来源**：

```
┌─────────────────────────────────────────────────────────────┐
│                        Mix 模式                              │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Local     │  │   Global    │  │   Vector Chunks     │  │
│  │  (实体)     │  │  (关系)     │  │   (原始文本)        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │               │                    │              │
│         └───────────────┼────────────────────┘              │
│                         ▼                                   │
│                  ┌─────────────┐                            │
│                  │   Reranker  │                            │
│                  └─────────────┘                            │
│                         │                                   │
│                         ▼                                   │
│                  ┌─────────────┐                            │
│                  │  LLM 生成   │                            │
│                  └─────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

**为什么推荐 Mix 模式**：

1. **覆盖全面**：结合 KG 结构化知识和原始文本。
2. **Reranker 增强**：多来源数据通过 Reranker 重排序，提升相关性。
3. **容错性强**：即使 KG 提取不完整，向量检索也能补充。

## Naive 模式：纯向量检索

Naive 模式不使用知识图谱，直接进行向量检索。适用于没有构建 KG 或 KG 质量不高的场景。

**源码位置**: `lightrag/operate.py:4756-4996`

```python
async def naive_query(
    query: str,
    chunks_vdb: BaseVectorStorage,
    query_param: QueryParam,
    global_config: dict[str, str],
    ...
) -> QueryResult | None:
    # 1. 向量检索
    chunks = await _get_vector_context(query, chunks_vdb, query_param, None)

    if chunks is None or len(chunks) == 0:
        return None

    # 2. Token 预算计算
    available_chunk_tokens = max_total_tokens - 1500  # 预留 LLM 响应空间

    # 3. Rerank + 截断
    processed_chunks = await process_chunks_unified(
        query=query,
        unique_chunks=chunks,
        query_param=query_param,
        source_type="vector",
        chunk_token_limit=available_chunk_tokens,
    )

    # 4. LLM 生成
    response = await use_model_func(user_query, system_prompt=sys_prompt, ...)

    return QueryResult(content=response, raw_data=raw_data)
```

**与 Mix 模式的差异**：

| 维度 | Naive | Mix |
|------|-------|-----|
| 数据源 | 仅向量检索 | KG + 向量检索 |
| 关键词提取 | 无 | 有（双层级） |
| 图谱检索 | 无 | 有（local + global） |
| 适用场景 | 简单问答 | 复杂推理 |

## Token 预算控制

LightRAG 有精确的 Token 预算控制系统，避免 LLM 上下文溢出。

**源码位置**: `lightrag/operate.py:3692-3721`

```python
async def _apply_token_truncation(
    search_result: dict,
    query_param: QueryParam,
    global_config: dict,
):
    tokenizer = global_config["tokenizer"]

    # 1. 获取 Token 限制
    max_entity_tokens = query_param.max_entity_tokens or 6000
    max_relation_tokens = query_param.max_relation_tokens or 8000
    max_total_tokens = query_param.max_total_tokens or 30000

    # 2. 截断实体
    entity_tokens = 0
    truncated_entities = []
    for entity in search_result["entities"]:
        entity_text = json.dumps(entity, ensure_ascii=False)
        tokens = len(tokenizer.encode(entity_text))
        if entity_tokens + tokens <= max_entity_tokens:
            truncated_entities.append(entity)
            entity_tokens += tokens
        else:
            break

    # 3. 截断关系（类似逻辑）
    # 4. 截断块（类似逻辑）
```

**预算分配策略**：

```
总预算 30000 Token
├── 实体预算：6000 Token
├── 关系预算：8000 Token
├── 块预算：剩余空间
└── LLM 响应预留：1500 Token
```

## Reranker 集成

Reranker 对初步检索结果进行二次排序，显著提升检索质量。

**调用方式**（`lightrag/operate.py:3950-3957`）：

```python
truncated_chunks = await process_chunks_unified(
    query=query,
    unique_chunks=merged_chunks,
    query_param=query_param,
    global_config=global_config,
    source_type=query_param.mode,
    chunk_token_limit=available_chunk_tokens,
)
```

**Rerank 触发条件**：

- `query_param.enable_rerank = True`（默认启用）
- `global_config["rerank_func"]` 已配置

**支持的 Reranker**：

- BAAI/bge-reranker-v2-m3
- Jina rerankers

## 查询模式选择指南

### 场景 1：特定实体查询

**问题**："苹果公司的 CEO 是谁？"

**推荐模式**：local

**原因**：问题聚焦于特定实体（苹果公司），local 模式能精准定位。

### 场景 2：宏观主题查询

**问题**："公司的发展战略是什么？"

**推荐模式**：global

**原因**：问题涉及宏观主题，global 模式能捕获关系网络。

### 场景 3：复杂推理查询

**问题**："张三在公司的角色是什么？他和哪些项目有关？"

**推荐模式**：mix

**原因**：问题涉及实体（张三）和关系（项目），mix 模式最全面。

### 场景 4：简单问答

**问题**："什么是机器学习？"

**推荐模式**：naive

**原因**：问题简单，不需要 KG 增强，向量检索足够。

---

## 章节衔接

**本章回顾**：
- 我们学习了五种检索模式的实现原理
- 关键收获：双层级关键词、轮询合并、Token 预算控制

**下一章预告**：
- 在 `04-llm.md` 中，我们将学习 LLM 提供商集成
- 为什么需要学习：LLM 是 RAG 的核心，集成方式影响成本和质量
- 关键问题：如何抽象多个提供商？缓存策略是什么？Prompt 如何设计？
