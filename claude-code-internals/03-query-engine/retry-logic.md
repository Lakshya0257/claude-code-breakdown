# Retry Logic

`src/services/api/withRetry.ts` implements all retry behavior for API calls — exponential backoff, fast mode fallback, context overflow recovery, and persistent retry mode.

---

## Retry Options

```typescript
type RetryOptions = {
  maxRetries: number              // Default: 10
  model: string
  fallbackModel?: string          // Model to use after 3 consecutive 529s
  thinkingConfig?: ThinkingConfig
  signal?: AbortSignal
  querySource?: string            // For background vs foreground classification
  initialConsecutive529Errors?: number  // Pre-seed for streaming→non-streaming fallback
}
```

---

## Core Retry Loop

```
FOR attempt = 1 to maxRetries + 1:
  IF need fresh client (previous 401/403/stale):
    getAnthropicClient() fresh

  TRY:
    result = await operation(client, attempt, retryCtx)
    RETURN result (success)

  CATCH error:
    handle fast mode cooldown/fallback
    check consecutive 529 count for model fallback
    apply backoff delay
    CONTINUE if retryable
    THROW if not retryable
```

---

## Backoff Formula

```typescript
function getRetryDelay(attempt: number, maxDelayMs: number): number {
  const baseDelay = Math.min(500 * Math.pow(2, attempt - 1), maxDelayMs)
  const jitter = Math.random() * 0.25 * baseDelay
  return baseDelay + jitter
}
```

| Attempt | Base Delay | Max Jitter |
|---------|-----------|-----------|
| 1 | 500ms | 125ms |
| 2 | 1000ms | 250ms |
| 3 | 2000ms | 500ms |
| 4 | 4000ms | 1000ms |
| 5 | 8000ms | 2000ms |
| 6+ | 16000ms (capped) | 4000ms |

---

## Retryable vs Non-Retryable

| Status | Retryable? | Notes |
|--------|-----------|-------|
| 429 (rate limit) | Conditional | Only for non-Claude.ai subscribers OR enterprise |
| 529 | Foreground only | Background queries bail immediately |
| 5xx | Yes | All server errors |
| 408, 409 | Yes | Timeout/conflict |
| 401, 403 | Yes (auth retry) | Clear credential cache, get fresh client |
| 4xx other | No | Client errors |

---

## Fast Mode Fallback

When on fast mode (Opus 4.6 with faster output):

```typescript
ON 429 or 529:
  check retryAfter header

  IF retryAfter < 20 seconds:
    retry with fast mode still active

  IF retryAfter >= 20 seconds:
    trigger cooldown: switch to standard model
    log tengu_fast_mode_cooldown

ON "Fast mode not enabled" error:
  permanently disable fast mode for this session
```

---

## Consecutive 529 → Model Fallback

```typescript
if (consecutive529Count >= 3 && fallbackModel) {
  // Switch to fallback model (e.g., from Opus to Sonnet)
  // Throw FallbackTriggeredError to signal the switch
  // Caller catches and restarts with new model
}
```

---

## Context Overflow Recovery

When the API returns a context overflow error:

```typescript
// Error regex: "input length and `max_tokens` exceed context limit: (\d+) \+ (\d+) > (\d+)"
function parseMaxTokensContextOverflowError(error): ParsedOverflow | null {
  // Returns: { inputLength, maxTokens, contextLimit }
}

// Recovery:
const available = contextLimit - inputLength
const newMaxTokens = Math.max(FLOOR_OUTPUT_TOKENS, available, thinkingBudget + 1)
// Retry with reduced maxTokensOverride
```

This handles cases where the context window is too small for the requested `max_tokens`.

---

## Persistent Retry Mode

For unattended/background execution (`CLAUDE_CODE_UNATTENDED_RETRY` env + `UNATTENDED_RETRY` feature):

```typescript
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000   // 5 minutes max per backoff
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000 // 6 hours total

// Behavior:
// - Retries 429/529 indefinitely (no maxRetries cap)
// - Yields keep-alive messages every 30 seconds
// - Honors window-based reset timestamps from anthropic-ratelimit-unified-reset header
// - Caps individual backoff at 5 minutes
// - Caps total wait at 6 hours then gives up
```

Keep-alive messages prevent the session from being marked as idle by monitoring systems.

---

## `CannotRetryError` and `FallbackTriggeredError`

Two special error classes that signal specific outcomes to the caller:

```typescript
class CannotRetryError extends Error {
  // Used when all retries exhausted — no more attempts possible
}

class FallbackTriggeredError extends Error {
  constructor(public readonly fallbackModel: string) { ... }
  // Used when consecutive 529s trigger model downgrade
}
```

The caller catches `FallbackTriggeredError` and restarts the query with `fallbackModel`.
