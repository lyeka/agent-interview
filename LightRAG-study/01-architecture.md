# LightRAG 整体架构

## 项目定位与技术栈

LightRAG 是一个**知识图谱增强的轻量级 RAG 框架**，由香港大学数据科学实验室（HKUDS）开发。它的核心创新在于将传统向量检索与知识图谱结合，通过实体-关系提取构建结构化知识表示。

**技术栈**：
```
Python >= 3.10 + aiohttp + networkx + nano-vectordb + pydantic + tenacity + tiktoken
```

**核心依赖**：
- **异步框架**：aiohttp（HTTP 客户端）、asyncio（并发控制）
- **图存储**：networkx（默认）、neo4j、memgraph、postgresql
- **向量存储**：nano-vectordb（默认）、milvus、qdrant、faiss
- **数据验证**：pydantic（类型安全）
- **容错机制**：tenacity（重试策略）
- **Token 计数**：tiktoken（精确控制）

## 目录结构

```
lightrag/
├── __init__.py                 # 导出 LightRAG, QueryParam
├── lightrag.py                 # 核心编排类（4500+ 行）
├── operate.py                  # 核心操作逻辑（4500+ 行）
├── base.py                     # 抽象基类（30KB）
├── prompt.py                   # Prompt 模板（29KB）
├── utils.py                    # 工具函数（123KB）
├── utils_graph.py              # 图操作工具（71KB）
├── rerank.py                   # Reranker 集成（20KB）
├── types.py                    # Pydantic 模型
├── constants.py                # 常量定义
├── exceptions.py               # 异常类
├── namespace.py                # 命名空间管理
│
├── kg/                         # 存储实现（15 个文件）
│   ├── __init__.py             # 存储注册表 STORAGES
│   ├── json_kv_impl.py         # JSON KV 存储
│   ├── nano_vector_db_impl.py  # NanoVectorDB（默认）
│   ├── networkx_impl.py        # NetworkX 图（默认）
│   ├── neo4j_impl.py           # Neo4j 图（81KB）
│   ├── postgres_impl.py        # PostgreSQL 全家桶（245KB）
│   └── shared_storage.py       # 共享存储逻辑（64KB）
│
├── llm/                        # LLM 提供商（16 个文件）
│   ├── openai.py               # OpenAI（41KB）
│   ├── anthropic.py            # Anthropic（10KB）
│   ├── ollama.py               # Ollama（8KB）
│   └── binding_options.py      # 绑定选项（30KB）
│
├── api/                        # FastAPI 服务（14 个文件）
│   ├── lightrag_server.py      # 主服务器（60KB）
│   ├── config.py               # 配置管理（20KB）
│   ├── auth.py                 # JWT 认证（3KB）
│   └── routers/                # 路由模块
│
└── evaluation/                 # 评估模块
    └── eval_rag_quality.py     # RAGAS 集成
```

## LightRAG 核心类设计

LightRAG 的核心是一个用 `@dataclass` 装饰的 final 类，包含 60+ 个配置字段。这种设计体现了"配置即代码"的理念——所有配置都是类型安全的，支持 IDE 自动补全。

**源码位置**: `lightrag/lightrag.py:128-443`

```python
@final
@dataclass
class LightRAG:
    # ========== 存储配置 ==========
    working_dir: str = "./rag_storage"
    kv_storage: str = "JsonKVStorage"
    vector_storage: str = "NanoVectorDBStorage"
    graph_storage: str = "NetworkXStorage"
    doc_status_storage: str = "JsonDocStatusStorage"

    # ========== 多租户隔离 ==========
    workspace: str = field(default_factory=lambda: os.getenv("WORKSPACE", ""))

    # ========== 查询参数 ==========
    top_k: int = 60                    # 向量检索返回数量
    chunk_top_k: int = 20              # 块检索返回数量
    max_entity_tokens: int = 6000      # 实体 Token 预算
    max_relation_tokens: int = 8000    # 关系 Token 预算
    max_total_tokens: int = 30000      # 总 Token 预算

    # ========== LLM/Embedding 配置 ==========
    llm_model_func: Callable | None = None
    embedding_func: EmbeddingFunc | None = None
    llm_model_max_async: int = 8       # LLM 并发数
    embedding_func_max_async: int = 8  # Embedding 并发数
```

**设计亮点**：

1. **@final 装饰器**：禁止继承，这是一个封闭设计。LightRAG 不鼓励通过继承扩展，而是通过配置和依赖注入实现定制。

2. **@dataclass**：字段即配置，声明式风格。所有配置都有类型注解和默认值，IDE 可以提供完整的自动补全。

3. **环境变量优先**：所有配置都支持通过 `get_env_value()` 从环境变量读取，方便容器化部署。

4. **函数式注入**：`llm_model_func` 和 `embedding_func` 通过依赖注入传入，支持任意 LLM 提供商。

## 三阶段初始化流程

LightRAG 的初始化分为三个阶段，这种设计避免了构造函数中的异步陷阱。

### Phase 1: __post_init__（同步）

**源码位置**: `lightrag/lightrag.py:444-668`

```python
def __post_init__(self):
    # 1. 创建工作目录
    if not os.path.exists(self.working_dir):
        os.makedirs(self.working_dir)

    # 2. 验证存储实现兼容性
    for storage_type, storage_name in storage_configs:
        verify_storage_implementation(storage_type, storage_name)
        check_storage_env_vars(storage_name)

    # 3. 初始化 Tokenizer
    if self.tokenizer is None:
        self.tokenizer = TiktokenTokenizer(self.tiktoken_model_name)

    # 4. 包装 Embedding 函数（并发控制）
    wrapped_func = priority_limit_async_func_call(
        self.embedding_func_max_async,
        llm_timeout=self.default_embedding_timeout,
    )(self.embedding_func.func)
    self.embedding_func = replace(self.embedding_func, func=wrapped_func)
```

**关键设计决策**：

- **不可变性保护**：使用 `dataclasses.replace()` 创建新的 EmbeddingFunc 实例，避免污染调用者的对象。
- **并发控制**：通过 `priority_limit_async_func_call` 装饰器限制并发数，防止 API 限流。

### Phase 2: initialize_storages()（异步）

**源码位置**: `lightrag/lightrag.py:670-708`

```python
async def initialize_storages(self):
    if self._storages_status == StoragesStatus.CREATED:
        # 1. 设置默认 workspace
        default_workspace = get_default_workspace()
        if default_workspace is None:
            set_default_workspace(self.workspace)

        # 2. 初始化 pipeline_status
        await initialize_pipeline_status(workspace=self.workspace)

        # 3. 逐个初始化存储
        for storage in (self.full_docs, self.text_chunks, ...):
            await storage.initialize()

        self._storages_status = StoragesStatus.INITIALIZED
```

**为什么必须手动调用**：

- **防止死锁**：存储初始化涉及文件锁、数据库连接，必须串行执行。
- **明确生命周期**：用户显式控制初始化时机，避免构造函数中的异步陷阱。

### Phase 3: check_and_migrate_data()（可选）

**源码位置**: `lightrag/lightrag.py:758-1057`

这个阶段处理数据迁移，支持从旧版本数据结构平滑升级。迁移策略是批量处理（每批 500 条），保证幂等性。

## 12 个存储实例

LightRAG 在初始化时创建 12 个存储实例，分为四类：

```python
# KV 存储（6 个）
self.llm_response_cache: BaseKVStorage       # LLM 响应缓存
self.text_chunks: BaseKVStorage              # 文本块存储
self.full_docs: BaseKVStorage                # 完整文档存储
self.full_entities: BaseKVStorage            # 实体元数据
self.full_relations: BaseKVStorage           # 关系元数据
self.entity_chunks: BaseKVStorage            # 实体-块映射
self.relation_chunks: BaseKVStorage          # 关系-块映射

# 向量存储（3 个）
self.entities_vdb: BaseVectorStorage         # 实体向量库
self.relationships_vdb: BaseVectorStorage    # 关系向量库
self.chunks_vdb: BaseVectorStorage           # 块向量库

# 图存储（1 个）
self.chunk_entity_relation_graph: BaseGraphStorage  # 知识图谱

# 文档状态存储（1 个）
self.doc_status: DocStatusStorage            # 文档处理状态追踪
```

**设计意图**：四层分离让每种存储专注于自己的职责，可以独立选择最适合的后端。例如，KV 存储用 Redis，向量存储用 Milvus，图存储用 Neo4j。

## 核心数据流

### 文档插入流程

```
用户调用 ainsert(content)
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 1: 文档入队 (apipeline_enqueue_documents)               │
│   - 生成 doc_id（MD5 hash 或用户指定）                         │
│   - 去重检查（已存在的文档标记为 FAILED）                       │
│   - 存储文档内容和状态                                         │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 2: 文档处理 (apipeline_process_enqueue_documents)       │
│   - 获取 pipeline 锁（防止并发冲突）                           │
│   - 文本分块（Token-aware 滑动窗口）                           │
│   - 存储块数据到 text_chunks 和 chunks_vdb                     │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 3: 实体关系提取 (_process_extract_entities)             │
│   - LLM 调用提取实体和关系                                     │
│   - Multi-gleaning（迭代补充遗漏实体）                         │
│   - 解析 LLM 输出为结构化数据                                  │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 4: 知识图谱合并 (merge_nodes_and_edges)                 │
│   - 实体去重和合并                                             │
│   - 关系去重和合并                                             │
│   - 更新向量索引                                               │
│   - 更新文档状态为 COMPLETED                                   │
└───────────────────────────────────────────────────────────────┘
```

### 查询流程

```
用户调用 aquery(query, param)
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 1: 关键词提取 (get_keywords_from_query)                 │
│   - LLM 提取 high_level_keywords（主题、概念）                 │
│   - LLM 提取 low_level_keywords（实体、专有名词）              │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 2: 数据检索 (_perform_kg_search)                        │
│   - local 模式：实体向量检索 → 图谱扩展                        │
│   - global 模式：关系向量检索 → 社区摘要                       │
│   - hybrid 模式：local + global 轮询合并                       │
│   - mix 模式：hybrid + 向量检索                                │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 3: Token 截断 (_apply_token_truncation)                 │
│   - 按 Token 预算截断实体/关系/块                              │
│   - Reranker 重排序（如果启用）                                │
└───────────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│ Phase 4: LLM 生成 (use_model_func)                            │
│   - 构建 RAG Prompt（Context + Query）                         │
│   - 调用 LLM 生成回答                                          │
│   - 支持流式/非流式响应                                        │
└───────────────────────────────────────────────────────────────┘
```

## 设计模式分析

### 1. 策略模式（Storage Backends）

存储后端通过注册表动态加载，统一接口，可插拔架构：

```python
# kg/__init__.py
STORAGES = {
    "JsonKVStorage": "lightrag.kg.json_kv_impl",
    "NanoVectorDBStorage": "lightrag.kg.nano_vector_db_impl",
    "NetworkXStorage": "lightrag.kg.networkx_impl",
    "Neo4JStorage": "lightrag.kg.neo4j_impl",
    "PGKVStorage": "lightrag.kg.postgres_impl",
}
```

### 2. 模板方法模式（Query Pipeline）

查询流程是固定的模板，但每个步骤可以定制：

```python
async def kg_query(...):
    # 1. 关键词提取（可配置）
    keywords = await get_keywords_from_query(query)

    # 2. 向量检索（可配置 top_k）
    context = await _get_vector_context(...)

    # 3. KG 检索（可配置模式）
    kg_context = await _perform_kg_search(...)

    # 4. Token 截断（可配置预算）
    final_context = await _apply_token_truncation(...)

    # 5. LLM 生成（可配置模型）
    response = await llm_model_func(prompt)
```

### 3. 装饰器模式（Caching & Retry）

通过装饰器实现横切关注点：

```python
@wrap_embedding_func_with_attrs(embedding_dim=1536, max_token_size=8192)
async def custom_embed(texts: list[str]) -> np.ndarray:
    ...

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10),
    retry=retry_if_exception_type(RateLimitError)
)
async def openai_complete_if_cache(...):
    ...
```

## 与其他框架对比

### vs. LangChain

| 维度 | LightRAG | LangChain |
|------|----------|-----------|
| **定位** | 专注 KG-RAG | 通用编排框架 |
| **复杂度** | 轻量级（单一职责） | 重量级（全能型） |
| **知识图谱** | 核心特性 | 需额外集成 |
| **学习曲线** | 陡峭（需理解 KG） | 平缓（文档丰富） |
| **性能** | 高（异步优先） | 中（同步为主） |

### vs. GraphRAG（Microsoft）

| 维度 | LightRAG | GraphRAG |
|------|----------|----------|
| **架构** | 轻量级（单机友好） | 重型（分布式） |
| **社区检测** | 可选 | 核心（Leiden 算法） |
| **成本** | 低（LLM 调用少） | 高（大量 LLM 调用） |
| **部署** | 简单（pip install） | 复杂（依赖多） |
| **适用场景** | 中小规模知识库 | 大规模企业知识库 |

### vs. LlamaIndex

| 维度 | LightRAG | LlamaIndex |
|------|----------|------------|
| **知识图谱** | 原生支持 | 需插件 |
| **存储抽象** | 四层分离 | 两层（Index/Store） |
| **查询模式** | 5 种 KG 特化模式 | 多种通用模式 |
| **集成性** | 独立框架 | 可集成 LightRAG |

---

## 章节衔接

**本章回顾**：
- 我们学习了 LightRAG 的整体架构和设计哲学
- 关键收获：四层存储分离、三阶段初始化、策略模式

**下一章预告**：
- 在 `02-kg.md` 中，我们将学习知识图谱构建
- 为什么需要学习：知识图谱是 LightRAG 区别于普通 RAG 的核心
- 关键问题：实体如何提取？关系如何建模？图如何合并？
