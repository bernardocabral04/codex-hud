# codex-hud

A two-line live HUD pane for OpenAI's [Codex CLI](https://github.com/openai/codex), rendered in a tmux split alongside the Codex TUI.

Mirrors the Claude Code statusline I use: model + reasoning effort, context window %, 5h/weekly rate limits with reset times, session id, cwd, and git branch with dirty + ahead/behind markers.

## How it works

Codex doesn't expose a `statusline.command` hook, so this wraps `codex` in a tmux session and spawns a sibling pane that tails the latest `~/.codex/sessions/**/rollout-*.jsonl` file. Rate limits and token usage come from `token_count` payloads embedded in the rollout — no API scraping needed.

## Install

```sh
ln -sf "$(pwd)/bin/cdx-hud" ~/.local/bin/cdx-hud
ln -sf "$(pwd)/bin/cdx-hud-render" ~/.local/bin/cdx-hud-render
```

Make sure `~/.local/bin` is in `PATH`. Replace your `cdx` alias:

```sh
alias cdx="cdx-hud"
```

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `CDX_HUD_CODEX_CMD` | `codex --dangerously-bypass-approvals-and-sandbox` | The actual codex invocation |
| `CDX_HUD_HEIGHT` | `2` | HUD pane height in lines |
| `CDX_HUD_REFRESH` | `2` | Render-loop interval, seconds |
| `CDX_HUD_ROLLOUT` | (auto) | Pin a specific rollout file (debug) |

## Color palette

- Model — Claude purple `#a78bfa`
- Context % / 5h / weekly — green → yellow ≥70% → bold bright red ≥90%
- Session id — dim grey
- Path — cyan
- Branch — yellow

## Caveats

- Codex's rollout JSONL schema is internal, not a public contract. Updates to Codex may break parsing.
- The HUD is a sibling tmux pane — it does **not** appear inside the Codex TUI itself.
