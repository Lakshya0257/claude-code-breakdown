# Streaming Architecture

How events flow from the Anthropic API through to the terminal UI.

---

## API Stream Events

The Anthropic SDK returns `Stream<RawMessageStreamEvent>`. Claude Code processes these event types:

| Event Type | Data | Action |
|------------|------|--------|
| `content_block_start` | block type, index | Begin new text or tool block |
| `content_block_delta` (text_delta) | text string | Yield `StreamEvent` immediately to REPL |
| `content_block_delta` (input_json_delta) | partial JSON | Accumulate tool args character by character |
| `content_block_stop` | block index | If tool block: schedule execution |
| `message_delta` | stop_reason, usage | Capture token counts |
| `message_stop` | — | Turn complete |

---

## Text Streaming

Text tokens are yielded immediately as they arrive:

```typescript
// Each text_delta event:
yield {
  type: 'stream_event',
  text: delta.text,
  blockIndex: event.index,
}
```

The REPL component receives these and re-renders, showing text appearing word by word.

---

## Tool Block Streaming

Tool use blocks have their JSON arguments streamed character by character:

```typescript
// Accumulate in toolUseAccumulator:
if (event.type === 'content_block_delta' && delta.type === 'input_json_delta') {
  toolUseAccumulator[blockIndex] += delta.partial_json
}

// On content_block_stop:
if (block.type === 'tool_use') {
  const args = JSON.parse(toolUseAccumulator[blockIndex])
  executor.add({
    id: block.id,
    name: block.name,
    input: args,
  })
}
```

---

## StreamingToolExecutor

`StreamingToolExecutor` manages concurrent tool execution:

```typescript
class StreamingToolExecutor {
  private concurrentQueue: ToolExecution[] = []
  private exclusiveQueue: ToolExecution[] = []
  private runningExclusive: boolean = false

  add(toolUse: ToolUse) {
    const tool = findTool(toolUse.name)

    if (tool.isConcurrencySafe()) {
      // Start immediately in parallel with others
      this.startConcurrent(toolUse, tool)
    } else {
      // Queue for exclusive (sequential) execution
      this.queueExclusive(toolUse, tool)
    }
  }
}
```

### Concurrent-Safe Tools
Run in parallel with each other and with the API stream:
- FileRead, Glob, Grep, WebSearch, WebFetch
- TaskCreate/Get/Update/List
- AskUserQuestion, Sleep
- ToolSearch

### Exclusive Tools
Wait for all running tools to finish, then run alone:
- Bash, FileEdit, FileWrite, Agent, Skill
- EnterPlanMode/ExitPlanMode
- Any tool that modifies state

### Result Ordering

Results are emitted in the **order they were received** (not completion order):

```typescript
// Example: tools A, B, C received at t=0, t=1, t=2
// B finishes at t=3, A finishes at t=5, C finishes at t=4
// Emitted order: A-result, B-result, C-result (original receive order)
```

This ensures the model sees tool results in the same order it called the tools.

---

## REPL Rendering Pipeline

```
StreamEvent arrives
    │
    ▼
VirtualMessageList receives event
    │
    ▼
Message component renders:
  - AssistantMessage → rendered text with markdown
  - ToolUseMessage   → spinner + tool name + args preview
  - ToolResultMessage → collapsible result block
    │
    ▼
React reconciler → Yoga layout → render-node-to-output
    │
    ▼
Frame diff vs prev screen → ANSI escape codes
    │
    ▼
Write to stdout
```

---

## Abort / Interruption

When Ctrl+C is pressed:

1. `AbortController.abort()` is called on the `context.abortController`
2. The API SDK stream check `signal.aborted` on next iteration and throws `AbortError`
3. Any running tools check `signal.aborted` via their context
4. `stopHooks.ts` catches the abort signal and yields `createUserInterruptionMessage()`
5. The loop exits cleanly

Tools that can be interrupted mid-execution (Bash) use `signal.aborted` polling in their execution loop.
