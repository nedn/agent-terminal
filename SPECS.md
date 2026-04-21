# Agent Terminal Specification

## 1. Overview

A standalone desktop GUI application that lets users share what's happening in their terminal with a 
coding agent **without** having to copy, paste, or screenshot by hand.

The core problem: today, when a developer hits a wall in the terminal and wants help from a coding 
agent, they have to manually copy text (often losing ANSI formatting, wrapping, or context), or take 
a screenshot and drag it into a chat. This is slow, breaks flow, and is especially painful over SSH 
or in long-running sessions where the relevant output has scrolled out of the visible window.

This application co-locates a working terminal, a chat surface with a coding agent, and a 
context-selection panel — so any part of a terminal session (a single command's output, the current 
viewport, or the full scrollback) can be handed to the agent in one action.

## 2. Goals & Non-Goals

**Goals**

- Eliminate the manual copy/paste/screenshot step between terminal and agent.
- Give users explicit, fine-grained control over *what* context is sent to the agent.
- Let the agent do useful work (research, scripting, experimentation) without requiring trust in arbitrary local filesystem access.

**Non-Goals**

- Letting the agent act directly inside the user's terminal session (e.g., typing commands for 
them). The agent operates in its own sandbox.
- Cross-device sync, team collaboration, or shared sessions.

## 3. Platform

Standalone desktop GUI application. Implementation stack is intentionally left open — it will be 
chosen and iterated on separately from this spec.

## 4. User Interface

The window is divided into three regions. All three are visible simultaneously; only one holds 
keyboard focus at a time.

### 4.1 Terminal Area

A fully functional terminal emulator where the user does real work: typing commands, running 
processes, SSH-ing into remote hosts, using TUI applications, etc.

Requirements:

- Behaves like a standard terminal (PTY-backed, ANSI/VT support, resize handling, scrollback).
- Maintains a **scrollback buffer** beyond what is currently visible in the viewport.
- Tracks **command history** at the shell level — the application must know which shell commands the 
user ran, and the stdout/stderr produced by each. This is what makes per-command context selection 
(§4.3) possible.

### 4.2 Chat Area

A conversation view showing the running dialogue with the coding agent.

Requirements:

- Shows user messages and agent responses in chronological order.
- Renders attachments the user sent as context (text blocks, screenshots, command transcripts) 
clearly and distinguishably from free-text messages.
- Supports a free-text input so the user can ask questions or give instructions that aren't tied to 
terminal content.
- Renders agent output that includes code, file references, or links from web research in a readable 
way.

### 4.3 Panel Area

A control panel that does two things: shows the user where focus currently is, and exposes actions 
for sending context to the agent and managing the session.

**Focus indicator.** At all times, the panel clearly shows whether keyboard focus is in the 
Terminal, the Chat, or the Panel itself. This matters because keystrokes behave differently in each 
region (e.g., `Ctrl+C` in the terminal kills a process; in the chat it copies text).

**Send-context actions.** The user picks exactly what to send. At minimum, the following options are 
available:

Options & Descriptions:

Send visible viewport (text): The text content currently rendered in the terminal viewport, 
preserving layout where reasonable.

Send visible viewport (screenshot): A rasterized image of the terminal viewport, useful for TUIs, 
colored output, or anything where visual layout matters.

Send full scrollback buffer: The entire scrollback history as text, not just what's currently 
visible.

Send a specific command: Opens a picker listing previously executed shell commands. The user selects 
one, and the application sends the command itself plus its stdout and stderr to the agent as a 
single context block. Multi-select is supported so the user can send several related commands at 
once.

After selecting one of these, the context is attached to the user's next message to the agent (or 
sent immediately, depending on the user's chosen flow — see §4.4).

**Session actions.**

- **Reset conversation.** Clears the chat history and starts a fresh conversation with the agent. 
The terminal session is unaffected. Because this is destructive, it requires a confirmation step.

### 4.4 Sending Messages

The user composes a message in the chat input (optional — can be empty), optionally attaches one or 
more context items via the panel, and sends. The agent replies in the chat area. Context attachments 
are visible in the chat history attached to the message they went with, so the user can see what the 
agent actually saw.

## 5. Context Selection — Detailed Behavior

1. **Viewport vs. scrollback are distinct.** Users often want *just* what they can see (the failing error) rather than thousands of lines of scrollback. The default picks should reflect this — "visible viewport" is the common case.

2. **Text is the default, screenshot is an option.** Text is cheaper, searchable, and 
lets the agent quote back exact strings, thus being the default. 
Screenshots preserve color, cursor position, and TUI layout, thus being an option.
The user can decide which matters for the question they're asking.

3. **Command picker shows executed shell commands, not keystrokes.** If the user typed `lss -la`, 
backspaced, and retyped `ls -la`, only the executed command appears. Each entry in the picker shows 
at least: the command string, a timestamp, exit status, and a truncated preview of its output. 
Selecting one expands to show the full output before sending.

## 6. The Agent & Its Workspace

For this version, the agent harness runs inside a dedicated workspace directory on the user's machine at `$HOME/_terminal_agent_workspace/`. This is a convention-based boundary — not an OS-level sandbox — that keeps the agent's files in one well-known location, separate from the user's real projects.

- **Inside the workspace,** the agent may read from the internet, do web research, create and edit files, and write, install, and execute scripts and tools.
- **Outside the workspace,** the agent is not given access to the user's real filesystem, does not execute commands in the user's terminal session, and does not act on the user's real system.

The workspace is reset per conversation — resetting the conversation (§4.3) also clears `$HOME/_terminal_agent_workspace/` back to an empty state.

This design is intentional: the agent is helpful for research, reproducing issues, and drafting solutions, but the user remains the one who decides what — if anything — from the agent's work gets applied to their real system. Copying a script out of the chat and running it in the terminal is a deliberate, visible action.

## 7. Future Work

Extra features that are out of scope for this version:

* How to group commands in the picker: by time, by session / SSH host, by application, etc.
* For text context, we can also show the user a preview of the text before sending and allow them to edit the text before sending.
