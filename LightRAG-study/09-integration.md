# 集成实战

本章提供 LightRAG 的快速上手指南、进阶配置和最佳实践，帮助你在实际项目中应用所学知识。

## 快速上手

### 安装

```bash
# 基础安装
pip install lightrag-hku

# 完整安装（包含所有可选依赖）
pip install "lightrag-hku[all]"

# 指定存储后端
pip install "lightrag-hku[neo4j]"
pip install "lightrag-hku[postgres]"
pip install "lightrag-hku[milvus]"
```

### 最小示例

```python
import asyncio
from lightrag import LightRAG, QueryParam
from lightrag.llm.openai import gpt_4o_mini_complete, openai_embed

async def main():
    # 1. 初始化 LightRAG
    rag = LightRAG(
        working_dir="./rag_storage",
        llm_model_func=gpt_4o_mini_complete,
        embedding_func=openai_embed,
    )

    # 2. 初始化存储（必须调用！）
    await rag.initialize_storages()

    # 3. 插入文档
    await rag.ainsert("苹果公司是一家美国科技公司，CEO 是 Tim Cook。")

    # 4. 查询
    response = await rag.aquery(
        "苹果公司的 CEO 是谁？",
        param=QueryParam(mode="mix")
    )
    print(response)

    # 5. 清理资源
    await rag.finalize_storages()

asyncio.run(main())
```

### 使用 Ollama 本地模型

```python
from lightrag.llm.ollama import ollama_model_complete, ollama_embed

rag = LightRAG(
    working_dir="./rag_storage",
    llm_model_func=ollama_model_complete,
    llm_model_kwargs={"model": "llama3.2"},
    embedding_func=ollama_embed,
    embedding_func_kwargs={"model": "nomic-embed-text"},
)
```

## 进阶配置

### 存储后端配置

#### Neo4j 图存储

```python
import os

# 设置环境变量
os.environ["NEO4J_URI"] = "bolt://localhost:7687"
os.environ["NEO4J_USERNAME"] = "neo4j"
os.environ["NEO4J_PASSWORD"] = "password"

rag = LightRAG(
    working_dir="./rag_storage",
    graph_storage="Neo4JStorage",
    llm_model_func=gpt_4o_mini_complete,
    embedding_func=openai_embed,
)
```

#### PostgreSQL 全家桶

```python
os.environ["POSTGRES_HOST"] = "localhost"
os.environ["POSTGRES_PORT"] = "5432"
os.environ["POSTGRES_USER"] = "postgres"
os.environ["POSTGRES_PASSWORD"] = "password"
os.environ["POSTGRES_DATABASE"] = "lightrag"

rag = LightRAG(
    working_dir="./rag_storage",
    kv_storage="PGKVStorage",
    vector_storage="PGVectorStorage",
    graph_storage="PGGraphStorage",
    doc_status_storage="PGDocStatusStorage",
    llm_model_func=gpt_4o_mini_complete,
    embedding_func=openai_embed,
)
```

#### Milvus 向量存储

```python
os.environ["MILVUS_URI"] = "http://localhost:19530"
os.environ["MILVUS_TOKEN"] = ""

rag = LightRAG(
    working_dir="./rag_storage",
    vector_storage="MilvusVectorDBStorage",
    llm_model_func=gpt_4o_mini_complete,
    embedding_func=openai_embed,
)
```

### 查询参数配置

```python
from lightrag import QueryParam

# 详细配置
param = QueryParam(
    mode="mix",                    # 检索模式
    top_k=60,                      # 向量检索返回数量
    chunk_top_k=20,                # 块检索返回数量
    max_entity_tokens=6000,        # 实体 Token 预算
    max_relation_tokens=8000,      # 关系 Token 预算
    max_total_tokens=30000,        # 总 Token 预算
    enable_rerank=True,            # 启用 Reranker
    include_references=True,       # 包含引用
    stream=False,                  # 流式响应
)

response = await rag.aquery("问题", param=param)
```

### 多租户配置

```python
# 创建不同 workspace 的实例
rag_project_a = LightRAG(
    working_dir="./rag_storage",
    workspace="project_a",
    llm_model_func=gpt_4o_mini_complete,
    embedding_func=openai_embed,
)

rag_project_b = LightRAG(
    working_dir="./rag_storage",
    workspace="project_b",
    llm_model_func=gpt_4o_mini_complete,
    embedding_func=openai_embed,
)

# 数据完全隔离
await rag_project_a.ainsert("Project A 的文档")
await rag_project_b.ainsert("Project B 的文档")
```

## API 服务部署

### 环境变量配置

```bash
# .env 文件
HOST=0.0.0.0
PORT=8000
WORKERS=4

# LLM 配置
LLM_BINDING=openai
OPENAI_API_KEY=sk-...
LLM_MODEL=gpt-4o-mini

# Embedding 配置
EMBEDDING_BINDING=openai
EMBEDDING_MODEL=text-embedding-3-small

# 存储配置
LIGHTRAG_KV_STORAGE=JsonKVStorage
LIGHTRAG_VECTOR_STORAGE=NanoVectorDBStorage
LIGHTRAG_GRAPH_STORAGE=NetworkXStorage

# 认证配置
AUTH_ACCOUNTS=admin:password123
TOKEN_SECRET=your-secret-key
TOKEN_EXPIRE_HOURS=24
```

### 启动服务

```bash
# 开发模式（单进程）
python -m lightrag.api.lightrag_server

# 生产模式（多进程）
python -m lightrag.api.run_with_gunicorn
```

### API 调用示例

```python
import requests

# 登录获取 Token
response = requests.post(
    "http://localhost:8000/login",
    data={"username": "admin", "password": "password123"}
)
token = response.json()["access_token"]

# 插入文档
headers = {"Authorization": f"Bearer {token}"}
response = requests.post(
    "http://localhost:8000/documents/text",
    headers=headers,
    json={"text": "文档内容"}
)

# 查询
response = requests.post(
    "http://localhost:8000/query",
    headers=headers,
    json={"query": "问题", "mode": "mix"}
)
print(response.json())
```

## 最佳实践

### 1. 文档预处理

```python
import re

def preprocess_document(text: str) -> str:
    # 移除多余空白
    text = re.sub(r'\s+', ' ', text)

    # 移除特殊字符
    text = re.sub(r'[^\w\s\u4e00-\u9fff.,!?;:，。！？；：]', '', text)

    # 分段处理（保持语义完整性）
    paragraphs = text.split('\n\n')
    paragraphs = [p.strip() for p in paragraphs if len(p.strip()) > 50]

    return '\n\n'.join(paragraphs)

# 使用
clean_text = preprocess_document(raw_text)
await rag.ainsert(clean_text)
```

### 2. 批量插入优化

```python
async def batch_insert(rag, documents: list[str], batch_size: int = 10):
    """批量插入文档，避免内存溢出"""
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]
        await rag.ainsert(batch)
        print(f"Inserted batch {i // batch_size + 1}")
```

### 3. 查询结果后处理

```python
async def query_with_postprocess(rag, query: str) -> dict:
    """查询并后处理结果"""
    # 获取结构化数据
    result = await rag.aquery_llm(query, QueryParam(mode="mix"))

    # 提取引用
    raw_data = result.get("raw_data", {})
    entities = raw_data.get("entities", [])
    relations = raw_data.get("relations", [])

    # 构建响应
    return {
        "answer": result["llm_response"]["content"],
        "entities": [e["entity_name"] for e in entities[:5]],
        "relations": [f"{r['src_id']} -> {r['tgt_id']}" for r in relations[:5]],
        "confidence": len(entities) > 0 and len(relations) > 0,
    }
```

### 4. 错误处理

```python
from lightrag.exceptions import LightRAGException

async def safe_query(rag, query: str) -> str:
    """带错误处理的查询"""
    try:
        response = await rag.aquery(query, QueryParam(mode="mix"))
        return response
    except LightRAGException as e:
        logger.error(f"LightRAG error: {e}")
        return "抱歉，查询失败，请稍后重试。"
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return "系统错误，请联系管理员。"
```

### 5. 性能监控

```python
import time
from functools import wraps

def monitor_performance(func):
    """性能监控装饰器"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.time()
        result = await func(*args, **kwargs)
        elapsed = time.time() - start

        logger.info(f"{func.__name__} took {elapsed:.2f}s")

        if elapsed > 5:
            logger.warning(f"{func.__name__} is slow!")

        return result
    return wrapper

@monitor_performance
async def query_with_monitoring(rag, query: str):
    return await rag.aquery(query)
```

## 常见问题

### Q: 为什么查询结果不准确？

**可能原因**：
1. 文档分块不合理（块太小丢失上下文，块太大噪音多）
2. 实体提取质量不高（检查 Prompt 配置）
3. 检索模式选择不当（尝试不同模式）

**解决方案**：
```python
# 调整分块参数
rag = LightRAG(
    chunk_token_size=1500,  # 增大块大小
    chunk_overlap_token_size=200,  # 增大重叠
)

# 启用 Reranker
param = QueryParam(mode="mix", enable_rerank=True)
```

### Q: 为什么插入文档很慢？

**可能原因**：
1. LLM 调用延迟高
2. 并发数设置不当
3. 存储后端性能问题

**解决方案**：
```python
# 增加并发数
rag = LightRAG(
    llm_model_max_async=16,  # 增加 LLM 并发
    embedding_func_max_async=16,  # 增加 Embedding 并发
)

# 使用更快的存储后端
rag = LightRAG(
    kv_storage="RedisKVStorage",
    vector_storage="MilvusVectorDBStorage",
)
```

### Q: 如何处理大文档？

**解决方案**：
```python
# 使用字符分割
await rag.ainsert(
    large_document,
    split_by_character="\n\n",  # 按段落分割
)

# 或者预处理分割
chunks = split_document(large_document, max_chars=10000)
for chunk in chunks:
    await rag.ainsert(chunk)
```

---

## 章节衔接

**本章回顾**：
- 我们学习了 LightRAG 的集成实战
- 关键收获：快速上手、进阶配置、最佳实践

**下一章预告**：
- 在 `10-summary.md` 中，我们将总结所有知识点
- 为什么需要学习：总结是巩固学习成果的最佳方式
- 关键内容：架构图、检查清单、学习路径
