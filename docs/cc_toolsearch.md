# ToolSearch / Deferred Tools 机制分析

> 分析对象：Claude Code 的 ToolSearch（动态工具加载）系统。
> 核心问题：哪些工具会被"延迟加载"、ToolSearch 何时启用、以及第三方代理（如 cloudbash）为何会导致工具定义被一次性全量加载、token 浪费。

相关源文件：

- `src/tools/ToolSearchTool/prompt.ts` — `isDeferredTool()` 判定 + ToolSearchTool 的 description 文本
- `src/tools/ToolSearchTool/ToolSearchTool.ts` — 工具实现 + `isEnabled()`
- `src/utils/toolSearch.ts` — `isToolSearchEnabled()` / `isToolSearchEnabledOptimistic()` / 模式与阈值
- `src/services/api/claude.ts` — 把工具 schema 拼进 API 请求（动态过滤、defer_loading）
- `src/utils/model/providers.ts` — `isFirstPartyAnthropicBaseUrl()`

---

## 1. 核心概念：什么是 Deferred Tool

Deferred tool（延迟加载工具）= 初始 prompt 里**只给名字、不给完整 schema** 的工具。模型需要时用 `ToolSearchTool` 搜索，才会拿到完整 JSONSchema 并变得可调用。目的是省 token——不必把所有工具（尤其大量 MCP 工具）的完整定义一次性塞进上下文。

### 判定函数 `isDeferredTool()`（prompt.ts:62）

按优先级**短路**判断：

| 优先级 | 条件 | 是否 deferred |
|---|---|---|
| ① | `tool.alwaysLoad === true` | ❌ 否（最高优先，MCP 也能借此 opt-out）|
| ② | `tool.isMcp === true` | ✅ 是（MCP 工具默认全部 deferred）|
| ③ | `tool.name === ToolSearchTool` | ❌ 否（否则没法加载别的工具）|
| ④ | Agent 工具（`FORK_SUBAGENT` 开启时）| ❌ 否（第一轮就要可用）|
| ⑤ | Brief 工具（KAIROS）| ❌ 否（主通信通道）|
| ⑥ | SendUserFile 工具（KAIROS + ReplBridge）| ❌ 否（文件投递通道）|
| ⑦ | 其它 | `tool.shouldDefer === true` |

`alwaysLoad === true`（MCP 工具通过 `_meta['anthropic/alwaysLoad']` 设置）最先检查，因此 MCP 工具可以显式 opt-out 延迟加载。

---

## 2. 两个判定函数的分工

ToolSearch 是否启用由**两个**函数判定，职责不同：

| 函数 | 含 Base URL 检测 | 用途 |
|---|---|---|
| `isToolSearchEnabledOptimistic()` | ✅ **有** | 乐观检查，决定**ToolSearchTool 是否进工具池**、是否保留 tool_reference 字段。ToolSearchTool 的 `isEnabled()` 直接调它 |
| `isToolSearchEnabled()` | ❌ 无 | 权威检查（模型支持 / 工具可用 / 阈值），发 API 调用时用 |

> ⚠️ 关键：第三方代理那道"关闭 ToolSearch"的闸在**乐观版**里。它在更早阶段就把 ToolSearchTool 挡在工具池外，所以权威版 `isToolSearchEnabled()` 根本轮不到判断。只看 `isToolSearchEnabled()` 会找不到这段逻辑。

---

## 3. `isToolSearchEnabledOptimistic()` 逻辑（toolSearch.ts:270）

```ts
export function isToolSearchEnabledOptimistic(): boolean {
  const mode = getToolSearchMode()
  if (mode === 'standard') return false

  // 第三方代理 gate
  if (
    !process.env.ENABLE_TOOL_SEARCH &&        // ① 用户没显式配置
    getAPIProvider() === 'firstParty' &&      // ② provider 仍是 Anthropic 直连
    !isFirstPartyAnthropicBaseUrl()           // ③ 但 base URL 不是官方域名
  ) {
    return false
  }
  return true
}
```

### 第三方代理 gate（toolSearch.ts:299-311）—— **本文重点**

三个条件**同时满足**才关闭 ToolSearch：

1. `!process.env.ENABLE_TOOL_SEARCH` — 用户未显式设置（未设 / 空字符串都算）
2. `getAPIProvider() === 'firstParty'` — 不是 Vertex/Bedrock/Foundry（它们有自己的 endpoint 和 beta header，不受影响）
3. `!isFirstPartyAnthropicBaseUrl()` — `ANTHROPIC_BASE_URL` 指向的不是 Anthropic 官方主机

**原因**：`tool_reference` 是 beta content block，第三方代理网关通常不转发/不支持它，会用 **400 错误**拒绝请求。所以检测到"自称 firstParty 但 base URL 被改到非官方域名"时，保守地默认关闭 ToolSearch，避免直接失败。
对应 issue：`anthropics/claude-code#30912`。

**重要反转**（注释 toolSearch.ts:289-298）：有些代理其实支持 tool_reference（LiteLLM passthrough、Cloudflare AI Gateway、转发 beta header 的企业网关）。一刀切关闭反而坑了这些用户（gh-31936 / CC-457，被认为是 CC-330 "v2.1.70 defer_loading 回归"的真正原因）。因此这个 gate **只在 `ENABLE_TOOL_SEARCH` 未设时生效**——显式设值即视为用户断言"我的代理支持"。

### `isFirstPartyAnthropicBaseUrl()`（providers.ts:25）

```ts
export function isFirstPartyAnthropicBaseUrl(): boolean {
  const baseUrl = process.env.ANTHROPIC_BASE_URL
  if (!baseUrl) return true                         // 没设 → 视为官方
  const host = new URL(baseUrl).host
  const allowedHosts = ['api.anthropic.com']
  if (process.env.USER_TYPE === 'ant') {
    allowedHosts.push('api-staging.anthropic.com')  // 内部 staging
  }
  return allowedHosts.includes(host)                // 解析失败 → false
}
```

官方域名白名单只有 `api.anthropic.com`（ant 内部多一个 staging）。任何其它 host 都算"非官方"。

---

## 4. `isToolSearchEnabled()` 权威逻辑（toolSearch.ts:385）

从上往下执行，**遇到第一个满足的条件就返回**。共 5 个返回点：

```
① modelSupportsToolReference(model)?     ── 否 → false  (model_unsupported)
② isToolSearchToolAvailable(tools)?       ── 否 → false  (mcp_search_unavailable)
③ switch(getToolSearchMode()):
     'tst'      → true                     (tst_enabled)
     'tst-auto' → checkAutoThreshold(...)  (auto_above/below_threshold)
     'standard' → false                    (standard_mode)
```

### 返回 `false` 的 4 种情况

1. 模型是 Haiku（或命中不支持列表）—— 见第 5 节
2. ToolSearchTool 被禁用（如 `--disallowedTools`），不在工具池里
3. mode == `standard`（`ENABLE_TOOL_SEARCH=false` / `auto:100` / kill switch）
4. `tst-auto` 模式且 deferred 工具体量 **没超过阈值**

### 返回 `true` 的 2 种情况（前提：通过 ① ②）

1. mode == `tst`（`ENABLE_TOOL_SEARCH=true` / `auto:0` / **未设（默认）**）
2. `tst-auto` 模式且 deferred 工具体量 **超过阈值**

### 模式由环境变量决定 `getToolSearchMode()`（toolSearch.ts:172）

| `ENABLE_TOOL_SEARCH` | mode | 结果（模型支持 + 工具在池）|
|---|---|---|
| **未设（默认）** | `tst` | true |
| `true` / `auto:0` | `tst` | true |
| `auto` / `auto:1`~`99` | `tst-auto` | 看阈值 |
| `false` / `auto:100` | `standard` | false |
| `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1` | `standard` | false（覆盖一切）|

### `tst-auto` 阈值判定 `checkAutoThreshold()`（toolSearch.ts:712）

1. **优先精确 token 数**：`getDeferredToolTokenCount()` 调 token 计数 API（缓存，工具集变化才重算），与 `getAutoToolSearchTokenThreshold(model)` 比较，`>=` 阈值 → enabled
2. **降级字符启发式**：token API 不可用时，用 deferred 工具描述总字符数与 char 阈值比较

意图：deferred 工具体量小时不值得引入 ToolSearch 这层间接，直接内联；超过"上下文的 N%"才启用。

---

## 5. `modelSupportsToolReference()` 逻辑（toolSearch.ts:239）

```ts
export function modelSupportsToolReference(model: string): boolean {
  const normalizedModel = model.toLowerCase()
  for (const pattern of getUnsupportedToolReferencePatterns()) {
    if (normalizedModel.includes(pattern.toLowerCase())) return false
  }
  return true
}
```

**负向判定（默认支持，黑名单排除）**：不是白名单，而是"黑名单里有才不支持，否则一律支持"。`includes()` 子串匹配，所以 `'haiku'` 命中所有含 haiku 的变体。

黑名单来源 `getUnsupportedToolReferencePatterns()`（toolSearch.ts:210），两级优先：

1. **GrowthBook 远程优先**：feature `tengu_tool_search_unsupported_models`（string[]），非空则用。Anthropic 可不发版动态调整黑名单
2. **本地默认兜底**：读失败/空 → `DEFAULT_UNSUPPORTED_MODEL_PATTERNS = ['haiku']`

**设计意图**（注释 226-238）：新模型默认假定支持 tool_reference，发布新模型无需改代码即自动启用 ToolSearch；只有已知不支持的（目前仅 Haiku）才显式列黑名单。这是它作为 `isToolSearchEnabled` 第一道门的根因——Haiku 子 agent 直接禁用 ToolSearch。

| model | 含黑名单 pattern | 返回 |
|---|---|---|
| `claude-opus-4-8` | 否 | true |
| `claude-sonnet-4-6` | 否 | true |
| `claude-haiku-4-5` | 含 `haiku` | false |
| 任意未来/第三方模型 | 一般不含 | true（乐观）|

---

## 6. 工具 schema 如何拼进 API 请求（claude.ts）

`tools: allTools`（claude.ts:1712）**每次请求都发送**。`useToolSearch` 不决定"发不发 tools 字段"，而是决定 `allTools` 里**装什么**。

### 第 1 步：算 `useToolSearch`（claude.ts:1121）

```ts
let useToolSearch = await isToolSearchEnabled(model, tools, ...)

// 二次兜底：模式开着但没 deferred 工具、也没 pending MCP → 关掉
if (useToolSearch && deferredToolNames.size === 0 && !hasPendingMcpServers) {
  useToolSearch = false
}
```

### 第 2 步：用 `useToolSearch` 决定 `filteredTools`（claude.ts:1155）

```ts
if (useToolSearch) {
  const discoveredToolNames = extractDiscoveredToolNames(messages)
  filteredTools = tools.filter(tool => {
    if (!deferredToolNames.has(tool.name)) return true            // 非 deferred → 全发
    if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true // ToolSearchTool → 永远发
    return discoveredToolNames.has(tool.name)                     // deferred → 只发已发现的
  })
} else {
  filteredTools = tools.filter(t => !toolMatchesName(t, TOOL_SEARCH_TOOL_NAME)) // 剔除 ToolSearchTool，其余全发
}
```

### 两种场景下 `allTools` 的差异

| | `useToolSearch = true` | `useToolSearch = false` |
|---|---|---|
| ToolSearchTool | 包含（带 `getPrompt()` 描述）| 剔除 |
| deferred 工具 | 只含**已被 ToolSearch 发现**的 | 全部内联 |
| 未发现的 deferred 工具 | 不发 schema（按需加载）| 全部内联发送 |
| defer_loading 标记 | `willDefer(tool)` 打标 | 不打 |
| 非 deferred 工具 | 全发 | 全发 |

随后 `toolToAPISchema`（claude.ts:1236）转成 schema → `allTools = [...toolSchemas, ...extraToolSchemas]`（claude.ts:1397）→ `tools: allTools`（claude.ts:1712）。

> 注：`toolToAPISchema` 有 schema 缓存（api.ts:147），ToolSearchTool 的 `getPrompt()` 实际每会话只执行一次。

---

## 7. 第三方代理（cloudbash 等）问题完整链路

抓包发现的现象——用反向代理时所有工具定义被一次性加载、token 浪费——的完整成因：

```
cloudbash 改了 ANTHROPIC_BASE_URL → 指向非官方域名
        │
        ▼
用户未设 ENABLE_TOOL_SEARCH + provider=firstParty + 非官方 baseUrl
        │  三条件命中（toolSearch.ts:299）
        ▼
isToolSearchEnabledOptimistic() → false
        │
        ▼
ToolSearchTool.isEnabled() → false（ToolSearchTool.ts:305）
        │
        ▼
ToolSearchTool 根本不进工具池
        │
        ▼
useToolSearch = false → 所有 MCP/deferred 工具完整 schema 一次性内联进 tools 参数
        │
        ▼
        token 浪费 ❌
```

### 解决方案

显式设置 **`ENABLE_TOOL_SEARCH=true`**（或 `auto` / `auto:N`）：

- 第 ① 个条件 `!process.env.ENABLE_TOOL_SEARCH` 变为 false → 整个代理 gate 被跳过 → ToolSearch 恢复启用
- 这等于向 CC 断言"我的代理支持 tool_reference beta block"
- 错误日志里也直接给出该提示（toolSearch.ts:307）：
  > `Set ENABLE_TOOL_SEARCH=true (or auto / auto:N) if your proxy forwards tool_reference blocks.`

**前提**：代理（cloudbash）必须确实转发 tool_reference beta content block，否则 API 会返回 400。若代理不支持，则维持默认关闭才是正确行为。

---

## 8. 速查总结

- **Deferred tool** = 只给名字、按需经 ToolSearch 加载 schema 的工具；MCP 工具默认全是
- **两个判定函数**：`...Optimistic()`（含代理 gate，决定进不进工具池）/ `isToolSearchEnabled()`（权威，含模型/阈值）
- **默认 mode 是 `tst`**（`ENABLE_TOOL_SEARCH` 未设即启用）
- **Haiku 模型**直接禁用（`modelSupportsToolReference` 黑名单）
- **第三方代理**默认关闭 ToolSearch（除非显式 `ENABLE_TOOL_SEARCH=true`）—— 这是反向代理 token 浪费的根因
- 即使权威函数返回 true，**没有任何 deferred 工具**时外层（claude.ts:1140）仍会关闭
