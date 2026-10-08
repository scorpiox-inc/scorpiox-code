# Using the /keepalive Command

Long sessions live and die on the prompt cache. Every turn, SCORPIOX CODE builds one — system instructions, tools, and the entire conversation — and the provider keeps it hot for about **five minutes**. Step away for a coffee and your next message pays to re-send and re-process all of it: slower first token, higher cost, nothing gained for it. **Cache keep-alive** fixes exactly that. It watches the clock while you are away and sends one tiny message just before the cache expires, so the cache is still warm the moment you come back.

You drive the whole feature from one built-in slash command — **`/keepalive`** — plus a small draggable popup that shows the live state, the countdown to the next ping, and your hit/miss statistics. A **`/goal`** shortcut covers the most common case in a single line.

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** while you are idle, SCORPIOX CODE auto-sends a short real message right before the provider's cache expires. Because it travels the same path as your own messages, it lands as a cache read instead of a rebuild — which both refreshes the cache and resets the countdown.

---

## Why the cache needs keeping alive

Hosted providers do not keep your context on a hot shelf forever. They cache the prompt for a short window — on the order of five minutes — because serving a cached prefix is far cheaper for them than re-processing it. While that window is open, your turns are fast and inexpensive. The moment it closes, the next request pays full price again.

| You come back after | What your next message pays |
|---------------------|-----------------------------|
| Under ~4 minutes | A cache read. Fast first token, low cost. |
| Over ~5 minutes | A full rebuild of the context, then a fresh cache write on top. |

For a ten-second gap that is irrelevant. For a long session where you step away — to review something, take a call, or run a build in parallel — it is the difference between coming back to a warm session and coming back to a cold start.

### What a keep-alive ping actually is

It is not a background HTTP call and not a webhook. It is **a real user message sent through the same agent path you use for everything else** — same provider, same model, same tools, same settings. The only difference is that a timer typed it for you. Two consequences fall out of that:

- **The cache fingerprint matches perfectly.** A ping is indistinguishable from a message you typed, so the provider serves it from the cache — which is precisely the mechanism that refreshes the expiry clock.
- **It costs a little.** Each ping is a small model call. That is the trade: a few cheap pings to avoid one expensive cold rebuild. You control the cadence and the total through the command and the configuration keys below.

> **It is off by default.** Keep-alive ships disabled. You opt in with `/keepalive on`, a duration, or `/goal` — or set `CACHE_KEEPALIVE=1` to have it armed at startup.
>
> **It only pings when idle.** While the agent is mid-run it never fires. A ping that comes due during a turn is held until the loop is free, so it never interrupts your work.

---

## The five states

The popup title and the status-bar marker both reflect one of five states:

| State | Meaning | Popup title | Status bar |
|-------|---------|-------------|------------|
| **disabled** | Feature off — no timer running. | `Keep-Alive (disabled)` | *(blank)* |
| **idle** | Armed, but no live cache yet. Auto-activates as soon as a response creates or reads a cache. | `Keep-Alive (idle)` | `KA-` |
| **active** | Watching the clock; will ping just before expiry. | `Keep-Alive (active)` | `KA` or `KA:n` |
| **pinging** | A keep-alive message is in flight right now. | `Keep-Alive (pinging)` | `KA*` |
| **paused** | Stopped — a ping found no cache to read, the ping budget ran out, or you paused it. | `Keep-Alive (paused)` | `KA\|\|` |

So a normal session reads: enable it (`idle`) → your next real response creates a cache (`active`) → it pings to stay alive (`pinging` → back to `active`) → if a ping comes back with nothing cached (`paused`), it stops spending until you re-arm it.

> **A miss means it stopped at the right time.** If a ping returns with nothing to read *and* nothing written, the cache is already gone — pinging further cannot help. Keep-alive pauses rather than burn calls on a dead cache. Any real message re-arms it with a fresh budget.

---

## The `/keepalive` command

`/keepalive` is your control surface. Type it with no argument to toggle the popup, or add a subcommand:

| Command | What it does |
|---------|--------------|
| `/keepalive` | Toggle the popup open or closed. |
| `/keepalive on` | Turn keep-alive on. If it was **disabled** this enables it; if it had **paused**, this resumes it and restarts the countdown from now. |
| `/keepalive off` | Pause pings. The timer stays loaded, it just stops firing. If it was already off, the transcript says so. |
| `/keepalive enable` | Full enable: start the timer and reset the counters. |
| `/keepalive disable` | Stop the timer entirely for the rest of the session. |
| `/keepalive show` | Show the popup. |
| `/keepalive hide` | Hide the popup. |
| `/keepalive message <text>` | Set the text sent as each ping (up to 256 characters). |
| `/keepalive interval <duration>` | Set how often it pings — e.g. `5m`, `270s`. |
| `/keepalive <duration>` | Keep the cache alive for a **total** window — e.g. `45m`, `1h30m`, `2h`. |

Durations accept hours, minutes, seconds, or a bare number of seconds, and you can combine them: `270`, `270s`, `5m`, `45m`, `2h`, `1h30m`. As you type, the input line offers the subcommands plus two example durations (`45m`, `2h`), so you never have to memorize the list.

### Arm it for a window

You are deep in a refactor and need to step away for 45 minutes, but you want the session ready the instant you are back:

```
/keepalive 45m
```

The transcript confirms the plan and the popup opens:

```
Keep-alive for 45m (10 pings every 270s).
```

The ping count is computed for you — total window divided by the current interval, rounded up. If no cache exists yet, the message adds that it is armed and will start once your next message creates one. This form also turns keep-alive on if it was off, and it **overrides the ping budget from the config file for this session**: the cap is recalculated from the window you asked for, not from `CACHE_KEEPALIVE_MAX_PINGS`.

### Tune the cadence

```
/keepalive interval 5m
```

This changes how often pings fire and confirms with `Keep-alive interval set to 300s.` The countdown re-anchors to your last response, so changing the interval mid-window does not fire an instant ping. A bare duration like `/keepalive 45m` sets *how long* to keep the cache alive; `/keepalive interval 5m` sets *how often* it pings. They compose: set the interval once, then arm windows with plain durations.

### `/goal` — the one-liner

`/goal` is an alias for setting the ping message, with a convenience twist: if keep-alive is off, it **turns itself on** and opens the popup.

```
/goal keep the API tests green
```

That sets the ping text and enables keep-alive in one step. Running `/goal` with no argument prints the current goal instead of changing it. Use `/goal` when you want the ping to double as a reminder of what to focus on; use `/keepalive message <text>` when you only want to change the text.

---

## The keep-alive popup

Type `/keepalive` (or `/keepalive show`) to open the popup: a compact draggable card that defaults to the top-right of the terminal.

```
 Keep-Alive (active)                    [X]
 Interval:    4m 30s
 Next ping:   2m 12s
 Pings:       3 / 10
 Hits: 3      Misses: 0
 Max tries:   1 (streak: 0)
 Message:     keepalive, respond ok
```

| Field | What it tells you |
|-------|-------------------|
| **Title** | Current state — `disabled`, `idle`, `active`, `pinging`, or `paused` — color-coded so you can read it at a glance. |
| **Interval** | The idle time before each ping. |
| **Next ping** | A live countdown to the next ping. Shows `sending...` while one is in flight, `--` when nothing is counting down. |
| **Pings** | Pings sent against the budget (`sent / max`), or a bare count when the budget is unlimited. |
| **Hits / Misses** | Pings that read from the cache versus pings that found nothing. Misses turn red. |
| **Max tries** | The consecutive-miss threshold, plus the current miss streak. |
| **Message** | The exact text sent as each ping. |

**Drag it** by the title bar to move it out of the way; **close it** with the `[X]` button, `/keepalive hide`, or by running `/keepalive` again. Closing the popup changes nothing about the timer — keep-alive keeps running either way.

A healthy session sits in `active` with a steadily counting-down **Next ping** and a growing **Hits** count. If **Misses** start climbing, the cache is not surviving between pings — the provider window is shorter than your interval, or the session was interrupted — and the feature pauses itself after the configured number of consecutive misses rather than keep spending.

---

## The status bar

You do not need the popup to keep an eye on things. Once the session has had at least one response, the bottom status bar shows two live fields, flush right:

- **Cache timer** — `T mm:ss`, the countdown to cache expiry. Green while plenty of window remains, yellow as it narrows, red inside the last minute, and `T --:--` once it has lapsed.
- **Keep-alive marker** — a short fixed-width field:

| Marker | Meaning |
|--------|---------|
| `KA` | Active, no pings sent yet. |
| `KA:3` | Active, 3 pings sent so far. |
| `KA*` | A ping is in flight. |
| `KA-` | Armed and idle, waiting for the first cache. |
| `KA\|\|` | Paused. |
| *(blank)* | Disabled. |

The countdown window follows the keep-alive interval (270 s by default), and the green-to-yellow boundary sits at 60% of that window — so with the default interval the timer turns yellow at 2:42, not 3:00. Every real response, and every successful ping, resets it. The same numbers are mirrored in the session's `stats.json` under `cache_timer` and `keepalive` — see [Token Usage Observability](usage-observability.md).

---

## Configuration keys

Keep-alive is configured through `scorpiox-env.txt` — any cascade tier, a named profile, or an OS environment variable. See [Configuration and Profiles](scorpiox-env.md) for the cascade and precedence rules. Values set at runtime with `/keepalive` or `/goal` win for the current session; the file sets the starting point.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `CACHE_KEEPALIVE` | bool | `0` | Master switch. `0` = keep-alive starts disabled; `1` = armed at startup. |
| `CACHE_KEEPALIVE_TRIGGER` | int (seconds) | `270` | Seconds of idle before a ping fires. 4:30, a 30-second margin inside a typical 5:00 cache TTL. |
| `CACHE_KEEPALIVE_MAX_TRIES` | int | `1` | Consecutive cache-miss pings tolerated before the feature pauses itself. |
| `CACHE_KEEPALIVE_MAX_PINGS` | int | `2` | Total pings before it stops on its own. `0` = unlimited. |
| `CACHE_KEEPALIVE_MESSAGE` | text | *(empty)* | The ping text. Empty means the built-in default `keepalive, respond ok`. |

Example block:

```ini
# Keep the prompt cache warm while I step away.
CACHE_KEEPALIVE=1
CACHE_KEEPALIVE_TRIGGER=270
CACHE_KEEPALIVE_MAX_TRIES=1
CACHE_KEEPALIVE_MAX_PINGS=2
CACHE_KEEPALIVE_MESSAGE=keepalive, respond ok
```

In the config editor (`/config` in a session, or `scorpiox-config` from a shell) these keys live in the **Cache** section. The four tuning keys appear only once `CACHE_KEEPALIVE` is switched on, which keeps the editor list short when you are not using the feature.

---

## How the loop behaves

The behavior is deliberately conservative — it spends a little to save a lot, and it stands down the moment that stops being true:

1. **Armed, waiting for a cache.** When enabled, keep-alive starts idle. It has nothing to protect until a real response has written a cache, so it does nothing on a brand-new session.
2. **Active once a cache exists.** The first response that reads or writes cache tokens arms the countdown. From then on it watches time since the last response.
3. **Ping just before expiry.** As the idle time reaches the interval, it queues the ping. The main loop sends it through the normal agent path when the agent is idle — never mid-turn — and the state flips to `pinging` until the response lands.
4. **Score the ping.** A ping that reads from cache is a hit and resets the clock. A ping that *recreates* the cache (read zero, write non-zero) also counts as a hit — a cache now exists and the next ping will read it. Only a ping with neither counts as a miss.
5. **Stand down when it stops working.** After the configured consecutive misses, or once the ping budget is spent, it pauses instead of continuing to spend.

Two things hand it a fresh start: **any real message you send** resets the ping count and the miss streak, and **`/clear`** re-arms the feature from idle, since the conversation it was protecting is gone. The default budget of 2 pings covers about nine minutes at the 4:30 interval — enough for a short gap, and deliberately short so an forgotten session does not quietly spend all night. Arm a longer window with `/keepalive 2h` when you actually want one.

---

## Recipes

**Step away over lunch.** `/keepalive 1h` before you go. Fourteen pings at 4:30 keep the cache warm for the full hour, and the feature stops itself at the end of the window instead of pinging into the evening.

**Hold a session across a long build.** Kick off the build, then `/keepalive 30m`. When the build finishes and you come back, your next message is served from cache instead of rebuilding a context that now includes a large diff.

**Make the ping do something useful.** `/goal check whether the dev server is still up` keeps the cache alive *and* leaves a small standing instruction in the conversation, so each ping is a chance to notice a crash rather than an empty nod. Keep it short and unambiguous — the model will act on it.

**Bound an overnight run.** If you leave a callback-driven loop working overnight (see [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)), the agent's own activity keeps the cache warm on its own — add `/keepalive 8h` only for the gaps between rounds, and accept the token cost knowingly.

---

## Gotchas

- **Nothing pings until you turn it on.** The feature ships disabled; `CACHE_KEEPALIVE=1`, `/keepalive on`, `/keepalive enable`, a bare duration, or `/goal` all arm it.
- **Pings cost a little each.** Each one is a real model call at your current model and settings. If you are budget-sensitive and away for a long time, arm a bounded window (`/keepalive 30m`) rather than leaving the budget unlimited.
- **The interval must sit inside the cache window.** The default 270 s assumes a 5:00 TTL. Set an interval of 300 s or more and the ping arrives after the cache has already expired — it will miss every time, and the feature will pause on the first miss.
- **A pause is self-protection, not an error.** `paused` means a ping found no cache to read. Send any real message, or run `/keepalive on`, to re-arm it.
- **`off` pauses; `disable` stops.** `off` leaves the timer loaded so `on` can resume it instantly. `disable` tears the timer down for the session; `enable` starts it again from zero counters.
- **Runtime changes do not touch the file.** `/keepalive` and `/goal` update the live session only. To make a change survive a restart, set the matching key in `scorpiox-env.txt`.
- **A fresh session resets it.** If keep-alive is running, `/clear` re-arms it to `idle` and clears the counters, so the next cached response starts a fresh budget. If it is disabled, `/clear` leaves it disabled.
- **Providers that do not report cache usage get nothing out of it.** The hit/miss accounting depends on the provider reporting cached tokens. On a backend that never reports them, every ping is a miss and the timer pauses after the first one — which is the correct outcome, since there is no cache to protect.
- **Nothing fires in the browser build.** The WASM/browser target has no background timer; keep-alive is a no-op there. Native runs and embedded hosts are unaffected.
- **Keep the ping message short.** It is a real turn: whatever you set is sent and answered. `keepalive, respond ok` stays cheap; a paragraph of instructions does not.
- **Pings are visible turns, not hidden calls.** Each ping appears in the transcript as a user message and takes its own turn in the usage history, so a session you left pinging shows a line of small turns when you scroll back. That is deliberate — you can audit exactly what was sent while you were away.

---

## Related

- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md) — the other idle-time timer: it wakes the agent to do work, where keep-alive only keeps the cache warm.
- [Token Usage Observability](usage-observability.md) — the `T mm:ss` countdown, the `KA` marker, and the `keepalive` block in `stats.json`.
- [Configuration and Profiles](scorpiox-env.md) — the cascade every `CACHE_KEEPALIVE_*` key is read through, and `/profile` vs `/use`.
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md) — what happens to the conversation when the cache is allowed to lapse and the session keeps growing.
