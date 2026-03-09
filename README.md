# cmux Claude Code Skill

> Give Claude Code native control of [cmux](https://cmux.dev) — browser automation, split panes, notifications, and sidebar metadata for the Ghostty-powered macOS terminal.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![cmux](https://img.shields.io/badge/cmux-macOS_Terminal-000000)](https://cmux.dev)

---

## Why This Exists

When you run Claude Code inside [cmux](https://cmux.dev), Claude has no idea it's sitting in a terminal with an embedded browser, a notification system, sidebar metadata, and a full socket API. It just sees a shell.

**This skill changes that.** Install it, and Claude can:

- Open a browser pane next to your terminal and debug your web app visually
- Send you a notification when a long task finishes
- Show build progress in the sidebar without cluttering your terminal
- Automate browser interactions — click, fill, screenshot, inspect — all from the CLI

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

Once installed, Claude Code automatically activates this skill when you mention cmux, browser debugging, terminal automation, or related topics. Here's what it gains:

### Terminal & Workspace Management

Claude can create workspaces, split panes, send commands to specific surfaces, and navigate between them — all programmatically.

```bash
# Claude can split your workspace and run commands in parallel
cmux new-split right
cmux send-surface --surface surface:2 "npm run dev\n"
```

### Browser Automation for Debugging

This is the headline feature. Claude can open a browser pane right next to your terminal, navigate to your dev server, and debug interactively:

```bash
# Open your app in a split browser pane
cmux browser open-split http://localhost:3000

# Wait for it to load, then inspect the page
cmux browser surface:2 wait --load-state complete --timeout-ms 15000
cmux browser surface:2 snapshot --interactive --compact

# Fill a form and verify the result
cmux browser surface:2 fill "#email" --text "test@example.com"
cmux browser surface:2 click "button[type='submit']" --snapshot-after
cmux browser surface:2 wait --text "Welcome"

# Grab a screenshot for reference
cmux browser surface:2 screenshot --out /tmp/debug.png
```

No need to switch to Chrome DevTools. No need to describe what you see on screen. Claude can see and interact with your web app directly.

### Notifications When Tasks Complete

Never wonder "is Claude done yet?" again. With the included hook integration, Claude notifies you through cmux's notification system:

```bash
# Simple notification from any script
cmux notify --title "Build Complete" --body "All 42 tests passed"

# Or use OSC 777 escape codes from any language
printf '\e]777;notify;Task Done;Ready for review\a'
```

The skill includes a ready-to-use Claude Code hook script that sends notifications on session completion and agent task finish. Just drop it in `~/.claude/hooks/`.

### Sidebar Status & Progress

Surface build state without terminal noise. Claude can set status pills, progress bars, and log entries in the cmux sidebar:

```bash
# Show what's happening at a glance
cmux set-status build "compiling" --icon hammer --color "#ff9500"
cmux set-progress 0.5 --label "Building..."
cmux log --level success -- "All tests passed"

# Clean up when done
cmux set-status build "done" --icon checkmark --color "#30d158"
cmux clear-progress
```

### Full Socket API Access

Every cmux command is available through both CLI and Unix socket. Claude knows both interfaces and can write automation scripts in Python or shell:

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
├── SKILL.md                          # Core skill — concepts, essential commands, workflows
└── references/
    ├── api-reference.md              # Complete CLI + socket API (every command)
    ├── browser-automation.md         # Full browser automation reference
    └── keyboard-shortcuts.md         # All keyboard shortcuts by category
```

| File | Coverage |
|---|---|
| **SKILL.md** | Hierarchy, detection, CLI quick-reference, browser essentials, notifications, sidebar metadata, Claude Code integration patterns |
| **api-reference.md** | Socket connection, workspace/surface/input/notification/sidebar commands, environment variables, Python + shell examples |
| **browser-automation.md** | Navigation, waiting, DOM interaction, inspection, JS eval, cookies/storage, tabs, console, dialogs, frames, downloads, common patterns |
| **keyboard-shortcuts.md** | Workspaces, surfaces, split panes, browser, notifications, find, terminal, window shortcuts |

## How It Works

Claude Code skills are markdown files that Claude automatically loads when relevant triggers match in your conversation. This skill activates when you mention:

- cmux, cmux browser, cmux API
- Split panes, terminal automation
- Browser debugging in cmux
- Notifications, sidebar status

The `SKILL.md` file provides the core knowledge (~1,000 words, loads fast), with three reference files available for deep dives into specific command categories.

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
