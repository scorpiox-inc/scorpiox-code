# Using the /keepalive Command

You start a long agent run, step away for ten minutes to handle something else, and when you come back the first reply takes far longer than it should. The cause is not your machine. Cloud providers cache your prompt — the system instructions, the tool definitions, the whole conversation — so each new message only pays for the delta. That cache is fast and cheap, but it is short-lived: it carries a **five-minute time-to-live (TTL)**. If nothing touches it for five minutes, it is gone, and your next message pays the full price to rebuild everything from scratch.

The `/keepalive` command fixes exactly that. It arms a background timer that, a little before the cache is about to expire, automatically sends a short ping message through the normal agent path — same model, same conversation, same tool settings. Because nothing about the request changed, the provider's cache still matches, and the ping **reads from the cache instead of rewriting it**, which refreshes the TTL. You come back and the first reply is fast, as if you had never left.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** a background timer watches the idle time since the last response, and just before the provider's prompt cache expires it fires a tiny ping through the normal agent path to refresh it — so stepping away never costs you a cold start.

---

## What the timer actually does

The keep-alive timer is a background watcher. It tracks how long it has been since the last API response. When the configured trigger time is reached, it raises a flag, and the main loop picks it up and sends a real user message through the standard agent path — exactly as if you had typed it. Because the conversation, model, and tool configuration are all unchanged, the provider's cache fingerprint still matches, and the ping lands as a cache **read** rather than a write.

The default trigger of **270 s (4 m 30 s)** is deliberately set **30 s inside** the 300 s TTL. That margin matters: even if the provider's clock runs a fraction fast, the ping still lands before the cache expires, so your next real message benefits from the refreshed cache.

Everything the timer controls is a `scorpiox-env.txt` key, so you can tune it in your config, a profile, or an environment variable:

| Property | Default | Config key |
|----------|---------|------------|
| Enabled at startup | No (opt-in) | `CACHE_KEEPALIVE` |
| Ping interval (idle seconds before a ping fires) | 270 s (4 m 30 s) | `CACHE_KEEPALIVE_TRIGGER` |
| Max consecutive misses before pausing | 1 (pause after the first miss) | `CACHE_KEEPALIVE_MAX_TRIES` |
| Max total pings before auto-stop (0 = unlimited) | 2 | `CACHE_KEEPALIVE_MAX_PINGS` |
| Ping message text | `keepalive, respond ok` | `CACHE_KEEPALIVE_MESSAGE` |

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

---

## Command reference

All commands are entered in the chat input line. Type `/keepalive` and press **Tab** to see the available subcommands plus a couple of example durations.

### Toggle the status popup

```
/keepalive
```

With no argument, the command toggles the **keep-alive popup** open or closed. The popup is a floating status panel — it shows live state, the next-ping countdown, and the ping statistics. It does not change any setting; it is a pure status view.

### Pause and resume

```
/keepalive on
/keepalive off
```

`on` starts the timer. If it was previously paused (for example after a cache miss), `on` resumes it and clears the miss streak. `off` **pauses** the timer but keeps its state intact — `on` resumes it later without losing the configuration.

### Full enable and disable

```
/keepalive enable
/keepalive disable
```

`enable` is the fresh-start path: it starts the background timer, clears all statistics, and resets the state to **idle** — armed, waiting for the first response that contains a cache. `disable` stops the timer entirely and discards its state. Use `disable` when you want the feature fully off for the rest of the session, and `off`/`on` when you want a pause you can pick back up.

### Arm for a total duration

```
/keepalive 45m
/keepalive 1h30m
/keepalive 900s
/keepalive 270
```

Pass a duration directly (no subcommand word) to arm the keep-alive for a **total time window**. SCORPIOX CODE works out how many pings fit inside that window at the current interval and sets the ping cap accordingly. The duration format accepts any combination of `h` (hours), `m` (minutes), `s` (seconds), or a bare number (seconds): `1h30m`, `5m`, `900s`, `270`.

If no cache exists yet — you have not sent a message that created one — the timer arms in **idle** and activates automatically on the first cached response. The chat confirms this:

```
Keep-alive for 1h 30m (20 pings every 270s). Armed - starts after your next message creates a cache.
```

### Change the ping interval

```
/keepalive interval 5m
/keepalive interval 270s
```

Changes the trigger time — how long you may stay idle before a ping fires. The value uses the same duration format as above. The change takes effect from the next ping cycle.

### Change the ping message

```
/keepalive message do nothing
```

Sets the text that is sent as the keep-alive ping. Because the ping travels through the normal agent path, make it something the model can answer in one short reply without kicking off a long turn. The default `keepalive, respond ok` is a good template.

### Show and hide the popup

```
/keepalive show
/keepalive hide
```

Explicitly show or hide the popup without toggling. Useful if you want to dismiss the panel without touching the timer.

---

## The /goal shortcut

`/goal` is a one-word alias for setting the keep-alive message. It is handy when you want the ping to carry a specific reminder the model should keep in mind while you are away.

```
/goal finish the refactor
```

This sets the keep-alive message to `finish the refactor`, **auto-enables the keep-alive timer if it was disabled**, and opens the popup so you can see the new message. With no argument, `/goal` just prints the current message:

```
/goal
```

---

## The popup

Type `/keepalive` (no argument) or `/keepalive show` to open the popup. It appears in the **top-right corner** of the terminal by default.

```
 Keep-Alive (active)                          [X]
──────────────────────────────────────────────
 Interval:   4m 30s
 Next ping:  2m 15s
 Pings:      3 / 20
 Hits:       3    Misses: 0
 Max tries:  1 (streak: 0)
 Message:    keepalive, respond ok
```

| Field | Meaning |
|-------|---------|
| **Interval** | The trigger time in human-readable form. |
| **Next ping** | Live countdown to the next scheduled ping. Shows `sending...` while a ping is in flight, and `--` when no ping is pending. |
| **Pings** | Total pings sent in the current window, and the cap. |
| **Hits / Misses** | Pings that read from the cache (hits) versus pings that did not (misses). A miss means the cache was already gone when the ping arrived. |
| **Max tries** | How many consecutive misses are tolerated before the timer pauses. The `streak` count is the current run of consecutive misses. |
| **Message** | The text that will be sent as the next ping. |

### Moving the popup

Drag it by its **title bar** — the coloured header row. Click and hold on the title bar, move, and release to drop it where you want it.

### Closing the popup

Click the `[X]` button in the corner of the title bar, type `/keepalive hide`, or run `/keepalive` again to toggle it closed. Closing the popup does not touch the timer — the keep-alive keeps running.

### Title colours

The title text and its colour reflect the current state:

| State | Title text | Colour |
|-------|-----------|--------|
| Disabled | `Keep-Alive (disabled)` | grey |
| Idle (armed) | `Keep-Alive (idle)` | grey |
| Active (timer running) | `Keep-Alive (active)` | green |
| Sending a ping | `Keep-Alive (pinging)` | purple |
| Paused (after max misses) | `Keep-Alive (paused)` | amber |

---

## Status bar indicator

Once a conversation exists, the bottom status bar shows a small live indicator flush right on the bottom line, next to the cache countdown timer:

| Indicator | Meaning |
|-----------|---------|
| `KA`     | Keep-alive active, no pings sent yet |
| `KA:N`   | Keep-alive active, N pings sent so far |
| `KA*`    | Keep-alive is currently sending a ping |
| `KA\|\|` | Keep-alive is paused |
| `KA-`    | Keep-alive is idle (armed, waiting for the first cache) |
| *(blank)* | Keep-alive is disabled |

---

## How the states work

The keep-alive moves through five states. Reading them helps you interpret the popup and the status bar.

```
 DISABLED  ──/keepalive on──►  IDLE  ──first cached response──►  ACTIVE
    ▲                              │                                  │
    │                              │  (armed, no cache yet)          │
    │                              │                                  │  trigger time reached
    └──/keepalive disable── PAUSED ◄────────── max consecutive misses ─┘
                              │
                              │  /keepalive on
                              └──► IDLE or ACTIVE (depending on whether a cache exists)
```

- **DISABLED** — Timer is off; no timer thread is running. Set with `/keepalive disable`.
- **IDLE** — Timer is running but no cache has been observed yet. It is waiting for the first response that contains cached tokens. The status bar shows `KA-`.
- **ACTIVE** — A cache exists and the timer is counting down to the next ping. The status bar shows `KA` or `KA:N`.
- **PINGING** — A ping is currently being sent through the agent path. The status bar shows `KA*`. This is a brief transitional state.
- **PAUSED** — The timer hit its max consecutive miss count, or you paused it with `/keepalive off`. The status bar shows `KA||`. Resume with `/keepalive on`.

---

## Watching it from outside the terminal

Two session files keep the state visible to anything else on the machine:

- **`keepalive.json`** lives in the session folder (`.scorpiox/sessions/<id>/keepalive.json`). It carries both the control settings (enabled, pause, interval, ping cap, message) and the live status (state, last response time, ping statistics). The remote SCORPIOX BOT control surface edits the control fields there, and the session picks the changes up on its next sync — so you can flip the keep-alive on or off, or change its interval, from a browser or another terminal without touching this one.
- **`stats.json`**, dumped once a second into the same session folder, carries a `keepalive` block with the same state, statistics, and countdown. That is what the web status surface reads.

You can also inspect either file directly while a session runs.

---

## What to keep in mind

- **Per-session and per-process.** Keep-alive state belongs to the running session. Restarting SCORPIOX CODE resets it. If you want it on at startup, set `CACHE_KEEPALIVE=1` in `scorpiox-env.txt`.
- **The ping is a real message.** It goes through the normal agent path, so it consumes a turn and uses the same model, token budget, and tool list as any user message. If the model decides to do something non-trivial in response, it will. Keep the ping message short and unambiguous — `do nothing`, `keepalive, respond ok` — not an open-ended question.
- **A miss is not an error.** If the provider's cache expires between the trigger time and the actual round-trip, the ping will not read from the cache. That is expected under load. `CACHE_KEEPALIVE_MAX_TRIES` controls how many consecutive misses are tolerated before the timer pauses, and the pause is never fatal — `/keepalive on` resumes it.
- **The trigger must be under 300 s.** The cache TTL is five minutes. Set the trigger at 300 s or above and every ping arrives after the cache has already expired, so it always misses.
- **`/keepalive off` is a pause, not a disable.** The timer keeps running in the background. Use `/keepalive disable` to stop it entirely for the rest of the session.
- **The duration argument replaces the ping cap.** If your config says `CACHE_KEEPALIVE_MAX_PINGS=2` and you then run `/keepalive 2h`, the cap is recalculated to fit two hours at the current interval. The config value is only the starting default; the runtime command wins.
- **Local models have nothing to keep alive.** This feature only pays off with providers that maintain a prompt cache. Against a local model the pings never produce a cache hit, so the timer pauses after the first miss.
- **`/clear` re-arms the timer.** Clearing the conversation invalidates the old cache fingerprint, so the keep-alive resets to **idle** and re-activates when your next message creates a fresh cache.

---

*Related: [Scheduled Callbacks](callbacks.md) covers the sibling mechanism for timed work that should keep a session busy on purpose. The configuration cascade, profiles, and where each key may live are covered in [Configuration and Profiles](scorpiox-env.md).*
