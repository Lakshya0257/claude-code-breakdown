# Data Flow Diagrams

## Complete Query Lifecycle

```
User Input
    │
    ▼
┌─────────────────────────────────────────────────┐
│  REPL (screens/REPL.tsx)                        │
│  - Captures keypress via useInput               │
│  - Renders messages as StreamEvents arrive      │
└────────────────────┬────────────────────────────┘
                     │ user submits prompt
                     ▼
┌─────────────────────────────────────────────────┐
│  user-prompt-submit-hook                        │
│  (utils/hooks/execHook.ts)                      │
│  - Shell/LLM/HTTP/Agent hooks fire              │
│  - Can modify input text                        │
│  - Can block submission entirely                │
└────────────────────┬────────────────────────────┘
                     │ (if not blocked)
                     ▼
┌─────────────────────────────────────────────────┐
│  query() async generator (query.ts)             │
│  - Takes messages[], options, tools             │
│  - Yields StreamEvent | Message | Tombstone     │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  Per-turn loop iteration:                       │
│                                                 │
│  1. normalizeMessages()                         │
│     - Coerce all message types to SDK format    │
│     - Strip tool results exceeding budget       │
│     - Inject date-change attachments            │
│                                                 │
│  2. microcompactMessages() or snipMessages()    │
│     - Time-based: clear old tool results        │
│     - Cached: delete via cache_edits API        │
│                                                 │
│  3. buildSystemPromptBlocks()                   │
│     - Assemble static + dynamic sections        │
│     - Inject memory, git status, MCP info       │
│     - Apply SYSTEM_PROMPT_DYNAMIC_BOUNDARY      │
│                                                 │
│  4. recordPromptState()                         │
│     - Hash system prompt, tools for cache diff  │
│                                                 │
│  5. API call → withRetry() → queryModelWithStreaming()
│     - getAnthropicClient() for provider         │
│     - Stream events: text_delta, tool_use, etc. │
│                                                 │
│  6. checkResponseForCacheBreak()                │
│     - Compare cache tokens vs previous          │
│     - Log tengu_prompt_cache_break if dropped   │
│                                                 │
│  7. StreamingToolExecutor                       │
│     - Concurrent-safe tools run in parallel     │
│     - Non-concurrent: exclusive execution       │
│     - Results emitted in receive order          │
│                                                 │
│  8. checkTokenBudget()                          │
│     - < 90% used AND not diminishing? Continue  │
│     - Otherwise: emit completion, stop          │
└────────────────────┬────────────────────────────┘
                     │ loop ends
                     ▼
┌─────────────────────────────────────────────────┐
│  handleStopHooks() (query/stopHooks.ts)         │
│                                                 │
│  1. Fire-and-forget background tasks:           │
│     - executePromptSuggestion()                 │
│     - executeExtractMemories()                  │
│     - executeAutoDream()                        │
│                                                 │
│  2. Execute Stop hooks (user-configured)        │
│     - Collect blocking errors                   │
│     - Check preventContinuation flags           │
│                                                 │
│  3. If teammate: TaskCompleted + TeammateIdle   │
│                                                 │
│  4. Emit StopHookSummaryMessage                 │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────┐
│  autoCompactIfNeeded()                          │
│  - Check token count vs threshold               │
│  - Trigger session memory or full compaction    │
└────────────────────┬────────────────────────────┘
                     │
                     ▼
                 REPL re-renders
```

---

## Tool Execution Data Flow

```
API response → tool_use content block
    │
    ▼
StreamingToolExecutor.add(toolUse)
    │
    ├─ isConcurrencySafe = true?
    │      YES → add to parallel queue
    │      NO  → wait for exclusive slot
    │
    ▼
Tool.checkPermissions(input, context)
    │
    ├─ 'allow' → call tool immediately
    ├─ 'ask'   → render PermissionRequest UI → wait for user
    └─ 'deny'  → return error tool_result
    │
    ▼ (if allowed)
PreToolUse hooks fire
    │
    ├─ hook output modifies input?
    │      YES → use modified input
    │      NO  → use original
    │
    ▼
Tool.call(input, context, canUseTool, parentMessage, onProgress)
    │
    ├─ Progress callbacks → REPL renders spinner/progress
    │
    ▼
Tool result returned
    │
    ▼
PostToolUse hooks fire
    │
    ▼
Tool result injected into messages as tool_result block
    │
    ▼
Next API call in loop (with tool results included)
```

---

## Streaming Render Data Flow

```
API SDK Stream<RawMessageStreamEvent>
    │
    ▼
Events processed in order:
  content_block_start → new block started
  content_block_delta → text/json appended
  content_block_stop  → block complete
  message_delta       → usage updated
  message_stop        → turn complete
    │
    ├─ Text blocks → yield StreamEvent immediately
    │                 REPL renders token by token
    │
    └─ Tool blocks → accumulate JSON args
                     On block_stop: add to executor
                     Executor runs tool
                     Yield ToolResultMessage when done
```

---

## Memory Injection Data Flow

```
Session start
    │
    ▼
loadMemoryPrompt()
    │
    ├─ Read ~/.claude/projects/<slug>/memory/MEMORY.md
    ├─ Build memory system instructions
    └─ Inject into system prompt dynamic section
    │
    ▼
Each query turn:
    │
    ▼
startRelevantMemoryPrefetch(userQuery)
    │
    ├─ Scan memory dir for .md files
    ├─ Call Sonnet selector with query + manifest
    └─ Return up to 5 relevant memory paths
    │
    ▼
getAttachmentMessages(relevantPaths)
    │
    ├─ Read each memory file
    ├─ Apply token budget (50K total, 5K per file)
    └─ Return as AttachmentMessage blocks
    │
    ▼
filterDuplicateMemoryAttachments()
    │
    └─ Deduplicate vs previous turns
    │
    ▼
Prepend to messages[] before API call
```

---

## Compaction Decision Tree

```
Before API call:
    │
    ▼
Is time-based microcompact needed?
  (gap > 60 min since last assistant message)
    │
    ├─ YES → Clear old tool results inline
    │
    └─ NO  → Is cached microcompact needed?
              (tool count > threshold AND feature enabled)
                  │
                  ├─ YES → Generate cache_edits payload
                  │         Send with next API call
                  │
                  └─ NO  → Continue normally

After API call / before next turn:
    │
    ▼
Token count > autoCompactThreshold?
  (contextWindow - 13,000)
    │
    ├─ YES → Try session memory compaction first
    │           │
    │           ├─ Applicable? (>10K tokens, >5 text messages)
    │           │     YES → Summarize into session_memory.md
    │           │            Keep recent messages
    │           │
    │           └─ NOT applicable → Full compaction
    │                               Summarize everything
    │                               Reset message history
    │
    └─ NO  → Continue
```
