# Task Board — Self-Organizing Worker Coordination

For complex projects with more tasks than workers, use a shared task board file instead of assigning tasks one by one. Workers claim tasks themselves, update status, and move on to the next unclaimed task.

## Create the Board

```bash
mkdir -p ./swarm-tasks
cat > ./swarm-tasks/board.md << 'EOF'
# Swarm Task Board

## Tasks

- [ ] Research API authentication options (OAuth2 vs JWT vs API keys)
- [ ] Implement user login endpoint with tests
- [ ] Write database migration scripts
- [ ] Design monitoring dashboard layout
- [ ] Draft API documentation

## Rules
- Claim a task by replacing `[ ]` with `[@yourname🐝]`
- Mark done by replacing with `[x @yourname🐝]`
- Add output path next to completed task
- Do NOT claim more than one task at a time
- When your task is done, check the board for unclaimed tasks
EOF
```

## Notify Workers

```bash
tmux send-keys -t $W1 -l "[@Queen👑] Task board at ./swarm-tasks/board.md — read it, claim one task, work on it, update the board when done, then claim the next."
tmux send-keys -t $W1 Enter
tmux send-keys -t $W2 -l "[@Queen👑] Task board at ./swarm-tasks/board.md — read it, claim one task, work on it, update the board when done, then claim the next."
tmux send-keys -t $W2 Enter
```

## Board Evolution

As workers progress, the board updates itself:

```markdown
- [x @researcher🐝] Research API authentication options → ./swarm-output/auth-comparison.md
- [@coder🐝] Implement user login endpoint with tests
- [ ] Write database migration scripts
- [x @coder🐝] Design monitoring dashboard layout → ./swarm-output/dashboard.html
- [ ] Draft API documentation
```

## Queen's Role

Queen monitors progress by reading the file:

```bash
cat ./swarm-tasks/board.md
```

Queen intervenes only to:
- Add new tasks discovered during execution
- Resolve conflicts (two workers claimed the same task)
- Reassign stalled work (worker went silent on a claimed task)

The task board is the single source of truth — Queen does not need to micromanage.
