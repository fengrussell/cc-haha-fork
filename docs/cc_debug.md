# Claude Code 调试指南

## 一、VSCode 调试

按 `Cmd+Shift+D` 打开 Run and Debug 面板，选择配置后按 F5 启动。

前置条件：安装 [Bun for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=oven.bun-vscode) 扩展。

### 配置说明 (.vscode/launch.json)

| 配置名 | 模式 | 用途 |
|---|---|---|
| Debug claude-haha (CLI) | launch | 直接启动主 CLI 调试，已默认带 `--debug` 参数 |
| Debug Recovery CLI | launch | 启动恢复模式 CLI |
| Attach to Bun (Inspector) | attach | 附加到已运行的 Bun 进程 |

### Launch 模式（推荐）

1. 选择 **"Debug claude-haha (CLI)"**，按 F5
2. 该配置已带 `args: ["--debug"]`，`logForDebugging` 自动开启
3. 设置断点即可调试

### Attach 模式

Attach 是附加到已运行进程，**不启动新进程**，所以 launch.json 中的 `args` 对它无效。`--debug` 必须在启动进程时传入。

步骤：

1. 在终端启动带 inspector 的进程（`--debug` 在 `--` 之后传给应用）：

   ```bash
   bun --inspect-wait=ws://127.0.0.1:6499/claude-debug --preload ./preload.ts --env-file=.env ./src/entrypoints/cli.tsx -- --debug
   ```

   - `--inspect-wait` 暂停执行直到调试器连接；用 `--inspect` 则立即启动
   - `-- --debug` 将 `--debug` 传给 CLI 应用，开启 logForDebugging

2. 在 VSCode 选 **"Attach to Bun (Inspector)"** 按 F5，自动连接 `ws://127.0.0.1:6499/claude-debug`

也可用环境变量替代 `--debug`：

```bash
DEBUG=1 bun --inspect=ws://127.0.0.1:6499/claude-debug --preload ./preload.ts --env-file=.env ./src/entrypoints/cli.tsx
```

### 浏览器 DevTools

```bash
bun --inspect=ws://127.0.0.1:6499/claude-debug --preload ./preload.ts --env-file=.env ./src/entrypoints/cli.tsx
```

打开 `https://debug.bun.sh/#127.0.0.1:6499/claude-debug` 使用 Bun 内置 Web 调试器。

## 二、Anti-Debug 保护

`src/main.tsx` 中有调试检测：

```typescript
if (false && "external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);
}
```

已被禁用（`if (false && ...)`）。如果被还原需重新禁用。

## 三、logForDebugging 开关

核心逻辑在 `src/utils/debug.ts`。

### 启动时开启

```bash
# --debug 标志（Launch 配置已默认带此参数）
claude --debug

# 简写
claude -d

# 环境变量
DEBUG=1 claude

# 输出到 stderr
claude --debug-to-stderr    # 或 -d2e

# 指定输出文件
claude --debug-file=/tmp/claude-debug.log

# 带过滤模式（只记录匹配的消息）
claude --debug=autocompact
```

### 运行中开启

在会话中输入 `/debug` 命令，调用 `enableDebugLogging()` 设置 `runtimeDebugEnabled = true`，无需重启。

### 日志级别

默认级别 `debug`，更详细用 `verbose`：

```bash
CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose claude --debug
```

### 日志输出位置

- 默认：`~/.claude/debug/<sessionId>.txt`
- 环境变量：`CLAUDE_CODE_DEBUG_LOGS_DIR`
- 命令行：`--debug-file=<path>`
- `--debug-to-stderr` 直接输出到终端

### 注意

`shouldLogDebugMessage()` 中，`USER_TYPE=ant`（Anthropic 内部用户）始终记录日志；外部用户必须显式开启 debug 模式。
