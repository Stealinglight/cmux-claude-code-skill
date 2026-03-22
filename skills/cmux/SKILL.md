---
name: cmux
description: "You are running inside cmux, a native macOS terminal built on Ghostty. Use this skill to control cmux via the Bash tool: open browser tiles, split panes, spawn Claude Code agents in new surfaces, manage workspaces, automate the embedded Chromium browser, send notifications, and update sidebar metadata. Triggers on: cmux, cmux browser, cmux notify, cmux API, open browser in cmux, browser tile, split pane, spawn agent, cmux workspace, terminal automation, browser debugging in cmux, cmux sidebar, cmux status, worktree tile, new tab, new tile."
user-invocable: false
---

# cmux — You Are Here

You are running inside cmux, a native macOS terminal built on Ghostty for AI coding agents. You have full programmatic control of this terminal through the Bash tool.

Docs: https://cmux.dev/docs

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
- After creating any split or surface, run `cmux list-surfaces --json` to capture the new surface ID
- Chain related commands with `&&` for atomicity
- Always include `\n` at the end of text sent via `send-surface` to execute the command

---

## Action Protocols

### Open a Browser Tile

To open a browser pane alongside your terminal:

1. Create the browser split:
   ```bash
   cmux browser open-split http://localhost:3000
   ```
2. Get the new surface ID:
   ```bash
   cmux list-surfaces --json
   ```
   Parse the JSON output. The new browser surface will have a different ID from your `$CMUX_SURFACE_ID`.
3. Wait for the page to load:
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

1. Create a split:
   ```bash
   cmux new-split right
   ```
2. Discover the new surface ID:
   ```bash
   cmux list-surfaces --json
   ```
   The new surface is the one that is NOT your `$CMUX_SURFACE_ID`.
3. Send the claude command to the new surface:
   ```bash
   cmux send-surface --surface <new-surface-id> "claude \"<task description>\"\n"
   ```
   For autonomous operation, add `--dangerously-skip-permissions`:
   ```bash
   cmux send-surface --surface <new-surface-id> "claude --dangerously-skip-permissions \"<task>\"\n"
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

Note: You also have the `EnterWorktree` tool for working in a worktree in YOUR session. Use cmux splits + `send-surface` when you want a SEPARATE agent running in parallel.

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

After splitting, always run `cmux list-surfaces --json` to capture the new surface ID.

### Send Commands to Other Surfaces

```bash
cmux send-surface --surface <id> "command here\n"
cmux send-key-surface --surface <id> enter
```

### Open a New Tab in Current Pane

```bash
cmux list-surfaces --json   # note current surfaces
# Cmd+T creates a new surface tab, or use the API:
cmux send-key tab           # if needed
```

To send a command to a specific surface tab, always target by surface ID.

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

The cmux embedded browser is full Chromium. You have three ways to interact with it:

### Option A: cmux browser CLI (via Bash tool)

Best for quick inspections, screenshots, simple interactions.

```bash
# Navigation
cmux browser open https://example.com               # new browser surface
cmux browser open-split https://localhost:3000       # browser in split pane
cmux browser surface:<ID> navigate https://example.com --snapshot-after
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

# JavaScript
cmux browser surface:<ID> eval "document.title"
cmux browser surface:<ID> addstyle "#debug { display: none !important; }"
```

For the complete browser command reference, see `references/browser-automation.md`.

### Option B: Playwright MCP

Best for complex multi-step browser automation, form flows, and testing. If Playwright MCP tools are available (check your available tools for `mcp__playwright__*`), they can drive the same Chromium instance that cmux opened.

1. Open a browser surface with cmux: `cmux browser open-split <URL>`
2. Use Playwright MCP tools (`browser_navigate`, `browser_snapshot`, `browser_click`, etc.) for complex automation
3. Use cmux browser CLI for quick checks alongside Playwright

### Option C: Chrome DevTools MCP

Best for performance profiling, network inspection, and memory analysis. If Chrome DevTools MCP tools are available (`mcp__chrome-devtools__*`), use them for DevTools-level debugging on the cmux browser.

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
| Split right / down | `cmux new-split right` / `down` |
| List surfaces | `cmux list-surfaces --json` |
| Focus surface | `cmux focus-surface --surface <id>` |
| Send to surface | `cmux send-surface --surface <id> "cmd\n"` |
| Send key to surface | `cmux send-key-surface --surface <id> enter` |
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
