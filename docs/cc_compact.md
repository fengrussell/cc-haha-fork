# Compact 压缩系统分析

## 一、microCompact vs autoCompact 对比

| 维度 | microCompact | autoCompact |
|---|---|---|
| **文件** | `src/services/compact/microCompact.ts` | `src/services/compact/autoCompact.ts` |
| **粒度** | 单条工具结果级别 | 整个对话级别 |
| **操作** | 清除/删除旧的工具结果内容 | 调用 LLM 生成整个对话的摘要替换全部历史 |
| **成本** | 零 LLM 调用（本地操作或 cache_edits API） | 需要一次完整的 LLM 调用 |
| **触发位置** | queryLoop 第 416 行（autoCompact 之前） | queryLoop 第 456 行（microCompact 之后） |
| **目标工具** | 仅 8 种：Read/Write/Edit/Bash/Grep/Glob/WebFetch/WebSearch | 所有消息 |
| **保留策略** | 保留最近 N 条工具结果，清除更早的 | 压缩为一条摘要消息 |
| **对缓存影响** | cached 路径通过 cache_edits 保留 prompt cache | 完全破坏 prompt cache |

**设计意图**：microCompact 是"修剪树枝"（清理单条工具结果），autoCompact 是"砍树重栽"（摘要替换整个历史）。系统优先用轻量的 microCompact，不够再用重量级的 autoCompact。

## 二、queryLoop 中的预处理管线调用顺序

```
messagesForQuery
  │
  ├─1→ applyToolResultBudget   [query.ts:381]  裁剪超大工具结果
  ├─2→ snipCompact             [query.ts:403]  删除旧历史消息
  ├─3→ microcompact ★          [query.ts:416]  清除旧工具结果内容
  ├─4→ contextCollapse         [query.ts:443]  折叠上下文段
  └─5→ autoCompact ★           [query.ts:456]  整体摘要压缩
```

microCompact 在 autoCompact 之前运行，轻量清理后可能使 token 数降到 autoCompact 阈值以下，从而避免昂贵的 LLM 摘要调用。

---

## 三、microCompact 详解

### 函数列表

| 函数 | 行号 | 导出 | 用途 |
|---|---|---|---|
| `microcompactMessages` | 256 | yes | 入口函数，分发到三条路径 |
| `maybeTimeBasedMicrocompact` | 452 | no | 时间触发路径 |
| `cachedMicrocompactPath` | 309 | no | 缓存编辑路径 |
| `evaluateTimeBasedTrigger` | 427 | yes | 时间触发判断（供外部复用） |
| `collectCompactableToolIds` | 228 | no | 收集可压缩的工具 ID |
| `calculateToolResultTokens` | 138 | no | 计算工具结果 token 数 |
| `estimateMessageTokens` | 165 | yes | 估算消息 token 总数 |

### 三条内部路径

```
microcompactMessages() [256行, 入口]
  │
  ├─1→ maybeTimeBasedMicrocompact()     时间触发：距上次回复超过阈值(缓存已冷)
  │    └── 直接修改消息内容为 "[Old tool result content cleared]"
  │
  ├─2→ cachedMicrocompactPath()         缓存编辑：通过 cache_edits API 删除(不改本地消息)
  │    └── 仅 ant 用户 + 支持的模型 + 主线程
  │
  └─3→ return { messages }              不压缩：交给后续 autoCompact 处理
```

### 可压缩工具列表（COMPACTABLE_TOOLS）

FileRead, Bash(shell), Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite

### 完整函数列表

| 函数 | 行号 | 导出 | 用途 |
|---|---|---|---|
| `calculateToolResultTokens` | 138 | no | 计算单个 tool_result 的 token 数 |
| `estimateMessageTokens` | 165 | yes | 估算消息列表总 token（含 4/3 保守系数） |
| `collectCompactableToolIds` | 228 | no | 收集可压缩工具的 tool_use ID |
| `isMainThreadSource` | 252 | no | 判断 querySource 是否主线程 |
| `microcompactMessages` | 256 | yes | **入口函数**，分发到三条路径 |
| `cachedMicrocompactPath` | 309 | no | 缓存编辑路径（cache_edits API） |
| `evaluateTimeBasedTrigger` | 427 | yes | 时间触发判断（供外部复用） |
| `maybeTimeBasedMicrocompact` | 452 | no | 时间触发路径（直接改消息内容） |
| `getCachedMCModule` | 62 | no | 懒加载 cachedMicrocompact 模块 |
| `ensureCachedMCState` | 71 | no | 确保 cachedMCState 已初始化 |
| `consumePendingCacheEdits` | 88 | yes | 取出待发送的 cache_edits（供 API 层消费） |
| `getPinnedCacheEdits` | 100 | yes | 获取已固定的 cache_edits（重发保持缓存命中） |
| `pinCacheEdits` | 111 | yes | 将 cache_edits 固定到指定 user message 位置 |
| `markToolsSentToAPIState` | 124 | yes | 标记工具已发送到 API |
| `resetMicrocompactState` | 130 | yes | 重置 cachedMC 状态 |

### 入口函数完整调用流程

```
microcompactMessages(messages, toolUseContext, querySource) [256行]
  │
  ├─ clearCompactWarningSuppression()                        [263]
  │
  ├─1→ maybeTimeBasedMicrocompact(messages, querySource)     [271]
  │    │  时间触发：缓存已冷，直接改内容
  │    │
  │    ├── evaluateTimeBasedTrigger(messages, querySource)    [457]
  │    │   ├── getTimeBasedMCConfig()                        [432]
  │    │   ├── isMainThreadSource(querySource)                [437]
  │    │   └── 计算 gapMinutes（距上次 assistant 消息的分钟数）[444]
  │    │       → 小于阈值返回 null，否则返回 { gapMinutes, config }
  │    │
  │    ├── collectCompactableToolIds(messages)                [463]
  │    │   └── 遍历 assistant 消息，收集 COMPACTABLE_TOOLS 中的 tool_use ID
  │    │
  │    ├── 保留最近 N 条 (keepSet)，其余加入清除集 (clearSet) [468-470]
  │    │
  │    ├── 遍历 messages，将 clearSet 中的 tool_result 替换为
  │    │   "[Old tool result content cleared]"                [477-499]
  │    │
  │    ├── suppressCompactWarning()                           [518]
  │    ├── resetMicrocompactState()                           [524]
  │    ├── notifyCacheDeletion()                              [533]
  │    └── return { messages: result }
  │
  │    如果返回 null → 落入下一路径
  │
  ├─2→ cachedMicrocompactPath(messages, querySource)         [288]
  │    │  缓存编辑：不改本地消息，通过 cache_edits API 删除
  │    │  前置条件: CACHED_MICROCOMPACT feature gate
  │    │            + isCachedMicrocompactEnabled()
  │    │            + isModelSupportedForCacheEditing(model)
  │    │            + isMainThreadSource(querySource)
  │    │
  │    ├── getCachedMCModule()                                [314]
  │    ├── ensureCachedMCState()                              [315]
  │    ├── getCachedMCConfig()                                [316]
  │    │
  │    ├── collectCompactableToolIds(messages)                [318]
  │    ├── 遍历 user 消息，注册 tool_result:
  │    │   ├── registerToolResult(state, tool_use_id)         [329]
  │    │   └── registerToolMessage(state, groupIds)           [333]
  │    │
  │    ├── getToolResultsToDelete(state)                      [337]
  │    │   → 根据 triggerThreshold / keepRecent 决定删除哪些
  │    │
  │    ├── (有删除时):
  │    │   ├── createCacheEditsBlock(state, toolsToDelete)    [341]
  │    │   │   → 设置 pendingCacheEdits（模块级变量）
  │    │   ├── suppressCompactWarning()                       [364]
  │    │   ├── notifyCacheDeletion()                          [371]
  │    │   └── return { messages, compactionInfo: { pendingCacheEdits } }
  │    │       消息不改，API 层通过 consumePendingCacheEdits() 取出编辑指令
  │    │
  │    └── (无删除时):
  │        └── return { messages }
  │
  └─3→ return { messages }                                   [296]
       不压缩，交给后续 autoCompact 处理
```

### 三条路径的核心区别

```
路径1: 时间触发 (maybeTimeBasedMicrocompact)
  触发条件: 距上次 assistant 消息 > gapThresholdMinutes
  操作方式: 直接修改消息内容 → content = "[Old tool result content cleared]"
  适用场景: 用户离开一段时间回来，服务端缓存已过期（冷缓存）
  影响: 改变 prompt 内容，下次请求完整重写

路径2: 缓存编辑 (cachedMicrocompactPath)
  触发条件: ant 用户 + 支持的模型 + 主线程 + 工具数超过阈值
  操作方式: 不改本地消息，生成 cache_edits 指令由 API 层发送
  适用场景: 正常交互中，服务端缓存温热（热缓存）
  影响: 保留 prompt cache，仅删除服务端缓存中的 tool_result 条目

路径3: 不压缩
  触发条件: 以上都不满足（外部用户 / 子 agent / 工具数不够）
  后续: 交给 autoCompact 处理上下文压力
```

### 模块级状态（缓存编辑路径专用）

```
cachedMCModule    ← 懒加载的 cachedMicrocompact.js 模块
cachedMCState     ← 追踪已注册/已删除的工具 ID
pendingCacheEdits ← 待发送的 cache_edits 块（API 层通过 consumePendingCacheEdits 取出）
```

生命周期：
- `registerToolResult` / `registerToolMessage` 在每次 microcompact 时累积
- `markToolsSentToAPIState` 在 API 响应后标记已发送
- `resetMicrocompactState` 在时间触发路径或 `/clear` 时重置（因为缓存已失效）

### 与 API 层的协作流程（缓存编辑路径）

```
queryLoop 迭代:
  │
  ├── microcompactMessages()
  │   └── cachedMicrocompactPath() → pendingCacheEdits = {...}
  │
  ├── claude.ts API 调用前:
  │   ├── consumePendingCacheEdits() → 取出 pending 块
  │   ├── getPinnedCacheEdits() → 取出所有已固定块
  │   └── 将 cache_edits 插入请求体
  │
  ├── API 响应后:
  │   ├── markToolsSentToAPIState()
  │   ├── pinCacheEdits(userMessageIndex, block)
  │   └── 读取 cache_deleted_input_tokens 计算实际删除量
  │
  └── queryLoop yield 延迟的 boundary message（用实际删除量而非估算）
```

---

## 四、autoCompact 详解

### 函数列表

| 函数 | 行号 | 导出 | 类型 |
|---|---|---|---|
| `getEffectiveContextWindowSize` | 33 | yes | 同步 |
| `getAutoCompactThreshold` | 74 | yes | 同步 |
| `calculateTokenWarningState` | 96 | yes | 同步 |
| `isAutoCompactEnabled` | 151 | yes | 同步 |
| `shouldAutoCompact` | 164 | yes | async |
| `autoCompactIfNeeded` | 246 | yes | async |

### 调用关系图

```
autoCompactIfNeeded (入口，246行)
├── isAutoCompactEnabled()                   [261] 判断DISABLE_COMPACT
├── shouldAutoCompact()                      [275]
│   ├── isAutoCompactEnabled()               [190]
│   │   ├── isEnvTruthy()                    [外部]
│   │   └── getGlobalConfig()                [外部]
│   ├── getFeatureValue_CACHED_MAY_BE_STALE()[201] (feature gate内)
│   ├── isContextCollapseEnabled()           [225] (feature gate内, require动态导入)
│   ├── tokenCountWithEstimation()           [外部, 230]
│   ├── getAutoCompactThreshold()            [231]
│   │   └── getEffectiveContextWindowSize()  [76]
│   │       ├── getMaxOutputTokensForModel() [外部]
│   │       ├── getContextWindowForModel()   [外部]
│   │       └── getSdkBetas()                [外部]
│   ├── getEffectiveContextWindowSize()      [232]
│   ├── logForDebugging()                    [外部, 234]
│   └── calculateTokenWarningState()         [238]
│       ├── getAutoCompactThreshold()        [107]
│       ├── isAutoCompactEnabled()           [108]
│       └── getEffectiveContextWindowSize()  [110, 126]
├── getAutoCompactThreshold()                [292]
├── trySessionMemoryCompaction()             [295, 外部]
│   (成功时):
│   ├── setLastSummarizedMessageId()         [303]
│   ├── runPostCompactCleanup()              [304]
│   ├── notifyCompaction()                   [311, feature gate内]
│   └── markPostCompaction()                 [312]
├── compactConversation()                    [320, 外部]
│   (成功时):
│   ├── setLastSummarizedMessageId()         [332]
│   └── runPostCompactCleanup()              [333]
│   (失败时):
│   ├── hasExactErrorMessage()               [342]
│   ├── logError()                           [343]
│   └── logForDebugging()                    [351]
```

### 核心流程

1. **`autoCompactIfNeeded`** 是唯一的顶层入口，由外部 query loop 调用。
2. 它先做两层"短路"检查：`DISABLE_COMPACT` 环境变量 -> 连续失败断路器（最多 3 次）-> `shouldAutoCompact()`。
3. **`shouldAutoCompact`** 内部依次过滤：递归 guard（`session_memory`/`compact`/`marble_origami`）-> `isAutoCompactEnabled()` -> reactive/context-collapse feature gate -> 最终通过 `calculateTokenWarningState` 判断 token 是否超过阈值。
4. 需要压缩时，**两阶段策略**：先尝试轻量的 `trySessionMemoryCompaction`，失败后才走完整的 `compactConversation`。
5. **`getEffectiveContextWindowSize`** 是最底层的基础函数，被 `getAutoCompactThreshold` 和 `calculateTokenWarningState` 多处复用，负责计算"模型上下文窗口 - 预留输出 token"。

---

## 五、关键变量：snipTokensFreed

`snipTokensFreed` 用于**补偿 Snip 操作已释放但 token 计数器尚未感知到的空间**。

### 背景问题

Snip 是一种消息裁剪机制，会删除部分旧消息来腾出空间。但幸存的 assistant 消息的 `usage` 字段记录的仍是 snip 之前的 token 数（API 返回值不会变），导致 `tokenCountWithEstimation()` **高估**当前实际用量，可能误触发 autoCompact。

### 修正方式

在 `shouldAutoCompact` 第 230 行：

```ts
const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
```

用 snip 操作时已计算好的释放量做减法修正，避免 autoCompact 被误触发。

### 传递链路

```
autoCompactIfNeeded(snipTokensFreed)     [246行, 参数]
  └── shouldAutoCompact(snipTokensFreed)  [275行, 透传]
        └── tokenCountWithEstimation(messages) - snipTokensFreed  [230行, 使用]
```

---

## 六、tokenCountWithEstimation 详解

**文件**: `src/utils/tokens.ts:226`

这是系统中**测量上下文窗口大小的标准函数**，所有阈值判断（autoCompact、session memory init 等）都应使用它。

### 算法三步走

**第 1 步：从后向前找最近一条有 usage 的 assistant 消息**

```ts
let i = messages.length - 1
while (i >= 0) {
  const usage = getTokenUsage(messages[i])  // 过滤掉 SYNTHETIC_MESSAGES/SYNTHETIC_MODEL
  if (usage) { ... }
  i--
}
```

**第 2 步：处理并行 tool call 的消息拆分**

当模型一次返回多个 tool_use 时，streaming 代码将每个 content block 拆成**独立的 assistant 记录**（共享同一 `message.id` 和 `usage`），消息数组形如：

```
[..., assistant(id=A), user(result_1), assistant(id=A), user(result_2), ...]
```

如果只从最后一条 `id=A` 开始估算，会漏掉前面交错的 tool_result。代码向前回溯到同一 `message.id` 的第一条拆分记录：

```ts
const responseId = getAssistantMessageId(message)
let j = i - 1
while (j >= 0) {
  const priorId = getAssistantMessageId(messages[j])
  if (priorId === responseId) i = j        // 同一 API 响应的更早拆分，锚点前移
  else if (priorId !== undefined) break     // 碰到不同 API 响应，停止
  // priorId === undefined: user/tool_result 消息，继续向前
  j--
}
```

**第 3 步：精确 + 粗估组合**

```ts
return getTokenCountFromUsage(usage) + roughTokenCountEstimationForMessages(messages.slice(i + 1))
```

- 前半：API 返回的 usage（`input_tokens + cache_creation + cache_read + output_tokens`）作为精确基数
- 后半：对 `i+1` 之后所有新增消息做粗略估算（字符数 / 4）

**兜底**：如果没有任何带 usage 的消息，对全部消息做粗略估算。

### roughTokenCountEstimation 按类型估算规则

| Content Block 类型 | 估算方式 |
|---|---|
| `text` / `thinking` | `chars / 4` |
| `image` / `document` | 固定 2000 tokens（避免 base64 高估） |
| `tool_use` | `(name + JSON.stringify(input))` 的 `chars / 4` |
| `tool_result` | 递归处理其 content |
| `redacted_thinking` | `data.length / 4` |
| 其他（server_tool_use 等） | `JSON.stringify(block)` 的 `chars / 4` |

### 设计要点

1. **hybrid 策略**：精确 API usage + 粗估新增消息，避免纯累加导致的双重计数
2. **并行 tool call 感知**：回溯到同一 `message.id` 的第一条拆分，确保交错的 tool_result 全部纳入估算
3. **image/document 固定值**：避免 base64 字符数（远大于实际 token）导致高估

---

## 七、autoCompact 阈值计算与触发条件

### 阈值公式

```
effectiveContextWindow = contextWindowForModel - min(maxOutputTokensForModel, 20000)
autoCompactThreshold   = effectiveContextWindow - 13000 (AUTOCOMPACT_BUFFER_TOKENS)
```

可通过环境变量覆盖：
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` — 限制 contextWindow 上限
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` — 按百分比设置阈值（取百分比值和默认值的较小者）

### shouldAutoCompact 完整过滤链

```
shouldAutoCompact(messages, model, querySource, snipTokensFreed)
  │
  ├─ querySource === 'session_memory' | 'compact' → false    递归 guard，防死锁
  ├─ querySource === 'marble_origami' → false                 ctx-agent guard (feature gate)
  ├─ !isAutoCompactEnabled() → false
  │   ├─ DISABLE_COMPACT=1 → false
  │   ├─ DISABLE_AUTO_COMPACT=1 → false
  │   └─ globalConfig.autoCompactEnabled === false → false
  ├─ REACTIVE_COMPACT gate: tengu_cobalt_raccoon → false      reactive-only 模式抑制
  ├─ CONTEXT_COLLAPSE gate: isContextCollapseEnabled() → false collapse 拥有上下文管理
  │
  └─ tokenCountWithEstimation(messages) - snipTokensFreed >= autoCompactThreshold → true
```

### autoCompactIfNeeded 执行流程

```
autoCompactIfNeeded(messages, toolUseContext, cacheSafeParams, querySource, tracking, snipTokensFreed)
  │
  ├─ DISABLE_COMPACT → return { wasCompacted: false }
  │
  ├─ tracking.consecutiveFailures >= 3 → return { wasCompacted: false }   断路器
  │
  ├─ shouldAutoCompact() → false → return { wasCompacted: false }
  │
  ├─ 构建 recompactionInfo（诊断上下文）
  │
  ├─ trySessionMemoryCompaction()                         优先尝试轻量 session memory 压缩
  │   └─ 成功 → setLastSummarizedMessageId(undefined)
  │           → runPostCompactCleanup()
  │           → notifyCompaction() (feature gate)
  │           → markPostCompaction()
  │           → return { wasCompacted: true, compactionResult }
  │
  ├─ compactConversation()                                重量级 LLM 摘要压缩
  │   └─ 成功 → setLastSummarizedMessageId(undefined)
  │           → runPostCompactCleanup()
  │           → return { wasCompacted: true, consecutiveFailures: 0 }
  │
  └─ catch → logError()
          → consecutiveFailures++
          → return { wasCompacted: false, consecutiveFailures }
```

### compactConversation 核心步骤

1. **PreCompact hooks** — 执行用户配置的预压缩钩子，合并自定义指令
2. **streamCompactSummary** — 调用 LLM 生成对话摘要
   - 优先走 forked agent 路径（复用主对话的 prompt cache）
   - 失败回退到普通 streaming 路径
   - 如果摘要请求本身 prompt-too-long，truncateHeadForPTLRetry 丢弃最旧的 API-round 组重试（最多 3 次）
3. **清理缓存** — `readFileState.clear()`, `loadedNestedMemoryPaths.clear()`
4. **构建 post-compact 附件** — 并行生成：
   - 最近访问文件（最多 5 个，token 预算 50K）
   - 异步 agent 状态
   - Plan 文件 / Plan mode 指令
   - Skill 内容（每个最多 5K tokens，总预算 25K）
   - Deferred tools / Agent listing / MCP instructions delta
5. **SessionStart hooks** — 执行会话启动钩子
6. **分析与遥测** — 记录 tengu_compact 事件（压缩前后 token 数、是否会重触发等）
7. **PostCompact hooks** — 执行用户配置的后压缩钩子

### 相关常量一览

| 常量 | 值 | 用途 |
|---|---|---|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | autoCompact 阈值 = effective - 13K |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | 20,000 | 上下文用量警告阈值 |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | 20,000 | 上下文用量错误阈值 |
| `MANUAL_COMPACT_BUFFER_TOKENS` | 3,000 | 阻塞限制 = effective - 3K |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | 压缩摘要预留输出 token |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | 断路器阈值 |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | 压缩后恢复最近文件数 |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | 文件恢复 token 预算 |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | 单文件恢复 token 上限 |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5,000 | 单 skill 恢复 token 上限 |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25,000 | skill 恢复总 token 预算 |
