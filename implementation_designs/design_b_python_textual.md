# Design Plan B: Python + Textual

## 1. Overview

A rapid-development TUI implementation using Python and the modern `textual` framework. Textual's reactive data model, CSS-like styling, and rich built-in widgets (DataTable, Markdown, Input) allow fast iteration on complex UI interactions like the command picker and chat history. This design trades a small amount of runtime overhead for extreme developer velocity and expressive UI composition.

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    AgentTerminalApp (Textual)                 │
│  ┌──────────────────┐ ┌──────────────────┐ ┌───────────────┐ │
│  │  TerminalPane    │ │    ChatPane      │ │   PanelPane   │ │
│  │  (Rich + Pyte)   │ │  (Textual Widget)│ │(Textual Widget│ │
│  │                  │ │                  │ │               │ │
│  │  - PTY via pty   │ │  - Message Log   │ │ - Focus Bar   │ │
│  │  - Pyte Screen   │ │  - Input Area    │ │ - Actions     │ │
│  │  - Scrollback    │ │  - Attachments   │ │ - Picker      │ │
│  └────────┬─────────┘ └────────┬─────────┘ └───────┬───────┘ │
│           │                    │                   │         │
│  ┌────────▼────────────────────▼───────────────────▼───────┐ │
│  │              Reactive State Store (textual.reactive)     │ │
│  └──────────────────────────────────────────────────────────┘ │
│           │                    │                   │         │
│  ┌────────▼─────────┐ ┌────────▼─────────┐ ┌──────▼────────┐│
│  │  ShellTracker    │ │   AgentClient    │ │   Workspace   ││
│  │  (async thread)  │ │  (aiohttp/SSE)   │ │  (pathlib)    ││
│  └──────────────────┘ └──────────────────┘ └───────────────┘│
└──────────────────────────────────────────────────────────────┘
```

## 3. Component Breakdown

### 3.1 Terminal Area

**Libraries:** `pyte` (VTxxx emulator), `ptyprocess` (PTY + process control), `rich.text.Text`
- **PTY:** `ptyprocess.PtyProcessUnicode.spawn([os.environ.get("SHELL", "/bin/bash")])` creates the pseudo-terminal. Textual's worker API (`@work(exclusive=True, thread=True)`) runs a blocking read loop on the PTY master fd.
- **Screen Emulation:** `pyte.Screen` and `pyte.ByteStream` process the raw byte stream from the PTY. `Screen` maintains a 2D array of `pyte.Char` objects with `data`, `fg`, `bg`, `bold`, etc.
- **Rendering:** A custom Textual `Widget` iterates over `screen.display` (or `screen.buffer`) and converts each `pyte.Char` into `textual` `Segment` objects with inline styles. This renders inside a `textual.widgets.Static` or a custom `TerminalCanvas`.
- **Scrollback:** `pyte.HistoryScreen` extends `Screen` with scrollback lines. We configure a large history limit (e.g., 50,000 lines). When rendering, we compute the visible slice based on the current scroll offset (updated by mouse wheel or `Shift+PageUp`/`Shift+PageDown`).
- **Resize:** On `Resize` messages from Textual, we call `screen.resize(lines, columns)` and send `SIGWINCH` to the PTY process.
- **Command Tracking:** We inject a shell integration script (via `PROMPT_COMMAND` for Bash, `precmd` for Zsh) that prints an OSC 1337/Ansi sequence or a custom sentinel like `\e]133;C;cmd=<base64>\a`. The PTY read loop detects these sequences, parses the command string, and appends to a thread-safe `deque[CommandRecord]`. On command completion (detected by the next prompt sentinel), we capture the output range from scrollback indices.

### 3.2 Chat Area

**Libraries:** `textual` containers, `textual.widgets.Input`, `textual.widgets.Markdown`
- **Message Log:** A `textual.containers.ScrollableContainer` holding a dynamic list of message widgets. Each message is a custom `ChatMessage` widget:
  - User messages: right-aligned bubble with a light blue border.
  - Agent messages: left-aligned bubble with a neutral border.
  - Code blocks inside agent messages render as `Markdown` fenced blocks or a custom `Static` with syntax highlighting via `rich.syntax.Syntax`.
- **Attachments:** Context attachments are rendered as collapsible `Tree` nodes or `Accordion` children under the parent message. A viewport screenshot attachment stores raw ANSI or a `PIL` thumbnail and renders as a `rich.text.Text` with color preserved, or a simple `[Image: 80x24]` placeholder if the terminal cannot display it.
- **Input:** A `textual.widgets.Input` with `multiline=True` (or a custom `TextArea` if available) sits at the bottom. `Enter` sends if the input is non-empty or there are pending attachments; `Shift+Enter` inserts a newline.
- **Agent Streaming:** An async worker (`@work`) opens an SSE connection to the agent. Each event yields a partial response; we append to the last `AgentMessage` widget in the log and call `scroll_end()` on the container.

### 3.3 Panel Area

**Libraries:** `textual.widgets`, `textual.reactive`
- **Focus Indicator:** A horizontal `textual.widgets.RadioSet` or custom `Horizontal` container with three `Button` widgets: `Terminal`, `Chat`, `Panel`. The active focus is driven by a reactive `Reactive[FocusTarget]` variable. Buttons are styled via Textual CSS (e.g., `Button:focus { background: $success; }`).
- **Send-Context Actions:** A `textual.widgets.ListView` of `ListItem` widgets:
  - Send visible viewport (text)
  - Send visible viewport (screenshot)
  - Send full scrollback buffer
  - Send specific command...
  Selecting an item triggers a callback that builds a `ContextAttachment` and appends it to `AppState.pending_attachments`.
- **Command Picker Overlay:** A modal `textual.screen.Screen` overlay containing a `DataTable`. Columns: `Time`, `Command`, `Exit`, `Preview`. Multi-select is enabled via a checkbox column. Row selection updates a `selected_commands` set. A `Submit` button at the bottom confirms and closes the modal, pushing `ContextAttachment::CommandTranscript` to the pending list.
- **Session Actions:** A `Button` labeled "Reset Conversation" with an `on_button_pressed` handler that calls `self.push_screen(ConfirmScreen("Reset chat and workspace?", on_confirm=self.do_reset))`.

### 3.4 Styling

Textual's CSS (`agent_terminal.tcss`) defines the three-pane layout:
```css
Screen { layout: grid; grid-size: 2 2; }
TerminalPane { row-span: 2; }
ChatPane { column-span: 1; }
PanelPane { column-span: 1; }
```
This keeps all three regions visible simultaneously, as required by SPECS.md §4.

## 4. Agent Workspace Integration

**Library:** `pathlib`, `shutil`
- On app startup: `Path.home() / "_terminal_agent_workspace"` is created if absent.
- On conversation reset: `shutil.rmtree(workspace)` followed by `workspace.mkdir()`.
- The `AgentClient` includes the workspace path in its initialization payload so the agent knows where to write files. Chat messages can reference workspace files as clickable `textual.widgets.Link` objects.

## 5. Data Flow

### Extracting Context (Command Picker)
1. User presses `F3` → Panel gets focus.
2. User selects "Send a specific command..." → `self.push_screen(CommandPickerScreen)`.
3. `CommandPickerScreen` reads `ShellTracker.history` into `DataTable` rows.
4. User `Space`-selects rows 2 and 3, clicks Submit.
5. For each selected row, `TerminalArea` extracts scrollback lines between `start_index` and `end_index`.
6. A `CommandTranscript { command, timestamp, exit_status, stdout, stderr }` is created.
7. It appears as a badge in the Chat input area: `[2 commands attached]`.
8. User sends message; `AgentClient` POSTs JSON with `message` and `attachments` list.

## 6. Key Dependencies

| Package | Purpose |
|---------|---------|
| `textual` | TUI framework, CSS, widgets, reactive state |
| `pyte` | VT102/UTF-8 terminal emulator with scrollback |
| `ptyprocess` | Spawn PTY, send signals, resize |
| `aiohttp` | Async HTTP client for agent, SSE support |
| `rich` | Syntax highlighting, text styling (Textual already bundles it) |
| `Pillow` | Screenshot rasterization for viewport images |
| `pydantic` | Data validation for messages, attachments, commands |

## 7. Implementation Phases

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| 1. Skeleton | 2 days | Textual app with CSS grid layout, 3 placeholder panes, focus cycling |
| 2. Terminal Emulation | 4 days | PTY spawn, `pyte` integration, ANSI rendering into Static widget, scrollback, resize |
| 3. Shell Tracking | 3 days | Shell hook injection, sentinel parsing, `CommandRecord` dataclass, history deque |
| 4. Chat Surface | 3 days | Message log widgets, Input, Markdown rendering, attachment badges |
| 5. Agent Integration | 3 days | `AgentClient` with SSE streaming, workspace path handling, error toasts |
| 6. Panel & Polish | 3 days | Context action list, command picker modal, reset confirmation, help overlay, key bindings |

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| `pyte` performance on huge scrollback | Cap history at 50k lines; offer "Send full scrollback" as a streamed file upload rather than in-memory string |
| Textual CSS layout edge cases | Use `grid` layout with `fr` units; test at 80x24 and 200x60 terminal sizes |
| GIL contention in PTY read loop | Keep PTY read in a Textual `work(thread=True)` worker; communicate via `self.call_from_thread()` to update reactive state |
| Agent SSE reconnection | Implement exponential backoff in `AgentClient`; show disconnected state in Panel |
| Shell hook incompatibility (Fish, Nushell) | Provide per-shell hook scripts; gracefully degrade if hook cannot be installed |

## 9. Why This Design?

Textual's reactive programming model eliminates an entire class of UI state bugs. Its built-in `DataTable`, `Markdown`, and `Input` widgets mean we spend less time building low-level UI primitives and more time on the domain logic (context extraction, agent communication). Python's `pyte`/`ptyprocess` combo is mature and well-documented. The ecosystem allows rapid prototyping; if a feature needs changing, the iteration cycle is seconds, not minutes.
