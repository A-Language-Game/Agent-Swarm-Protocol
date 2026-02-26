# Known Issues & Solutions

Collected from real multi-agent testing sessions (11+ test cases, 3-worker Claude Code swarm).

**Note:** Examples below use `%0`, `%1`, `%2` as illustrative pane IDs. In practice, always use your discovered pane IDs (`$QUEEN`, `$W1`, `$W2`) as established in SKILL.md Setup step 2.

---

## 1. Message Interleaving

**Problem:** Multiple workers `send-keys` to queen (`%0`) at the same time. Characters from different messages interleave, producing garbled output.

**Fix:** Workers keep reports to a single short line. Detailed results go in a file; the report message includes the file path.

```bash
# Worker writes details to file, sends one-line summary
tmux send-keys -t %0 -l "[@researcher🐝] Done. Results in ./swarm-output/auth-comparison.md"
tmux send-keys -t %0 Enter
```

---

## 2. Identity Confusion

**Problem:** Worker misidentifies itself or uses inconsistent names (e.g. `[Team Agent 2]` instead of `[@analyst🐝]`).

**Fix:** The init message must explicitly state the worker's name. Require the `[@{name}🐝]` prefix on every reply. Queen should reject messages without the prefix.

---

## 3. Report Format Inconsistency

**Problem:** Workers use varying formats: `[Team Agent 2 汇报]`, `[team_agent_2]`, `[@coder🐝]`. Queen cannot reliably parse.

**Fix:** Standardize on `[@{role-name}🐝]` as the only accepted prefix. Include this exact template in the handshake init message.

---

## 4. Path Specification Issues

**Problem:** Queen specifies internal file paths for workers (e.g. "write to `/workspace/output/report.md`"), causing nested or incorrect directories because the worker's CWD differs.

**Fix:** Distinguish two types of paths:
- **Shared comms directory** (`./swarm-tasks/`, `./swarm-output/`): Queen MAY specify these — they are the agreed convention for exchanging files.
- **Worker's internal paths**: Queen must NOT specify these. Let each worker manage its own working files and report output paths back.

---

## 5. Special Characters / Quote Hang

**Problem:** Sending messages without `-l` flag causes shell interpretation of quotes, backticks, or other metacharacters. The pane hangs waiting for a closing quote.

**Fix:** Always use `-l` (literal mode) and send `Enter` separately:

```bash
# Correct
tmux send-keys -t %1 -l "message with 'quotes' and $variables"
tmux send-keys -t %1 Enter

# Wrong — will hang or misinterpret
tmux send-keys -t %1 "message with 'quotes'" Enter
```

---

## 6. Over-Engineering

**Problem:** A simple task triggers the agent to install skills, spawn sub-agents within sub-agents, or create elaborate multi-layer pipelines — wasting time and tokens.

**Fix:** Add explicit constraints to the task description: "Work directly, do not install extra tools or create sub-agents." Reserve the swarm for tasks that genuinely benefit from parallelism.

---

## 7. Concurrent Report Collision

**Problem:** Two workers finish at the same time and both `send-keys` to queen. Even with `-l`, rapid concurrent sends can overlap.

**Fix:** Same as issue #1 — one-line summary + file path. If collision still occurs, queen uses `capture-pane` on its own pane to read the interleaved output, then reads each worker's result file for the actual data.

---

## 8. Message Delay Queuing (not loss)

**Problem:** Worker sends `tmux send-keys -t %0 -l "[@role🐝] message"` to queen, but queen's `capture-pane` shows nothing. Previously thought to be message loss — actually a **delay queue**.

**Mechanism:**
1. Worker executes `send-keys -t %0` → text enters queen's terminal input buffer
2. Queen is mid-turn (generating a response) → cannot process input → messages queue
3. Queen finishes turn, returns to input prompt → queued messages release in FIFO order
4. Queen's CLI misidentifies these as user input and processes them one by one

**Tested result:** 7/7 messages delivered with zero loss. All arrived after queen became idle.

**Fix:**
- **Do not rely on `send-keys` for real-time communication.** Messages arrive only when queen is idle.
- **Use `capture-pane` polling:** Queen proactively checks worker panes for status rather than waiting for `send-keys` reports.
- **File-based reporting for important results:** Worker writes to a file; queen reads via `read_file` or `cat`.

```bash
# Queen: proactively check worker status (don't wait for send-keys)
tmux capture-pane -p -t %1 -S -50

# Worker: write result to file + send notification
# ... (worker writes ./swarm-output/analysis.md) ...
tmux send-keys -t %0 -l "[@analyst🐝] Analysis complete. See ./swarm-output/analysis.md"
tmux send-keys -t %0 Enter
```

---

## 9. Worker Identity Rejection

**Problem:** Some worker agents refuse to participate in the swarm protocol entirely. They classify the handshake as "prompt injection" and decline to execute `tmux send-keys` back to queen, citing security concerns. This is non-deterministic — the same model may cooperate in one pane and refuse in another due to LLM sampling stochasticity.

**Root cause:** The worker's own system prompt establishes a strong self-identity and security policy that conflicts with the swarm role assignment. There is no way to signal "you are a swarm worker" at the system level — the role is assigned via chat, which the agent may distrust.

**Observed failure rate:** ~33% in testing (1 of 3 workers refused, unrecoverable after 3 attempts).

**Fix (short-term):** If a worker refuses handshake after 1–2 attempts, kill and replace. Due to sampling randomness, the replacement may cooperate.

```bash
# Kill refusing worker
tmux kill-pane -t %1

# Spawn replacement
tmux split-window
tmux select-layout main-vertical
# Re-launch CLI and retry handshake
```

**Fix (medium-term):** Use file-based communication instead of `send-keys` to avoid triggering security heuristics. Worker reads tasks from a file and writes results to a file — no `send-keys` back to queen's input stream.

```bash
# Queen writes task
echo "[@Queen👑] Research OAuth2 vs JWT for our auth system" > ./swarm-tasks/task-researcher.md

# Worker reads task, writes result
# ... worker writes ./swarm-output/result-researcher.md ...

# Queen reads result
cat ./swarm-output/result-researcher.md
```

**Fix (long-term):** A `--swarm-role` CLI flag that injects swarm identity into the system prompt at launch, bypassing the chat-level negotiation entirely.

**Key insight:** A swarm protocol that relies on persuading agents via chat will always have a non-zero failure rate. The most robust path is to establish swarm identity at the system/launch level, not the conversation level.

---

## 10. `/tmp/` Path Unreachable by Workers

**Problem:** Init or task files written to `/tmp/` (e.g. `cat > /tmp/init-researcher.md`) cannot be read by some CLIs. Claude Code restricts file access to the workspace directory — all three workers reported "The file doesn't exist in the workspace or anywhere in the project."

**Impact:** Handshake fails completely. Workers cannot read init instructions.

**Fix:** Always write shared files within the workspace directory:

```bash
# Wrong — workers can't access /tmp/
cat > /tmp/init-researcher.md << 'EOF'
...
EOF

# Correct — workspace-relative path
mkdir -p ./swarm-init
cat > ./swarm-init/init-researcher.md << 'EOF'
...
EOF
```

Use `./swarm-init/` for init files, `./swarm-tasks/` for task files, `./swarm-output/` for results.

---

## 11. Permission Prompt Blocking (Claude Code)

**Problem:** Claude Code prompts for permission on every shell command by default — including `tmux send-keys`, `mkdir -p`, `python script.py`. Each worker gets blocked 3–5 times per task. With 3 workers, queen must manually approve 10–15 permission prompts, destroying the "parallel unattended" design.

**Root cause:** Claude Code's security policy requires per-command user confirmation for all bash operations.

**Fix (recommended):** Launch workers with `claude --dangerously-skip-permissions`:

```bash
tmux send-keys -t %1 "claude --dangerously-skip-permissions" Enter
```

**Fix (alternative):** On the first permission prompt, select "Yes, and don't ask again for this type of command" to progressively reduce prompts.

**Note:** This only affects Claude Code. Other CLIs (Codex, Gemini, Python REPL, Shell) do not have per-command permission prompts.

---

## 12. Sandbox Isolation (EvoSci-specific)

**Problem:** EvoSci's `execute` tool runs in a sandboxed environment. When queen uses `execute("ls swarm-output/")` to verify worker output, it shows an empty directory — even though the files exist on the host filesystem. Workers (running outside the sandbox) can confirm files exist via `ls -la`.

**Root cause:** `execute` runs in an isolated sandbox; `read_file` / `glob` tools access the host filesystem directly.

**Fix:** When queen is EvoSci, use `read_file` or `glob` tools to verify worker output — not `execute` with `ls`:

```bash
# Wrong (sandbox, sees nothing)
execute("ls swarm-output/")

# Correct (host filesystem)
glob("swarm-output/*")
read_file("swarm-output/summary.md")
```

**Note:** This is EvoSci-specific. If queen is Claude Code, Codex, or another CLI without sandboxed execution, `ls` works normally.

---

## 13. Copy-Mode Swallows `send-keys`

**Problem:** If a worker's pane is in tmux copy-mode (user scrolled up, or mouse mode triggered it), `send-keys -l` silently sends text into the copy-mode buffer instead of the shell. The worker never receives the message. Queen sees no error.

**Root cause:** tmux's `send-keys` targets whatever mode the pane is in. In copy-mode, keystrokes navigate the scrollback buffer rather than reaching the shell.

**Fix:** Cancel copy-mode before sending:

```bash
tmux send-keys -t $W1 -X cancel 2>/dev/null   # exit copy-mode (no-op if not active)
tmux send-keys -t $W1 -l "[@Queen👑] message"
tmux send-keys -t $W1 Enter
```

---

## 14. Pane ID Drift After Respawn

**Problem:** After `respawn-pane -k -t $W1` or manual pane replacement, the pane ID stored in `$W1` may become stale in some edge cases (e.g., if the entire pane was killed and recreated instead of respawned). Subsequent `send-keys -t $W1` silently fail or target the wrong pane.

**Fix:** Use title markers for stable identity. Assign titles during setup and recover IDs by title:

```bash
# Setup: assign title
tmux select-pane -t $W1 -T "researcher"

# Recovery: find pane by title
W1=$(tmux list-panes -F "#{pane_id} #{pane_title}" | grep "researcher" | awk '{print $1}')
```

See `references/reliability.md` for full details.
