# 05 - Tool 系统与 MCP 集成

## 本章目标

理解 kimi-cli 的工具系统设计，掌握依赖注入机制、MCP 工具加载和 Approval 集成。

## 工具系统概述

kimi-cli 的工具系统有三个核心特性：

1. **依赖注入**：通过反射自动注入工具依赖
2. **MCP 集成**：支持 Model Context Protocol 动态扩展
3. **Approval 机制**：工具执行前需用户批准

## KimiToolset 核心实现

### 工具加载机制

```python
# src/kimi_cli/soul/toolset.py:152-177
def load_tools(self, tool_paths: list[str], dependencies: dict[type[Any], Any]) -> None:
    """Load tools from paths like `kimi_cli.tools.shell:Shell`."""

    good_tools: list[str] = []
    bad_tools: list[str] = []

    for tool_path in tool_paths:
        try:
            tool = self._load_tool(tool_path, dependencies)
        except SkipThisTool:
            logger.info("Skipping tool: {tool_path}", tool_path=tool_path)
            continue
        if tool:
            self.add(tool)
            good_tools.append(tool_path)
        else:
            bad_tools.append(tool_path)

    logger.info("Loaded tools: {good_tools}", good_tools=good_tools)
    if bad_tools:
        raise InvalidToolError(f"Invalid tools: {bad_tools}")
```

**工具路径格式**：`module_path:ClassName`

例如：
- `kimi_cli.tools.shell:Shell`
- `kimi_cli.tools.file:ReadFile`
- `kimi_cli.tools.multiagent:Task`

### 依赖注入实现

```python
# src/kimi_cli/soul/toolset.py:178-200
@staticmethod
def _load_tool(tool_path: str, dependencies: dict[type[Any], Any]) -> ToolType | None:
    logger.debug("Loading tool: {tool_path}", tool_path=tool_path)

    # 解析模块路径和类名
    module_name, class_name = tool_path.rsplit(":", 1)

    try:
        module = importlib.import_module(module_name)
    except ImportError:
        return None

    tool_cls = getattr(module, class_name, None)
    if tool_cls is None:
        return None

    # 依赖注入
    args: list[Any] = []
    if "__init__" in tool_cls.__dict__:
        for param in inspect.signature(tool_cls).parameters.values():
            if param.kind == inspect.Parameter.KEYWORD_ONLY:
                break  # 遇到 keyword-only 参数停止注入
            if param.annotation not in dependencies:
                raise ValueError(f"Tool dependency not found: {param.annotation}")
            args.append(dependencies[param.annotation])

    return tool_cls(*args)
```

**设计亮点**：

1. **反射式注入**：通过 `inspect.signature` 检查构造函数参数类型
2. **类型驱动**：依赖字典的 key 是类型（`type[Any]`），不是字符串
3. **位置参数优先**：只注入位置参数，遇到 `KEYWORD_ONLY` 参数停止

### 依赖字典构建

```python
# src/kimi_cli/soul/agent.py:224-235
toolset = KimiToolset()
tool_deps = {
    KimiToolset: toolset,
    Runtime: runtime,
    Config: runtime.config,
    BuiltinSystemPromptArgs: runtime.builtin_args,
    Session: runtime.session,
    DenwaRenji: runtime.denwa_renji,
    Approval: runtime.approval,
    LaborMarket: runtime.labor_market,
    Environment: runtime.environment,
}
```

**使用示例**：

```python
# Shell 工具的构造函数
class Shell(CallableTool2[Params]):
    def __init__(self, approval: Approval, environment: Environment):
        # approval 和 environment 会被自动注入
        self._approval = approval
        self._environment = environment
```

## MCP 工具加载

### 异步加载机制

```python
# src/kimi_cli/soul/toolset.py:203-226
async def load_mcp_tools(
    self, mcp_configs: list[MCPConfig], runtime: Runtime, in_background: bool = True
) -> None:
    """Load MCP tools from specified MCP configs."""
    import fastmcp

    async def _check_oauth_tokens(server_url: str) -> bool:
        """Check if OAuth tokens exist for the server."""
        try:
            from fastmcp.client.auth.oauth import FileTokenStorage
            storage = FileTokenStorage(server_url=server_url)
            tokens = await storage.get_tokens()
            return tokens is not None
        except Exception:
            return False

    # ... OAuth 预检查 ...

    if in_background:
        self._mcp_loading_task = asyncio.create_task(_connect())
    else:
        await _connect()
```

**设计决策**：
- **后台加载**：默认 `in_background=True`，避免阻塞主流程启动
- **OAuth 预检查**：连接前检查 token，避免无效连接尝试

### 并发连接

```python
# src/kimi_cli/soul/toolset.py:239-284
async def _connect_server(
    server_name: str, server_info: MCPServerInfo
) -> tuple[str, Exception | None]:
    if server_info.status != "pending":
        return server_name, None

    server_info.status = "connecting"
    try:
        async with server_info.client as client:
            for tool in await client.list_tools():
                server_info.tools.append(
                    MCPTool(server_name, tool, client, runtime=runtime)
                )

        for tool in server_info.tools:
            self.add(tool)

        server_info.status = "connected"
        return server_name, None
    except Exception as e:
        server_info.status = "failed"
        return server_name, e

# 并发连接所有服务器
tasks = [
    asyncio.create_task(_connect_server(server_name, server_info))
    for server_name, server_info in self._mcp_servers.items()
    if server_info.status == "pending"
]
results = await asyncio.gather(*tasks) if tasks else []
```

**状态机**：`pending → connecting → connected/failed/unauthorized`

### MCPTool 实现

```python
# src/kimi_cli/soul/toolset.py:355-406
class MCPTool[T: ClientTransport](CallableTool):
    def __init__(
        self,
        server_name: str,
        mcp_tool: mcp.Tool,
        client: fastmcp.Client[T],
        *,
        runtime: Runtime,
    ):
        super().__init__(
            name=mcp_tool.name,
            description=(
                f"This is an MCP tool from server `{server_name}`.\n\n"
                f"{mcp_tool.description or 'No description.'}"
            ),
            parameters=mcp_tool.inputSchema,
        )
        self._mcp_tool = mcp_tool
        self._client = client
        self._runtime = runtime
        self._action_name = f"mcp:{mcp_tool.name}"

    async def __call__(self, *args: Any, **kwargs: Any) -> ToolReturnValue:
        # Approval 检查
        if not await self._runtime.approval.request(
            self.name, self._action_name, f"Call MCP tool `{self._mcp_tool.name}`."
        ):
            return ToolRejectedError()

        try:
            async with self._client as client:
                result = await client.call_tool(
                    self._mcp_tool.name,
                    kwargs,
                    timeout=self._timeout,
                    raise_on_error=False,
                )
                return convert_mcp_tool_result(result)
        except Exception as e:
            if "timeout" in str(e).lower():
                return ToolError(message="Timeout", brief="Timeout")
            raise
```

**设计亮点**：
- **泛型类型约束**：`[T: ClientTransport]` 支持不同传输层
- **描述增强**：在原始描述前添加 MCP 服务器信息
- **Approval 集成**：action_name 格式为 `mcp:{tool_name}`

## 内置工具实现

### Shell 工具

```python
# src/kimi_cli/tools/shell/__init__.py:32-66
class Shell(CallableTool2[Params]):
    name: str = "Shell"
    params: type[Params] = Params

    def __init__(self, approval: Approval, environment: Environment):
        is_powershell = environment.shell_name == "Windows PowerShell"
        super().__init__(
            description=load_desc(
                Path(__file__).parent / ("powershell.md" if is_powershell else "bash.md"),
                {"SHELL": f"{environment.shell_name} (`{environment.shell_path}`)"},
            )
        )
        self._approval = approval
        self._is_powershell = is_powershell

    async def __call__(self, params: Params) -> ToolReturnValue:
        if not params.command:
            return ToolError(message="Command cannot be empty.", brief="Empty command")

        # Approval 请求
        if not await self._approval.request(
            self.name,
            "run command",
            f"Run command `{params.command}`",
            display=[ShellDisplayBlock(language="bash", command=params.command)],
        ):
            return ToolRejectedError()

        # 执行命令
        return await self._run_command(params.command, params.timeout)
```

**设计亮点**：
- **环境适配**：根据 `Environment` 自动选择 bash/powershell
- **Display Block**：在 Approval 请求中附带代码块，UI 可高亮显示

### ReadFile 工具

```python
# src/kimi_cli/tools/file/read.py:62-127
class ReadFile(CallableTool2[Params]):
    async def __call__(self, params: Params) -> ToolReturnValue:
        p = self._work_dir / params.path

        # 路径验证
        if error := await self._validate_path(p):
            return error

        # 文件类型检测
        header = await p.read_bytes(MEDIA_SNIFF_BYTES)
        file_type = detect_file_type(str(p), header=header)
        if file_type.kind in ("image", "video"):
            return ToolError(
                message=f"`{params.path}` is a {file_type.kind} file.",
                brief="Unsupported file type",
            )

        # 分页读取
        lines: list[str] = []
        n_bytes = 0
        async for line in p.read_lines(errors="replace"):
            if len(lines) >= MAX_LINES or n_bytes >= MAX_BYTES:
                break
            truncated = truncate_line(line, MAX_LINE_LENGTH)
            lines.append(truncated)
            n_bytes += len(truncated.encode("utf-8"))

        # 格式化输出（cat -n 风格）
        lines_with_no = [f"{i:6d}\t{line}" for i, line in enumerate(lines, params.line_offset)]
        return ToolOk(output="\n".join(lines_with_no))
```

**设计亮点**：
- **安全边界**：工作目录外的文件必须用绝对路径
- **文件类型嗅探**：读取前 N 字节检测类型
- **三重限制**：行数、字节数、行长度

### Grep 工具（ripgrep 集成）

```python
# src/kimi_cli/tools/file/grep_local.py:125-225
async def _get_rg_path() -> Path:
    """Get ripgrep binary path, downloading if necessary."""
    bin_name = "rg.exe" if platform.system() == "Windows" else "rg"

    # 三级查找
    if existing := _find_existing_rg(bin_name):
        return existing

    # 自动下载
    async with _RG_DOWNLOAD_LOCK:
        if existing := _find_existing_rg(bin_name):
            return existing
        return await _download_and_install_rg(bin_name)

def _find_existing_rg(bin_name: str) -> Path | None:
    # 1. share_dir
    share_bin = get_share_dir() / "bin" / bin_name
    if share_bin.is_file():
        return share_bin

    # 2. 本地 deps
    local_dep = Path(kimi_cli.__file__).parent / "deps" / "bin" / bin_name
    if local_dep.is_file():
        return local_dep

    # 3. 系统 PATH
    system_rg = shutil.which("rg")
    if system_rg:
        return Path(system_rg)

    return None
```

**设计亮点**：
- **零依赖安装**：自动下载 ripgrep 二进制
- **三级查找**：share_dir → 本地 deps → 系统 PATH
- **下载锁**：防止并发下载

## Approval 系统

### 请求流程

```python
# src/kimi_cli/soul/approval.py:50-100
async def request(
    self,
    sender: str,
    action: str,
    description: str,
    display: list[DisplayBlock] | None = None,
) -> bool:
    """Request approval for the given action."""
    tool_call = get_current_tool_call_or_none()
    if tool_call is None:
        raise RuntimeError("Approval must be requested from a tool call.")

    # YOLO 模式：全部自动批准
    if self._state.yolo:
        return True

    # Session 级自动批准
    if action in self._state.auto_approve_actions:
        return True

    # 创建请求
    request = Request(
        id=str(uuid.uuid4()),
        tool_call_id=tool_call.id,
        sender=sender,
        action=action,
        description=description,
        display=display or [],
    )

    # 等待用户响应
    approved_future = asyncio.Future[bool]()
    self._request_queue.put_nowait(request)
    self._requests[request.id] = (request, approved_future)
    return await approved_future
```

### 响应处理

```python
# src/kimi_cli/soul/approval.py:118-147
def resolve_request(self, request_id: str, response: Response) -> None:
    """Resolve an approval request."""
    request_tuple = self._requests.pop(request_id, None)
    if request_tuple is None:
        raise KeyError(f"No pending request with ID {request_id}")
    request, future = request_tuple

    match response:
        case "approve":
            future.set_result(True)
        case "approve_for_session":
            self._state.auto_approve_actions.add(request.action)
            future.set_result(True)
        case "reject":
            future.set_result(False)
```

**三级批准策略**：
1. **YOLO 模式**：全部自动批准（`--yolo` 参数）
2. **Session 级**：`approve_for_session` 后同类操作不再询问
3. **单次批准**：`approve` 仅批准当前操作

### ContextVar 传递

```python
# src/kimi_cli/soul/toolset.py:51-59
current_tool_call = ContextVar[ToolCall | None]("current_tool_call", default=None)

def get_current_tool_call_or_none() -> ToolCall | None:
    return current_tool_call.get()

# 工具执行时设置
async def _execute_tool(tool_call: ToolCall):
    token = current_tool_call.set(tool_call)
    try:
        result = await tool.__call__(...)
    finally:
        current_tool_call.reset(token)
```

**设计亮点**：
- 使用 ContextVar 在异步调用链中传递当前 tool_call
- 工具内部可以通过 `get_current_tool_call_or_none()` 获取上下文

## 工具执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Tool 执行流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. LLM 返回 ToolCall                                           │
│     ┌─────────────┐                                             │
│     │ kosong.step │                                             │
│     └──────┬──────┘                                             │
│            │                                                    │
│  2. Toolset 分发                                                │
│     ┌──────▼──────┐                                             │
│     │ KimiToolset │                                             │
│     │   handle()  │                                             │
│     └──────┬──────┘                                             │
│            │                                                    │
│  3. 设置 ContextVar                                             │
│     ┌──────▼──────┐                                             │
│     │current_tool │                                             │
│     │  _call.set  │                                             │
│     └──────┬──────┘                                             │
│            │                                                    │
│  4. Approval 检查                                               │
│     ┌──────▼──────┐     ┌─────────────┐                         │
│     │   Tool      │────►│  Approval   │                         │
│     │  __call__   │     │  request()  │                         │
│     └──────┬──────┘     └──────┬──────┘                         │
│            │                   │                                │
│            │◄──────────────────┘                                │
│            │                                                    │
│  5. 执行 + 返回结果                                              │
│     ┌──────▼──────┐                                             │
│     │ ToolResult  │                                             │
│     └─────────────┘                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 章节衔接

**本章回顾**：
- 我们学习了工具系统的依赖注入、MCP 集成、Approval 机制
- 关键收获：反射式注入、后台 MCP 加载、三级批准策略

**下��章预告**：
- 在 `06-multi-agent.md` 中，我们将学习 Multi-Agent 与 LaborMarket
- 为什么需要学习：Multi-Agent 是复杂任务的解决方案，理解它才能理解任务委派
- 关键问题：Fixed vs Dynamic Subagent 有什么区别？Runtime 如何克隆？
