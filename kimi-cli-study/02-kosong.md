# 02 - Kosong LLM 框架深度解析

## 本章目标

理解 Kosong 框架的设计哲学，掌握 vendor-agnostic LLM 接口的实现方式，学习如何处理不同厂商的协议差异。

## Kosong 是什么？

Kosong 是 kimi-cli 自研的 LLM 抽象框架，名字来自印尼语"空"（kosong），寓意"空杯心态"——不预设任何 LLM 厂商的特性，只定义最小公共接口。

**设计目标**：
1. **Vendor-agnostic**：一套代码支持 Kimi、Anthropic、Gemini
2. **流式优先**：所有响应都是流式的，支持实时渲染
3. **类型安全**：通过 Protocol 和泛型确保类型正确

## 核心抽象

### ChatProvider 协议

```python
# packages/kosong/src/kosong/chat_provider/__init__.py:10-56
@runtime_checkable
class ChatProvider(Protocol):
    """The interface of chat providers."""

    name: str

    @property
    def model_name(self) -> str:
        """The name of the model to use."""
        ...

    async def generate(
        self,
        system_prompt: str,
        tools: Sequence[Tool],
        history: Sequence[Message],
    ) -> "StreamedMessage":
        """Generate a new message based on the given system prompt, tools, and history."""
        ...
```

**设计决策**：

1. **Protocol 而非 ABC**：使用 Python 的 Protocol 实现结构化子类型，避免继承耦合
2. **Async-first**：所有 I/O 操作都是异步的，支持高并发
3. **StreamedMessage 返回**：统一流式响应接口

**为什么用 Protocol？**

```python
# Protocol 的优势：无需继承，只要实现相同签名
class MyProvider:  # 没有继承任何类
    name = "my"

    @property
    def model_name(self) -> str:
        return "my-model"

    async def generate(self, system_prompt, tools, history):
        ...

# 类型检查通过
provider: ChatProvider = MyProvider()  # OK

# 运行时检查也通过
assert isinstance(MyProvider(), ChatProvider)  # True (因为 @runtime_checkable)
```

### Message 统一消息结构

```python
# packages/kosong/src/kosong/message.py:243-304
class Message(BaseModel):
    """A message in a conversation."""

    role: Role  # "system" | "user" | "assistant" | "tool"
    name: str | None = None
    content: list[ContentPart]
    tool_calls: list[ToolCall] | None = None
    tool_call_id: str | None = None

    @field_validator("content", mode="before")
    @classmethod
    def _coerce_none_content(cls, value: Any) -> Any:
        if value is None:
            return []
        if isinstance(value, str):
            return [TextPart(text=value)]
        return value
```

**设计亮点**：

1. **content 自动转换**：字符串自动转为 `[TextPart(text=...)]`，None 转为 `[]`
2. **多态 ContentPart**：支持 TextPart、ImageURLPart、ThinkPart 等多种类型
3. **tool_calls 分离**：工具调用单独字段，不混入 content

### ContentPart 类型注册表

这是 Kosong 最精妙的设计之一——通过元类编程实现运行时类型分发。

```python
# packages/kosong/src/kosong/message.py:16-72
class ContentPart(BaseModel, ABC, MergeableMixin):
    __content_part_registry: ClassVar[dict[str, type["ContentPart"]]] = {}

    def __init_subclass__(cls, **kwargs: Any) -> None:
        super().__init_subclass__(**kwargs)
        type_value = getattr(cls, "type", None)
        if type_value is None or not isinstance(type_value, str):
            raise ValueError(f"ContentPart subclass {cls.__name__} must have a `type` field")
        cls.__content_part_registry[type_value] = cls

    @classmethod
    def __get_pydantic_core_schema__(cls, source_type: Any, handler: GetCoreSchemaHandler):
        if cls.__name__ == "ContentPart":
            def validate_content_part(value: Any) -> Any:
                if isinstance(value, dict) and "type" in value:
                    type_value = value.get("type")
                    target_class = cls.__content_part_registry[type_value]
                    return target_class.model_validate(value)
                raise ValueError(f"Cannot validate {value} as ContentPart")
            return core_schema.no_info_plain_validator_function(validate_content_part)
        return handler(source_type)
```

**工作原理**：

1. **`__init_subclass__`**：每当定义 ContentPart 子类时自动调用，将子类注册到 `__content_part_registry`
2. **`__get_pydantic_core_schema__`**：自定义 Pydantic 验证逻辑，根据 `type` 字段分发到正确的子类

```python
# 使用示例
class TextPart(ContentPart):
    type: Literal["text"] = "text"  # 自动注册到 registry["text"]
    text: str

class ThinkPart(ContentPart):
    type: Literal["think"] = "think"  # 自动注册到 registry["think"]
    think: str

# 反序列化时自动分发
data = {"type": "text", "text": "hello"}
part = ContentPart.model_validate(data)  # 返回 TextPart 实例
```

### MergeableMixin 增量合并

流式响应需要将多个片段合并成完整消息，MergeableMixin 提供了这个能力。

```python
# packages/kosong/src/kosong/message.py:80-110
class MergeableMixin:
    """Mixin for content parts that can be merged."""

    def merge_in_place(self, other: "MergeableMixin") -> bool:
        """Try to merge another part into this one.

        Returns True if the merge was successful, False otherwise.
        """
        return False  # 默认不可合并

# TextPart 的合并实现
class TextPart(ContentPart, MergeableMixin):
    type: Literal["text"] = "text"
    text: str

    def merge_in_place(self, other: MergeableMixin) -> bool:
        if isinstance(other, TextPart):
            self.text += other.text  # 直接拼接文本
            return True
        return False
```

**使用场景**：

```python
# 流式响应：["Hel", "lo", " ", "World"]
parts = [TextPart(text="Hel"), TextPart(text="lo"), TextPart(text=" "), TextPart(text="World")]

# 合并后：["Hello World"]
merged = TextPart(text="Hel")
for part in parts[1:]:
    merged.merge_in_place(part)
# merged.text == "Hello World"
```

## generate() vs step()

Kosong 提供两个核心 API，理解它们的差异是关键。

### generate() - 底层流式生成

```python
# packages/kosong/src/kosong/_generate.py:17-81
async def generate(
    chat_provider: ChatProvider,
    system_prompt: str,
    tools: Sequence[Tool],
    history: Sequence[Message],
    *,
    on_message_part: Callback[[StreamedMessagePart], None] | None = None,
    on_tool_call: Callback[[ToolCall], None] | None = None,
) -> "GenerateResult":
    message = Message(role="assistant", content=[])
    pending_part: StreamedMessagePart | None = None

    stream = await chat_provider.generate(system_prompt, tools, history)
    async for part in stream:
        if on_message_part:
            await callback(on_message_part, part.model_copy(deep=True))

        if pending_part is None:
            pending_part = part
        elif not pending_part.merge_in_place(part):  # 尝试合并
            _message_append(message, pending_part)
            if isinstance(pending_part, ToolCall) and on_tool_call:
                await callback(on_tool_call, pending_part)
            pending_part = part

    # 处理最后一个 pending_part
    if pending_part is not None:
        _message_append(message, pending_part)
        if isinstance(pending_part, ToolCall) and on_tool_call:
            await callback(on_tool_call, pending_part)

    return GenerateResult(id=stream.id, message=message, usage=stream.usage)
```

**职责**：
- 纯粹的流式响应消费
- 增量合并 StreamedMessagePart
- 不处理工具执行，只触发 `on_tool_call` 回调

### step() - 高层工具编排

```python
# packages/kosong/src/kosong/__init__.py:104-180
async def step(
    chat_provider: ChatProvider,
    system_prompt: str,
    toolset: Toolset,
    history: Sequence[Message],
    *,
    on_message_part: Callback[[StreamedMessagePart], None] | None = None,
    on_tool_result: Callable[[ToolResult], None] | None = None,
) -> "StepResult":
    tool_calls: list[ToolCall] = []
    tool_result_futures: dict[str, ToolResultFuture] = {}

    async def on_tool_call(tool_call: ToolCall):
        tool_calls.append(tool_call)
        result = toolset.handle(tool_call)  # 立即开始执行

        if isinstance(result, ToolResult):
            future = ToolResultFuture()
            future.set_result(result)
            tool_result_futures[tool_call.id] = future
        else:
            tool_result_futures[tool_call.id] = result  # 异步 Future

    try:
        result = await generate(
            chat_provider, system_prompt, toolset.tools, history,
            on_message_part=on_message_part,
            on_tool_call=on_tool_call,
        )
    except (ChatProviderError, asyncio.CancelledError):
        # 取消所有 pending 的工具执行
        for future in tool_result_futures.values():
            future.cancel()
        await asyncio.gather(*tool_result_futures.values(), return_exceptions=True)
        raise

    return StepResult(result.id, result.message, result.usage, tool_calls, tool_result_futures)
```

**职责**：
- 在 generate() 基础上增加工具调度
- 异步执行工具（通过 `toolset.handle()` 返回 Future）
- 错误时自动取消所有 pending 的工具执行

### 关键差异对比

| 维度 | generate() | step() |
|------|-----------|--------|
| 抽象层级 | 底层流式生成 | 高层工具编排 |
| 工具处理 | 仅触发回调 | 异步执行工具 |
| 返回值 | GenerateResult | StepResult（含 tool_result_futures） |
| 错误处理 | 传播异常 | 取消所有 pending 工具 |

## 多厂商适配

### Kimi 适配器

```python
# packages/kosong/src/kosong/chat_provider/kimi.py:368-414
async def _convert_stream_response(
    self,
    response: AsyncIterator[ChatCompletionChunk],
) -> AsyncIterator[StreamedMessagePart]:
    try:
        async for chunk in response:
            if chunk.id:
                self._id = chunk.id
            if usage := extract_usage_from_chunk(chunk):
                self._usage = usage

            if not chunk.choices:
                continue

            delta = chunk.choices[0].delta

            # 提取 thinking content (Kimi 扩展)
            if reasoning_content := getattr(delta, "reasoning_content", None):
                yield ThinkPart(think=reasoning_content)

            # 提取文本内容
            if delta.content:
                yield TextPart(text=delta.content)

            # 提取工具调用
            for tool_call in delta.tool_calls or []:
                if tool_call.function.name:
                    yield ToolCall(
                        id=tool_call.id or str(uuid.uuid4()),
                        function=ToolCall.FunctionBody(
                            name=tool_call.function.name,
                            arguments=tool_call.function.arguments,
                        ),
                    )
                elif tool_call.function.arguments:
                    yield ToolCallPart(arguments_part=tool_call.function.arguments)
    except (OpenAIError, httpx.HTTPError) as e:
        raise convert_error(e) from e
```

**设计亮点**：
1. **增量 Tool Call 处理**：先 yield ToolCall（带 name），后续 yield ToolCallPart（arguments 片段）
2. **Thinking 内容提取**：支持 Kimi 的 `reasoning_content` 扩展字段
3. **统一错误转换**：将 OpenAI SDK 异常转换为 Kosong 统一异常

### Anthropic 适配器

Anthropic 的 API 与 OpenAI 差异较大，需要特殊处理。

```python
# packages/kosong/contrib/chat_provider/anthropic.py:267-291
def _convert_message(self, message: Message) -> MessageParam:
    """Convert a single internal message into Anthropic wire format."""
    role = message.role

    if role == "system":
        # Anthropic 不支持 system role，转换为特殊 user 消息
        return MessageParam(
            role="user",
            content=[
                TextBlockParam(
                    type="text",
                    text=f"<system>{message.extract_text(sep='\\n')}</system>"
                )
            ],
        )
    elif role == "tool":
        # tool 消息转换为 user 消息中的 ToolResultBlockParam
        block = _tool_result_message_to_block(message.tool_call_id, content)
        return MessageParam(role="user", content=[block])
```

**协议差异处理**：
1. **System Role**：Anthropic 不支持 system role，转换为 `<system>` 标签包裹的 user 消息
2. **Tool Result**：tool 消息打包到 user 消息中

**Prompt Caching 自动注入**：

```python
# packages/kosong/contrib/chat_provider/anthropic.py:186-203
if messages:
    last_message = messages[-1]
    last_content = last_message["content"]

    # 在最后一条消息注入 cache_control
    if isinstance(last_content, list) and last_content:
        content_blocks = cast(list[ContentBlockParam], last_content)
        last_block = content_blocks[-1]
        match last_block["type"]:
            case ("text" | "image" | "document" | ...):
                last_block["cache_control"] = CacheControlEphemeralParam(type="ephemeral")
```

### Google Gemini 适配器

Gemini 对 tool messages 有严格的顺序要求。

```python
# packages/kosong/contrib/chat_provider/google_genai.py:492-551
def _tool_messages_to_google_genai_content(
    messages: Sequence[Message],
    *,
    tool_name_by_id: dict[str, str],
    expected_tool_call_ids: Sequence[str] | None = None,
) -> Content:
    """Pack one-or-more tool results into a single Gemini "user" turn.

    VertexAI-backed Gemini enforces that, for a tool-calling turn, the next
    turn contains the same number of `functionResponse` parts as the preceding
    `functionCall` parts.
    """
    # 按 expected_tool_call_ids 排序，确保与 function_call 顺序一致
    indexed_messages.sort(
        key=lambda t: (expected_index.get(cast(str, t[1].tool_call_id), 10**9), t[0])
    )

    parts: list[Part] = []
    for _, message in indexed_messages:
        parts.append(
            _tool_message_to_function_response_part(message, tool_name_by_id=tool_name_by_id)
        )
    return Content(role="user", parts=parts)
```

**设计亮点**：
1. **并行工具调用顺序保证**：按 expected_tool_call_ids 排序
2. **VertexAI 严格验证**：强制要求 functionResponse 数量与 functionCall 匹配

## Token 计数与 Usage 追踪

### 统一 TokenUsage 结构

```python
# packages/kosong/src/kosong/chat_provider/__init__.py:80-101
class TokenUsage(BaseModel):
    """Token usage statistics."""

    input_other: int
    """Input tokens excluding cache."""
    output: int
    """Total output tokens."""
    input_cache_read: int = 0
    """Cached input tokens."""
    input_cache_creation: int = 0
    """Input tokens used for cache creation (Anthropic only)."""

    @property
    def total(self) -> int:
        return self.input + self.output

    @property
    def input(self) -> int:
        return self.input_other + self.input_cache_read + self.input_cache_creation
```

**设计亮点**：
1. **细粒度拆分**：区分 cache_read、cache_creation、other
2. **厂商差异抹平**：Kimi 的 `cached_tokens`、Anthropic 的 `cache_read_input_tokens` 都映射到统一结构

## 工具系统抽象

### CallableTool2 泛型设计

```python
# packages/kosong/src/kosong/tooling/__init__.py:232-317
class CallableTool2[Params: BaseModel](ABC):
    """Tool with typed parameters."""

    name: str
    description: str
    params: type[Params]

    def __init__(self, name: str | None = None, ...):
        self.name = name or getattr(cls, "name", "")
        self.params = params or getattr(cls, "params", None)

        # 从 Pydantic 模型自动生成 JSON Schema
        self._base = Tool(
            name=self.name,
            description=self.description,
            parameters=deref_json_schema(
                self.params.model_json_schema(schema_generator=_GenerateJsonSchemaNoTitles)
            ),
        )

    async def call(self, arguments: JsonType) -> ToolReturnValue:
        try:
            params = self.params.model_validate(arguments)
        except pydantic.ValidationError as e:
            return ToolValidateError(str(e))

        ret = await self.__call__(params)
        return ret
```

**设计亮点**：
1. **泛型参数约束**：`[Params: BaseModel]` 确保类型安全
2. **JSON Schema 自动生成**：从 Pydantic 模型自动生成 LLM 可理解的 schema
3. **防御性编程**：参数验证失败返回 ToolValidateError，不抛异常

## 设计模式总结

### 1. Protocol-based Polymorphism（协议多态）

- **位置**：ChatProvider、Toolset、StreamedMessage
- **优势**：避免继承耦合，支持结构化子类型

### 2. Registry Pattern（注册表模式）

- **位置**：ContentPart、DisplayBlock
- **实现**：`__init_subclass__` + `__content_part_registry`
- **优势**：运行时类型分发，易于扩展

### 3. Adapter Pattern（适配器模式）

- **位置**：Kimi/Anthropic/Gemini 适配器
- **职责**：将厂商 API 转换为统一的 StreamedMessagePart

### 4. Future-based Async Orchestration（Future 异步编排）

- **位置**：SimpleToolset.handle()、StepResult.tool_results()
- **优势**：非阻塞工具执行，支持并发

---

## 章节衔接

**本章回顾**：
- 我们学习了 Kosong 框架的核心抽象（ChatProvider、Message、ContentPart）
- 关键收获：Protocol 多态、类型注册表、generate() vs step() 的差异

**下一章预告**：
- 在 `03-agent-loop.md` 中，我们将学习 Agent 循环的实现
- 为什么需要学习：Agent 循环是 kimi-cli 的核心，理解它才能理解 Agent 如何"思考"和"行动"
- 关键问题：Step-based 循环如何设计？重试机制如何实现？停止条件是什么？
