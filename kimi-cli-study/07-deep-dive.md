# 第七层：核心源码深度解析

> 本章逐行分析 kimi-cli 的核心文件，理解每个设计决策的原因。

---

## 7.1 KimiSoul 主循环

**文件**: `src/kimi_cli/soul/kimisoul.py`

### 关键片段 1：用户输入处理

```python
# src/kimi_cli/soul/kimisoul.py:120-130
async def _run_turn(self, user_input: str) -> TurnOutcome:
    # 为什么要 strip()？
    # A: 防止用户输入前后空格影响 LLM 理解
    user_input = user_input.strip()

    # 为什么要检查空输入？
    # A: 避免无意义的 LLM 调用，节省 token 和时间
    if not user_input:
        return TurnOutcome.ignored()
```

### 关键片段 2：ReAct 循环

```python
# src/kimi_cli/soul/kimisoul.py:140-167
step_count = 0
while step_count < self._loop_control.max_steps:
    # 为什么要 deepcopy messages？
    # A: 防止 kosong.step 修改原始消息，保证幂等性
    messages = copy.deepcopy(self._context.messages)

    result = await kosong.step(
        model=self._model,
        messages=messages,
        tools=self._toolset.tools,
    )

    # 为什么要立即添加到上下文？
    # A: 保证上下文的完整性，即使后续出错也能恢复
    self._context.add_message(result.message)

    if result.message.tool_calls:
        # 为什么批量审批而非单个审批？
        # A: 提高用户体验，避免多次确认
        approval = await self._approve_tools(result.message.tool_calls)

        if not approval.approved:
            # 为什么这里要 break 而不是 return？
            # A: break 可以执行后续的清理逻辑，return 直接退出
            break

        await self._execute_tools(result.message.tool_calls)
    else:
        # 为什么没有工具调用就 break？
        # A: LLM 已经完成响应，无需继续循环
        break

    step_count += 1
```

**设计亮点**：
1. **幂等性**：deepcopy 保证多次调用结果一致
2. **原子性**：批量审批要么全部执行，要么全部不执行
3. **可恢复性**：即使工具执行失败，上下文已保存

---

## 7.2 工具依赖注入

**文件**: `src/kimi_cli/soul/toolset.py`

### 关键片段：自动依赖注入

```python
# src/kimi_cli/soul/toolset.py:80-120
def load_tools(self, **deps) -> None:
    for tool_func in discover_tools():
        sig = inspect.signature(tool_func)

        # 为什么要检查所有参数？
        # A: 提前发现缺失的依赖，而非运行时才报错
        for param_name, param in sig.parameters.items():
            if param_name not in deps:
                raise ValueError(f"Missing dependency: {param_name}")

        @wraps(tool_func)
        async def wrapper(**kwargs):
            # 为什么要用 sig.parameters 而非直接用 **deps？
            # A: 只注入函数需要的依赖，避免传递无关参数
            for param_name, param in sig.parameters.items():
                if param_name not in kwargs:
                    kwargs[param_name] = deps[param_name]

            return await tool_func(**kwargs)

        self._tools[tool_func.__name__] = wrapper
```

**设计亮点**：
1. **提前检查**：启动时验证依赖，而非运行时才发现
2. **精确注入**：只注入需要的依赖，避免污染
3. **类型安全**：通过 inspect 保证类型匹配

---

## 7.3 Context 压缩

**文件**: `src/kimi_cli/soul/compaction.py`

### 关键片段：LLM 压缩

```python
# src/kimi_cli/soul/compaction.py:60-90
async def _summarize(self, messages: list[Message]) -> str:
    """用 LLM 压缩历史消息"""

    # 为什么要构建 summary prompt？
    # A: 引导 LLM 生成结构化的摘要，而非简单拼接
    prompt = f"""请总结以下对话历史，保留关键信息：

    {format_messages(messages)}

    摘要格式：
    1. 用户的主要诉求
    2. 讨论过的方案
    3. 最终结论
    """

    # 为什么要用更快的模型？
    # A: 压缩是辅助任务，不需要最强的推理能力
    result = await kosong.step(
        model=self._summary_model,  # 可能是更便宜的模型
        messages=[Message(role="user", content=prompt)],
    )

    return result.message.content[0].text
```

**设计亮点**：
1. **结构化输出**：引导 LLM 生成易于理解的摘要
2. **成本优化**：用更便宜的模型做压缩
3. **信息保留**：明确要求保留关键信息

---

## 7.4 子 Agent 创建

**文件**: `src/kimi_cli/tools/multiagent/task.py`

### 关键片段：动态 Agent 生成

```python
# src/kimi_cli/tools/multiagent/task.py:80-120
async def _create_dynamic_agent(self, task: str, context: Context) -> Agent:
    """根据任务动态创建 Agent"""

    # 为什么要用 LLM 生成 Agent 规格？
    # A: 任务不确定，无法预定义所有可能的 Agent
    spec_prompt = f"""根据以下任务生成 Agent 规格：

    任务：{task}

    请返回：
    1. Agent 名称
    2. System prompt（指导 Agent 如何完成任务）
    3. 需要的工具列表
    """

    result = await kosong.step(
        model=self._model,
        messages=[Message(role="user", content=spec_prompt)],
    )

    # 解析 LLM 输出
    spec = parse_agent_spec(result.message.content)

    # 为什么要创建独立的 Runtime？
    # A: 子 Agent 的操作不应该影响父 Agent 的状态
    sub_runtime = Runtime(
        path_resolver=self._runtime.path_resolver,
        fs=self._runtime.fs.clone(),  # 独立的文件系统视图
    )

    agent = Agent(
        name=spec.name,
        system_prompt=spec.system_prompt,
        tools=self._filter_tools(spec.tools),  # 只给需要的工具
        runtime=sub_runtime,
    )

    return agent
```

**设计亮点**：
1. **动态生成**：根据任务自动生成 Agent 规格
2. **隔离性**：独立的 Runtime 保证子 Agent 不影响父 Agent
3. **最小权限**：只给子 Agent 必需的工具

---

## 本章小结

源码深度解析揭示了设计的细节：

1. **幂等性**：deepcopy 保证重复调用的一致性
2. **提前检查**：启动时验证依赖，运行时更可靠
3. **成本优化**：用更便宜的模型做辅助任务
4. **隔离性**：子 Agent 有独立的 Runtime 和工具集

这些细节体现了 kimi-cli 对可靠性和性能的追求。
