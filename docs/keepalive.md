# Using the /keepalive Command

You pay to build a **prompt cache** every time SCORPIOX CODE talks to the model — and that cache is only good for a short while. If you step away for more than about five minutes, the cache expires, and your very next message has to pay to rebuild it from scratch. **Cache keep-alive** solves exactly that: it quietly pings the model on a timer so the cache never dies, and your next real message comes back cheap and fast.

You drive everything from two places:

- **The `/keepalive` slash command** — enable, pause, tune the timing, and pick the ping message.
- **The keep-alive popup** — a small draggable window that shows live state, the next-ping countdown, and your hit/miss stats.

Source of truth: the keep-alive timer and its popup, at commit `5fd054b`.

---

## What keep-alive actually does

When you pause mid-conversation — reading, thinking, switching tabs — the model-side cache for that conversation starts a countdown (a five-minute time-to-live). Once it hits zero, the next request has to re-send and re-cache the whole context.

Keep-alive arms a background timer. About **4:30** after the last response (a 30-second safety margin under the 5:00 TTL), it automatically sends a tiny real message — by default `keepalive, respond ok` — through the **normal agent path**, exactly like something you typed. Because it goes through the same provider settings, the cache fingerprint still matches, so the ping is a clean **cache read** instead of a rebuild. Every ping refreshes the clock. The moment you send a real message, the timer resets and the ping budget re-arms for the next idle gap.

The result: walk away for an hour, come back, and your first message is still served from cache. No rebuild, no cold-start latency.

> **It's off by default.** Keep-alive ships disabled. You opt in with `/keepalive on` (or by setting `CACHE_KEEPALIVE=1` — see [Configuration](#configuration-keys-in-scorpiox-envtxt) below).

> **It only pings when idle.** While the agent is mid-run it never fires. A ping is held and re-based until the loop is free, so it never interrupts an active turn.

---

## The five states

The popup title and the status-bar indicator both reflect one of five states:

| State | Meaning | Status bar |
|-------|---------|-----------|
| **disabled** | Feature turned off — no timer running. | *(blank)* |
| **idle** | Armed, but no live cache yet. Auto-activates the instant a response creates or reads a cache. | `KA-` |
| **active** | Monitoring the timer; will ping just before expiry. | `KA` or `KA:<n>` (n = pings so far) |
| **pinging** | A keep-alive message is being sent right now. | `KA*` |
| **paused** | Stopped on its own — the cache is already gone (misses hit the limit). | `KA\|\|` |

So the lifecycle in a normal session is: you enable it (`idle`) → your next real response creates a cache (`active`) → it pings to stay alive (`pinging` → back to `active`) → if a ping comes back with no cache (`paused`), it stops wasting pings until you re-arm it.

> **A "miss" is a good sign it stopped at the right time.** If a ping returns with nothing to read *and* nothing created, the cache is already gone — pinging further can't help, so keep-alive pauses. Your real messages always reset the budget, so the next time you work it arms fresh.

---

## The `/keepalive` slash command

`/keepalive` is your control surface. Type it with no argument to toggle the popup, or add a subcommand:

| Command | What it does |
|---------|--------------|
| `/keepalive` | Toggle the popup open/closed. |
| `/keepalive on` | Enable keep-alive. If it had **paused** on a miss, this resumes it. |
| `/keepalive off` | Pause pings (the thread keeps running, it just stops firing). |
| `/keepalive enable` | Start the keep-alive thread for the session. |
| `/keepalive disable` | Stop the keep-alive thread entirely. |
| `/keepalive show` | Force the popup open. |
| `/keepalive hide` | Hide the popup. |
| `/keepalive message <text>` | Set the ping message (max 256 chars). |
| `/keepalive interval <duration>` | Set how often it pings — the idle time before a ping. |
| `/keepalive <duration>` | Set the **total** time to keep the cache alive (it computes how many pings that takes). |

### Durations

Both `interval` and the bare duration take compact time specs — `s` = seconds, `m` = minutes, `h` = hours — and you can combine them:

```
/keepalive interval 270s      # ping every 4:30
/keepalive interval 5m        # ping every 5 minutes
/keepalive 45m                # keep the cache alive for 45 minutes total
/keepalive 2h                 # keep it alive for 2 hours total
/keepalive 1h30m              # 1 hour 30 minutes total
```

A **bare duration** sets the *total* window: SCORPIOX CODE works out the ping count from your interval (so at the default 4:30 interval, `45m` ≈ 10 pings). An **interval** value sets the *cadence* between pings.

### `/goal` — the shortcut for your ping message

`/goal` is an alias for `/keepalive message`:

```
/goal keepalive, respond ok   # set the ping message (and auto-enable if it's off)
/goal                        # show the current ping message
```

Setting a `/goal` is the quick way to arm keep-alive with a custom message — if keep-alive was disabled, `/goal <text>` switches it on for you automatically.

---

## The keep-alive popup

Type `/keepalive` (or `/keepalive show`) to bring up the popup. It's a **draggable** overlay — grab the title bar and move it out of the way — and it defaults to the top-right of the screen. Close it with the `[X]` button or `/keepalive hide`.

It shows a live snapshot:

```
 Keep-Alive (active)                    [X]
  Interval:    4m 30s
  Next ping:   2m 12s
  Pings:       3 / 12
  Hits: 3  Misses: 0
  Max tries:   1 (streak: 0)
  Message:     keepalive, respond ok
```

| Row | What it tells you |
|-----|-------------------|
| **Title** | Current state — `disabled`, `idle`, `active`, `pinging`, or `paused` — colored so you can tell at a glance. |
| **Interval** | The idle time before each ping (your `interval`). |
| **Next ping** | The live countdown to the next ping. Shows `sending...` while one is in flight, `--` when not counting down. |
| **Pings** | Pings sent this run, over the total budget (`<sent> / <max>`). |
| **Hits / Misses** | Pings that read the cache (`cache_read`) vs. pings that found nothing. Misses turn red. |
| **Max tries** | How many consecutive misses are allowed before it pauses, and the current miss streak. |
| **Message** | The exact text it sends as a ping. |

You can also glance at the **status bar** (bottom line, far right) for a one-token version of the state — `KA`, `KA:3`, `KA*`, `KA-`, or `KA\|\|` — without ever opening the popup.

---

## A typical session

Here's how it feels in practice:

1. **You're deep in a task** and get a good response that builds a big cache.
2. **You need 20 minutes** — a meeting, a review, a stretch. Run `/keepalive 30m` (or `/keepalive on` to use the defaults) so it pings every 4:30 and keeps the cache warm for half an hour.
3. **You watch the popup** (optional): it ticks `active → pinging → active`, the next-ping countdown resets, and your pings keep landing as hits.
4. **You come back and type.** The timer resets, the ping budget re-arms, and your message is served from the cache you spent the whole time keeping alive.

Want a custom nudge instead of the default text? `/goal continue, respond ok` does it in one line.

---

## Configuration keys in `scorpiox-env.txt`

Every key below is settable in any `scorpiox-env.txt` tier or profile (see the [configuration cascade and profiles](./scorpiox-env.md) page for where those live). Values set at runtime via `/keepalive` win for the current session; the file sets your starting point.

| Key | Default | Meaning |
|-----|---------|---------|
| `CACHE_KEEPALIVE` | `0` | Master switch. `0` = off, `1` = on. Keep-alive ships **off**; enable with `/keepalive on` or set this to `1`. |
| `CACHE_KEEPALIVE_TRIGGER` | `270` | Seconds of idle before a ping fires. `270` = 4:30, the default 30-second margin under the 5:00 cache TTL. |
| `CACHE_KEEPALIVE_MAX_TRIES` | `1` | Consecutive **misses** allowed before it pauses. `1` = pause on the first miss (the cache is already gone, so stop pinging). |
| `CACHE_KEEPALIVE_MAX_PINGS` | `12` | Total pings before it auto-stops (≈ an hour at the 4:30 interval). `0` = unlimited. |
| `CACHE_KEEPALIVE_MESSAGE` | *(empty)* | The ping text. Empty means the built-in default `keepalive, respond ok`. Override at runtime with `/goal <text>` or `/keepalive message <text>`. |

Example block:

```ini
# Keep the prompt cache warm while I step away.
CACHE_KEEPALIVE=1
CACHE_KEEPALIVE_TRIGGER=270
CACHE_KEEPALIVE_MAX_TRIES=1
CACHE_KEEPALIVE_MAX_PINGS=12
CACHE_KEEPALIVE_MESSAGE=keepalive, respond ok
```

---

## Gotchas

- **It's off until you turn it on.** Nothing pings unless `CACHE_KEEPALIVE=1` or you run `/keepalive on`/`enable`/a duration/`/goal`.

- **A bare duration sets the window, not the cadence.** `/keepalive 2h` keeps the cache alive for two hours using whatever `interval` you have; `/keepalive interval 5m` only changes how often it pings.

- **Pausing is self-protective, not an error.** `paused` means a ping came back with no cache to read — keep-alive stops pinging rather than burn tokens. Send any real message (or `/keepalive on`) to re-arm it.

- **`/clear` resets it.** Starting a fresh session resets keep-alive to `idle`; the ping count and budget start over.

- **The ping is a real message.** It goes through the normal agent path with the same model and settings, so it *can* cost tokens — that's the price of a live cache. The default short message keeps that cost minimal, and the budget (`MAX_PINGS` / duration) caps how long you keep paying.

- **Runtime changes don't touch the file.** `/keepalive` and `/goal` update the live session only. To make a change stick across restarts, set the matching key in `scorpiox-env.txt`.
