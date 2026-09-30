# Using the /keepalive Command

Long agent sessions are expensive to interrupt. Every time you stop and come back, the model's **prompt cache** may have expired, and your next message has to re-send and re-create the entire context from scratch — a slower, more expensive turn. SCORPIOX CODE's **cache keep-alive** solves this: it quietly keeps your context warm while you are away, so when you return, your next message lands on a hot cache instead of a cold start.

The whole thing is driven by one built-in command — **`/keepalive`** — plus a small, draggable popup that shows exactly what is happening. There is also a **`/goal`** shortcut for the most common case: "keep my session alive, and here is what to focus on."

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** while you are idle, SCORPIOX CODE auto-sends a tiny ping through the normal conversation just before the provider's cache expires. The ping hits the existing cache (it is the same prompt), which refreshes the timer — so your next real message is a cache hit, not a rebuild.

---

## Why a prompt cache expires in the first place

Most hosted model providers do not keep your full context on a hot shelf forever. They cache it for a short window (on the order of a few minutes) to save themselves cost. While the window is open, follow-up turns are cheap and fast because the provider reads the cached prefix instead of re-processing it.

The moment the window closes, the next turn pays the full price again. For a short task that does not matter. For a long session where you step away — to review something, take a call, or run a build in parallel — it does. You lose the exact speed and cost you were enjoying.

Keep-alive is the fix: instead of letting the window lapse, SCORPIOX CODE nudges the conversation right before it does. Because the nudge is a normal message with the same prompt, it is served from the cache (a **cache read**) and, in doing so, resets the expiry clock. Your context stays warm for as long as you want it to.

### What a keep-alive ping actually is

It is not a background process and not a webhook. It is **a real user message sent through the same agent path you use for everything else** — same provider settings, same tools, same model. The only difference is that the timer typed it for you. That has two consequences worth knowing:

- **The cache fingerprint matches perfectly.** A ping is indistinguishable from your own message, so the provider treats it as a cache read — the mechanism that keeps the cache alive.
- **It costs a little.** Each ping is a small model call. That is the trade: a few cheap pings to avoid one expensive cold rebuild. You control the rate and the total via the command and the config keys below.

---

## The `/keepalive` command

Type `/keepalive` on its own to **toggle the status popup**. With arguments, it controls behavior:

| Command | What it does |
|---------|--------------|
| `/keepalive` | Toggle the popup (show / hide). |
| `/keepalive on` | Turn keep-alive on — or resume it if it paused. |
| `/keepalive off` | Pause keep-alive. The context will be allowed to lapse. |
| `/keepalive enable` | Start the background timer (the full on-ramp). |
| `/keepalive disable` | Stop the background timer entirely. |
| `/keepalive show` | Show the popup. |
| `/keepalive hide` | Hide the popup. |
| `/keepalive interval <duration>` | Set how often it pings — e.g. `5m`, `270s`, `1h`. |
| `/keepalive message <text>` | Set the text of the ping. |
| `/keepalive <duration>` | Keep the cache alive for a **total** duration — e.g. `45m`, `2h`, `1h30m`. The ping count is computed for you. |

Durations accept the same compact forms throughout: `90` (seconds), `270s`, `5m`, `2h`, or a mix like `1h30m`.

### A practical example

You are deep in a refactor and need to step away for an hour, but you want the session to be ready the instant you are back. Keep the interval at the default (~4:30) and just set a total:

```
/keepalive 1h
```

SCORPIOX CODE responds with the plan — how many pings at what interval, and whether it is armed and waiting for your next message to create a cache. The popup opens so you can watch it work.

### The `/goal` shortcut

`/goal` is an alias for setting the keep-alive message, with a convenience twist: if keep-alive is off, it **turns itself on**.

```
/goal keep the API tests green
```

This sets the ping text to "keep the API tests green" and enables keep-alive if it was disabled. Running `/goal` with no text prints the current goal instead of changing it.

Use `/goal` when you want the ping to double as a reminder of what to focus on; use `/keepalive message <text>` when you just want to change the text.

---

## The keep-alive popup

The popup is a small, draggable card (it opens in the top-right by default) that shows the live state of the feature. Drag it anywhere; click **`[X]`** to close it without changing behavior.

It shows:

- **State** — in the title: `disabled`, `idle`, `active`, `pinging`, or `paused`.
- **Interval** — the configured ping interval.
- **Next ping** — a live countdown to the next ping (while active), or `sending...` while a ping is in flight.
- **Pings** — how many pings have been sent, against the maximum (when one is set).
- **Hits / Misses** — how many pings actually read from the cache versus how many came back cold.
- **Max tries** — the consecutive-miss threshold, with the current streak.
- **Message** — the text being sent as each ping.

A healthy session sits in `active` with a steadily counting-down **Next ping** and a growing **Hits** count. If you start to see **Misses**, the cache is not surviving between pings — the provider window is shorter than your interval, or the session was interrupted — and the feature will pause itself after the configured number of consecutive misses rather than keep spending on pings that are not helping.

---

## The status-bar indicator

You do not need the popup to keep an eye on things. The status bar carries a compact, color-coded indicator:

- **Cache timer** — a `T mm:ss` countdown to cache expiry, green when healthy, yellow as it nears, red when it is about to lapse.
- **Keep-alive state** — a short `KA` marker: `KA` when active, `KA:n` when active with `n` pings already sent, `KA*` while a ping is in flight, `KA-` when armed and idle, and `KA||` when paused.

Together these let you glance at the bottom of the screen and know whether your context is warm, without opening anything.

---

## Configuration keys

Keep-alive is configured through `scorpiox-env.txt` (any cascade tier, a named profile, or an OS environment variable — see [Configuration and Profiles](scorpiox-env.md)). Command-line changes made with `/keepalive` take effect for the current session; these keys set the defaults it starts from.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `CACHE_KEEPALIVE` | bool | `0` (off) | Master switch. When `0`, keep-alive starts disabled and you turn it on with `/keepalive on`. |
| `CACHE_KEEPALIVE_TRIGGER` | int (seconds) | `270` | Seconds of idle before a ping is sent. Default is 4:30 — just inside a typical provider cache window. |
| `CACHE_KEEPALIVE_MAX_TRIES` | int | `1` | Consecutive cache-miss pings before the feature pauses itself. |
| `CACHE_KEEPALIVE_MAX_PINGS` | int | `12` | Maximum total pings before it stops on its own. `0` means unlimited. The default (~12 pings at 4:30) covers roughly an hour. |
| `CACHE_KEEPALIVE_MESSAGE` | text | `keepalive, respond ok` | The text of each ping. Override it here, or at runtime with `/keepalive message <text>` / `/goal <text>`. |

Because a `/keepalive <duration>` command derives the ping count from the interval and the total time, the keys are the steady-state defaults and the command is the session-level override. Pick whichever you find natural — the popup reflects whichever is in effect.

---

## How the loop behaves

The behavior is deliberately conservative — it spends a little to save a lot, and it stops spending the moment that is no longer true:

1. **Armed, waiting for a cache.** When enabled, keep-alive starts in an *idle/armed* state. It has nothing to protect until a real response has written a cache.
2. **Active once a cache exists.** The first cached response arms the timer. From then on it watches time-since-last-response.
3. **Ping just before expiry.** As the idle time approaches the trigger, it sends the ping. The ping is a cache read, which refreshes the window, and the clock resets.
4. **Self-stops when it stops working.** If pings start coming back cold (cache misses) up to the max-tries threshold, or the total ping budget is reached, it pauses or stops instead of continuing to spend.

The net effect: while you are away, your context stays warm on a quiet cadence; the moment the cadence is no longer buying you a cache hit, it stands down.

---

## Gotchas

- **Pings cost a little each.** A ping is a real model call. The feature exists to avoid a full cold rebuild, but if you are on a tight budget and away for a very long time, set a bounded duration (`/keepalive 30m`) rather than leaving it unlimited.
- **The provider decides the cache window.** The trigger default assumes a typical few-minute window. If your provider expires caches faster than your interval, you will see **Misses** in the popup and the feature will pause. Lower `CACHE_KEEPALIVE_TRIGGER` to match a short window.
- **`off` pauses, it does not destroy context.** Pausing lets the cache lapse naturally — the conversation is intact on disk. It just means your next message may pay for a rebuild. See [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md).
- **`enable`/`disable` vs `on`/`off`.** `enable`/`disable` start and stop the background timer outright; `on`/`off` resume and pause a timer that is already running. For day-to-day use, `on` and `off` are what you will reach for.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
