# cmux Claude Code Skill

> Give Claude Code native control of [cmux](https://cmux.dev) — open browser tiles, spawn agents in splits, manage workspaces, automate the embedded WebKit browser, and coordinate worktree-based parallel development.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![cmux](https://img.shields.io/badge/cmux-macOS_Terminal-000000)](https://cmux.dev)

---

## Why This Exists

When you run Claude Code inside [cmux](https://cmux.dev), Claude has no idea it's sitting in a terminal with an embedded browser, a notification system, sidebar metadata, and a full socket API. It just sees a shell.

**This skill changes that.** Install it, and Claude can:

- Open a browser pane next to your terminal and debug your web app visually
- Spawn other Claude Code agents in new tiles for parallel work
- Create git worktrees and run isolated agents on feature branches
- Send you a notification when a long task finishes
- Show build progress in the sidebar without cluttering your terminal
- Automate browser interactions — click, fill, screenshot, inspect — all from the CLI
- Inspect console output, cookies, localStorage, and inject JavaScript — all from the CLI

It turns cmux from "a nice terminal" into an integrated development environment that Claude actually knows how to use.

## What is cmux?

[cmux](https://cmux.dev) is a native macOS terminal built on [Ghostty](https://ghostty.org). It's designed for AI coding agents and features:

- **Vertical sidebar tabs** (workspaces) for organizing sessions
- **Split panes** with embedded browser surfaces
- **Notification system** with desktop alerts and workspace badges
- **Sidebar metadata** — status pills, progress bars, log entries
- **Socket API** for full programmatic control
- **Browser automation** — navigate, inspect, interact with web pages from the CLI

## Quick Install

```
/plugin marketplace add Stealinglight/cmux-claude-code-skill
/plugin install cmux-claude-code-skill@Stealinglight-cmux-claude-code-skill
```

Or clone and install locally:

```bash
git clone https://github.com/Stealinglight/cmux-claude-code-skill.git
# Then in Claude Code:
/plugin marketplace add /path/to/cmux-claude-code-skill
```

## What Claude Learns

Once installed, Claude Code automatically activates this skill when you mention cmux, browser debugging, terminal automation, or related topics. The skill is **action-oriented** — it gives Claude step-by-step protocols, not just reference documentation.

### Open a Browser Tile

Claude can open a browser pane right next to your terminal and debug interactively:

```bash
# Open your app in a split browser pane
cmux browser open-split http://localhost:3000

# Wait for load, inspect, interact
cmux browser surface:2 wait --load-state complete --timeout-ms 15000
cmux browser surface:2 snapshot --interactive --compact
cmux browser surface:2 fill "#email" --text "test@example.com"
cmux browser surface:2 click "button[type='submit']" --snapshot-after
cmux browser surface:2 screenshot --out /tmp/debug.png
```

The embedded browser is WebKit-based (WKWebView) with a full automation CLI — console, cookies, JS eval, DOM inspection, and more.

### Spawn Agents in New Tiles

Claude can create split panes and launch other Claude Code instances for parallel work:

```bash
# Create a split and send a new Claude agent to it
NEW_SURFACE=$(cmux new-split right | awk '{print $2}')
cmux send --surface "$NEW_SURFACE" "claude \"Run all tests and fix failures\"\n"
```

### Worktree + Agent Tiles

Combine git worktrees with cmux for isolated parallel development:

```bash
# Create a worktree, open a workspace, launch an agent
git worktree add /tmp/wt-feature -b feature/new-thing
cmux new-workspace
cmux send "cd /tmp/wt-feature && claude \"Implement the feature\"\n"
```

### Notifications & Sidebar

```bash
# Notify when tasks finish
cmux notify --title "Build Complete" --body "All 42 tests passed"

# Show progress in sidebar
cmux set-status build "compiling" --icon hammer --color "#ff9500"
cmux set-progress 0.5 --label "Building..."
cmux log --level success -- "All tests passed"
```

### Full Socket API Access

Every cmux command is available through both CLI and Unix socket. Claude knows both interfaces:

```python
import json, os, socket

SOCKET_PATH = os.environ.get("CMUX_SOCKET_PATH", "/tmp/cmux.sock")

def rpc(method, params=None, req_id=1):
    payload = {"id": req_id, "method": method, "params": params or {}}
    with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as sock:
        sock.connect(SOCKET_PATH)
        sock.sendall(json.dumps(payload).encode("utf-8") + b"\n")
        return json.loads(sock.recv(65536).decode("utf-8"))

rpc("notification.create", {"title": "Done", "body": "From Python!"})
```

## Skill Contents

```
skills/cmux/
├── SKILL.md                          # Core skill — action protocols, env detection, browser, agents
└── references/
    ├── api-reference.md              # Complete CLI + socket API (every command)
    ├── browser-automation.md         # Full browser automation reference
    ├── agent-patterns.md             # Agent spawning, worktree patterns, multi-agent orchestration
    └── keyboard-shortcuts.md         # All keyboard shortcuts by category
```

| File | Coverage |
|---|---|
| **SKILL.md** | Environment detection, Bash tool directives, action protocols (browser tiles, agent spawning, worktrees, workspaces, splits, notifications, sidebar), WebKit browser automation, quick reference |
| **api-reference.md** | Socket connection, workspace/surface/input/notification/sidebar commands, environment variables, configuration, Python + shell examples |
| **browser-automation.md** | Navigation, waiting, DOM interaction, inspection, JS eval, cookies/storage, tabs, console, dialogs, frames, downloads, common patterns |
| **agent-patterns.md** | Single/multi-agent spawning, worktree lifecycle, agent communication, Claude Code hooks, safety considerations, parallel development examples |
| **keyboard-shortcuts.md** | Workspaces, surfaces, split panes, browser, notifications, find, terminal, window shortcuts |

## How It Works

Claude Code skills are markdown files that Claude automatically loads when relevant triggers match in your conversation. This skill activates when you mention:

- cmux, browser tile, split pane, spawn agent
- Browser debugging, terminal automation
- Workspaces, worktree tiles
- Notifications, sidebar status

The `SKILL.md` file provides action-oriented protocols that tell Claude exactly how to execute cmux operations via the Bash tool, with reference files available for deep dives.

## Requirements

- [cmux](https://cmux.dev) (macOS 14.0+)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

## Contributing

Found a bug or want to add a workflow? PRs welcome.

1. Fork the repo
2. Make your changes
3. Submit a PR with a description of what you changed and why

## License

[MIT](LICENSE)

---

Built by [@Stealinglight](https://github.com/Stealinglight) — using AI coding agents to build better AI coding workflows.
