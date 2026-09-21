# Swarm-PR

An autonomous Coder ↔ Reviewer loop with context isolation (SwarmForge-style).

Swarm-PR picks a harness, model, and reasoning effort dynamically from whichever CLIs are installed on your machine. It streams the agents' progress live and checks the Git/PR state before opening a pull request.

## How it works

1. A "coder" agent implements the task on a new branch and pushes its commits.
2. Swarm-PR opens (or reuses) a pull request via `gh`.
3. A "reviewer" agent reviews the diff against the base branch.
4. If changes are requested, the coder addresses the feedback and pushes again.
5. This review cycle repeats until the reviewer approves or `--max-iter` is reached.

Swarm-PR supports multiple harnesses as coder or reviewer, detected dynamically via `PATH`:

- `claude-w` / `claude` (Claude Code)
- `cursor-agent` / `agent` (Cursor CLI)
- `codex` (Codex CLI)
- `opencode`

For each harness it queries the installed CLI for its available models, and reasoning-effort levels where applicable, instead of hardcoding a model list. That way the menu stays current as new models ship.

## Requirements

- Python 3
- `git`
- At least one supported harness on `PATH`: `claude` / `claude-w`, `cursor-agent` / `agent`, `codex`, `opencode`
- `gh` (optional, needed to open pull requests)

## Installation

```bash
git clone git@github.com:pedrosatin/swarm-pr.git ~/Work/personal/swarm-pr
ln -sfn ~/Work/personal/swarm-pr/swarm-pr ~/.local/bin/swarm-pr
```

## Usage

```bash
swarm-pr "implement X"
swarm-pr --max-iter 5 "implement X"
swarm-pr -y --max-iter 3 "implement X"
swarm-pr --skip-initial "just review what's already on the branch"
```

Without `-y`/`--yes`, Swarm-PR interactively asks which harness, model, and reasoning effort to use for the coder and the reviewer roles.

### Flags

| Flag | Description |
|------|-------------|
| `--branch` | Git branch name (default: `swarm/<task-slug>`) |
| `--base` | Base branch (default: auto-detected `main`/`master`) |
| `--max-iter` | Maximum number of Coder ↔ Reviewer cycles (default: 3) |
| `--skip-initial` | Skip the initial implementation step and start directly at review |
| `-y`, `--yes` | Accept defaults without interactive prompts |

## Config

The last harness/model choice is stored at:

```text
~/.config/swarm-pr/last_config.json
```

This file is local to the user and is not part of the repository.

## License

MIT, see [LICENSE](LICENSE).
