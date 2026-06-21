# Claude Code Tool 延迟加载（ToolSearch / defer_loading）汇总

> 行号基于本仓库当前 `main` 快照，已逐一核验。
> 核心结论：**启动/挂载阶段只把工具对象造出来，并不打 deferred 标；真正的 defer 决策发生在每次 API 请求时的 `queryModel` 内**（`tools.ts:251` 注释亦明确说明）。

---

## 1. 三阶段时序总览

```
【启动】 main.tsx
  ├─ mcpTools = []                          MCP 异步连接，不阻塞
  └─ render <REPL initialTools={[]} />

【挂载】 REPL 组件
  ├─ useMergedTools(initial, mcp.tools, permCtx)              REPL.tsx:812
  │     └─ assembleToolPool → getTools → getAllBaseTools       全量内置工具（纯字面量，无 I/O）
  └─ mergedTools 仅供 UI 渲染；MCP 连上后 re-render 重算

【首次提交】 用户发第一条消息
  └─ getToolUseContext(...)                                   REPL.tsx:2391
        └─ computeTools()  ← 从 store.getState() 实时读        REPL.tsx:2403
              ├─ assembleToolPool(permCtx, state.mcp.tools)
              └─ 结果存入 toolUseContext.options.tools
                 同时 refreshTools = computeTools（轮间可重算，纳入新连上的 MCP）  REPL.tsx:2435
        ↓
      query → queryLoop → callModel → queryModel(...)          claude.ts:1018
```

> 关键：**首次请求真正使用的 tools 来自 `computeTools()` 的实时快照**，而非挂载时给 UI 的 `mergedTools`。MCP 早连上就进首请，晚连上靠 `refreshTools` 进后续轮。

---

## 2. 每次请求内：defer 判定与过滤（核心，全在 `queryModel`）

```
queryModel(messages, systemPrompt, tools, ...)                claude.ts:1018
  │
  ├─ useToolSearch = await isToolSearchEnabled(model, tools, ...)        toolSearch.ts:385
  │
  ├─ deferredToolNames = Set<已 isDeferredTool 的工具名>                 claude.ts:1130-1135
  │     ← 每请求现算（isDeferredTool 每次做 2 次 GrowthBook 查询，故预算一次）
  │
  ├─ 若 deferredToolNames 为空 且 无 pending MCP → useToolSearch=false   claude.ts:1140-1149
  │
  ├─ discoveredToolNames = extractDiscoveredToolNames(messages)         toolSearch.ts:545
  │     ← 扫历史里的 tool_reference 块，得出“已被发现”的 deferred 工具
  │
  ├─ filteredTools = tools.filter:                                      claude.ts:1161-1168
  │     • 非 deferred         → 恒保留
  │     • ToolSearchTool      → 恒保留（用于继续发现）
  │     • deferred            → 仅当 ∈ discoveredToolNames 才保留
  │   （useToolSearch=false 时：仅剔除 ToolSearchTool）                  claude.ts:1170-1172
  │
  ├─ willDefer(t) = useToolSearch && (deferredToolNames.has(t.name) || shouldDeferLspTool(t))  claude.ts:1209
  │
  ├─ toolSchemas = filteredTools.map(toolToAPISchema(t, {deferLoading: willDefer(t), tools, ...}))  claude.ts:1236
  │     ← deferLoading=true 时在 schema 上加 defer_loading:true          api.ts:224
  │     ← context 里传【完整 tools】（非 filteredTools），好让 ToolSearchTool 描述能列出全部  claude.ts:1233-1235
  │
  ├─ 若 useToolSearch：在 betas 加 tool-search beta header               claude.ts:1178-1183
  │     （Bedrock 走 extraBodyParams；1P/Foundry 用 advanced-tool-use；Vertex/Bedrock 用 tool-search-tool）
  │
  ├─ 公告模式 A（即时前插） 若 !isDeferredToolsDeltaEnabled():            claude.ts:1331-1346
  │     在 messages 头插一条 isMeta 用户消息：
  │     "<available-deferred-tools>\nCronCreate\nWebFetch\n...\n</available-deferred-tools>"
  │   公告模式 B（delta） delta 启用时改用持久化 deferred_tools_delta 附件，避免每次变动击穿缓存
  │
  └─ stream.create({ model, system, messages: messagesForAPI, tools: toolSchemas, ... })
```

---

## 3. 跨请求的发现闭环

```
第 1 次请求
  ├─ tools（发给 API）: 非 deferred 全量 schema + ToolSearch schema
  └─ 模型看到: deferred 工具仅以“名字”出现在 <available-deferred-tools>

模型调用 ToolSearch("select:WebFetch,WebSearch")
  ├─ call() 按名匹配 → matches:["WebFetch","WebSearch"]
  └─ 返回 tool_reference 块 → API 端自动把完整 schema 注入模型上下文

第 2 次请求
  ├─ extractDiscoveredToolNames(messages) → {"WebFetch","WebSearch"}
  ├─ filteredTools 现含 WebFetch/WebSearch（带 defer_loading:true）
  └─ 模型可直接调用
```

---

## 4. 关键函数清单（已核验）

| 函数 / 标识 | 位置 | 职责 |
|---|---|---|
| `getAllBaseTools()` | `tools.ts:195` | 注册全部内置工具（按 feature flag），纯内存 |
| `getTools(permCtx)` | `tools.ts:274`（arrow const） | 基于权限上下文取内置工具集 |
| `filterToolsByDenyRules()` | `tools.ts:265` | 按 deny 规则过滤 |
| `assembleToolPool(permCtx, mcpTools)` | `tools.ts:349` | 合并内置 + MCP，去重过滤，产出工具池 |
| `useMergedTools(...)` | `REPL.tsx:812` | UI 用的工具集（挂载 / re-render 重算） |
| `getToolUseContext(...)` | `REPL.tsx:2391` | 构造本轮 ToolUseContext |
| `computeTools()` / `refreshTools` | `REPL.tsx:2403 / 2435` | 从 store 实时算请求真正使用的 tools |
| `queryModel(...)` | `claude.ts:1018` | 单次 API 请求的组装与发起 |
| `isToolSearchEnabled(...)` | `toolSearch.ts:385` | 是否启用 ToolSearch 模式（tst / tst-auto） |
| `isDeferredTool(t)` | `tools/ToolSearchTool/prompt.ts`（被 claude.ts 导入） | 判定单个工具是否 deferred |
| `extractDiscoveredToolNames(msgs)` | `toolSearch.ts:545` | 从历史 tool_reference 提取已发现工具名 |
| `toolToAPISchema(t, {deferLoading})` | `utils/api.ts:119`（`defer_loading` 设于 `:224`） | Tool → BetaTool JSON schema |
| `<available-deferred-tools>` 注入 | `claude.ts:1331-1346` | 非 delta 模式下的 deferred 名单公告 |

---

## 5. 哪些工具算 deferred

deferred 集合 = **所有 MCP 工具** + **`shouldDefer: true` 的内置工具**（+ 满足 `shouldDeferLspTool` 的 LSP 工具）。

`shouldDefer` 是 `Tool` 上的可选属性（`Tool.ts:445`）。被标记的内置工具包括：
`EnterPlanMode`、`EnterWorktree`、`CronCreate`、`SendMessage`、`Task*`（Create / Update / Get / List / Output / Stop）、`ListMcpResources`、`ReadMcpResource`、`RemoteTrigger`、`NotebookEdit`、`TeamDelete` 等。

> 订正：deferred 的“标记”动作发生在**每次请求**（`claude.ts:1130`），不在启动 / 挂载阶段。启动期只把工具对象造出来，不打 deferred 标。

---

## 6. 两种 deferred 公告模式

| 模式 | 触发 | 机制 | 代价 |
|---|---|---|---|
| **A 即时前插** | `!isDeferredToolsDeltaEnabled()` | 每请求在 messages 头插一条 `<available-deferred-tools>` 临时 meta 消息 | 工具池一变就击穿 prompt cache |
| **B delta 附件** | delta 启用 | 用持久化的 `deferred_tools_delta` 附件公告 | 避免击穿缓存 |

---

## 一句话总结

启动只造工具不打标；每次请求才在 `queryModel` 里现算 deferred 集、按“历史是否发现过”过滤出 `filteredTools` 发给 API、给 deferred 的 schema 打 `defer_loading:true`，并通过 `<available-deferred-tools>`（或 delta 附件）只把名字告诉模型；模型用 ToolSearch 选中后经 `tool_reference` 展开完整 schema，下一请求即可直调。
