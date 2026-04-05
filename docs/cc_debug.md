# Debug Guide

## Prerequisites

1. Install [Bun for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=oven.bun-vscode) extension
2. Ensure `bun` is in your PATH (`~/.bun/bin/bun`)

## Anti-Debug Protection

The source code has a debug detection guard in `src/main.tsx`:

```typescript
if (false && "external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);
}
```

This has been disabled (`if (false && ...)`). If it gets reverted, you'll need to disable it again to allow debugging.

## Method 1: Launch Mode (Recommended)

Directly launch and debug from VS Code:

1. Open VS Code Debug panel (`Ctrl+Shift+D`)
2. Select **"Debug claude-haha (CLI)"** from the dropdown
3. Press `F5`
4. Set breakpoints as needed

This mode is managed by the Bun extension and handles the inspector connection automatically.

### Recovery CLI

To debug the recovery CLI instead, select **"Debug Recovery CLI"** from the dropdown.

## Method 2: Attach Mode

Attach to an already running process。使用固定 WebSocket 路径 `/claude-debug`，无需每次手动更新 URL。

### Steps

1. Start the program with inspector in a terminal (use the fixed path `claude-debug`):

   ```bash
   bun --inspect-wait=ws://127.0.0.1:6499/claude-debug --preload ./preload.ts --env-file=.env ./src/entrypoints/cli.tsx
   ```

   (`--inspect-wait` pauses execution until the debugger connects; use `--inspect` to start immediately)

2. In VS Code Debug panel, select **"Attach to Bun (Inspector)"** and press `F5`

   The `launch.json` already has the matching fixed URL `ws://127.0.0.1:6499/claude-debug`, so it connects directly.

## Method 3: Browser DevTools

1. Start with inspector:

   ```bash
   bun --inspect=ws://127.0.0.1:6499/claude-debug --preload ./preload.ts --env-file=.env ./src/entrypoints/cli.tsx
   ```

2. Open in browser:

   ```
   https://debug.bun.sh/#127.0.0.1:6499/claude-debug
   ```

   This opens Bun's built-in web debugger.
