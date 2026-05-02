# codex-hud

A live HUD pane for OpenAI's [Codex CLI](https://github.com/openai/codex), rendered in a tmux split alongside the Codex TUI. Mirrors the Claude Code statusline: model + reasoning effort, context window %, 5h/weekly rate limits with reset times, session id, cwd, and git branch with dirty + ahead/behind markers. A third line lights up red when Codex is running in YOLO mode.

> **macOS only.** Uses BSD `stat -f %m`. Won't run on Linux without changes (audit issue #5).

## How it works

Codex doesn't expose a `statusline.command` hook, so this wraps `codex` in a tmux session and spawns a sibling pane that tails the latest `~/.codex/sessions/**/rollout-*.jsonl` file. Rate limits and token usage come from `token_count` payloads embedded in the rollout — no API scraping needed.

The render watchdogs the codex pane and self-exits if the pane disappears (belt-and-suspenders to the wrapper's `EXIT/INT/TERM/HUP` trap).

## Install

```sh
ln -sf "$(pwd)/bin/cdx-hud" ~/.local/bin/cdx-hud
ln -sf "$(pwd)/bin/cdx-hud-render" ~/.local/bin/cdx-hud-render
```

Make sure `~/.local/bin` is in `PATH`. Two suggested aliases:

```sh
alias cdx="cdx-hud"
alias cdxy="CDX_HUD_CODEX_CMD='codex --dangerously-bypass-approvals-and-sandbox' cdx-hud"
```

`cdx` runs Codex normally; `cdxy` enables YOLO mode and lights up the red banner.

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `CDX_HUD_CODEX_CMD` | `codex` | Resolved codex invocation. Set to include `--dangerously-bypass-approvals-and-sandbox` for YOLO. |
| `CDX_HUD_HEIGHT` | `3` | HUD pane height in lines (3 fits all three lines including the YOLO banner). |
| `CDX_HUD_REFRESH` | `5` | Render-loop interval in seconds. |
| `CDX_HUD_LOADING_REFRESH` | `1` | Tighter interval used while the "Loading Codex session…" placeholder is showing. |
| `CDX_HUD_SESSION_PREFIX` | `cdxhud` | Prefix for the tmux session name when launched outside an existing tmux. |
| `CDX_HUD_ROLLOUT` | (auto) | Pin a specific rollout file (debug). |
| `CDX_HUD_YOLO` | (auto) | Force the YOLO banner. The wrapper sets this when it sees `--dangerously-bypass-approvals-and-sandbox`. |
| `CDX_HUD_CODEX_PANE` | (auto) | tmux pane id of the codex pane. The wrapper sets this so the render can self-exit when codex's pane is gone. |
| `CDX_HUD_START_EPOCH` | (auto) | Unix epoch when the wrapper launched, used to ignore older rollouts. Set by the wrapper. |
| `CDX_HUD_LAUNCH_CWD` | (auto) | The wrapper's `$PWD` at spawn time. Used as the cwd fallback before codex flushes its first rollout JSONL — without this, a brand-new session would render `~/` until the user sent a turn. |
| `CODEX_HOME` | `$HOME/.codex` | Codex's data directory. Honored by the render. |
| `CDX_HUD_DEBUG` | `0` | Set to `1` to log detection + per-tick state to `$CODEX_HOME/cdx-hud-debug.log`. `tail -f` it while reproducing an issue. |
| `CDX_HUD_GIT_FETCH` | `1` | Set to `0` to disable the background `git fetch` daemon (the thing that keeps `↑n ↓n` accurate). |
| `CDX_HUD_GIT_FETCH_INTERVAL` | `5` | Seconds between background fetches. Increase on metered/cellular networks. |
| `CDX_HUD_GIT_FETCH_IDLE_EXIT` | `1800` | Seconds without a render heartbeat before the fetch daemon self-exits. |
| `CDX_HUD_GIT_FETCH_CACHE` | `${XDG_CACHE_HOME:-~/.cache}/codex-hud/git-fetch` | Cache dir for fetch-daemon PID files, heartbeats, and per-repo error logs. |
| `CDX_HUD_NETWORK` | `1` | Set to `0` to disable the background network-status daemon and the inline indicator on line 2. |
| `CDX_HUD_NETWORK_TARGET` | `1.1.1.1` | Ping target. Anything reachable from your machine works; pick a server you don't mind probing every 5 s. |
| `CDX_HUD_NETWORK_INTERVAL` | `5` | Seconds between probes. |
| `CDX_HUD_NETWORK_PING_TIMEOUT` | `2` | Per-probe timeout (seconds). |
| `CDX_HUD_NETWORK_SLOW_MS` | `300` | Latency threshold (ms) for the `slow` indicator. |
| `CDX_HUD_NETWORK_OFFLINE_THRESHOLD` | `3` | Consecutive failed probes before flipping to `offline`. |
| `CDX_HUD_NETWORK_SLOW_COUNT` | `2` | Consecutive slow probes before flipping to `slow`. |
| `CDX_HUD_NETWORK_RECONNECTED_DURATION` | `5` | Seconds the green `Reconnected` indicator lingers after coming back from `offline`. |
| `CDX_HUD_NETWORK_IDLE_EXIT` | `1800` | Seconds without a render heartbeat before the network daemon self-exits. |
| `CDX_HUD_NETWORK_CACHE` | `${XDG_CACHE_HOME:-~/.cache}/codex-hud/network` | Cache dir for the network daemon's PID file, heartbeat, and state file. |

## Display

Lines (top → bottom):

1. `model (effort) | <ctx>% (~tokens) | 5h:N% (Xh) 7d:N% (Yd) | session-id`
2. `~/path/to/cwd/ | (branch*) ↑n ↓n` (or `No Git` in dim grey when not in a repo)
3. `▶▶ yolo mode on` — only when YOLO mode is active

A trailing `*` after a rate-limit segment means the value is **stale** (last-known from a prior session — the active session has reported `null` for that window).

The `↑n ↓n` markers reflect the most recent fetched state of `@{upstream}`. A small per-repo daemon (`bin/git-fetch-daemon`) runs `git fetch` in the background every 5 s while the HUD is open so you don't need to fetch manually. It only fires when the repo has an upstream configured, never prompts for credentials (`GIT_TERMINAL_PROMPT=0`), self-exits after 30 min of HUD inactivity, and can be disabled with `CDX_HUD_GIT_FETCH=0`.

A sibling daemon (`bin/network-daemon`) pings a target every 5 s and writes the result to a small state file. Line 2 picks it up and appends an inline segment when the network is **offline** (red `✗ Not connected`), **slow** (yellow `⚠ Slow connection`, latency over 300 ms for two probes in a row), or **reconnected** (green `✔ Reconnected`, lingers for 5 s after coming back). When the connection is healthy the segment is silent so the line stays clean. Same atomic-link locking, same heartbeat-driven idle exit (30 min), same disable knob (`CDX_HUD_NETWORK=0`).

## Color palette

- Model — Claude purple `#a78bfa`
- Context % / 5h / weekly — green `#32` → yellow `#33` ≥70% → bold bright red `#1;91` ≥90%; rate limits use a steel-blue base `rgb(120,160,230)`
- Session id, separators, "No Git", stale `*` — dim grey `#90`
- Path — cyan `#36`
- Branch — yellow `#33`
- YOLO banner — bold bright red `#1;91`

## Caveats

- **macOS-only.** Uses BSD `stat -f`. Linux support not implemented.
- Codex's rollout JSONL schema, `state_5.sqlite` shape, and `shell_snapshots` filenames are private internals. Upstream changes can break parsing — see `audit.md` for the brittle assumptions.
- The HUD is a sibling tmux pane — it does **not** appear inside the Codex TUI itself.
- Output goes through tmux's alt-screen, so HUD content does not pollute terminal scrollback.