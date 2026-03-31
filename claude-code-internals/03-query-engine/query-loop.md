# The Query Loop

The query loop is an **async generator function** in `src/query/`. It drives every conversational turn.

---

## Why an Async Generator?

```typescript
async function* query(
  messages: Message[],
  options: QueryOptions,
  tools: Tool[],
  context: ToolUseContext
): AsyncGenerator<StreamEvent | RequestStartEvent | Message | TombstoneMessage>
```

Being an async generator allows the REPL to:
1. Start rendering immediately as text tokens arrive
2. Show tool progress while execution is in flight
3. Handle interruption (Ctrl+C) at any yield point
4. Compose multiple turns without blocking the render loop

---

## `QueryConfig` — Immutable Per-Query Configuration

`src/query/config.ts` captures a snapshot of all gates at query entry:

```typescript
type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean    // Statsig gate (cached)
    emitToolUseSummaries: boolean      // env: CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES
    isAnt: boolean                     // USER_TYPE === 'ant'
    fastModeEnabled: boolean           // !CLAUDE_CODE_DISABLE_FAST_MODE
  }
}
```

**Why snapshot?** Prevents mid-query gate flips from changing behavior halfway through a turn. `feature()` gates are excluded — those are tree-shaking compile-time boundaries.

---

## `QueryDeps` — Dependency Injection

`src/query/deps.ts` provides narrow DI for testing:

```typescript
type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}

function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: () => crypto.randomUUID(),
  }
}
```

Using `typeof fn` keeps the types in sync automatically — no manual duplication.

---

## Per-Turn Loop Steps

Each iteration of the query loop:

### Step 1: Normalize Messages
```
normalizeMessages(messages, context)
```
- Coerce all message types to SDK-compatible format
- Strip tool results exceeding token budget (`applyToolResultBudget()`)
- Inject date-change attachment if date changed since session start
- Handle compaction boundary metadata (headUuid, anchorUuid, tailUuid)
- Normalize `ExitPlanModeV2` tool inputs for the API

### Step 2: Compaction (if needed)
Two compaction checks before each API call:

**Time-Based Microcompact:**
```typescript
if (timeSinceLastAssistantMsg > 60 * 60 * 1000) {
  // Server-side cache expired; clear old tool results inline
  // Keeps most recent N results (default: 5)
  // Marks cleared results: "[Old tool result content cleared]"
}
```

**Cached Microcompact** (feature-gated, ant-only):
```typescript
if (feature('CACHED_MICROCOMPACT') && toolCount > threshold) {
  // Generate cache_edits payload
  // Attached to next API call — deletes tool results without breaking cache prefix
}
```

### Step 3: Build System Prompt Blocks
```
buildSystemPromptBlocks(context, options)
```
- Assemble static prefix (globally cacheable, hashed by Blake2b)
- Apply `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker
- Append dynamic sections (memory, git status, MCP instructions, etc.)
- Return array of `SystemPromptBlock` with `cache_control` where applicable

### Step 4: Record Prompt State
```
recordPromptState(querySource, system, tools, model)
```
- Hash `systemText`, `cacheControlValues`, `toolSchemas` separately
- Store per `querySource+agentId` for cache break detection
- Lazy: hashing only happens on change, not every call

### Step 5: API Call
```
withRetry(options, async (client, attempt, retryCtx) => {
  return queryModelWithStreaming(client, {
    model,
    system: systemBlocks,
    messages: normalizedMessages,
    tools: toolSchemas,
    max_tokens: maxTokens,
    ...extraBodyParams
  })
})
```

### Step 6: Stream Processing
As events arrive from the API:
- `content_block_delta` with `text_delta` → yield `StreamEvent` immediately
- `content_block_delta` with `input_json_delta` → accumulate tool JSON args
- `content_block_stop` for tool use → `StreamingToolExecutor.add(toolUse)`
- `message_delta` → capture usage (input_tokens, output_tokens, cache_*)
- `message_stop` → turn complete

### Step 7: Check Cache Break
```
checkResponseForCacheBreak(querySource, cacheReadTokens, cacheCreationTokens)
```
After each API call, compare cache token counts with previous turn. If cache read drops significantly (> 5%), log `tengu_prompt_cache_break` event with attribution.

### Step 8: Collect Tool Results
`StreamingToolExecutor` runs tools as they complete. Results are:
- Emitted as `ToolResultMessage` objects in receive order
- Appended to the message history
- Handed back to the loop for the next API call

### Step 9: Token Budget Check
```
checkTokenBudget(tracker, budgetConfig)
```
Returns `{action: 'continue', nudgeMessage}` or `{action: 'stop', completionEvent}`.

See [token-budget.md](token-budget.md) for details.

---

## Message Types Yielded by the Generator

| Type | When |
|------|------|
| `RequestStartEvent` | Before each API call |
| `StreamEvent` | Each text token from the API |
| `ToolUseMessage` | When a tool starts executing |
| `ToolResultMessage` | When a tool completes |
| `SystemMessage` | System-injected messages (e.g., from hooks) |
| `TombstoneMessage` | Replaces a message that was removed |
| `AssistantMessage` | Complete assistant turn |
| `StopHookSummaryMessage` | After stop hooks fire |

---

## Continuation Logic

The loop continues as long as:
1. The most recent assistant message ended with `stop_reason: 'tool_use'` (tools were called)
2. The token budget says `{action: 'continue'}`
3. No blocking stop hook returned `{preventContinuation: true}`
4. No user abort signal

If the model returns `stop_reason: 'end_turn'` and no tools were called, the loop exits immediately.
