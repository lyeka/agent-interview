# 第七章：MCP 集成

## 7.1 MCP 协议概述

MCP（Model Context Protocol）是 Anthropic 提出的工具调用标准协议，旨在统一 LLM 与外部工具的交互方式。Codex 同时实现了 MCP 客户端和服务器。

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP Architecture                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐         ┌─────────────┐                    │
│  │   Codex     │ ──────► │ MCP Server  │  Codex as Client   │
│  │   (Client)  │ ◄────── │ (External)  │                    │
│  └─────────────┘         └─────────────┘                    │
│                                                              │
│  ┌─────────────┐         ┌─────────────┐                    │
│  │   External  │ ──────► │   Codex     │  Codex as Server   │
│  │   (Client)  │ ◄────── │  (Server)   │                    │
│  └─────────────┘         └─────────────┘                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7.2 MCP 客户端实现

### McpConnectionManager 结构

```rust
// mcp_connection_manager.rs:308-313
pub(crate) struct McpConnectionManager {
    clients: HashMap<String, AsyncManagedClient>,
    elicitation_requests: ElicitationRequestManager,
}
```

### 连接管理

```rust
// 异步初始化模式：使用 Shared<BoxFuture> 实现懒加载
pub struct AsyncManagedClient {
    init_future: Shared<BoxFuture<'static, Result<ManagedClient, Error>>>,
    client: OnceCell<ManagedClient>,
}

impl AsyncManagedClient {
    pub async fn get(&self) -> Result<&ManagedClient, Error> {
        if let Some(client) = self.client.get() {
            return Ok(client);
        }

        let client = self.init_future.clone().await?;
        Ok(self.client.get_or_init(|| client))
    }
}
```

### 工具名称限定

MCP 工具名称需要加前缀避免冲突：

```rust
// mcp_connection_manager.rs:118-121
// 格式：mcp__<server>__<tool>
fn qualify_tool_name(server_name: &str, tool_name: &str) -> String {
    format!("mcp__{}__{}",
        sanitize_responses_api_tool_name(server_name),
        sanitize_responses_api_tool_name(tool_name)
    )
}

// 工具名称清洗：符合 OpenAI ^[a-zA-Z0-9_-]+$ 规范
fn sanitize_responses_api_tool_name(name: &str) -> String {
    let mut sanitized = String::with_capacity(name.len());
    for c in name.chars() {
        if c.is_ascii_alphanumeric() || c == '_' || c == '-' {
            sanitized.push(c);
        } else {
            sanitized.push('_');  // 替换非法字符
        }
    }
    sanitized
}
```

### 长名称处理

```rust
// mcp_connection_manager.rs:110-150
const MAX_TOOL_NAME_LENGTH: usize = 64;

fn qualify_tools<I>(tools: I) -> HashMap<String, ToolInfo> {
    let mut result = HashMap::new();

    for (server_name, tool) in tools {
        let qualified_name = qualify_tool_name(&server_name, &tool.name);

        // 长度超限时使用 SHA1 哈希截断
        let final_name = if qualified_name.len() > MAX_TOOL_NAME_LENGTH {
            let sha1_str = sha1_hex(&qualified_name);
            let prefix_len = MAX_TOOL_NAME_LENGTH - sha1_str.len();
            format!("{}{}", &qualified_name[..prefix_len], sha1_str)
        } else {
            qualified_name
        };

        result.insert(final_name, ToolInfo { server_name, tool });
    }

    result
}
```

---

## 7.3 MCP 工具转换

### MCP → OpenAI 格式

```rust
// spec.rs:1088-1121
pub(crate) fn mcp_tool_to_openai_tool(
    fully_qualified_name: String,
    tool: mcp_types::Tool,
) -> Result<ResponsesApiTool, serde_json::Error> {
    let mcp_types::Tool {
        description,
        mut input_schema,
        ..
    } = tool;

    // 关键修复 1: OpenAI 强制要求 properties 字段
    if input_schema.properties.is_none() {
        input_schema.properties = Some(serde_json::Value::Object(serde_json::Map::new()));
    }

    // 关键修复 2: 清洗 JSON Schema
    let mut serialized_input_schema = serde_json::to_value(input_schema)?;
    sanitize_json_schema(&mut serialized_input_schema);
    let input_schema = serde_json::from_value::<JsonSchema>(serialized_input_schema)?;

    Ok(ResponsesApiTool {
        name: fully_qualified_name,
        description: description.unwrap_or_default(),
        strict: false,
        parameters: input_schema,
    })
}
```

### Schema 清洗

```rust
// spec.rs:1150-1179
fn sanitize_json_schema(value: &mut JsonValue) {
    match value {
        JsonValue::Bool(_) => {
            // JSON Schema 布尔形式 → 强制转为 string 类型
            *value = json!({ "type": "string" });
        }

        JsonValue::Object(map) => {
            // 1. 递归清洗嵌套 schema
            if let Some(props) = map.get_mut("properties") {
                sanitize_json_schema(props);
            }
            if let Some(items) = map.get_mut("items") {
                sanitize_json_schema(items);
            }

            // 2. 处理 oneOf/anyOf/allOf 组合器
            for combiner in ["oneOf", "anyOf", "allOf", "prefixItems"] {
                if let Some(v) = map.get_mut(combiner) {
                    sanitize_json_schema(v);
                }
            }

            // 3. 类型推断：缺失 type 时根据关键字推断
            if !map.contains_key("type") {
                let inferred_type = if map.contains_key("properties") {
                    "object"
                } else if map.contains_key("items") {
                    "array"
                } else {
                    "string"  // 默认兜底
                };
                map.insert("type".to_string(), JsonValue::String(inferred_type.to_string()));
            }
        }

        _ => {}
    }
}
```

**设计哲学**：这是协议适配层的防御性编程——通过主动清洗和类型推断，容忍 MCP 服务器的 schema 不规范性。

---

## 7.4 资源访问

### 资源列表聚合

```rust
// mcp_connection_manager.rs:478-539
pub async fn list_all_resources(&self) -> HashMap<String, Vec<Resource>> {
    let mut join_set = JoinSet::new();

    // 并行查询所有 MCP 服务器
    for (server_name, client) in &self.clients {
        join_set.spawn(async move {
            let mut collected: Vec<Resource> = Vec::new();
            let mut cursor: Option<String> = None;

            // 分页获取资源
            loop {
                let params = cursor.as_ref().map(|next| ListResourcesRequestParams {
                    cursor: Some(next.clone()),
                });
                let response = client.list_resources(params, timeout).await?;
                collected.extend(response.resources);

                match response.next_cursor {
                    Some(next) => {
                        // 防御性检查：避免无限循环
                        if cursor.as_ref() == Some(&next) {
                            return Err(anyhow!("duplicate cursor"));
                        }
                        cursor = Some(next);
                    }
                    None => return Ok(collected),
                }
            }
        });
    }

    // 聚合结果
    let mut aggregated = HashMap::new();
    while let Some(join_res) = join_set.join_next().await {
        match join_res {
            Ok((server_name, Ok(resources))) => {
                aggregated.insert(server_name, resources);
            }
            Ok((server_name, Err(err))) => {
                warn!("Failed to list resources for {server_name}: {err:#}");
            }
            Err(err) => {
                warn!("Task panic: {err:#}");
            }
        }
    }

    aggregated
}
```

**设计亮点**：
1. **并发聚合**：使用 `JoinSet` 并行查询所有服务器
2. **分页处理**：自动处理 cursor 分页
3. **错误隔离**：单个服务器失败不影响其他服务器

---

## 7.5 OAuth 认证

### OAuth Transport 创建

```rust
// rmcp-client/src/rmcp_client.rs:489-535
async fn create_oauth_transport_and_runtime(
    server_name: &str,
    url: &str,
    initial_tokens: StoredOAuthTokens,
    credentials_store: OAuthCredentialsStoreMode,
    default_headers: HeaderMap,
) -> Result<(StreamableHttpClientTransport<AuthClient<reqwest::Client>>, OAuthPersistor)> {
    let http_client = reqwest::Client::builder()
        .default_headers(default_headers)
        .build()?;

    let mut oauth_state = OAuthState::new(url.to_string(), Some(http_client.clone())).await?;

    // 设置初始凭证
    oauth_state.set_credentials(
        &initial_tokens.client_id,
        initial_tokens.token_response.0.clone(),
    ).await?;

    let manager = match oauth_state {
        OAuthState::Authorized(manager) => manager,
        OAuthState::Unauthorized(manager) => manager,
        _ => return Err(anyhow!("unexpected OAuth state")),
    };

    let auth_client = AuthClient::new(http_client, manager);
    let auth_manager = auth_client.auth_manager.clone();

    let transport = StreamableHttpClientTransport::with_client(
        auth_client,
        StreamableHttpClientTransportConfig::with_uri(url.to_string()),
    );

    let runtime = OAuthPersistor::new(
        server_name.to_string(),
        url.to_string(),
        auth_manager,
        credentials_store,
        Some(initial_tokens),
    );

    Ok((transport, runtime))
}
```

### Token 自动刷新

```rust
// rmcp-client/src/rmcp_client.rs:480-486
async fn refresh_oauth_if_needed(&self) {
    if let Some(runtime) = self.oauth_persistor().await
        && let Err(error) = runtime.refresh_if_needed().await
    {
        warn!("failed to refresh OAuth tokens: {error}");
    }
}
```

---

## 7.6 MCP 服务器实现

### 服务器架构

```rust
// mcp-server/src/lib.rs:46-149
pub async fn run_main(
    codex_linux_sandbox_exe: Option<PathBuf>,
    cli_config_overrides: CliConfigOverrides,
) -> IoResult<()> {
    // 1. 设置通道
    let (incoming_tx, mut incoming_rx) = mpsc::channel::<JSONRPCMessage>(128);
    let (outgoing_tx, mut outgoing_rx) = mpsc::unbounded_channel::<OutgoingMessage>();

    // 2. 任务 1: 从 stdin 读取 JSON-RPC 消息
    let stdin_reader_handle = tokio::spawn(async move {
        let stdin = io::stdin();
        let reader = BufReader::new(stdin);
        let mut lines = reader.lines();

        while let Some(line) = lines.next_line().await.unwrap_or_default() {
            match serde_json::from_str::<JSONRPCMessage>(&line) {
                Ok(msg) => {
                    if incoming_tx.send(msg).await.is_err() {
                        break;
                    }
                }
                Err(e) => error!("Failed to deserialize: {e}"),
            }
        }
    });

    // 3. 任务 2: 处理消息
    let processor_handle = tokio::spawn(async move {
        let mut processor = MessageProcessor::new(outgoing_tx, codex_linux_sandbox_exe, config);
        while let Some(msg) = incoming_rx.recv().await {
            match msg {
                JSONRPCMessage::Request(r) => processor.process_request(r).await,
                JSONRPCMessage::Response(r) => processor.process_response(r).await,
                JSONRPCMessage::Notification(n) => processor.process_notification(n).await,
                JSONRPCMessage::Error(e) => processor.process_error(e),
            }
        }
    });

    // 4. 任务 3: 写入 stdout
    let stdout_writer_handle = tokio::spawn(async move {
        let mut stdout = io::stdout();
        while let Some(msg) = outgoing_rx.recv().await {
            let json = serde_json::to_string(&msg)?;
            stdout.write_all(json.as_bytes()).await?;
            stdout.write_all(b"\n").await?;
        }
    });

    // 5. 等待所有任务完成
    let _ = tokio::join!(stdin_reader_handle, processor_handle, stdout_writer_handle);
    Ok(())
}
```

### 工具定义

```rust
// mcp-server/src/codex_tool_config.rs:109-137
pub(crate) fn create_tool_for_codex_tool_call_param() -> Tool {
    Tool {
        name: "codex".to_string(),
        title: Some("Codex".to_string()),
        input_schema: tool_input_schema,
        output_schema: Some(codex_tool_output_schema()),
        description: Some(
            "Run a Codex session. Accepts configuration parameters matching the Codex Config struct."
                .to_string(),
        ),
        annotations: None,
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema, Default)]
#[serde(rename_all = "kebab-case")]
pub struct CodexToolCallParam {
    pub prompt: String,
    pub model: Option<String>,
    pub profile: Option<String>,
    pub cwd: Option<String>,
    pub approval_policy: Option<CodexToolCallApprovalPolicy>,
    pub sandbox: Option<CodexToolCallSandboxMode>,
    pub config: Option<HashMap<String, serde_json::Value>>,
    pub base_instructions: Option<String>,
    pub developer_instructions: Option<String>,
    pub compact_prompt: Option<String>,
}
```

---

## 7.7 消息处理器

```rust
// mcp-server/src/message_processor.rs:39-73
pub(crate) struct MessageProcessor {
    outgoing: Arc<OutgoingMessageSender>,
    initialized: bool,
    codex_linux_sandbox_exe: Option<PathBuf>,
    thread_manager: Arc<ThreadManager>,
    running_requests_id_to_codex_uuid: Arc<Mutex<HashMap<RequestId, ThreadId>>>,
}

impl MessageProcessor {
    pub async fn process_request(&mut self, request: Request) {
        match request.method.as_str() {
            "initialize" => self.handle_initialize(request.id, request.params).await,
            "tools/list" => self.handle_list_tools(request.id, request.params).await,
            "tools/call" => self.handle_call_tool(request.id, request.params).await,
            "resources/list" => self.handle_list_resources(request.id, request.params).await,
            "resources/read" => self.handle_read_resource(request.id, request.params).await,
            _ => self.send_error(request.id, "Method not found").await,
        }
    }

    async fn handle_list_tools(&self, id: RequestId, _params: Option<Value>) {
        let result = ListToolsResult {
            tools: vec![
                create_tool_for_codex_tool_call_param(),
                create_tool_for_codex_tool_call_reply_param(),
            ],
            next_cursor: None,
        };
        self.send_response::<ListToolsRequest>(id, result).await;
    }
}
```

---

## 7.8 与 Claude Code 的对比

| 维度 | Codex | Claude Code |
|------|-------|-------------|
| **MCP 角色** | 客户端 + 服务器 | 仅客户端 |
| **工具名称** | `mcp__<server>__<tool>` | 类似 |
| **Schema 清洗** | 主动清洗 + 类型推断 | 类似 |
| **OAuth 支持** | 完整实现 | 部分支持 |
| **资源访问** | 分页 + 并发聚合 | 类似 |

**关键差异**：Codex 同时实现了 MCP 服务器，可以被其他 MCP 客户端调用。这使得 Codex 可以作为"Agent 中的 Agent"被编排。

---

## 章节衔接

**本章回顾**：
- 我们深入分析了 MCP 协议集成
- 关键收获：客户端 + 服务器双角色 + 工具转换 + OAuth 认证

**下一章预告**：
- 在 `08-deep-dive.md` 中，我们将进行核心源码深度解析
- 为什么需要学习：源码是最权威的文档
- 关键问题：关键函数的实现细节？设计决策的 trade-off？
