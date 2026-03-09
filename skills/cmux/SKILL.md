---
name: cmux
description: "Interact with cmux, the native macOS terminal built on Ghostty. Use for cmux CLI commands, socket API, browser automation, notifications, sidebar metadata, split panes, workspace management, and terminal automation. Triggers on: cmux, cmux browser, cmux notify, cmux API, open browser in cmux, split pane, cmux workspace, terminal automation, browser debugging in cmux, cmux sidebar, cmux status."
user-invocable: false
---

# cmux - Native macOS Terminal

cmux is a native macOS terminal built on Ghostty, designed for AI coding agents. It provides vertical tabs, notification rings, split panes, an embedded browser, and a socket-based control API.

Docs: https://cmux.dev/docs

---

## Hierarchy

cmux organizes terminals in a five-level hierarchy:

```
Window > Workspace (sidebar entry) > Pane (split region) > Surface (tab within pane) > Panel (terminal or browser)
```

- **Window** - macOS window (`Cmd+Shift+N`)
- **Workspace** - sidebar entry, called "tabs" in UI (`Cmd+N`, `Cmd+1-9`)
- **Pane** - split region within workspace (`Cmd+D` right, `Cmd+Shift+D` down)
- **Surface** - tab within a pane (`Cmd+T`), identified by `CMUX_SURFACE_ID`
- **Panel** - terminal or browser content inside a surface

## Detecting cmux

```bash
# Check socket
SOCK="${CMUX_SOCKET_PATH:-/tmp/cmux.sock}"
[ -S "$SOCK" ] && echo "Socket available"

# Check CLI
command -v cmux &>/dev/null && echo "cmux CLI available"

# Inside a cmux terminal (auto-set env vars)
[ -n "${CMUX_WORKSPACE_ID:-}" ] && [ -n "${CMUX_SURFACE_ID:-}" ] && echo "Inside cmux"
```

Key environment variables: `CMUX_WORKSPACE_ID`, `CMUX_SURFACE_ID`, `CMUX_SOCKET_PATH`, `TERM_PROGRAM=ghostty`, `TERM=xterm-ghostty`.

## Essential CLI Commands

| Command | Description |
|---|---|
| `cmux list-workspaces` | List all open workspaces |
| `cmux new-workspace` | Create a new workspace |
| `cmux select-workspace --workspace <id>` | Switch to workspace |
| `cmux new-split right` / `down` | Split pane horizontally/vertically |
| `cmux list-surfaces` | List surfaces in current workspace |
| `cmux focus-surface --surface <id>` | Focus a surface |
| `cmux send "command\n"` | Send text to focused terminal |
| `cmux send-surface --surface <id> "cmd"` | Send text to specific surface |
| `cmux send-key enter` | Send key press (enter, tab, escape, etc.) |
| `cmux notify --title "T" --body "B"` | Send notification |
| `cmux identify` | Show focused window/workspace/pane/surface |
| `cmux ping` | Check if cmux is running |

All CLI commands also accept `--json` for JSON output. Use `--socket PATH` to override socket path.

## Socket API

Socket at `/tmp/cmux.sock` (release) or `/tmp/cmux-debug.sock` (debug). Send newline-terminated JSON:

```json
{"id":"req-1","method":"workspace.list","params":{}}
// Response: {"id":"req-1","ok":true,"result":{"workspaces":[...]}}
```

## Browser Automation

cmux has a full embedded browser with automation via `cmux browser`. This is the primary way to debug web apps from within cmux.

### Opening a browser

```bash
cmux browser open https://localhost:3000          # New browser surface
cmux browser open-split https://localhost:3000     # Browser in split pane
```

### Core workflow: navigate, wait, inspect

```bash
cmux browser surface:2 navigate https://example.com --snapshot-after
cmux browser surface:2 wait --load-state complete --timeout-ms 15000
cmux browser surface:2 snapshot --interactive --compact
cmux browser surface:2 screenshot --out /tmp/page.png
```

### DOM interaction

```bash
cmux browser surface:2 click "button[type='submit']" --snapshot-after
cmux browser surface:2 fill "#email" --text "user@example.com"
cmux browser surface:2 type "#search" "query"
cmux browser surface:2 select "#region" "us-east"
cmux browser surface:2 press Enter
```

### Inspection

```bash
cmux browser surface:2 get title
cmux browser surface:2 get text "h1"
cmux browser surface:2 get html "main"
cmux browser surface:2 get value "#email"
cmux browser surface:2 is visible "#checkout"
cmux browser surface:2 find role button --name "Continue"
cmux browser surface:2 console list        # View console output
cmux browser surface:2 errors list         # View JS errors
```

### JavaScript eval

```bash
cmux browser surface:2 eval "document.title"
cmux browser surface:2 addstyle "#debug { display: none !important; }"
```

For the complete browser automation reference, see `references/browser-automation.md`.

## Notifications

### CLI

```bash
cmux notify --title "Build Done" --body "All tests passed"
cmux notify --title "Claude Code" --subtitle "Waiting" --body "Agent needs input"
```

### OSC 777 (works in any terminal)

```bash
printf '\e]777;notify;Title;Body\a'
```

### Claude Code hooks integration

Create `~/.claude/hooks/cmux-notify.sh`:

```bash
#!/bin/bash
[ -S /tmp/cmux.sock ] || exit 0
EVENT=$(cat)
EVENT_TYPE=$(echo "$EVENT" | jq -r '.event // "unknown"')
TOOL=$(echo "$EVENT" | jq -r '.tool_name // ""')
case "$EVENT_TYPE" in
    "Stop") cmux notify --title "Claude Code" --body "Session complete" ;;
    "PostToolUse") [ "$TOOL" = "Task" ] && cmux notify --title "Claude Code" --body "Agent finished" ;;
esac
```

Configure in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": ["~/.claude/hooks/cmux-notify.sh"],
    "PostToolUse": [{"matcher": "Task", "hooks": ["~/.claude/hooks/cmux-notify.sh"]}]
  }
}
```

## Sidebar Metadata

Surface build state, progress, and logs in the sidebar for at-a-glance visibility.

```bash
# Status pills
cmux set-status build "compiling" --icon hammer --color "#ff9500"
cmux clear-status build

# Progress bar (0.0 to 1.0)
cmux set-progress 0.5 --label "Building..."
cmux set-progress 1.0 --label "Done"
cmux clear-progress

# Log entries (levels: info, progress, success, warning, error)
cmux log "Build started"
cmux log --level error --source build "Compilation failed"
cmux log --level success -- "All 42 tests passed"
cmux clear-log

# Dump all sidebar state
cmux sidebar-state
```

## Configuration

cmux reads Ghostty config from `~/.config/ghostty/config` or `~/Library/Application Support/com.mitchellh.ghostty/config`.

```ini
font-family = JetBrains Mono
font-size = 14
theme = Dracula
scrollback-limit = 10000
unfocused-split-opacity = 0.7
```

App settings via `Cmd+,`: theme mode, automation/socket access level, browser link behavior.

### tmux passthrough

If using tmux inside cmux, enable passthrough for notifications:

```
# .tmux.conf
set -g allow-passthrough on
```

```bash
printf '\ePtmux;\e\e]777;notify;Title;Body\a\e\\'
```

## Claude Code + cmux Workflows

### Split browser for debugging
```bash
# Open your dev server in a split browser pane
cmux browser open-split http://localhost:3000
# Inspect, interact, screenshot from Claude Code
cmux browser surface:2 snapshot --interactive --compact
cmux browser surface:2 screenshot --out /tmp/debug.png
```

### Build notifications
```bash
npm run build && cmux notify --title "Build Success" --body "Ready" || cmux notify --title "Build Failed" --body "Check logs"
```

### Sidebar status during builds
```bash
cmux set-status build "running" --icon hammer
npm run build
cmux set-status build "done" --icon checkmark --color "#30d158"
```

## Reference Files

- **`references/api-reference.md`** - Complete CLI + socket API commands
- **`references/browser-automation.md`** - Full browser automation command reference
- **`references/keyboard-shortcuts.md`** - All keyboard shortcuts
