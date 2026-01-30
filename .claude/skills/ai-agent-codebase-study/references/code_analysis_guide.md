# 代码分析指南

> 深度探索源码的步骤和方法

## Phase 1: 识别关键文件

使用 Glob/Grep 定位核心文件：

```python
# 查找 Agent 核心实现
glob("**/*agent*.py")
glob("**/soul/**/*.py")

# 查找工具定义
glob("**/tools/**/*.py")

# 查找 MCP 相关
grep("mcp", "*.py", "*.md")
```

## Phase 2: 读取完整文件

**不要只读摘要**，使用 Read 工具读取完整文件内容：

```python
# 读取核心文件
read("src/kimi_cli/soul/kimisoul.py")
read("src/kimi_cli/soul/agent.py")
```

## Phase 3: 提取关键代码片段

**精炼原则**：
- 只展示关键逻辑（省略样板代码）
- 保留行号引用
- 添加"为什么这样写"的注释

```python
# src/kimi_cli/soul/kimisoul.py:145-167
async def _execute_tools(self, tool_calls: list[ToolCall]):
    for tool_call in tool_calls:
        # 为什么遍历而非并发？
        # A: 工具间可能有依赖关系，串行保证顺序
        result = await self._toolset.run(tool_call)
```

## Phase 4: 分析设计决策

对每个核心模块，回答：

1. **为什么这样设计？**（架构决策）
2. **有哪些 trade-off？**（权衡考量）
3. **如果重构会怎么改？**（改进方向）

## Phase 5: 识别可面试的技术点

从代码中反推问题：

- 发现有趣实现 → "你能解释一下 {file}:{line} 的设计思路吗？"
- 发现特殊处理 → "这里为什么需要 {condition} 分支？"
- 发现抽象层 → "为什么要抽象这一层，解决了什么问题？"

## 可视化辅助

生成：

- 架构图（组件关系）
- 流程图（执行路径）
- 时序图（交互过程）
- 对比表格（框架对比、技术选型）
