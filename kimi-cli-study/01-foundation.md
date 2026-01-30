# 第一层：基础抽象层

> 本章介绍 kimi-cli 的基础数据结构和抽象层。理解这些是深入 Agent 核心的前提。

---

## 1.1 Kosong 消息结构

### 设计意图

为什么需要统一的 LLM 抽象层？

1. **多提供商兼容性**：不同 LLM 提供商的消息格式不统一（OpenAI vs Anthropic vs Moonshot）
2. **扩展类型支持**：需要支持思考模式、流式输出、多模态内容
3. **异步工具编排**：工具调用需要统一的接口建模

### 核心数据结构

**源码位置**: `packages/kosong/src/kosong/message.py:15-50`

```python
@dataclass
class Message:
    role: str  # "system" | "user" | "assistant" | "tool"
    content: list[ContentPart]

@dataclass
class ContentPart:
    type: str

    # 为什么用 list[ContentPart] 而非单一字符串？
    # A: 多模态场景需要混合 text + image + tool_use
    #    例如："帮我分析这张图片" + image_content
```

### ContentPart 多态设计

**源码位置**: `packages/kosong/src/kosong/message.py:52-120`

```python
class TextPart(ContentPart):
    type: Literal["text"] = "text"
    text: str

class ThinkPart(ContentPart):
    type: Literal["think"] = "think"
    # 思考模式：让 LLM 输出推理过程，但不计入最终输出

class ImagePart(ContentPart):
    type: Literal["image"] = "image"
    image_url: str

class ToolUsePart(ContentPart):
    type: Literal["tool_use"] = "tool_use"
    id: str  # 工具调用唯一标识，用于匹配 tool_result
    name: str
    input: dict

class ToolResultPart(ContentPart):
    type: Literal["tool_result"] = "tool_result"
    tool_use_id: str  # 必须匹配 ToolUsePart.id
    content: str | list[ContentPart]
    is_error: bool = False
```

### 设计分析

**Q: 为什么用 list[ContentPart] 而非单一字符串？**

A: 三个原因：

1. **多模态支持**：用户可能发送"帮我分析这张图片" + 图片内容，单一字符串无法表达
2. **工具调用结构化**：ToolUsePart 需要携带 id/name/input，无法塞进字符串
3. **流式扩展**：未来可能支持视频、音频等类型，list 架构更易扩展

**Q: ThinkPart 的作用是什么？**

A: 让 LLM 输出推理过程，但不计入最终输出给用户。例如：

```
用户："今天是星期几？"
Assistant (Think): 我需要查看当前日期...
Assistant (Text): 今天是星期五。
```

用户只看到"今天是星期五"，但 Think 内容可用于调试和上下文理解。

**Q: 为什么 ToolUsePart 需要 id？**

A: 异步场景下可能有多个工具并发执行，id 用于匹配 tool_result。例如：

```python
[
    ToolUsePart(id="call_1", name="search", input={...}),
    ToolUsePart(id="call_2", name="calculator", input={...})
]

# 工具返回时必须匹配 id
[
    ToolResultPart(tool_use_id="call_1", content="..."),
    ToolResultPart(tool_use_id="call_2", content="...")
]
```

### 对比其他框架

| 维度 | Kosong | LangChain | Anthropic |
|------|--------|-----------|-----------|
| 消息结构 | 统一 Message | 多种类型（Human/AI/System） | 原生支持思考 |
| 多模态 | 高（自定义 Part） | 中 | 低 |
| 扩展性 | 高（类型注册） | 中 | 低 |
| 工具调用 | 结构化 Part | 混合在 content 中 | 结构化 |

---

## 1.2 Tool 抽象演进

### 设计意图

工具输出需要满足三方的不同需求：

1. **LLM**：需要结构化数据用于推理
2. **开发者**：需要自然语言解释用于调试
3. **最终用户**：需要渲染友好的显示内容

### 核心数据结构

**源码位置**: `packages/kosong/src/kosong/tooling/base.py:20-60`

```python
@dataclass
class ToolReturnValue:
    # 第一层：给 LLM 的结构化输出
    output: str | dict | list

    # 第二层：给开发者的自然语言解释
    message: str | None = None

    # 第三层：给用户的渲染内容
    display: str | None = None
```

### 设计分析

**Q: 为什么要分离这三层？**

A: 每层的语义目标不同：

| 层 | 目标受众 | 语义目标 | 示例 |
|---|---------|---------|------|
| output | LLM | 结构化数据，用于推理 | `{"files": ["a.py", "b.py"]}` |
| message | 开发者 | 自然语言，便于调试 | "列出当前目录的 Python 文件" |
| display | 用户 | 渲染友好，可读性强 | "**Found 2 files:**<br>- a.py<br>- b.py" |

**Q: 如果工具返回流式数据（如 tail -f），如何适配？**

A: 当前模型不支持流式。如果要支持，需要让 `output` 变成 AsyncIterator，但这会带来复杂性：

1. 上下文管理：何时关闭流？
2. Token 计费：流式输出的 token 如何计算？
3. 中断处理：用户中断时如何清理流？

**Q: 为什么不用 Protocol/Structural Typing？**

A: 继承的好处是类型明确，IDE 支持更好。Protocol 的好处是灵活性高，但会增加类型推导的复杂度。对于工具系统这种核心基础设施，明确性优于灵活性。

### 对比 REST API 设计

ToolReturnValue 的三层设计与 REST API 的响应设计有共通之处：

| REST API | ToolReturnValue |
|----------|-----------------|
| `data` (结构化数据) | `output` |
| `message` (人类可读) | `message` |
| `view` (渲染模板) | `display` |

---

## 1.3 PyKAOS 系统抽象

### 设计意图

为什么需要系统抽象层？

1. **跨平台兼容**：macOS/Linux/Windows 的路径操作不同
2. **安全性**：沙箱环境需要限制文件访问
3. **可测试性**：单元测试需要 Mock 文件系统

### 核心抽象

**源码位置**: `packages/kaos/src/kaos/path.py`

```python
class AbsolutePath:
    """绝对路径，防止目录遍历攻击"""

    def __init__(self, path: str):
        self._path = Path(path).resolve()

    def join(self, *parts: str) -> "AbsolutePath":
        """安全拼接路径，自动解析 .."""
        new_path = self._path.joinpath(*parts).resolve()
        return AbsolutePath(str(new_path))
```

### 设计分析

**Q: 为什么不用 pathlib 直接？**

A: pathlib 的一个问题是 `joinpath` 不会自动解析 `..`，可能导致目录遍历：

```python
# 危险：允许跳出项目目录
base = Path("/home/project")
evil = base.joinpath("../etc/passwd")  # 没有报错！

# 安全：自动解析，阻止跳出
base = AbsolutePath("/home/project")
safe = base.joinpath("../etc/passwd")  # 解析后仍在 /home/project 内
```

**Q: 沙箱机制如何实现？**

A: 通过 `ProjectRoot` 类限制访问范围：

```python
class ProjectRoot:
    def __init__(self, root: AbsolutePath):
        self._root = root

    def resolve(self, path: str) -> AbsolutePath:
        """解析路径，确保在 root 范围内"""
        resolved = self._root.joinpath(path).resolve()
        if not resolved.startswith(self._root):
            raise PermissionError(f"Access denied: {path}")
        return resolved
```

---

## 本章小结

基础抽象层的设计哲学：

1. **统一接口，多样实现**：Message 统一格式，ContentPart 多态表达
2. **分层语义，各取所需**：ToolReturnValue 三层输出满足不同角色
3. **安全第一，可测试性**：PyKAOS 提供安全的系统抽象

这些抽象为上层的 Agent 核心提供了坚实的基础。
