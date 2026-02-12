# 面试 QA（15 题）

> 以面试官视角设计，从 OpenClaw 源码出发，覆盖架构设计、Memory 系统、Agent 运行时、插件架构和生产工程。

---

### Q1: OpenClaw 的整体架构是怎样的？为什么选择 Gateway 作为中心化控制平面？
**难度**: ★★★ | **类别**: 架构决策

OpenClaw 采用三层架构：通道层（Channel Layer）、控制层（Gateway）、智能层（Agent Runtime）。

Gateway 是运行在本地的 WebSocket + HTTP 服务器（默认端口 18789），所有通道、Agent、CLI、Web UI 和移动节点都通过它交互。选择中心化控制平面而非点对点架构的原因：

1. **状态一致性**：会话状态、通道状态、Agent 执行状态需要一个单一真相源。分布式状态在单用户场景下是不必要的复杂性。
2. **事件路由**：Agent 的一个回复可能需要同时推送给 WebSocket 客户端和原始通道（如 WhatsApp），中心化控制平面天然适合做事件扇出。
3. **部署简单性**：单进程、单端口，用户只需 `npm install -g openclaw` 就能运行，不需要 Docker 编排或多进程管理。

Trade-off：单点故障。如果 Gateway 崩溃，所有通道和 Agent 都不可用。但对于个人助手场景，这是可接受的——重启即恢复。

请回答：
1. 如果要支持多用户（SaaS 模式），Gateway 架构需要怎么改？
2. WebSocket 和 HTTP 为什么需要共存？能不能只用 WebSocket？
3. Gateway 的三帧协议（req/res/event）对比 gRPC 有什么优劣？

**源码引用**:
- `src/gateway/server.impl.ts:152-456` (Gateway 启动流程)
- `src/gateway/protocol/schema/frames.ts:129-169` (协议帧定义)

---

### Q2: Memory 系统为什么设计成双层架构（memory-core + memory-lancedb）？
**难度**: ★★★★ | **类别**: Memory 设计

memory-core 和 memory-lancedb 代表了两种记忆哲学：

- **memory-core**（文件级记忆）：数据源是 Markdown 文件和会话转录，使用 SQLite + sqlite-vec + FTS5 存储。"记忆即文件"——用户显式控制记忆内容。
- **memory-lancedb**（对话级记忆）：数据源是对话中的重要片段，使用 LanceDB 存储。通过 `shouldCapture` 规则自动提取记忆，通过 `before_agent_start` 钩子自动注入上下文。

两者通过插件槽位机制互斥：`plugins.slots.memory` 只能设为 `"memory-core"` 或 `"memory-lancedb"` 或 `"none"`。

设计原因：
1. **不同的信任模型**：文件记忆是用户主动写入的，可信度高；对话记忆是自动捕获的，可能包含噪声。
2. **不同的搜索需求**：文件记忆需要混合搜索（语义 + 关键词），对话记忆只需向量搜索。
3. **不同的生命周期**：文件记忆随文件存在，对话记忆有 GDPR 删除需求（`memory_forget`）。

请回答：
1. 为什么两个 Memory 插件不能同时激活？如果允许会有什么问题？
2. memory-lancedb 的 `shouldCapture` 用正则而不是 LLM 判断，这个决策的 trade-off 是什么？
3. 如果让你重新设计，会把两个 Memory 合并成一个还是保持分离？

**源码引用**:
- `src/plugins/slots.ts:12-18` (槽位定义)
- `extensions/memory-lancedb/index.ts:185-219` (shouldCapture 规则)

---

### Q3: 混合搜索（Hybrid Search）的加权融合策略是怎样的？为什么默认 vector 0.7 / text 0.3？
**难度**: ★★★★ | **类别**: Memory 设计

混合搜索同时执行向量搜索（语义匹配）和 FTS5 关键词搜索（精确匹配），然后通过加权平均融合结果。

核心算法（`src/memory/hybrid.ts:41-115`）：
1. 以 chunk id 为键建立映射表
2. 向量结果和关键词结果中的同一 chunk 被合并
3. 最终分数 = `vectorWeight * vectorScore + textWeight * textScore`
4. 按分数降序排列

默认权重 0.7/0.3 的原因：用户查询通常是自然语言（"我之前说的项目偏好"），语义匹配更贴合这种查询模式。关键词匹配作为补充，处理精确术语搜索。

BM25 分数通过 `1 / (1 + rank)` 映射到 0-1 范围，这是一个 S 型衰减函数，确保与余弦相似度的值域对齐。

请回答：
1. 这个 0.7/0.3 的权重如何动态调优？有没有更好的融合策略（如 Reciprocal Rank Fusion）？
2. 当 sqlite-vec 不可用时，系统如何降级？对搜索质量有什么影响？
3. FTS5 查询构建中的防注入策略（`buildFtsQuery`）是否足够安全？

**源码引用**:
- `src/memory/hybrid.ts:41-115` (mergeHybridResults)
- `src/memory/hybrid.ts:36-39` (bm25RankToScore)

---

### Q4: 嵌入提供者的 fallback 链设计有什么考量？
**难度**: ★★★ | **类别**: Memory 设计

`createEmbeddingProvider`（`src/memory/embeddings.ts:130-213`）实现了一个多层 fallback 策略：

- `auto` 模式：local（如果本地模型可用）→ openai → gemini → voyage
- 显式模式：primary provider → fallback provider

设计考量：
1. **本地优先**：如果 `embeddinggemma-300M` 模型文件已下载，优先使用本地推理（零成本、零延迟、离线可用）。
2. **云端回退**：本地不可用时，按 API Key 可用性依次尝试云端提供者。
3. **缓存兼容**：embedding_cache 表使用 `(provider, model, provider_key, hash)` 四元组主键，切换提供者不会导致缓存冲突。

请回答：
1. 如果在 auto 模式下，本地模型和云端模型产生不同维度的嵌入怎么办？
2. 嵌入缓存的四元组主键设计有什么优势？如果简化为 `(model, hash)` 会有什么问题？
3. 离线场景下，如果没有预下载本地模型，Memory 搜索会完全失败还是优雅降级？

**源码引用**:
- `src/memory/embeddings.ts:130-213` (createEmbeddingProvider)
- `src/memory/memory-schema.ts:38-49` (embedding_cache schema)

---

### Q5: Gateway 的 WebSocket 认证体系是如何设计的？
**难度**: ★★★★ | **类别**: 架构决策

Gateway 支持四种认证方式：Token、Password、Tailscale、Device Pairing。

认证在 WebSocket 握手阶段完成（`src/gateway/auth.ts:92-276`）。ConnectParams 中携带 `auth.token` 或 `auth.password`，Gateway 与配置中的凭据比对。Tailscale 认证利用 Tailscale 代理注入的身份头，Device Pairing 使用公钥签名验证。

角色体系：`operator`（管理者）和 `node`（设备节点）。Operator 可以调用所有 RPC 方法，Node 只能执行受限操作。

请回答：
1. 为什么 DM 策略默认是 `pairing`（需要配对码批准）而不是 `open`？
2. 如何防止 Gateway 暴露在公网时的暴力破解攻击？
3. 如果 Tailscale 身份头被伪造怎么办？

**源码引用**:
- `src/gateway/auth.ts:92-276` (认证逻辑)
- `src/gateway/server/ws-connection/message-handler.ts:131-766` (握手处理)

---

### Q6: 插件系统的发现-注册-激活流程是怎样的？
**难度**: ★★★ | **类别**: 架构决策

完整流程：
1. `discoverOpenClawPlugins()` — 扫描 4 个目录层级
2. `loadPluginManifestRegistry()` — 读取 `openclaw.plugin.json`
3. `resolveEnableState()` — 9 级决策链决定启用/禁用
4. `jiti(candidate.source)` — 运行时加载 TypeScript
5. `register(api)` — 插件注册工具、通道、钩子等
6. `initializeGlobalHookRunner()` — 构建全局钩子运行器

关键设计：使用 `jiti` 实现运行时 TypeScript 加载，避免了 build step。`openclaw/plugin-sdk` 别名确保插件可以用标准 import 引用 SDK。

请回答：
1. 为什么用 jiti 而不是 esbuild/tsx/tsup 做运行时加载？trade-off 是什么？
2. 插件之间如果有依赖关系怎么处理？当前设计支持吗？
3. 如果一个插件的 `register` 函数抛出异常，其他插件会受影响吗？

**源码引用**:
- `src/plugins/loader.ts:168-341` (插件加载主流程)
- `src/plugins/config-state.ts:163-193` (启用决策)

---

### Q7: 消息路由的层级匹配设计为什么不用 HashMap 而用线性扫描？
**难度**: ★★★★ | **类别**: 架构决策

路由匹配（`src/routing/resolve-route.ts:168-262`）按优先级线性扫描 bindings 列表：peer → guild → team → account → channel → default。

选择线性扫描而非 HashMap 的原因：
1. bindings 列表通常 < 20 条，O(n) 扫描比 HashMap 的哈希计算 + 碰撞处理更快
2. 匹配优先级是多维的（同一条 binding 可能同时匹配 channel 和 account），HashMap 难以表达
3. 代码简单，易于调试和审计

请回答：
1. 如果 bindings 增长到 1000+（企业场景），路由延迟会如何变化？需要什么优化？
2. 路由结果是否应该缓存？如果缓存，失效策略怎么设计？
3. 当前的匹配顺序是否可以自定义？如果用户想让 account 优先于 guild 呢？

**源码引用**:
- `src/routing/resolve-route.ts:168-262` (路由匹配)
- `src/routing/session-key.ts:129-184` (会话键构建)

---

### Q8: Pi Agent 嵌入模式 vs HTTP RPC 模式的 trade-off 是什么？
**难度**: ★★★★ | **类别**: Agent 核心

OpenClaw 选择让 Pi Agent 在 Gateway 进程内嵌入运行，而不是作为独立服务通过 HTTP RPC 调用。

优势：
- 零 IPC 开销：Agent 事件直接通过函数调用传递
- 共享内存：Agent 可以直接访问 Gateway 的配置和状态
- 部署简单：单进程，不需要管理多个服务

劣势：
- 隔离性差：Agent 崩溃可能导致 Gateway 崩溃
- 无法独立扩展：Agent 和 Gateway 绑定在同一个进程
- 内存压力：大模型的上下文可能占用大量内存

请回答：
1. 如果要支持多个并发 Agent 执行，当前架构能承受吗？
2. Agent 的工具执行（如 browser、bash）如果超时或卡死，如何保护 Gateway？
3. 如果重新设计，你会选择嵌入模式还是进程隔离模式？

**源码引用**:
- `src/agents/pi-embedded-runner/run/attempt.ts:479-491` (createAgentSession)
- `src/agents/pi-embedded-subscribe.ts:602` (事件订阅)

---

### Q9: 会话键（Session Key）的编码设计有什么深意？
**难度**: ★★★ | **类别**: Agent 核心

会话键格式：`agent:{agentId}:{channel}:{accountId}:{peerKind}:{peerId}`

这种编码在一个字符串中包含了完整的路由信息。好处：
1. 从键值即可反推源通道和对话方
2. 不同通道的相同用户（如 WhatsApp 和 Telegram 上的同一个人）有不同的会话，避免上下文混淆
3. 群组和私聊天然隔离（peerKind = `dm` vs `group`）

请回答：
1. 如果同一个用户在 WhatsApp 和 Telegram 上都联系 Agent，应该合并还是分离会话？
2. 会话键如果改成 UUID 会有什么影响？
3. 当前设计如何支持"跨通道续聊"（用户从 WhatsApp 切换到 WebChat）？

**源码引用**:
- `src/routing/session-key.ts:129-184` (会话键构建)

---

### Q10: 钩子系统为什么区分 void 和 modifying 两种执行模式？
**难度**: ★★★★ | **类别**: 架构决策

Void 钩子（如 `agent_end`）并行执行，一个失败不影响其他。Modifying 钩子（如 `before_agent_start`）按 priority 串行执行，后一个可以看到前一个的修改。

设计原因：
- `before_agent_start` 可能注入上下文（memory-lancedb 的 auto-recall），多个钩子可能同时注入，必须串行合并
- `agent_end` 用于日志、监控、记忆捕获等观察性操作，并行执行性能更好
- `tool_result_persist` 是同步执行的（不是 async），因为它在热路径上，延迟敏感

请回答：
1. 如果两个 modifying 钩子修改了同一个字段，如何合并？
2. 钩子执行超时怎么处理？有 timeout 机制吗？
3. 如何保证钩子不会拖慢 Agent 的响应速度？

**源码引用**:
- `src/plugins/hooks.ts:80-88` (钩子排序)
- `src/plugins/types.ts:292-306` (钩子名称定义)

---

### Q11: 通道抽象层（Channel Dock）如何做到通道无关的消息处理？
**难度**: ★★★ | **类别**: 架构决策

Channel Dock 为每个通道声明了能力元数据（chatTypes、media、threading 等），Gateway 根据能力声明决定行为，而不是检查通道类型。

这种"能力声明"模式使得新增通道不需要修改 Gateway 核心代码。Gateway 只问"你支持图片吗？"而不是"你是 Telegram 吗？"。

请回答：
1. 如果一个通道声明支持图片但实际发送失败，Gateway 如何处理？
2. 不同通道的消息长度限制不同（WhatsApp 65K, Telegram 4096），如何统一处理？
3. 如果要加一个视频通话通道，需要修改哪些层？

**源码引用**:
- `src/channels/dock.ts:91-395` (通道能力声明)

---

### Q12: 增量索引的变更检测为什么用 SHA-256 哈希而不是 mtime？
**难度**: ★★★★ | **类别**: Memory 设计

Memory 的增量索引使用文件内容的 SHA-256 哈希来检测变更。虽然 `MemoryFileEntry` 中也包含 `mtimeMs`，但实际的变更判断基于 `hash`。

原因：
1. `mtime` 在某些操作（`touch`、备份恢复、git checkout）后会变化但内容不变，导致不必要的重新嵌入
2. `mtime` 在某些文件系统（NFS、部分 WSL2）上精度不够
3. SHA-256 是内容寻址的，保证了"内容不变 = 不需要重新索引"

Trade-off：读取文件内容计算哈希比只读 stat 更慢。但 Memory 文件通常很小（几 KB 到几百 KB），读取开销可以忽略。

请回答：
1. 如果 workspace 中有大文件（10MB+ 的 MEMORY.md），哈希计算会成为瓶颈吗？
2. 嵌入向量的缓存是否也用哈希来判断？
3. 如何优化"大量小文件"的同步场景（如 memory/ 下有 1000 个文件）？

**源码引用**:
- `src/memory/internal.ts:146-148` (hashText)
- `src/memory/internal.ts:150-164` (buildFileEntry)

---

### Q13: memory-lancedb 的去重策略是否可靠？
**难度**: ★★★★★ | **类别**: Memory 设计

Auto-capture 在存储前使用高阈值向量搜索去重：

```typescript
const existing = await db.search(vector, 1, 0.95);
if (existing.length > 0) continue;
```

0.95 的相似度阈值意味着几乎相同的内容才会被判为重复。

潜在问题：
1. **语义重复**："我喜欢咖啡"和"我偏好咖啡"语义相同但向量相似度可能低于 0.95
2. **时间维度**：用户的偏好可能变化，"我喜欢 TypeScript"和一年后的"我喜欢 Rust"不应被去重
3. **模型依赖**：不同 embedding 模型的相似度分布不同，0.95 不是通用阈值

请回答：
1. 如何设计一个更可靠的去重策略？是否需要引入时间衰减？
2. 如果用户显式说"更新我的偏好"，应该覆盖旧记忆还是追加新记忆？
3. memory_forget 工具的 GDPR 合规性如何保证完整删除（向量数据库中的删除是否是真删除）？

**源码引用**:
- `extensions/memory-lancedb/index.ts:339-355` (memory_store 去重)
- `extensions/memory-lancedb/index.ts:580-596` (auto-capture 去重)

---

### Q14: 如果让你从零设计 OpenClaw 的 Memory 系统，你会做什么不同？
**难度**: ★★★★★ | **类别**: 场景设计

这是一道开放题。以下是几个值得讨论的改进方向：

1. **统一搜索**：当前 memory-core 和 memory-lancedb 的搜索是分开的。可以设计一个统一的搜索层，同时查询文件记忆和对话记忆。

2. **记忆分级**：引入 Working Memory（当前会话上下文）→ Short-term Memory（最近几天的关键信息）→ Long-term Memory（持久化偏好和事实）的三级模型。

3. **智能遗忘**：基于时间衰减和访问频率自动降低记忆的重要度，类似 LRU 但基于语义重要性。

4. **图结构记忆**：将实体和关系存储为知识图谱，而不是平铺的向量条目。允许"A 的公司的竞争对手"这种多跳查询。

请回答：
1. 三级记忆模型（Working/Short-term/Long-term）的实现复杂度如何？值得吗？
2. 图结构记忆和向量记忆的 trade-off 是什么？
3. 如何评估 Memory 系统的质量（准确率、召回率、用户满意度）？

---

### Q15: OpenClaw 的架构在 AI Agent 发展趋势下会如何演进？
**难度**: ★★★★★ | **类别**: 场景设计

OpenClaw 当前是单用户、本地运行的个人助手。随着 AI Agent 领域的发展，几个可能的演进方向：

1. **Multi-Agent 协作**：当前的 `sessions_*` 工具提供了基础的 Agent 间通信，但缺乏协作框架（如任务分配、结果聚合、冲突解决）。

2. **MCP 生态**：OpenClaw 已经集成了 ACP（Agent Client Protocol），未来可能需要更深入的 MCP（Model Context Protocol）支持，让外部工具无缝接入。

3. **边缘计算**：将部分 Agent 能力下放到移动设备节点，实现真正的端侧智能（当前节点只做设备操作，不做推理）。

4. **记忆共享**：不同 Agent 之间共享记忆（如工作助手记住的项目信息，生活助手也能引用）。

请回答：
1. Multi-Agent 协作框架应该建在 Gateway 层还是 Agent 层？
2. 如果要支持 MCP Server 模式（OpenClaw 作为 MCP Server 暴露工具），架构需要怎么改？
3. 端侧推理和云端推理的混合调度怎么设计？
