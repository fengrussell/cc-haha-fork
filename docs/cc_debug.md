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

Attach to an already running process. Note: Bun's inspector generates a **random WebSocket path** on each launch, so the URL in `launch.json` must be updated each time.

### Steps

1. Start the program with inspector in a terminal:

   ```bash
   bun --inspect-wait=127.0.0.1:6499 --preload ./preload.ts --env-file=.env ./src/entrypoints/cli.tsx
   ```

   (`--inspect-wait` pauses execution until the debugger connects; use `--inspect` to start immediately)

2. Copy the WebSocket URL from the output:

   ```
   --------------------- Bun Inspector ---------------------
   Listening:
     ws://127.0.0.1:6499/xxxxxxxxxx    <-- copy this
   --------------------- Bun Inspector ---------------------
   ```

3. Update `.vscode/launch.json` with the URL:

   ```json
   {
     "name": "Attach to Bun (Inspector)",
     "type": "bun",
     "request": "attach",
     "url": "ws://127.0.0.1:6499/xxxxxxxxxx",
     "stopOnEntry": false
   }
   ```

4. In VS Code Debug panel, select **"Attach to Bun (Inspector)"** and press `F5`

### Common Error

**`Unexpected server response: 404`** - This means the WebSocket path in `launch.json` is stale. Copy the fresh URL from step 2.

## Method 3: Browser DevTools

1. Start with inspector:

   ```bash
   bun --inspect=127.0.0.1:6499 --preload ./preload.ts --env-file=.env ./src/entrypoints/cli.tsx
   ```

2. Open the URL printed in the output:

   ```
   https://debug.bun.sh/#127.0.0.1:6499/xxxxxxxxxx
   ```

   This opens Bun's built-in web debugger.
