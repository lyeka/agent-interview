# 插件与扩展架构

OpenClaw 的插件系统是整个平台的扩展骨架。通道（Telegram、Discord）、Memory 后端、LLM 提供者、CLI 子命令——这些都不是硬编码的，而是通过统一的插件接口动态加载。理解插件系统就是理解 OpenClaw 如何在保持核心精简的同时支持丰富的功能。

## 插件系统全景

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Plugin Lifecycle                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Discovery (发现)                                                  │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ discoverOpenClawPlugins()                                 │        │
│  │  → extraPaths → workspace/.openclaw/extensions            │        │
│  │  → ~/.openclaw/extensions → bundled extensions            │        │
│  └─────────────────────────┬────────────────────────────────┘        │
│                            ▼                                          │
│  2. Manifest Loading (清单加载)                                       │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ loadPluginManifestRegistry()                              │        │
│  │  → 读取 openclaw.plugin.json → 校验 schema               │        │
│  └─────────────────────────┬────────────────────────────────┘        │
│                            ▼                                          │
│  3. Enable/Disable (启停决策)                                         │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ resolveEnableState()                                      │        │
│  │  → 全局开关 → 黑名单 → 白名单 → 槽位 → 配置 → 默认规则    │        │
│  └─────────────────────────┬────────────────────────────────┘        │
│                            ▼                                          │
│  4. Registration (注册)                                               │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ jiti(candidate.source) → register(api)                    │        │
│  │  → registerTool / registerChannel / registerHook / ...    │        │
│  └─────────────────────────┬────────────────────────────────┘        │
│                            ▼                                          │
│  5. Hook Runner (钩子运行器)                                          │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │ initializeGlobalHookRunner(registry)                      │        │
│  │  → 所有已注册钩子按 priority 排序 → 运行时调用              │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## 插件发现（Discovery）

插件发现（`src/plugins/discovery.ts:301-361`）按以下优先级扫描目录：

```
1. plugins.load.paths  — 用户配置的额外路径
2. {workspace}/.openclaw/extensions — 工作空间级
3. ~/.openclaw/extensions — 全局级
4. bundled extensions — 内置（随 npm 包分发）
```

每个目录下的 `.ts`/`.js` 文件或含 `index.ts`/`index.js` 的子目录都是候选插件。发现过程使用 `seen` 集合去重，确保同一插件不会被加载两次。

## 插件清单（Plugin Manifest）

每个插件必须有 `openclaw.plugin.json`：

```json
{
  "id": "memory-core",
  "kind": "memory",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

清单的类型定义（`src/plugins/manifest.ts:9-21`）：

```typescript
type PluginManifest = {
  id: string;                    // 唯一标识
  kind?: PluginKind;             // "memory" 等
  channels?: string[];           // 提供的通道列表
  providers?: string[];          // 提供的 LLM 提供者
  skills?: string[];             // 提供的 skill 目录
  configSchema: Record<string, unknown>;  // JSON Schema
  name?: string;
  description?: string;
  version?: string;
};
```

`kind` 字段触发槽位机制。目前只有 `"memory"` 一种 kind，意味着只能有一个 Memory 插件激活。

## 启停决策

插件的启用/禁用逻辑（`src/plugins/config-state.ts:163-193`）是一个精确的决策链：

```typescript
// 决策优先级（从高到低）
1. 全局开关关闭 → 禁用
2. 黑名单包含 → 禁用
3. 白名单存在但不包含 → 禁用
4. 是当前 Memory 槽位 → 启用
5. 配置显式 enabled: true → 启用
6. 配置显式 enabled: false → 禁用
7. 内置且默认启用 → 启用
8. 内置但非默认 → 禁用
9. 非内置 → 启用（第三方插件默认启用）
```

这个设计的核心思想是：安全优先。内置插件默认关闭（除非在白名单中），第三方插件默认开启（假设用户主动安装意味着想使用）。

## Plugin SDK API

插件通过 `register(api)` 函数接收一个 API 对象（`src/plugins/types.ts:243-281`），提供以下注册能力：

| 注册方法 | 用途 | 示例 |
|---------|------|------|
| `registerTool` | 注册 Agent 工具 | memory_search |
| `registerChannel` | 注册消息通道 | Telegram |
| `registerHook` | 注册生命周期钩子 | before_agent_start |
| `registerHttpHandler` | 注册 HTTP 处理器 | Webhook |
| `registerHttpRoute` | 注册 HTTP 路由 | `/api/custom` |
| `registerGatewayMethod` | 注册 RPC 方法 | `custom.method` |
| `registerCli` | 注册 CLI 子命令 | `openclaw memory` |
| `registerService` | 注册后台服务 | start/stop 生命周期 |
| `registerProvider` | 注册 LLM 提供者 | Google Antigravity |
| `registerCommand` | 注册聊天命令 | `/custom` |

API 对象还提供了 `runtime` 属性，这是一个大型 facade，暴露了 OpenClaw 内部子系统的能力：配置管理、媒体处理、TTS、工具创建、通道操作等。

## 钩子系统（Hooks）

钩子是插件参与 Agent 生命周期的主要方式。当前支持的钩子（`src/plugins/types.ts:292-306`）：

```typescript
type PluginHookName =
  | "before_agent_start"    // Agent 处理前（可注入上下文）
  | "agent_end"             // Agent 处理后（可捕获记忆）
  | "before_compaction"     // 上下文压缩前
  | "after_compaction"      // 上下文压缩后
  | "message_received"      // 收到通道消息
  | "message_sending"       // 消息发送前（可修改内容）
  | "message_sent"          // 消息发送后
  | "before_tool_call"      // 工具调用前（可修改参数）
  | "after_tool_call"       // 工具调用后（可修改结果）
  | "tool_result_persist"   // 工具结果持久化（同步）
  | "session_start"         // 会话开始
  | "session_end"           // 会话结束
  | "gateway_start"         // Gateway 启动
  | "gateway_stop";         // Gateway 停止
```

钩子的执行模式分为两种：

**Void 钩子**（并行、fire-and-forget）：`agent_end`, `message_received`, `message_sent`
- 所有处理器并发执行
- 一个失败不影响其他

**Modifying 钩子**（串行、可修改）：`before_agent_start`, `message_sending`, `before_tool_call`
- 按 priority 顺序串行执行
- 后一个处理器可以看到前一个的修改结果
- 返回值会被合并到事件上下文中

这个区分的设计意图很清晰：观察性操作可以并行（性能），修改性操作必须串行（一致性）。

## 模块加载

插件的 TypeScript 模块通过 `jiti` 运行时加载（`src/plugins/loader.ts:168-341`）：

```typescript
const jiti = createJiti(import.meta.url, {
  interopDefault: true,
  extensions: [".ts", ".tsx", ".mts", ".cts"],
  alias: {
    "openclaw/plugin-sdk": pluginSdkAlias,
  },
});
// 加载插件模块
mod = jiti(candidate.source);
```

`jiti` 是一个运行时 TypeScript 转译器。关键是 `alias` 配置——它将 `openclaw/plugin-sdk` 映射到实际的 SDK 路径，使得插件可以用标准的 import 语法引用 SDK，而不需要知道 SDK 的真实位置。

插件模块可以导出：
- 一个函数（作为 `register`）
- 一个对象（包含 `register` 或 `activate` 方法）

## Skills vs Plugins

Skills 和 Plugins 是两个完全不同层次的扩展机制：

| 维度 | Skills | Plugins |
|------|--------|---------|
| 格式 | Markdown (SKILL.md) + YAML | TypeScript + JSON manifest |
| 运行层 | Prompt 层（指令注入） | Runtime 层（代码执行） |
| 能力 | 指导 Agent 行为 | 注册工具、通道、钩子、服务 |
| 安全 | 沙箱内（Agent 权限） | 主进程（完全权限） |
| 加载 | 按需加载到 Context | 启动时全部注册 |
| 创建门槛 | 写 Markdown | 写 TypeScript |

Skills 是给"Agent 使用的说明书"，Plugins 是给"系统使用的扩展代码"。一个 Skill 告诉 Agent"当用户问天气时，调用 weather API"；一个 Plugin 在系统中注册了一个名为 `weather` 的工具。

Skills 的加载路径：

```
1. bundled skills/          — 内置 skill
2. ~/.openclaw/skills       — 全局 skill
3. {workspace}/skills       — 工作空间 skill
4. plugins.skills[]         — 插件附带的 skill
```

## 安全模型

### 预安装扫描

安装第三方插件前，OpenClaw 会扫描代码中的危险模式（`src/security/skill-scanner.ts:79-137`）：

```typescript
const LINE_RULES = [
  { ruleId: "dangerous-exec", severity: "critical",
    pattern: /\b(exec|execSync|spawn)\s*\(/,
    requiresContext: /child_process/ },
  { ruleId: "dynamic-code-execution", severity: "critical",
    pattern: /\beval\s*\(|new\s+Function\s*\(/ },
  { ruleId: "crypto-mining", severity: "critical", ... },
  { ruleId: "env-harvesting", severity: "critical",
    message: "Environment variable access combined with network send" },
];
```

扫描结果是建议性的（advisory）——发现问题会警告但不阻止安装。这是一个务实的权衡：严格阻止可能导致合法插件无法安装，而用户已经通过安装行为表达了信任。

### 路径安全

插件安装时的路径验证（`src/plugins/install.ts:74-80`）：

```typescript
function isPathInside(basePath: string, candidatePath: string): boolean {
  const rel = path.relative(base, candidate);
  return rel === "" || (!rel.startsWith(`..${path.sep}`)
    && rel !== ".." && !path.isAbsolute(rel));
}
```

确保插件文件不能逃逸出安装目录，防止路径穿越攻击。

### 工具白名单

群组/非主会话中的插件工具受白名单控制：

```typescript
// src/plugins/tools.ts:24-39
function isOptionalToolAllowed(params) {
  if (params.allowlist.has(toolName)) return true;
  if (params.allowlist.has(pluginKey)) return true;
  return params.allowlist.has("group:plugins");
}
```

这意味着在群组聊天中，即使插件注册了工具，也需要管理员显式批准才能使用。

## 实际插件剖析

### memory-core 插件

最简洁的内置插件示例：

```typescript
// extensions/memory-core/index.ts:4-38
const memoryCorePlugin = {
  id: "memory-core",
  name: "Memory (Core)",
  kind: "memory",
  register(api) {
    // 注册两个工具
    api.registerTool((ctx) => {
      return [
        api.runtime.tools.createMemorySearchTool({...}),
        api.runtime.tools.createMemoryGetTool({...}),
      ];
    }, { names: ["memory_search", "memory_get"] });
    // 注册 CLI 子命令
    api.registerCli(({ program }) => {
      api.runtime.tools.registerMemoryCli(program);
    }, { commands: ["memory"] });
  },
};
```

注意它使用了 `api.runtime.tools.createMemorySearchTool` 而不是自己实现搜索逻辑。核心实现在 `src/memory/` 中，插件只是将它暴露为 Agent 工具。

### telegram 插件

通道类插件的典型结构：

```typescript
// extensions/telegram/index.ts:6-17
const plugin = {
  id: "telegram",
  name: "Telegram",
  register(api) {
    setTelegramRuntime(api.runtime);
    api.registerChannel({
      plugin: telegramPlugin as ChannelPlugin,
    });
  },
};
```

通道插件通过 `registerChannel` 注册一个 `ChannelPlugin` 实现，Gateway 的 ChannelManager 会调用其 `startAccount`/`stopAccount` 方法管理生命周期。

---

## 章节衔接

**本章回顾**：
- 我们学习了 OpenClaw 的完整插件生命周期（发现 → 清单 → 决策 → 注册 → 运行）
- 关键收获：Skills 和 Plugins 是两个不同层次的扩展，钩子系统区分了观察性和修改性操作

**下一章预告**：
- 在 `06-deep-dive.md` 中，我们将对核心源码进行深度解析
- 为什么需要学习：前面的章节是架构视角，Deep Dive 是实现视角——面试官会问"这行代码为什么这样写"
- 关键问题：关键算法的实现细节、设计权衡、代码质量
