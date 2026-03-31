# Claude Code Internals — Complete Technical Reference

This is a comprehensive "book" of the Claude Code codebase — every subsystem, every data flow, every prompt, every algorithm. It is derived from reading the actual source code of the 1,903-file TypeScript codebase.

---

## What is Claude Code?

Claude Code is Anthropic's official CLI for Claude. It is a full agentic coding assistant that:
- Runs in your terminal (or IDE via extension)
- Executes tools (Bash, file read/write/edit, web search, etc.) autonomously
- Manages multi-turn conversations with the Claude API
- Supports multi-agent orchestration (swarms, teammates, subagents)
- Integrates with MCP (Model Context Protocol) servers
- Manages context window compaction automatically
- Persists memory across sessions

---

## How to Read This Book

Each chapter is a folder. Each folder has a `README.md` summarizing the chapter, and individual files for each subsystem.

```
claude-code-internals/
├── README.md                        ← You are here
├── 01-architecture/                 ← Big-picture overview
├── 02-startup-and-cli/              ← How the process starts
├── 03-query-engine/                 ← The core agent loop
├── 04-api-layer/                    ← Anthropic API integration
├── 05-system-prompts/               ← All prompts sent to the model
├── 06-context-management/           ← Token counting, compaction
├── 07-tools/                        ← Every tool (Bash, Read, Edit, Agent, ...)
├── 08-permissions/                  ← Permission modes, rule matching, safety
├── 09-hooks/                        ← Lifecycle hooks (PreToolUse, Stop, ...)
├── 10-memory/                       ← Persistent memory across sessions
├── 11-mcp/                          ← Model Context Protocol integration
├── 12-multi-agent/                  ← Subagents, swarms, coordinator
├── 13-skills/                       ← Skills and slash commands
├── 14-state-management/             ← AppState, store, settings
├── 15-ui-and-rendering/             ← Terminal UI (custom React renderer)
├── 16-analytics-and-telemetry/      ← Event tracking, OTEL
├── 17-plugins/                      ← Plugin system
└── 18-bridge-and-remote/            ← Remote sessions, CCR bridge
```

---

## End-to-End Query Lifecycle (30-second summary)

```
1. main.tsx starts → migrations → prefetch git/auth
2. REPL renders → user types prompt
3. user-prompt-submit-hook fires (can block/modify)
4. query() async generator starts
5. Messages normalized, memory injected, git context added
6. Microcompact or snip compaction if needed
7. System prompt assembled (static cached + dynamic session parts)
8. Anthropic API called (streaming)
9. Text blocks streamed → rendered token by token
10. Tool blocks assembled → StreamingToolExecutor runs tools concurrently
11. Tool results injected back → loop continues
12. post-sampling-hooks fire (transcript classifier for auto mode)
13. Token budget checked → continue or stop
14. turn-complete-hook fires (memory extraction, analytics, prompt suggestion)
15. REPL re-renders final response
```

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Source files | 1,903 |
| Source directories | 180+ |
| Tool implementations | 45+ |
| Slash commands | 80+ |
| Migration versions | 11 |
| Max tool result size | 30,000–100,000 chars |
| Context window auto-compact threshold | `contextWindow - 13,000 tokens` |
| Session memory init threshold | 50,000 tokens |
| Max hook timeout | 60s (agent hooks) |
| Max retries on API error | 10 |
| Cache TTL options | 5 min or 1 hour |

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Bun (not Node.js) |
| UI framework | React 19 + custom Ink renderer |
| Layout engine | Yoga (WASM) |
| API client | Anthropic SDK (TypeScript) |
| Type system | TypeScript with Zod validation |
| Feature flags | GrowthBook |
| Observability | OpenTelemetry + Datadog + BigQuery |
| Shell parsing | tree-sitter-bash (WASM, ant-only) |
| Text search | ripgrep |
| Auth | OAuth 2.0 + keychain storage |
