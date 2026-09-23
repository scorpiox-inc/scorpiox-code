# Using the /keepalive Command

Every time SCORPIOX CODE sends a request to a cloud model, the provider caches the prompt — system instructions, tools, and the full conversation — so the next request in the same session can skip re-reading it. That cache is fast and cheap, but it is short-lived: providers give it a **five-minute time-to-live (TTL)**. If nothing touches it for five minutes, it is gone, and your next message pays the full price to rebuild it from scratch.

For a long agent run where you step away for ten minutes, that means the first reply when you come back is slow and expensive — the provider is re-processing everything.

The `/keepalive` command solves this. It arms a background timer that, just before the cache is about to expire, automatically sends a lightweight ping through the normal agent path — same model, same settings, same conversation. Because nothing about the request changed, the provider's cache fingerprint still matches, so the ping **reads from the cache instead of writing to it**, refreshing the TTL. You come back and the first reply is fast, as if you had never left.

Source of truth: `sx_cache_keepalive.c`, `sx_cache_keepalive.h`, `sxui_keepalive.c`, `sxui_keepalive.h`, `sx_slashcmd.c`, `sx.c`, `sx_statusbar.c`, and `scorpiox-env.txt` at commit `6c70ad6`.

---

## What the timer actually does

The keep-alive timer is a background watcher. It tracks the time since the last API response, and when the configured trigger point is reached it raises a flag. The main loop picks that flag up and sends a real message through the standard agent path — exactly as if you had typed it. Because the conversation, model, and tool configuration are unchanged, the provider's cache fingerprint matches, and the ping lands as a cache **read** rather than a write.

| Property | Default | Config key |
|----------|---------|------------|
| Enabled at startup | No (opt-in) | `CACHE_KEEPALIVE` |
| Ping interval (seconds idle before pinging) | 270 s (4 min 30 s) | `CACHE_KEEPALIVE_TRIGGER` |
| Max consecutive misses before pausing | 1 | `CACHE_KEEPALIVE_MAX_TRIES` |
| Max total pings before auto-stop | 2 (0 = unlimited) | `CACHE_KEEPALIVE_MAX_PINGS` |
| Ping message text | `keepalive, respond ok` | `CACHE_KEEPALIVE_MESSAGE` |

The default trigger of 270 s is deliberately set **30 s inside** the 300 s TTL. That leaves a safety margin: even if the provider's clock runs a fraction fast, the ping still lands before the cache expires, so the next real message benefits from the refreshed cache.

---

## Command reference

All commands are entered in the chat input line. Type `/keepalive` and press **Tab** to see the available subcommands and a couple of example durations.

### Toggle the popup

```
/keepalive
```

With no argument, the command toggles the **keep-alive popup** open or closed. The popup is a floating panel that shows live state, the next-ping countdown, and ping statistics. It does not change any settings — it is a pure status view.

### Enable and disable

```
/keepalive on
/keepalive off
```

`on` enables the timer. If it was previously paused (for example after a cache miss), `on` resumes it. `off` pauses the timer but does not destroy state — `on` resumes it later.

```
/keepalive enable
/keepalive disable
```

`enable` is the full enable path: it starts the background thread and resets the state to **idle** (waiting for the first cached response). `disable` stops the thread entirely and discards all statistics. Use `disable` when you want to switch the feature off for the rest of the session.

### Arm for a total duration

```
/keepalive 45m
/keepalive 1h30m
/keepalive 900s
```

Pass a duration directly (no subcommand word) to arm the keep-alive for a **total time window**. The command works out how many pings fit in that window at the current interval and sets the ping cap accordingly. The duration format accepts any combination of `h` (hours), `m` (minutes), `s` (seconds), or a bare number (seconds): `1h30m`, `5m`, `900s`, `270`.

At the default 270 s interval, the window maps to a ping count like this:

| Command | Window | Pings armed |
|---------|--------|-------------|
| `/keepalive 45m`  | 2700 s | 10 |
| `/keepalive 1h`   | 3600 s | 14 |
| `/keepalive 1h30m`| 5400 s | 20 |
| `/keepalive 2h`   | 7200 s | 27 |

If no cache exists yet (you have not sent a message that created one), the timer arms in **idle** and activates automatically on the first cached response. The chat confirms this:

```
Keep-alive for 1h 30m (20 pings every 270s). Armed - starts after your next message creates a cache.
```

### Change the ping interval

```
/keepalive interval 5m
/keepalive interval 270s
```

Changes the trigger time — how long you can stay idle before a ping fires. The value is a duration, in the same format as the total-duration argument. The change takes effect from the next ping cycle.

### Change the ping message

```
/keepalive message do nothing
```

Sets the text sent as the keep-alive ping. The message travels through the normal agent path, so it should be something the model can answer briefly without kicking off a long turn. The default `keepalive, respond ok` is a good template.

### Show and hide the popup

```
/keepalive show
/keepalive hide
```

Explicitly show or hide the popup without toggling. Useful if you want to dismiss the popup without affecting the timer.

---

## The /goal shortcut

`/goal` is a one-word alias for setting the keep-alive message. It is handy when you want the ping to carry a specific reminder the model should keep in mind.

```
/goal finish the refactor
```

This sets the keep-alive message to `finish the refactor` **and** auto-enables the keep-alive timer if it was disabled. The popup opens to confirm the new message.

With no argument, `/goal` just prints the current message:

```
/goal
```

---

## The popup

Type `/keepalive` (no args) or `/keepalive show` to open the popup. It appears in the **top-right corner** of the terminal by default.

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
| **Hits / Misses** | Pings that read from cache (hits) versus pings that did not (misses). A miss means the cache was already gone when the ping arrived. |
| **Max tries** | How many consecutive misses are tolerated before the timer pauses. The `streak` count is the current run of consecutive misses. |
| **Message** | The text that will be sent as the next ping. |

### Moving the popup

Drag it by its **title bar** (the coloured header row): click and hold on the title bar, move, and release to drop it where you want.

### Closing the popup

Click the `[X]` button in the top-right corner, press `/keepalive hide`, or type `/keepalive` again to toggle it closed. Closing the popup does not affect the timer — the keep-alive keeps running.

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

When a conversation exists, the bottom status bar shows a small live indicator flush on the right of the bottom line, next to the idle timer:

| Indicator | Meaning |
|-----------|---------|
| `KA`    | Keep-alive active, no pings sent yet |
| `KA:N`  | Keep-alive active, N pings sent so far |
| `KA*`   | Keep-alive is currently sending a ping |
| `KA\|\|`| Keep-alive is paused |
| `KA-`   | Keep-alive is idle (armed, waiting for first cache) |
| *(blank)* | Keep-alive is disabled |

The indicator is colour-coded — green for active, yellow for pinging, orange for paused, grey for idle — and is purely informational. The same data is in the popup if you want more detail.

---

## Configuration keys

All keys are set in `scorpiox-env.txt` in your SCORPIOX CODE data directory, or in the environment when launching. Every one of them can also be changed at runtime with the `/keepalive` commands above. See the [scorpiox-env reference](scorpiox-env.md) for the full config file.

### `CACHE_KEEPALIVE`

| Value | Meaning |
|-------|---------|
| `0` | Disabled at startup (default). You can still enable at runtime with `/keepalive on` or a duration command. |
| `1` | Enabled at startup. The timer arms as soon as the first cached response is received. |

### `CACHE_KEEPALIVE_TRIGGER`

Seconds of idle time before a ping is triggered. Default **270** (4 min 30 s). It must be less than 300 (the five-minute TTL) to be useful. Lower values ping more frequently and keep the cache warmer, at the cost of more ping API calls.

### `CACHE_KEEPALIVE_MAX_TRIES`

Maximum number of **consecutive** cache misses before the timer pauses itself. Default **1**. A miss means the provider did not return any cached tokens for the ping — the cache was already gone. After N consecutive misses the timer pauses to avoid burning API calls on a dead cache. Resume with `/keepalive on`.

### `CACHE_KEEPALIVE_MAX_PINGS`

Maximum **total** number of pings before the timer stops, regardless of hits. Default **2** (roughly 9 minutes at the default 270 s interval). Set to `0` for unlimited. When you use a duration argument (`/keepalive 1h`), this value is overwritten automatically to fit the window.

### `CACHE_KEEPALIVE_MESSAGE`

The text sent as the keep-alive ping. Empty (default) resolves to `keepalive, respond ok`. Maximum **256 characters**. The message is sent as a real user message through the standard agent path, so it should be something the model can answer briefly.

---

## When to use keep-alive

**Use it when:**

- You are running a long agent task and expect to step away for more than five minutes between checks.
- You are iterating on a large conversation (100+ turns) and want to avoid the cold-start penalty each time you return.
- The agent is in a loop (building, testing, fixing) and you want to keep the conversation warm between your interventions.

**You do not need it when:**

- You are actively typing every few minutes. Your own messages already refresh the cache.
- You are using a local model. Local inference has no prompt-cache TTL to maintain, so there is nothing to keep alive.
- You are in a new session with very little conversation. The cache is small and cheap to rebuild.

Keep-alive only has an effect with providers that report cache usage (the Claude/Anthropic-style prompt cache). With providers that do not maintain a prompt cache, the pings will never produce a cache hit, and the timer will pause after the first miss.

---

## How the states work

The keep-alive moves through five states. Understanding them helps you read the popup and the status bar.

```
 DISABLED  ──/keepalive on──►  IDLE  ──first cached response──►  ACTIVE
    ▲                              │                                  │
    │                              │  (armed, no cache yet)           │
    │                              │                                  │  trigger time reached
    └──/keepalive disable── PAUSED ◄────────── max consecutive misses ─┘
                              │
                              │  /keepalive on
                              └──► IDLE or ACTIVE (depending on whether a cache exists)
```

- **DISABLED** — Timer is off. No thread running. Set with `/keepalive disable`.
- **IDLE** — Timer is running but no cache has been observed yet. It is waiting for the first response that contains cached tokens. The status bar shows `KA-`.
- **ACTIVE** — A cache exists and the timer is counting down to the next ping. The status bar shows `KA` or `KA:N`.
- **PINGING** — A ping is currently being sent through the agent path. The status bar shows `KA*`. This is a brief transitional state.
- **PAUSED** — The timer hit its max consecutive miss count, or you paused it with `/keepalive off`. The status bar shows `KA||`. Resume with `/keepalive on`.

---

## Gotchas

- **Keep-alive is per-session and per-process.** Restarting SCORPIOX CODE resets all keep-alive state. If you want it on at startup, set `CACHE_KEEPALIVE=1` in `scorpiox-env.txt`.

- **The ping goes through the normal agent path.** It consumes a turn and uses the same model, the same token budget, and the same tool list as a normal user message. If the model decides to do something non-trivial in response to the ping message, it will. Keep the message short and unambiguous.

- **A miss is not an error.** If the provider's cache expires between your trigger and the actual API round-trip, the ping will not read from cache. This is expected under load. `CACHE_KEEPALIVE_MAX_TRIES` controls how many misses are tolerated before the timer pauses.

- **The trigger must be less than 300 s.** The provider cache TTL is five minutes. If you set the trigger to 300 s or more, the ping arrives after the cache has already expired and will always miss.

- **`/keepalive off` is a pause, not a full disable.** The background thread keeps running. If you want to stop the thread entirely and free it, use `/keepalive disable`.

- **The duration argument replaces the ping cap.** If you set `CACHE_KEEPALIVE_MAX_PINGS=2` in your config file and then run `/keepalive 2h`, the runtime cap is recalculated to fit two hours at the current interval. The config file value is only the initial default at startup.

- **Local models have no cache to keep alive.** The feature is only meaningful with providers that maintain a prompt cache. It will still run, but the pings will never produce a cache hit, so the timer pauses after the first miss.

---

*Docs for SCORPIOX CODE @ 6c70ad6*
