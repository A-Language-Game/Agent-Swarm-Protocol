# CLI Compatibility

Compatible with any CLI that accepts text input in a terminal pane.

## Supported CLIs

| CLI | Launch command | Installation | Best for |
|-----|----------------|--------------|----------|
| EvoSci | `EvoSci` | `pip install evoscientist` | Scientific discovery, experiment automation |
| Claude Code | `claude` or `claude --dangerously-skip-permissions` (recommended for swarm) | `curl -fsSL https://claude.ai/install.sh` or `brew install --cask claude-code` | General coding, refactoring, complex reasoning |
| Codex | `codex` | `npm install -g @openai/codex` or `brew install --cask codex` | Code generation, rapid prototyping |
| Gemini CLI | `gemini` | `npm install -g @google/gemini-cli` or `brew install gemini-cli` | Long-context analysis, multimodal tasks |
| DeepAgents | `deepagents` | `uv tool install deepagents-cli` | Multi-agent orchestration, LangChain workflows |
| smolagents | `smolagent` | `pip install "smolagents[toolkit]"` | Lightweight tool-use agents, HF model integration |
| Python REPL | `python3` | Pre-installed on most systems | Data processing, quick scripts, numeric computation |
| Shell | (default, no launch needed) | — | File operations, system commands, pipelines |

## Layout Options

| Layout | Commands | Description |
|--------|----------|-------------|
| Left-right | `select-layout main-vertical` + `resize-pane -t %0 -R 20` | Queen left, workers stacked right |
| Top-bottom | `select-layout main-horizontal` + `resize-pane -t %0 -D 10` | Queen top, workers stacked below |
