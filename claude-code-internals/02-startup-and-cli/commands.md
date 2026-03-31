# Slash Commands

Claude Code has 80+ slash commands. They live in `src/commands/`, with each command in its own subdirectory.

---

## Command Architecture

Each command directory contains:
- `index.ts`: Command definition (name, description, type, loader function)
- Implementation file(s): The actual command logic

### Command Types

| Type | Description |
|------|-------------|
| `local` | Synchronous, runs in-process |
| `local-jsx` | Returns a React/Ink JSX component (interactive) |
| `remote` | Server-side command |
| `remote-jsx` | Remote JSX component |

Commands are lazy-loaded: the `loader` function dynamically imports the implementation only when the command is invoked.

---

## Complete Command List

### Session Management
| Command | Type | Description |
|---------|------|-------------|
| `/clear` | local | Clear conversation history, start fresh |
| `/compact` | local-jsx | Compact context with optional summary instruction |
| `/resume` | local-jsx | Resume a previous session from history |
| `/rewind` | local-jsx | Rewind to a previous point in the session |
| `/export` | local | Export conversation to file |
| `/summary` | local | Show conversation summary |
| `/context` | local-jsx | Show current context window usage |
| `/cost` | local | Show token and cost statistics |
| `/stats` | local | Show session statistics |
| `/usage` | local-jsx | Detailed usage breakdown |

### Configuration
| Command | Type | Description |
|---------|------|-------------|
| `/config` | local-jsx | Interactive settings editor |
| `/model` | local-jsx | Select AI model |
| `/theme` | local-jsx | Select color theme |
| `/output-style` | local-jsx | Select output style |
| `/vim` | local | Toggle Vim keybinding mode |
| `/permissions` | local-jsx | View/edit permission rules |
| `/env` | local-jsx | Manage environment variables |
| `/sandbox-toggle` | local | Toggle sandbox mode |
| `/color` | local-jsx | Configure colors |
| `/effort` | local-jsx | Set effort/thinking level |
| `/fast` | local | Toggle fast mode |

### Agent & Task Management
| Command | Type | Description |
|---------|------|-------------|
| `/agents` | local-jsx | View and manage agents |
| `/tasks` | local-jsx | View and manage tasks |
| `/plan` | local-jsx | Plan mode management |
| `/session` | local-jsx | Session information |

### MCP & Plugins
| Command | Type | Description |
|---------|------|-------------|
| `/mcp` | local-jsx | MCP server management |
| `/plugin` | local-jsx | Plugin install/enable/disable |
| `/reload-plugins` | local | Reload all plugins |

### Skills
| Command | Type | Description |
|---------|------|-------------|
| `/skills` | local-jsx | List and manage skills |

### Memory
| Command | Type | Description |
|---------|------|-------------|
| `/memory` | local-jsx | View and edit memory files |

### Auth & Account
| Command | Type | Description |
|---------|------|-------------|
| `/login` | local-jsx | Authenticate with Anthropic |
| `/logout` | local | Log out |
| `/upgrade` | local | Upgrade subscription |

### Git & Code
| Command | Type | Description |
|---------|------|-------------|
| `/diff` | local-jsx | View file changes as diff |
| `/branch` | local-jsx | Git branch operations |
| `/review` | remote | Code review (remote) |
| `/pr_comments` | remote | Review PR comments |
| `/autofix-pr` | remote | Auto-fix PR issues |
| `/install-github-app` | local | GitHub App installation |
| `/install-slack-app` | local | Slack App installation |

### Help & Info
| Command | Type | Description |
|---------|------|-------------|
| `/help` | local-jsx | Show help information |
| `/doctor` | local-jsx | Run diagnostics/health check |
| `/release-notes` | local-jsx | Show release notes |
| `/feedback` | local | Submit feedback |
| `/issue` | local | Report an issue |

### Remote & Bridge
| Command | Type | Description |
|---------|------|-------------|
| `/bridge` | local-jsx | Configure bridge mode |
| `/teleport` | local-jsx | Teleport session to remote |
| `/remote-setup` | local-jsx | Remote environment setup |
| `/remote-env` | local-jsx | Remote environment management |

### Keybindings
| Command | Type | Description |
|---------|------|-------------|
| `/keybindings` | local-jsx | View and edit keybindings |

### Diagnostic & Debug (mostly ant-only)
| Command | Type | Description |
|---------|------|-------------|
| `/debug` | local | Show debug information |
| `/perf-issue` | local | Report performance issue |
| `/heapdump` | local | Memory heap dump |
| `/ant-trace` | local | Ant tracing |
| `/ctx_viz` | local-jsx | Context visualization |
| `/debug-tool-call` | local | Debug a specific tool call |
| `/extra-usage` | local | Extended usage stats |
| `/break-cache` | local | Force cache break |
| `/thinkback` | local-jsx | Thinkback mode |
| `/thinkback-play` | local-jsx | Thinkback replay |
| `/btw` | local | Background task wakeup |
| `/bughunter` | local | Bug hunting mode |

### Utility
| Command | Type | Description |
|---------|------|-------------|
| `/copy` | local | Copy last response to clipboard |
| `/paste` | local | Paste clipboard content |
| `/tag` | local | Tag current session |
| `/rename` | local | Rename session |
| `/share` | local | Share session |
| `/status` | local-jsx | Show connection status |
| `/files` | local-jsx | File browser |
| `/add-dir` | local | Add directory to allowed paths |
| `/exit` | local | Exit Claude Code |
| `/stickers` | local-jsx | Easter egg sticker collection |
| `/passes` | local-jsx | Subscription passes |
| `/good-claude` | local | Positive reinforcement |

### Voice & IDE
| Command | Type | Description |
|---------|------|-------------|
| `/voice` | local-jsx | Voice input mode |
| `/ide` | local-jsx | IDE integration setup |
| `/lsp-recommendation` | local | LSP server recommendation |

### Admin (Ant-only)
| Command | Type | Description |
|---------|------|-------------|
| `/privacy-settings` | local-jsx | Privacy configuration |
| `/rate-limit-options` | local-jsx | Rate limiting |
| `/reset-limits` | local | Reset usage limits |
| `/mock-limits` | local | Mock rate limits for testing |
| `/passes` | local-jsx | API passes |
| `/backfill-sessions` | local | Backfill session data |

---

## Bundled Skills as Slash Commands

Several "commands" are actually backed by Skills (markdown prompt templates):

| Command | Backed By |
|---------|-----------|
| `/commit` | commit.md skill |
| `/review` | review.md skill |
| `/loop` | loop skill (cron scheduling) |
| `/schedule` | scheduleRemoteAgents skill |
| `/update-config` | updateConfig skill |
| `/simplify` | simplify skill |
| `/claude-api` | claude-api skill |
| `/claude-code-guide` | guide agent |

When `/commit` is typed, it gets expanded to the full skill prompt text as if the user typed it.
