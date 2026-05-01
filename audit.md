# codex-hud — code audit

Two independent Opus reviews, consolidated. Findings are deduplicated and grouped by severity. Line numbers reference the state of the repo at the time of review (commit `12b1721`).

> **Status:** all critical, high, and medium findings have been addressed in subsequent commits — see `git log` for `(audit #N)` markers. The post-audit work also turned up and fixed a series of issues the audit didn't catch: `/resume`-induced session drift, terminal-zoom clipping (both width and height), bash 3.2 signal-vs-builtin interactions around the WINCH-redraw path, and a `bash -c "..."` regression that orphaned the HUD pane on Ctrl+C. The current code uses tmux `client-resized` / `window-resized` hooks to pin pane height across resizes, queries pane geometry from the tmux server (not `tput`), and clips by both width and height as defense-in-depth. Remaining items are low-severity correctness nuance (`#15`) and stylistic nits.

---

## Critical

### 1. `eval` of jq output is an injection sink — `bin/cdx-hud-render:330`

`extract_state` pipes jq's `to_entries[] | "\(.key|ascii_upcase)=\(.value)"` directly into `eval "$line"`. jq does no shell quoting; values flow from `session_meta.cwd`, `turn_context.model`, `turn_context.effort`, `rate_limits.plan_type`, etc. A path containing a space breaks the assignment loudly; a path or model name containing `;`, `$(...)`, backticks, or quotes executes as bash. Practical risk is low (these values originate from the user's own machine), but the latent bomb is real and the code will break the first time a user has a directory with a space.

**Fix.** Emit `KEY\tVALUE` from jq and read with:

```bash
while IFS=$'\t' read -r k v; do
  case "$k" in [A-Z_][A-Z0-9_]*) printf -v "$k" '%s' "$v" ;; esac
done
```

Both reviewers flagged this as the single most important fix.

---

## High

### 2. Quoting traverses two shell layers but `printf %q` only escapes one — `bin/cdx-hud:61`

In the outside-tmux branch, codex args are built via `printf '%q'`, then concatenated into a string passed to `tmux new-session "..."`. tmux runs the command via `sh -c`, which then runs the inner `bash -c "..."`. That's two shell parses but `%q` only escapes for one. Args containing `$`, backticks, `\`, or `!` get re-expanded or break the trap string.

**Fix.** Pass the codex argv via env or a small dispatch tempfile rather than splicing into nested shell strings.

### 3. Single quotes around `$RENDER_BIN` break if the path contains `'` — `bin/cdx-hud:48,67`

`'$RENDER_BIN'` is single-quoted inside a double-quoted tmux command. `$SCRIPT_DIR` is computed from `BASH_SOURCE[0]` and could contain a single quote (legal on macOS). Combined with finding #2, the failure mode is shell injection.

**Fix.** `printf %q` the path or pass via env (`CDX_HUD_RENDER_BIN=$RENDER_BIN exec "$CDX_HUD_RENDER_BIN"`).

### 4. Empty-array dereference under bash 3.2 — `bin/cdx-hud-render:175,201`

macOS ships bash 3.2. Under `set -u`, `"${files[@]}"` errors on an empty array. The script does `[ ${#files[@]} -eq 0 ]` later (line 192), but `for f in "${files[@]}"` (line 175) and `cat "${files[@]}"` (line 201) run beforehand on a possibly-empty array.

**Fix.** Move the empty-check above the loop, or use `${files[@]+"${files[@]}"}` everywhere.

### 5. `stat -f %m` is BSD-only — `bin/cdx-hud-render:80,97`

GNU stat (Linux) needs `-c %Y`. README implies the tool runs wherever tmux runs.

**Fix.** Either detect once and branch (`stat -f %m "$0" >/dev/null 2>&1 || STAT_FMT='-c %Y'`), or document the tool as macOS-only and exit early with a clear error otherwise.

### 6. README configuration table is out-of-sync — `README.md`

Listed defaults disagree with code:

| Var                  | README     | Actual code   |
|----------------------|------------|---------------|
| `CDX_HUD_CODEX_CMD`  | with bypass | `codex`       |
| `CDX_HUD_HEIGHT`     | 2           | 3             |
| `CDX_HUD_REFRESH`    | 2           | 5             |

`CDX_HUD_LOADING_REFRESH`, `CDX_HUD_YOLO`, `CDX_HUD_CODEX_PANE`, `CDX_HUD_SESSION_PREFIX`, and `CODEX_HOME` are undocumented. The first thing a contributor reads is wrong.

**Fix.** Regenerate the table from the code.

---

## Medium

### 7. Three `find` traversals per render tick — `bin/cdx-hud-render:60–62, 132, 305`

Every 5 s, the render loop runs `find` over the entire session tree three times (latest, recent×5, plus inside `detect_active_session`). For accounts with months of rollouts this dominates cost.

**Fix.** Single `find -newer` per tick, or cache the active rollout path and only rescan when `SESSIONS_DIR` mtime changes.

### 8. Whole-rollout reread every tick — `bin/cdx-hud-render:201`

`cat "${files[@]}" | jq -n '[inputs]'` reads each rollout in full (potentially several MB) every refresh. Acceptable today, slow on long sessions.

**Fix.** Cache by `(path, mtime)`; only re-jq when mtime changes. Or maintain byte offsets and parse the tail.

### 9. UUID extraction duplicated and partially dead — `bin/cdx-hud-render:101–107, 121–122`

Lines 101–103 derive UUID via `${var##*-}` then lines 106–107 immediately overwrite it with sed. The first block is dead; the regex is duplicated elsewhere.

**Fix.** Extract once into a `rollout_uuid()` helper.

### 10. Alt-screen restore trap missing `HUP` — `bin/cdx-hud-render:473`

Closing the terminal can leave the user staring at clobbered scrollback because the cleanup trap only catches `EXIT INT TERM`.

**Fix.** Add `HUP` (and `QUIT`) to match the wrapper's trap list.

### 11. TOML "parser" is regex over the whole file — `bin/cdx-hud-render:43–45`

`read_config_defaults` uses awk to find `model = "..."` and `model_reasoning_effort = "..."` *anywhere* in `config.toml`, including inside `[mcp_servers.*]` or `[profiles.foo]` tables. Mid-line `#` comments, multiline strings, and unicode also break it.

**Fix.** Either drop the fallback (turn_context will arrive within the first turn anyway) or shell out to `tomlq`/`yq -t` if installed.

### 12. Concurrent `cdx` invocations may bind to each other's session — *general*

When two `cdx` instances launch within the cutoff slack window (5 s), both may resolve the same "newest" rollout. The watchdog and start-epoch don't disambiguate by *which wrapper* spawned the codex.

**Fix.** Once `session_meta` is observed, pin that UUID for the lifetime of the render process; ignore later detection sweeps.

### 13. Resize stale-content bug — `bin/cdx-hud-render:308`

`\033[H\033[J` clears from cursor down, but inside the alt screen, terminal resize can leave wide content from a previous render on lines 2–3. `\033[K` only clears the current line.

**Fix.** Clear the whole pane with `\033[2J\033[H` on each render, or trap `SIGWINCH` and force a full redraw.

### 14. No debug mode — *general*

Errors from `jq`, `awk`, `find`, and `git` are routed to `/dev/null` throughout. When the HUD shows `?` for model or blank limits, the user has no recourse except reading source.

**Fix.** Add `CDX_HUD_DEBUG=1` redirecting stderr to `$CODEX_HOME/hud-debug.log` plus a one-line dump of the resolved rollout + last `extract_state` output.

---

## Low

### 15. `// ""` then `tonumber? // 0` for numeric fields — `bin/cdx-hud-render:240,249`

`(used_percent // "" | tostring)` then `(.rl_primary_pct | length) > 0` treats stale-zero data as "has data". Mixing string-empty and number-zero defaults invites edge cases.

**Fix.** Use `// 0` consistently and check `> 0` numerically.

### 16. `resets_at` assumed numeric — `bin/cdx-hud-render:266–271`

If a future Codex build emits ISO-8601, `tonumber? // 0` yields 0 and the "reset already passed" branch wipes live data.

**Fix.** Try `fromdateiso8601` if `tonumber?` fails.

### 17. YOLO sniff matches substrings — `bin/cdx-hud:36`

`case " $CODEX_CMD $* "` false-positives if any user arg literally contains `--dangerously-bypass-approvals-and-sandbox` (e.g. a path with that name). Mostly cosmetic since `turn_context` is the source of truth.

**Fix.** Iterate args explicitly: `for a in "$@"; do [[ "$a" == --dangerously-bypass-approvals-and-sandbox ]] && yolo_hint=…; done`.

### 18. `${cwd/#$HOME/~}` is glob-substitution — `bin/cdx-hud-render:416`

A `$HOME` containing `[` or `*` (legal on macOS) would mis-replace.

**Fix.** Strip with `case` or compare literally.

### 19. Watchdog probe scans all panes — `bin/cdx-hud-render:486`

`tmux list-panes -a | grep -qFx` scans every pane on the server. `tmux display-message -t "$pane" -p '#{pane_id}'` is O(1) and returns nonzero when the pane is gone.

**Fix.** Switch to `display-message -t`.

---

## Nit

- `META_TIMESTAMP` is reset in render but never emitted by jq (`bin/cdx-hud-render:309`).
- Redundant `local f mtime uuid` re-declaration (`bin/cdx-hud-render:94`).
- `human_tokens` lacks a default for empty input — defensive `${n:-0}` is cheap.
- `tput cols`/`tput lines` captured once at launch (`bin/cdx-hud:60`); brittle if invoked from a non-TTY parent.
- README doesn't mention: the watchdog behavior, alt-screen use, or that `*` after a percentage means "stale/last-known" rate limit data.

---

## Solid sections both reviewers called out

- **Cross-session rate-limit recovery** in `extract_state` — `currently_null`/`hit`/`stale` distinction is well thought through, with comments in the right places.
- **Dual EXIT/INT/TERM/HUP traps** in the wrapper plus the pane watchdog in the render — genuine belt-and-suspenders against the orphaned-pane footgun.
- **Loading-state handling** correctly zeros per-session fields when `meta_id` doesn't match `active_id`, avoiding the obvious "previous session's numbers shown as current" bug.

---

## Top three to fix first

1. **#1 — `eval` of jq output.** Latent injection + breaks on first path-with-space.
2. **#6 — README defaults are wrong.** First thing a contributor reads.
3. **#5 — `stat -f` portability.** High impact if Linux is ever a target; otherwise document the tool as macOS-only.
