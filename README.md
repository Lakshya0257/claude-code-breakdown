# Claude Code: Under the Hood

A deep-dive architectural review of Claude Code. While it presents itself as a simple terminal chat interface, the source code reveals a sophisticated 11-layer multi-agent orchestration platform.

This document translates those source code mechanics into actionable insights for power users, explaining how to properly leverage context, parallelism, and system hooks.

---

## 1. The Context Engine (`CLAUDE.md`)

The system does not treat your configuration as a one-time startup load.

The source code reveals that `CLAUDE.md` files are loaded and parsed on **every single query iteration**. This turn-based injection means you can update your rules dynamically mid-session, and the model will immediately adapt.

The architecture enforces a strict hierarchy for context gathering, allowing up to 40,000 characters:

- `~/.claude/CLAUDE.md`: Global preferences (e.g., universal coding styles).
- `./CLAUDE.md`: Project-level laws (e.g., architecture decisions, framework conventions).
- `.claude/rules/*.md`: Modular, specific rule sets.
- `CLAUDE.local.md`: Private, git-ignored notes.

## 2. Zero-Cost Parallelism (Prompt Caching)

Using Claude Code for one sequential task at a time drastically underutilizes the system.

When a subagent is forked, it creates a byte-identical copy of the parent context. Because the underlying API aggressively caches this identical context, spawning multiple agents simultaneously is highly cost-efficient.

The execution models for these agents include:

- **Fork:** Inherits the parent context directly, optimized for cache hits.
- **Teammate:** Operates in a separate terminal pane (tmux/iTerm), communicating via a file-based mailbox system.
- **Worktree:** Automatically provisions an isolated Git worktree and branch for the agent to safely execute risky refactors.

## 3. The Permission Cascade

Manual approval dialogs are a symptom of under-configuration. The permission system evaluates actions through a 5-level priority cascade:
`Policy > Flag > Local > Project > User`

By defining glob patterns in `~/.claude/settings.json`, you can permanently bypass prompts for trusted directories or commands. The system supports three distinct permission modes:

- **Bypass:** Disables all checks (maximum speed, high risk).
- **AllowEdits:** Automatically permits file modifications within the active working directory.
- **Auto:** Routes actions through a parallelized LLM classifier to intelligently allow or deny based on context.

## 4. Context Pressure & Compaction Strategies

Managing the 200K (or optional 1M via `[1m]` suffix) token window is a core focus of the architecture. To prevent context overflow, the system utilizes five distinct compaction strategies:

- **Microcompact:** Time-based pruning of stale tool results.
- **Context Collapse:** Summarizes specific spans of older conversation.
- **Session Memory:** Extracts and saves critical workflow states, errors, and task specs to a persistent file.
- **Full Compact:** A complete summarization of the entire history.
- **PTL Truncation:** Drops the oldest message groups entirely when limits are breached.

**Best Practice:** Proactively use the `/compact` command to preserve critical context before the system is forced to auto-truncate. Be aware that massive files pasted into the terminal are often truncated to an 8KB preview; use targeted inputs instead.

## 5. The Extension API (Lifecycle Hooks)

The most powerful extensibility feature is the hook system, exposing over 25 lifecycle events. You can attach 5 types of hooks (command, prompt, agent, HTTP, function) to events like:

- `PreToolUse` & `PostToolUse`: Ideal for injecting auto-linters or running test suites around file writes.
- `UserPromptSubmit`: Allows for automatic context injection (like appending `git status` or recent logs) every time you hit Enter.
- `SessionStart` & `SessionEnd`: Perfect for environment bootstrapping or teardown.

## 6. Session Persistence & Resumability

Conversations are not ephemeral. Every session is serialized as JSONL at `~/.claude/projects/{hash}/{sessionId}.jsonl`.

Because of the "Session Memory" compaction strategy mentioned above, long-running sessions accumulate highly valuable context regarding project state, past errors, and learned conventions.

- Use `--continue` to resume the last session and leverage accumulated knowledge.
- Use `--resume` to select a specific historical session.
- Use `--fork-session` to branch off a previous conversation without altering its history.

## 7. Smart Tool Batching & Deferred MCP Loading

Claude Code handles over 60 built-in tools by automatically partitioning them:

- **Concurrent:** Read-only operations (globbing, searching, reading) are batched and executed in parallel.
- **Serial:** Mutating operations (writing files, bash commands) are queued sequentially to prevent race conditions and merge conflicts.

External capabilities are added via the Model Context Protocol (MCP). The architecture uses **deferred loading** for MCP servers—tools are only loaded into memory when contextually required, meaning you can connect multiple external systems (databases, CI/CD pipelines) with zero idle performance penalty.

## 8. Streaming Architecture & Interruption

The execution pipeline is built on asynchronous generators yielding individual events.
This means you can cleanly interrupt the model (via Escape) the moment you see it hallucinating or taking the wrong path. The stream aborts instantly, the interrupted response is cleanly discarded, and your previous context remains intact with zero penalty.

## 9. The Self-Healing Retry System

The engine is built for unattended execution. The source code outlines a robust error-handling infrastructure:

- 10 automatic retries utilizing exponential backoff and jitter (500ms base).
- Native OAuth token refresh handling for 401/403 errors.
- Automatic model fallback (e.g., from Opus to Sonnet) after consecutive 529 API errors.
- A 90-second watchdog timer that transitions stalled streams into non-streaming modes.

---

## 📚 Repository Roadmap

_This repository is a living document. The following sections will be elaborated upon with complete workflows, code examples, and best practices in upcoming commits. Navigate to the respective files as they become available._

- **[Module 1: Advanced CLAUDE.md Templates]** - Structuring architectural guardrails.
- **[Module 2: Subagent Patterns]** - Workflows for multi-agent security audits and refactors.
- **[Module 3: The Hook Gallery]** - Ready-to-use Bash and JS scripts for lifecycle events.
- **[Module 4: Persistence Engineering]** - Maximizing the Session Memory state.
- **[Module 5: MCP Integrations]** - Connecting external databases and cloud providers.
