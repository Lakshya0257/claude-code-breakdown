# Stop Hooks

`src/query/stopHooks.ts` implements the post-turn lifecycle. It fires after the model's turn ends but before the REPL returns control to the user.

---

## Overview

`handleStopHooks()` is an async generator:

```typescript
async function* handleStopHooks(
  messages: Message[],
  context: ToolUseContext,
  systemPrompt: string,
  tools: Tool[]
): AsyncGenerator<...> {
  // Returns: StopHookResult { blockingErrors, preventContinuation }
}
```

---

## Execution Order

### 1. State Snapshot
```typescript
const ctx = createREPLHookContext({
  messages,
  systemPrompt,
  context,
  tools,
})
```

Creates a serializable snapshot of the current session state to pass to hooks.

### 2. Parameter Persistence
For `repl_main_thread` or `sdk` sources only, saves cache-safe parameters. This enables `/btw` (background task wakeup) to restore the session state.

### 3. Job Classification (Ant-only, feature-gated)
```typescript
if (feature('TEMPLATES') && process.env.CLAUDE_JOB_DIR) {
  // Classify the job and write state to CLAUDE_JOB_DIR
  // Races against a timeout to prevent blocking
  await Promise.race([
    jobClassifierModule.classifyAndWriteState(),
    timeout(5000),
  ])
}
```

### 4. Background Tasks (fire-and-forget)
In non-bare mode, these run asynchronously — they do NOT block the response:

```typescript
// Prompt suggestion (if CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION)
executePromptSuggestion(messages, context)

// Memory extraction (if EXTRACT_MEMORIES feature)
extractMemoriesModule.executeExtractMemories(messages, context)

// Auto-dream (for all non-agent queries)
executeAutoDream(messages, context)
```

**Note:** These are genuinely fire-and-forget. Errors are caught and logged but don't surface to the user.

### 5. Computer Use Cleanup (ant-only, Chicago MCP)
```typescript
if (feature('CHICAGO_MCP')) {
  cleanupComputerUseAfterTurn()
}
```

### 6. Stop Hook Execution
Iterates results from `executeStopHooks()`:

```typescript
for await (const result of executeStopHooks(ctx)) {
  // Track per-hook: hookId, hookName, hookEvent, duration
  // Check for blocking errors
  // Check for preventContinuation signal
  // Yield progress messages to REPL
}
```

Each hook can:
- **Block nothing** (exit 0) → normal
- **Block with error** (exit 2) → `blockingErrors` populated, shown to user
- **Prevent continuation** → `{preventContinuation: true}` → loop won't continue even if tools were called

### 7. Summary Message
```typescript
yield createStopHookSummaryMessage({
  hookCount,
  hookErrors,
  hookInfos,
  blockingErrors,
  preventContinuation,
})
```

### 8. Teammate Hooks (if `isTeammate()`)
If this is a swarm teammate agent:
```typescript
// Run TaskCompleted hooks for in-progress tasks owned by this teammate
for (const task of getInProgressTasksForTeammate()) {
  await executeTaskCompletedHooks(task, ctx)
}

// Run TeammateIdle hooks
await executeTeammateIdleHooks(ctx)
```

These also collect blocking errors and `preventContinuation` flags.

---

## Error Handling

Errors in stop hooks are caught at the generator level:
```typescript
try {
  // ...hook execution
} catch (err) {
  logEvent('tengu_stop_hook_error', { durationMs, error: err.message })
  yield createSystemMessage({ content: err.message })
  return { blockingErrors: [], preventContinuation: false }
}
```

Stop hook errors never crash the main session.

---

## Return Value

```typescript
type StopHookResult = {
  blockingErrors: Message[]     // Error messages to show user
  preventContinuation: boolean  // Whether to stop the loop
}
```

The query loop checks this after `handleStopHooks()` resolves to determine whether to continue iterating.
