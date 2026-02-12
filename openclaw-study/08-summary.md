# OpenClaw 知识图谱总结

## 核心概念关系图

```
                         ┌────────────────────┐
                         │     OpenClaw        │
                         │ (个人 AI 助手网关)   │
                         └─────────┬──────────┘
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
    ┌──────────────┐    ┌──────────────────┐   ┌──────────────┐
    │  通道层       │    │    控制层          │   │   智能层      │
    │ (Ch.01/05)   │    │  (Ch.01/02)       │   │  (Ch.03/04)  │
    └──────┬───────┘    └────────┬─────────┘   └──────┬───────┘
           │                     │                     │
    ┌──────┴───────┐    ┌───────┴────────┐    ┌──────┴───────┐
    │ Channel      │    │   Gateway       │    │  Pi Agent    │
    │ Plugins      │───>│   WebSocket     │<───│  Runtime     │
    │ (extensions/)│    │   + HTTP        │    │  (Pi SDK)    │
    └──────────────┘    └───────┬────────┘    └──────┬───────┘
                                │                     │
                       ┌────────┴────────┐    ┌──────┴───────┐
                       │ 路由 + 会话      │    │   Memory     │
                       │ (src/routing/)  │    │  (src/memory/)│
                       │ (src/sessions/) │    │  (extensions/)│
                       └─────────────────┘    └──────────────┘

  通道层: Telegram, Discord, Slack, WhatsApp, Signal, iMessage, WebChat...
  控制层: WS 协议(3帧) + RPC(40方法) + 事件广播 + 认证(4方式)
  智能层: ReAct循环 + 工具系统 + 会话管理 + Memory(双层)
```

## Memory 系统关系图

```
┌─────────────────────────────────────────────────────────────┐
│                     Memory 架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐           ┌──────────────────┐        │
│  │  memory-core    │           │  memory-lancedb   │        │
│  │  (默认激活)      │    OR     │  (可选替代)        │        │
│  └────────┬────────┘           └────────┬─────────┘        │
│           │                             │                   │
│  ┌────────▼────────┐           ┌────────▼─────────┐        │
│  │ SQLite + FTS5   │           │    LanceDB       │        │
│  │ + sqlite-vec    │           │  + OpenAI embed   │        │
│  └────────┬────────┘           └────────┬─────────┘        │
│           │                             │                   │
│  数据源:                        数据源:                      │
│  • MEMORY.md                   • 对话中自动捕获              │
│  • memory/*.md                 • 用户显式存储                │
│  • 会话转录 (JSONL)             • 规则过滤:                  │
│                                  - shouldCapture()          │
│  搜索:                           - detectCategory()         │
│  • 混合搜索                                                 │
│    (vector 0.7 + FTS 0.3)     搜索:                        │
│  • 增量索引                    • 纯向量搜索 (L2 距离)        │
│    (SHA-256 变更检测)          • 自动注入上下文               │
│                                  (before_agent_start 钩子)  │
│  嵌入提供者:                                                 │
│  OpenAI | Gemini | Voyage | Local (auto fallback)          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 学习检查清单

### 基础理解（必须掌握）

- [ ] 能用一句话解释 OpenClaw 的核心定位（本地优先的多通道个人 AI 助手网关）
- [ ] 能画出三层架构图（通道层 → 控制层 → 智能层）
- [ ] 能说出 WebSocket 协议的三种帧类型（req / res / event）
- [ ] 能解释 Gateway 为什么选择中心化控制平面而非分布式
- [ ] 能说出 Memory 系统的两个插件及其区别（memory-core vs memory-lancedb）
- [ ] 能解释混合搜索的加权融合策略（vector 0.7 + text 0.3）
- [ ] 能说出会话键的编码格式和设计意图

### 深度理解（面试加分）

- [ ] 能解释 Pi Agent 嵌入模式 vs HTTP RPC 模式的 trade-off
- [ ] 能说出插件系统的完整生命周期（发现 → 清单 → 决策 → 注册 → 钩子）
- [ ] 能解释钩子系统 void 和 modifying 两种模式的区别和原因
- [ ] 能分析嵌入提供者 fallback 链的设计（auto → local → openai → gemini → voyage）
- [ ] 能说出 embedding_cache 四元组主键的设计意图
- [ ] 能对比 OpenClaw 与 LangChain Serve / AutoGen 的架构差异
- [ ] 能解释路由匹配为什么用线性扫描而不是 HashMap

### 实战能力（高级要求）

- [ ] 能设计一个新的 Memory 后端插件（如基于 Pinecone 的向量搜索）
- [ ] 能设计一个新的通道插件（如 LINE 或 Zalo）
- [ ] 能提出 Memory 系统的改进方案（三级记忆、智能遗忘、图结构记忆）
- [ ] 能分析 auto-capture 的 shouldCapture 函数的误判场景并给出改进
- [ ] 能设计"跨通道续聊"功能的架构方案

## 知识点速查表

| 章节 | 核心知识点 | 面试高频问题 |
|------|-----------|-------------|
| Ch.01 架构 | 三层架构、三帧协议、层级路由、会话键 | 为什么选中心化控制平面？路由匹配优先级设计？ |
| Ch.02 Gateway | WS 握手、RPC 方法、事件广播、慢消费者保护 | 认证体系设计？慢消费者怎么处理？ |
| Ch.03 Agent | 嵌入模式、ReAct 循环、工具注册、会话压缩 | 嵌入 vs RPC 的 trade-off？工作空间设计？ |
| Ch.04 Memory | 双层架构、混合搜索、增量索引、嵌入 fallback | 为什么双层？混合搜索权重？auto-capture 去重可靠吗？ |
| Ch.05 插件 | 插件生命周期、SDK API、钩子系统、Skills | void vs modifying 钩子？Skills vs Plugins？ |
| Ch.06 Deep Dive | 余弦相似度、worker pool、FTS 防注入、规则引擎 | 为什么手写余弦？FTS 查询安全吗？ |

## 推荐学习路径

### 新手路径

```
00-index (术语速查) → 01-architecture (全景) → 02-gateway (核心)
    → 03-agent-runtime (执行) → 04-memory (记忆)
    → 05-plugin (扩展) → 08-summary (自测)
```

### 进阶路径

```
07-interview-qa (先看问题) → 遇到不懂回溯到章节
    → 06-deep-dive (源码分析) → 08-summary (查漏补缺)
```

### 速成路径

```
00-index (术语 + 架构图) → 08-summary (检查清单 + 速查表)
    → 07-interview-qa (面试前过一遍)
```

## 项目核心数据

| 指标 | 数值 |
|------|------|
| 总文件数 | 2000+ |
| 核心源码 (src/) | 52 子目录 |
| 扩展 (extensions/) | 37 个插件 |
| 内置技能 (skills/) | 54 个 skill |
| 支持通道 | 14+ (Telegram/Discord/Slack/WhatsApp/Signal/iMessage/Teams/Matrix/Zalo/WebChat...) |
| 嵌入提供者 | 4 (OpenAI/Gemini/Voyage/Local) |
| RPC 方法 | ~40 |
| 钩子类型 | 15 |
| 技术栈 | TypeScript + Node.js 22+ + SQLite + Express + ws + Pi SDK |
| 包管理 | pnpm monorepo |
| 许可证 | MIT |
