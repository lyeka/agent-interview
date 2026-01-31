# 06 - Multi-Agent 与 LaborMarket

## 本章目标

理解 kimi-cli 的 Multi-Agent 架构，掌握 LaborMarket 设计、Fixed/Dynamic Subagent 的区别和任务委派机制。

## Multi-Agent 概述

kimi-cli 的 Multi-Agent 设计基于"人才市场"（LaborMarket）模型：

1. **主 Agent**：接收用户输入，决定是否委派任务
2. **Subagent**：执行具体任务，返回结果给主 Agent
3. **LaborMarket**：管理所有 subagent，支持动态创建

```
┌─────────────────────────────────────────────────────────────────┐
│                     Multi-Agent 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Main Agent                           │   │
│  │  ┌─────────────┐                                        │   │
│  │  │  KimiSoul   │                                        │   │
│  │  └──────┬──────┘                                        │   │
│  │         │                                               │   │
│  │         │ Task Tool                                     │   │
│  │         ▼                                               │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │              LaborMarket                        │    │   │
│  │  │                                                 │    │   │
│  │  │  ┌───────────────┐    ┌───────────────────┐    │    │   │
│  │  │  │ Fixed Subagent│    │ Dynamic Subagent  │    │    │   │
│  │  │  │   (coder)     │    │  (user-created)   │    │    │   │
│  │  │  └───────────────┘    └───────────────────┘    │    │   │
│  │  │                                                 │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## LaborMarket 实现

### 数据结构

```python
# src/kimi_cli/soul/agent.py:167-186
class LaborMarket:
    def __init__(self):
        self.fixed_subagents: dict[str, Agent] = {}
        self.fixed_subagent_descs: dict[str, str] = {}
        self.dynamic_subagents: dict[str, Agent] = {}

    @property
    def subagents(self) -> Mapping[str, Agent]:
        """Get all subagents in the labor market."""
        return {**self.fixed_subagents, **self.dynamic_subagents}

    def add_fixed_subagent(self, name: str, agent: Agent, description: str):
        """Add a fixed subagent."""
        self.fixed_subagents[name] = agent
        self.fixed_subagent_descs[name] = description

    def add_dynamic_subagent(self, name: str, agent: Agent):
        """Add a dynamic subagent."""
        self.dynamic_subagents[name] = agent
```

**设计决策**：
- **两类 Subagent**：Fixed（配置文件定义）和 Dynamic（运行时创建）
- **描述分离**：Fixed subagent 的描述单独存储，用于 Task 工具的提示

## Fixed vs Dynamic Subagent

### Fixed Subagent

从配置文件加载，有独立的 LaborMarket。

```yaml
# src/kimi_cli/agents/default/agent.yaml
version: 1
agent:
  name: ""
  system_prompt_path: ./system.md
  tools:
    - "kimi_cli.tools.shell:Shell"
    - "kimi_cli.tools.file:ReadFile"
  subagents:
    coder:
      path: ./sub.yaml
      description: "Good at general software engineering tasks."
```

**Runtime 克隆**：

```python
# src/kimi_cli/soul/agent.py:125-140
def copy_for_fixed_subagent(self) -> Runtime:
    """Clone runtime for fixed subagent."""
    return Runtime(
        config=self.config,
        oauth=self.oauth,
        llm=self.llm,
        session=self.session,
        builtin_args=self.builtin_args,
        denwa_renji=DenwaRenji(),      # 独立时间线
        approval=self.approval.share(), # 共享批准状态
        labor_market=LaborMarket(),    # 独立人才市场
        environment=self.environment,
        skills=self.skills,
    )
```

**特点**：
- 独立的 `DenwaRenji`：不能影响主 Agent 的时间线
- 独立的 `LaborMarket`：不能创建子 subagent
- 共享的 `Approval`：YOLO 模式和 auto-approve 列表共享

### Dynamic Subagent

运行时通过 `CreateSubagent` 工具创建。

```python
# src/kimi_cli/tools/multiagent/create.py:33-50
async def __call__(self, params: Params) -> ToolReturnValue:
    if params.name in self._runtime.labor_market.subagents:
        return ToolError(
            message=f"Subagent with name '{params.name}' already exists.",
            brief="Subagent already exists",
        )

    subagent = Agent(
        name=params.name,
        system_prompt=params.system_prompt,
        toolset=self._toolset,  # 共享工具集
        runtime=self._runtime.copy_for_dynamic_subagent(),
    )
    self._runtime.labor_market.add_dynamic_subagent(params.name, subagent)
    return ToolOk(
        output="Available subagents: " + ", ".join(self._runtime.labor_market.subagents.keys()),
        message=f"Subagent '{params.name}' created successfully.",
    )
```

**Runtime 克隆**：

```python
# src/kimi_cli/soul/agent.py:142-153
def copy_for_dynamic_subagent(self) -> Runtime:
    """Clone runtime for dynamic subagent."""
    return Runtime(
        config=self.config,
        oauth=self.oauth,
        llm=self.llm,
        session=self.session,
        builtin_args=self.builtin_args,
        denwa_renji=DenwaRenji(),      # 独立时间线
        approval=self.approval.share(), # 共享批准状态
        labor_market=self.labor_market, # 共享人才市场！
        environment=self.environment,
        skills=self.skills,
    )
```

**特点**：
- 共享的 `LaborMarket`：可以递归创建 subagent
- 共享的 `Toolset`：使用与主 Agent 相同的工具

### 对比表

| 维度 | Fixed Subagent | Dynamic Subagent |
|------|----------------|------------------|
| 定义方式 | agent.yaml 配置 | CreateSubagent 工具 |
| LaborMarket | 独立 | 共享 |
| 能否创建子 subagent | 否 | 是 |
| Toolset | 独立（从配置加载） | 共享（与主 Agent 相同） |
| DenwaRenji | 独立 | 独立 |
| Approval | 共享 | 共享 |

## Task 工具实现

### 任务委派

```python
# src/kimi_cli/tools/multiagent/task.py:60-100
class Task(CallableTool2[Params]):
    async def __call__(self, params: Params) -> ToolReturnValue:
        # 查找 subagent
        agent = self._runtime.labor_market.subagents.get(params.subagent_name)
        if agent is None:
            return ToolError(
                message=f"Subagent '{params.subagent_name}' not found.",
                brief="Subagent not found",
            )

        return await self._run_subagent(agent, params.prompt)
```

### Subagent 执行

```python
# src/kimi_cli/tools/multiagent/task.py:101-161
async def _run_subagent(self, agent: Agent, prompt: str) -> ToolReturnValue:
    super_wire = get_wire_or_none()
    current_tool_call = get_current_tool_call_or_none()
    current_tool_call_id = current_tool_call.id

    # Wire 事件转发
    def _super_wire_send(msg: WireMessage) -> None:
        if isinstance(msg, ApprovalRequest | ApprovalResponse | ToolCallRequest):
            # 请求直接冒泡到根 Wire
            super_wire.soul_side.send(msg)
            return

        # 其他事件包装为 SubagentEvent
        event = SubagentEvent(
            task_tool_call_id=current_tool_call_id,
            event=msg,
        )
        super_wire.soul_side.send(event)

    # UI 循环
    async def _ui_loop_fn(wire: Wire) -> None:
        wire_ui = wire.ui_side(merge=True)
        while True:
            msg = await wire_ui.receive()
            _super_wire_send(msg)

    # 创建独立 Context
    subagent_context_file = await self._get_subagent_context_file()
    context = Context(file_backend=subagent_context_file)
    soul = KimiSoul(agent, context=context)

    try:
        await run_soul(soul, prompt, _ui_loop_fn, asyncio.Event())
    except MaxStepsReached as e:
        return ToolError(
            message=f"Max steps {e.n_steps} reached.",
            brief="Max steps reached",
        )

    # 响应质量控制
    final_response = context.history[-1].extract_text(sep="\n")
    if len(final_response) < 200:
        # 响应太短，追加 CONTINUE_PROMPT
        await run_soul(soul, CONTINUE_PROMPT, _ui_loop_fn, asyncio.Event())
        final_response = context.history[-1].extract_text(sep="\n")

    return ToolOk(output=final_response)
```

**设计亮点**：

1. **独立 Context**：每个 subagent 有独立的 context 文件（`_sub.1.jsonl`）
2. **Wire 事件转发**：subagent 的事件包装为 `SubagentEvent` 转发到根 Wire
3. **Approval 冒泡**：ApprovalRequest 直接发送到根 Wire，由主 UI 处理
4. **响应质量控制**：如果响应少于 200 字符，自动追问

## SubagentEvent 冒泡机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    SubagentEvent 冒泡                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Subagent                                                       │
│  ┌─────────────┐                                                │
│  │  KimiSoul   │                                                │
│  │   _step()   │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         │ TextPart / ToolCall / ToolResult                      │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │ Subagent    │                                                │
│  │   Wire      │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         │ _super_wire_send()                                    │
│         ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                         │   │
│  │  if ApprovalRequest:                                    │   │
│  │      super_wire.send(msg)  # 直接冒泡                   │   │
│  │  else:                                                  │   │
│  │      super_wire.send(SubagentEvent(                     │   │
│  │          task_tool_call_id=...,                         │   │
│  │          event=msg                                      │   │
│  │      ))                                                 │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│         │                                                       │
│         ▼                                                       │
│  Main Agent Wire                                                │
│  ┌─────────────┐                                                │
│  │   Shell UI  │                                                │
│  │ visualize() │                                                │
│  └─────────────┘                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**UI 渲染**：

```python
# src/kimi_cli/ui/shell/visualize.py:746-762
def handle_subagent_event(self, event: SubagentEvent) -> None:
    block = self._tool_call_blocks.get(event.task_tool_call_id)
    if block is None:
        return

    match event.event:
        case ToolCall() as tool_call:
            block.append_sub_tool_call(tool_call)
        case ToolResult() as tool_result:
            block.finish_sub_tool_call(tool_result)
```

Subagent 的 tool call 会显示在父级 Task tool call 下方。

## Context 文件命名

```python
# src/kimi_cli/tools/multiagent/task.py:45-58
async def _get_subagent_context_file(self) -> Path:
    """Get a unique context file path for subagent."""
    base_path = self._session.context_file
    suffix = base_path.suffix  # .jsonl

    # 查找可用的编号
    i = 1
    while True:
        candidate = base_path.with_suffix(f"_sub.{i}{suffix}")
        if not candidate.exists():
            return candidate
        i += 1
```

**文件命名示例**：
- 主 Agent：`context.jsonl`
- Subagent 1：`context_sub.1.jsonl`
- Subagent 2：`context_sub.2.jsonl`

## 设计决策分析

### 为什么 Fixed Subagent 有独立 LaborMarket？

**现象层**：Fixed subagent 不能创建子 subagent。

**本质层**：
1. **防止无限递归**：如果 fixed subagent 可以创建 subagent，可能导致无限嵌套
2. **职责清晰**：fixed subagent 是预定义的专家，不需要再委派任务
3. **资源控制**：限制 subagent 的深度，避免资源耗尽

### 为什么 Dynamic Subagent 共享 LaborMarket？

**现象层**：Dynamic subagent 可以递归创建 subagent。

**本质层**：
1. **灵活性**：用户创建的 subagent 可能需要进一步分解任务
2. **协作能力**：多个 dynamic subagent 可以互相调用
3. **统一管理**：所有 subagent 在同一个 LaborMarket 中，便于查找

### 为什么 Approval 共享？

**现象层**：所有 subagent 共享 Approval 状态。

**本质层**：
1. **用户体验**：用户只需批准一次，所有 subagent 都生效
2. **安全一致**：YOLO 模式对所有 subagent 生效
3. **简化交互**：避免重复询问相同操作

---

## 章节衔接

**本章回顾**：
- 我们学习了 Multi-Agent 的 LaborMarket 设计
- 关键收获：Fixed vs Dynamic Subagent、Runtime 克隆策略、SubagentEvent 冒泡

**下一章预告**：
- 在 `07-wire.md` 中，我们将学习 Wire 协议与 UI 解耦
- 为什么需要学习：Wire 是 kimi-cli 的通信骨架，理解它才能理解 Soul 和 UI 如何协作
- 关键问题：Wire 如何实现双向通信？事件如何合并？不同 UI 如何消费 Wire？
