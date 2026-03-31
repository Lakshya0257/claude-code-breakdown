# Architecture Overview

Claude Code is structured as a layered system. The layers from top to bottom:

```
┌──────────────────────────────────────────────┐
│           Terminal UI (Ink/React)             │  src/ink/, src/screens/, src/components/
├──────────────────────────────────────────────┤
│         REPL & Slash Commands                 │  src/commands/, src/screens/REPL.tsx
├──────────────────────────────────────────────┤
│              Query Engine                     │  src/query/, src/main.tsx
├──────────────────────────────────────────────┤
│     Tools | Permissions | Hooks | Memory      │  src/tools/, src/utils/permissions/,
│                                               │  src/utils/hooks/, src/memdir/
├──────────────────────────────────────────────┤
│           API Layer (Anthropic)               │  src/services/api/
├──────────────────────────────────────────────┤
│     State | Settings | Analytics              │  src/state/, src/utils/settings/,
│                                               │  src/services/analytics/
└──────────────────────────────────────────────┘
```

---

## Major Subsystems

### 1. Entry Point & CLI (`src/main.tsx`, `src/cli/`, `src/entrypoints/`)
- ~4,700-line monolith that handles startup
- Runs migrations, prefetches git/auth context, registers all commander.js commands
- Chooses between interactive REPL, headless (`--print`), or bridge mode
- Launches deferred prefetches after first render

### 2. Query Engine (`src/query/`)
- The **async generator** loop that drives every turn
- Coordinates: message normalization → compaction → system prompt assembly → API call → tool execution → stop hooks
- `query.ts`: The outer generator, yields `StreamEvent | Message | TombstoneMessage`
- `tokenBudget.ts`: Per-turn token consumption tracking and continuation decisions
- `stopHooks.ts`: Post-turn lifecycle (memory extraction, prompt suggestions, hook execution)
- `config.ts`: Immutable snapshot of feature gates taken at query start
- `deps.ts`: Dependency injection interface (model caller, compaction, UUID)

### 3. API Layer (`src/services/api/`)
- `claude.ts`: Builds API request, handles prompt caching, 1h/5min TTL cache controls
- `client.ts`: Creates Anthropic SDK client for direct API, Bedrock, Vertex, Foundry
- `withRetry.ts`: Exponential backoff, fast mode fallback, context overflow handling, persistent retry mode
- `promptCacheBreakDetection.ts`: Hash-based change tracking, detects when cache breaks and why
- `dumpPrompts.ts`: Debug JSONL logging of all API requests (ant-only)
- `errors.ts`: API error formatting, refusal detection

### 4. Tools (`src/tools/`)
- 45+ tool implementations, each in its own directory
- Each tool: `ToolName.ts` (logic) + `prompt.ts` (model instructions) + `UI.tsx` (rendering)
- Orchestrated by `StreamingToolExecutor` — runs concurrency-safe tools in parallel
- Base interface in `src/Tool.ts`

### 5. System Prompts (`src/constants/`)
- `prompts.ts`: Assembles the ~10,000+ character system prompt dynamically
- Split at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` for Anthropic prompt caching
- Static prefix is globally cacheable; dynamic suffix is session-specific
- Feature-gated sections activated by GrowthBook flags

### 6. Context Management (`src/services/compact/`, `src/memdir/`)
- Auto-compact triggers at `contextWindow - 13,000 tokens`
- 4 compaction strategies: microcompact, cached microcompact, session memory compaction, full compaction
- Memory system: persistent markdown files in `~/.claude/projects/<slug>/memory/`

### 7. Permissions (`src/utils/permissions/`)
- 7 permission modes: default, plan, acceptEdits, bypassPermissions, dontAsk, auto (ant), bubble (ant)
- Rule matching via wildcard/exact/glob patterns
- Filesystem safety: dangerous file/dir allowlist, TOCTOU prevention, UNC blocking
- Auto-mode: ML classifier (transcript-based) auto-allows/denies

### 8. Hooks (`src/utils/hooks/`, `src/hooks/`)
- 16 hook event types: PreToolUse, PostToolUse, Stop, SessionStart, UserPromptSubmit, etc.
- 5 hook command types: command (bash), prompt (LLM), agent (full subagent), http (webhook), bash_command
- File watcher via chokidar for FileChanged events
- AsyncHookRegistry: polling-based completion detection with timeout cleanup

### 9. Multi-Agent (`src/tools/AgentTool/`, `src/tasks/`, `src/utils/swarm/`)
- Agent tool spawns nested QueryEngine instances with isolated state
- Task types: LocalAgent, RemoteAgent, Dream, InProcessTeammate, LocalShell
- Swarm backends: tmux, iTerm2, in-process (auto-detected)
- Coordinator mode: orchestrates worker agents with shared scratchpad

### 10. MCP (`src/services/mcp/`, `src/utils/mcp/`)
- Supports 7 transport types: stdio, SSE, HTTP, WebSocket, SDK control, SSE-IDE, WS-IDE
- Tool names: `mcp__{normalized_server}__{normalized_tool}`
- OAuth 2.0 authentication with RFC 9728/8414 discovery
- Elicitation protocol for form-based and URL-based interactive flows

### 11. State Management (`src/state/`)
- `AppState` is a `DeepImmutable<T>` branded store
- `createStore()` with `Object.is()` comparison (no spurious updates)
- `onChangeAppState()`: reactive side-effects on state change (credential cache clearing, settings sync, CCR notification)

### 12. Terminal UI (`src/ink/`)
- Custom React 19 reconciler for terminal rendering (not react-dom)
- Yoga WASM layout engine (flexbox in the terminal)
- Double-buffered Screen with pool-based style/char/hyperlink interning
- Frame diffing optimizer: blit unchanged regions
- Full keyboard protocol support: Kitty, xterm modifyOtherKeys, X10 mouse, SGR mouse

---

## Key Design Principles

1. **Generator-based streaming**: `query()` is an async generator, enabling progressive rendering without buffering
2. **Immutable state**: All settings/permissions wrapped in `DeepImmutable<T>`, mutations via updater functions
3. **Feature-flag dead code elimination**: `feature()` calls are tree-shaking boundaries; Bun removes ungated code entirely
4. **Prompt cache preservation**: Static prefix cached globally, dynamic suffix per-session; memoized to avoid mid-turn cache busts
5. **Dependency injection**: `QueryDeps` provides narrow DI (4 functions) for easy test mocking
6. **Lazy initialization**: Heavy modules dynamically imported after first render
7. **Security by default**: Path validation, UNC blocking, TOCTOU prevention, shell injection prevention
