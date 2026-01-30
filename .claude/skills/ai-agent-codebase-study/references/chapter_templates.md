# 章节模板（动态选择）

> 根据项目实际选择适用的章节模板

## 通用章节结构

### 概念解释

**是什么**：用 1-2 段话解释概念

### 设计意图

**为什么需要**：解释这个模块解决了什么问题

### 核心数据结构/代码

**源码位置**: `{文件}:{行号}`

```python
# 精炼的代码片段（关键逻辑）
def example():
    # 为什么这样写？
    pass
```

### 设计分析

**Q: 为什么这样设计？**
A: ...

**Q: 有哪些 trade-off？**
A: ...

### 对比其他方案

| 维度 | 本方案 | 方案 A | 方案 B |
|------|--------|--------|--------|
| 优势 | ... | ... | ... |
| 劣势 | ... | ... | ... |

### 实战案例

**场景**: ...
**执行流程**: ...

## 专用章节模板

### foundation.md - 基础抽象层

用于：自研框架、消息结构、路径抽象等

### core_agent.md - Agent 核心

用于：ReAct 循环、状态管理、LLM 调用等

### context_management.md - Context 管理

用于：Context 压缩、Window 管理、动态检索等

### prompt_system.md - Prompt 系统

用于：System prompt、Few-shot、CoT 等

### memory_system.md - Memory 系统

用于：Episodic/Semantic/Working/Long-term Memory

### tools_mcp.md - Tool & MCP

用于：工具定义、MCP 集成、依赖注入等

### multi_agent.md - Multi-Agent

用于：协作模式、通信协议、调度机制等

### protocols.md - 协议层

用于：ACP、MCP、Wire 等协议实现
