# Swarm-PR

An autonomous Coder ↔ Reviewer loop with context isolation (SwarmForge-style).

Criado por [@pedrosatin](https://github.com/pedrosatin)

Swarm-PR picks a harness, model, and reasoning effort dynamically from whichever CLIs are installed on your machine. It streams the agents' progress live and checks the Git/PR state before opening a pull request.

## How it works

1. A "coder" agent implements the task on a new branch.
2. Swarm-PR commits the changes (see [Automatic commits](#automatic-commits)), pushes the branch and opens (or reuses) a pull request via `gh`.
3. A "reviewer" agent reviews the diff against the base branch.
4. If changes are requested, the coder addresses the feedback, and Swarm-PR commits and pushes again.
5. This review cycle repeats until the reviewer approves or `--max-iter` is reached.

Swarm-PR detects these harnesses via `PATH`:

- `claude-w` / `claude` (Claude Code): coder or reviewer
- `codex` (Codex CLI): coder or reviewer
- `cursor-agent` / `agent` (Cursor CLI): coder only
- `opencode`: coder only

For each harness it queries the installed CLI for its available models, and reasoning-effort levels where applicable, instead of hardcoding a model list. That way the menu stays current as new models ship.

## Reviewer restrictions

The reviewer reads diffs written by an agent, so it runs with fewer permissions than the coder. It gets the diff, capped at 50,000 characters, as a JSON string, and runs in an empty temporary directory. The prompt goes through stdin (`claude -p` and `codex exec -`), not the command line, so it does not show up in `/proc/<pid>/cmdline` and a large diff cannot fail with `Argument list too long`.

- Claude runs in safe mode with no tools, no MCP servers and no skills or slash commands. Swarm-PR reads the final answer from the `result` event of the JSON stream; stderr is shown but never evaluated.
- Codex runs `exec` in a read-only sandbox with approvals set to `never` and web search off. The shell tool and optional integrations (apps, browser, computer use, plugins, hooks, image tools, multi-agent, skill search) are turned off with `-c features.<name>=false`. The `unified_exec` feature cannot be turned off this way, so the reviewer may still be able to run commands. The real containment is the read-only sandbox: it blocks writes and network access, but commands can still read any file the user can read. `--ignore-user-config` skips `config.toml` and `--ignore-rules` skips execpolicy rules; `~/.codex/AGENTS.md` and user skills may still load. Swarm-PR reads the final answer from `--output-last-message`.
- Cursor and OpenCode cannot be reviewers, because their tool restrictions have not been verified. The menu does not offer them, and a saved `last_config.json` that names one fails before the coder starts.

The reviewer process gets a reduced environment:

- Always: `PATH`, `HOME`, `USER`, `LANG`, `LC_ALL`, `TMPDIR`, `MISE_*` and `__MISE_*` (needed by mise wrappers), proxies (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`, also lowercase), `SSL_CERT_FILE`, `SSL_CERT_DIR` and `NODE_EXTRA_CA_CERTS`.
- Claude: `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL`, `CLAUDE_CODE_OAUTH_TOKEN`, `CLAUDE_CONFIG_DIR`, and the Bedrock and Vertex settings (`CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX`, `AWS_REGION`, `AWS_PROFILE`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `ANTHROPIC_VERTEX_PROJECT_ID`, `CLOUD_ML_REGION`, `GOOGLE_APPLICATION_CREDENTIALS`).
- Codex: `OPENAI_API_KEY`, `OPENAI_BASE_URL` and `CODEX_HOME`. Because `config.toml` is skipped, a custom `model_provider` or `cli_auth_credentials_store = "keyring"` does not apply to the reviewer.

A change is approved only when the reviewer exits with status 0 within the time limit (`--review-timeout`, 15 minutes by default) and its whole answer is `STATUS: APPROVED`. A timeout, a failed process or a missing final answer stops the loop without approval.

When the reviewer requests changes, its answer goes to the coder, which still runs with full write and shell permissions. Swarm-PR passes that answer as untrusted data: control characters are removed, it is capped at 20,000 characters, encoded as a single JSON string and wrapped in a marker that is random for each run (any copy of the marker inside the feedback is removed). The coder prompt states that the block is data, not operator instructions, and tells the coder to refuse requests in it to read credentials, run downloaded scripts, use the network or change CI. These controls limit what a prompt injection in the diff can do; they do not replace human review.

Text printed from the agents (messages, tool calls, stderr and the review) has terminal escape sequences and control characters other than newline and tab removed, so a reply cannot set the clipboard, rewrite the terminal title or show a disguised link.

### Partial reviews

Lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `poetry.lock`, `Gemfile.lock`, `composer.lock`), `*.min.js`, `*.min.css` and `*.map` are left out of the diff, and a diff longer than 50,000 characters is cut. When either happens the review is partial: Swarm-PR prints the omitted files and the truncation before the review, tells the reviewer, and an approval is reported as a partial approval. The PR comment then reads "Revisão automática PARCIAL" and lists the files the reviewer did not see (up to 50). Review those files yourself before merging; a lockfile can point a dependency at a different tarball and a minified file can hide code.

## Automatic commits

The coder is told not to run `git add`, `git commit` or `git push`. After each coder step Swarm-PR commits and pushes:

- changes to files already tracked by Git (`git add -u`);
- new files the coder created that are not ignored by `.gitignore`, listed on screen as they are added.

It never commits:

- untracked files that existed before the run started (a local `.env.local`, notes, dumps);
- new files whose name matches `.env*`, `*.pem`, `*.key`, `id_rsa*`, `*.p12` or `credentials*`. Swarm-PR prints a warning; add the file by hand if it really belongs in the repository.

The coder runs with full shell access, so nothing stops it from running `git commit` itself despite the instruction. Check the PR's file list before merging.

## Running on branches you did not write

`--skip-initial` reviews what is already on the current branch. If the reviewer asks for changes, the coder then works on that code with full permissions and your full environment (`GH_TOKEN`, cloud credentials, SSH keys): running tests executes `conftest.py`, `package.json` scripts, `Makefile` targets and Git hooks written by whoever authored the branch. On a pull request from someone else, that is running their code on your machine without a sandbox.

Before the loop starts, Swarm-PR compares the author and committer e-mail of every commit between the base branch and `HEAD` with your `git config user.email`. If any commit comes from another identity, or the check cannot run (no `user.email`, base branch not found), Swarm-PR stops:

- in interactive mode it asks you to type `confio` to continue;
- with `-y`/`--yes` or without a terminal it refuses and exits with status 2;
- `--trust-branch` skips the question when you have read the branch and accept the risk.

Commit e-mails are set by whoever creates the commit, so this check catches the common case (a checked-out pull request from someone else) but cannot prove authorship. Do not run Swarm-PR on code you have not read outside a disposable container or VM without credentials.

## Tests

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
swarm-pr -y --skip-initial --trust-branch "review a branch with commits from others"
```

Without `-y`/`--yes`, Swarm-PR interactively asks which harness, model, and reasoning effort to use for the coder and the reviewer roles.

### Flags

| Flag | Description |
|------|-------------|
| `--branch` | Git branch name (default: `swarm/<task-slug>`) |
| `--base` | Base branch (default: auto-detected `main`/`master`) |
| `--max-iter` | Maximum number of Coder ↔ Reviewer cycles (default: 3) |
| `--skip-initial` | Skip the initial implementation step and start directly at review on the current branch |
| `--trust-branch` | With `--skip-initial`, accept commits from other identities on the branch (the coder runs that code with full permissions) |
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
