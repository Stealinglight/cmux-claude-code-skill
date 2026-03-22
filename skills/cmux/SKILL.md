---
name: cmux
description: "You are running inside cmux, a native macOS terminal built on Ghostty. Use this skill to control cmux via the Bash tool: open browser tiles, split panes, spawn Claude Code agents in new surfaces, manage workspaces, automate the embedded WebKit browser, send notifications, and update sidebar metadata. Triggers on: cmux, cmux browser, cmux notify, cmux API, open browser in cmux, browser tile, split pane, spawn agent, cmux workspace, terminal automation, browser debugging in cmux, cmux sidebar, cmux status, worktree tile, new tab, new tile."
user-invocable: false
---

# cmux — You Are Here

You are running inside cmux, a native macOS terminal built on Ghostty for AI coding agents. You have full programmatic control of this terminal through the Bash tool.

Docs: https://cmux.com/docs (note: cmux.dev redirects to cmux.com)

## First: Confirm Your Environment

Before any cmux operation, run this check via the Bash tool:

```bash
echo "SURFACE=${CMUX_SURFACE_ID:-unset}"
echo "WORKSPACE=${CMUX_WORKSPACE_ID:-unset}"
echo "SOCKET=${CMUX_SOCKET_PATH:-/tmp/cmux.sock}"
command -v cmux &>/dev/null && echo "CLI=available" || echo "CLI=missing"
[ -S "${CMUX_SOCKET_PATH:-/tmp/cmux.sock}" ] && echo "SOCKET=connected" || echo "SOCKET=disconnected"
```

- If `CLI=missing` or `SOCKET=disconnected`: cmux is not available. Fall back to standard terminal behavior and tell the user.
- `$CMUX_SURFACE_ID` is YOUR surface. When you create splits, new surfaces get new IDs. Always track them.

## How to Run cmux Commands

ALL cmux operations use the **Bash tool**. Every command in this skill is meant to be executed via Bash.

- Use `--json` for machine-readable output when you need to parse results
- `cmux new-split` returns `OK surface:<id> workspace:<id>` — parse this to get the new surface ID directly
- Chain related commands with `&&` for atomicity
- Always include `\n` at the end of text sent via `send --surface` to execute the command
- Use `cmux list-pane-surfaces` to list surfaces in the current pane

---

## Action Protocols

### Open a Browser Tile

To open a browser pane alongside your terminal:

1. Create the browser split (returns the new surface ID):
   ```bash
   cmux browser open-split http://localhost:3000
   ```
   Output: `OK surface=surface:<ID> pane=pane:<ID> placement=split` — parse the surface ref from this.
2. Wait for the page to load:
   ```bash
   cmux browser surface:<ID> wait --load-state complete --timeout-ms 15000
   ```
4. Inspect the page:
   ```bash
   cmux browser surface:<ID> snapshot --interactive --compact
   ```
5. Take a screenshot if needed:
   ```bash
   cmux browser surface:<ID> screenshot --out /tmp/page.png
   ```

### Spawn a Claude Code Agent in a New Tile

To run another Claude Code instance in a new split pane:

1. Create a split (returns the new surface ID):
   ```bash
   cmux new-split right
   ```
   Output: `OK surface:<ID> workspace:<ID>` — parse the surface ref (e.g. `surface:27`).
2. Send the claude command to the new surface:
   ```bash
   cmux send --surface <new-surface-id> "claude \"<task description>\"\n"
   ```
   For autonomous operation, add `--dangerously-skip-permissions`:
   ```bash
   cmux send --surface <new-surface-id> "claude --dangerously-skip-permissions \"<task>\"\n"
   ```
4. Track progress via sidebar:
   ```bash
   cmux set-status agent "running" --icon brain --color "#007aff"
   ```

### Worktree + Agent Tile

For isolated parallel work combining git worktrees with cmux tiles:

1. Create a git worktree:
   ```bash
   git worktree add /tmp/worktree-<name> -b <branch-name>
   ```
2. Create a new workspace for isolation:
   ```bash
   cmux new-workspace
   ```
3. Get the new workspace and switch to it:
   ```bash
   WORKSPACE=$(cmux list-workspaces --json | jq -r '.workspaces[-1].id')
   cmux select-workspace --workspace "$WORKSPACE"
   ```
4. Send the agent to the worktree:
   ```bash
   cmux send "cd /tmp/worktree-<name> && claude \"<task>\"\n"
   ```

Note: You also have the `EnterWorktree` tool for working in a worktree in YOUR session. Use cmux splits + `send --surface` when you want a SEPARATE agent running in parallel.

### Agent Teams (Split-Pane Mode)

Claude Code Agent Teams can use cmux's native panes instead of tmux. This plugin includes a tmux shim that translates tmux commands to cmux equivalents.

**Setup** (run before starting Claude, or add to `~/.zshrc`):
```bash
eval "$(/path/to/cmux-claude-code-skill/bin/cmux-agent-teams-setup)"
```

This sets `$TMUX`, puts the shim in `$PATH`, and enables agent teams. Then use agent teams normally:
```
Create an agent team with 3 teammates to review this PR from different angles
```

Claude Code will create cmux split panes for each teammate, with full sidebar visibility, notification rings, and workspace integration.

For details, see `references/agent-patterns.md`.

### Create a Workspace

```bash
cmux new-workspace
cmux list-workspaces --json
cmux select-workspace --workspace <id>
```

### Split Panes

```bash
cmux new-split right    # horizontal split
cmux new-split down     # vertical split
```

`new-split` returns `OK surface:<id> workspace:<id>` — parse the surface ref from the output.

### Send Commands to Other Surfaces

```bash
cmux send --surface <id> "command here\n"
cmux send-key --surface <id> enter
```

### Open a New Tab in Current Pane

```bash
cmux new-surface                        # new terminal tab in current pane
cmux new-surface --type browser --url https://example.com  # new browser tab
cmux list-pane-surfaces                 # list all surfaces in current pane
```

### Notify the User

```bash
cmux notify --title "Build Complete" --body "All 42 tests passed"
```

Use after long-running operations complete. Also works with OSC escape sequences:
```bash
printf '\e]777;notify;Title;Body\a'
```

### Update Sidebar Status

```bash
# Status pills (key-value with optional icon and color)
cmux set-status build "compiling" --icon hammer --color "#ff9500"
cmux set-status build "done" --icon checkmark --color "#30d158"
cmux clear-status build

# Progress bar (0.0 to 1.0)
cmux set-progress 0.5 --label "Building..."
cmux set-progress 1.0 --label "Done"
cmux clear-progress

# Log entries (levels: info, progress, success, warning, error)
cmux log --level success -- "All tests passed"
cmux log --level error --source build "Compilation failed"
cmux clear-log
```

---

## Browser Automation

The cmux embedded browser is WebKit-based (WKWebView). Control it entirely via the `cmux browser` CLI through the Bash tool.

```bash
# Navigation
cmux browser open https://example.com               # new browser surface
cmux browser open-split https://localhost:3000       # browser in split pane
cmux browser surface:<ID> navigate 'https://example.com' --snapshot-after
# ^ IMPORTANT: Quote the URL when using flags like --snapshot-after
cmux browser surface:<ID> back
cmux browser surface:<ID> forward
cmux browser surface:<ID> reload

# Waiting
cmux browser surface:<ID> wait --load-state complete --timeout-ms 15000
cmux browser surface:<ID> wait --selector "#checkout" --timeout-ms 10000
cmux browser surface:<ID> wait --text "Order confirmed"

# Inspection
cmux browser surface:<ID> snapshot --interactive --compact
cmux browser surface:<ID> screenshot --out /tmp/screenshot.png
cmux browser surface:<ID> get title
cmux browser surface:<ID> get text "h1"
cmux browser surface:<ID> get html "main"
cmux browser surface:<ID> console list
cmux browser surface:<ID> errors list

# DOM interaction
cmux browser surface:<ID> click "button[type='submit']" --snapshot-after
cmux browser surface:<ID> fill "#email" --text "user@example.com"
cmux browser surface:<ID> type "#search" "query"
cmux browser surface:<ID> select "#region" "us-east"
cmux browser surface:<ID> press Enter

# JavaScript & CSS injection
cmux browser surface:<ID> eval "document.title"
cmux browser surface:<ID> eval "JSON.stringify({url: location.href, cookies: document.cookie.length})"
cmux browser surface:<ID> addstyle "#debug { display: none !important; }"

# Cookies & storage
cmux browser surface:<ID> cookies get
cmux browser surface:<ID> cookies set session_id abc123 --domain example.com
cmux browser surface:<ID> storage local get theme
cmux browser surface:<ID> storage local set theme dark

# Session persistence
cmux browser surface:<ID> state save /tmp/browser-state.json
cmux browser surface:<ID> state load /tmp/browser-state.json
```

For the complete browser command reference, see `references/browser-automation.md`.

---

## Quick Reference

| Action | Command |
|---|---|
| Identify self | `cmux identify --json` |
| Ping | `cmux ping` |
| List workspaces | `cmux list-workspaces --json` |
| New workspace | `cmux new-workspace` |
| Switch workspace | `cmux select-workspace --workspace <id>` |
| Close workspace | `cmux close-workspace --workspace <id>` |
| Split right / down | `cmux new-split right` / `down` (returns `OK surface:<id>`) |
| List pane surfaces | `cmux list-pane-surfaces` |
| Focus pane | `cmux focus-pane --pane <id>` |
| Send to surface | `cmux send --surface <id> "cmd\n"` |
| Send key to surface | `cmux send-key --surface <id> enter` |
| New surface/tab | `cmux new-surface [--type browser] [--url <url>]` |
| Close surface | `cmux close-surface --surface <id>` |
| Read screen | `cmux read-screen --surface <id>` |
| Open browser | `cmux browser open <URL>` |
| Open browser split | `cmux browser open-split <URL>` |
| Browser snapshot | `cmux browser surface:<id> snapshot --interactive --compact` |
| Browser screenshot | `cmux browser surface:<id> screenshot --out /tmp/shot.png` |
| Browser click | `cmux browser surface:<id> click "<selector>"` |
| Notify | `cmux notify --title "T" --body "B"` |
| Sidebar status | `cmux set-status <key> "<val>" --icon <i> --color "<hex>"` |
| Sidebar progress | `cmux set-progress <0-1> --label "..."` |
| Sidebar log | `cmux log --level <level> "message"` |
| Dump sidebar | `cmux sidebar-state` |

## cmux Hierarchy

```
Window > Workspace > Pane > Surface > Panel
```

- **Workspace** — sidebar tab (`Cmd+N`). Each has its own panes.
- **Pane** — split region (`Cmd+D` right, `Cmd+Shift+D` down).
- **Surface** — tab within a pane (`Cmd+T`). Identified by `$CMUX_SURFACE_ID`.
- **Panel** — terminal or browser content inside a surface.

## References

- `references/api-reference.md` — Complete CLI + socket API for all commands
- `references/browser-automation.md` — Full browser automation command reference
- `references/keyboard-shortcuts.md` — All keyboard shortcuts
- `references/agent-patterns.md` — Agent spawning, multi-agent orchestration, worktree patterns, hooks
