# Swarm-PR

An autonomous Coder ↔ Reviewer loop with context isolation (SwarmForge-style).

Criado por [@pedrosatin](https://github.com/pedrosatin)

Swarm-PR picks a harness, model, and reasoning effort dynamically from whichever CLIs are installed on your machine. It streams the agents' progress live and checks the Git/PR state before opening a pull request.

## How it works

1. A "coder" agent implements the task on a new branch and pushes its commits.
2. Swarm-PR opens (or reuses) a pull request via `gh`.
3. A "reviewer" agent reviews the diff against the base branch.
4. If changes are requested, the coder addresses the feedback and pushes again.
5. This review cycle repeats until the reviewer approves or `--max-iter` is reached.

Swarm-PR detects these harnesses via `PATH`:

- `claude-w` / `claude` (Claude Code): coder or reviewer
- `codex` (Codex CLI): coder or reviewer
- `cursor-agent` / `agent` (Cursor CLI): coder only
- `opencode`: coder only

For each harness it queries the installed CLI for its available models, and reasoning-effort levels where applicable, instead of hardcoding a model list. That way the menu stays current as new models ship.

## Reviewer restrictions

The reviewer reads diffs written by an agent, so it runs with fewer permissions than the coder. It gets the diff, capped at 50,000 characters, as a JSON string, and runs in an empty temporary directory.

- Claude runs in safe mode with no tools, no MCP servers and no skills or slash commands. Swarm-PR reads the final answer from the `result` event of the JSON stream; stderr is shown but never evaluated.
- Codex runs `exec` in a read-only sandbox with approvals set to `never` and web search off. The shell tool and optional integrations (apps, browser, computer use, plugins, hooks, image tools, multi-agent, skill search) are turned off with `-c features.<name>=false`. The `unified_exec` feature cannot be turned off this way, so the reviewer may still be able to run commands. The real containment is the read-only sandbox: it blocks writes and network access, but commands can still read any file the user can read. `--ignore-user-config` skips `config.toml` and `--ignore-rules` skips execpolicy rules; `~/.codex/AGENTS.md` and user skills may still load. Swarm-PR reads the final answer from `--output-last-message`.
- Cursor and OpenCode cannot be reviewers, because their tool restrictions have not been verified. The menu does not offer them, and a saved `last_config.json` that names one fails before the coder starts.

The reviewer process gets a reduced environment:

- Always: `PATH`, `HOME`, `USER`, `LANG`, `LC_ALL`, `TMPDIR`, `MISE_*` and `__MISE_*` (needed by mise wrappers), proxies (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`, also lowercase), `SSL_CERT_FILE`, `SSL_CERT_DIR` and `NODE_EXTRA_CA_CERTS`.
- Claude: `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL`, `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CONFIG_DIR`, and the Bedrock and Vertex settings (`CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `AWS_REGION`, `AWS_PROFILE`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `ANTHROPIC_VERTEX_PROJECT_ID`, `CLOUD_ML_REGION`, `GOOGLE_APPLICATION_CREDENTIALS`).
- Codex: `OPENAI_API_KEY`, `OPENAI_BASE_URL` and `CODEX_HOME`. Because `config.toml` is skipped, a custom `model_provider` or `cli_auth_credentials_store = "keyring"` does not apply to the reviewer.

A change is approved only when the reviewer exits with status 0 within the time limit (`--review-timeout`, 15 minutes by default) and its whole answer is `STATUS: APPROVED`. A timeout, a failed process or a missing final answer stops the loop without approval.

When the reviewer requests changes, its answer goes to the coder, which still runs with full write and shell permissions. These controls limit what a prompt injection in the diff can do; they do not replace human review.

Offline checks: `python3 -m unittest discover -p 'test_*.py' -v`.

## Requirements

- Python 3
- `git`
- At least one supported harness on `PATH`: `claude` / `claude-w`, `cursor-agent` / `agent`, `codex`, `opencode`. The reviewer needs `claude` / `claude-w` or `codex`.
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
| `--review-timeout` | Maximum minutes per review before the reviewer is killed (default: 15) |
| `-y`, `--yes` | Accept defaults without interactive prompts |

## Config

The last harness/model choice is stored at:

```text
~/.config/swarm-pr/last_config.json
```

This file is local to the user and is not part of the repository.

## Contributing

Report bugs and suggest contributions at https://github.com/pedrosatin/swarm-pr/issues.

## License

MIT, see [LICENSE](LICENSE).
