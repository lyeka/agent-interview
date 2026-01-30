# 第二层：单机 Agent 核心

> 本章深入 KimiSoul 主循环，理解 Agent 如何处理用户输入、调用 LLM、执行工具。

---

## 2.1 KimiSoul 主循环

### 架构图

```
用户输入 "列出当前目录的 Python 文件"
   ↓
┌─────────────────────────────────────────────────────┐
│         KimiSoul._run_turn(user_input)              │
│                                                      │
│  Step 1: 添加到 Context                              │
│  ┌─────────────────────────────────────┐            │
│  │ context.add_message(                 │            │
│  │   Message(role="user", content=...)  │            │
│  │ )                                    │            │
│  └─────────────────────────────────────┘            │
│                 ↓                                    │
│  Step 2: ReAct 循环 (max_steps=10)                   │
│  ┌─────────────────────────────────────┐            │
│  │ while step_count < max_steps:       │            │
│  │   result = await kosong.step(...)   │            │
│  │                                     │            │
│  │   if result.message.tool_calls:     │            │
│  │     # 用户审批                      │            │
│  │     if approved:                    │            │
│  │       await _execute_tools(...)     │            │
│  │     else:                           │            │
│  │       break  # 停止循环             │            │
│  │   else:                             │            │
│  │     break  # LLM 完成响应           │            │
│  │                                     │            │
│  │   step_count += 1                   │            │
│  └─────────────────────────────────────┘            │
│                 ↓                                    │
│  Step 3: Context 压缩（如果过长）                     │
│  ┌─────────────────────────────────────┐            │
│  │ if context.length > threshold:      │            │
│  │   await _compact_context()          │            │
│  └─────────────────────────────────────┘            │
│                 ↓                                    │
│  返回 TurnOutcome                                     │
└─────────────────────────────────────────────────────┘
```

### 核心代码

**源码位置**: `src/kimi_cli/soul/kimisoul.py:120-200`

```python
async def _run_turn(self, user_input: str) -> TurnOutcome:
    # 1. 添加用户输入到上下文
    self._context.add_message(Message(role="user", content=user_input))

    # 2. ReAct 循环
    step_count = 0
    while step_count < self._loop_control.max_steps:
        # 调用 LLM
        result = await kosong.step(
            model=self._model,
            messages=self._context.messages,
            tools=self._toolset.tools,
        )

        # 添加 LLM 响应到上下文
        self._context.add_message(result.message)

        # 判断是否有工具调用
        if result.message.tool_calls:
            # 用户审批（交互式）
            approval = await self._approve_tools(result.message.tool_calls)

            if not approval.approved:
                # 用户拒绝，停止循环
                return TurnOutcome.interrupted(approval.reason)

            # 执行工具
            await self._execute_tools(result.message.tool_calls)

            # 继续循环，让 LLM 基于工具结果继续思考
        else:
            # LLM 完成响应，退出循环
            break

        step_count += 1

    # 3. 返回最终响应
    final_message = self._context.messages[-1]
    return TurnOutcome.success(final_message)
```

### 设计分析

**Q: 为什么用 ReAct 循环而非一次性生成？**

A: Agent 需要通过工具调用与环境交互，单次生成无法支持多步推理。

ReAct = Reasoning + Acting

1. **Reasoning**：LLM 分析当前状态，决定下一步行动
2. **Acting**：执行工具，获取环境反馈
3. **Observation**：基于反馈继续推理

例如"帮我查一下今天的天气，然后决定穿什么"：

```
Step 1: LLM 决定调用 weather 工具
Step 2: 工具返回"今天晴，25°C"
Step 3: LLM 基于结果决定"建议穿 T 恤"
```

一次性生成无法实现这种多步交互。

**Q: max_steps 的作用是什么？如何避免无限循环？**

A: max_steps 防止 Agent 陷入无限循环。例如：

```
LLM: 我需要搜索 "cat"
Tool: 返回关于 cat 的结果
LLM: 我需要再搜索 "cat"（陷入循环）
```

kimi-cli 默认 max_steps=10，可根据任务复杂度调整。如果超过限制仍未完成，Agent 会停止并返回错误。

**Q: 当工具被用户拒绝时，停止策略是什么？**

A: 立即停止循环，返回 interrupted 状态。这可能导致"半成品"响应，但保护了用户控制权。

潜在问题：如果工具 A 拒绝后，工具 B 的结果已经处理，可能导致不一致。但 kimi-cli 的设计中，工具是批量审批的，要么全部执行，要么全部不执行。

**Q: 如果要实现流式工具调用（工具执行过程中返回部分结果），需要修改哪些代码？**

A: 需要修改三处：

1. `_execute_tools`: 改为流式处理
2. `context.add_message`: 支持增量更新
3. `kosong.step`: 支持流式输入

但流式会带来复杂性：
- Token 计费：部分结果的 token 如何计算？
- 中断处理：用户中断时如何清理已执行的工具？
- 上下文管理：如何表示"部分结果"？

---

## 2.2 工具执行

### 核心代码

**源码位置**: `src/kimi_cli/soul/kimisoul.py:145-167`

```python
async def _execute_tools(self, tool_calls: list[ToolCall]):
    for tool_call in tool_calls:
        # 为什么遍历而非并发？
        # A: 工具间可能有依赖关系，串行保证顺序
        result = await self._toolset.run(tool_call)

        # 为什么这里立即添加到上下文？
        # A: 让后续工具能看到前面的结果
        self._context.add_message(tool_result_to_message(result))
```

### 设计分析

**Q: 为什么串行执行而非并发？**

A: 三个原因：

1. **依赖关系**：工具 B 可能依赖工具 A 的结果
2. **上下文顺序**：串行保证结果按顺序添加到上下文
3. **错误处理**：串行更容易处理失败和重试

如果要支持并发，需要：
- 分析工具间的依赖关系
- 处理并发失败的场景
- 保证上下文的最终一致性

**Q: 工具失败时如何处理？**

A: 工具返回 `ToolReturnValue(is_error=True)`，LLM 可以基于错误信息决定下一步：

```python
# 工具返回错误
ToolReturnValue(
    output={"error": "File not found"},
    message="文件不存在",
    is_error=True
)

# LLM 可以决定：
# 1. 重试（可能是临时错误）
# 2. 改用其他工具
# 3. 告知用户失败
```

---

## 2.3 状态管理

### 状态转换图

```
IDLE
  ↓ 用户输入
THINKING
  ↓ LLM 返回工具调用
TOOL_APPROVAL
  ├─ 批准 → TOOL_EXECUTING
  │            ↓ 工具完成
  │         THINKING (继续循环)
  │
  └─ 拒绝 → INTERRUPTED
               ↓ 停止
           IDLE

THINKING
  ↓ LLM 返回文本响应
COMPLETED
  ↓ 返回给用户
IDLE
```

### 设计分析

**Q: 如何保证状态转换的一致性？**

A: kimi-cli 使用单一入口点（`_run_turn`）保证状态转换的线性：

1. 每次用户输入开始新的 turn
2. Turn 内部状态是线性的（THINKING → TOOL_APPROVAL → THINKING → ...）
3. Turn 结束后返回 IDLE

这种设计避免了复杂的状态机，但也限制了并发能力。

---

## 本章小结

KimiSoul 的核心设计：

1. **ReAct 循环**：推理-行动-观察的循环，实现多步推理
2. **用户审批**：工具执行前需要用户确认，保证安全性
3. **保护机制**：max_steps 防止无限循环
4. **串行执行**：工具按顺序执行，保证依赖关系和上下文一致性

相比复杂的流式或并发设计，kimi-cli 选择了简单可靠的串行模型。
