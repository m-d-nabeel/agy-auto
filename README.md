# agy-auto

Run `agy` interactively, exactly as normal, while a background watcher clears the recoverable failures for you — network drops, rate limits, and silent mid-turn freezes — by typing `continue` at the right moment. You keep full control of the terminal the whole time.

## Why

`agy` occasionally hits a transient error, a rate limit, or just stops producing output mid-task. Normally you'd notice, switch back to the terminal, and type `continue`. agy-auto watches the rendered pane and does that for you, so a long-running session doesn't stall for minutes waiting on you to notice.

It does **not** touch anything it can't confidently classify as a known recoverable state — real questions, plans, and decisions are always left for you.

## How it works

`agy-auto` starts `agy` inside a [tmux](https://github.com/tmux/tmux) session and attaches you to it — you type into it exactly like running `agy` directly. A background watcher polls the rendered pane (not raw output) and reacts to three cases:

| Class | Trigger | Behavior |
|---|---|---|
| **FAST** | A crash with an `Error ID`, or known transient error text | Retries as soon as the screen settles |
| **SLOW** | "verifying your account", rate limits, "try again shortly" | Holds a minimum wait before retrying (the server asked you to) |
| **STALL** | The animated spinner glyph is on screen but hasn't advanced in 60s | Sends `Escape` then `continue` — a genuine mid-turn freeze, not you thinking |

No TUI-specific calibration: "idle" is inferred from screen stability, not hardcoded prompt strings. Each class escalates its retry delay only when a retry demonstrably didn't help — a confirmed fix resets straight back to a fast retry, so a string of unrelated incidents doesn't inherit backoff from an earlier, already-resolved one. A circuit breaker (`AGY_AUTO_MAX_STREAK`, default 8) pauses auto-recovery entirely if retries keep failing, and hands control back to you.

## Requirements

- **bash ≥ 4.2** (for the `printf '%()T'` builtin). macOS ships 3.2 by default — `brew install bash` and invoke the script with that binary.
- **tmux** — developed against 3.7; older versions should degrade gracefully (unsupported terminal options are set with `|| true`) but aren't verified.
- **`agy`** on your `PATH`.
- Standard `grep`, `awk`, `tput`, `stty`, `cksum` — no GNU-only or PCRE features used; should work with BSD/macOS tool versions as well as GNU. Not yet verified on an actual Mac — please report back if you try it.

Works from any interactive shell (zsh, bash, fish, ...) since it's invoked as a standalone executable (`#!/usr/bin/env bash`) — your login shell never runs its code.

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/<you>/agy-auto/main/agy-auto -o ~/.local/bin/agy-auto
chmod +x ~/.local/bin/agy-auto
```

(or just clone and symlink/copy the `agy-auto` script anywhere on your `PATH`)

## Usage

```sh
agy-auto [agy args...]   # start (or reattach) — this is all you normally run
agy-auto --status        # session + watcher state
agy-auto --log           # tail the decision log
agy-auto --stop          # kill session and watcher
agy-auto --dry-run ...   # detect and log, but never inject (trial run)
```

If keyboard input ever fully stops responding with no error text and no spinner (rare — e.g. after navigating some interactive overlay), that's deliberately left undetected: there's no reliable signal to trigger on without risking a false positive on a session that's simply waiting on you. Closing the terminal won't help either — the tmux session survives and you'll reattach to the same wedged state. Recover with:

```sh
agy-auto --stop && agy-auto --continue [your usual flags]
```

`--continue` resumes from `agy`'s own persisted conversation state, not from tmux, so this is safe even though it kills the stuck session outright.

## Configuration

Optional, via `~/.config/agy-auto.conf` or env vars — nothing needs tuning to get started.

```sh
AGY_AUTO_PATTERNS='...'          # FAST-class error text (newline-separated, regex)
AGY_AUTO_SLOW_PATTERNS='...'     # SLOW-class error text
AGY_AUTO_STALL_SECS=60           # spinner-frozen threshold
AGY_AUTO_SLOW_WAIT=60            # min hold for SLOW-class messages
AGY_AUTO_COOLDOWN=15             # base backoff, doubles per unconfirmed retry
AGY_AUTO_MAX_BACKOFF=300         # backoff ceiling
AGY_AUTO_MAX_STREAK=8            # unconfirmed retries before pausing entirely
AGY_AUTO_CONTINUE_TEXT=continue  # text injected to resume
```

## License

MIT — see [LICENSE](LICENSE).
