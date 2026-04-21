# Design Plan A: Rust + Ratatui + Alacritty Terminal

## 1. Overview

A high-performance TUI implementation using Rust, built on `ratatui` for the interface and `alacritty_terminal` for the embedded terminal emulator. This stack prioritizes correctness, memory safety, and low-latency rendering while handling complex ANSI sequences and scrollback buffers efficiently.

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                          App (tokio)                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐  │
│  │  Terminal Area  │  │   Chat Area     │  │  Panel Area  │  │
│  │ alacritty_term  │  │  ratatui Widget │  │ratatui Widget│  │
│  │  + PTY (async)  │  │                 │  │              │  │
│  └────────┬────────┘  └────────┬────────┘  └──────┬───────┘  │
│           │                    │                  │          │
│  ┌────────▼────────────────────▼──────────────────▼───────┐  │
│  │              State Manager (Arc<RwLock<AppState>>)      │  │
│  └─────────────────────────────────────────────────────────┘  │
│           │                    │                  │          │
│  ┌────────▼────────┐  ┌────────▼────────┐  ┌─────▼──────┐  │
│  │  Shell Tracker  │  │   Agent Client  │  │  Workspace │  │
│  │  (Command Audit)│  │  (HTTP/JSON)    │  │  Manager   │  │
│  └─────────────────┘  └─────────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 3. Component Breakdown

### 3.1 Terminal Area

**Library:** `alacritty_terminal` crate (v0.24+)
- **PTY:** `alacritty_terminal::tty` spawns the user's shell with a PTY. Crossterm handles the raw mode switching for our TUI, while the PTY feeds raw bytes to the terminal grid.
- **Grid & Scrollback:** `alacritty_terminal::Term` provides a full VT500 emulator with an unlimited scrollback buffer. We configure it with a custom `Config` to set scrollback limits and colors.
- **Rendering:** On every frame, we call `term.renderable_content()` to get the current grid state, then map `alacritty_terminal`'s cells into `ratatui::buffer::Cell` objects within a custom `ratatui::widgets::Widget`. ANSI colors, bold, italic, and URLs are preserved.
- **Resize:** When the user resizes the application, we relay the new dimensions to both `alacritty_terminal::Term` (via `resize`) and the PTY (via `Tty::on_resize`).
- **Command Tracking:** A sidecar process or `PROMPT_COMMAND`/`precmd` hook writes a sentinel string to a Unix domain socket or FIFO. A `ShellTracker` task parses these to build a `Vec<ShellCommand>` with: `command_string`, `timestamp`, `exit_status`, `output_range` (start/end indices into scrollback). This is mapped to SPECS.md §4.3/§5.

### 3.2 Chat Area

**Library:** `ratatui` built-in + custom widgets
- **Message List:** A `List` widget wrapped in a custom `ChatHistory` widget. Each `ListItem` can be a `Message` enum variant:
  - `UserText { content, attachments }`
  - `AgentText { content, code_blocks, file_refs }`
  - `SystemNotice { content }`
- **Attachments:** When a message contains context attachments (viewport text, screenshot, command transcript), they render as collapsible `Block` widgets with distinct borders (`─ Context: Viewport Text ─`). Screenshots are stored as raw RGBA bytes and rendered as half-block Unicode characters (▀▄) or ASCII art previews if sixel/iterm image protocols are unavailable.
- **Input:** A custom widget wrapping `tui_textarea` (or a hand-rolled editor) at the bottom. Supports free-text input, `Enter` to send, and `Shift+Enter` for newlines. Input state is held in `AppState.chat_input`.
- **Agent Integration:** Messages are sent via an async HTTP client (`reqwest`) to a local agent server (or stdio to a spawned agent process). Responses stream back via SSE (Server-Sent Events) or chunked JSON and are appended to the history in real-time.

### 3.3 Panel Area

**Library:** `ratatui` `Block`, `Paragraph`, `List`
- **Focus Indicator:** A 3-segment horizontal bar at the top of the panel: `[ Terminal ] [ Chat ] [ Panel ]`. The active segment uses `Style::fg(Color::Green).add_modifier(Modifier::BOLD)` and a `▶` prefix. The inactive segments use dim gray.
- **Send-Context Actions:** A vertical `List` of selectable items:
  - `▢ Send visible viewport (text)`
  - `▢ Send visible viewport (screenshot)`
  - `▢ Send full scrollback buffer`
  - `▢ Send a specific command...` → opens sub-panel overlay
- **Command Picker Overlay:** When triggered, a floating `Clear`-backed overlay shows a `Table` of commands with columns: `Time`, `Command`, `Exit`, `Preview`. Multi-select is supported via `Space`. `Enter` confirms and attaches to the pending message.
- **Session Actions:**
  - `Reset conversation` → triggers a confirmation modal (`Dialog` widget: "Clear chat history and workspace? [Y/n]"). On confirm, sends reset to agent client and wipes `AppState.chat_history` and `WorkspaceManager`.

### 3.4 Key Bindings & Focus Management

Focus is an explicit state machine (`FocusTarget` enum: `Terminal`, `Chat`, `Panel`).
- `F1` → Terminal
- `F2` → Chat
- `F3` → Panel
- `Tab` / `Shift+Tab` → cycle focus
- Global: `Ctrl+Q` → quit. `Ctrl+C` is only passed to the Terminal when `focus == Terminal`; otherwise it copies selected text to clipboard via `arboard`.

## 4. Agent Workspace Integration

**Workspace Manager:** `tokio::fs` operations on `$HOME/_terminal_agent_workspace/`.
- On startup: ensure directory exists.
- On `Reset conversation`: `tokio::fs::remove_dir_all` then `create_dir_all`.
- The Agent Client POSTs attachments to the agent server, which writes any generated files into the workspace. The TUI can display workspace file listings if needed.

## 5. Data Flow

### Sending Context (Viewport → Agent)
1. User focuses Panel, selects "Send visible viewport (text)".
2. Panel widget calls `TerminalArea::extract_viewport_text()` which calls `term.renderable_content()` and serializes visible rows to a `String`, preserving whitespace.
3. A `ContextAttachment::ViewportText { content }` is pushed to `AppState.pending_attachments`.
4. User types optional text in Chat input and presses `Enter`.
5. `AppState` constructs a `UserMessage { text, attachments }`.
6. Agent Client serializes to JSON and POSTs to agent endpoint.
7. Agent response streams back; each chunk appends to `AppState.chat_history`.

## 6. Key Dependencies

| Crate | Purpose |
|-------|---------|
| `ratatui` | TUI framework, widgets, buffer, layout |
| `crossterm` | Input events, raw mode, terminal resize, clipboard |
| `alacritty_terminal` | VT500 emulator, PTY, scrollback grid |
| `tokio` | Async runtime for PTY I/O, HTTP, file watcher |
| `reqwest` | HTTP client for agent communication |
| `serde` / `serde_json` | Message serialization |
| `arboard` | Clipboard access for non-terminal focus |
| `chrono` | Timestamps for command history |
| `tracing` | Structured logging |

## 7. Implementation Phases

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| 1. Foundation | 3 days | App loop with crossterm input, focus cycling, empty 3-pane layout in ratatui |
| 2. Terminal Emulator | 5 days | PTY spawn, `alacritty_terminal` integration, ANSI rendering into ratatui buffer, scrollback |
| 3. Shell Tracking | 3 days | `PROMPT_COMMAND` hook protocol, `ShellTracker` parsing, command history Vec |
| 4. Chat & Agent | 4 days | Chat history widget, text input, HTTP agent client, SSE streaming, attachment rendering |
| 5. Panel & Context | 3 days | Focus indicator, context action list, command picker overlay, viewport/scrollback extraction |
| 6. Workspace & Polish | 2 days | Workspace directory management, reset confirmation, error modals, keybinding help overlay |

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| `alacritty_terminal` API churn | Pin to a specific minor version; fork/vendor if necessary |
| Shell tracking across SSH | Track locally only; SSH sessions are opaque but local commands before/after are captured |
| High CPU on large scrollback | Limit `alacritty_terminal` scrollback to 100k lines; use `renderable_content()` only for visible viewport |
| Image rendering in TUI | Fallback to half-block Unicode or ASCII art; document lack of sixel as known limitation |
| Clipboard in SSH/headless | `arboard` handles X11/Wayland; over SSH, user can disable clipboard features gracefully |

## 9. Why This Design?

Rust's ownership model eliminates data races between the PTY reader thread, UI render thread, and async agent I/O. `alacritty_terminal` is battle-tested (it powers Alacritty) and gives us correct ANSI handling and infinite scrollback "for free." `ratatui` is the de facto standard for Rust TUIs with excellent performance and a mature ecosystem.
