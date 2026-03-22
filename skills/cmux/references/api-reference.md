# cmux API Reference

Complete CLI and socket API reference for cmux. The main `SKILL.md` covers essential commands and action protocols. Consult this file for the full command set, socket API details, advanced flags, and configuration.

## Socket Connection

| Build | Path |
|---|---|
| Release | `/tmp/cmux.sock` |
| Debug | `/tmp/cmux-debug.sock` |
| Tagged debug | `/tmp/cmux-debug-<tag>.sock` |

Override with `CMUX_SOCKET_PATH` environment variable.

### Request format

Send newline-terminated JSON:

```json
{"id":"req-1","method":"workspace.list","params":{}}
```

Response:

```json
{"id":"req-1","ok":true,"result":{"workspaces":[...]}}
```

### Access modes

| Mode | Description | How to enable |
|---|---|---|
| **Off** | Socket disabled | Settings UI or `CMUX_SOCKET_MODE=off` |
| **cmux processes only** | Only cmux-spawned processes can connect | Default |
| **allowAll** | Any local process can connect | `CMUX_SOCKET_MODE=allowAll` (env only) |

## CLI Options

| Flag | Description |
|---|---|
| `--socket PATH` | Custom socket path |
| `--json` | JSON output |
| `--window ID` | Target window |
| `--workspace ID` | Target workspace |
| `--surface ID` | Target surface |
| `--id-format refs\|uuids\|both` | Identifier format in JSON |

---

## Workspace Commands

### list-workspaces

```bash
cmux list-workspaces
cmux list-workspaces --json
```

Socket: `{"id":"ws-list","method":"workspace.list","params":{}}`

### new-workspace

```bash
cmux new-workspace
```

Socket: `{"id":"ws-new","method":"workspace.create","params":{}}`

### select-workspace

```bash
cmux select-workspace --workspace <id>
```

Socket: `{"id":"ws-select","method":"workspace.select","params":{"workspace_id":"<id>"}}`

### current-workspace

```bash
cmux current-workspace
cmux current-workspace --json
```

Socket: `{"id":"ws-current","method":"workspace.current","params":{}}`

### close-workspace

```bash
cmux close-workspace --workspace <id>
```

Socket: `{"id":"ws-close","method":"workspace.close","params":{"workspace_id":"<id>"}}`

---

## Split / Surface Commands

### new-split

Create a new split pane. Directions: `left`, `right`, `up`, `down`.

```bash
cmux new-split right
cmux new-split down
```

Socket: `{"id":"split-new","method":"surface.split","params":{"direction":"right"}}`

### list-surfaces

```bash
cmux list-surfaces
cmux list-surfaces --json
```

Socket: `{"id":"surface-list","method":"surface.list","params":{}}`

### focus-surface

```bash
cmux focus-surface --surface <id>
```

Socket: `{"id":"surface-focus","method":"surface.focus","params":{"surface_id":"<id>"}}`

---

## Input Commands

### send

Send text input to the focused terminal.

```bash
cmux send "echo hello"
cmux send "ls -la\n"
```

Socket: `{"id":"send-text","method":"surface.send_text","params":{"text":"echo hello\n"}}`

### send-key

Send a key press. Keys: `enter`, `tab`, `escape`, `backspace`, `delete`, `up`, `down`, `left`, `right`.

```bash
cmux send-key enter
```

Socket: `{"id":"send-key","method":"surface.send_key","params":{"key":"enter"}}`

### send-surface

Send text to a specific surface.

```bash
cmux send-surface --surface <id> "command"
```

Socket: `{"id":"send-surface","method":"surface.send_text","params":{"surface_id":"<id>","text":"command"}}`

### send-key-surface

```bash
cmux send-key-surface --surface <id> enter
```

Socket: `{"id":"send-key-surface","method":"surface.send_key","params":{"surface_id":"<id>","key":"enter"}}`

---

## Notification Commands

### notify

```bash
cmux notify --title "Title" --body "Body"
cmux notify --title "T" --subtitle "S" --body "B"
```

Socket: `{"id":"notify","method":"notification.create","params":{"title":"Title","subtitle":"S","body":"Body"}}`

### list-notifications

```bash
cmux list-notifications
cmux list-notifications --json
```

Socket: `{"id":"notif-list","method":"notification.list","params":{}}`

### clear-notifications

```bash
cmux clear-notifications
```

Socket: `{"id":"notif-clear","method":"notification.clear","params":{}}`

---

## Sidebar Metadata Commands

### set-status

Set a sidebar status pill with a unique key.

```bash
cmux set-status build "compiling" --icon hammer --color "#ff9500"
cmux set-status deploy "v1.2.3" --workspace workspace:2
```

Socket: `set_status build compiling --icon=hammer --color=#ff9500 --tab=<workspace-uuid>`

### clear-status

```bash
cmux clear-status build
```

Socket: `clear_status build --tab=<workspace-uuid>`

### list-status

```bash
cmux list-status
```

Socket: `list_status --tab=<workspace-uuid>`

### set-progress

Set a progress bar (0.0 to 1.0).

```bash
cmux set-progress 0.5 --label "Building..."
cmux set-progress 1.0 --label "Done"
```

Socket: `set_progress 0.5 --label=Building... --tab=<workspace-uuid>`

### clear-progress

```bash
cmux clear-progress
```

Socket: `clear_progress --tab=<workspace-uuid>`

### log

Append a log entry. Levels: `info`, `progress`, `success`, `warning`, `error`.

```bash
cmux log "Build started"
cmux log --level error --source build "Compilation failed"
cmux log --level success -- "All 42 tests passed"
```

Socket: `log --level=error --source=build --tab=<workspace-uuid> -- Compilation failed`

### clear-log

```bash
cmux clear-log
```

Socket: `clear_log --tab=<workspace-uuid>`

### list-log

```bash
cmux list-log
cmux list-log --limit 5
```

Socket: `list_log --limit=5 --tab=<workspace-uuid>`

### sidebar-state

Dump all sidebar metadata (cwd, git branch, ports, status, progress, logs).

```bash
cmux sidebar-state
cmux sidebar-state --workspace workspace:2
```

Socket: `sidebar_state --tab=<workspace-uuid>`

---

## Utility Commands

### ping

```bash
cmux ping
```

Socket: `{"id":"ping","method":"system.ping","params":{}}` -> `{"id":"ping","ok":true,"result":{"pong":true}}`

### capabilities

```bash
cmux capabilities
cmux capabilities --json
```

Socket: `{"id":"caps","method":"system.capabilities","params":{}}`

### identify

Show focused window/workspace/pane/surface context.

```bash
cmux identify
cmux identify --json
```

Socket: `{"id":"identify","method":"system.identify","params":{}}`

---

## Environment Variables

| Variable | Description |
|---|---|
| `CMUX_SOCKET_PATH` | Override socket path |
| `CMUX_SOCKET_ENABLE` | Force enable/disable socket (`1`/`0`, `true`/`false`, `on`/`off`) |
| `CMUX_SOCKET_MODE` | Override access mode (`cmuxOnly`, `allowAll`, `off`) |
| `CMUX_WORKSPACE_ID` | Auto-set: current workspace ID |
| `CMUX_SURFACE_ID` | Auto-set: current surface ID |
| `TERM_PROGRAM` | Set to `ghostty` |
| `TERM` | Set to `xterm-ghostty` |

---

## Client Examples

### Python

```python
import json, os, socket

SOCKET_PATH = os.environ.get("CMUX_SOCKET_PATH", "/tmp/cmux.sock")

def rpc(method, params=None, req_id=1):
    payload = {"id": req_id, "method": method, "params": params or {}}
    with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as sock:
        sock.connect(SOCKET_PATH)
        sock.sendall(json.dumps(payload).encode("utf-8") + b"\n")
        return json.loads(sock.recv(65536).decode("utf-8"))

print(rpc("workspace.list", req_id="ws"))
print(rpc("notification.create", {"title": "Hello", "body": "From Python!"}, req_id="notify"))
```

### Shell

```bash
#!/bin/bash
SOCK="${CMUX_SOCKET_PATH:-/tmp/cmux.sock}"

cmux_cmd() {
    printf "%s\n" "$1" | nc -U "$SOCK"
}

cmux_cmd '{"id":"ws","method":"workspace.list","params":{}}'
cmux_cmd '{"id":"notify","method":"notification.create","params":{"title":"Done","body":"Task complete"}}'
```

### Build script with notification

```bash
#!/bin/bash
npm run build
if [ $? -eq 0 ]; then
    cmux notify --title "Build Success" --body "Ready to deploy"
else
    cmux notify --title "Build Failed" --body "Check the logs"
fi
```

---

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
