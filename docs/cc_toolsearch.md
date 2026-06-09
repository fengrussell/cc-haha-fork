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

## 8. ToolSearchTool 实现分析（ToolSearchTool.ts）

ToolSearch 机制的"执行端"——模型用它把 deferred 工具的名字换成可调用的完整 schema。
文件做四件事：定义 I/O schema → 关键字搜索算法 → 工具定义 → 把结果包装成 `tool_reference` 块。

### 8.1 输入 / 输出 schema

**输入**（inputSchema，ToolSearchTool.ts:21）：
```ts
{ query: string, max_results?: number = 5 }
```
- `query` 两种形态：`select:<tool_name>`（直接选择，逗号分隔多选）或关键字搜索
- `lazySchema()` 包裹，延迟构造 zod schema

**输出**（outputSchema，ToolSearchTool.ts:37）：
```ts
{ matches: string[], query: string, total_deferred_tools: number, pending_mcp_servers?: string[] }
```
`matches` 只是工具名数组；真正的 schema 展开在 `mapToolResultToToolResultBlockParam` 里通过 `tool_reference` 完成。

### 8.2 关键字搜索算法（searchToolsWithKeywords，ToolSearchTool.ts:186）

分层匹配 + 加权打分：

1. **精确名匹配**（199）：query 精确等于某工具名（先 deferred 后全量）→ 直接返回。处理裸工具名（子 agent/压缩后常见）
2. **MCP 前缀匹配**（208）：query 以 `mcp__server` 开头 → 前缀匹配
3. **必需/可选拆分**（220）：`+slack` 为必需项（AND 预过滤），其余为可选项（打分）
4. **加权打分**（259-295）：

   | 匹配位置 | 普通工具 | MCP 工具 |
   |---|---|---|
   | 工具名 part 精确命中 | +10 | +12 |
   | 工具名 part 子串命中 | +5 | +6 |
   | 全名子串（score 仍为 0）| +3 | +3 |
   | `searchHint` 命中（词边界）| +4 | +4 |
   | description 命中（词边界）| +2 | +2 |

   MCP 权重略高（server/action 信号更明确）。按分数降序取 `maxResults`。

**性能优化**：
- `compileTermPatterns`（167）：预编译词边界正则，每次搜索一次而非 `工具数×词数×2` 次
- `getToolDescriptionMemoized`（66）：按工具名记忆化 description（打分要读，来自 `tool.prompt()` 有成本）
- `parseToolName`（132）：`mcp__slack__send_message` → `[slack,send,message]`；CamelCase `ReadFile` → `[read,file]`

### 8.3 缓存失效（maybeInvalidateCache，ToolSearchTool.ts:91）

description 记忆化缓存 key = 当前 deferred 工具名排序拼接。MCP server 连接/断开导致 deferred 集变化 → key 变化 → 清空缓存，保证打分用最新描述。

### 8.4 工具定义（buildTool，ToolSearchTool.ts:304）

| 字段 | 值 | 含义 |
|---|---|---|
| `isEnabled()` | `isToolSearchEnabledOptimistic()` | **进不进工具池的总开关**（含代理 gate）|
| `isConcurrencySafe()` | true | 可并行 |
| `isReadOnly()` | true | 只读，无需权限确认 |
| `maxResultSizeChars` | 100_000 | 结果上限 |
| `description()`/`prompt()` | `getPrompt()` | 给人看/给模型看 |
| `renderToolUseMessage()` | null | TUI 不渲染 |
| `userFacingName()` | `''` | 隐形工具 |

### 8.5 call() 执行逻辑（ToolSearchTool.ts:328）

```
1. deferredTools = tools.filter(isDeferredTool)   // 只在 deferred 集里搜
2. maybeInvalidateCache(deferredTools)
3. 匹配 select: 前缀?
     是 → 逐个 findToolByName(deferred → 全量兜底)；found 为空→空+pending提示；否则返回 found
     否 → searchToolsWithKeywords()
4. 无匹配 → 附带 pending MCP server 信息返回
```

两个鲁棒性设计：
- **already-loaded 兜底**（360-362）：select/精确匹配时，工具不在 deferred 集但在全量集（已加载）→ 仍返回，无害 no-op，避免 retry
- **pending MCP 提示**（335）：搜不到时若有正在连接的 MCP server，提示"稍后再搜"

### 8.6 核心输出转换（mapToolResultToToolResultBlockParam，ToolSearchTool.ts:444）

**ToolSearch 的魔法所在**——把工具名数组转成 `tool_reference` 块：
```ts
content: content.matches.map(name => ({ type: 'tool_reference', tool_name: name }))
```
- `tool_reference` 是 beta content block，API 收到后把对应工具的**完整 schema 动态注入**模型可用工具集——这就是 deferred 工具"被发现"后变可调用的机制
- 无匹配 → 纯文本 `No matching deferred tools found`（+ pending 提示）
- 仅 1P/Foundry 可用；Bedrock/Vertex 可能不支持 client-side 展开
- ⚠️ 正是这个 beta 块导致第三方代理默认关闭 ToolSearch（代理不转发会 400），与第 7 节 cloudbash 问题闭环

### 8.7 数据流闭环

```
模型看到 deferred 工具名 (system-reminder / available-deferred-tools)
   ▼ 调用 ToolSearchTool(query="slack send" 或 "select:mcp__slack__send")
call() 搜索/select → matches: string[]
   ▼ mapToolResultToToolResultBlockParam()
matches → [{type:'tool_reference', tool_name}] 返回 API
   ▼ claude.ts extractDiscoveredToolNames 从消息历史提取
discoveredToolNames 加入这些名字
   ▼ 下次请求 filteredTools 过滤 (claude.ts:1167)
这些 deferred 工具完整 schema 进入 tools 参数 → 可调用 ✅
```

---

## 9. 速查总结

- **Deferred tool** = 只给名字、按需经 ToolSearch 加载 schema 的工具；MCP 工具默认全是
- **两个判定函数**：`...Optimistic()`（含代理 gate，决定进不进工具池）/ `isToolSearchEnabled()`（权威，含模型/阈值）
- **默认 mode 是 `tst`**（`ENABLE_TOOL_SEARCH` 未设即启用）
- **Haiku 模型**直接禁用（`modelSupportsToolReference` 黑名单）
- **第三方代理**默认关闭 ToolSearch（除非显式 `ENABLE_TOOL_SEARCH=true`）—— 这是反向代理 token 浪费的根因
- 即使权威函数返回 true，**没有任何 deferred 工具**时外层（claude.ts:1140）仍会关闭
