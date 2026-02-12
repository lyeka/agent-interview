# OpenClaw Memory 系统设计

Memory 系统是 OpenClaw 中最具技术深度的子系统之一。它解决了一个核心问题：让 AI 助手"记住"用户的偏好、决策和历史对话，而不仅仅是在当前会话的 Context Window 内工作。

OpenClaw 的 Memory 设计采用了双层架构：文件级记忆（memory-core）和对话级记忆（memory-lancedb），两者通过插件槽位机制互斥挂载，同时共享底层的嵌入服务。

## Memory 系统全景

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MEMORY 系统全景                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌────────────────┐    ┌─────────────────┐    ┌──────────────────┐ │
│  │ memory_search  │    │  memory_recall   │    │   memory_store   │ │
│  │ memory_get     │    │  memory_forget   │    │                  │ │
│  │ (memory-core)  │    │ (memory-lancedb) │    │ (memory-lancedb) │ │
│  └───────┬────────┘    └────────┬─────────┘    └────────┬─────────┘ │
│          │                      │                        │          │
│          ▼                      ▼                        ▼          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │          Plugin Slot: memory (只能激活一个)                    │   │
│  └──────────────────┬───────────────────────────┬──────────────┘   │
│                     │                           │                   │
│          ┌──────────▼──────────┐     ┌──────────▼──────────┐       │
│          │  MemoryIndexManager │     │     MemoryDB        │       │
│          │  (SQLite + FTS5 +   │     │   (LanceDB)         │       │
│          │   sqlite-vec)       │     │                     │       │
│          └──────────┬──────────┘     └──────────┬──────────┘       │
│                     │                           │                   │
│          ┌──────────▼──────────────────────────▼──────────────┐    │
│          │            Embedding Providers                      │    │
│          │   OpenAI | Gemini | Voyage | Local (node-llama-cpp) │    │
│          └────────────────────────────────────────────────────┘    │
│                                                                     │
│  数据源:                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐   │
│  │ MEMORY.md   │  │ memory/*.md  │  │ Session Transcripts      │   │
│  │ memory.md   │  │ (workspace)  │  │ (~/.openclaw/sessions/)  │   │
│  └─────────────┘  └──────────────┘  └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## 数据模型

### 核心类型定义

Memory 系统的数据模型由四个关键类型构成。理解这些类型是理解整个系统的前提。

文件级入口和内容块的定义：

```typescript
// src/memory/internal.ts:6-19 — 基础数据类型
type MemoryFileEntry = {
  path: string;      // 相对于 workspace 的路径
  absPath: string;   // 绝对路径
  mtimeMs: number;   // 修改时间（变更检测的基础）
  size: number;
  hash: string;      // SHA-256 内容哈希（增量索引的依据）
};

type MemoryChunk = {
  startLine: number;  // 块的起始行号
  endLine: number;    // 块的结束行号
  text: string;       // 块的文本内容
  hash: string;       // 块内容的哈希
};
```

搜索结果的统一接口：

```typescript
// src/memory/types.ts:1-11 — 搜索结果
type MemorySearchResult = {
  path: string;
  startLine: number;
  endLine: number;
  score: number;       // 0-1 的相关性分数
  snippet: string;     // 截断后的文本片段
  source: "memory" | "sessions";  // 来源标识
  citation?: string;   // 可选的引用标记
};
```

**设计意图**：`source` 字段区分了"用户主动写的记忆"和"对话历史中提取的记忆"。这两种记忆的可信度和用途不同——主动记忆通常是高质量的参考资料，而会话记忆可能包含噪声。

### MemorySearchManager 接口

这是 Memory 系统对外暴露的唯一契约：

```typescript
// src/memory/types.ts:61-80 — 搜索管理器接口
interface MemorySearchManager {
  search(query: string, opts?: {
    maxResults?: number;
    minScore?: number;
    sessionKey?: string;
  }): Promise<MemorySearchResult[]>;

  readFile(params: {
    relPath: string; from?: number; lines?: number;
  }): Promise<{ text: string; path: string }>;

  status(): MemoryProviderStatus;
  sync?(params?: { reason?: string; force?: boolean }): Promise<void>;
  probeEmbeddingAvailability(): Promise<MemoryEmbeddingProbeResult>;
  probeVectorAvailability(): Promise<boolean>;
  close?(): Promise<void>;
}
```

**为什么是接口而不是具体类？** 因为 Memory 系统有两个后端实现：内置的 `MemoryIndexManager`（SQLite）和外部的 `QmdMemoryManager`。通过接口抽象，上层代码不关心底层存储。

## 存储架构

### SQLite Schema

内置后端使用 Node.js 原生 SQLite（`node:sqlite`），表结构如下：

```sql
-- src/memory/memory-schema.ts:9-36 — 核心表结构
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);

CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,    -- 生成 embedding 的模型名
  text TEXT NOT NULL,
  embedding TEXT NOT NULL, -- JSON 序列化的向量
  updated_at INTEGER NOT NULL
);
```

关键设计点：

1. **`source` 字段**：区分 `memory`（文件）和 `sessions`（对话历史），允许按来源过滤
2. **`model` 字段**：记录生成 embedding 的模型，当切换模型时可以识别需要重新嵌入的块
3. **`embedding` 以 JSON TEXT 存储**：这是一个刻意的简化——避免了自定义二进制格式的复杂性

此外还有三个扩展表：

```sql
-- FTS5 全文搜索虚拟表
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  id UNINDEXED, path UNINDEXED, source UNINDEXED,
  model UNINDEXED, start_line UNINDEXED, end_line UNINDEXED
);

-- sqlite-vec 向量搜索虚拟表
-- 动态创建，维度由 embedding 模型决定

-- 嵌入缓存表
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

**为什么用 SQLite 而不是专用向量数据库？** 这是 OpenClaw "本地优先"哲学的体现。SQLite 是零配置、零依赖的嵌入式数据库，配合 `sqlite-vec` 扩展就能提供向量搜索。用户不需要安装 Pinecone、Qdrant 或 Milvus。

## 文档分块算法

分块是将长文档切成适合嵌入的小段的过程。OpenClaw 的实现在 `chunkMarkdown` 函数中：

```typescript
// src/memory/internal.ts:166-246 — Markdown 分块
function chunkMarkdown(
  content: string,
  chunking: { tokens: number; overlap: number },
): MemoryChunk[] {
  const lines = content.split("\n");
  const maxChars = Math.max(32, chunking.tokens * 4);
  const overlapChars = Math.max(0, chunking.overlap * 4);
  // ...
  // 当累积字符数超过 maxChars 时，flush 当前块
  // carryOverlap() 从当前块尾部保留 overlapChars 字符
  // 作为下一块的开头
}
```

分块策略的三个关键参数：
- `tokens`（默认 512）：每块的目标 token 数
- `overlap`（默认 64）：相邻块的重叠 token 数
- 使用 `tokens * 4` 估算字符数（粗略但高效）

**Overlap 的意义**：如果一个关键信息恰好跨越两个块的边界，没有重叠就会被割裂。重叠确保边界区域的信息在两个块中都有副本。

**为什么基于行而不是基于句子分块？** 因为 Memory 文件是 Markdown 格式，基于行的分块天然尊重 Markdown 的结构（标题、列表、代码块），而基于句子的分块可能在段落中间截断。

## 嵌入提供者（Embedding Provider）

OpenClaw 支持四种嵌入提供者，通过策略模式切换：

| 提供者 | 默认模型 | 特点 |
|--------|---------|------|
| OpenAI | `text-embedding-3-small` | 最稳定，需要 API Key |
| Gemini | `gemini-embedding-001` | Google 生态 |
| Voyage | `voyage-4-large` | 高质量嵌入 |
| Local | `embeddinggemma-300M` | 零成本，通过 node-llama-cpp |

提供者的创建使用了 fallback 链：

```typescript
// src/memory/embeddings.ts:130-213 — 提供者创建与回退
async function createEmbeddingProvider(
  options: EmbeddingProviderOptions,
): Promise<EmbeddingProviderResult> {
  if (requestedProvider === "auto") {
    // 尝试顺序：local（如果本地模型可用） → openai → gemini → voyage
    if (canAutoSelectLocal(options)) {
      try { return await createProvider("local"); } catch {}
    }
    for (const provider of ["openai", "gemini", "voyage"]) {
      try { return await createProvider(provider); } catch {}
    }
  }
  // 显式指定的提供者失败时，尝试 fallback
  try {
    return await createProvider(requestedProvider);
  } catch {
    if (fallback && fallback !== "none") {
      return await createProvider(fallback);
    }
  }
}
```

**设计亮点**：`auto` 模式优先尝试本地嵌入（如果模型文件已下载），这意味着在离线环境下 Memory 系统也能工作。只有本地不可用时才回退到云端 API。

嵌入向量的后处理确保了数值稳定性：

```typescript
// src/memory/embeddings.ts:11-18 — L2 归一化
function sanitizeAndNormalizeEmbedding(vec: number[]): number[] {
  const sanitized = vec.map(v => Number.isFinite(v) ? v : 0);
  const magnitude = Math.sqrt(
    sanitized.reduce((sum, v) => sum + v * v, 0)
  );
  if (magnitude < 1e-10) return sanitized;
  return sanitized.map(v => v / magnitude);
}
```

归一化后的向量使得余弦相似度等价于点积，简化了后续的相似度计算。

## 混合搜索（Hybrid Search）

这是 Memory 系统最精巧的部分。单纯的向量搜索擅长语义匹配（"天气预报"能匹配"气象信息"），但对精确关键词匹配效果不佳。FTS5 全文搜索擅长关键词匹配，但无法理解语义。混合搜索将两者结合。

搜索流程：

```typescript
// src/memory/manager.ts:266-314 — 混合搜索主流程
async search(query, opts?) {
  // 1. 同步检查：如果数据脏了，先同步
  if (this.settings.sync.onSearch && (this.dirty || this.sessionsDirty)) {
    void this.sync({ reason: "search" });
  }
  // 2. 关键词搜索（FTS5 + BM25）
  const keywordResults = hybrid.enabled
    ? await this.searchKeyword(cleaned, candidates)
    : [];
  // 3. 向量搜索（sqlite-vec 或内存回退）
  const queryVec = await this.embedQueryWithTimeout(cleaned);
  const vectorResults = hasVector
    ? await this.searchVector(queryVec, candidates)
    : [];
  // 4. 结果融合
  const merged = mergeHybridResults({
    vector: vectorResults,
    keyword: keywordResults,
    vectorWeight: hybrid.vectorWeight,  // 默认 0.7
    textWeight: hybrid.textWeight,      // 默认 0.3
  });
  return merged.filter(e => e.score >= minScore).slice(0, maxResults);
}
```

结果融合算法（`src/memory/hybrid.ts:41-115`）的核心逻辑：

```typescript
// src/memory/hybrid.ts:102-114 — 加权融合
const merged = Array.from(byId.values()).map(entry => {
  const score = vectorWeight * entry.vectorScore
              + textWeight * entry.textScore;
  return { path, startLine, endLine, score, snippet, source };
});
return merged.toSorted((a, b) => b.score - a.score);
```

融合策略的关键在于 `vectorWeight: 0.7` 和 `textWeight: 0.3` 的默认权重。这意味着语义匹配的权重是关键词匹配的 2.3 倍。这个比例反映了一个实际观察：用户查询通常是自然语言而非精确关键词，语义匹配更贴合真实意图。

BM25 分数到 0-1 范围的映射使用了一个巧妙的变换：

```typescript
// src/memory/hybrid.ts:36-39
function bm25RankToScore(rank: number): number {
  const normalized = Number.isFinite(rank) ? Math.max(0, rank) : 999;
  return 1 / (1 + normalized);
}
```

这是一个 S 型衰减函数，BM25 rank 为 0（最佳匹配）时 score = 1.0，rank 越大 score 越接近 0。

## 增量索引与同步

Memory 系统不会在每次搜索时重新索引所有文件。它使用文件哈希进行变更检测，只重新嵌入发生变化的块：

```
同步流程:
1. listMemoryFiles() — 扫描 workspace 下的 MEMORY.md + memory/*.md
2. 对每个文件计算 SHA-256 哈希
3. 与数据库中存储的哈希对比
4. 哈希不同 → 重新分块 → 重新嵌入 → 更新数据库
5. 哈希相同 → 跳过
6. 数据库中有但文件已删除 → 清除对应记录
```

同步的触发时机有四种：
- **文件变更**：chokidar 监视 `memory/` 目录，文件增删改触发同步
- **会话更新**：`onSessionTranscriptUpdate` 事件触发（有 5 秒去抖）
- **搜索前**：如果 `sync.onSearch` 启用且数据脏了
- **定时**：`sync.intervalMs` 配置的周期性同步

会话转录的增量同步特别精巧。它使用 delta 阈值避免频繁同步：

```
deltaBytes: 100KB — 对话内容增长超过 100KB 才触发
deltaMessages: 50 — 新增超过 50 条消息才触发
```

## memory-core vs memory-lancedb

这两个插件代表了两种完全不同的 Memory 哲学。

### memory-core：文件级记忆

memory-core 是默认的 Memory 插件。它的数据源是文件系统中的 Markdown 文件和会话转录日志。

```typescript
// extensions/memory-core/index.ts:10-26
register(api) {
  api.registerTool((ctx) => {
    const memorySearchTool = api.runtime.tools.createMemorySearchTool({...});
    const memoryGetTool = api.runtime.tools.createMemoryGetTool({...});
    return [memorySearchTool, memoryGetTool];
  }, { names: ["memory_search", "memory_get"] });
}
```

memory-core 提供两个工具：
- `memory_search`：语义搜索（向量 + FTS5 混合）
- `memory_get`：按路径和行号精确读取文件内容

它的哲学是"记忆即文件"——用户在 `MEMORY.md` 或 `memory/` 目录中写下的任何内容都会被索引。这是一种显式、可控的记忆模型。

### memory-lancedb：对话级记忆

memory-lancedb 使用 LanceDB 向量数据库，实现了自动化的对话记忆。

它的数据模型更丰富：

```typescript
// extensions/memory-lancedb/index.ts:38-45
type MemoryEntry = {
  id: string;
  text: string;
  vector: number[];
  importance: number;       // 0-1 重要度评分
  category: MemoryCategory; // preference | decision | entity | fact | other
  createdAt: number;
};
```

提供三个工具：
- `memory_recall`：向量搜索历史记忆
- `memory_store`：显式存储新记忆
- `memory_forget`：删除记忆（GDPR 合规）

最独特的是它的生命周期钩子：

**Auto-recall（`before_agent_start`）**：在 Agent 开始处理用户消息之前，自动搜索相关历史记忆并注入上下文：

```typescript
// extensions/memory-lancedb/index.ts:496-521
api.on("before_agent_start", async (event) => {
  const vector = await embeddings.embed(event.prompt);
  const results = await db.search(vector, 3, 0.3);
  if (results.length === 0) return;
  const memoryContext = results
    .map(r => `- [${r.entry.category}] ${r.entry.text}`)
    .join("\n");
  return {
    prependContext: `<relevant-memories>\n${memoryContext}\n</relevant-memories>`,
  };
});
```

**Auto-capture（`agent_end`）**：在 Agent 处理完成后，分析对话内容，自动提取并存储有价值的记忆：

```typescript
// extensions/memory-lancedb/index.ts:526-606
api.on("agent_end", async (event) => {
  const texts = extractTextsFromMessages(event.messages);
  const toCapture = texts.filter(shouldCapture);
  for (const text of toCapture.slice(0, 3)) {
    const category = detectCategory(text);
    const vector = await embeddings.embed(text);
    // 去重：相似度 > 0.95 的不重复存储
    const existing = await db.search(vector, 1, 0.95);
    if (existing.length > 0) continue;
    await db.store({ text, vector, importance: 0.7, category });
  }
});
```

Auto-capture 使用了基于规则的过滤器来判断哪些内容值得记忆：

```typescript
// extensions/memory-lancedb/index.ts:185-219
const MEMORY_TRIGGERS = [
  /remember|prefer|decided/i,
  /my\s+\w+\s+is|is\s+my/i,
  /always|never|important/i,
  // ... 电话号码、邮件地址等模式
];
function shouldCapture(text: string): boolean {
  if (text.length < 10 || text.length > 500) return false;
  if (text.includes("<relevant-memories>")) return false; // 避免递归
  return MEMORY_TRIGGERS.some(r => r.test(text));
}
```

**为什么用正则而不是 LLM 判断？** 性能。`agent_end` 钩子在每次对话结束后同步执行，用 LLM 判断会增加数百毫秒的延迟和 API 成本。正则模式匹配是零成本的。

### 对比分析

| 维度 | memory-core | memory-lancedb |
|------|-------------|----------------|
| 数据源 | Markdown 文件 + 会话日志 | 自由格式对话片段 |
| 存储后端 | SQLite + sqlite-vec | LanceDB |
| 搜索方式 | 混合搜索（向量 + FTS5） | 纯向量搜索（L2 距离） |
| 嵌入提供者 | 多提供者（OpenAI/Gemini/Voyage/Local） | 仅 OpenAI |
| 写入方式 | 被动（文件变更触发） | 主动 + 被动（工具 + 自动捕获） |
| 删除能力 | 无（随文件删除） | 有（memory_forget 工具） |
| 分类能力 | 无 | 有（preference/decision/entity/fact） |
| 记忆格式 | 非结构化文本块 | 结构化记忆条目（重要度 + 分类） |
| 默认状态 | 启用 | 需显式配置 |

## 插件槽位机制

Memory 系统通过插件槽位（slot）机制确保同一时刻只有一个 Memory 插件激活：

```typescript
// src/plugins/slots.ts:12-18
const SLOT_BY_KIND = { memory: "memory" };
const DEFAULT_SLOT_BY_KEY = { memory: "memory-core" };
```

配置 `plugins.slots.memory` 决定激活哪个插件。设为 `"none"` 则完全禁用 Memory。这个设计避免了两个 Memory 插件同时运行时的工具名冲突和数据不一致问题。

## Memory 与 Agent 的集成

Memory 工具通过 System Prompt 注入指令告诉 Agent 何时使用记忆：

```
当用户询问之前的工作、决策、偏好或历史对话时，
先使用 memory_search 工具检索相关记忆，
再基于检索结果回答。
```

这是一种"检索增强生成"（RAG）模式在 Agent 场景中的自然应用——Agent 不是被动地接收上下文，而是主动决定"我需要先查一下记忆"。

---

## 章节衔接

**本章回顾**：
- 我们深入学习了 OpenClaw 的双层 Memory 架构（文件级 + 对话级）
- 关键收获：混合搜索的加权融合算法和自动记忆捕获的规则引擎

**下一章预告**：
- 在 `05-plugin-extension.md` 中，我们将学习插件与扩展架构
- 为什么需要学习：Memory 系统本身就是一个插件，理解插件系统才能理解它是如何被加载和激活的
- 关键问题：插件发现、注册、生命周期管理和安全模型
