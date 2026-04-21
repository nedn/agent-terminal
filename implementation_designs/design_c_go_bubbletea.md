# Design Plan C: Go + Bubble Tea + Charm

## 1. Overview

A lightweight, composable TUI built in Go using Charm's `bubbletea` (Elm architecture), `bubbles` (pre-built components), and `lipgloss` (declarative styling). The terminal emulator is handled by embedding a small VT parser and leveraging Go's excellent concurrency primitives (goroutines + channels) to bridge the PTY and the Elm message loop cleanly.

## 2. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Bubble Tea Model                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐  │
│  │  TerminalModel  │  │    ChatModel    │  │  PanelModel  │  │
│  │  (viewport.Model│  │  (list.Model    │  │(list.Model + │  │
│  │   + PTY + VT)   │  │   + textarea)   │  │  table.Model)│  │
│  └────────┬────────┘  └────────┬────────┘  └──────┬───────┘  │
│           │                    │                  │          │
│  ┌────────▼────────────────────▼──────────────────▼───────┐  │
│  │              Root Model (focus + orchestration)          │  │
│  └─────────────────────────────────────────────────────────┘  │
│           │                    │                  │          │
│  ┌────────▼────────┐  ┌────────▼────────┐  ┌─────▼──────┐  │
│  │   PTY Goroutine │  │   Agent Client  │  │  Workspace │  │
│  │   + VT Parser   │  │   (HTTP/SSE)    │  │  (os pkg)  │  │
│  └─────────────────┘  └─────────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 3. Component Breakdown

### 3.1 Terminal Area

**Libraries:** `creack/pty` (PTY creation), custom VT parser + `github.com/charmbracelet/bubbles/viewport`
- **PTY:** `creack/pty.Start(cmd)` spawns the shell with a PTY. A dedicated goroutine reads from the PTY master fd via `bufio.Reader` and sends raw byte slices into a channel: `ptyCh <- PtyMsg{data: buf}`.
- **VT Parser:** A lightweight state machine parses ANSI sequences (SGR colors, cursor positioning, erase) into a grid of `Cell{rune, fg, bg, style}`. This is simpler than a full VT500 emulator but sufficient for common CLI tools (ls, git, vim/neovim in basic mode, htop). TUI applications that use alternate screen buffers (e.g., `less`, `vim`) are captured as-is; the user sees them in the terminal area just as they would in a normal terminal.
- **Grid & Scrollback:** A circular buffer of `[]Cell` rows implements scrollback. The visible viewport is a slice into this buffer. `viewport.Model` (from `bubbles`) handles the scroll offset and mouse wheel events.
- **Rendering:** The TerminalModel's `View()` method renders rows as strings, applying `lipgloss.Style` per-cell for colors and attributes. Each row is joined and the full grid is returned as a `lipgloss.NewStyle().Border(...).Render(...)` block.
- **Resize:** On `tea.WindowSizeMsg`, we update `pty.Setsize`, the VT parser's column count, and `viewport.Width/Height`.
- **Command Tracking:** A small shell wrapper script (injected via `SHELL` env var or `.bashrc` source) emits OSC sequences `\033]133;A;cmd=<cmd>\007` and `\033]133;B;status=<exit>\007`. The VT parser intercepts these as invisible sequences (they set a flag on the current row metadata). A `CommandTracker` goroutine consumes these metadata events and maintains a `[]Command` slice.

### 3.2 Chat Area

**Libraries:** `bubbles/list`, `bubbles/textarea`, `lipgloss`
- **Message List:** `list.Model` from `bubbles` is repurposed as a chat history. Each item implements `list.Item` interface. Items are:
  - `UserItem` (styled with `lipgloss.NewStyle().Align(lipgloss.Right).Foreground(lipgloss.Color("39"))`)
  - `AgentItem` (left-aligned, neutral color)
  - `AttachmentItem` (indented, dim border, showing context type and size)
- **Input:** `textarea.Model` provides a multiline input with word wrap. `Ctrl+Enter` sends; plain `Enter` inserts a newline. The textarea sits below the list inside a vertical `lipgloss` layout box.
- **Streaming:** Agent responses arrive as `AgentChunkMsg` in the Bubble Tea update loop. The last `AgentItem` in the list is updated by appending the chunk to its `content` string. `list.Model` is re-built and a `cmd()` returns `nil` so the UI re-renders immediately.
- **Code Rendering:** Agent messages containing triple-backtick blocks are split. Code is rendered with a `lipgloss.NewStyle().Background(lipgloss.Color("#1e1e1e")).Padding(1)` box and syntax highlighting via `github.com/alecthomas/chroma` (or simply dim/italic styling if avoiding external deps).

### 3.3 Panel Area

**Libraries:** `bubbles/list`, `bubbles/table`, `bubbles/help`, `lipgloss`
- **Focus Indicator:** Three `lipgloss.Style`-rendered boxes in a horizontal layout. The active box has a thick border (`Border(lipgloss.RoundedBorder())`) and bright color; inactive boxes have a thin `Border(lipgloss.HiddenBorder())` and gray. Example:
  ```go
  activeStyle := lipgloss.NewStyle().Border(lipgloss.RoundedBorder()).BorderForeground(lipgloss.Color("170")).Padding(0, 2)
  ```
- **Send-Context Actions:** A `list.Model` of action strings. `Enter` on an item triggers an action. Items map to functions in the Root Model that produce `tea.Cmd` messages:
  - "Send visible viewport (text)" → `func() tea.Cmd { return func() tea.Msg { return AttachViewportTextMsg{...} } }`
  - "Send visible viewport (screenshot)" → captures the current grid's RGB values and encodes to PNG bytes, wrapping in `AttachViewportImageMsg{...}`
  - "Send full scrollback" → serializes entire circular buffer
  - "Send a specific command..." → pushes a modal overlay state
- **Command Picker Overlay:** When active, the PanelModel switches to a `table.Model` populated from `CommandTracker.Commands`. Columns: `Time`, `Command`, `Exit`. A custom key handler allows `Space` to toggle a `Selected` boolean on the row data. A footer shows `3 selected  [Enter] confirm  [Esc] cancel`. Confirming emits `AttachCommandsMsg{commands: selected}`.
- **Session Actions:** A `help.Model` shows key hints. `Ctrl+R` triggers a confirmation overlay (a centered `lipgloss` box: "Reset conversation? [y/N]").

### 3.4 Key Bindings & Focus

Focus is managed by the Root Model as an enum (`FocusTerminal`, `FocusChat`, `FocusPanel`).
- `F1`/`F2`/`F3` or `Ctrl+H`/`Ctrl+J`/`Ctrl+K` cycle focus.
- Global `tea.KeyMsg` is intercepted in Root's `Update()` before delegating to sub-models.
- `Ctrl+C`:
  - If `FocusTerminal`: forwarded to PTY via `pty.Write([]byte{0x03})` (SIGINT).
  - Otherwise: ignored or used for "cancel current action".
- `Ctrl+Q`: global quit.

## 4. Agent Workspace Integration

**Library:** `os`, `path/filepath`, `io/fs`
- Path: `filepath.Join(os.Getenv("HOME"), "_terminal_agent_workspace")`
- On `ResetConversationMsg`:
  ```go
  os.RemoveAll(workspacePath)
  os.MkdirAll(workspacePath, 0755)
  ```
- The `AgentClient` is initialized with the workspace path. Agent responses may include `file_create` actions; the TUI displays these as system notices in the chat log.

## 5. Data Flow

### PTY → Terminal Grid → Screen
1. `ptyGoroutine` reads bytes from PTY master fd.
2. Bytes are sent via `tea.Cmd` to Bubble Tea's central `Update()` loop as `PtyOutputMsg{data: []byte}`.
3. `TerminalModel.Update()` feeds bytes to the VT parser.
4. VT parser updates the `[][]Cell` grid and scrollback buffer.
5. `Root.View()` calls `TerminalModel.View()`, which reads the visible slice of the grid and renders it with `lipgloss`.
6. Frame renders at 60 FPS (throttled by Bubble Tea's default behavior).

### Context Extraction → Agent
1. User navigates PanelModel to "Send visible viewport (text)".
2. PanelModel emits `AttachViewportTextMsg{content: terminalModel.VisibleText()}`.
3. RootModel stores it in `pendingAttachments`.
4. User types in `textarea` and presses `Ctrl+Enter`.
5. RootModel constructs `UserMessage{text, attachments}`.
6. RootModel returns a `tea.Cmd` that POSTs JSON to the agent server.
7. Response is streamed via an `io.Reader`; a goroutine sends `AgentChunkMsg`s back into the Update loop.

## 6. Key Dependencies

| Package | Purpose |
|---------|---------|
| `charmbracelet/bubbletea` | Elm architecture TUI framework |
| `charmbracelet/bubbles` | Reusable components: viewport, list, textarea, table, help, key |
| `charmbracelet/lipgloss` | Declarative styling, layout, colors |
| `creack/pty` | PTY allocation and resize |
| `alecthomas/chroma` | Syntax highlighting for code blocks (optional) |
| `golang.org/x/term` | Terminal state restoration, size queries |
| `github.com/hashicorp/go-retryablehttp` | Resilient HTTP client for agent |

## 7. Implementation Phases

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| 1. Shell | 2 days | Bubble Tea app loop, 3-pane lipgloss layout, focus switching, quit handling |
| 2. PTY & VT | 4 days | `creack/pty` integration, basic VT state machine (SGR + cursor + scroll), viewport rendering, scrollback ring buffer |
| 3. Shell Hooks | 2 days | OSC sentinel parser, `Command` struct slice, command picker table |
| 4. Chat | 3 days | `textarea` input, `list` history, user/agent message styling, attachment badges |
| 5. Agent IO | 3 days | HTTP client with SSE, streaming chunks into Bubble Tea messages, workspace path initialization |
| 6. Panel & Extras | 2 days | Context action list, screenshot encoding (ANSI art fallback), reset confirmation, help keymap |

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Custom VT parser incompleteness | Document supported sequences; for edge cases, fall back to raw passthrough or recommend user runs `TERM=dumb` |
| Bubble Tea's strict message loop complexity | Keep Root Model thin; delegate all domain logic to sub-models. Use `tea.Batch` for parallel commands |
| Large scrollback memory use | Use a ring buffer of rows; evict old rows to a temp file if exceeding a memory cap (e.g., 10MB) |
| Image rendering in pure TUI | Encode viewport as ANSI-colored text by default; provide optional truecolor PNG generation that opens in external viewer or saves to workspace |
| Cross-platform PTY (`creack/pty`) | Works on Linux/macOS/Windows (with pseudo-console on Win10+). CI tests on all three platforms |

## 9. Why This Design?

Go's simplicity and fast compile times make iterating on TUI behavior extremely productive. Bubble Tea's Elm architecture forces a clean separation between state, events, and rendering, which prevents the spaghetti code common in interactive terminal apps. The Charm ecosystem (`bubbles`, `lipgloss`) provides polished, accessible components out of the box. Go's goroutines map naturally to the concurrent problem domain (PTY I/O, HTTP streaming, file system operations) without the complexity of async/await runtimes. The resulting binary is a single, dependency-free executable (aside from the shell), ideal for distribution.

## 10. Comparison with Other Designs

| Concern | Design A (Rust) | Design B (Python) | This Design (Go) |
|---------|----------------|-------------------|------------------|
| Binary size | ~5MB | N/A (interpreter) | ~15MB |
| Startup latency | ~50ms | ~200ms | ~30ms |
| ANSI correctness | Excellent (Alacritty) | Good (Pyte) | Good (custom VT) |
| Developer velocity | Medium | High | High |
| Memory safety | Compile-time | Runtime (GC) | Compile-time (GC) |
| Ecosystem maturity | High | Very High | High (growing fast) |
