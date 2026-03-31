# Entry Point: main.tsx

`src/main.tsx` is a ~4,700-line monolith that serves as the CLI entry point. It handles everything from early startup to launching the interactive REPL.

---

## Startup Sequence

### 1. Very Early (line 1–20)
Executed before any heavy imports:

```typescript
profileCheckpoint('main_tsx_entry')     // Performance profiling
startMdmRawRead()                        // Spawn macOS/Windows MDM reads (subprocess)
startKeychainPrefetch()                  // Pre-load OAuth + API key from keychain
```

Both MDM and keychain reads are fire-and-forget background processes started immediately to hide latency.

### 2. Heavy Imports (line 21–207)
All module imports and commander.js setup happen here. The import order is carefully managed to avoid circular dependencies and defer expensive initializations.

### 3. Debug Detection (line 232–271)
If `--inspect` flag is detected (non-ant only), process exits immediately with guidance. This prevents accidental debugging sessions with full permissions.

---

## Key Initialization Functions

### `logSessionTelemetry()`
Called before `runHeadless()` and interactive startup.
- Logs model in use
- Calls `logSkillsLoaded()` — reports which skills loaded and from where
- Logs plugin enabled/error states

### `logStartupTelemetry()`
```typescript
function logStartupTelemetry() {
  // Logs: git status, worktree count, GitHub auth, sandbox state,
  //       auto-updater, reduced motion prefs, cert env vars
}
```

### `runMigrations()`
```typescript
const CURRENT_MIGRATION_VERSION = 11

function runMigrations() {
  // Checks stored migration version vs CURRENT_MIGRATION_VERSION
  // Runs all pending migrations sequentially
  // Non-blocking async migration: migrateChangelogFromConfig()
  // Updates stored version after success
}
```

### `prefetchSystemContextIfSafe()`
```typescript
function prefetchSystemContextIfSafe() {
  // Non-interactive: always prefetch git context (trust implicit)
  // Interactive: prefetch ONLY if trust dialog accepted
  //              Reason: avoid running untrusted .git hooks
}
```

### `startDeferredPrefetches()`
Called AFTER REPL renders (post-first-paint). This hides the cost of expensive prefetches behind the rendering of the first frame:
```typescript
function startDeferredPrefetches() {
  // Skipped if CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER or --bare
  //
  // Spawns:
  //   initUser()                    ← user profile + tier info
  //   getUserContext()              ← IDE context if connected
  //   countFilesRoundedRg()         ← project file count
  //   getModelCapabilities()        ← model context window size
  //   change detectors              ← watch for settings/memory changes
  //
  // Bedrock/Vertex: prefetch AWS/GCP credentials
  //
  // Fire-and-forget:
  //   analytics initialization
  //   MCP URL fetching
  //   event loop stall detector
}
```

---

## Main Handler

```typescript
// line 4504-4512
await program.parseAsync(process.argv)
profileReport()  // Log startup performance checkpoints
```

---

## Telemetry at Startup

`logTenguInit()` fires with ~20 initialization parameters:
- `entrypoint`: CLI mode (interactive, headless, sdk, etc.)
- `flags`: Which CLI flags were set
- `model`: Model in use
- `tools`: Which tools are available
- `permissions`: Permission mode
- `thinking_config`: Extended thinking settings
- Plus: auth provider, worktree count, sandbox state, plugin count, etc.

---

## Command Registration

`program` is a commander.js `Command` instance. Commands registered include:

| Command | Mode | Description |
|---------|------|-------------|
| `claude [prompt...]` | interactive | Main REPL |
| `claude --print/-p` | headless | Non-interactive one-shot |
| `claude skills` | | List/install/uninstall skills |
| `claude agents` | | List agent definitions |
| `claude plugin` | | Install/enable/disable/update |
| `claude mcp` | | Add/remove MCP servers |
| `claude logs` | | Show logs |
| `claude export` | | Export session |
| `claude task` | | Task management |
| `claude update` | | Self-update |
| `claude doctor` | | Health check |
| `claude auto-mode` | ant-only | Auto mode settings |
| `claude remote-control` | feature-gated | Bridge mode |
| `claude assistant` | KAIROS-gated | Autonomous assistant |

---

## Feature-Gated Entry Paths

```typescript
// main.tsx uses feature() guards for entire code paths
if (feature('TRANSCRIPT_CLASSIFIER')) {
  program.command('auto-mode')...
}
if (feature('BRIDGE_MODE')) {
  program.command('remote-control')...
}
if (feature('KAIROS')) {
  program.command('assistant')...
}
```

Bun's bundler performs dead-code elimination: if `feature('X')` returns `false` at build time, the entire branch is removed from the bundle.

---

## `maybeActivateProactive()`

```typescript
function maybeActivateProactive() {
  // Checks CLAUDE_CODE_PROACTIVE env or --proactive flag
  // Calls proactiveModule.activateProactive('command')
  // This enables autonomous "tick" mode
}
```

## `maybeActivateBrief()`

```typescript
function maybeActivateBrief() {
  // Checks CLAUDE_CODE_BRIEF env or --brief flag
  // Validates entitlement via isBriefEntitled()
  // Sets setUserMsgOptIn(true) if entitled
  // Logs tengu_brief_mode_enabled event
}
```

---

## System Prompt Prefixes

Three different intro lines selected based on context (`src/constants/system.ts`):

1. **Interactive / Vertex**: `"You are Claude Code, Anthropic's official CLI for Claude."`
2. **Non-interactive + append system prompt**: `"You are Claude Code, running within the Claude Agent SDK."`
3. **Non-interactive**: `"You are a Claude agent, built on Anthropic's Claude Agent SDK."`

The prefix is chosen in `getCLISyspromptPrefix()` at startup.

---

## Attribution Header

When `CLAUDE_CODE_ATTRIBUTION_HEADER` is set and GrowthBook gate passes:

```
x-anthropic-billing-header: cc_version={VERSION}.{fingerprint}; cc_entrypoint={entrypoint};{cch}{workload}
```

- `cch=349a9` placeholder is overwritten by Bun's HTTP stack with a computed hash (same byte length, prevents Content-Length drift)
- `cc_workload={workload}` routes requests to different API capacity pools
- Server verifies the computed hash to confirm this is a real Claude Code client
