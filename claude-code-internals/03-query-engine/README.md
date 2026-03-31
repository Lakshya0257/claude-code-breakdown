# Chapter 3: Query Engine

The query engine is the heart of Claude Code — the async generator loop that drives every turn of conversation.

## Files in This Chapter

- [query-loop.md](query-loop.md) — The core loop: normalization, compaction, API call, tool execution
- [token-budget.md](token-budget.md) — Token consumption tracking and continuation decisions
- [stop-hooks.md](stop-hooks.md) — Post-turn lifecycle: memory extraction, hooks, analytics
- [streaming.md](streaming.md) — How streaming events flow from API to terminal
- [retry-logic.md](retry-logic.md) — Retry strategies, fast mode fallback, context overflow
