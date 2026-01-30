# 第八层：面试 QA

> 本章包含 15 道核心面试题，参考业界顶级面试题格式（Sierra AI、OpenAI/Anthropic 联合评估标准）。

---

## 架构与权衡 (5题)

---

### Q1: 架构选择与 Trade-off

**难度**: 三星 | **类别**: 架构决策

你设计了 UI → Soul → Infrastructure 三层架构。请回答：

1. 为什么选择这种分层？相比六边形架构或洋葱架构，这种设计的 trade-off 是什么？
2. 如果要支持多租户（多个用户同时使用），现有架构需要做哪些调整？
3. Wire 事件总线使用 SPMC 模型。相比传统回调或观察者模式，这种设计的核心优势和潜在风险是什么？

**源码引用**:
- `src/kimi_cli/soul/run_soul.py:45-89` (Wire 连接)
- `src/kimi_cli/wire/types.py` (事件定义)

**参考答案**:

**1. 分层选择的理由**

**UI → Soul → Infrastructure** 的分层：
- **UI 层**：处理用户交互（命令行、IDE 集成）
- **Soul 层**：Agent 核心逻辑（ReAct 循环、工具执行）
- **Infrastructure 层**：底层能力（文件系统、LLM 调用）

**优势**：
- **简单**：每层职责清晰，易于理解和维护
- **灵活性**：可以替换 UI（CLI → Web）或 Infrastructure（OpenAI → Anthropic）

**Trade-off**：

| 架构 | 优势 | 劣势 |
|------|------|------|
| 三层架构 | 简单，易理解 | 耦合度高，难以独立演化 |
| 六边形架构 | 完全解耦，可测试 | 复杂度高，学习曲线陡峭 |
| 洋葱架构 | 依赖方向清晰 | 过度抽象，小项目 overkill |

kimi-cli 选择三层架构是因为：
- 项目规模中等，不需要过度的抽象
- 演化路径清晰，可以逐步重构

**2. 多租户支持**

需要新增的组件：

```
┌─────────────────────────────────────────────────────┐
│                    Tenant Manager                    │
│  负责租户隔离、权限控制、资源配额                     │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│           Soul per Tenant                            │
│  每个租户有独立的 Soul 实例                           │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│            Shared Infrastructure                     │
│  LLM 调用、文件系统等可共享                           │
└─────────────────────────────────────────────────────┘
```

具体调整：
1. **租户隔离**：每个租户有独立的 Context 和 Runtime
2. **权限控制**：租户只能访问自己的资源
3. **资源配额**：限制每个租户的 token 使用量

**3. SPMC 的优势和风险**

**优势**：
1. **完全解耦**：发布者不需要知道谁在订阅
2. **异步**：订阅者可以异步处理，不阻塞发布者
3. **可扩展**：新增订阅者无需修改发布者代码

**风险**：
1. **调试困难**：事件流不直观，难以追踪
2. **错误处理**：订阅者报错不影响其他订阅者，但可能丢失错误
3. **内存泄漏**：订阅者忘记取消订阅会导致内存泄漏

---

### Q2: 自研 vs 开源框架

**难度**: 四星 | **类别**: 框架选择

你自研了 Kosong 作为 LLM 抽象层，而非直接使用 LangChain。请回答：

1. 什么情况下应该自研，什么情况用开源？你的决策标准是什么？
2. Kosong 解决了哪些现有框架无法解决的问题？
3. 如果 LangChain 明天添加了 Kosong 的所有特性，你会切换过去吗？为什么？
4. 反面思考：如果重来一次，哪些部分你会直接用现成框架？

**源码引用**:
- `packages/kosong/src/kosong/message.py:15-50` (消息结构)
- `packages/kosong/src/kosong/step.py:80-150` (LLM 调用)

**参考答案**:

**1. 自研 vs 开源的决策标准**

**自研的条件**：
1. **核心差异化**：这部分是你的核心竞争力
2. **现有方案不足**：开源框架无法满足需求
3. **团队能力**：团队有足够的经验维护

**使用开源的条件**：
1. **非核心功能**：如日志、配置、数据库 ORM
2. **成熟方案**：如 SQLAlchemy、pytest
3. **快速迭代**：需要快速验证想法

kimi-cli 的决策：
- **自研 Kosong**：LLM 抽象是核心，需要深度定制
- **自研 PyKAOS**：系统抽象需要跨平台兼容
- **使用 fastmcp**：MCP 是标准协议，无需自研

**2. Kosong 解决的问题**

1. **统一消息格式**：OpenAI/Anthropic/Moonshot 的消息格式不统一
2. **思考模式**：原生支持 ThinkPart，而 LangChain 需要定制
3. **三层工具输出**：ToolReturnValue 满足 LLM/开发者/用户三方的需求

**3. 如果 LangChain 添加所有特性**

不会切换。原因：
1. **迁移成本**：已有大量代码依赖 Kosong
2. **控制力**：自研可以快速响应需求变化
3. **学习成本**：团队已经熟悉 Kosong 的设计

**4. 如果重来一次**

可能会使用开源的部分：
1. **MCP 集成**：直接使用 fastmcp，无需自研
2. **配置管理**：使用 Pydantic Settings
3. **日志系统**：使用 structlog

---

### Q3: Agent 适用边界

**难度**: 三星 | **类别**: 系统设计

这是业界面试的核心问题：什么时候 Agent 是 overkill？

1. 给出 3 个不应该用 Agent 的场景，解释为什么直接用 LLM 或规则系统更好
2. 给出 3 个必须用 Agent 的场景，解释为什么简单的 RAG 或 Chain 不足够
3. kimi-cli 的核心价值是什么？为什么它需要是 Agent 而非"增强版的 Copilot"？

**源码引用**:
- `src/kimi_cli/soul/kimisoul.py:120-200` (ReAct 循环)

**参考答案**:

**1. 不应该用 Agent 的场景**

| 场景 | 为什么不用 Agent | 更好的方案 |
|------|-----------------|-----------|
| 文本分类 | 无需工具调用，直接分类即可 | 单个 LLM 调用 |
| 问答系统 | 答案在文档中，无需推理 | RAG（向量检索） |
| 代码补全 | 局部上下文，无需多步推理 | 专用的代码补全模型 |

**核心判断标准**：如果任务不需要"观察-思考-行动"的循环，Agent 就是 overkill。

**2. 必须用 Agent 的场景**

| 场景 | 为什么需要 Agent | 为什么 RAG/Chain 不够 |
|------|-----------------|---------------------|
| 代码审查 | 需要读取文件、运行测试、生成报告 | RAG 无法执行工具 |
| 数据分析 | 需要查询数据库、可视化、迭代 | Chain 无法动态调整 |
| 自动化运维 | 需要监控、诊断、修复 | 规则系统无法处理未知情况 |

**核心判断标准**：如果任务需要多步推理、工具调用、动态调整，Agent 是必需的。

**3. kimi-cli 的核心价值**

kimi-cli 不是"增强版的 Copilot"，而是：

1. **主动的助手**：不是等待用户调用，而是主动调用工具
2. **多步推理**：可以分解复杂任务，逐步执行
3. **记忆能力**：可以记住历史对话，累积上下文

**示例**：

```
用户："帮我部署这个应用"

Copilot: 给你部署命令...
kimi-cli:
  1. 读取配置文件
  2. 检查依赖
  3. 构建镜像
  4. 推送到仓库
  5. 更新 Kubernetes
  6. 验证部署成功
```

---

### Q4: 框架对比深度分析

**难度**: 五星 | **类别**: 行业认知

请对比你的设计与主流框架的本质区别：

1. vs LangGraph：LangGraph 用状态机建模 Agent 工作流，你的设计与状态机模型有何异同？
2. vs AutoGen：AutoGen 用对话驱动多 Agent 协作，你的 LaborMarket 模型与对话模型的优劣对比？
3. vs CrewAI：CrewAI 用角色分配组织 Agent，你的 Fixed/Dynamic 子 Agent 的设计哲学差异？
4. 如果你要说服一家公司用你的架构而非 LangGraph，你的核心论据是什么？

**源码引用**:
- `src/kimi_cli/soul/agent.py:200-280` (LaborMarket)
- `src/kimi_cli/tools/multiagent/task.py` (Task 工具)

**参考答案**:

**1. vs LangGraph**

| 维度 | LangGraph | kimi-cli |
|------|-----------|----------|
| 建模方式 | 显式状态机 | 隐式 ReAct 循环 |
| 优势 | 可视化、可验证 | 简单、灵活 |
| 劣势 | 复杂、学习曲线陡 | 难以验证 |

**异同**：
- **相同**：都是"状态 + 转换"的模型
- **不同**：LangGraph 显式定义状态，kimi-cli 隐式表达

**2. vs AutoGen**

| 维度 | AutoGen | kimi-cli |
|------|---------|----------|
| 协作模式 | 对话驱动 | 层级分配 |
| 优势 | 灵活、自组织 | 清晰、可控 |
| 劣势 | 可能无限对话 | 父 Agent 成为瓶颈 |

**优劣对比**：
- **AutoGen**：适合探索性场景（如研究）
- **kimi-cli**：适合任务明确的场景（如代码审查）

**3. vs CrewAI**

| 维度 | CrewAI | kimi-cli |
|------|--------|----------|
| Agent 组织 | 预定义角色 | Fixed + Dynamic |
| 优势 | 简单、易用 | 灵活、可扩展 |
| 劣势 | 不够灵活 | 需要更多配置 |

**设计哲学差异**：
- **CrewAI**：假设所有 Agent 可以预定义
- **kimi-cli**：承认某些 Agent 需要动态创建

**4. 说服公司用 kimi-cli 的核心论据**

1. **简单性**：学习曲线低，新人快速上手
2. **可维护性**：代码结构清晰，易于调试
3. **成本**：自研 = 控制成本，无需支付框架授权费

---

### Q5: 架构演进路径

**难度**: 四星 | **类别**: 演化设计

假设 kimi-cli 要从个人工具演进为企业级 Agent 平台：

1. 需要新增哪些核心组件？（考虑多租户、权限、审计、计费）
2. 现有架构中哪些部分会成为瓶颈？
3. 如果要支持 Agent App Store（第三方发布 Agent），需要什么样的 API 和安全边界？
4. 演进过程中，如何保证向后兼容？

**参考答案**:

**1. 需要新增的核心组件**

```
┌─────────────────────────────────────────────────────┐
│                  API Gateway                         │
│  负责认证、限流、路由                                   │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│                  Tenant Manager                      │
│  负责租户隔离、权限控制                                 │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│                  Soul Pool                           │
│  管理多个 Soul 实例                                    │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│              Audit & Billing                         │
│  负责审计日志、计费                                     │
└─────────────────────────────────────────────────────┘
```

**2. 现有架构的瓶颈**

1. **单机 Soul**：无法横向扩展，需要改造为 Soul Pool
2. **内存 Context**：无法持久化，需要改造为 Redis-backed Context
3. **同步工具调用**：无法并发，需要改造为异步工具执行

**3. Agent App Store 的 API 和安全边界**

**API 设计**：
```python
# Agent 规格
{
  "name": "code-reviewer",
  "version": "1.0.0",
  "description": "自动审查代码",
  "system_prompt": "...",
  "tools": ["file.read", "github.api"],
  "permissions": {
    "file": ["read"],
    "github": ["read", "write"]
  }
}
```

**安全边界**：
1. **沙箱**：Agent 运行在受限环境
2. **权限声明**：Agent 必须声明需要的权限
3. **用户审批**：首次运行需要用户确认

**4. 向后兼容**

策略：
1. **版本化 API**：v1/v2/v3 共存
2. **适配器模式**：旧 API 转换为新 API
3. **弃用警告**：提前通知用户

---

## Agent 核心 (4题)

---

### Q6: Agent 循环与状态管理

**难度**: 五星 | **类别**: Agent 核心

`KimiSoul._run_turn()` 是系统的心脏。请回答：

1. 请画出完整的状态转换图：从用户输入到最终响应，所有可能的中间状态和转换条件
2. max_steps 限制是多少？如何避免无限循环？
3. 当工具被用户拒绝时，Agent 的停止策略是什么？这种设计是否会导致 deadlock？
4. 如果要实现流式工具调用（工具执行过程中返回部分结果），需要修改哪些代码？

**源码引用**:
- `src/kimi_cli/soul/kimisoul.py:120-200` (主循环)
- `src/kimi_cli/soul/context.py` (状态管理)

**参考答案**:

**1. 完整的状态转换图**

```
┌─────────┐
│  IDLE   │ ← 初始状态
└────┬────┘
     │ 用户输入
     ▼
┌─────────────┐
│  THINKING   │ ← LLM 推理中
└──────┬──────┘
       │ LLM 返回工具调用
       ▼
┌──────────────────┐
│ TOOL_APPROVAL    │ ← 等待用户审批
└────┬────────┬────┘
     │        │
     │ 批准   │ 拒绝
     ▼        ▼
┌──────────┐  ┌─────────────┐
│ TOOL_    │  │ INTERRUPTED │
│ EXECUTING│  └──────┬──────┘
└────┬─────┘         │
     │ 工具完成       │ 停止
     ▼                ▼
┌─────────────┐    ┌─────────┐
│  THINKING   │    │  IDLE   │
└──────┬──────┘    └─────────┘
       │
       │ LLM 返回文本响应
       ▼
┌─────────────┐
│  COMPLETED  │ ← 任务完成
└──────┬──────┘
       │
       ▼
┌─────────┐
│  IDLE   │ ← 返回初始状态
└─────────┘
```

**2. max_steps 限制**

默认值：10

避免无限循环的策略：
1. **硬性限制**：max_steps 防止无限循环
2. **检测重复**：如果 LLM 重复调用同一工具，自动停止
3. **用户中断**：用户可以随时 Ctrl+C 中断

**3. 工具被拒绝时的停止策略**

当前设计：立即 break，返回 INTERRUPTED 状态

**潜在 deadlock**：不会，因为：
1. 用户拒绝是最终决定，Agent 无法"争辩"
2. BREAK 后清理资源，不会卡住

**改进建议**：
- 允许 Agent"申诉"一次，解释为什么需要这个工具

**4. 流式工具调用**

需要修改的代码：

```python
# 修改 1: ToolReturnValue 支持流式
@dataclass
class ToolReturnValue:
    output: str | dict | list | AsyncIterator  # 新增 AsyncIterator

# 修改 2: _execute_tools 支持流式
async def _execute_tools(self, tool_calls: list[ToolCall]):
    for tool_call in tool_calls:
        result_stream = await self._toolset.run_stream(tool_call)

        # 流式添加到上下文
        async for chunk in result_stream:
            self._context.add_message(
                Message(role="tool", content=chunk)
            )
            # 立即发送给 LLM
            await self._stream_to_llm(chunk)

# 修改 3: kosong.step 支持流式输入
async def step_stream(
    messages: AsyncIterator[Message],
    ...
) -> AsyncIterator[Message]:
    async for msg in messages:
        # 增量处理
        ...
```

**复杂性**：
- Token 计费：如何计费流式输出？
- 中断处理：用户中断时如何清理？
- 上下文管理：如何表示"部分结果"？

---

### Q7: Context 压缩策略

**难度**: 四星 | **类别**: 状态管理

当上下文过长时，你使用 LLM 进行历史压缩。请回答：

1. 压缩算法的核心决策：保留多少条消息？如何保证压缩后的信息密度？
2. 相比滑动窗口、向量检索、CRDT 摘要等方案，LLM 压缩的优势和劣势是什么？
3. 如果压缩后的 LLM 输出丢失了关键上下文，如何检测和恢复？
4. kimi-cli 的压缩策略是否适用于长期记忆（Agent 运行数月）？为什么？

**源码引用**:
- `src/kimi_cli/soul/compaction.py:42-117` (SimpleCompaction)

**参考答案**:

**1. 压缩算法的核心决策**

**保留策略**：
1. **system 消息**：始终保留（定义 Agent 行为）
2. **最近 N 条**：保留最近 20 条（keep_recent=20）
3. **中间的**：用 LLM 压缩成摘要

**信息密度保证**：
```python
prompt = """请总结以下对话历史，保留关键信息：

1. 用户的主要诉求
2. 讨论过的方案
3. 最终结论
4. 重要的约定或决策
"""
```

**2. 对比其他方案**

| 方案 | 优势 | 劣势 |
|------|------|------|
| 滑动窗口 | 简单，无成本 | 硬性丢弃，可能丢失关键信息 |
| 向量检索 | 保留语义相关片段 | 需要额外存储和检索，增加延迟 |
| CRDT 摘要 | 结构化，可恢复 | 实现复杂，通用性差 |
| **LLM 压缩** | 保留语义，通用性强 | 有成本，可能丢失细节 |

**选择 LLM 压缩的原因**：
1. 适用于任意对话场景
2. 成本可接受（摘要通常只需要几百 tokens）
3. 实现简单

**3. 检测和恢复**

**检测**：
- 如果压缩后 LLM 重复问同一问题，说明丢失了关键信息
- 如果 LLM 输出质量下降，说明压缩过于激进

**恢复**：
1. 增加 keep_recent 数量
2. 使用向量检索保留关键历史片段
3. 手动恢复 checkpoint

**4. 是否适用于长期记忆**

不适用。长期记忆需要：

1. **持久化存储**：将重要信息写入数据库
2. **语义检索**：根据当前问题检索相关历史
3. **层次化记忆**：
   - 工作记忆（当前会话）← SimpleCompaction
   - 短期记忆（最近几天）← 需要
   - 长期记忆（所有历史）← 需要

---

### Q8: 工具注册与依赖注入

**难度**: 四星 | **类别**: 依赖注入

`KimiToolset.load_tools()` 使用 `inspect.signature` 实现依赖注入。请回答：

1. 为什么不用 DI 容器？自研 DI 的边界在哪里？
2. 如何解决循环依赖？如果 Tool A 需要 Tool B，Tool B 又需要 Tool A，会发生什么？
3. keyword-only 参数作为注入边界的理由是什么？这种约定是否过于隐式？
4. 如果工具需要延迟加载（仅在首次调用时初始化），如何实现？

**源码引用**:
- `src/kimi_cli/soul/toolset.py:80-150` (工具加载)

**参考答案**:

**1. 为什么不用 DI 容器**

DI 容器（如 dependency-injector）的优势：
1. 自动解析依赖图
2. 生命周期管理
3. 配置化

kimi-cli 的场景：
- 依赖关系是线性的
- 所有依赖都是 singleton
- 不需要热替换

**自研 DI 的边界**：
当出现以下需求时，应该使用成熟的 DI 框架：
1. 循环依赖
2. 生命周期管理（transient、scoped）
3. AOP 支持

**2. 循环依赖**

当前实现不支持循环依赖，会抛出异常：

```python
# Tool A 需要 Tool B
async def tool_a(b: ToolB): ...

# Tool B 需要 Tool A
async def tool_b(a: ToolA): ...

# 结果：ValueError: Circular dependency detected
```

**解决方案**：
1. **重构**：将公共依赖提取到第三个模块
2. **延迟注入**：通过函数参数延迟注入
3. **使用 DI 容器**：自动处理循环依赖

**3. keyword-only 参数**

约定：
- **位置参数**：用户输入（如 `path: str`）
- **关键字参数**：注入的依赖（如 `runtime: Runtime`）

**好处**：
1. 类型安全：IDE 可以提示用户需要输入哪些参数
2. 清晰性：一眼看出哪些是用户输入，哪些是系统注入
3. 扩展性：新增依赖不影响用户调用

**是否过于隐式**：
- 对于新用户，可能不清楚哪些参数会被自动注入
- 改进：在文档中明确说明，或在函数签名中用注释标注

**4. 延迟加载**

```python
def lazy_tool(name: str, factory: Callable[[], Any]):
    """创建懒加载的工具"""
    value = None

    async def get():
        nonlocal value
        if value is None:
            value = await factory()
        return value

    return get

# 使用
@lazy_tool("database", connect_to_database)
async def query_db(db: Database = UseLazy("database")):
    return await db.query("SELECT * FROM users")
```

---

### Q9: 工具输出的三层语义

**难度**: 三星 | **类别**: 数据建模

`ToolReturnValue` 包含 `output`（给 LLM）、`message`（解释）、`display`（给用户）三层。请回答：

1. 为什么分离这三层？每层的语义目标是什么？
2. 如果工具返回的是流式数据（如 `tail -f`），如何适配现有模型？
3. 是否考虑过使用 Protocol/Structural Typing 而非继承？为什么？
4. 这三层输出的设计哲学，与 REST API 的响应设计有何共通之处？

**源码引用**:
- `packages/kosong/src/kosong/tooling/base.py` (ToolReturnValue)

**参考答案**:

**1. 为什么分离这三层**

每层的语义目标不同：

| 层 | 目标受众 | 语义目标 | 示例 |
|---|---------|---------|------|
| output | LLM | 结构化数据，用于推理 | `{"files": ["a.py", "b.py"]}` |
| message | 开发者 | 自然语言，便于调试 | "列出当前目录的 Python 文件" |
| display | 用户 | 渲染友好，可读性强 | "**Found 2 files:**<br>- a.py<br>- b.py" |

**分离的好处**：
1. **LLM 得到结构化数据**：易于解析和推理
2. **开发者得到自然语言**：易于调试
3. **用户得到渲染友好**：易于理解

**2. 流式数据适配**

当前模型不支持流式。如果要支持：

```python
@dataclass
class ToolReturnValue:
    output: str | dict | list | AsyncIterator  # 新增 AsyncIterator

# 流式工具
async def tail_file(path: str) -> ToolReturnValue:
    async def stream_output():
        async for line in tail_f(path):
            yield {"line": line}

    return ToolReturnValue(
        output=stream_output(),  # AsyncIterator
        message="实时监控文件",
        display="正在监控文件...",
    )
```

**复杂性**：
1. Context 如何表示"部分结果"？
2. Token 如何计费？
3. 用户中断时如何清理流？

**3. Protocol vs 继承**

**继承**（当前方案）：
```python
@dataclass
class ToolReturnValue:
    output: str | dict | list
    ...
```

**Protocol**（替代方案）```python
from typing import Protocol

class ToolReturnValue(Protocol):
    @property
    def output(self) -> str | dict | list: ...

    @property
    def message(self) -> str | None: ...

    @property
    def display(self) -> str | None: ...
```

**为什么选择继承**：
1. **明确性**：类型明确，IDE 支持更好
2. **简单性**：不需要理解 Protocol 的概念
3. **性能**：继承的实现更高效

**4. 与 REST API 的共通之处**

| REST API | ToolReturnValue |
|----------|-----------------|
| `data` (结构化数据) | `output` |
| `message` (人类可读) | `message` |
| `view` (渲染模板) | `display` |

**设计哲学**：
- **分层输出**：不同受众需要不同的表示
- **单一职责**：每层只服务于一个目标
- **可扩展性**：可以新增更多层（如 `debug`、`audit`）

---

## 工具系统 (3题)

---

### Q10: MCP 集成与工具生态

**难度**: 四星 | **类别**: 协议集成

你通过 `fastmcp` 集成了 Model Context Protocol。请回答：

1. MCP 的核心价值是什么？它解决了什么问题？
2. MCP 工具与内置工具在安全权限上应该有什么区别？
3. 如果 MCP Server 无响应或返回恶意数据，如何保护 Agent？
4. 让 kimi-cli 作为 MCP Server 暴露自己的工具，需要实现什么？这会带来什么商业机会？

**源码引用**:
- `src/kimi_cli/mcp.py:45-120` (MCP 管理)

**参考答案**:

**1. MCP 的核心价值**

解决"工具孤岛"问题：

**没有 MCP**：
```
Agent A 需要集成 GitHub API → 自己写 GitHub 工具
Agent B 需要集成 GitHub API → 重复写 GitHub 工具
Agent C 需要集成 GitHub API → 又重复写...
```

**有了 MCP**：
```
GitHub 官方发布 MCP Server → 所有 Agent 都能使用
```

**核心价值**：
1. **工具生态**：第三方可以发布 MCP Server
2. **跨平台**：同一套工具可以在不同 Agent 框架中使用
3. **安全隔离**：MCP Server 运行在独立进程

**2. 安全权限区别**

| 维度 | 内置工具 | MCP 工具 |
|------|----------|----------|
| 信任度 | 高（自己写的） | 低（第三方） |
| 权限 | 完全访问 | 限制访问 |
| 隔离 | 同进程 | 独立进程 |
| 审计 | 无需审计 | 需要审计 |

**安全措施**：
1. **沙箱**：MCP Server 运行在受限环境
2. **权限控制**：每个 Server 需要声明需要的权限
3. **审批机制**：首次调用 MCP 工具需要用户确认

**3. 保护 Agent**

**防护措施**：

1. **超时机制**：
```python
async def call_tool(self, ...):
    try:
        result = await asyncio.wait_for(
            server.call_tool(...),
            timeout=30,  # 30 秒超时
        )
    except asyncio.TimeoutError:
        return ToolReturnValue.error("Tool timeout")
```

2. **输入验证**：
```python
if not isinstance(result, dict):
    return ToolReturnValue.error("Invalid response format")
```

3. **沙箱隔离**：MCP Server 运行在独立进程

**4. kimi-cli 作为 MCP Server**

**需要实现**：
```python
class KimiMCPServer:
    async def list_tools(self) -> list[Tool]:
        """列出 kimi-cli 的所有工具"""
        return [
            Tool(name="file.list", description="列出文件"),
            Tool(name="file.read", description="读取文件"),
            ...
        ]

    async def call_tool(self, name: str, args: dict) -> ToolResult:
        """调用工具"""
        tool = self._toolset.get(name)
        result = await tool(**args)
        return ToolResult(content=result.output)
```

**商业机会**：
1. **工具商店**：用户可以发布自己的 Agent 工具
2. **订阅模式**：高级工具需要订阅
3. **分成机制**：工具作者可以获得收益

---

## Multi-Agent (2题)

---

### Q11: 分形 Agent 架构 vs 业界模式

**难度**: 五星 | **类别**: 分布式系统

你设计了 Fixed 和 Dynamic 两种子 Agent。请回答：

1. Fixed vs Dynamic 的选择标准：什么时候用预定义子 Agent，什么时候动态创建？
2. 与 Microsoft AutoGen 的对话驱动多 Agent 相比，你的分形架构的独特价值是什么？
3. 子 Agent 有独立的 Runtime 和 Context，但共享 LaborMarket。这种设计的一致性风险在哪里？
4. 如果子 Agent 陷入无限循环，父 Agent 如何检测和中断？

**源码引用**:
- `src/kimi_cli/soul/agent.py:200-280` (LaborMarket)
- `src/kimi_cli/tools/multiagent/task.py:52-150` (Task 工具)

**参考答案**:

**1. Fixed vs Dynamic 的选择标准**

```
任务是否可预见？
├─ 是 → 使用 Fixed Agent
│   └─ 原因：性能更好，可复用，可优化
└─ 否 → 使用 Dynamic Agent
    └─ 原因：灵活性高，适应性强
```

**示例**：

| 任务类型 | Agent 类型 | 原因 |
|----------|-----------|------|
| 代码审查 | Fixed | 任务明确，可预定义审查规则 |
| 研究"量子计算" | Dynamic | 任务不确定，需要动态生成研究策略 |

**2. vs AutoGen 的独特价值**

| 维度 | kimi-cli 分形架构 | AutoGen 对话模型 |
|------|------------------|------------------|
| 协作模式 | 层级（父 Agent 分配任务） | 对话（Agent 互相协商） |
| 优势 | 清晰的责任边界，易于调试 | 灵活性高，自组织 |
| 劣势 | 父 Agent 成为瓶颈 | 可能陷入无限对话 |

**独特价值**：
1. **可预测性**：层级结构更容易理解
2. **可调试性**：父子关系清晰
3. **可扩展性**：可以动态添加新的子 Agent

**3. 一致性风险**

**共享 LaborMarket 的风险**：
1. **状态竞争**：多个子 Agent 可能同时修改 LaborMarket
2. **资源泄漏**：子 Agent 未正确释放资源
3. **依赖冲突**：子 Agent 依赖不同版本的 LaborMarket

**缓解措施**：
1. **锁机制**：保护共享状态
2. **引用计数**：自动释放资源
3. **版本管理**：隔离不同版本的依赖

**4. 检测和中断无限循环**

**检测**：
```python
class SubAgent:
    async def run(self, task: str, context: Context) -> str:
        step_count = 0
        while step_count < self.max_steps:
            # ...
            step_count += 1

            # 检测是否陷入循环
            if self._is_looping(context):
                raise LoopDetectedError("Sub agent stuck in loop")
```

**中断**：
```python
class ParentAgent:
    async def _run_sub_agent(self, task: str) -> str:
        try:
            result = await self.sub_agent.run(task, context)
        except LoopDetectedError:
            # 强制中断子 Agent
            await self.sub_agent.terminate()
            return "子 Agent 陷入循环，已中断"
```

---

### Q12: 跨 Agent 通信与协调

**难度**: 四星 | **类别**: 通信模式

1. 子 Agent 能否访问父 Agent 的历史？如果不能，如何传递必要上下文？
2. 子 Agent 的工具权限是继承还是独立的？如何控制能力边界？
3. 如果要实现子 Agent 之间的直接通信（非通过父 Agent），需要什么机制？
4. 场景设计：设计一个代码审查 Agent 系统，包含分析、测试、生成报告三个子 Agent。它们如何协作？

**源码引用**:
- `src/kimi_cli/tools/multiagent/task.py` (上下文隔离)

**参考答案**:

**1. 访问父 Agent 历史**

不能直接访问，但可以传递"相关上下文"：

```python
async def _extract_relevant_context(
    task: str,
    parent_context: Context,
) -> list[Message]:
    """从父 Agent 的历史中提取与任务相关的上下文"""
    # 策略 1: 只传递最近 N 条消息
    return parent_context.messages[-10:]

    # 策略 2: 使用向量检索
    # relevant = vector_store.search(task)
    # return [parent_context.messages[i] for i in relevant]
```

**2. 工具权限**

独立的。每个 Agent 有自己的工具集：

```python
# 父 Agent：有所有工具
parent_agent = Agent(tools=[file_tool, web_tool, database_tool])

# 子 Agent：只有部分工具
sub_agent = Agent(tools=[file_tool])  # 只能操作文件
```

**权限控制**：
1. **最小权限原则**：只给必需的工具
2. **审批机制**：使用危险工具时需要审批
3. **审计日志**：记录所有操作

**3. 子 Agent 之间直接通信**

需要实现消息总线：

```python
class AgentMessageBus:
    """Agent 间通信的消息总线"""

    def __init__(self):
        self._channels: dict[str, list[Agent]] = {}

    def subscribe(self, agent: Agent, channel: str) -> None:
        """Agent 订阅频道"""
        if channel not in self._channels:
            self._channels[channel] = []
        self._channels[channel].append(agent)

    async def publish(self, channel: str, message: Message) -> None:
        """发布消息到频道"""
        agents = self._channels.get(channel, [])
        for agent in agents:
            await agent.receive(message)
```

**4. 代码审查 Agent 系统**

```python
# 三个子 Agent
analyzer_agent = Agent(name="Analyzer", role="分析代码")
tester_agent = Agent(name="Tester", role="运行测试")
reporter_agent = Agent(name="Reporter", role="生成报告")

# 协作流程
analyzer_agent.publish("code.analyzed", result)
tester_agent.subscribe("code.analyzed")  # 收到分析结果，开始测试
tester_agent.publish("tests.completed", results)
reporter_agent.subscribe("tests.completed")  # 收到测试结果，生成报告
```

---

## 场景设计 (1题)

---

### Q13: 系统设计实战

**难度**: 五星 | **类别**: 系统设计

场景：一家 GitHub 上的开源项目希望接入 AI Agent，实现"自动修复 Issue"的功能。

具体需求：
- 监听 GitHub Issues
- 分析 Issue 描述和相关代码
- 生成修复代码
- 创建 Pull Request
- 如果测试失败，自动迭代

请回答：

1. Agent 架构：你会设计几个 Agent？它们如何分工？（单一 Agent vs 多 Agent）
2. 工具选择：需要哪些工具？（GitHub API、代码分析、测试运行）
3. 安全边界：如何防止 Agent"恶意修改"代码？（审批、沙箱、权限控制）
4. 评估体系：如何判断 Agent 生成的 PR 是"好"的？（代码审查、测试通过率、人工反馈）
5. 失败恢复：如果 Agent 生成的代码导致测试失败，如何自动修复？

**参考答案**:

**1. Agent 架构**

推荐 **Multi-Agent**：

```
┌─────────────────────────────────────────────────────┐
│                    OrchestratorAgent                 │
│  负责任务分配、进度跟踪、最终决策                     │
└─────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ AnalyzerAgent│    │ CoderAgent   │    │ TesterAgent  │
│ 分析 Issue   │───→│ 生成修复代码 │───→│ 运行测试     │
└──────────────┘    └──────────────┘    └──────────────┘
         │                                      │
         └──────────┬───────────────────────────┘
                    ▼
            ┌──────────────┐
            │ReporterAgent │
            │ 生成 PR 描述 │
            └──────────────┘
```

**原因**：
- 单一 Agent：逻辑复杂，难以维护
- Multi-Agent：职责清晰，易于扩展

**2. 工具选择**

| Agent | 工具 |
|-------|------|
| OrchestratorAgent | github.list_issues, github.create_pr |
| AnalyzerAgent | github.read_issue, file.read, code.analyze |
| CoderAgent | file.write, code.generate |
| TesterAgent | test.run, git.create_branch |
| ReporterAgent | github.create_pr, markdown.format |

**3. 安全边界**

1. **沙箱执行**：所有代码修改在临时分支进行
2. **人工审批**：创建 PR 前需要人工确认
3. **测试门禁**：必须通过所有测试才能创建 PR
4. **回滚机制**：如果 PR 被拒绝，自动回滚分支

**4. 评估体系**

| 指标 | 说明 |
|------|------|
| 测试通过率 | 所有测试必须通过 |
| 代码审查 | 自动运行 linter 和 type checker |
| 人工反馈 | PR 被合并的频率 |
| 修复时间 | 从 Issue 创建到 PR 合并的时间 |

**5. 失败恢复**

```python
class TesterAgent:
    async def run_tests(self, code: str) -> TestResult:
        result = await self._run_tests(code)

        if result.failed:
            # 分析失败原因
            cause = await self._analyze_failure(result.output)

            # 迭代修复
            for attempt in range(3):
                fixed_code = await self._coder_agent.fix(code, cause)
                new_result = await self._run_tests(fixed_code)

                if new_result.passed:
                    return new_result

            # 3 次失败后放弃
            return TestResult.failed("Max retries exceeded")

        return result
```

---

## Bonus 挑战题

---

### Bonus 1: 技术债务与重构

如果给你 2 周时间重构 kimi-cli 的一个模块，你会选择哪个？为什么？

**参考答案**:

我会选择重构 **Context 管理**模块。

**原因**：
1. **当前问题**：Context 全部存储在内存中，无法持久化
2. **影响**：Agent 重启后丢失所有历史
3. **重构方向**：Redis-backed Context，支持持久化和分布式

**具体方案**：
```python
class RedisContext(Context):
    def __init__(self, redis_url: str):
        self._redis = redis.from_url(redis_url)

    def add_message(self, message: Message) -> None:
        # 写入 Redis
        self._redis.lpush("messages", message.json())

    def get_messages(self) -> list[Message]:
        # 从 Redis 读取
        data = self._redis.lrange("messages", 0, -1)
        return [Message.parse_raw(d) for d in data]
```

---

### Bonus 2: 未来趋势预测

未来 3 年，AI Agent 领域最重要的技术突破会是什么？kimi-cli 如何准备？

**参考答案**:

**最重要的突破**：**Agent 间通信协议的标准化**

类似于 HTTP 之于 Web，MCP 之于工具，Agent 通信协议将使不同 Agent 能够无缝协作。

**kimi-cli 的准备**：
1. **积极参与标准制定**：加入 ACP/MCP 标准化组织
2. **提前布局**：在架构中预留协议扩展点
3. **生态建设**：发布更多的 MCP Server

---

### Bonus 3: 如果用 Rust 重写

如果用 Rust 重写 kimi-cli 的核心模块，你会选择哪些部分？Rust 会带来什么本质优势？

**参考答案**:

**选择重写的模块**：
1. **LLM 调用层**（Kosong）：需要高性能和并发
2. **工具执行引擎**：需要稳定性和安全性
3. **MCP 协议实现**：需要内存安全

**Rust 的本质优势**：
1. **内存安全**：无 GC，避免内存泄漏
2. **并发性能**：零成本抽象，真正的多线程
3. **类型安全**：编译时检查，减少运行时错误

**权衡**：
- **开发速度**：Rust 开发速度慢于 Python
- **生态成熟度**：Python 的 AI 生态更成熟
- **学习曲线**：Rust 学习曲线陡峭

**结论**：核心模块用 Rust，外层用 Python（PyO3 绑定）
