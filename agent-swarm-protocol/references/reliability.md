# Reliability Patterns

Techniques for robust long-running swarms: output logging, pane identity recovery, stall detection, auto-layout, and defensive send-keys.

---

## 1. Output Logging via `pipe-pane`

`capture-pane` only shows the visible terminal buffer (~50 lines). For full output history, enable `pipe-pane` during setup:

```bash
mkdir -p ./swarm-logs
tmux pipe-pane -t $W1 "cat >> ./swarm-logs/researcher.log"
tmux pipe-pane -t $W2 "cat >> ./swarm-logs/coder.log"
```

**Reading logs:**

```bash
cat ./swarm-logs/researcher.log              # full history
tail -50 ./swarm-logs/researcher.log         # recent output
grep "DONE:" ./swarm-logs/researcher.log     # scan for completion markers
```

**Disable logging** when no longer needed:

```bash
tmux pipe-pane -t $W1                        # empty command = stop piping
```

**Tip:** Log files can grow large in long sessions. For multi-hour swarms, consider `tail -c 100000` (last 100KB) instead of `cat`.

---

## 2. Pane Title Markers & Recovery

Pane IDs (`%3`, `%7`) are ephemeral — they change on respawn. Assign stable title markers during setup:

```bash
tmux select-pane -t $W1 -T "researcher"
tmux select-pane -t $W2 -T "coder"
```

**Recovery after respawn** — re-discover pane IDs by title:

```bash
# Find pane ID by title
tmux list-panes -F "#{pane_id} #{pane_title}" | grep "researcher"
# Output: %12 researcher

# Re-bind variable
W1=$(tmux list-panes -F "#{pane_id} #{pane_title}" | grep "researcher" | awk '{print $1}')
```

**Naming convention:** `swarm-{role}` — matches the worker's `[@{role}🐝]` prefix for consistency.

---

## 3. Respawn vs Kill+Recreate

When a worker gets stuck, prefer `respawn-pane` over kill+split+rebind:

```bash
# Preferred: restart in place — pane ID preserved, title preserved
tmux respawn-pane -k -t $W1

# Old approach: kill + recreate — pane ID changes, must rebind
tmux kill-pane -t $W1
W1=$(tmux split-window -P -F "#{pane_id}")   # new ID!
tmux select-layout main-vertical               # must re-layout
```

After `respawn-pane`, you still need to re-run steps 3–5 (sync env → launch CLI → handshake), but the pane ID and title marker remain intact.

**When to kill instead of respawn:**
- Reducing worker count (scaling down)
- Pane is in an unrecoverable state (zombie process)

---

## 4. Worker Health Check & Stall Detection

### Quick liveness check

```bash
# Check if pane is alive (dead panes show '1')
tmux list-panes -F "#{pane_id} #{pane_dead}" | grep "$W1"
```

### Stall detection via output diff

If a worker appears idle for too long, compare two snapshots:

```bash
# Snapshot 1
tmux capture-pane -p -t $W1 -S -10 | md5sum > /tmp/w1-check1

# ... wait 30-60 seconds ...

# Snapshot 2
tmux capture-pane -p -t $W1 -S -10 | md5sum > /tmp/w1-check2

# Compare — identical hashes = likely stalled
diff /tmp/w1-check1 /tmp/w1-check2
```

With `pipe-pane` logging enabled, you can also check if the log file has grown:

```bash
wc -c ./swarm-logs/researcher.log    # check size now
# ... wait ...
wc -c ./swarm-logs/researcher.log    # check again — no growth = stalled
```

### Recovery for stalled workers

1. Try sending a nudge: `tmux send-keys -t $W1 -l "[@Queen👑] Status update?" && tmux send-keys -t $W1 Enter`
2. If no response after 30s: `tmux respawn-pane -k -t $W1` → re-run setup steps 3–5
3. If respawn fails: `tmux kill-pane -t $W1` → spawn fresh worker

---

## 5. Auto-Layout by Worker Count

The default `main-vertical` works well for 2 workers but degrades with more. Choose layout based on count:

| Workers | Layout command | Result |
|---------|----------------|--------|
| 1 | `tmux select-layout main-vertical` | Queen left, 1 worker right |
| 2 | `tmux select-layout main-vertical && tmux resize-pane -t $QUEEN -R 20` | Queen left (wider), 2 workers stacked right |
| 3 | `tmux select-layout tiled` | Balanced 2x2 grid (queen + 3 workers) |
| 4+ | `tmux select-layout tiled` | Even grid distribution |

**Rebalance after adding/removing workers:**

```bash
tmux select-layout tiled              # redistribute evenly
tmux resize-pane -t $QUEEN -R 10      # optional: give queen more width
```

---

## 6. Copy-Mode Guard

If a pane is in tmux copy-mode (e.g. user scrolled up), `send-keys` silently fails — text goes to the copy-mode buffer instead of the shell. Exit copy-mode before sending:

```bash
tmux send-keys -t $W1 -X cancel 2>/dev/null   # exit copy-mode (no-op if not in copy-mode)
tmux send-keys -t $W1 -l "[@Queen👑] ..."
tmux send-keys -t $W1 Enter
```

**When does this happen?**
- User manually scrolled a worker pane
- Some CLIs enter copy-mode during long output rendering
- Mouse mode is enabled in tmux config

---

## 7. Structured Task IDs

Use a consistent naming convention for task IDs in DONE markers:

**Format:** `{role}-{seq}` (e.g. `researcher-01`, `coder-02`)

```
[@researcher🐝] DONE: researcher-01 → ./swarm-output/literature-review.md
[@researcher🐝] DONE: researcher-02 → ./swarm-output/methodology.md
[@coder🐝] DONE: coder-01 → ./swarm-output/prototype.py
```

Queen tracks completion by parsing `DONE:` lines — each `{role}-{seq}` pair is unique, avoiding confusion when a worker handles multiple tasks.

For task-board mode, use the board's task numbering: `board-01`, `board-02`, etc.

---

## 8. Pane Border Labels (Visual UI)

tmux can display colored labels on each pane's border, making it easy to identify Queen vs Workers at a glance.

### Basic setup

```bash
tmux set-option pane-border-status top       # show labels on top border
tmux set-option pane-border-format " #{pane_index} #{pane_title} "
tmux set-option pane-active-border-style "fg=cyan,bold"
tmux set-option pane-border-style "fg=colour240"
```

### Color-coded labels (Queen vs Workers)

Use conditional formatting to give Queen and Workers different colors:

```bash
# Queen = magenta background, Workers = blue background
tmux set-option pane-border-format \
  "#{?#{==:#{pane_id},$QUEEN},#[fg=white,bg=magenta,bold] 👑 #{pane_index} #{pane_title} ,#[fg=white,bg=blue,bold] 🐝 #{pane_index} #{pane_title} }#[default]"
```

Result: Queen pane shows `👑 0 Queen` in magenta, workers show `🐝 1 researcher` in blue.

### Per-worker colors

For distinct colors per worker (like CCB's red/green/yellow per provider):

```bash
# Assign colors via pane-specific environment
tmux set-option pane-border-format \
  "#{?#{==:#{pane_title},Queen},#[fg=white,bg=magenta,bold] 👑 ,\
#{?#{==:#{pane_title},researcher},#[fg=white,bg=colour166,bold] 🐝 ,\
#{?#{==:#{pane_title},coder},#[fg=white,bg=green,bold] 🐝 ,\
#[fg=white,bg=blue,bold] 🐝 }}}#{pane_index} #{pane_title} #[default]"
```

Color palette suggestions:

| Role | Color | tmux code |
|------|-------|-----------|
| Queen | Magenta | `bg=magenta` |
| researcher | Orange | `bg=colour166` |
| coder | Green | `bg=green` |
| writer | Blue | `bg=blue` |
| reviewer | Yellow | `bg=yellow,fg=black` |
| analyst | Cyan | `bg=cyan,fg=black` |

### Reset to default (remove labels)

```bash
tmux set-option pane-border-status off
```

**Tip:** Pane border labels require tmux 2.6+. Use `tmux -V` to verify.

---

## Checklist — Robust Swarm Setup

Apply these during Setup step 2 for maximum reliability:

- [ ] Title markers: `select-pane -T "{role}"` for each pane
- [ ] Pane border labels: `pane-border-status top` + colored `pane-border-format`
- [ ] Pipe-pane logging: `pipe-pane "cat >> ./swarm-logs/{role}.log"` for each worker
- [ ] Layout: `main-vertical` for 1-2 workers, `tiled` for 3+
- [ ] Shared directories: `mkdir -p ./swarm-tasks ./swarm-output ./swarm-logs`
