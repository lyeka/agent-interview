# 二、Tool Calling / Function Calling

> 15 道题目，覆盖 Tool Calling 基础、高级话题、MCP 协议

---

## 2.1 基础概念

### Q16: 解释 Tool Calling 的工作原理，LLM 如何决定调用哪个工具？

**答案要点：**

- **工作流程**：
  ```
  1. 用户输入 + Tool Schema → LLM
  2. LLM 分析意图，决定是否需要工具
  3. 如需要，生成结构化的工具调用请求
  4. 系统执行工具，返回结果
  5. LLM 基于结果生成最终响应
  ```

- **决策机制**：
  - LLM 通过训练学会理解工具描述
  - 根据用户意图匹配最相关的工具
  - 从工具描述中提取参数要求

- **关键要素**：
  ```json
  {
    "name": "get_weather",
    "description": "获取指定城市的天气信息",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {"type": "string", "description": "城市名称"}
      },
      "required": ["city"]
    }
  }
  ```

- **影响因素**：
  - Tool Description 的清晰度
  - 参数定义的准确性
  - 用户意图的明确程度

---

### Q17: Function Calling vs Tool Use 的区别？

**答案要点：**

- **术语来源**：
  - **Function Calling**：OpenAI 的术语
  - **Tool Use**：Anthropic 的术语
  - 本质上是同一概念的不同命名

- **细微差异**：
  | 维度 | OpenAI Function Calling | Anthropic Tool Use |
  |------|------------------------|-------------------|
  | 返回格式 | function_call 字段 | tool_use 内容块 |
  | 并行调用 | parallel_tool_calls | 原生支持 |
  | 强制调用 | tool_choice: required | tool_choice: any |

- **统一理解**：
  - 都是让 LLM 生成结构化的函数调用
  - 都需要定义 Schema
  - 都支持多轮调用

---

### Q18: 如何设计一个好的 Tool Schema？有哪些最佳实践？

**答案要点：**

- **命名规范**：
  - 使用动词开头：`get_`, `create_`, `update_`, `delete_`
  - 清晰表达功能：`search_documents` 而非 `docs`

- **描述要点**：
  ```json
  {
    "name": "search_products",
    "description": "在商品库中搜索商品。支持按名称、类别、价���范围筛选。返回匹配的商品列表，包含名称、价格、库存信息。",
    "parameters": {
      "properties": {
        "query": {
          "type": "string",
          "description": "搜索关键词，支持商品名称或描述"
        },
        "category": {
          "type": "string",
          "enum": ["electronics", "clothing", "food"],
          "description": "商品类别，可选"
        },
        "max_price": {
          "type": "number",
          "description": "最高价格限制，单位：元"
        }
      },
      "required": ["query"]
    }
  }
  ```

- **最佳实践**：
  1. 描述要具体，说明输入输出
  2. 使用 enum 限制可选值
  3. 明确标注 required 字段
  4. 提供参数示例
  5. 说明边界情况处理

---

### Q19: 什么是 Parallel Tool Calling？什么场景下使用？

**答案要点：**

- **定义**：LLM 在单次响应中同时请求调用多个工具

- **示例**：
  ```json
  {
    "tool_calls": [
      {"name": "get_weather", "arguments": {"city": "北京"}},
      {"name": "get_weather", "arguments": {"city": "上海"}},
      {"name": "get_stock_price", "arguments": {"symbol": "AAPL"}}
    ]
  }
  ```

- **适用场景**：
  - 多个独立查询（如多城市天气）
  - 数据聚合（从多个源获取信息）
  - 批量操作（创建多个资源）

- **注意事项**：
  - 确保工具调用相互独立
  - 处理部分失败情况
  - 控制并发数量
  - 结果聚合逻辑

---

### Q20: 如何处理 Tool Calling 的错误和重试？

**答案要点：**

- **错误类型**：
  1. **参数错误**：LLM 生成了无效参数
  2. **执行错误**：工具执行失败（网络、权限等）
  3. **超时错误**：工具执行超时
  4. **业务错误**：工具返回业务异常

- **处理策略**：
  ```python
  async def execute_tool_with_retry(tool_call, max_retries=3):
      for attempt in range(max_retries):
          try:
              result = await execute_tool(tool_call)
              return result
          except ValidationError as e:
              # 参数错误，让 LLM 重新生成
              return {"error": f"参数错误: {e}", "retry_hint": "请检查参数"}
          except TimeoutError:
              if attempt < max_retries - 1:
                  await asyncio.sleep(2 ** attempt)  # 指数退避
                  continue
              return {"error": "工具执行超时"}
          except Exception as e:
              return {"error": str(e)}
  ```

- **反馈给 LLM**：
  - 清晰描述错误原因
  - 提供修正建议
  - 让 LLM 决定是否重试或换方案

---

## 2.2 高级话题

### Q21: 如何实现动态工具注册和发现？

**答案要点：**

- **动态注册**：
  ```python
  class ToolRegistry:
      def __init__(self):
          self.tools = {}

      def register(self, tool: Tool):
          self.tools[tool.name] = tool

      def unregister(self, name: str):
          del self.tools[name]

      def get_schemas(self, context: Context) -> list:
          # 根据上下文过滤可用工具
          return [t.schema for t in self.tools.values()
                  if t.is_available(context)]
  ```

- **发现机制**：
  - 基于用户权限过滤
  - 基于任务类型推荐
  - 基于历史使用频率排序

- **MCP 方式**：
  - Server 动态暴露工具
  - Client 自动发现
  - 支持热更新

---

### Q22: Tool 的权限控制如何设计？如何防止 Agent 滥用工具？

**答案要点：**

- **权限层级**：
  ```
  L1: 只读工具（查询、搜索）
  L2: 写入工具（创建、更新）
  L3: 危险工具（删除、执行代码）
  L4: 管理工具（系统配置）
  ```

- **控制策略**：
  1. **白名单**：只允许调用预定义工具
  2. **参数校验**：限制参数范围
  3. **频率限制**：防止滥用
  4. **审批流程**：高危操作需人工确认

- **实现示例**：
  ```python
  @tool(permission_level=3, requires_approval=True)
  def delete_user(user_id: str):
      """删除用户账号（危险操作）"""
      pass

  class ToolExecutor:
      async def execute(self, tool_call, user_context):
          tool = self.get_tool(tool_call.name)
          if tool.permission_level > user_context.max_permission:
              raise PermissionDenied()
          if tool.requires_approval:
              await self.request_approval(tool_call)
          return await tool.execute(tool_call.arguments)
  ```

---

### Q23: 如何优化 Tool Description 以提高调用准确率？

**答案要点：**

- **优化维度**：

  1. **清晰的功能描述**：
     ```
     ❌ "获取数据"
     ✅ "从用户数据库中获取指定用户的详细信息，包括姓名、邮箱、注册时间"
     ```

  2. **明确的使用场景**：
     ```
     ✅ "当用户询问订单状态、物流信息、退款进度时使用此工具"
     ```

  3. **参数说明详尽**：
     ```json
     {
       "order_id": {
         "type": "string",
         "description": "订单ID，格式为 ORD-YYYYMMDD-XXXXX，如 ORD-20240101-12345"
       }
     }
     ```

  4. **返回值说明**：
     ```
     "返回订单详情对象，包含 status(状态)、items(商品列表)、total(总金额)"
     ```

- **A/B 测试**：
  - 对比不同描述的调用准确率
  - 收集失败案例，针对性优化

---

### Q24: 什么是 Tool Permissioning？如何实现细粒度的工具权限？

**答案要点：**

- **Tool Permissioning 定义**：对工具调用进行细粒度的权限控制

- **权限维度**：
  | 维度 | 示例 |
  |------|------|
  | 工具级别 | 允许/禁止调用某工具 |
  | 参数级别 | 限制参数值范围 |
  | 资源级别 | 只能访问特定资源 |
  | 操作级别 | 只读/读写/删除 |
  | 时间级别 | 工作时间内可用 |

- **实现方案**：
  ```python
  class ToolPermission:
      tool_name: str
      allowed_operations: list[str]
      resource_filter: dict  # {"user_id": "self"}
      parameter_constraints: dict  # {"limit": {"max": 100}}
      time_window: TimeWindow

  def check_permission(tool_call, user_permissions):
      perm = user_permissions.get(tool_call.name)
      if not perm:
          return False
      if not perm.check_parameters(tool_call.arguments):
          return False
      if not perm.check_time_window():
          return False
      return True
  ```

---

### Q25: 如何处理工具调用的超时和并发限制？

**答案要点：**

- **超时处理**：
  ```python
  async def execute_with_timeout(tool_call, timeout=30):
      try:
          result = await asyncio.wait_for(
              execute_tool(tool_call),
              timeout=timeout
          )
          return result
      except asyncio.TimeoutError:
          return ToolResult(
              success=False,
              error="工具执行超时，请稍后重试或尝试其他方式"
          )
  ```

- **并发限制**：
  ```python
  class RateLimiter:
      def __init__(self, max_concurrent=5, max_per_minute=60):
          self.semaphore = asyncio.Semaphore(max_concurrent)
          self.rate_limiter = TokenBucket(max_per_minute)

      async def execute(self, tool_call):
          await self.rate_limiter.acquire()
          async with self.semaphore:
              return await execute_tool(tool_call)
  ```

- **策略选择**：
  - 快速失败 vs 排队等待
  - 优先级队列
  - 降级方案

---

## 2.3 MCP (Model Context Protocol)

### Q26: 什么是 MCP？它解决了什么问题？

**答案要点：**

- **MCP 定义**：Anthropic 提出的开放协议，标准化 LLM 与外部数据源/工具的连接方式

- **解决的问题**：
  1. **碎片化**：每个应用都要单独实现工具集成
  2. **重复开发**：相同工具在不同应用中重复实现
  3. **互操作性**：不同 LLM 应用无法共享工具

- **核心价值**：
  ```
  Before MCP:
  App1 ←→ Tool1, Tool2, Tool3 (各自实现)
  App2 ←→ Tool1, Tool2, Tool4 (重复实现)

  After MCP:
  App1 ←→ MCP Client ←→ MCP Server ←→ Tools
  App2 ←→ MCP Client ←→ MCP Server ←→ Tools
  ```

- **类比**：MCP 之于 LLM 工具，如同 USB 之于外设

---

### Q27: MCP 的核心组件：Server、Client、Transport 如何协作？

**答案要点：**

- **组件职责**：
  - **MCP Server**：暴露工具、资源、提示模板
  - **MCP Client**：连接 Server，调用能力
  - **Transport**：通信层（stdio、HTTP、WebSocket）

- **协作流程**：
  ```
  1. Client 连接 Server（通过 Transport）
  2. Client 发送 initialize 请求
  3. Server 返回能力列表（tools, resources, prompts）
  4. Client 调用 tools/call 执行工具
  5. Server 返回执行结果
  ```

- **协议示例**：
  ```json
  // 请求
  {"jsonrpc": "2.0", "method": "tools/call", "params": {
    "name": "read_file",
    "arguments": {"path": "/tmp/test.txt"}
  }, "id": 1}

  // 响应
  {"jsonrpc": "2.0", "result": {
    "content": [{"type": "text", "text": "文件内容..."}]
  }, "id": 1}
  ```

---

### Q28: MCP 与传统 Function Calling 的区别和优势？

**答案要点：**

- **对比**：
  | 维度 | Function Calling | MCP |
  |------|-----------------|-----|
  | 标准化 | 各厂商不同 | 统一协议 |
  | 工具来源 | 应用内定义 | 外部 Server |
  | 动态性 | 静态 | 动态发现 |
  | 复用性 | 低 | 高 |
  | 生态 | 封闭 | 开放 |

- **MCP 优势**：
  1. **即插即用**：连接 Server 即可使用工具
  2. **生态共享**：社区贡献的 Server 可直接使用
  3. **关注点分离**：工具开发与应用开发解耦
  4. **安全隔离**：Server 独立进程运行

- **适用场景**：
  - Function Calling：简单、内置工具
  - MCP：复杂、需要复用的工具生态

---

### Q29: 如何设计一个 MCP Server？有哪些最佳实践？

**答案要点：**

- **基本结构**：
  ```python
  from mcp.server import Server
  from mcp.types import Tool, TextContent

  server = Server("my-server")

  @server.list_tools()
  async def list_tools():
      return [
          Tool(
              name="my_tool",
              description="工具描述",
              inputSchema={...}
          )
      ]

  @server.call_tool()
  async def call_tool(name: str, arguments: dict):
      if name == "my_tool":
          result = do_something(arguments)
          return [TextContent(type="text", text=result)]
  ```

- **最佳实践**：
  1. **清晰的工具描述**：帮助 LLM 正确调用
  2. **完善的错误处理**：返回有意义的错误信息
  3. **资源管理**：正确处理连接、文件等资源
  4. **日志记录**：便于调试和监控
  5. **版本管理**：支持协议版本协商

---

### Q30: MCP 的安全模型是什么？如何防止恶意 Server？

**答案要点：**

- **安全威胁**：
  - 恶意 Server 返回有害内容
  - Server 窃取敏感信息
  - Server 执行未授权操作

- **安全机制**：
  1. **Server 来源验证**：
     - 只连接可信 Server
     - 签名验证

  2. **能力限制**：
     - 限制 Server 可访问的资源
     - 沙箱执行

  3. **内容审核**：
     - 过滤 Server 返回的内容
     - 检测注入攻击

  4. **审计日志**：
     - 记录所有 Server 交互
     - 异常检测

- **用户侧防护**：
  ```json
  {
    "mcpServers": {
      "trusted-server": {
        "command": "npx",
        "args": ["-y", "@trusted/mcp-server"],
        "permissions": ["read_files", "search_web"]
      }
    }
  }
  ```

---

[PROTOCOL]: 变更时更新此头部，然后检查 CLAUDE.md
