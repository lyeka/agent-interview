# 03 - Agent 循环与 Step 机制

## 本章目标

理解 kimi-cli 的 Agent 循环实现，掌握 Step-based 执行模型、重试机制和停止条件。

## Agent 循环概述

kimi-cli 的 Agent 循环采用 **Step-based** 设计，每个 Step 对应一次 LLM 调用 + 工具执行。这种设计的优势：

1. **粒度适中**：便于监控、中断、重试
2. **状态可控**：每个 step 前创建 checkpoint，支持回退
3. **资源可控**：通过 max_steps_per_turn 限制单轮最大步数

```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent 循环模型                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Turn (一轮对话)                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                         │   │
│  │  Step 1          Step 2          Step 3      ...       │   │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐            │   │
│  │  │ LLM Call│    │ LLM Call│    │ LLM Call│            │   │
│  │  │ + Tools │    │ + Tools │    │ (no tool)│            │   │
│  │  └────┬────┘    └────┬────┘    └────┬────┘            │   │
│  │       │              │              │                  │   │
│  │       ▼              ▼              ▼                  │   │
│  │  has_tool_calls  has_tool_calls  no_tool_calls        │   │
│  │  → continue      → continue      → STOP               │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 核心实现

### _agent_loop 主循环

```python
# src/kimi_cli/soul/kimisoul.py:306-385
async def _agent_loop(self) -> TurnOutcome:
    """The main agent loop for one run."""
    assert self._runtime.llm is not None

    # 等待 MCP 工具加载完成
    if isinstance(self._agent.toolset, KimiToolset):
        await self._agent.toolset.wait_for_mcp_tools()

    # Approval 请求管道：后台持续处理批准请求
    async def _pipe_approval_to_wire():
        while True:
            request = await self._approval.fetch_request()
            wire_request = ApprovalRequest(
                id=request.id,
                tool_call_id=request.tool_call_id,
                action=request.action,
                description=request.description,
                display=request.display,
            )
            wire_send(wire_request)
            resp = await wire_request.wait()
            self._approval.resolve_request(request.id, resp)

    step_no = 0
    while True:
        step_no += 1

        # 步数限制检查
        if step_no > self._loop_control.max_steps_per_turn:
            raise MaxStepsReached(self._loop_control.max_steps_per_turn)

        wire_send(StepBegin(n=step_no))
        approval_task = asyncio.create_task(_pipe_approval_to_wire())
        back_to_the_future: BackToTheFuture | None = None
        step_outcome: StepOutcome | None = None

        try:
            # Context 压缩检查
            reserved = self._loop_control.reserved_context_size
            if self._context.token_count + reserved >= self._runtime.llm.max_context_size:
                logger.info("Context too long, compacting...")
                await self.compact_context()

            # 创建 checkpoint
            await self._checkpoint()
            self._denwa_renji.set_n_checkpoints(self._context.n_checkpoints)

            # 执行单步
            step_outcome = await self._step()

        except BackToTheFuture as e:
            back_to_the_future = e
        except Exception:
            wire_send(StepInterrupted())
            raise
        finally:
            approval_task.cancel()

        # 处理 D-Mail 时间旅行
        if back_to_the_future is not None:
            await self._context.revert_to(back_to_the_future.checkpoint_id)
            for msg in back_to_the_future.messages:
                await self._context.append_message(msg)
            step_no = 0  # 重置步数
            continue

        # 检查停止条件
        if step_outcome is not None:
            return TurnOutcome(
                stop_reason=step_outcome.stop_reason,
                final_message=step_outcome.assistant_message if step_outcome.stop_reason == "no_tool_calls" else None,
                step_count=step_no,
            )
```

**设计亮点**：

1. **Approval 异步管道**：`_pipe_approval_to_wire()` 在后台持续处理批准请求，解耦工具执行和用户交互
2. **异常驱动的时间旅行**：用 `BackToTheFuture` 异常实现 D-Mail，优雅地跳出循环并回退 context
3. **自动压缩触发**：在每个 step 开始前检查 token 使用量，主动触发压缩避免超限

### _step 单步执行

```python
# src/kimi_cli/soul/kimisoul.py:386-459
async def _step(self) -> StepOutcome | None:
    """Run a single step and return a stop outcome, or None to continue."""
    assert self._runtime.llm is not None
    chat_provider = self._runtime.llm.chat_provider

    # 带重试的 kosong.step 调用
    @tenacity.retry(
        retry=retry_if_exception(self._is_retryable_error),
        before_sleep=partial(self._retry_log, "step"),
        wait=wait_exponential_jitter(initial=0.3, max=5, jitter=0.5),
        stop=stop_after_attempt(self._loop_control.max_retries_per_step),
        reraise=True,
    )
    async def _kosong_step_with_retry() -> StepResult:
        return await kosong.step(
            chat_provider,
            self._agent.system_prompt,
            self._agent.toolset,
            self._context.history,
            on_message_part=wire_send,
            on_tool_result=wire_send,
        )

    result = await _kosong_step_with_retry()

    # 更新 token 计数
    if result.usage is not None:
        await self._context.update_token_count(result.usage.input)

    # 等待所有工具执行完成
    results = await result.tool_results()

    # 原子性保护 context 更新
    await asyncio.shield(self._grow_context(result, results))

    # 检查是否有工具被拒绝
    rejected = any(isinstance(r.return_value, ToolRejectedError) for r in results)
    if rejected:
        _ = self._denwa_renji.fetch_pending_dmail()
        return StepOutcome(stop_reason="tool_rejected", assistant_message=result.message)

    # 处理 D-Mail
    if dmail := self._denwa_renji.fetch_pending_dmail():
        raise BackToTheFuture(
            dmail.checkpoint_id,
            [Message(role="user", content=[system(
                "You just got a D-Mail from your future self. "
                f"D-Mail content:\n\n{dmail.message.strip()}"
            )])],
        )

    # 检查停止条件
    if result.tool_calls:
        return None  # 继续循环
    return StepOutcome(stop_reason="no_tool_calls", assistant_message=result.message)
```

**设计亮点**：

1. **Tenacity 重试机制**：对网络错误、超时、429/500 等错误自动重试，指数退避 + jitter 避免雷鸣群效应
2. **asyncio.shield 保护**：`_grow_context` 用 shield 包裹，确保 context 更新的原子性
3. **D-Mail 检测时机**：在工具执行完成后立即检查，确保 D-Mail 能在下一个 step 前生效

## 重试机制详解

### 可重试错误判断

```python
# src/kimi_cli/soul/kimisoul.py:461-485
def _is_retryable_error(self, e: BaseException) -> bool:
    """Check if an error is retryable."""
    if isinstance(e, ChatProviderError):
        match e:
            case APIConnectionError():
                return True
            case APITimeoutError():
                return True
            case APIStatusError(status_code=status_code):
                # 429: Rate limit, 500/502/503: Server error
                return status_code in (429, 500, 502, 503)
    return False
```

**可重试的错误类型**：
- `APIConnectionError`：网络连接失败
- `APITimeoutError`：请求超时
- `APIStatusError(429)`：速率限制
- `APIStatusError(500/502/503)`：服务器错误

### 重试策略

```python
@tenacity.retry(
    retry=retry_if_exception(self._is_retryable_error),
    before_sleep=partial(self._retry_log, "step"),
    wait=wait_exponential_jitter(initial=0.3, max=5, jitter=0.5),
    stop=stop_after_attempt(self._loop_control.max_retries_per_step),
    reraise=True,
)
```

**策略解析**：
- **指数退避**：初始 0.3 秒，最大 5 秒
- **Jitter**：添加 0.5 秒随机抖动，避免多个客户端同时重试
- **最大重试次数**：默认 3 次（`max_retries_per_step`）
- **重试前日志**：`before_sleep` 记录重试信息

## 停止条件

Agent 循环有两种正常停止条件：

### 1. no_tool_calls

LLM 响应中没有工具调用，表示任务完成。

```python
if result.tool_calls:
    return None  # 继续循环
return StepOutcome(stop_reason="no_tool_calls", assistant_message=result.message)
```

### 2. tool_rejected

用户拒绝了工具执行请求。

```python
rejected = any(isinstance(r.return_value, ToolRejectedError) for r in results)
if rejected:
    return StepOutcome(stop_reason="tool_rejected", assistant_message=result.message)
```

### 异常停止条件

- **MaxStepsReached**：超过单轮最大步数
- **BackToTheFuture**：D-Mail 触发时间旅行（不是真正的停止，而是回退重来）
- **其他异常**：网络错误、API 错误等

## Context 增长

每个 step 执行后，需要将 LLM 响应和工具结果追加到 context。

```python
# src/kimi_cli/soul/kimisoul.py:487-520
async def _grow_context(self, result: StepResult, tool_results: list[ToolResult]) -> None:
    """Grow the context with the step result."""
    # 追加 assistant 消息
    await self._context.append_message(result.message)

    # 追加工具结果
    for tool_result in tool_results:
        tool_message = Message(
            role="tool",
            tool_call_id=tool_result.tool_call_id,
            content=tool_result.return_value.to_content_parts(),
        )
        await self._context.append_message(tool_message)
```

**为什么用 asyncio.shield？**

```python
await asyncio.shield(self._grow_context(result, results))
```

如果用户在工具执行期间按 Ctrl+C，`_grow_context` 可能被中断，导致 context 状态不一致。`asyncio.shield` 确保即使外部取消，`_grow_context` 也会完成执行。

## LoopControl 配置

```python
# src/kimi_cli/soul/kimisoul.py:45-65
@dataclass
class LoopControl:
    """Control parameters for the agent loop."""

    max_steps_per_turn: int = 100
    """Maximum number of steps per turn."""

    max_retries_per_step: int = 3
    """Maximum number of retries per step."""

    reserved_context_size: int = 50000
    """Reserved context size for LLM generation."""
```

**配置说明**：
- `max_steps_per_turn`：单轮最大步数，防止无限循环
- `max_retries_per_step`：单步最大重试次数
- `reserved_context_size`：预留 context 空间（默认 50k tokens），为 LLM 生成留足空间

## 执行流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        _agent_loop 流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐                                                │
│  │ 等待 MCP    │                                                │
│  │ 工具加载    │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ step_no = 0 │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐     ┌─────────────┐                           │
│  │ step_no++   │────►│ > max_steps?│──Yes──► MaxStepsReached   │
│  └──────┬──────┘     └──────┬──────┘                           │
│         │                   │No                                 │
│         │◄──────────────────┘                                   │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ 检查 Context│                                                │
│  │ 是否需压缩  │──Yes──► compact_context()                      │
│  └──────┬──────┘                                                │
│         │No                                                     │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ checkpoint()│                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │   _step()   │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│    ┌────┴────┐                                                  │
│    │         │                                                  │
│    ▼         ▼                                                  │
│  BackTo   StepOutcome                                           │
│  Future      │                                                  │
│    │         │                                                  │
│    ▼         ▼                                                  │
│  revert   ┌─────────────┐                                       │
│  context  │ stop_reason │                                       │
│    │      └──────┬──────┘                                       │
│    │             │                                              │
│    │      ┌──────┴──────┐                                       │
│    │      │             │                                       │
│    │      ▼             ▼                                       │
│    │   no_tool_calls  tool_rejected                             │
│    │      │             │                                       │
│    │      └──────┬──────┘                                       │
│    │             │                                              │
│    │             ▼                                              │
│    │      ┌─────────────┐                                       │
│    │      │ TurnOutcome │                                       │
│    │      └─────────────┘                                       │
│    │                                                            │
│    └──────────► 继续循环 (step_no = 0)                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 与其他 Agent 框架对比

| 维度 | kimi-cli | LangChain | AutoGen |
|------|----------|-----------|---------|
| **循环模型** | Step-based | Chain-based | Turn-based |
| **重试机制** | Tenacity | 自定义 | 无 |
| **状态保存** | Checkpoint | Memory | 无 |
| **时间旅行** | D-Mail | 无 | 无 |
| **中断恢复** | asyncio.shield | 无 | 无 |

---

## 章节衔接

**本章回顾**：
- 我们学习了 Agent 循环的 Step-based 设计
- 关键收获：重试机制（Tenacity）、停止条件（no_tool_calls/tool_rejected）、asyncio.shield 保护

**下一章预告**：
- 在 `04-context.md` 中，我们将学习 Context 管理与 Checkpoint 系统
- 为什么需要学习：Context 是 Agent 的"记忆"，Checkpoint 是 D-Mail 的基础
- 关键问题：Context 如何持久化？Checkpoint 如何创建和回退？Compaction 如何工作？
