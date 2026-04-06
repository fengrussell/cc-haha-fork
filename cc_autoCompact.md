# autoCompact.ts 函数调用关系分析

## 函数列表

| 函数 | 行号 | 导出 | 类型 |
|---|---|---|---|
| `getEffectiveContextWindowSize` | 33 | yes | 同步 |
| `getAutoCompactThreshold` | 74 | yes | 同步 |
| `calculateTokenWarningState` | 96 | yes | 同步 |
| `isAutoCompactEnabled` | 151 | yes | 同步 |
| `shouldAutoCompact` | 164 | yes | async |
| `autoCompactIfNeeded` | 246 | yes | async |

## 调用关系图

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

## 核心流程总结

1. **`autoCompactIfNeeded`** 是唯一的顶层入口，由外部 query loop 调用。
2. 它先做两层"短路"检查：`DISABLE_COMPACT` 环境变量 -> 连续失败断路器（最多 3 次）-> `shouldAutoCompact()`。
3. **`shouldAutoCompact`** 内部依次过滤：递归 guard（`session_memory`/`compact`/`marble_origami`）-> `isAutoCompactEnabled()` -> reactive/context-collapse feature gate -> 最终通过 `calculateTokenWarningState` 判断 token 是否超过阈值。
4. 需要压缩时，**两阶段策略**：先尝试轻量的 `trySessionMemoryCompaction`，失败后才走完整的 `compactConversation`。
5. **`getEffectiveContextWindowSize`** 是最底层的基础函数，被 `getAutoCompactThreshold` 和 `calculateTokenWarningState` 多处复用，负责计算"模型上下文窗口 - 预留输出 token"。

## 关键变量：snipTokensFreed

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
