# 核心源码深度解析

本章不讲架构，只讲代码。选取 OpenClaw 中最能体现设计品味的五段源码，逐行剖析其设计意图和技术权衡。

## 1. 余弦相似度：手写而非调库

`src/memory/internal.ts:279-298`

```typescript
export function cosineSimilarity(a: number[], b: number[]): number {
  if (a.length === 0 || b.length === 0) return 0;
  const len = Math.min(a.length, b.length);
  let dot = 0;
  let normA = 0;
  let normB = 0;
  for (let i = 0; i < len; i += 1) {
    const av = a[i] ?? 0;
    const bv = b[i] ?? 0;
    dot += av * bv;
    normA += av * av;
    normB += bv * bv;
  }
  if (normA === 0 || normB === 0) return 0;
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

**为什么手写？** 这是 sqlite-vec 不可用时的内存回退路径。依赖一个 WASM 向量库（如 hnswlib.js）会增加几十 MB 的二进制依赖。对于 OpenClaw 的规模（通常几百到几千个 chunk），O(n) 线性扫描的手写余弦足够快。

**细节品味**：
- `Math.min(a.length, b.length)` 处理维度不匹配（不同模型可能产生不同维度的向量）
- `a[i] ?? 0` 防御性处理稀疏向量
- 先检查 `normA === 0 || normB === 0` 避免除零，而不是在循环内加条件

## 2. 并发控制器：手写 worker pool

`src/memory/internal.ts:300-336`

```typescript
export async function runWithConcurrency<T>(
  tasks: Array<() => Promise<T>>,
  limit: number,
): Promise<T[]> {
  const resolvedLimit = Math.max(1, Math.min(limit, tasks.length));
  const results: T[] = Array.from({ length: tasks.length });
  let next = 0;
  let firstError: unknown = null;

  const workers = Array.from({ length: resolvedLimit }, async () => {
    while (true) {
      if (firstError) return;
      const index = next;
      next += 1;
      if (index >= tasks.length) return;
      try {
        results[index] = await tasks[index]();
      } catch (err) {
        firstError = err;
        return;
      }
    }
  });

  await Promise.allSettled(workers);
  if (firstError) throw firstError;
  return results;
}
```

**设计分析**：这是一个经典的 worker pool 模式。`resolvedLimit` 个 worker 共享一个任务队列（通过递增 `next` 实现无锁分配）。

**为什么不用 `p-limit` 或 `async-pool`？** 减少依赖。这 30 行代码做的事情和 `p-limit` 完全一样，但避免了一个外部依赖。在 OpenClaw 的场景中（嵌入批处理），并发度通常是 4，任务数量通常是几十到几百。

**快速失败**：`firstError` 标志确保一个任务失败后，其他 worker 立即停止拉取新任务。这对于 embedding API 调用特别重要——如果第一个批次就因为 API Key 无效而失败，没必要继续尝试后续批次。

## 3. FTS5 查询构建：防注入的精巧设计

`src/memory/hybrid.ts:23-34`

```typescript
export function buildFtsQuery(raw: string): string | null {
  const tokens = raw
    .match(/[A-Za-z0-9_]+/g)
    ?.map(t => t.trim())
    .filter(Boolean) ?? [];
  if (tokens.length === 0) return null;
  const quoted = tokens.map(t => `"${t.replaceAll('"', "")}"`);
  return quoted.join(" AND ");
}
```

**为什么不直接传递用户输入？** FTS5 有自己的查询语法（`AND`、`OR`、`NOT`、`NEAR`），直接传递用户输入可能导致语法错误或 FTS 注入。

**处理流程**：
1. 正则提取纯单词 token（去除所有特殊字符）
2. 每个 token 用双引号包裹（精确匹配）
3. 删除 token 中的引号（防止引号注入）
4. 用 `AND` 连接（所有关键词必须出现）

**权衡**：这种策略牺牲了 FTS5 的高级查询能力（短语搜索、proximity），换取了安全性。对于 Memory 搜索场景，用户查询通常是"项目偏好"这类短语，AND 连接已经足够精确。

## 4. 嵌入缓存：多维复合主键

`src/memory/memory-schema.ts:38-49`

```sql
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);
```

**四元组主键的设计意图**：

- `provider` + `model`：同一段文本在不同模型下的嵌入完全不同
- `provider_key`：同一个 provider 可能有不同的 API Key/endpoint，产生的嵌入可能不同（比如 fine-tuned 模型）
- `hash`：文本内容的 SHA-256

这意味着切换 embedding 模型时，旧缓存不会被错误使用，也不会被删除——它们仍然有效，只是不再被命中。当用户切换回旧模型时，缓存依然可用。

## 5. Auto-capture 的规则引擎

`extensions/memory-lancedb/index.ts:185-236`

```typescript
const MEMORY_TRIGGERS = [
  /zapamatuj si|pamatuj|remember/i,     // 多语言"记住"
  /preferuji|radši|nechci|prefer/i,     // 多语言"偏好"
  /rozhodli jsme|budeme používat/i,     // 捷克语"我们决定"
  /\+\d{10,}/,                          // 电话号码
  /[\w.-]+@[\w.-]+\.\w+/,              // 邮件地址
  /můj\s+\w+\s+je|je\s+můj/i,         // 捷克语"我的...是"
  /my\s+\w+\s+is|is\s+my/i,           // 英语"我的...是"
  /i (like|prefer|hate|love|want|need)/i,
  /always|never|important/i,
];

function shouldCapture(text: string): boolean {
  if (text.length < 10 || text.length > 500) return false;
  if (text.includes("<relevant-memories>")) return false;
  if (text.startsWith("<") && text.includes("</")) return false;
  if (text.includes("**") && text.includes("\n-")) return false;
  const emojiCount = (text.match(/[\u{1F300}-\u{1F9FF}]/gu) || []).length;
  if (emojiCount > 3) return false;
  return MEMORY_TRIGGERS.some(r => r.test(text));
}
```

**分层过滤设计**：

1. **长度过滤**（10-500 字符）：太短没有信息量，太长可能是整段对话
2. **递归保护**：`<relevant-memories>` 检查避免把自己注入的上下文再次捕获
3. **系统内容过滤**：XML-like 标签是系统生成的结构化内容
4. **格式化内容过滤**：包含 `**` 和 `-` 列表的通常是 Agent 的格式化回复
5. **Emoji 密度过滤**：Emoji 密集的内容通常是 Agent 的情感化输出
6. **关键词触发**：只有命中特定模式才认为值得记忆

**多语言支持**的出现（捷克语 + 英语）暗示了项目的实际使用场景——开发者是捷克语使用者，这些规则来自真实使用反馈，而非理论设计。

**分类函数**（`detectCategory`）同样使用正则：

```typescript
function detectCategory(text: string): MemoryCategory {
  if (/prefer|like|love|hate|want/i.test(text)) return "preference";
  if (/decided|will use/i.test(text)) return "decision";
  if (/\+\d{10,}|@[\w.-]+\.\w+/i.test(text)) return "entity";
  if (/is|are|has|have/i.test(text)) return "fact";
  return "other";
}
```

这是一个有意的简化——用 LLM 分类可以更准确，但成本和延迟不可接受。正则分类在 80% 的场景下足够好，而且是零成本的。

## 6. WebSocket 慢消费者保护

`src/gateway/server-broadcast.ts:34-119`

广播的慢消费者保护逻辑：

```typescript
// 伪代码简化
broadcast(event, payload, opts) {
  for (const client of clients) {
    // 作用域检查
    if (!hasRequiredScope(client, opts.scopes)) continue;
    // 慢消费者检查
    if (client.socket.bufferedAmount > MAX_BUFFERED_BYTES) {
      if (opts.dropIfSlow) {
        // 丢弃事件，客户端不会知道
        continue;
      } else {
        // 关闭连接，强制重连
        client.socket.close(1008, "slow consumer");
        continue;
      }
    }
    client.socket.send(JSON.stringify({ type: "event", event, payload }));
  }
}
```

**两种策略的适用场景**：
- `dropIfSlow`：适用于 Agent 流式事件（丢失几个 delta 不影响最终结果）
- 关闭连接：适用于关键状态事件（必须收到，否则客户端状态不一致）

这种设计借鉴了消息队列的背压（backpressure）理念。在单用户场景下，慢消费者通常是网络问题导致的，强制重连后客户端可以从 snapshot 重建状态。

## 7. 路由匹配的层级回退

`src/routing/resolve-route.ts:168-262`

路由匹配使用了一个优雅的线性扫描，而不是复杂的路由树：

```typescript
// 简化后的匹配逻辑
for (const binding of cfg.bindings) {
  const match = binding.match;
  // 精确匹配链：从最具体到最宽泛
  if (match.peer && match.peer === input.peerId) return binding.agentId;
  if (match.guildId && match.guildId === input.guildId) return binding.agentId;
  if (match.teamId && match.teamId === input.teamId) return binding.agentId;
  if (match.accountId && match.accountId === input.accountId) return binding.agentId;
  if (match.channel && match.channel === input.channel) return binding.agentId;
}
return resolveDefaultAgentId(cfg);
```

**为什么线性扫描而不是 Trie/HashMap？** 因为 bindings 列表通常只有几个到十几个条目。线性扫描的 O(n) 在 n < 20 时比 HashMap 的常数开销更小，而且代码简单得多。

**匹配顺序的设计意图**：peer > guild > team > account > channel > default。越精确的匹配越优先，确保"给特定用户配置专属 Agent"不会被"给整个通道配置默认 Agent"覆盖。

---

## 章节衔接

**本章回顾**：
- 我们深入分析了 7 段核心源码的实现细节和设计意图
- 关键收获：OpenClaw 在多处选择"手写简单实现"而非引入依赖，体现了本地优先产品的设计哲学

**下一章预告**：
- 在 `07-interview-qa.md` 中，我们将面对 15 道高质量面试题
- 为什么需要学习：前面所有的技术分析都是为了在面试中能从容回答
- 关键问题：从架构决策、Memory 设计到生产工程的全方位考验
