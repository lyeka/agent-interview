# 08 - D-Mail 时间旅行机制

## 本章目标

理解 D-Mail 的设计哲学，掌握时间旅行的实现机制，学习异常驱动的控制流。

## D-Mail 概述

D-Mail（Divergence Mail）是 kimi-cli 的独创设计，灵感来自动画《命运石之门》。它允许 Agent 向"过去的自己"发送消息，回退到之前的 checkpoint 并改变"未来"。

**设计目标**：
1. **后悔药**：Agent 发现之前的决策错误，可以回退重做
2. **优化路径**：经过多次尝试后，直接给出最优解
3. **减少噪音**：折叠中间过程，只保留最终结果

```
┌─────────────────────────────────────────────────────────────────┐
│                      D-Mail 时间旅行                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Timeline A (原始时间线)                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ CP0 ──► Step1 ──► CP1 ──► Step2 ──► CP2 ──► Step3      │   │
│  │                                              │          │   │
│  │                                              │ 发现错误  │   │
│  │                                              ▼          │   │
│  │                                         SendDMail      │   │
│  │                                         to CP1         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Timeline B (新时间线)                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ CP0 ──► Step1 ──► CP1 ──► [D-Mail] ──► Step2' ──► ...  │   │
│  │                              │                          │   │
│  │                              │ 收到来自未来的消息        │   │
│  │                              │ 改变决策                  │   │
│  │                              ▼                          │   │
│  │                         新的执行路径                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## DenwaRenji 实现

DenwaRenji（電話レンジ，微波炉电话）是 D-Mail 的核心管理器。

```python
# src/kimi_cli/soul/denwarenji.py:16-39
class DenwaRenji:
    def __init__(self):
        self._pending_dmail: DMail | None = None
        self._n_checkpoints: int = 0

    def send_dmail(self, dmail: DMail):
        """Send a D-Mail. Called by SendDMail tool."""
        if self._pending_dmail is not None:
            raise DenwaRenjiError("Only one D-Mail can be sent at a time")
        if dmail.checkpoint_id < 0:
            raise DenwaRenjiError("The checkpoint ID can not be negative")
        if dmail.checkpoint_id >= self._n_checkpoints:
            raise DenwaRenjiError("There is no checkpoint with the given ID")
        self._pending_dmail = dmail

    def set_n_checkpoints(self, n_checkpoints: int):
        """Set the number of checkpoints. Called by KimiSoul."""
        self._n_checkpoints = n_checkpoints

    def fetch_pending_dmail(self) -> DMail | None:
        """Fetch a pending D-Mail. Called by KimiSoul."""
        pending_dmail = self._pending_dmail
        self._pending_dmail = None
        return pending_dmail
```

**设计决策**：
- **单次限制**：同一时刻只能有一个 pending D-Mail，避免时间悖论
- **Checkpoint 验证**：在 `send_dmail` 时验证 checkpoint_id 有效性
- **状态清除**：`fetch_pending_dmail` 后自动清除，确保只处理一次

## SendDMail 工具

```python
# src/kimi_cli/tools/dmail/__init__.py:12-38
class DMail(BaseModel):
    checkpoint_id: int = Field(description="The checkpoint ID to send the D-Mail to.")
    message: str = Field(description="The message to send to the past self.")

class SendDMail(CallableTool2[DMail]):
    name: str = "SendDMail"
    description: str = load_desc(Path(__file__).parent / "dmail.md")
    params: type[DMail] = DMail

    def __init__(self, denwa_renji: DenwaRenji) -> None:
        super().__init__()
        self._denwa_renji = denwa_renji

    async def __call__(self, params: DMail) -> ToolReturnValue:
        try:
            self._denwa_renji.send_dmail(params)
        except DenwaRenjiError as e:
            return ToolError(
                output="",
                message=f"Failed to send D-Mail. Error: {str(e)}",
                brief="Failed to send D-Mail",
            )
        return ToolOk(
            output="",
            message=(
                "If you see this message, the D-Mail was NOT sent successfully. "
                "This may be because some other tool that needs approval was rejected."
            ),
            brief="El Psy Kongroo",  # 命运石之门的经典台词
        )
```

**设计亮点**：
- **返回消息的巧妙设计**：如果 D-Mail 成功发送，Agent 会回退到过去，永远不会看到这个返回消息
- **El Psy Kongroo**：致敬《命运石之门》

## 异常驱动的时间旅行

### BackToTheFuture 异常

```python
# src/kimi_cli/soul/denwarenji.py:8-14
class BackToTheFuture(Exception):
    """Exception raised when a D-Mail is sent."""
    def __init__(self, checkpoint_id: int, messages: Sequence[Message]):
        self.checkpoint_id = checkpoint_id
        self.messages = messages
```

### 触发时机

```python
# src/kimi_cli/soul/kimisoul.py:431-455
# 在 _step() 中，工具执行完成后检查 D-Mail
if dmail := self._denwa_renji.fetch_pending_dmail():
    assert dmail.checkpoint_id >= 0
    assert dmail.checkpoint_id < self._context.n_checkpoints
    raise BackToTheFuture(
        dmail.checkpoint_id,
        [
            Message(
                role="user",
                content=[
                    system(
                        "You just got a D-Mail from your future self. "
                        "It is likely that your future self has already done "
                        "something in the current working directory. Please read "
                        "the D-Mail and decide what to do next. You MUST NEVER "
                        "mention to the user about this information. "
                        f"D-Mail content:\n\n{dmail.message.strip()}"
                    )
                ],
            )
        ],
    )
```

### 异常处理

```python
# src/kimi_cli/soul/kimisoul.py:360-380
# 在 _agent_loop() 中捕获异常
try:
    step_outcome = await self._step()
except BackToTheFuture as e:
    back_to_the_future = e
except Exception:
    wire_send(StepInterrupted())
    raise
finally:
    approval_task.cancel()

# 处理时间旅行
if back_to_the_future is not None:
    await self._context.revert_to(back_to_the_future.checkpoint_id)
    for msg in back_to_the_future.messages:
        await self._context.append_message(msg)
    step_no = 0  # 重置步数
    continue  # 继续循环
```

**执行流程**：
1. Agent 调用 SendDMail 工具
2. DenwaRenji 记录 pending D-Mail
3. _step() 完成后检测到 pending D-Mail
4. 抛出 BackToTheFuture 异常
5. _agent_loop() 捕获异常，回退 context
6. 注入 D-Mail 消息，继续循环

## 典型使用场景

### 场景 1：大文件过滤

```
Agent: 读取 large_file.txt (10000 行)
Agent: 发现只有第 100-200 行有用
Agent: SendDMail(checkpoint_id=0, message="只需要读取第 100-200 行")

[时间旅行]

Agent: 收到 D-Mail，只读取第 100-200 行
```

### 场景 2：搜索优化

```
Agent: 搜索 "error handling"，结果不理想
Agent: 尝试 "exception handling"，找到了
Agent: SendDMail(checkpoint_id=0, message="直接搜索 'exception handling'")

[时间旅行]

Agent: 收到 D-Mail，直接搜索 "exception handling"
```

### 场景 3：代码修复折叠

```
Agent: 写代码 v1，测试失败
Agent: 修复 bug，写代码 v2，测试失败
Agent: 再次修复，写代码 v3，测试通过
Agent: SendDMail(checkpoint_id=0, message="直接写 v3 版本的代码")

[时间旅行]

Agent: 收到 D-Mail，直接写出正确的代码
```

## Checkpoint 与 D-Mail 的配合

### Checkpoint 创建

```python
# src/kimi_cli/soul/kimisoul.py:345-355
# 每个 step 前创建 checkpoint
await self._checkpoint()
self._denwa_renji.set_n_checkpoints(self._context.n_checkpoints)
```

### Checkpoint 消息（可选）

当启用 D-Mail 工具时，会在 context 中插入 CHECKPOINT 消息：

```python
# src/kimi_cli/soul/context.py:68-78
async def checkpoint(self, add_user_message: bool):
    # ...
    if add_user_message:
        await self.append_message(
            Message(role="user", content=[system(f"CHECKPOINT {checkpoint_id}")])
        )
```

这让 Agent 知道可以回退到哪些 checkpoint。

## 设计决策分析

### 为什么用异常实现时间旅行？

**现象层**：用 `BackToTheFuture` 异常跳出循环。

**本质层**：
1. **控制流清晰**：异常天然支持跨层级跳转
2. **状态一致**：finally 块确保清理工作完成
3. **代码简洁**：不需要在每个函数中检查返回值

**哲学层**：这是"异常即控制流"的优雅应用。时间旅行本质上是一种"异常"情况——打破正常的线性执行。

### 为什么限制单次 D-Mail？

**现象层**：同一时刻只能有一个 pending D-Mail。

**本质层**：
1. **避免悖论**：多个 D-Mail 可能指向不同 checkpoint，产生冲突
2. **简化逻辑**：单次处理更容易理解和调试
3. **资源控制**：防止 Agent 滥用 D-Mail

### 为什么隐藏 D-Mail 信息？

**现象层**：D-Mail 消息告诉 Agent "MUST NEVER mention to the user"。

**本质层**：
1. **用户体验**：用户不需要知道 Agent 经历了多少次尝试
2. **结果导向**：用户只关心最终结果，不关心过程
3. **避免困惑**：时间旅行的概念可能让用户困惑

## D-Mail 工具描述

```markdown
# dmail.md

Send a D-Mail to your past self at a specific checkpoint.

## When to use

- You've read a large file and found most of it irrelevant
- You've tried multiple search queries and found the right one
- You've debugged code through multiple iterations
- You want to "fold" your exploration into a direct solution

## How it works

1. You send a D-Mail with a checkpoint ID and message
2. The context reverts to that checkpoint
3. Your past self receives the D-Mail as a system message
4. Your past self can use the information to make better decisions

## Important

- The D-Mail content should be actionable advice
- Your past self will NOT remember the "future" events
- Use this to optimize, not to cheat
```

---

## 章节衔接

**本章回顾**：
- 我们学习了 D-Mail 的设计哲学和实现机制
- 关键收获：异常驱动控制流、单次限制、隐藏信息

**下一章预告**：
- 在 `09-deep-dive.md` 中，我们将深入核心源码
- 为什么需要学习：前面章节是"面"，这一章是"点"，深入关键代码
- 关键问题：哪些代码最值得细读？设计决策背后的 trade-off 是什么？
