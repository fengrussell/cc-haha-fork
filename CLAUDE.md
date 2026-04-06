# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is this

This is a local fork of Claude Code (Anthropic's CLI agent), renamed to `claude-haha`. It's the full source of the Claude Code CLI, running on Bun with React/Ink for terminal UI.

## Running

```bash
# Start the CLI
bun run start
# or
./bin/claude-haha

# With debug logging
./bin/claude-haha --debug

# Recovery CLI (simple readline REPL, no Ink TUI)
CLAUDE_CODE_FORCE_RECOVERY_CLI=1 ./bin/claude-haha
```

## Configuration

Copy `.env.example` to `.env` and configure your API provider. The `.env` file is loaded automatically via `--env-file=.env` in the bin script. Supports Anthropic direct, MiniMax, OpenRouter, LiteLLM proxy (OpenAI/DeepSeek), etc.

Key environment variables:
- `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_BASE_URL` / `ANTHROPIC_MODEL` — API connection
- `DISABLE_TELEMETRY=1` / `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` — recommended for local dev
- `DEBUG=1` or `--debug` — enable `logForDebugging()` output
- `CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose` — include verbose-level logs
- `DISABLE_COMPACT` / `DISABLE_AUTO_COMPACT` — disable compaction

## Architecture

### Entry flow

```
bin/claude-haha (bash)
  → bun --env-file=.env src/entrypoints/cli.tsx
    → src/main.tsx (Commander CLI setup, side-effect prefetches)
      → src/replLauncher.tsx (Ink React app mount)
        → src/screens/REPL.tsx (main interactive screen)
```

`preload.ts` sets build-time macros (VERSION, BUILD_TIME, etc.) for local dev.

### Core loop

```
REPL.tsx
  → query() [src/query.ts:219]  — exported entry, wraps queryLoop
    → queryLoop() [src/query.ts:242]  — while(true) AsyncGenerator
      per iteration:
        1. Message preprocessing pipeline (budget → snip → microcompact → collapse → autocompact)
        2. API call via deps.callModel() with streaming
        3. Tool execution (StreamingToolExecutor or runTools)
        4. Stop hooks / recovery logic
        5. Attachments (memory, skills, queued commands)
        → continue (next turn) or return (terminal reason)
```

### Key directories

- `src/entrypoints/` — CLI, MCP server, init, SDK type exports
- `src/query.ts` + `src/query/` — core query loop, config, deps injection, stop hooks, token budget
- `src/Tool.ts` — Tool type definition, ToolUseContext, tool registry helpers
- `src/tools/` — ~40 tool implementations (BashTool, FileReadTool, AgentTool, etc.), each in its own directory
- `src/services/compact/` — compaction system: autoCompact, microCompact, snipCompact, reactiveCompact, sessionMemoryCompact, contextCollapse
- `src/services/api/` — Claude API client (`claude.ts`), retry logic, prompt cache detection
- `src/services/mcp/` — MCP server/client management
- `src/commands/` — ~100 slash commands (/compact, /debug, /clear, /mcp, etc.)
- `src/screens/` — Ink React screens (REPL, Doctor, ResumeConversation)
- `src/components/` — React components (PromptInput, Message, Spinner, etc.)
- `src/ink/` — Low-level terminal rendering, keypress parsing, focus management
- `src/utils/` — Shared utilities (debug, messages, tokens, config, permissions, hooks, model selection)
- `src/state/` — AppState management
- `src/skills/` — Skill definitions (slash command expansions)

### Tool pattern

Each tool in `src/tools/<ToolName>/` typically has:
- `prompt.ts` — tool name constant, description, input schema
- `<ToolName>.ts` or `tool.ts` — `Tool` interface implementation with `call()` method

Tools are registered in `src/tools.ts` and matched by name/aliases via `toolMatchesName()` in `Tool.ts`.

### Compaction system (src/services/compact/)

The message preprocessing pipeline runs 5 stages per query loop iteration, lightest to heaviest:
1. `applyToolResultBudget` — cap individual tool result sizes
2. `snipCompact` — drop old messages, keep tail
3. `microcompact` — clear old tool result content (3 paths: time-based, cached cache_edits, or skip)
4. `contextCollapse` — fold committed context segments
5. `autoCompact` — full LLM summary replacing entire history (last resort)

### Feature gates

`feature('FLAG_NAME')` from `bun:bundle` is a compile-time tree-shaking boundary. Used for ant-only features (CACHED_MICROCOMPACT, CONTEXT_COLLAPSE, REACTIVE_COMPACT, etc.). These are `if` guards — code inside is DCE'd from external builds.

### Debug system (src/utils/debug.ts)

- `logForDebugging(msg)` — gated by `isDebugMode()`, writes to `~/.claude/debug/<sessionId>.txt`
- `trace(label, ...args)` — lightweight function entry tracing
- Enable via: `--debug`, `-d`, `DEBUG=1`, `/debug` in-session, or `--debug=<filter>`
- `USER_TYPE=ant` always logs; external users must explicitly enable

### Query deps injection (src/query/deps.ts)

`QueryDeps` provides 4 injectable dependencies for testing: `callModel`, `microcompact`, `autocompact`, `uuid`. Production uses `productionDeps()`.

## Detailed analysis docs

See `docs/` for deeper analysis:
- `docs/cc_queryLoop.md` — queryLoop 6-phase iteration flow, all continue/return paths, handleStopHooks detail
- `docs/cc_compact.md` — microCompact vs autoCompact comparison, call graphs, snipTokensFreed
- `docs/cc_debug.md` — VSCode debugging guide, logForDebugging switches, attach mode
