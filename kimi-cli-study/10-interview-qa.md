# 10 - 面试 QA（15 题）

## 本章目标

将 kimi-cli 的技术知识转化为面试回答能力，覆盖架构设计、框架选择、Agent 核心、工具系统等方向。

---

## Q1: 请介绍一下 kimi-cli 的整体架构

**类别**：架构设计

**回答要点**：

kimi-cli 采用分层架构，从上到下分为四层：

1. **表现层（UI Layer）**：Shell UI、Print UI、ACP UI，负责用户交互
2. **协议层（Wire）**：事件流抽象，解耦 Soul 和 UI
3. **业务层（Soul）**：KimiSoul、Context、Approval、LaborMarket，负责 Agent 核心逻辑
4. **框架层（Kosong）**：LLM 抽象框架，统一 Kimi、Anthropic、Gemini 接口

**设计哲学**：
- 分层解耦：每层独立演化
- 协议优先：通过 Protocol 定义契约
- 消除特殊情况：Wire 统一所有 UI，Kosong 统一所有 LLM

**源码引用**：`src/kimi_cli/soul/kimisoul.py`、`packages/kosong/`

---

## Q2: 为什么要自研 Kosong 框架而不用 LangChain？

**类别**：框架选择

**回答要点**：

**LangChain 的问题**：
1. 抽象过重：Chain/Agent/Memory 概念复杂
2. 类型不安全：大量 Any 类型
3. 扩展困难：需要继承 ABC，耦合度高

**Kosong 的优势**：
1. **Protocol-based**：使用 Python Protocol 实现结构化子类型，无需继承
2. **流式优先**：所有响应都是 AsyncIterator，支持实时渲染
3. **类型安全**：泛型约束 + Pydantic 验证
4. **轻量级**：只做 LLM 抽象，不做 Agent 框架

**代码示例**：
```python
# Kosong 的 ChatProvider 是 Protocol，不是 ABC
@runtime_checkable
class ChatProvider(Protocol):
    async def generate(self, ...) -> StreamedMessage: ...
```

**源码引用**：`packages/kosong/src/kosong/chat_provider/__init__.py:10-56`

---

## Q3: kimi-cli 的 Agent 循环是如何设计的？

**类别**：Agent 核心

**回答要点**：

kimi-cli 采用 **Step-based** 循环设计：

```
Turn (一轮对话)
├── Step 1: LLM Call + Tool Execution
├── Step 2: LLM Call + Tool Execution
├── Step 3: LLM Call (no tool) → STOP
```

**核心流程**：
1. 每个 step 前创建 checkpoint
2. 调用 `kosong.step()` 获取 LLM 响应
3. 并行执行工具调用
4. 更新 context
5. 检查停止条件（no_tool_calls / tool_rejected）

**设计优势**：
- 粒度适中：便于监控、中断、重试
- 状态可控：checkpoint 支持回退
- 资源可控：max_steps_per_turn 限制

**源码引用**：`src/kimi_cli/soul/kimisoul.py:306-459`

---

## Q4: 如何实现工具的依赖注入？

**类别**：工具系统

**回答要点**：

kimi-cli 使用**反射式依赖注入**：

```python
# 工具定义
class Shell(CallableTool2[Params]):
    def __init__(self, approval: Approval, environment: Environment):
        self._approval = approval

# 加载时自动注入
for param in inspect.signature(tool_cls).parameters.values():
    if param.annotation not in dependencies:
        raise ValueError(f"Dependency not found: {param.annotation}")
    args.append(dependencies[param.annotation])
```

**设计决策**：
1. **类型驱动**：依赖字典的 key 是类型，不是字符串
2. **位置参数优先**：遇到 KEYWORD_ONLY 参数停止注入
3. **失败快速**：依赖缺失时立即抛出异常

**源码引用**：`src/kimi_cli/soul/toolset.py:178-200`

---

## Q5: Wire 协议是如何解耦 Soul 和 UI 的？

**类别**：架构设计

**回答要点**：

Wire 是事件流抽象层，实现了 Soul 和 UI 的完全解耦：

```
Soul (soul_side) ──send()──► Wire ──receive()──► UI (ui_side)
```

**核心设计**：
1. **双队列架构**：raw_queue（原始消息）+ merged_queue（合并消息）
2. **BroadcastQueue**：SPMC 模式，支持多 UI 消费
3. **事件类型**：TurnBegin、StepBegin、TextPart、ToolCall、ApprovalRequest 等

**为什么需要两个队列**：
- Shell UI 需要低延迟（逐字渲染），使用 raw_queue
- WireRecorder 需要低存储，使用 merged_queue

**源码引用**：`src/kimi_cli/wire/__init__.py:18-128`

---

## Q6: D-Mail 时间旅行是如何实现的？

**类别**：独创设计

**回答要点**：

D-Mail 允许 Agent 向"过去的自己"发送消息，回退到之前的 checkpoint。

**实现机制**：
1. Agent 调用 SendDMail 工具
2. DenwaRenji 记录 pending D-Mail
3. _step() 完成后检测到 pending D-Mail
4. 抛出 `BackToTheFuture` 异常
5. _agent_loop() 捕获异常，回退 context
6. 注入 D-Mail 消息，继续循环

**为什么用异常**：
- 控制流清晰：天然支持跨层级跳转
- 状态一致：finally 块确保清理工作完成

**使用场景**：
- 大文件过滤：发现只需要部分内容
- 搜索优化：找到更好的搜索词
- 代码修复折叠：直接给出最终版本

**源码引用**：`src/kimi_cli/soul/denwarenji.py`、`src/kimi_cli/soul/kimisoul.py:431-455`

---

## Q7: Multi-Agent 的 LaborMarket 是如何设计的？

**类别**：Multi-Agent

**回答要点**：

LaborMarket 是"人才市场"，管理所有 subagent：

```python
class LaborMarket:
    fixed_subagents: dict[str, Agent]    # 配置文件定义
    dynamic_subagents: dict[str, Agent]  # 运行时创建
```

**Fixed vs Dynamic Subagent**：

| 维度 | Fixed | Dynamic |
|------|-------|---------|
| 定义方式 | agent.yaml | CreateSubagent 工具 |
| LaborMarket | 独立 | 共享 |
| 能否创建子 subagent | 否 | 是 |

**Runtime 克隆策略**：
- Fixed：独立 DenwaRenji + 独立 LaborMarket
- Dynamic：独立 DenwaRenji + 共享 LaborMarket

**源码引用**：`src/kimi_cli/soul/agent.py:125-186`

---

## Q8: Context Compaction 是如何工作的？

**类别**：Context 管理

**回答要点**：

当 context 接近 max_context_size 时，自动压缩历史对话。

**触发条件**：
```python
if token_count + reserved >= max_context_size:
    await self.compact_context()
```

**压缩策略（SimpleCompaction）**：
1. 保留最近 2 轮对话（`max_preserved_messages=2`）
2. 将其余消息发送给 LLM 进行总结
3. 用总结替换原始消息

**为什么用 LLM 压缩**：
- 比简单截断更智能
- 保留关键信息
- 保持上下文连贯性

**源码引用**：`src/kimi_cli/soul/compaction.py:42-116`

---

## Q9: MCP 工具是如何加载的？

**类别**：工具系统

**回答要点**：

kimi-cli 通过 fastmcp 集成 MCP 协议：

**加载流程**：
1. 解析 MCP 配置（HTTP/stdio 传输）
2. OAuth 预检查（如果需要）
3. 后台并发连接所有服务器
4. 动态注册工具到 KimiToolset

**状态机**：`pending → connecting → connected/failed/unauthorized`

**设计决策**：
- **后台加载**：避免阻塞主流程启动
- **并发连接**：`asyncio.gather` 并行连接
- **优雅降级**：单个服务器失败不影响其他

**源码引用**：`src/kimi_cli/soul/toolset.py:203-336`

---

## Q10: Approval 系统的三级策略是什么？

**类别**：安全机制

**回答要点**：

Approval 系统提供三级批准策略：

1. **YOLO 模式**：`--yolo` 参数，全部自动批准
2. **Session 级**：`approve_for_session`，同类操作不再询问
3. **单次批准**：`approve`，仅批准当前操作

**实现机制**：
```python
if self._state.yolo:
    return True
if action in self._state.auto_approve_actions:
    return True
# 否则等待用户响应
return await approved_future
```

**设计优势**：
- 安全与效率平衡
- 用户可控
- 支持批量操作

**源码引用**：`src/kimi_cli/soul/approval.py:50-147`

---

## Q11: 如何处理不同 LLM 厂商的协议差异？

**类别**：框架设计

**回答要点**：

Kosong 通过适配器模式处理厂商差异：

**Anthropic 差异**：
- 不支持 system role → 转换为 `<system>` 标签包裹的 user 消息
- tool 消息 → 打包到 user 消息中的 ToolResultBlockParam

**Gemini 差异**：
- functionResponse 数量必须与 functionCall 匹配
- 需要按 expected_tool_call_ids 排序

**Kimi 扩展**：
- 支持 `reasoning_content` 字段（思考内容）
- 支持 `cached_tokens` 字段

**源码引用**：`packages/kosong/contrib/chat_provider/anthropic.py`、`google_genai.py`

---

## Q12: 为什么 ContentPart 使用类型注册表模式？

**类别**：设计模式

**回答要点**：

ContentPart 使用 `__init_subclass__` + `__get_pydantic_core_schema__` 实现类型注册表：

```python
class ContentPart(BaseModel, ABC):
    __content_part_registry: ClassVar[dict[str, type["ContentPart"]]] = {}

    def __init_subclass__(cls, **kwargs):
        cls.__content_part_registry[cls.type] = cls
```

**优势**：
1. **自动注册**：定义子类时自动注册
2. **类型安全**：编译时检查
3. **易于扩展**：添加新类型只需定义子类

**为什么不用 Pydantic discriminator**：
- 更灵活：可以处理任意嵌套结构
- 更可控：完全掌控验证逻辑

**源码引用**：`packages/kosong/src/kosong/message.py:16-72`

---

## Q13: asyncio.shield 在 kimi-cli 中的应用场景？

**类别**：并发编程

**回答要点**：

`asyncio.shield` 用于保护 `_grow_context` 不被中断：

```python
await asyncio.shield(self._grow_context(result, results))
```

**问题**：用户按 Ctrl+C 时，`_grow_context` 可能被中断，导致 context 状态不一致。

**解决方案**：shield 确保即使外部取消，`_grow_context` 也会完成执行。

**trade-off**：
- 优点：保证 context 一致性
- 缺点：用户需要等待操作完成

**源码引用**：`src/kimi_cli/soul/kimisoul.py:423`

---

## Q14: kimi-cli 与 Claude Code CLI 的主要区别？

**类别**：对比分析

**回答要点**：

| 维度 | kimi-cli | Claude Code CLI |
|------|----------|-----------------|
| **LLM 框架** | Kosong (自研) | 直接调用 Anthropic SDK |
| **时间旅行** | D-Mail | 无 |
| **Multi-Agent** | LaborMarket | Task tool |
| **UI 解耦** | Wire 协议 | 直接耦合 |
| **Skill 系统** | Standard + Flow | Standard |

**kimi-cli 独特优势**：
1. D-Mail 时间旅行：业界唯一
2. Kosong 框架：vendor-agnostic
3. Wire 协议：彻底解耦
4. Flow Skill：可视化流程图

---

## Q15: 如果让你重构 kimi-cli，你会改进什么？

**类别**：架构决策

**回答要点**：

**可能的改进方向**：

1. **Context 存储**：
   - 当前：JSONL 文件
   - 改进：支持 SQLite 选项，便于查询和索引

2. **MCP 连接管理**：
   - 当前：每次启动重新连接
   - 改进：连接池 + 心跳保活

3. **Compaction 策略**：
   - 当前：简单保留最近 N 条
   - 改进：基于重要性评分的智能压缩

4. **测试覆盖**：
   - 当前：E2E 测试为主
   - 改进：增加单元测试，特别是 Kosong 框架

**需要保留的设计**：
- Wire 协议的解耦设计
- D-Mail 的异常机制
- 依赖注入的反射实现

---

## 章节衔接

**本章回顾**：
- 我们学习了 15 道核心面试题
- 关键收获：如何组织回答、如何引用源码、如何分析 trade-off

**下一章预告**：
- 在 `11-summary.md` 中，我们将总结知识图谱
- 为什么需要学习：将零散知识串联成体系
- 关键问题：核心概念之间的关系是什么？学习检查清单是什么？
