# 第四层：工具系统

> 本章介绍 kimi-cli 如何定义工具、实现依赖注入、集成 MCP 协议。

---

## 4.1 KimiToolset 依赖注入

### 设计意图

工具执行时需要访问各种依赖（如 Runtime、Context、其他工具）。如何优雅地注入这些依赖？

**核心问题**：
1. **循环依赖**：Tool A 需要 Tool B，Tool B 又需要 Tool A
2. **懒加载**：某些工具只在首次使用时才初始化
3. **类型安全**：编译时检查依赖是否满足

### 核心实现

**源码位置**: `src/kimi_cli/soul/toolset.py:80-150`

```python
class KimiToolset:
    def __init__(self):
        self._tools: dict[str, Callable] = {}
        self._singletons: dict[str, Any] = {}

    def load_tools(self, **deps) -> None:
        """
        加载工具，使用 inspect.signature 自动注入依赖

        Args:
            **deps: 可用的依赖（如 runtime, context, mcp_manager）
        """
        for tool_func in discover_tools():
            sig = inspect.signature(tool_func)

            # 检查依赖是否满足
            for param_name, param in sig.parameters.items():
                if param_name not in deps:
                    raise ValueError(f"Missing dependency: {param_name}")

            # 创建工具包装器，自动注入依赖
            @wraps(tool_func)
            async def wrapper(**kwargs):
                # 注入依赖
                for param_name, param in sig.parameters.items():
                    if param_name not in kwargs:
                        kwargs[param_name] = deps[param_name]

                return await tool_func(**kwargs)

            self._tools[tool_func.__name__] = wrapper

    async def run(self, tool_call: ToolCall) -> ToolReturnValue:
        """执行工具"""
        tool_func = self._tools.get(tool_call.name)
        if not tool_func:
            return ToolReturnValue.error(f"Tool not found: {tool_call.name}")

        return await tool_func(**tool_call.input)
```

### 工具定义示例

**源码位置**: `src/kimi_cli/tools/file/list.py`

```python
async def list_files(
    path: str,  # 用户输入
    runtime: Runtime,  # 注入的依赖
    context: Context,  # 注入的依赖
) -> ToolReturnValue:
    """
    列出目录下的文件

    Args:
        path: 目录路径
        runtime: Runtime 实例（自动注入）
        context: Context 实例（自动注入）
    """
    abs_path = runtime.path_resolver.resolve(path)
    files = await runtime.fs.listdir(abs_path)

    return ToolReturnValue(
        output={"files": files},
        message=f"列出 {path} 下的文件",
        display=f"**Found {len(files)} files:**\n" + "\n".join(files),
    )
```

### 设计分析

**Q: 为什么不用 DI 容器？自研 DI 的边界在哪里？**

A: DI 容器（如 dependency-injector）的优势：

1. **自动解析依赖图**：自动处理复杂的依赖关系
2. **生命周期管理**：支持 singleton、transient 等模式
3. **配置化**：通过配置文件定义依赖关系

但 kimi-cli 的场景相对简单：
- 依赖关系是线性的（无复杂的多层嵌套）
- 所有依赖都是 singleton
- 不需要热替换或动态配置

自研 DI 的边界：当出现以下需求时，应该考虑使用成熟的 DI 框架：

1. **循环依赖**：需要复杂的依赖图解析
2. **生命周期管理**：需要 transient、scoped 等模式
3. **AOP 支持**：需要切面、拦截器等特性

**Q: 如何解决循环依赖？**

A: 当前实现不支持循环依赖。如果 Tool A 需要 Tool B，Tool B 又需要 Tool A，会抛出异常。

解决方案：
1. **重构**：将公共依赖提取到第三个模块
2. **延迟注入**：通过函数参数延迟注入
3. **使用 DI 容器**：自动处理循环依赖

**Q: keyword-only 参数作为注入边界的理由是什么？**

A: 约定：
- **位置参数**：用户输入（如 `path: str`）
- **关键字参数**：注入的依赖（如 `runtime: Runtime`）

这种约定的好处：
1. **类型安全**：IDE 可以提示用户需要输入哪些参数
2. **清晰性**：一眼看出哪些是用户输入，哪些是系统注入
3. **扩展性**：新增依赖不影响用户调用

**Q: 如果工具需要延迟加载（仅在首次调用时初始化），如何实现？**

A: 使用懒加载包装器：

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
```

---

## 4.2 MCP 集成

### 设计意图

MCP (Model Context Protocol) 是工具标准协议。为什么需要 MCP？

1. **工具生态**：第三方可以发布 MCP Server，提供标准化的工具
2. **跨平台**：同一套工具可以在不同 Agent 框架中使用
3. **安全隔离**：MCP Server 运行在独立进程，崩溃不影响 Agent

### 核心实现

**源码位置**: `src/kimi_cli/mcp.py:45-120`

```python
class MCPManager:
    def __init__(self):
        self._servers: dict[str, MCPServer] = {}

    async def start_server(self, name: str, command: str, args: list[str]) -> None:
        """
        启动 MCP Server

        Args:
            name: Server 名称
            command: 启动命令（如 "npx"）
            args: 启动参数（如 ["@modelcontextprotocol/server-github"]）
        """
        process = await asyncio.create_subprocess_exec(
            command,
            *args,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
        )

        server = MCPServer(process)
        await server.initialize()  # 握手协议
        self._servers[name] = server

    async def list_tools(self, server_name: str) -> list[Tool]:
        """获取 Server 提供的工具列表"""
        server = self._servers.get(server_name)
        if not server:
            raise ValueError(f"Server not found: {server_name}")

        return await server.list_tools()

    async def call_tool(
        self,
        server_name: str,
        tool_name: str,
        arguments: dict,
    ) -> ToolReturnValue:
        """调用 MCP 工具"""
        server = self._servers[server_name]
        result = await server.call_tool(tool_name, arguments)

        # 转换为内部格式
        return ToolReturnValue(
            output=result.content,
            message=f"调用 {server_name}.{tool_name}",
        )
```

### 使用示例

```python
# 启动 GitHub MCP Server
await mcp_manager.start_server(
    name="github",
    command="npx",
    args=["@modelcontextprotocol/server-github"],
)

# 列出 GitHub 工具
tools = await mcp_manager.list_tools("github")
# ["github.create_issue", "github.list_issues", ...]

# 调用 GitHub 工具
result = await mcp_manager.call_tool(
    server_name="github",
    tool_name="create_issue",
    arguments={"repo": "MoonshotAI/kimi-cli", "title": "Bug"},
)
```

### 设计分析

**Q: MCP 的核心价值是什么？**

A: MCP 解决了"工具孤岛"问题：

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

**Q: MCP 工具与内置工具在安全权限上应该有什么区别？**

A: MCP 工具来自第三方，需要更严格的安全控制：

| 维度 | 内置工具 | MCP 工具 |
|------|----------|----------|
| 信任度 | 高（自己写的） | 低（第三方） |
| 权限 | 完全访问 | 限制访问 |
| 隔离 | 同进程 | 独立进程 |
| 审计 | 无需审计 | 需要审计 |

安全措施：
1. **沙箱**：MCP Server 运行在受限环境
2. **权限控制**：每个 Server 需要声明需要的权限
3. **审批机制**：首次调用 MCP 工具需要用户确认

**Q: 如果 MCP Server 无响应或返回恶意数据，如何保护 Agent？**

A: 防护措施：

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
# 检查工具返回的数据是否合法
if not isinstance(result, dict):
    return ToolReturnValue.error("Invalid response format")
```

3. **沙箱隔离**：MCP Server 运行在独立进程，崩溃不影响 Agent

**Q: 让 kimi-cli 作为 MCP Server 暴露自己的工具，需要实现什么？**

A: 需要实现 MCP 协议的服务端：

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

商业机会：
- **工具商店**：用户可以发布自己的 Agent 工具
- **订阅模式**：高级工具需要订阅
- **分成机制**：工具作者可以获得收益

---

## 本章小结

工具系统的设计：

1. **依赖注入**：用 inspect.signature 自动注入依赖，避免手动传递
2. **MCP 集成**：支持标准化的工具协议，扩展工具生态
3. **安全隔离**：MCP Server 运行在独立进程，保护 Agent 安全

相比直接调用函数，kimi-cli 的工具系统更加灵活和安全。
