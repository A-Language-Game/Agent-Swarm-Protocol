# Mode C: Pipeline

Workers form a sequential chain: Worker A produces output → Worker B takes it as input → Worker C refines. Queen sets up the chain and monitors each handoff.

**When to use:** User says "draft then review", "refine", "polish", or implies sequential dependency between agents (e.g. "write code, then have someone review it").

---

## Setup

1. Use peer-aware handshake (SKILL.md Setup step 5, Mode B/C variant) — each worker knows who comes next in the chain.

2. Tell each worker their position and handoff target:

```bash
# W1 (first in chain): produce output, notify W2
tmux send-keys -t $W1 -l "[@Queen👑] You are step 1 of 3. Write a first draft to ./swarm-output/draft.md. When done, notify reviewer: tmux send-keys -t $W2 -l '[@coder🐝] DONE: step-1 → ./swarm-output/draft.md — your turn' && tmux send-keys -t $W2 Enter"
tmux send-keys -t $W1 Enter

# W2 (middle): read input, produce refined output, notify W3
tmux send-keys -t $W2 -l "[@Queen👑] You are step 2 of 3. Wait for coder's notification. Then read ./swarm-output/draft.md, write review + fixes to ./swarm-output/reviewed.md. When done, notify refactorer: tmux send-keys -t $W3 -l '[@reviewer🐝] DONE: step-2 → ./swarm-output/reviewed.md — your turn' && tmux send-keys -t $W3 Enter"
tmux send-keys -t $W2 Enter

# W3 (last): read input, produce final output, notify Queen
tmux send-keys -t $W3 -l "[@Queen👑] You are step 3 of 3. Wait for reviewer's notification. Then read ./swarm-output/reviewed.md, write final version to ./swarm-output/final.md. When done: tmux send-keys -t $QUEEN -l '[@refactorer🐝] DONE: step-3 → ./swarm-output/final.md' && tmux send-keys -t $QUEEN Enter"
tmux send-keys -t $W3 Enter
```

---

## Handoff Pattern

Each worker follows the same pattern:
1. Wait for notification from the previous worker (or Queen for the first worker)
2. Read the input file
3. Produce output to a new file
4. Send DONE marker with file path to the next worker (or Queen for the last worker)

```
Queen → W1 (coder) → W2 (reviewer) → W3 (refactorer) → Queen
         draft.md      reviewed.md      final.md
```

---

## Queen Monitoring

Queen tracks progress by polling each worker's pane:

```bash
tmux capture-pane -p -t $W1 -S -30   # check if W1 finished
tmux capture-pane -p -t $W2 -S -30   # check if W2 started / finished
tmux capture-pane -p -t $W3 -S -30   # check if W3 started / finished
```

Look for `DONE: step-N` markers. If a worker stalls, Queen intervenes:

```bash
tmux send-keys -t $W2 -l "[@Queen👑] Check ./swarm-output/draft.md — coder has finished step 1. Please proceed with your review."
tmux send-keys -t $W2 Enter
```

---

## Example: Code → Review → Refactor

1. Queen spawns 3 workers: coder (W1), reviewer (W2), refactorer (W3)
2. Coder writes implementation to `./swarm-output/draft.py`
3. Coder notifies reviewer: `DONE: step-1 → ./swarm-output/draft.py`
4. Reviewer reads, writes review + fixes to `./swarm-output/reviewed.py`
5. Reviewer notifies refactorer: `DONE: step-2 → ./swarm-output/reviewed.py`
6. Refactorer reads, writes final version to `./swarm-output/final.py`
7. Refactorer notifies Queen: `DONE: step-3 → ./swarm-output/final.py`
8. Queen reads final output, presents to user

---

## Tips

- **Two workers is fine.** A pipeline doesn't need 3 stages — draft → review is a valid 2-worker pipeline.
- **Queen can be a stage.** For simple pipelines, Queen can act as the final reviewer instead of spawning a third worker.
- **Handoff notifications can fail.** If a worker's `send-keys` notification doesn't arrive (see known-issues #8), Queen should detect via `capture-pane` polling and manually trigger the next worker.
- **Don't overwrite input files.** Each stage writes to a new file so earlier output is preserved for debugging.
