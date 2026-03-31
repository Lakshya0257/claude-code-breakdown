# Token Budget

`src/query/tokenBudget.ts` manages per-turn token consumption and decides whether to continue the loop or stop.

---

## BudgetTracker

```typescript
type BudgetTracker = {
  continuationCount: number    // How many times we've continued
  lastDeltaTokens: number      // Tokens used in last continuation
  lastGlobalTurnTokens: number // Total tokens at last check
  startedAt: number            // Timestamp for duration calculation
}

function createBudgetTracker(): BudgetTracker {
  return {
    continuationCount: 0,
    lastDeltaTokens: 0,
    lastGlobalTurnTokens: 0,
    startedAt: Date.now(),
  }
}
```

The tracker is mutable and passed through the loop. It accumulates state across continuations.

---

## Decision Logic

```typescript
type TokenBudgetDecision =
  | { action: 'continue'; nudgeMessage?: string; /* updated tracker */ }
  | { action: 'stop'; completionEvent: CompletionEvent }

const COMPLETION_THRESHOLD = 0.9  // Stop at 90% of budget
const DIMINISHING_THRESHOLD = 500 // < 500 token increment = diminishing returns
```

### `checkTokenBudget(tracker, budgetConfig)`

```
IF budgetConfig is null or <= 0:
  RETURN 'stop' (no budget configured)

IF agent context exists:
  RETURN 'stop' (let agent handle its own budget)

current_tokens = get current cumulative tokens
delta_tokens = current_tokens - tracker.lastGlobalTurnTokens

// Diminishing returns detection:
IF continuationCount >= 3
   AND current_tokens < COMPLETION_THRESHOLD * budget
   AND delta_tokens < DIMINISHING_THRESHOLD
   AND lastDeltaTokens < DIMINISHING_THRESHOLD:
  RETURN 'stop' (diminishing returns, not making progress)

IF current_tokens >= COMPLETION_THRESHOLD * budget:
  RETURN 'stop' (90% of budget consumed)

// Otherwise continue:
tracker.continuationCount++
tracker.lastDeltaTokens = delta_tokens
tracker.lastGlobalTurnTokens = current_tokens
RETURN 'continue' with optional nudge message
```

### Key thresholds:

| Condition | Result |
|-----------|--------|
| 90%+ of budget consumed | STOP |
| 3+ continuations + <500 tokens each time | STOP (diminishing returns) |
| Token budget is null/0 | STOP |
| Agent context exists | STOP (agent manages own budget) |
| Otherwise | CONTINUE |

---

## Nudge Messages

When returning `'continue'`, the decision may include a `nudgeMessage` injected into the next prompt to guide the model toward completion:

- When budget is getting low (70-90% used): a reminder to wrap up
- Injected as a system message before the next API call

---

## Duration Calculation

On STOP, `completionEvent` includes:
```typescript
{
  durationMs: Date.now() - tracker.startedAt,
  continuationCount: tracker.continuationCount,
  // ...other usage stats
}
```

This is logged for analytics to understand how much work each query turn does.

---

## Context Window vs Token Budget

Two separate limits:

1. **Context window** — physical limit of the model (e.g., 200K tokens). Managed by the compaction system (see [Chapter 6](../06-context-management/compaction-strategies.md)).

2. **Token budget** — user-specified budget per turn (feature-gated: `TOKEN_BUDGET`). Managed by `tokenBudget.ts`. Prevents a single turn from consuming too much.

These are independent: you can hit the token budget before the context window, or vice versa.
