# cmux Agent Patterns & Worktree Integration

Advanced reference for spawning agents, multi-agent orchestration, worktree lifecycles, and Claude Code hook integration within cmux.

---

## Spawning a Single Agent in a Split

Detailed protocol for launching a Claude Code instance in a new pane.

### Step-by-step

```bash
# 1. Create a split pane (returns the new surface ID directly)
SPLIT_OUTPUT=$(cmux new-split right)
# Output: "OK surface:27 workspace:10"
NEW_SURFACE=$(echo "$SPLIT_OUTPUT" | awk '{print $2}')

# 2. Send claude to the new surface
cmux send --surface "$NEW_SURFACE" "claude \"Implement the login form component\"\n"

# 3. Track via sidebar
cmux set-status agent-1 "running" --icon brain --color "#007aff"
```

### Notes

- `cmux new-split` returns `OK surface:<id> workspace:<id>` — parse the surface ref directly. No need for a follow-up list command.
- Use `--dangerously-skip-permissions` only when the user has explicitly approved autonomous operation.
- The spawned agent runs independently. It has its own context and cannot share memory with you.

---

## Multi-Agent Orchestration

Spawn N agents across splits or workspaces, monitor via sidebar, collect results.

### Pattern: Parallel splits in one workspace

```bash
# Create 2 right splits — each returns the new surface ID
SPLIT_A=$(cmux new-split right | awk '{print $2}')
SPLIT_B=$(cmux new-split right | awk '{print $2}')

# Send tasks to each surface
cmux send --surface "$SPLIT_A" "claude \"Write unit tests for auth module\"\n"
cmux send --surface "$SPLIT_B" "claude \"Write unit tests for payment module\"\n"

# Track all agents
cmux set-status agent-auth "testing" --icon flask --color "#ff9500"
cmux set-status agent-pay "testing" --icon flask --color "#ff9500"
```

### Pattern: Separate workspaces per agent

Better for complex tasks where each agent needs a full screen.

```bash
# Agent 1: new workspace
cmux new-workspace
WS1=$(cmux list-workspaces --json | jq -r '.workspaces[-1].id')
cmux select-workspace --workspace "$WS1"
cmux send "claude \"Refactor the database layer\"\n"

# Agent 2: another workspace
cmux new-workspace
WS2=$(cmux list-workspaces --json | jq -r '.workspaces[-1].id')
cmux select-workspace --workspace "$WS2"
cmux send "claude \"Add API rate limiting\"\n"

# Return to your workspace
cmux select-workspace --workspace "$CMUX_WORKSPACE_ID"

# Monitor from sidebar (visible across workspaces)
cmux set-status ws1-agent "refactoring" --icon hammer
cmux set-status ws2-agent "rate-limiting" --icon shield
```

---

## Worktree Lifecycle

Git worktrees provide isolated working directories. Combined with cmux, each agent can work on its own branch without conflicts.

### Create worktree + workspace + agent

```bash
# 1. Create worktree on a new branch
BRANCH="feature/login-form"
WORKTREE="/tmp/worktree-login-form"
git worktree add "$WORKTREE" -b "$BRANCH"

# 2. Create a dedicated workspace
cmux new-workspace
WS=$(cmux list-workspaces --json | jq -r '.workspaces[-1].id')
cmux select-workspace --workspace "$WS"

# 3. Launch agent in the worktree
cmux send "cd $WORKTREE && claude \"Implement login form with validation\"\n"

# 4. Return to your workspace
cmux select-workspace --workspace "$CMUX_WORKSPACE_ID"
```

### Cleanup after merge

```bash
# After the agent's branch is merged
git worktree remove "$WORKTREE"
git branch -d "$BRANCH"
cmux close-workspace --workspace "$WS"
cmux clear-status agent-login
```

### When to use worktrees vs. regular splits

| Scenario | Use |
|---|---|
| Quick task, same branch | Split pane with `new-split` |
| Feature branch, isolated changes | Worktree + workspace |
| Multiple parallel features | Multiple worktrees + workspaces |
| You need to work in a worktree yourself | `EnterWorktree` tool (stays in your surface) |

---

## Agent Communication Patterns

Spawned agents run independently. Coordinate via:

### File-based signals

```bash
# Agent writes result to a known path
cmux send --surface "$AGENT_SURFACE" "claude \"Run tests and write results to /tmp/test-results.json\"\n"

# Later, read the results
cat /tmp/test-results.json
```

### Notification-based signals

```bash
# Agent sends notification when done (agent can run this itself)
cmux notify --title "Agent Done" --body "Auth tests: 42 passed, 0 failed"
```

### Sidebar-based monitoring

```bash
# Set progress as agents complete
cmux set-progress 0.33 --label "1/3 agents done"
cmux set-progress 0.66 --label "2/3 agents done"
cmux set-progress 1.0 --label "All agents done"
```

---

## Claude Code Hook Integration

Set up hooks so cmux notifies you when Claude Code sessions complete.

### Create the hook script

Write to `~/.claude/hooks/cmux-notify.sh`:

```bash
#!/bin/bash
# Only run if cmux is available
[ -S "${CMUX_SOCKET_PATH:-/tmp/cmux.sock}" ] || exit 0
command -v cmux &>/dev/null || exit 0

EVENT=$(cat)
EVENT_TYPE=$(echo "$EVENT" | jq -r '.event // "unknown"')
TOOL=$(echo "$EVENT" | jq -r '.tool_name // ""')

case "$EVENT_TYPE" in
    "Stop")
        cmux notify --title "Claude Code" --body "Session complete"
        cmux set-status claude "idle" --icon checkmark --color "#30d158"
        ;;
    "PostToolUse")
        [ "$TOOL" = "Task" ] && cmux notify --title "Claude Code" --body "Agent task finished"
        ;;
esac
```

```bash
chmod +x ~/.claude/hooks/cmux-notify.sh
```

### Configure in settings.json

Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [{ "command": "~/.claude/hooks/cmux-notify.sh" }],
    "PostToolUse": [{
      "matcher": { "tool_name": "Task" },
      "command": "~/.claude/hooks/cmux-notify.sh"
    }]
  }
}
```

### Sidebar status on session lifecycle

Add to a `SessionStart` hook:

```bash
cmux set-status claude "active" --icon brain --color "#007aff"
cmux log --level info "Claude Code session started"
```

---

## Safety Considerations

- **`--dangerously-skip-permissions`**: Only use when the user has explicitly approved autonomous agent operation. This flag lets the spawned agent modify files, run commands, and make network requests without confirmation.
- **Task scoping**: Give spawned agents specific, bounded tasks. "Implement the login form" is better than "work on the frontend."
- **Resource limits**: Each spawned Claude Code instance uses its own context window and API quota. Avoid spawning excessive agents.
- **Worktree cleanup**: Always clean up worktrees after merging to avoid disk bloat. List with `git worktree list`.
- **Surface targeting**: Always verify surface IDs before sending commands. Sending to the wrong surface can disrupt another agent's work.

---

## Claude Code Agent Teams with cmux

Claude Code's experimental Agent Teams feature supports split-pane mode via tmux. This plugin includes a tmux shim that redirects all tmux commands to cmux, so agent teams creates native cmux panes instead.

### Setup

```bash
# Add to ~/.zshrc or run before starting Claude:
eval "$(/path/to/cmux-claude-code-skill/bin/cmux-agent-teams-setup)"
```

This does three things:
1. Puts the `bin/tmux` shim ahead of real tmux in `$PATH`
2. Sets `$TMUX=cmux-shim` so Claude Code detects "inside tmux" mode
3. Enables `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

### Usage

Start Claude normally, then ask for an agent team:

```
Create an agent team with 3 teammates:
- One focused on security review
- One checking performance impact
- One validating test coverage
```

Claude Code creates cmux split panes for each teammate. You get:
- **Sidebar visibility**: each teammate's workspace shows in the cmux sidebar
- **Notification rings**: teammates trigger cmux notifications when they need attention
- **Click to interact**: click any pane to message that teammate directly
- **Shift+Down**: cycle through teammates in the lead's pane

### How the shim works

The `bin/tmux` script intercepts tmux commands and translates them:

| tmux command | cmux equivalent |
|---|---|
| `tmux split-window -h` | `cmux new-split right` |
| `tmux split-window -v` | `cmux new-split down` |
| `tmux send-keys -t <pane> <text> Enter` | `cmux send --surface <surface> "text\n"` |
| `tmux list-panes` | `cmux list-pane-surfaces` |
| `tmux kill-pane -t <pane>` | `cmux close-surface --surface <surface>` |
| `tmux display-message -p "#{pane_id}"` | `cmux identify --json` (parse surface ref) |
| `tmux select-pane -t <pane> -T <title>` | `cmux rename-tab --surface <surface> <title>` |
| `tmux has-session` | `cmux ping` |

Commands without a cmux equivalent (set-option, select-layout, break-pane) are absorbed silently. When not inside cmux, the shim passes through to the real tmux binary.

### Limitations

- **Layout**: cmux manages layout automatically. The tmux `select-layout tiled` and `main-vertical` commands are absorbed. Pane sizes may not be perfectly balanced.
- **Pane border styling**: tmux pane border colors and titles translate to cmux tab names. The visual presentation differs from tmux's border styling.
- **External sessions**: the shim maps tmux `new-session` to `cmux new-workspace`. The semantics differ slightly.

---

## Example: Parallel Feature Development

Full workflow for developing three features in parallel:

```bash
# Create 3 worktrees
git worktree add /tmp/wt-auth -b feature/auth
git worktree add /tmp/wt-payments -b feature/payments
git worktree add /tmp/wt-dashboard -b feature/dashboard

# Create workspaces and launch agents
for FEATURE in auth payments dashboard; do
    cmux new-workspace
    WS=$(cmux list-workspaces --json | jq -r '.workspaces[-1].id')
    cmux select-workspace --workspace "$WS"
    cmux send "cd /tmp/wt-$FEATURE && claude --dangerously-skip-permissions \"Implement the $FEATURE feature per the spec in docs/$FEATURE.md\"\n"
    cmux set-status "agent-$FEATURE" "working" --icon hammer --color "#007aff"
done

# Return to your workspace
cmux select-workspace --workspace "$CMUX_WORKSPACE_ID"
cmux set-progress 0.0 --label "0/3 features complete"
```

## Example: Agent-Assisted Code Review

Spawn a reviewer agent alongside your work:

```bash
# Split and launch reviewer
REVIEWER=$(cmux new-split right | awk '{print $2}')
cmux send --surface "$REVIEWER" "claude \"Review the changes in the current branch. Run git diff main and provide feedback on code quality, bugs, and security issues. Write your review to /tmp/code-review.md\"\n"

# Continue working while review happens
cmux set-status reviewer "reviewing" --icon eye --color "#ff9500"
```
