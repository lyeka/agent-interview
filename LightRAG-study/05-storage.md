# 四层存储架构

LightRAG 的存储层是一个设计精妙的多层抽象系统，支持 15+ 存储后端，从 JSON 文件到 Neo4j 无缝切换。本章深入分析存储抽象设计和多租户隔离机制。

## 四层分离设计

LightRAG 将存储分为四层，每层专注于特定职责：

```
┌─────────────────────────────────────────────────────────────┐
│                    LightRAG 存储架构                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ KV Storage  │  │Vector Storage│  │   Graph Storage    │  │
│  │  (6 实例)   │  │  (3 实例)    │  │    (1 实例)        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │               │                    │              │
│         └───────────────┼────────────────────┘              │
│                         ▼                                   │
│                ┌─────────────────┐                          │
│                │ Doc Status      │                          │
│                │ Storage (1实例) │                          │
│                └─────────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

**设计意图**：四层分离让每种存储专注于自己的职责，可以独立选择最适合的后端。

## 存储注册表机制

**源码位置**: `lightrag/kg/__init__.py`

```python
STORAGE_IMPLEMENTATIONS = {
    "KV_STORAGE": {
        "implementations": ["JsonKVStorage", "RedisKVStorage", "PGKVStorage", "MongoKVStorage"],
        "required_methods": ["get_by_id", "upsert"],
    },
    "GRAPH_STORAGE": {
        "implementations": ["NetworkXStorage", "Neo4JStorage", "PGGraphStorage"],
        "required_methods": ["upsert_node", "upsert_edge"],
    },
    "VECTOR_STORAGE": {
        "implementations": ["NanoVectorDBStorage", "MilvusVectorDBStorage", "QdrantStorage"],
        "required_methods": ["query", "upsert"],
    },
    "DOC_STATUS_STORAGE": {
        "implementations": ["JsonDocStatusStorage", "RedisDocStatusStorage"],
        "required_methods": ["get_docs_by_status"],
    },
}

# 模块路径映射
STORAGES = {
    "NetworkXStorage": ".kg.networkx_impl",
    "Neo4JStorage": ".kg.neo4j_impl",
    "PGKVStorage": ".kg.postgres_impl",
    "PGVectorStorage": ".kg.postgres_impl",
    "PGGraphStorage": ".kg.postgres_impl",
}
```

**设计亮点**：

- **运行时动态加载**：通过字符串指定存储类型，框架自动导入。
- **接口验证**：`required_methods` 确保实现符合契约。
- **环境变量依赖声明**：每个实现声明所需的环境变量。

## 基础存储抽象

### StorageNameSpace 基类

**源码位置**: `lightrag/base.py:173-215`

```python
@dataclass
class StorageNameSpace(ABC):
    namespace: str      # 存储命名空间（如 "text_chunks"）
    workspace: str      # 多租户隔离标识
    global_config: dict[str, Any]

    async def initialize(self): pass
    async def finalize(self): pass

    @abstractmethod
    async def index_done_callback(self) -> None:
        """提交索引操作后的回调"""

    @abstractmethod
    async def drop(self) -> dict[str, str]:
        """删除所有数据并清理资源"""
```

### BaseVectorStorage 抽象

**源码位置**: `lightrag/base.py:218-352`

```python
@dataclass
class BaseVectorStorage(StorageNameSpace, ABC):
    embedding_func: EmbeddingFunc
    cosine_better_than_threshold: float = 0.2
    meta_fields: set[str] = field(default_factory=set)

    def _generate_collection_suffix(self) -> str | None:
        """根据 embedding 模型生成集合后缀"""
        model_name = self.embedding_func.model_name
        embedding_dim = self.embedding_func.embedding_dim
        safe_name = re.sub(r"[^a-zA-Z0-9_]", "_", model_name.lower())
        return f"{safe_name}_{embedding_dim}d"

    @abstractmethod
    async def query(self, query: str, top_k: int,
                   query_embedding: list[float] = None) -> list[dict]:
        """向量检索"""

    @abstractmethod
    async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
        """批量插入/更新向量"""
```

### BaseGraphStorage 抽象

**源码位置**: `lightrag/base.py:405-702`

```python
@dataclass
class BaseGraphStorage(StorageNameSpace, ABC):
    embedding_func: EmbeddingFunc

    @abstractmethod
    async def upsert_node(self, node_id: str, node_data: dict[str, str]) -> None:
        """插入/更新节点"""

    @abstractmethod
    async def upsert_edge(self, source_node_id: str, target_node_id: str,
                         edge_data: dict[str, str]) -> None:
        """插入/更新边（无向图）"""

    # 批量操作优化（默认实现为逐个调用）
    async def get_nodes_batch(self, node_ids: list[str]) -> dict[str, dict]:
        """批量获取节点（子类可重写以优化性能）"""
        result = {}
        for node_id in node_ids:
            node = await self.get_node(node_id)
            if node is not None:
                result[node_id] = node
        return result
```

## 默认实现：JSON + NetworkX

### JsonKVStorage

**源码位置**: `lightrag/kg/json_kv_impl.py`

```python
def __post_init__(self):
    working_dir = self.global_config["working_dir"]
    if self.workspace:
        workspace_dir = os.path.join(working_dir, self.workspace)
    else:
        workspace_dir = working_dir

    os.makedirs(workspace_dir, exist_ok=True)
    self._file_name = os.path.join(workspace_dir, f"kv_store_{self.namespace}.json")
```

**多租户隔离**：基于文件路径的 workspace 隔离。

**批量 Upsert 与时间戳管理**：

```python
async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
    if not data:
        return

    current_time = int(time.time())

    async with self._storage_lock:
        for k, v in data.items():
            # 时间戳管理
            if k in self._data:  # 更新
                v["update_time"] = current_time
            else:  # 新建
                v["create_time"] = current_time
                v["update_time"] = current_time

            v["_id"] = k

        self._data.update(data)
        await set_all_update_flags(self.namespace, workspace=self.workspace)
```

### NetworkXStorage

**源码位置**: `lightrag/kg/networkx_impl.py`

```python
def __post_init__(self):
    self._graphml_xml_file = os.path.join(workspace_dir, f"graph_{self.namespace}.graphml")

    # 预加载图数据
    preloaded_graph = NetworkXStorage.load_nx_graph(self._graphml_xml_file)
    if preloaded_graph is not None:
        logger.info(f"Loaded graph with {preloaded_graph.number_of_nodes()} nodes")
    else:
        logger.info(f"Created new empty graph")
    self._graph = preloaded_graph or nx.Graph()
```

**跨进程图更新检测**：

```python
async def _get_graph(self):
    async with self._storage_lock:
        # 检查其他进程是否修改了图
        if self.storage_updated.value:
            logger.info(f"Reloading graph due to modifications by another process")
            self._graph = NetworkXStorage.load_nx_graph(self._graphml_xml_file) or nx.Graph()
            self.storage_updated.value = False

        return self._graph
```

## 生产级实现：Neo4j

### 连接池配置

**源码位置**: `lightrag/kg/neo4j_impl.py:125-191`

```python
async def initialize(self):
    URI = os.environ.get("NEO4J_URI")
    USERNAME = os.environ.get("NEO4J_USERNAME")
    PASSWORD = os.environ.get("NEO4J_PASSWORD")
    MAX_CONNECTION_POOL_SIZE = int(os.environ.get("NEO4J_MAX_CONNECTION_POOL_SIZE", 100))

    self._driver: AsyncDriver = AsyncGraphDatabase.driver(
        URI,
        auth=(USERNAME, PASSWORD),
        max_connection_pool_size=MAX_CONNECTION_POOL_SIZE,
        connection_timeout=CONNECTION_TIMEOUT,
        max_transaction_retry_time=MAX_TRANSACTION_RETRY_TIME,
    )
```

### Workspace 隔离策略

Neo4j 使用 Label 实现多租户隔离：

```python
def _get_workspace_label(self) -> str:
    return self.workspace  # 用作 Neo4j Label
```

### 批量操作优化（UNWIND）

```python
async def get_nodes_batch(self, node_ids: list[str]) -> dict[str, dict]:
    workspace_label = self._get_workspace_label()
    async with self._driver.session(database=self._DATABASE) as session:
        query = f"""
        UNWIND $node_ids AS id
        MATCH (n:`{workspace_label}` {{entity_id: id}})
        RETURN n.entity_id AS entity_id, n
        """
        result = await session.run(query, node_ids=node_ids)
        nodes = {}
        async for record in result:
            entity_id = record["entity_id"]
            nodes[entity_id] = dict(record["n"])
        return nodes
```

### 重试机制

```python
READ_RETRY = retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10),
    retry=retry_if_exception_type(READ_RETRY_EXCEPTIONS),
)

@READ_RETRY
async def get_nodes_batch(self, node_ids: list[str]) -> dict[str, dict]:
    # ... 自动重试逻辑
```

## 生产级实现：PostgreSQL 全家桶

PostgreSQL 实现是最完整的，支持 KV、Vector、Graph 三种存储类型。

### 连接池管理

**源码位置**: `lightrag/kg/postgres_impl.py:128-196`

```python
class PostgreSQLDB:
    def __init__(self, config: dict[str, Any]):
        self.host = config["host"]
        self.port = config["port"]
        self.user = config["user"]
        self.password = config["password"]
        self.database = config["database"]
        self.workspace = config["workspace"]
        self.max = int(config["max_connections"])
        self.pool: Pool | None = None

        # 向量索引配置
        self.vector_index_type = config.get("vector_index_type")
        self.hnsw_m = config.get("hnsw_m")
        self.hnsw_ef = config.get("hnsw_ef")
```

### 索引名称安全处理

PostgreSQL 标识符有 63 字节限制，需要特殊处理：

```python
def _safe_index_name(table_name: str, index_suffix: str) -> str:
    full_name = f"idx_{table_name.lower()}_{index_suffix}"

    if len(full_name.encode("utf-8")) <= PG_MAX_IDENTIFIER_LENGTH:
        return full_name

    # 超长则使用 MD5 哈希
    hash_input = table_name.lower().encode("utf-8")
    table_hash = hashlib.md5(hash_input).hexdigest()[:12]
    shortened_name = f"idx_{table_hash}_{index_suffix}"
    return shortened_name
```

### Dollar-Quoting 防注入

```python
def _dollar_quote(s: str, tag_prefix: str = "AGE") -> str:
    """生成 PostgreSQL dollar-quoted 字符串，防止内容中的 $ 符号破坏查询"""
    s = "" if s is None else str(s)
    for i in itertools.count(1):
        tag = f"{tag_prefix}{i}"
        wrapper = f"${tag}$"
        if wrapper not in s:
            return f"{wrapper}{s}{wrapper}"
```

### 图操作（Apache AGE）

PostgreSQL 通过 Apache AGE 扩展支持图操作：

```python
async def upsert_node(self, node_id: str, node_data: dict[str, str]) -> None:
    label = self._normalize_node_id(node_id)
    properties = self._format_properties(node_data)

    cypher_query = f"""MERGE (n:base {{entity_id: "{label}"}})
                 SET n += {properties}
                 RETURN n"""

    query = f"SELECT * FROM cypher({_dollar_quote(self.graph_name)}, {_dollar_quote(cypher_query)}) AS (n agtype)"

    await self._query(query, readonly=False, upsert=True)
```

## 向量存储：NanoVectorDB

### 向量压缩存储

**源码位置**: `lightrag/kg/nano_vector_db_impl.py:96-142`

```python
async def upsert(self, data: dict[str, dict[str, Any]]) -> None:
    # 批量 embedding
    embeddings = await self.embedding_func(contents)

    for i, d in enumerate(list_data):
        # 压缩向量：Float16 + zlib + Base64
        vector_f16 = embeddings[i].astype(np.float16)
        compressed_vector = zlib.compress(vector_f16.tobytes())
        encoded_vector = base64.b64encode(compressed_vector).decode("utf-8")
        d["vector"] = encoded_vector
        d["__vector__"] = embeddings[i]

    client = await self._get_client()
    results = client.upsert(datas=list_data)
```

**压缩策略**：

- **Float16**：精度损失可接受，存储减半
- **zlib**：进一步压缩
- **Base64**：JSON 序列化友好

### 查询优化

```python
async def query(self, query: str, top_k: int,
               query_embedding: list[float] = None) -> list[dict[str, Any]]:
    # 使用预计算的 embedding（避免重复计算）
    if query_embedding is not None:
        embedding = query_embedding
    else:
        embedding = await self.embedding_func([query], _priority=5)  # 高优先级
        embedding = embedding[0]

    client = await self._get_client()
    results = client.query(
        query=embedding,
        top_k=top_k,
        better_than_threshold=self.cosine_better_than_threshold,
    )
```

## 共享存储逻辑

### 多进程锁管理

**源码位置**: `lightrag/kg/shared_storage.py:137-257`

```python
class UnifiedLock(Generic[T]):
    def __init__(self, lock, is_async: bool, async_lock=None):
        self._lock = lock
        self._is_async = is_async
        self._async_lock = async_lock  # 多进程模式下的协程同步锁

    async def __aenter__(self):
        # 多进程模式：先获取 async_lock（防止事件循环阻塞）
        if not self._is_async and self._async_lock is not None:
            await self._async_lock.acquire()

        # 获取主锁
        if self._is_async:
            await self._lock.acquire()
        else:
            self._lock.acquire()  # 同步调用

        return self
```

### Workspace 命名空间管理

```python
def get_final_namespace(namespace: str, workspace: str | None = None):
    global _default_workspace
    if workspace is None:
        workspace = _default_workspace

    if workspace is None:
        raise ValueError("Invoke namespace operation without workspace")

    final_namespace = f"{workspace}:{namespace}" if workspace else f"{namespace}"
    return final_namespace
```

## 存储后端选择指南

| 场景 | KV | Vector | Graph | 推荐理由 |
|------|-----|--------|-------|----------|
| 开发/测试 | JSON | NanoVectorDB | NetworkX | 零依赖，开箱即用 |
| 小规模生产 | Redis | Qdrant | NetworkX | 性能好，部署简单 |
| 中规模生产 | PostgreSQL | PostgreSQL | PostgreSQL | 统一后端，运维简单 |
| 大规模生产 | MongoDB | Milvus | Neo4j | 各自领域最强 |

---

## 章节衔接

**本章回顾**：
- 我们学习了四层存储架构和多租户隔离机制
- 关键收获：存储注册表、抽象基类、批量操作优化

**下一章预告**：
- 在 `06-deploy.md` 中，我们将学习 API 服务与生产部署
- 为什么需要学习：生产部署是框架落地的关键
- 关键问题：FastAPI 如何配置？JWT 认证如何实现？Gunicorn 如何优化？
