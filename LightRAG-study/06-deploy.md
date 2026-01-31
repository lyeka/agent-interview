# API 服务与生产部署

LightRAG 提供完整的 FastAPI 服务，支持 JWT 认证、Gunicorn 多进程部署、Docker 容器化。本章深入分析 API 架构和生产部署配置。

## FastAPI 应用架构

### 应用工厂模式

**源码位置**: `lightrag/api/lightrag_server.py:287-1363`

```python
def create_app(args):
    # 1. LLM 配置缓存
    config_cache = LLMConfigCache(args)

    # 2. Lifespan 上下文管理器
    @asynccontextmanager
    async def lifespan(app: FastAPI):
        await rag.initialize_storages()  # 启动时初始化存储
        await rag.check_and_migrate_data()  # 数据迁移
        yield
        await rag.finalize_storages()  # 关闭时清理
        finalize_share_data()  # 清理共享数据

    # 3. FastAPI 应用初始化
    app = FastAPI(
        title="LightRAG Server API",
        version=__api_version__,
        lifespan=lifespan,
        swagger_ui_parameters={"persistAuthorization": True}
    )

    # 4. CORS 中间件
    app.add_middleware(CORSMiddleware, allow_origins=["*"])

    # 5. 路由注册
    app.include_router(create_document_routes(rag, doc_manager, api_key))
    app.include_router(create_query_routes(rag, api_key, top_k))
    app.include_router(create_graph_routes(rag, api_key))
    app.include_router(ollama_api.router, prefix="/api")

    return app
```

**设计亮点**：

- **Lifespan 管理**：统一资源初始化/清理逻辑
- **Swagger 持久化**：`persistAuthorization=True` 保持认证状态
- **路由工厂**：每个路由模块通过工厂函数创建，支持依赖注入

## JWT 认证机制

### Token 生命周期管理

**源码位置**: `lightrag/api/auth.py:23-107`

```python
class AuthHandler:
    def __init__(self):
        self.secret = global_args.token_secret
        self.algorithm = global_args.jwt_algorithm
        self.expire_hours = global_args.token_expire_hours
        self.guest_expire_hours = global_args.guest_token_expire_hours

        # 从环境变量加载账户
        self.accounts = {}
        if global_args.auth_accounts:
            for account in global_args.auth_accounts.split(","):
                username, password = account.split(":", 1)
                self.accounts[username] = password

    def create_token(self, username, role="user", custom_expire_hours=None):
        expire_hours = custom_expire_hours or (
            self.guest_expire_hours if role == "guest" else self.expire_hours
        )
        expire = datetime.utcnow() + timedelta(hours=expire_hours)

        payload = TokenPayload(sub=username, exp=expire, role=role, metadata={})
        return jwt.encode(payload.dict(), self.secret, algorithm=self.algorithm)

    def validate_token(self, token):
        payload = jwt.decode(token, self.secret, algorithms=[self.algorithm])

        if datetime.utcnow() > datetime.utcfromtimestamp(payload["exp"]):
            raise HTTPException(status_code=401, detail="Token expired")

        return {
            "username": payload["sub"],
            "role": payload.get("role", "user"),
            "exp": datetime.utcfromtimestamp(payload["exp"])
        }
```

### 三层认证策略

**源码位置**: `lightrag/api/utils_api.py:80-200`

```python
def get_combined_auth_dependency(api_key: Optional[str] = None):
    oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login", auto_error=False)
    api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

    async def combined_dependency(request, response, token, api_key_header_value):
        path = request.url.path

        # 1️⃣ 白名单路径检查
        for pattern, is_prefix in whitelist_patterns:
            if (is_prefix and path.startswith(pattern)) or path == pattern:
                return  # 直接放行

        # 2️⃣ JWT Token 验证（优先级最高）
        if token:
            token_info = auth_handler.validate_token(token)

            # Token 自动续期逻辑
            if global_args.token_auto_renew:
                expire_time = token_info.get("exp")
                remaining_seconds = (expire_time - datetime.utcnow()).total_seconds()

                if remaining_seconds < threshold_seconds:
                    new_token = auth_handler.create_token(
                        username=token_info["username"],
                        role=token_info["role"]
                    )
                    response.headers["X-New-Token"] = new_token
            return

        # 3️⃣ API Key 验证
        if api_key_header_value and api_key_header_value == api_key:
            return

        # 4️⃣ 无认证配置时放行
        if not auth_configured and not api_key_configured:
            return

        # 5️⃣ 认证失败
        raise HTTPException(status_code=403, detail="Authentication required")

    return combined_dependency
```

**认证优先级**：

```
JWT Token > API Key > 白名单 > 无认证
```

### Token 自动续期

**触发条件**：
- 剩余时间 < 总时长 × 阈值（默认 50%）
- 同一用户 60 秒内只能续期一次
- 跳过路径：`/health`, `/documents/paginated`

**返回方式**：通过 `X-New-Token` 响应头返回新 Token

## 路由设计

### 查询路由

**源码位置**: `lightrag/api/routers/query_routes.py`

```python
class QueryRequest(BaseModel):
    query: str = Field(min_length=3)
    mode: Literal["local", "global", "hybrid", "naive", "mix", "bypass"] = "mix"

    # 检索参数
    top_k: Optional[int] = Field(ge=1, default=None)
    chunk_top_k: Optional[int] = Field(ge=1, default=None)

    # Token 控制
    max_entity_tokens: Optional[int] = None
    max_relation_tokens: Optional[int] = None
    max_total_tokens: Optional[int] = None

    # 对话历史
    conversation_history: Optional[List[Dict[str, Any]]] = None

    def to_query_params(self, is_stream: bool) -> QueryParam:
        request_data = self.model_dump(exclude_none=True, exclude={"query"})
        param = QueryParam(**request_data)
        param.stream = is_stream
        return param
```

**三种查询端点**：

| 端点 | 用途 | 返回格式 |
|------|------|----------|
| `/query` | 同步查询 | 完整响应 + 引用列表 |
| `/query/stream` | 流式查询 | NDJSON（首块包含引用） |
| `/query/data` | 结构化数据 | 实体/关系/分块详情 |

### 文档路由

**文件名安全化**（防止路径遍历攻击）：

```python
def sanitize_filename(filename: str, input_dir: Path) -> str:
    # 1. 移除路径分隔符和遍历序列
    clean_name = filename.replace("/", "").replace("\\", "").replace("..", "")

    # 2. 移除控制字符
    clean_name = "".join(c for c in clean_name if ord(c) >= 32 and c != "\x7f")

    # 3. 验证最终路径在输入目录内
    final_path = (input_dir / clean_name).resolve()
    if not final_path.is_relative_to(input_dir.resolve()):
        raise HTTPException(status_code=400, detail="Unsafe filename detected")

    return clean_name
```

### 图谱路由

**实体合并请求**：

```python
class EntityMergeRequest(BaseModel):
    entities_to_change: list[str] = Field(
        description="待合并的实体列表（重复/拼写错误）",
        examples=[["Elon Msk", "Ellon Musk"]]
    )
    entity_to_change_into: str = Field(
        description="目标实体（保留）",
        examples=["Elon Musk"]
    )
```

## Gunicorn 生产部署

### 核心配置

**源码位置**: `lightrag/api/gunicorn_config.py:24-92`

```python
# 预加载应用（共享内存优化）
preload_app = True

# Uvicorn Worker（异步支持）
worker_class = "uvicorn.workers.UvicornWorker"

# 日志配置
logconfig_dict = {
    "handlers": {
        "file": {
            "class": "logging.handlers.RotatingFileHandler",
            "filename": log_file_path,
            "maxBytes": log_max_bytes,  # 默认 10MB
            "backupCount": log_backup_count,  # 默认 5 个备份
        }
    },
    "filters": {
        "path_filter": {
            "()": "lightrag.utils.LightragPathFilter"  # 过滤健康检查日志
        }
    }
}
```

### 生命周期钩子

```python
def on_starting(server):
    """主进程启动前执行"""
    print(f"Gunicorn master process: {os.getpid()}")
    print(f"Workers: {workers}")

def on_exit(server):
    """主进程关闭时执行"""
    print("Finalizing shared storage...")
    finalize_share_data()  # 清理共享内存

def post_fork(server, worker):
    """Worker 进程 fork 后执行"""
    setup_logger("uvicorn", log_level, log_file_path=log_file_path)
    setup_logger("lightrag", log_level, log_file_path=log_file_path)
```

### macOS 兼容性检查

**源码位置**: `lightrag/api/run_with_gunicorn.py:50-110`

```python
def main():
    # 1️⃣ DOCLING + 多 Worker 检查
    if (platform.system() == "Darwin"
        and global_args.document_loading_engine == "DOCLING"
        and global_args.workers > 1):
        print("❌ ERROR: DOCLING with multi-worker not supported on macOS")
        sys.exit(1)

    # 2️⃣ NumPy Accelerate 框架检查
    if (platform.system() == "Darwin"
        and global_args.workers > 1
        and os.environ.get("OBJC_DISABLE_INITIALIZE_FORK_SAFETY") != "YES"):
        print("❌ ERROR: Missing OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES")
        sys.exit(1)

    # 3️⃣ 共享数据初始化
    if global_args.workers > 1:
        os.environ["LIGHTRAG_MAIN_PROCESS"] = "1"
        initialize_share_data(global_args.workers)
```

## 配置管理

### 三层优先级

```
命令行参数 > 环境变量 > 默认值
```

**源码位置**: `lightrag/api/config.py:77-462`

```python
def parse_args():
    parser = argparse.ArgumentParser()

    # 命令行参数 > 环境变量 > 默认值
    parser.add_argument(
        "--host",
        default=get_env_value("HOST", "0.0.0.0"),
    )

    # 动态绑定选项
    llm_binding_value = get_env_value("LLM_BINDING", "ollama")
    if llm_binding_value == "ollama":
        OllamaLLMOptions.add_args(parser)
    elif llm_binding_value in ["openai", "azure_openai"]:
        OpenAILLMOptions.add_args(parser)

    return args
```

### 懒加载代理模式

```python
class _GlobalArgsProxy:
    """自动初始化配置的代理对象"""

    def __getattribute__(self, name):
        global _initialized, _global_args

        # 首次访问时自动初始化
        if not _initialized:
            initialize_config()
        return getattr(_global_args, name)

# 全局配置实例
global_args = _GlobalArgsProxy()
```

## Docker 部署

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["python", "-m", "lightrag.api.run_with_gunicorn"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  lightrag:
    build: .
    ports:
      - "8000:8000"
    environment:
      - LLM_BINDING=openai
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - LIGHTRAG_KV_STORAGE=RedisKVStorage
      - REDIS_URI=redis://redis:6379
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

## 部署模式对比

| 模式 | 适用场景 | 优势 | 劣势 |
|------|----------|------|------|
| Uvicorn 单进程 | 开发/测试 | 简单、调试方便 | 无法利用多核 |
| Gunicorn 多进程 | 生产 | 高并发、容错 | 配置复杂 |
| Docker 单容器 | 小规模生产 | 隔离、可移植 | 资源受限 |
| K8s 集群 | 大规模生产 | 弹性伸缩 | 运维复杂 |

---

## 章节衔接

**本章回顾**：
- 我们学习了 FastAPI 服务架构和生产部署配置
- 关键收获：JWT 认证、Gunicorn 配置、Docker 部署

**下一章预告**：
- 在 `07-eval.md` 中，我们将学习评估与可观测性
- 为什么需要学习：评估是 RAG 质量保证的关键
- 关键问题：如何评估检索质量？如何集成 Langfuse？
