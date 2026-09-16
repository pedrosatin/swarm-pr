# Swarm-PR

Loop autônomo **Coder ↔ Reviewer** com isolamento de contexto (estilo SwarmForge).

Escolhe harness/modelo/reasoning effort dinamicamente a partir das CLIs instaladas, mostra progresso em tempo real e valida o estado do Git/PR.

## Requisitos

- Python 3
- `git`
- Pelo menos um harness no `PATH`: `claude` / `claude-w`, `cursor-agent` / `agent`, `codex`, `opencode`
- `gh` (opcional, para abrir PR)

## Instalação

```bash
git clone git@github.com:pedrosatin/swarm-pr.git ~/Work/personal/swarm-pr
ln -sfn ~/Work/personal/swarm-pr/swarm-pr ~/.local/bin/swarm-pr
```

## Uso

```bash
swarm-pr "implementa X"
swarm-pr --max-iter 5 "implementa X"
swarm-pr -y --max-iter 3 "implementa X"
swarm-pr --skip-initial "só revisa o que já está na branch"
```

### Flags

| Flag | Descrição |
|------|-----------|
| `--branch` | Nome da branch (padrão: `swarm/<task-slug>`) |
| `--base` | Branch base (padrão: main/master detectado) |
| `--max-iter` | Máximo de ciclos Coder↔Reviewer (padrão: 3) |
| `--skip-initial` | Pula implementação inicial e começa no review |
| `-y`, `--yes` | Aceita defaults sem prompts interativos |

## Config

Última escolha de harness/modelo fica em:

```text
~/.config/swarm-pr/last_config.json
```

Esse arquivo é local ao usuário e **não** entra no repositório.
