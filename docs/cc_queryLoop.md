# query.ts 关键函数调用流程分析

文件：`src/query.ts`（1732行）

## 一、文件级函数总览

| 函数 | 行号 | 导出 | 类型 | 用途 |
|---|---|---|---|---|
| `yieldMissingToolResultBlocks` | 123 | no | Generator | 为未完成的 tool_use 生成错误 tool_result |
| `isWithheldMaxOutputTokens` | 175 | no | 同步 | 判断是否为可恢复的 max_output_tokens 错误 |
| `query` | 219 | yes | AsyncGenerator | **外部入口**，调用 queryLoop 并通知命令完成 |
| `queryLoop` | 242 | no | AsyncGenerator | **核心循环**，承载全部主逻辑 |

## 二、整体架构

```
外部调用者 (REPL / SDK / HFI)
  │
  └── query(params) [219行, 导出入口]
        │
        ├── yield* queryLoop(params, consumedCommandUuids) [231]
        │     └── while(true) { ... } [309-1730]
        │
        └── notifyCommandLifecycle(uuid, 'completed') [236-238]
            (仅在 queryLoop 正常 return 后执行)
```

- `query()` 是外部入口，调用 `queryLoop()` 并在正常返回后通知已消费的命令 `completed`
- `queryLoop()` 是 `AsyncGenerator`，通过 `yield` 向外流式产出消息/事件，通过 `return` 返回终止原因
- 循环通过 `state = next; continue` 进入下一轮迭代，通过 `return { reason }` 退出

## 三、QueryParams 输入参数

```ts
type QueryParams = {
  messages: Message[]              // 对话消息历史
  systemPrompt: SystemPrompt       // 系统提示词
  userContext / systemContext       // 附加上下文
  canUseTool: CanUseToolFn         // 工具权限判断
  toolUseContext: ToolUseContext    // 工具执行上下文（含 abortController、agentId 等）
  fallbackModel?: string           // 备用模型（主模型失败时切换）
  querySource: QuerySource         // 来源标识（repl_main_thread / sdk / agent:xxx）
  maxTurns?: number                // 最大轮次限制
  taskBudget?: { total: number }   // API task_budget
  deps?: QueryDeps                 // 可注入的依赖（测试用）
}
```

## 四、State 状态对象

```ts
type State = {
  messages: Message[]                           // 当前消息列表
  toolUseContext: ToolUseContext                 // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState // 自动压缩追踪
  maxOutputTokensRecoveryCount: number          // max_output_tokens 恢复次数（上限3）
  hasAttemptedReactiveCompact: boolean          // 是否已尝试过响应式压缩
  maxOutputTokensOverride: number | undefined   // 输出 token 上限覆盖
  pendingToolUseSummary: Promise<...>           // 上一轮工具摘要（异步）
  stopHookActive: boolean | undefined           // stop hook 是否激活
  turnCount: number                             // 当前轮次计数
  transition: Continue | undefined              // 上一次 continue 的原因
}
```

每次 `continue` 都构造一个新的 `State` 对象赋给 `state`，在下一轮迭代顶部解构使用。

## 五、QueryConfig 不可变配置

```ts
// query/config.ts — 在 queryLoop 入口快照一次，整个循环期间不变
type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean  // 是否启用流式工具并行执行
    emitToolUseSummaries: boolean    // 是否生成工具摘要（Haiku 调用）
    isAnt: boolean                   // 是否 Anthropic 内部用户
    fastModeEnabled: boolean         // 是否启用 fast mode
  }
}
```

## 六、单次迭代完整流程

```
┌──────────────────────────────────────────────────────────────────┐
│ while(true) 循环体                                                │
│                                                                  │
│  ┌── 阶段 1: 初始化 ─────────────────────────────────────────┐   │
│  │ 1. 解构 state                                   [310-323] │   │
│  │ 2. Skill Discovery 预取（异步，不阻塞）          [333-337] │   │
│  │ 3. yield stream_request_start                   [339]     │   │
│  │ 4. 初始化/递增 queryTracking (chainId+depth)    [349-365] │   │
│  │ 5. 复制消息 messagesForQuery                     [367]     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── 阶段 2: 消息预处理管线（5层，顺序固定）────────────────┐   │
│  │ 5a. applyToolResultBudget (工具结果大小预算)     [381]    │   │
│  │ 5b. snipCompact (历史裁剪，保留尾部)            [403]    │   │
│  │ 5c. microcompact (微压缩/工具结果清除)          [416]    │   │
│  │ 5d. contextCollapse (上下文折叠投影)            [443]    │   │
│  │ 5e. autoCompact (自动摘要压缩)                  [456]    │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── 阶段 3: 预检查 ─────────────────────────────────────────┐   │
│  │ 6. 构建 fullSystemPrompt                        [451]     │   │
│  │ 7. 阻塞限制检查 (blocking limit)                [630-650] │   │
│  │    → return { reason: 'blocking_limit' }                  │   │
│  │ 8. 初始化 StreamingToolExecutor                 [563-570] │   │
│  │ 9. 获取当前模型 (getRuntimeMainLoopModel)       [574]     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── 阶段 4: API 调用 + 流式接收 ────────────────────────────┐   │
│  │ 10. deps.callModel() streaming loop             [661-865] │   │
│  │     ├── backfillObservableInput (工具输入回填)    [750]    │   │
│  │     ├── 扣留可恢复错误(PTL/max_output/media)    [801-824] │   │
│  │     ├── yield 正常消息给消费者                    [826]    │   │
│  │     ├── 收集 tool_use blocks → needsFollowUp    [831]    │   │
│  │     └── streamingToolExecutor.addTool (并行启动)  [843]    │   │
│  │                                                           │   │
│  │ 11. FallbackTriggeredError → 切换模型重试        [896]    │   │
│  │     └── 清空 assistantMessages, 重建 executor             │   │
│  │                                                           │   │
│  │ 12. 外层 catch: 致命错误处理                     [957-999] │   │
│  │     ├── ImageSizeError → return 'image_error'             │   │
│  │     ├── yieldMissingToolResultBlocks (补齐结果)            │   │
│  │     └── return { reason: 'model_error' }                  │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── 阶段 5: 后处理 ─────────────────────────────────────────┐   │
│  │ 13. executePostSamplingHooks (fire-and-forget)  [1002]    │   │
│  │ 14. 中断检查 (streaming abort)                  [1017]    │   │
│  │     → return { reason: 'aborted_streaming' }              │   │
│  │ 15. yield 上一轮 pendingToolUseSummary           [1057]    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── 阶段 6A: 无 tool_use 时 (needsFollowUp=false) ─────────┐   │
│  │ 16. PTL 413 恢复链:                                       │   │
│  │     ├── contextCollapse.recoverFromOverflow → continue     │   │
│  │     └── reactiveCompact.tryReactiveCompact → continue      │   │
│  │ 17. max_output_tokens 恢复:                               │   │
│  │     ├── escalate 到 64k → continue                        │   │
│  │     └── 注入 resume 提示（最多3次）→ continue              │   │
│  │ 18. handleStopHooks()                          [1269]     │   │
│  │     ├── executeStopHooks (用户 hook)                      │   │
│  │     ├── executePromptSuggestion (fire-and-forget)         │   │
│  │     ├── executeExtractMemories (fire-and-forget)          │   │
│  │     ├── executeAutoDream (fire-and-forget)                │   │
│  │     ├── executeTeammateIdleHooks (teammate 模式)          │   │
│  │     ├── → preventContinuation → return 'stop_hook_prevented'│  │
│  │     └── → blockingErrors → continue                       │   │
│  │ 19. checkTokenBudget → continue 或 return      [1310]    │   │
│  │ 20. return { reason: 'completed' }              [1359]    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌── 阶段 6B: 有 tool_use 时 (needsFollowUp=true) ──────────┐   │
│  │ 21. 执行工具:                                   [1382]    │   │
│  │     ├── streamingToolExecutor.getRemainingResults()        │   │
│  │     └── runTools() (非流式 fallback)                      │   │
│  │ 22. yield 工具结果                               [1387]    │   │
│  │ 23. 生成 nextPendingToolUseSummary (async)      [1414]    │   │
│  │ 24. 中断检查 → return 'aborted_tools'           [1487]    │   │
│  │ 25. hook 阻止 → return 'hook_stopped'           [1521]    │   │
│  │ 26. 收集 attachments:                           [1582]    │   │
│  │     ├── getAttachmentMessages (memory/edited files)       │   │
│  │     ├── pendingMemoryPrefetch.consume                     │   │
│  │     ├── collectSkillDiscoveryPrefetch                     │   │
│  │     └── 消费 queued commands                              │   │
│  │ 27. refreshTools (MCP 热更新)                   [1662]    │   │
│  │ 28. taskSummary (BG_SESSIONS, fire-and-forget)  [1687]    │   │
│  │ 29. maxTurns 检查 → return 'max_turns'          [1707]    │   │
│  │ 30. state = next; continue → 下一轮迭代         [1717]    │   │
│  └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## 七、所有 continue 路径（循环继续）

| 原因 | 行号 | 触发条件 |
|---|---|---|
| `collapse_drain_retry` | 1117 | PTL 413 错误，context collapse drain 成功释放空间 |
| `reactive_compact_retry` | 1167 | PTL/media 错误，reactive compact 成功压缩 |
| `max_output_tokens_escalate` | 1222 | 首次 max_output_tokens，升级到 64k 重试 |
| `max_output_tokens_recovery` | 1253 | max_output_tokens 恢复（最多3次），注入 resume 提示 |
| `stop_hook_blocking` | 1307 | stop hook 返回阻塞错误，重试 |
| `token_budget_continuation` | 1342 | token budget 未耗尽，注入 nudge 提示继续 |
| `next_turn` | 1728 | 正常的工具调用后进入下一轮 |

## 八、所有 return 路径（循环终止）

| reason | 行号 | 触发条件 |
|---|---|---|
| `blocking_limit` | 648 | token 达到阻塞限制，auto-compact 关闭时 |
| `model_error` | 998 | API 调用抛异常（非 FallbackTriggeredError） |
| `image_error` | 979/1177 | 图片大小/调整错误，或 media 恢复失败 |
| `prompt_too_long` | 1177/1184 | PTL 413 所有恢复手段耗尽 |
| `aborted_streaming` | 1053 | 流式接收期间用户中断 |
| `aborted_tools` | 1517 | 工具执行期间用户中断 |
| `hook_stopped` | 1522 | hook 阻止继续 |
| `completed` | 1266/1359 | 正常结束（无 tool_use / API 错误消息） |
| `stop_hook_prevented` | 1281 | stop hook 明确阻止继续 |
| `max_turns` | 1713 | 达到最大轮次限制 |

## 九、handleStopHooks 详解

文件：`src/query/stopHooks.ts`

模型停止输出（无 tool_use）时执行的后处理链：

```
handleStopHooks() [65行]
  │
  ├── saveCacheSafeParams()              [97]  保存上下文快照（供 /btw、side_question 用）
  ├── classifyAndWriteState()            [120] Template job 分类（TEMPLATES feature gate）
  │
  ├── 异步 fire-and-forget（非 --bare 模式）:
  │   ├── executePromptSuggestion()      [139] 生成下一步建议
  │   ├── executeExtractMemories()       [149] 提取记忆（EXTRACT_MEMORIES gate）
  │   └── executeAutoDream()             [155] 自动梦境
  │
  ├── cleanupComputerUseAfterTurn()      [166] 清理 Computer Use 状态
  │
  ├── executeStopHooks()                 [180] 用户配置的 stop hooks
  │   ├── → blockingError → 注入错误消息，返回 blockingErrors
  │   └── → preventContinuation → 返回 preventContinuation: true
  │
  └── (teammate 模式额外):
      ├── executeTaskCompletedHooks()    [353] 任务完成 hooks
      └── executeTeammateIdleHooks()     [403] 空闲 hooks
```

## 十、辅助函数调用关系

### yieldMissingToolResultBlocks [123行]

在以下场景调用，为已 yield 的 tool_use 补齐 tool_result（避免 API 协议错误）：

```
调用点:
  ├── FallbackTriggeredError 切换模型前   [902]
  ├── 外层 catch 致命错误后              [986]
  └── streaming abort 后（非流式执行器）  [1027]
```

### isWithheldMaxOutputTokens [175行]

```
调用点:
  ├── 流式接收中判断是否扣留            [822]
  └── 流式结束后判断是否进入恢复流程    [1190]
```

## 十一、关键外部依赖

| 依赖 | 来源 | 用途 |
|---|---|---|
| `deps.callModel` | `query/deps.ts` → `services/api/claude.ts` | 调用 Claude API 流式返回 |
| `deps.autocompact` | `query/deps.ts` → `services/compact/autoCompact.ts` | 自动压缩 |
| `deps.microcompact` | `query/deps.ts` → `services/compact/microCompact.ts` | 微压缩 |
| `deps.uuid` | `query/deps.ts` → `crypto.randomUUID` | 生成 UUID |
| `buildQueryConfig` | `query/config.ts` | 构建不可变查询配置（gates 快照） |
| `handleStopHooks` | `query/stopHooks.ts` | 处理停止钩子链 |
| `runTools` | `services/tools/toolOrchestration.ts` | 串行执行工具 |
| `StreamingToolExecutor` | `services/tools/StreamingToolExecutor.ts` | 流式并行执行工具 |
| `reactiveCompact` | `services/compact/reactiveCompact.ts` | 响应式压缩（413 恢复） |
| `contextCollapse` | `services/contextCollapse/index.ts` | 上下文折叠 |
| `snipModule` | `services/compact/snipCompact.ts` | 历史裁剪 |
| `checkTokenBudget` | `query/tokenBudget.ts` | Token 预算检查与自动续行 |
| `createBudgetTracker` | `query/tokenBudget.ts` | 创建预算追踪器 |
