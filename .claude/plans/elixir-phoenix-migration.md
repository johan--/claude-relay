# Claude Relay: Node.js to Elixir/Phoenix Migration Plan

## Context

claude-relay is a Web UI for Claude Code (~10,000 lines across 27 files). It relays Claude Code sessions from a local machine to the browser via WebSocket. This plan rewrites it in Elixir/Phoenix — the user's preferred stack — replacing the Claude Agent SDK with direct Anthropic Messages API calls, raw WebSocket with Phoenix LiveView, and JSONL files with Ecto + SQLite.

**Decisions:** Direct HTTP API (no Node.js dependency), LiveView frontend, SQLite now / Postgres later, core chat first.

## Architecture

```
Supervision tree:
  ClaudeRelay.Application
  ├── ClaudeRelay.Repo (Ecto + SQLite)
  ├── ClaudeRelayWeb.Endpoint (Phoenix/Bandit)
  ├── Phoenix.PubSub
  ├── Registry (ClaudeRelay.ProjectRegistry)
  └── ClaudeRelay.ProjectSupervisor (DynamicSupervisor)
      └── ClaudeRelay.ProjectServer (GenServer per project)
          └── ClaudeRelay.Conversation (GenServer per active conversation)
              ├── HTTP streaming to Anthropic API via Req
              ├── tool execution (ClaudeRelay.Tools.*)
              └── permission handling via PubSub → LiveView
```

**Why no daemon:** Phoenix IS the long-running process. OTP supervision handles restarts. Eliminates daemon.js, ipc.js, and all PID/socket management.

**Why direct API:** Removes Node.js dependency entirely. The Agent SDK is just a wrapper around the Messages API. Elixir's Req handles SSE streaming well.

**Why LiveView:** Automatic reconnection, diff-based DOM updates, server-rendered HTML. JS hooks only for streaming text, xterm.js, syntax highlighting, mermaid.

---

## Phase 1: Scaffold + Claude API Streaming (MVP)

**Goal:** Send a message, stream Claude's response, display in browser.

### Hex Dependencies

```elixir
{:phoenix, "~> 1.7"},
{:phoenix_live_view, "~> 1.0"},
{:ecto_sqlite3, "~> 0.17"},
{:req, "~> 0.5"},
{:jason, "~> 1.4"},
{:bandit, "~> 1.5"},
{:esbuild, "~> 0.8"},
{:tailwind, "~> 0.2"}
```

### Key Files

```
lib/claude_relay/
  application.ex
  repo.ex
  anthropic/
    client.ex              # Req streaming to Messages API
    stream_parser.ex       # SSE event parser
    tool_definitions.ex    # Tool JSON schemas
  conversations/
    conversation.ex        # GenServer: API loop, stream, tool dispatch
    session.ex             # Ecto schema
    message.ex             # Ecto schema

lib/claude_relay_web/
  router.ex
  live/session_live.ex     # Main chat LiveView
  components/
    message_component.ex   # User/assistant message rendering
    layouts.ex

assets/js/hooks/
  message_stream.js        # Append streaming deltas to DOM
  code_highlight.js        # highlight.js integration
  mermaid_render.js        # mermaid.js integration
```

### Ecto Schemas

```elixir
# sessions
:title, :string
:project_slug, :string
:is_processing, :boolean
timestamps()

# messages
:role, :string              # "user" | "assistant" | "system"
:content, :string           # JSON-encoded content blocks
:message_type, :string      # "delta", "tool_start", "tool_result", etc.
:metadata, :map
belongs_to :session
timestamps()
```

### Conversation GenServer Flow

1. `send_message(pid, text, images)` → builds messages payload, calls Anthropic API with `stream: true`
2. Parses SSE events: `content_block_delta` → broadcasts `{:delta, text}` via PubSub
3. LiveView receives via `handle_info`, pushes event to JS hook
4. JS hook appends text + renders markdown incrementally
5. On `message_stop` → persists to DB, broadcasts `:done`

### Verify

- `mix test` — Conversation GenServer streams and parses SSE correctly
- Manual: type message in browser, see streamed response render in real-time

---

## Phase 2: Tool Execution + Approval

**Goal:** Claude can use tools (Read, Edit, Write, Bash, Glob, Grep), user approves via UI.

### Tool Modules

```
lib/claude_relay/tools/
  tool.ex               # @callback execute(input, project_path) :: {:ok, result} | {:error, reason}
  read.ex               # File.read!/1 + path validation
  write.ex              # File.write!/2 + path validation
  edit.ex               # String replacement in files
  bash.ex               # System.cmd/3 with timeout + output truncation
  glob.ex               # Path.wildcard/2
  grep.ex               # File.stream! + Regex with line numbers
  web_fetch.ex          # Req.get/1
  executor.ex           # Dispatches tool_use blocks to correct module
  path_validator.ex     # safePath equivalent — prevents directory traversal
```

### Conversation GenServer Tool Loop

```
1. Stream receives content_block with type: "tool_use"
2. GenServer pauses, broadcasts {:permission_request, request_id, tool_name, input}
3. LiveView shows PermissionComponent modal
4. User clicks approve → handle_event → GenServer.cast(:permission_response, ...)
5. If approved: Executor runs tool → GenServer sends tool_result to API → resumes stream
6. If "always allow": stored in GenServer state, auto-approves future requests for that tool
7. If denied: sends tool_result with is_error: true → Claude adjusts
```

### LiveView Components

```
components/
  tool_component.ex          # Tool item: name, input summary, result, collapsible
  permission_component.ex    # Modal: approve / always allow / deny
  diff_component.ex          # Edit tool: old→new diff rendering
  thinking_component.ex      # Collapsible thinking block with timer
```

### JS Hooks

```
hooks/diff_display.js        # Unified diff rendering with syntax highlighting
```

### Verify

- Each tool module: unit tests with fixtures
- Path validator: rejects `../../etc/passwd`
- Full loop: send message → Claude requests Bash → approve → output displayed → Claude continues

---

## Phase 3: Sessions + Multi-Project

**Goal:** Persistent sessions, switching, search. Multiple project directories.

### Ecto Schema Addition

```elixir
# projects
:slug, :string
:path, :string
:title, :string
timestamps()
```

### Files

```
lib/claude_relay/
  projects/
    project.ex               # Ecto schema
    project_server.ex        # GenServer per project
    project_supervisor.ex    # DynamicSupervisor
```

### Router

```elixir
scope "/p/:slug", ClaudeRelayWeb do
  live "/", SessionLive
end

live "/", DashboardLive    # Multi-project dashboard
```

### Session Features

- **Sidebar LiveComponent**: session list grouped by date, search, rename, delete
- **Session switch**: update LiveView assigns, reload messages from DB (paginated, 200 at a time)
- **Search**: `LIKE` query on messages.content (SQLite FTS5 for v2)
- **Progressive history**: `LIMIT 200 OFFSET ?`, triggered by scroll sentinel (IntersectionObserver JS hook)

### PIN Auth

```elixir
# Plug — checks cookie against stored SHA256 hash
ClaudeRelayWeb.Plugs.PinAuth
```

### LiveView Components

```
components/
  sidebar_component.ex       # Session list, search, project switcher
live/
  dashboard_live.ex          # Multi-project card grid
```

### Verify

- Create/switch/delete sessions persists correctly
- Search finds matches in message content
- Multi-project: different slugs load different project contexts
- PIN auth blocks unauthenticated access

---

## Phase 4: Terminal (PTY)

**Goal:** Browser-based terminal via Erlang Port + xterm.js.

### Approach

Use a thin C wrapper (`priv/pty_wrapper.c`) that calls `forkpty()` + `exec(shell)` and relays via Erlang Port protocol. Alternatively, evaluate the `exile` hex package.

### Files

```
lib/claude_relay/terminals/
  terminal_manager.ex        # GenServer: manages up to 10 terminals per project
  terminal_session.ex        # GenServer per terminal (wraps Port, 50KB scrollback)

lib/claude_relay_web/
  components/terminal_component.ex

assets/js/hooks/
  xterm_hook.js              # xterm.js + FitAddon, sends input/resize via pushEvent
```

### Protocol

```
Client → Server: term_create, term_input, term_resize, term_close
Server → Client: push_event term_output, term_exited, term_list
```

### Verify

- Terminal spawns shell, echoes commands
- Resize works
- Multi-tab: create/switch/close tabs
- Scrollback replays on reconnect

---

## Phase 5: Polish

| Feature | Approach |
|---------|----------|
| **File browser** | LiveComponent + `FileSystem` hex package for watching |
| **Push notifications** | `web_push_encryption` hex + VAPID keys in DB |
| **Rewind** | Truncate messages after target point, resume API from there |
| **Mobile** | Port CSS media queries, PWA manifest + service worker |
| **Usage panel** | Req call to Anthropic OAuth usage endpoint |
| **Input sync** | PubSub broadcast of input text across devices |
| **Model switching** | Pass model param to Anthropic API, persist preference |

---

## Critical Source Files (for reference during implementation)

| Node.js file | Lines | Maps to |
|---|---|---|
| `lib/sdk-bridge.js` | 579 | `Anthropic.Client` + `Conversation` GenServer |
| `lib/project.js` | 819 | `ProjectServer` GenServer + LiveView event handling |
| `lib/public/app.js` | 1654 | `SessionLive` + JS hooks |
| `lib/sessions.js` | 341 | Ecto schemas + queries |
| `lib/public/modules/tools.js` | 1228 | LiveView tool/permission/diff components |
| `lib/server.js` | 500 | Phoenix Router + Endpoint + PinAuth plug |
| `lib/daemon.js` | 258 | Eliminated (OTP supervision replaces this) |
| `lib/terminal-manager.js` | 187 | `TerminalManager` + `TerminalSession` GenServers |
| `lib/config.js` | 184 | Application config + Ecto |

## New Repo Setup

```bash
mix phx.new claude_relay --no-mailer --no-dashboard --no-gettext
cd claude_relay
# Add ecto_sqlite3, req to mix.exs
mix deps.get
mix ecto.create
```
