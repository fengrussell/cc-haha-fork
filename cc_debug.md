# Claude Code 调试指南

## 一、VSCode 调试

按 `Cmd+Shift+D` 打开 Run and Debug 面板，选择配置后按 F5 启动。

### 配置说明 (.vscode/launch.json)

| 配置名 | 模式 | 用途 |
|---|---|---|
| Debug claude-haha (CLI) | launch | 直接启动主 CLI 调试，已默认带 `--debug` 参数 |
| Debug Recovery CLI | launch | 启动恢复模式 CLI |
| Attach to Bun (Inspector) | attach | 附加到已运行的 Bun 进程 |

### Attach 模式用法

```bash
# 先启动带 inspector 的进程
bun --inspect=ws://127.0.0.1:6499/claude-debug --preload ./preload.ts src/entrypoints/cli.tsx

# 然后在 VSCode 选 "Attach to Bun (Inspector)" 附加调试
```

## 二、logForDebugging 开关

核心逻辑在 `src/utils/debug.ts`。

### 启动时开启

```bash
# --debug 标志（launch.json 已默认配置）
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
