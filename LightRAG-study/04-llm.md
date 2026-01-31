# LLM 集成与 Prompt 工程

LightRAG 支持 16+ 个 LLM 提供商，通过函数式接口和装饰器模式实现提供商无关的抽象。本章深入分析 LLM 集成架构和 Prompt 设计。

## 提供商抽象架构

LightRAG 没有使用传统的类继承体系，而是通过**函数式接口 + 装饰器模式**实现提供商抽象。

### 统一函数签名

所有提供商的 LLM 调用函数遵循相同签名：

```python
async def {provider}_complete_if_cache(
    model: str,
    prompt: str,
    system_prompt: str | None = None,
    history_messages: list[dict[str, Any]] | None = None,
    enable_cot: bool = False,
    base_url: str | None = None,
    api_key: str | None = None,
    stream: bool | None = None,
    timeout: int | None = None,
    **kwargs: Any,
) -> Union[str, AsyncIterator[str]]:
```

### Embedding 函数签名

```python
@wrap_embedding_func_with_attrs(embedding_dim=X, max_token_size=Y, model_name="...")
@retry(...)
async def {provider}_embed(
    texts: list[str],
    model: str = "...",
    api_key: str | None = None,
    max_token_size: int | None = None,
    embedding_dim: int | None = None,
    **kwargs,
) -> np.ndarray:
```

## 重试策略：指数退避 + 异常分类

**源码位置**: `lightrag/llm/openai.py:187-196`

```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10),
    retry=(
        retry_if_exception_type(RateLimitError)
        | retry_if_exception_type(APIConnectionError)
        | retry_if_exception_type(APITimeoutError)
        | retry_if_exception_type(InvalidResponseError)
    ),
)
async def openai_complete_if_cache(...):
```

**重试策略本质**：

- **可重试异常**：网络、限流、超时（瞬态故障）
- **不可重试异常**：认证、参数错误（永久故障）
- **退避算法**：`wait = min(10, 4 * 2^attempt)`，避免雪崩

**异常分类示例**（Bedrock）：

```python
# 可重试
if error_code in ["ThrottlingException", "ProvisionedThroughputExceededException"]:
    raise BedrockRateLimitError(f"Rate limit error: {error_msg}")

# 可重试
elif error_code in ["ServiceUnavailableException", "InternalServerException"]:
    raise BedrockConnectionError(f"Service error: {error_msg}")

# 不可重试
else:
    raise BedrockError(f"Client error: {error_msg}")
```

## 流式响应处理

LightRAG 支持流式响应，通过异步生成器实现。

**源码位置**: `lightrag/llm/openai.py:358-446`

```python
async def inner():
    # COT (Chain of Thought) 状态追踪
    cot_active = False
    cot_started = False

    try:
        async for chunk in response:
            delta = chunk.choices[0].delta
            content = getattr(delta, "content", None)
            reasoning_content = getattr(delta, "reasoning_content", "")

            # 处理 COT 逻辑
            if enable_cot:
                if content:
                    if cot_active:
                        yield "</think>"  # 关闭思考标签
                        cot_active = False
                    yield content
                elif reasoning_content:
                    if not cot_active:
                        yield "<think>"  # 开启思考标签
                        cot_active = True
                    yield reasoning_content
```

**设计精髓**：

- **状态机模式**：`cot_active` 追踪当前是否在思考模式。
- **标签注入**：动态插入 `<think>` 标签，无需修改模型输出。
- **优雅降级**：`enable_cot=False` 时完全忽略 reasoning_content。

## Embedding 函数包装器

**源码位置**: `lightrag/utils.py:411-527`

```python
class EmbeddingFunc:
    embedding_dim: int
    func: callable
    max_token_size: int | None = None
    send_dimensions: bool = False
    model_name: str | None = None

    def __post_init__(self):
        """防止双重包装"""
        max_unwrap_depth = 3
        unwrap_count = 0
        while isinstance(self.func, EmbeddingFunc):
            unwrap_count += 1
            if unwrap_count > max_unwrap_depth:
                raise ValueError("Possible circular reference detected.")
            self.func = self.func.func  # 递归解包

    async def __call__(self, *args, **kwargs) -> np.ndarray:
        # 1. 参数注入
        if self.send_dimensions:
            kwargs["embedding_dim"] = self.embedding_dim

        # 2. 调用底层函数
        result = await self.func(*args, **kwargs)

        # 3. 维度验证
        total_elements = result.size
        if total_elements % self.embedding_dim != 0:
            raise ValueError(f"Embedding dimension mismatch")

        return result
```

**双重装饰器的协作**：

1. `@wrap_embedding_func_with_attrs`：外层，提供属性和参数注入
2. `@retry`：内层，提供重试逻辑
3. **调用链**：`EmbeddingFunc.__call__` → `@retry` → 实际实现

**防止双重包装的智慧**：

```python
# Azure OpenAI 包装器必须调用 .func 避免嵌套
return await openai_embed.func(  # ✅ 正确：访问底层函数
    texts=texts,
    model=deployment,
    use_azure=True,
)
# 而非 await openai_embed(...)  # ❌ 错误：会导致双重包装
```

## 缓存机制

LightRAG 对 LLM 响应进行缓存，避免重复调用。

**缓存键生成**：

```python
args_hash = compute_args_hash(
    query_param.mode,
    query,
    hl_keywords_str,
    ll_keywords_str,
    query_param.user_prompt or "",
    query_param.enable_rerank,
)
```

**缓存查询**：

```python
cached_result = await handle_cache(
    hashing_kv, args_hash, query, query_param.mode, cache_type="query"
)

if cached_result is not None:
    response = cached_result[0]
else:
    response = await use_model_func(...)
    # 保存缓存
    if hashing_kv and hashing_kv.global_config.get("enable_llm_cache"):
        await save_to_cache(hashing_kv, CacheData(...))
```

**缓存类型**：

- `extract`：实体提取缓存
- `query`：查询响应缓存

## Prompt 工程

### 实体提取 Prompt

**源码位置**: `lightrag/prompt.py:11-61`

实体提取 Prompt 的设计体现了"格式即协议"的理念：

```python
PROMPTS["entity_extraction_system_prompt"] = """---Role---
You are a Knowledge Graph Specialist responsible for extracting entities and relationships.

---Instructions---
1. **Entity Extraction & Output:**
   - entity_name: Title case, consistent naming
   - entity_type: {entity_types} or "Other"
   - entity_description: Based solely on input text
   - Format: entity{tuple_delimiter}entity_name{tuple_delimiter}entity_type{tuple_delimiter}entity_description

2. **Relationship Extraction & Output:**
   - N-ary decomposition: Alice-Bob-Carol → Alice-Bob, Bob-Carol
   - Format: relation{tuple_delimiter}source{tuple_delimiter}target{tuple_delimiter}keywords{tuple_delimiter}description

3. **Delimiter Protocol:**
   - {tuple_delimiter} = "<|#|>" (atomic marker, NOT fillable)
"""
```

**关键设计**：

1. **固定字段数量**：实体 4 字段，关系 5 字段
2. **首字段标识符**：`entity` / `relation` 前缀
3. **原子分隔符**：`<|#|>` 不可填充
4. **完成信号**：`{completion_delimiter}` 标记输出结束

### Few-shot 示例

Prompt 包含 3 个精心设计的示例，覆盖不同领域：

1. **人物关系网络**：展示人物关系提取
2. **金融市场分析**：展示抽象概念实体化
3. **体育赛事**：展示设备与人物关系

**Few-shot 设计原则**：

- **领域多样性**：覆盖不同场景
- **复杂度递增**：从简单到复杂
- **格式严格**：每个示例都以 `{completion_delimiter}` 结尾

### RAG 响应 Prompt

**源码位置**: `lightrag/prompt.py`

```python
PROMPTS["rag_response"] = """---Role---
You are a helpful assistant responding to questions about data in the tables provided.

---Goal---
Generate a response of the target length and format that responds to the user's question,
summarizing all information in the input data tables appropriate for the response length and format,
and incorporating any relevant general knowledge.

---Target response length and format---
{response_type}

---Data tables---
{context_data}

---User Prompt---
{user_prompt}
"""
```

**设计特点**：

- **角色定位**：明确助手身份
- **目标说明**：生成指定长度和格式的响应
- **数据注入**：`{context_data}` 包含检索结果
- **用户提示**：`{user_prompt}` 支持自定义指令

## 多语言支持

LightRAG 通过占位符系统支持多语言：

```python
{language}  # 目标语言
{entity_types}  # 实体类型列表
{tuple_delimiter}  # 字段分隔符
{completion_delimiter}  # 完成标志
```

**专有名词保留**：

```python
Proper nouns (e.g., personal names, place names, organization names)
should be retained in their original language
```

**示例**：
- 中文输出："Apple 公司发布了新产品"
- 而非："苹果公司发布了新产品"（丢失公司语义）

## 提供商配置

### BindingOptions 基类

**源码位置**: `lightrag/llm/binding_options.py:69-77`

```python
@dataclass
class BindingOptions:
    """Base class for binding options."""

    # mandatory name of binding
    _binding_name: ClassVar[str]

    # optional help message for each option
    _help: ClassVar[dict[str, str]]
```

**设计亮点**：

- 使用 `ClassVar` 区分类级别元数据和实例配置
- 通过 `dataclass` 自动生成 `__init__`
- 子类只需声明字段，框架自动生成 CLI 参数

### Ollama 配置示例

```python
@dataclass
class _OllamaOptionsMixin:
    # Core context and generation parameters
    num_ctx: int = 32768  # Context window size
    num_predict: int = 128  # Maximum tokens to predict

    # Sampling parameters
    temperature: float = DEFAULT_TEMPERATURE
    top_k: int = 40
    top_p: float = 0.9

    # Hardware parameters
    numa: bool = False
    num_gpu: int = -1  # -1 for auto
```

## 支持的提供商

| 提供商 | 文件 | 特点 |
|--------|------|------|
| OpenAI | `openai.py` | 最完整，支持 Azure |
| Anthropic | `anthropic.py` | Claude + Voyage AI Embedding |
| Ollama | `ollama.py` | 本地模型，云模型自动检测 |
| Google Gemini | `gemini.py` | 多模态支持 |
| AWS Bedrock | `bedrock.py` | 企业级，异常映射 |
| 智谱 AI | `zhipu.py` | 国内提供商 |
| HuggingFace | `hf.py` | 开源模型 |
| Jina | `jina.py` | Embedding 专用 |
| NVIDIA | `nvidia_openai.py` | OpenAI 兼容 API |

---

## 章节衔接

**本章回顾**：
- 我们学习了 LLM 提供商抽象和 Prompt 工程
- 关键收获：函数式接口、重试策略、缓存机制、Few-shot 设计

**下一章预告**：
- 在 `05-storage.md` 中，我们将学习四层存储架构
- 为什么需要学习：存储是 RAG 的基础，不同后端有不同特性
- 关键问题：四层分离的设计意图？多租户如何隔离？
