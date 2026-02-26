---
name: agent-swarm-protocol
description: "Coordinate multiple AI agents via tmux split panes. One Queen👑 orchestrates, Workers🐝 execute — all visible in real time. Three collaboration modes: independent work, discussion/debate, and pipeline chains. Requires tmux. ACTIVATE when user says: 'swarm', 'team', 'agents', 'split up', 'in parallel', 'simultaneously', 'at the same time', 'divide and conquer', 'brainstorm together', 'research while coding', 'I need multiple agents', 'clone yourself', 'work together', 'discuss', 'debate', 'peer review', 'critique each other', 'take turns', 'draft then review'. SKIP when: single-task request, user doesn't mention parallelism or collaboration, no tmux session available."
license: Complete terms in LICENSE.txt
metadata:
  author: Xi Zhang
  version: '0.0.1'
  tags: [agent swarm, tmux, parallel agent teams, CLI collaboration]
---

# Agent Swarm Protocol

Orchestrate multiple AI agents using tmux split panes. One **Queen👑** pane coordinates; **Worker🐝** panes execute. Three collaboration modes: independent work, discussion/debate, and pipeline chains. Pure tmux — no scripts, no daemons, no extra dependencies.

**Activate only when ALL are true:**
1. User wants **multiple agents** (2+ workstreams or roles)
2. Task benefits from **parallel or collaborative** work
3. User wants **real-time observation** in one terminal
4. A **tmux session** exists (or user agrees to start one)

**Skip** for single-thread tasks or when user doesn't need to observe intermediate work.

**Rules:**
- Verify each step with `capture-pane`. Never hard-code pane IDs — always discover them.
- **NEVER use `sleep`, `wait`, or any blocking delay.** No `sleep 5`, no `sleep 10`, no retry loops with waits. These freeze the entire conversation — the user cannot talk to you while you sleep.
- **No polling loops.** After dispatching tasks, check each worker ONCE with `capture-pane`, report status to user, then STOP and return to conversation. If workers aren't done yet, tell the user and let them ask you to check again later.

---

## Setup

### 1. Preflight

```bash
echo $TMUX
```

- **Verify:** Non-empty output.
- **If fail:** Stop. Tell user to run `tmux new -s swarm` first.

### 2. Discover & Bind Pane IDs

```bash
QUEEN=$(tmux display-message -p "#{pane_id}")
echo $QUEEN    # e.g. %0, %4, %17 — do NOT assume %0
```

Create workers and capture their IDs:

```bash
W1=$(tmux split-window -P -F "#{pane_id}")
W2=$(tmux split-window -P -F "#{pane_id}")
tmux select-layout main-vertical          # for 3+ workers, use 'tiled' instead
tmux resize-pane -t $QUEEN -R 20
```

Label panes with titles, enable border labels, and start output logging:

```bash
# Title markers — emoji in title for visual clarity
tmux select-pane -t $QUEEN -T "👑 Queen"
tmux select-pane -t $W1 -T "🐝 researcher"
tmux select-pane -t $W2 -T "🐝 coder"

# Show colored labels on pane borders (like CCB-style tags)
tmux set-option pane-border-status top
tmux set-option pane-border-format "#[fg=black,bg=green,bold] #{pane_index} #{pane_title} #[default]"
tmux set-option pane-active-border-style "fg=green,bold"
tmux set-option pane-border-style "fg=colour240"

# Output logging
mkdir -p ./swarm-logs
tmux pipe-pane -t $W1 "cat >> ./swarm-logs/researcher.log"
tmux pipe-pane -t $W2 "cat >> ./swarm-logs/coder.log"
```

All subsequent commands use `$QUEEN`, `$W1`, `$W2` — never literal pane IDs. Title markers allow recovery if pane IDs change (see `references/reliability.md`).

- **Verify:** `tmux list-panes -F "#{pane_id} #{pane_title}"` shows all panes with correct role titles.
- **If fail:** Re-run `split-window`. If still fails, check tmux session.

### 3. Sync Environment

New panes inherit queen's CWD but may lack the right virtualenv/PATH. Check and fix:

```bash
tmux capture-pane -p -t $W1 -S -3           # check worker's prompt/env
tmux send-keys -t $W1 -l "<activation>" && tmux send-keys -t $W1 Enter   # e.g. "conda activate myenv"
```

- **Verify:** `capture-pane` shows expected environment markers. **If fail:** re-run; re-verify.

### 4. Launch CLI

```bash
tmux send-keys -t $W1 -l "<cli-launch-command>" && tmux send-keys -t $W1 Enter
```

Examples: `EvoSci`, `codex`, `gemini`, `python3`. See `references/cli-compatibility.md`.

- **Verify:** `tmux capture-pane -p -t $W1 -S -5` shows CLI prompt. **If fail:** Kill pane, respawn, retry.
- **Important:** Some CLIs (e.g. Claude Code) prompt for permission on every shell command. Launch with permission-skip flags when available.

### 5. Handshake

**Mode A (Independent Work) — worker knows Queen only:**

```bash
tmux send-keys -t $W1 -l "[@Queen👑] I am Queen at $QUEEN. You are researcher at $W1. Prefix all replies with [@researcher🐝]. To reply run: tmux send-keys -t $QUEEN -l '[@researcher🐝] your message' && tmux send-keys -t $QUEEN Enter. Work directly. Do not install new tools or spawn sub-agents. Produce results as files + one-line DONE notification. Confirm by replying: [@researcher🐝] ready"
tmux send-keys -t $W1 Enter
```

**Mode B/C (Discussion or Pipeline) — worker knows peers:**

```bash
tmux send-keys -t $W1 -l "[@Queen👑] I am Queen at $QUEEN. You are poet at $W1. Your peer is writer at $W2. To message peer: tmux send-keys -t $W2 -l '[@poet🐝] message' && tmux send-keys -t $W2 Enter. To message queen: tmux send-keys -t $QUEEN -l '[@poet🐝] message' && tmux send-keys -t $QUEEN Enter. Work directly. Do not install new tools or spawn sub-agents. Confirm by replying: [@poet🐝] ready"
tmux send-keys -t $W1 Enter
```

- **Verify:** `tmux capture-pane -p -t $QUEEN -S -10` contains `[@researcher🐝] ready`.
- **If fail (attempt 1):** Re-check worker pane — it may still be processing.
- **If fail (attempt 2):** Worker may have refused (see known-issues #9). Kill pane, respawn, retry.
- **If fail (attempt 3):** Switch to file-based communication — write tasks to `./swarm-tasks/`, read results from `./swarm-output/`.

**Message prefix:** Queen → `[@Queen👑]`, Worker → `[@{name}🐝]`.

---

## Operations

### Choose a Mode

| Mode | Trigger | Best for |
|------|---------|----------|
| **A: Independent** (default) | "research X and implement Y separately" | Decomposable, parallelizable tasks |
| **B: Discussion** | "discuss", "debate", "brainstorm", "critique" | Peer review, creative collaboration |
| **C: Pipeline** | "draft then review", "refine", sequential dependency | Multi-stage: draft → review → polish |

### Mode A: Independent Work

Default. Queen assigns separate tasks; workers execute independently. Write task specs to files; use `send-keys` only for short control messages:

```bash
mkdir -p ./swarm-tasks ./swarm-output
cat > ./swarm-tasks/task-researcher.md << 'EOF'
Research best practices for API rate limiting. Write summary to ./swarm-output/rate-limiting.md
EOF
tmux send-keys -t $W1 -l "[@Queen👑] Read task from ./swarm-tasks/task-researcher.md"
tmux send-keys -t $W1 Enter
```

Worker signals completion with a DONE marker: `[@researcher🐝] DONE: task-001 → ./swarm-output/rate-limiting.md`

### Mode B: Discussion / Mode C: Pipeline

See reference files for detailed patterns:
- **Discussion:** `references/mode-discussion.md` — turn-based exchange via shared discussion file
- **Pipeline:** `references/mode-pipeline.md` — chain of handoffs between workers

### Monitor (check once, report, return)

**Correct pattern** — check each worker once, report, give control back to user:

```bash
tmux capture-pane -p -t $W1 -S -30     # check W1
tmux capture-pane -p -t $W2 -S -30     # check W2
```

Then tell the user what you found and STOP. Example:
> "researcher is still working on the literature review. coder has finished — results in ./swarm-output/code.py. Want me to check again or read the results?"

For full output history (if `pipe-pane` enabled): `cat ./swarm-logs/researcher.log`

**WRONG pattern** (blocks conversation, user cannot interact):
```
❌  while true; capture-pane; sleep 10; done
❌  capture-pane → "not done yet" → sleep 5 → capture-pane → sleep 5 → ...
❌  Any use of sleep/wait between checks
```

### Spawn, Respawn & Kill Workers

```bash
tmux split-window -P -F "#{pane_id}"    # add worker, prints new pane ID
tmux select-layout main-vertical         # re-balance
tmux respawn-pane -k -t $W1              # restart stuck worker, preserves pane ID
tmux kill-pane -t $W1                    # remove specific worker
```

### Teardown

```bash
tmux send-keys -t $W1 -l "[@Queen👑] ALL DONE. Session complete." && tmux send-keys -t $W1 Enter
tmux send-keys -t $W2 -l "[@Queen👑] ALL DONE. Session complete." && tmux send-keys -t $W2 Enter
# Optional: tmux kill-pane -t $W1 && tmux kill-pane -t $W2
```

Default: leave panes open for inspection. Kill only if user wants cleanup.

---

## Advanced (read on demand)

| Scenario | Reference |
|----------|-----------|
| Complex project → shared task board, workers self-assign | `references/task-board.md` |
| Discussion / debate mode — turn-based peer exchange | `references/mode-discussion.md` |
| Pipeline mode — sequential handoff chain | `references/mode-pipeline.md` |
| Supported CLIs, install commands, strengths | `references/cli-compatibility.md` |
| Reliability: logging, pane recovery, stall detection, auto-layout | `references/reliability.md` |
| Troubleshooting: interleaving, identity confusion, worker rejection | `references/known-issues.md` |
