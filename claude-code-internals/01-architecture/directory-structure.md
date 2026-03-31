# Directory Structure

Complete breakdown of every source directory in `src/`.

```
src/
├── main.tsx                    ← CLI entry point (~4,700 lines)
├── Tool.ts                     ← Base Tool interface and ToolUseContext
├── commands.ts                 ← Commander.js command registration helper
├── history.ts                  ← REPL input history

├── assistant/
│   └── sessionHistory.ts       ← Session event pagination via CCR API

├── bootstrap/                  ← Startup state (session ID, model, agent type)

├── bridge/                     ← Remote control / CCR session bridge
│   ├── bridgeMain.ts           ← Main bridge polling loop
│   ├── bridgeApi.ts            ← REST client for bridge API
│   ├── sessionRunner.ts        ← Child process spawning
│   ├── replBridge.ts           ← REPL bridge for interactive sessions
│   └── ...31 files total

├── buddy/                      ← "Companion" feature (pet/reaction system)

├── cli/                        ← CLI infrastructure
│   ├── handlers/               ← auth, agents, autoMode, mcp, plugins
│   ├── transports/             ← SSE, WebSocket, Hybrid, CCR client
│   ├── print.ts                ← Formatted output
│   ├── structuredIO.ts         ← JSON/NDJSON output
│   ├── remoteIO.ts             ← Remote I/O
│   └── exit.ts                 ← Process exit

├── commands/                   ← 80+ slash command implementations
│   ├── add-dir/                ← Add directory to allowed paths
│   ├── agents/                 ← Agent management
│   ├── compact/                ← Context compaction
│   ├── config/                 ← Settings management
│   ├── context/                ← Show context usage
│   ├── cost/                   ← Token cost tracking
│   ├── diff/                   ← File diff view
│   ├── doctor/                 ← Health diagnostics
│   ├── effort/                 ← Effort estimation
│   ├── export/                 ← Session export
│   ├── help/                   ← Help display
│   ├── hooks/                  ← Hook management
│   ├── login/logout/           ← Auth
│   ├── mcp/                    ← MCP server management
│   ├── memory/                 ← Memory editor
│   ├── model/                  ← Model selection
│   ├── plan/                   ← Plan mode
│   ├── plugins/                ← Plugin management
│   ├── resume/                 ← Resume previous session
│   ├── review/                 ← Code review
│   ├── rewind/                 ← Session rewind
│   ├── skills/                 ← Skills list/install
│   ├── stats/                  ← Statistics
│   ├── tasks/                  ← Task management
│   ├── teleport/               ← Remote teleport
│   ├── theme/                  ← Theme selection
│   ├── usage/                  ← Usage display
│   ├── vim/                    ← Vim mode toggle
│   └── ...80+ total

├── components/                 ← React/Ink UI components
│   ├── agents/                 ← Agent creation wizard
│   ├── design-system/          ← Pane (styled box)
│   ├── diff/                   ← Unified diff viewer
│   ├── mcp/                    ← MCP server UI, elicitation dialogs
│   ├── memory/                 ← Memory editor UI
│   ├── messages/               ← Message rendering components
│   ├── permissions/            ← Tool use confirmation dialogs
│   │   ├── BashPermissionRequest/
│   │   ├── FileEditPermissionRequest/
│   │   ├── FileWritePermissionRequest/
│   │   └── ...10 types
│   ├── PromptInput/            ← Text input with modes
│   ├── skills/                 ← Skills UI
│   ├── tasks/                  ← Task list UI
│   ├── Spinner/                ← Loading indicator
│   ├── HighlightedCode/        ← Syntax highlighted code
│   ├── StructuredDiff/         ← Structured diff display
│   └── VirtualMessageList/     ← Virtualized message scroll

├── constants/                  ← Constants and system prompts
│   ├── prompts.ts              ← Main system prompt assembly (~800KB)
│   ├── common.ts               ← Date utilities, session start date
│   ├── system.ts               ← CLI sysprompt prefixes, attribution header
│   └── xml.ts                  ← XML tag constants (<bash-input> etc.)

├── context/                    ← React context providers
│   └── mailbox.tsx             ← Inter-component message passing

├── coordinator/                ← Coordinator/orchestrator mode
│   └── coordinatorMode.ts

├── entrypoints/                ← Process entry points
│   ├── sdk/                    ← SDK mode entry
│   └── cli.tsx                 ← CLI entry (fast-paths for --version etc.)

├── hooks/                      ← React hooks (UI-level)
│   ├── notifs/                 ← Notification hooks
│   └── toolPermission/         ← Permission request hooks
│       └── handlers/           ← Per-tool permission handlers

├── ink/                        ← Custom React terminal renderer
│   ├── reconciler.ts           ← React 19 reconciler
│   ├── renderer.ts             ← Output generation pipeline
│   ├── screen.ts               ← Terminal cell buffer
│   ├── output.ts               ← Rendering operations accumulator
│   ├── render-node-to-output.ts← DOM → screen
│   ├── optimizer.ts            ← Frame diffing
│   ├── layout/                 ← Yoga WASM layout
│   ├── events/                 ← Event dispatcher, keyboard/mouse
│   ├── termio/                 ← ANSI escape code parser
│   ├── hooks/                  ← useInput, useStdin, useTerminalSize, etc.
│   └── components/             ← Box, Text, ScrollBox, Link, Button

├── keybindings/                ← Keyboard shortcut system
│   ├── defaultBindings.ts      ← All default key bindings
│   ├── parser.ts               ← "ctrl+x ctrl+e" → parsed chord
│   ├── resolver.ts             ← Hierarchical context resolution
│   └── validator.ts            ← Conflict detection

├── memdir/                     ← Persistent memory system
│   └── (memory loading, MEMORY.md management)

├── migrations/                 ← Version migration scripts (11 total)
│   ├── migrateAutoUpdatesToSettings.ts
│   ├── migrateFennecToOpus.ts
│   ├── migrateSonnet45ToSonnet46.ts
│   └── ...

├── native-ts/                  ← Native TypeScript wrappers
│   ├── color-diff/             ← Color-based diff
│   ├── file-index/             ← File indexing
│   └── yoga-layout/            ← Yoga layout bindings

├── outputStyles/               ← Custom output style definitions

├── plugins/                    ← Plugin system
│   ├── builtinPlugins.ts       ← Built-in plugin registry
│   └── bundled/                ← Bundled plugin assets

├── query/                      ← Core query engine
│   ├── tokenBudget.ts          ← Token consumption tracking
│   ├── stopHooks.ts            ← Post-turn lifecycle
│   ├── config.ts               ← Immutable query configuration
│   └── deps.ts                 ← Dependency injection interface

├── remote/                     ← Remote session management
│   └── RemoteSessionManager.ts ← WebSocket connection + message routing

├── schemas/                    ← Zod/JSON schemas
│   └── hooks.ts                ← Hook configuration schemas

├── screens/                    ← Top-level screen components
│   ├── REPL.tsx                ← Main interactive REPL
│   ├── Doctor.tsx              ← Health check screen
│   └── ResumeConversation.tsx  ← Session resume UI

├── services/                   ← Business logic services
│   ├── AgentSummary/           ← Agent result summarization
│   ├── analytics/              ← Event logging (Datadog, BigQuery)
│   ├── api/                    ← Anthropic API integration
│   │   ├── claude.ts           ← API call, prompt caching
│   │   ├── client.ts           ← SDK client factory
│   │   ├── withRetry.ts        ← Retry logic
│   │   ├── promptCacheBreakDetection.ts
│   │   ├── dumpPrompts.ts      ← Debug request logging
│   │   └── errors.ts           ← Error formatting
│   ├── autoDream/              ← Auto memory consolidation
│   ├── compact/                ← Compaction strategies
│   ├── extractMemories/        ← Memory extraction from conversation
│   ├── lsp/                    ← Language Server Protocol
│   ├── MagicDocs/              ← Magic documentation feature
│   ├── mcp/                    ← MCP protocol implementation
│   ├── oauth/                  ← OAuth token management
│   ├── plugins/                ← Plugin loading and execution
│   ├── policyLimits/           ← Policy enforcement
│   ├── PromptSuggestion/       ← Background prompt suggestions
│   ├── remoteManagedSettings/  ← CCR-pushed settings
│   ├── SessionMemory/          ← Auto session notes
│   ├── settingsSync/           ← Settings sync across devices
│   ├── teamMemorySync/         ← Team memory synchronization
│   ├── tips/                   ← User tips system
│   ├── tools/                  ← Tool service utilities
│   └── toolUseSummary/         ← Tool use summaries

├── skills/                     ← Skill definitions
│   └── bundled/                ← Built-in skills (verify, debug, loop, etc.)

├── state/                      ← Application state management
│   ├── AppStateStore.ts        ← AppState type definition
│   ├── store.ts                ← Store implementation
│   ├── onChangeAppState.ts     ← Reactive state-change handlers
│   └── selectors.ts            ← Pure state selectors

├── tasks/                      ← Task execution framework
│   ├── LocalAgentTask/         ← In-process agent tasks
│   ├── RemoteAgentTask/        ← CCR remote agent tasks
│   ├── DreamTask/              ← Memory consolidation task
│   ├── InProcessTeammateTask/  ← Swarm teammate tasks
│   └── LocalShellTask/         ← Background shell tasks

├── tools/                      ← Tool implementations
│   ├── AgentTool/              ← Agent spawning tool
│   │   └── built-in/           ← Built-in agent definitions
│   ├── AskUserQuestionTool/
│   ├── BashTool/
│   ├── ConfigTool/
│   ├── EnterPlanModeTool/
│   ├── ExitPlanModeTool/
│   ├── EnterWorktreeTool/
│   ├── ExitWorktreeTool/
│   ├── FileEditTool/
│   ├── FileReadTool/
│   ├── FileWriteTool/
│   ├── GlobTool/
│   ├── GrepTool/
│   ├── LSPTool/
│   ├── MCPTool/
│   ├── NotebookEditTool/
│   ├── PowerShellTool/
│   ├── REPLTool/
│   ├── RemoteTriggerTool/
│   ├── ScheduleCronTool/
│   ├── SendMessageTool/
│   ├── SkillTool/
│   ├── SleepTool/
│   ├── TaskCreateTool/
│   ├── TaskGetTool/
│   ├── TaskListTool/
│   ├── TaskOutputTool/
│   ├── TaskStopTool/
│   ├── TaskUpdateTool/
│   ├── TeamCreateTool/
│   ├── TodoWriteTool/
│   ├── ToolSearchTool/
│   ├── WebFetchTool/
│   ├── WebSearchTool/
│   └── shared/

├── types/                      ← TypeScript type definitions
│   └── generated/              ← Generated protobuf types for analytics

├── upstreamproxy/              ← HTTP proxy support

├── utils/                      ← Utility modules
│   ├── background/             ← Background task utilities
│   │   └── remote/             ← Remote precondition checks
│   ├── bash/                   ← Bash parsing (tree-sitter)
│   ├── deepLink/               ← Deep link handling
│   ├── dxt/                    ← DXT format support
│   ├── filePersistence/        ← File-based persistence
│   ├── git/                    ← Git utilities
│   ├── github/                 ← GitHub API utilities
│   ├── hooks/                  ← Hook execution utilities
│   ├── mcp/                    ← MCP utilities
│   ├── memory/                 ← Memory file utilities
│   ├── messages/               ← Message normalization/mapping
│   ├── model/                  ← Model configuration
│   ├── permissions/            ← Permission system
│   ├── plugins/                ← Plugin utilities
│   ├── processUserInput/       ← User input processing
│   ├── sandbox/                ← Sandbox configuration
│   ├── secureStorage/          ← Keychain access
│   ├── settings/               ← Settings loading
│   │   └── mdm/                ← MDM/enterprise settings
│   ├── shell/                  ← Shell utilities
│   ├── skills/                 ← Skill loading utilities
│   ├── suggestions/            ← Prompt suggestions
│   ├── swarm/                  ← Multi-agent swarm
│   │   └── backends/           ← tmux, iTerm2, in-process
│   ├── task/                   ← Task framework
│   ├── telemetry/              ← OpenTelemetry
│   ├── teleport/               ← Remote teleport API
│   ├── todo/                   ← Todo list utilities
│   └── ultraplan/              ← Ultraplan feature

├── vim/                        ← Vim keybinding mode
│   ├── types.ts                ← State machine types
│   ├── motions.ts              ← Movement commands
│   ├── operators.ts            ← Operation commands
│   ├── textObjects.ts          ← Text objects (word, bracket, etc.)
│   └── transitions.ts          ← State machine transitions

└── voice/                      ← Voice mode
    └── voiceModeEnabled.ts     ← Feature gate check
```
