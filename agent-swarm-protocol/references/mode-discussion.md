# Mode B: Discussion / Debate

Workers exchange ideas via a shared discussion file, taking turns. Queen monitors and calls stop after N rounds or when consensus is reached.

**When to use:** User says "discuss", "debate", "brainstorm", "critique each other", "peer review", or wants agents to build on each other's ideas.

---

## Setup

1. Create discussion file with topic, rules, and round limit:

```bash
mkdir -p ./swarm-discussion
cat > ./swarm-discussion/topic.md << 'EOF'
# Discussion: [Topic]

**Participants:** poet (W1), writer (W2)
**Rules:** Each participant reads the full file, appends their response under a new heading, then notifies the next participant. Use [@name🐝] prefix on your heading.
**Max rounds:** 3 per participant (6 total turns). Queen will call stop.

---

## Round 1
EOF
```

2. Use peer-aware handshake (SKILL.md Setup step 5, Mode B/C variant) — each worker knows the other's pane ID.

---

## Turn-Taking Flow

**Queen tells W1 to go first:**

```bash
tmux send-keys -t $W1 -l "[@Queen👑] Read ./swarm-discussion/topic.md. Add your perspective under a new heading. When done, notify writer by running: tmux send-keys -t $W2 -l '[@poet🐝] My turn is done. Read ./swarm-discussion/topic.md and add your response' && tmux send-keys -t $W2 Enter"
tmux send-keys -t $W1 Enter
```

**W1 appends to file, notifies W2.** W2 reads, appends, notifies W1. Cycle continues.

**Each turn, the worker:**
1. Reads the full discussion file
2. Appends their response with `[@name🐝]` heading
3. Sends a one-line notification to the next participant

---

## Queen Monitoring

Queen periodically checks discussion progress:

```bash
# Read the discussion file directly
cat ./swarm-discussion/topic.md

# Or check worker panes for activity
tmux capture-pane -p -t $W1 -S -20
tmux capture-pane -p -t $W2 -S -20
```

**Queen calls stop** when:
- Max rounds reached
- Workers reach consensus (repeated agreement in responses)
- Quality is sufficient for the user's goal

```bash
tmux send-keys -t $W1 -l "[@Queen👑] Discussion complete. Stop adding to the file." && tmux send-keys -t $W1 Enter
tmux send-keys -t $W2 -l "[@Queen👑] Discussion complete. Stop adding to the file." && tmux send-keys -t $W2 Enter
```

Queen then reads the final file and synthesizes results for the user.

---

## Example: Poet + Writer Collaborating on a Poem

1. Queen creates `./swarm-discussion/poem.md` with theme: "autumn in the city"
2. Queen tells poet (W1): "Write an initial draft stanza, notify writer when done"
3. Poet appends draft, notifies writer (W2)
4. Writer reads, appends critique + revised version, notifies poet
5. Poet reads feedback, appends refined version, notifies writer
6. Queen reads final file after 3 rounds, presents best version to user

---

## Tips

- **Keep rounds bounded.** Open-ended discussion wastes tokens. Default to 3 rounds per participant.
- **One file per topic.** Don't split discussion across multiple files — workers need full context each turn.
- **Queen synthesizes.** Workers debate; Queen makes the final decision and presents a unified result.
- **File conflicts are rare** because turns are sequential — only one worker writes at a time.
