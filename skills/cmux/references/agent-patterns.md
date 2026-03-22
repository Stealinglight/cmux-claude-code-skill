# cmux Agent Patterns & Worktree Integration

Advanced reference for spawning agents, multi-agent orchestration, worktree lifecycles, and Claude Code hook integration within cmux.

---

## Spawning a Single Agent in a Split

Detailed protocol for launching a Claude Code instance in a new pane.

### Step-by-step

```bash
# 1. Record your own surface ID
MY_SURFACE="$CMUX_SURFACE_ID"

# 2. Create a split pane
cmux new-split right

# 3. Discover the new surface ID
NEW_SURFACE=$(cmux list-surfaces --json | jq -r ".surfaces[] | select(.id != \"$MY_SURFACE\") | .id" | head -1)

# 4. Send claude to the new surface
cmux send-surface --surface "$NEW_SURFACE" "claude \"Implement the login form component\"\n"

# 5. Track via sidebar
cmux set-status agent-1 "running" --icon brain --color "#007aff"
```

### Notes

- Always parse `list-surfaces --json` to find new surface IDs. Do not hardcode `surface:2`.
- Use `--dangerously-skip-permissions` only when the user has explicitly approved autonomous operation.
- The spawned agent runs independently. It has its own context and cannot share memory with you.

---

## Multi-Agent Orchestration

Spawn N agents across splits or workspaces, monitor via sidebar, collect results.

### Pattern: Parallel splits in one workspace

```bash
# Create 2 right splits for 3 total panes
cmux new-split right
cmux new-split right

# Get all surface IDs
SURFACES=$(cmux list-surfaces --json)

# Parse and assign tasks
SURFACE_A=$(echo "$SURFACES" | jq -r '.surfaces[1].id')
SURFACE_B=$(echo "$SURFACES" | jq -r '.surfaces[2].id')

cmux send-surface --surface "$SURFACE_A" "claude \"Write unit tests for auth module\"\n"
cmux send-surface --surface "$SURFACE_B" "claude \"Write unit tests for payment module\"\n"

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
cmux send-surface --surface "$AGENT_SURFACE" "claude \"Run tests and write results to /tmp/test-results.json\"\n"

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
cmux new-split right
REVIEWER=$(cmux list-surfaces --json | jq -r ".surfaces[] | select(.id != \"$CMUX_SURFACE_ID\") | .id" | head -1)
cmux send-surface --surface "$REVIEWER" "claude \"Review the changes in the current branch. Run git diff main and provide feedback on code quality, bugs, and security issues. Write your review to /tmp/code-review.md\"\n"

# Continue working while review happens
cmux set-status reviewer "reviewing" --icon eye --color "#ff9500"
```
