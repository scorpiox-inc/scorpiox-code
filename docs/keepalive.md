# Using the /keepalive Command

Long agent sessions are expensive to interrupt. Every time you stop and come back, the model's **prompt cache** may have expired, and your next message has to re-send and re-create the entire context from scratch — a slower, more expensive turn. SCORPIOX CODE's **cache keep-alive** solves this: it quietly keeps your context warm while you are away, so when you return, your next message lands on a hot cache instead of a cold start.

The whole thing is driven by one built-in command — **`/keepalive`** — plus a small, draggable popup that shows exactly what is happening. There is also a **`/goal`** shortcut for the most common case: "keep my session alive, and here is what to focus on."

Docs for SCORPIOX CODE @ `e30b171`.

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

The timer itself is deliberately passive: it only watches the clock and raises a flag. The main loop picks that flag up and sends the ping **only when the agent is genuinely idle** — never in the middle of a turn or on top of another request. The timer never touches your conversation directly.

---

## The `/keepalive` command

Type `/keepalive` and press **Tab** to see the available subcommands and a couple of example durations. On its own, `/keepalive` **toggles the status popup**. With arguments, it controls behavior:

| Command | What it does |
|---------|--------------|
| `/keepalive` | Toggle the popup (show / hide). No setting changes. |
| `/keepalive on` | Turn keep-alive on — or resume it if it had paused. |
| `/keepalive off` | Pause keep-alive. The context will be allowed to lapse. |
| `/keepalive enable` | Start the background timer from a clean state (statistics reset). |
| `/keepalive disable` | Stop the background timer entirely. |
| `/keepalive show` | Show the popup. |
| `/keepalive hide` | Hide the popup. |
| `/keepalive interval <duration>` | Set how often it pings — e.g. `5m`, `270s`, `1h`. |
| `/keepalive message <text>` | Set the text of the ping. |
| `/keepalive <duration>` | Keep the cache alive for a **total** duration — e.g. `45m`, `2h`, `1h30m`. The ping count is computed for you. |

Durations accept the same compact forms throughout: `90` (a bare number means seconds), `270s`, `5m`, `2h`, or a mix like `1h30m`.

### `enable` / `disable` versus `on` / `off`

These four look alike but do different things:

- **`enable`** starts the timer from scratch: it resets the counters and waits for a cache. **`disable`** stops the background timer outright.
- **`on`** is the everyday switch. From a disabled state it enables the feature; from a paused state it **resumes** it without wiping your statistics.
- **`off`** **pauses** rather than destroys. The conversation is untouched on disk; keep-alive simply stops pinging and lets the cache lapse. Bring it back with `/keepalive on`.

For day-to-day use, reach for `on` and `off`.

### A practical example

You are deep in a refactor and need to step away for an hour, but you want the session to be ready the instant you are back. Keep the interval at the default (~4:30) and just set a total:

```
/keepalive 1h
```

SCORPIOX CODE works out how many pings fit in that window at the current interval and caps the ping count accordingly. If no cache exists yet, it arms in **idle** and starts automatically on the first cached response, and the chat line tells you so:

```
Keep-alive for 1h (14 pings every 270s). Armed - starts after your next message creates a cache.
```

At the default 270-second interval, the windows map to ping counts like this:

| Command | Window | Pings armed |
|---------|--------|-------------|
| `/keepalive 45m` | 2700 s | 10 |
| `/keepalive 1h` | 3600 s | 14 |
| `/keepalive 1h30m` | 5400 s | 20 |
| `/keepalive 2h` | 7200 s | 27 |

### Changing the interval

```
/keepalive interval 5m
/keepalive interval 270s
```

Sets how long you can stay idle before a ping fires. The value is a duration in the same format as the total-duration argument. Setting it also arms and activates keep-alive, and resets the countdown from that point.

### Changing the ping message

```
/keepalive message do nothing
```

Sets the text sent as each ping. Because the message travels through the normal agent path, keep it short and unambiguous — something the model can answer in a word without kicking off a long turn. The built-in default, `keepalive, respond ok`, is a good template.

---

## The `/goal` shortcut

`/goal` is a one-word alias for setting the keep-alive message, with a convenience twist: if keep-alive is off, it **turns itself on**.

```
/goal keep the API tests green
```

This sets the ping text to "keep the API tests green" and enables keep-alive if it was disabled. The popup opens to confirm the new message.

With no argument, `/goal` just prints the current message:

```
/goal
```

Use `/goal` when you want the ping to double as a reminder of what to focus on; use `/keepalive message <text>` when you only want to change the text.

---

## The keep-alive popup

Type `/keepalive` (no args) or `/keepalive show` to open the popup. It is a small card (40 columns by 9 rows) that opens in the **top-right corner** by default and shows the live state of the feature:

```
 Keep-Alive (active)                          [X]
──────────────────────────────────────────────────
 Interval:   4m 30s
 Next ping:  2m 15s
 Pings:      3 / 20
 Hits:       3    Misses: 0
 Max tries:  1 (streak: 0)
 Message:    keepalive, respond ok
```

| Field | Meaning |
|-------|---------|
| **Title state** | One of `disabled`, `idle`, `active`, `pinging`, or `paused`. The title's colour tracks it: green for active, grey for disabled/idle, orange for paused, violet while sending. |
| **Interval** | The configured ping interval, in human-readable form. |
| **Next ping** | A live countdown to the next ping while active; `sending...` while a ping is in flight; `--` when no ping is pending. |
| **Pings** | Pings sent in the current window, against the cap (when a cap is set). |
| **Hits / Misses** | Pings that actually read from the cache, versus pings that came back cold. |
| **Max tries** | The consecutive-miss threshold, with the current streak in parentheses. |
| **Message** | The text that will be sent as the next ping. |

Drag the popup anywhere by its **title bar** (click and hold the coloured header row, move, release). Click **`[X]`** or type `/keepalive hide` to dismiss it — hiding the popup does **not** touch the timer; keep-alive keeps pinging in the background.

A healthy session sits in `active` with a steadily counting-down **Next ping** and a growing **Hits** count. If you start to see **Misses**, the cache is not surviving between pings — the provider window is shorter than your interval, or the session was interrupted — and the feature will pause itself after the configured number of consecutive misses rather than keep spending on pings that are not helping.

---

## The status-bar indicator

You do not need the popup to keep an eye on things. The status bar carries a compact, colour-coded block on the bottom-right:

- **Cache timer** — a `T mm:ss` countdown to cache expiry. It is green while the cache is comfortably fresh, yellow as it nears the peak, and red in the last minute (or once expired, when it reads `T --:--`). The green-to-yellow boundary sits at 60% of the keep-alive interval, so with the default 4:30 interval the timer turns yellow at **2:42** remaining, not 3:00.
- **Keep-alive state** — a short marker next to it:

| Marker | Meaning |
|--------|---------|
| `KA` | Active, no pings sent yet. |
| `KA:n` | Active, with `n` pings sent so far. |
| `KA*` | A ping is in flight. |
| `KA-` | Armed and idle, waiting for the first cache. |
| `KA\|\|` | Paused. |
| *(blank)* | Disabled. |

Both fields sit at fixed positions, so the block never slides around as the values change.

---

## Configuration keys

Keep-alive is configured through `scorpiox-env.txt` — any cascade tier, a named profile, or an OS environment variable. See [Configuration and Profiles](scorpiox-env.md) for the cascade and precedence rules. Command-line changes made with `/keepalive` or `/goal` take effect for the current session; these keys set the defaults it starts from.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `CACHE_KEEPALIVE` | bool | `0` (off) | Master switch. `0` = keep-alive starts disabled and you turn it on with `/keepalive on`; `1` = armed at startup. |
| `CACHE_KEEPALIVE_TRIGGER` | int (seconds) | `270` | Seconds of idle before a ping is sent. The default, 4:30, sits 30 seconds inside a typical 5:00 cache TTL. |
| `CACHE_KEEPALIVE_MAX_TRIES` | int | `1` | Consecutive cache-miss pings tolerated before the feature pauses itself. |
| `CACHE_KEEPALIVE_MAX_PINGS` | int | `2` | Maximum total pings before it stops on its own. `0` means unlimited. A `/keepalive <duration>` command overwrites this to fit the window. |
| `CACHE_KEEPALIVE_MESSAGE` | text | *(empty)* | The text of each ping. Empty resolves to the built-in default `keepalive, respond ok`. Maximum 256 characters. |

A ready-to-use block:

```ini
# Keep the prompt cache warm while I step away.
CACHE_KEEPALIVE=1
CACHE_KEEPALIVE_TRIGGER=270
CACHE_KEEPALIVE_MAX_TRIES=1
CACHE_KEEPALIVE_MAX_PINGS=2
CACHE_KEEPALIVE_MESSAGE=keepalive, respond ok
```

In the config editor (`/config` in a session, or `scorpiox-config` from a shell) these keys live in the **Cache** section. The four tuning keys appear only once `CACHE_KEEPALIVE` is switched on, which keeps the editor list short when you are not using the feature.

Because a `/keepalive <duration>` command derives the ping count from the interval and the total time, the keys are the steady-state defaults and the command is the session-level override. Pick whichever you find natural — the popup reflects whichever is in effect.

---

## How the loop behaves

The behavior is deliberately conservative — it spends a little to save a lot, and it stops spending the moment that is no longer true:

1. **Armed, waiting for a cache.** When enabled, keep-alive starts in an *idle/armed* state. It has nothing to protect until a real response has written a cache, so it does nothing on a brand-new session.
2. **Active once a cache exists.** The first response that reads or writes cache tokens arms the countdown. From then on it watches time since the last response.
3. **Ping just before expiry.** As the idle time reaches the interval, it queues the ping. The main loop sends it through the normal agent path when the agent is idle — never mid-turn — and the state flips to `pinging` until the response lands.
4. **Score the ping.** A ping that reads from cache is a hit and resets the clock. A ping that *recreates* the cache (read zero, write non-zero) also counts as a hit — a cache now exists and the next ping will read it. Only a ping with neither counts as a miss.
5. **Stand down when it stops working.** After the configured consecutive misses, or once the ping budget is spent, it pauses instead of continuing to spend on a dead cache. Bring it back with `/keepalive on`.

Two things hand it a fresh start: **any real message you send** resets the ping count and the miss streak, and **`/clear`** re-arms the feature from idle, since the conversation it was protecting is gone. The default budget of 2 pings covers about nine minutes at the 4:30 interval — enough for a short gap, and deliberately short so a forgotten session does not quietly spend all night. Arm a longer window with `/keepalive 1h` when you actually want one.

---

## Driving keep-alive from other tools

The feature is not confined to the terminal. Its live state is published, and its controls are readable and writable, so a remote dashboard or your own script can see and steer it.

- **`stats.json`** — the session's telemetry file carries a `keepalive` block with the same numbers the popup shows: `state`, `ping_count`, `trigger_sec`, `max_pings`, `max_tries`, `cache_hits`, `cache_misses`, `consecutive_misses`, `remaining_s`, and `message`. That is the read-only view for scripts and dashboards; see [Token Usage Observability](usage-observability.md).
- **`keepalive.json`** — a small control-and-status document in the session folder (`.scorpiox/sessions/<session>/keepalive.json`). Anything that can write it can change `enabled`, `paused`, `trigger_sec`, `max_pings`, `max_tries`, and `message`, and the running session picks the change up within about half a second between turns. The agent publishes its status and statistics back into the same file, and every writer bumps a revision counter and swaps the file into place atomically, so a partial read is never visible. This is exactly the mechanism the SCORPIO BOT dashboard uses to show and edit keep-alive for a live session; see [Remote Agent Control and Fleet Management with SCORPIOX BOT](scorpiox-bot.md). It is the twin of the callbacks file described in [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md).
- **The file follows the session.** `/compact`, `/resume`, and `/clear` all rotate the session folder, and the keep-alive file is re-bound to the new folder so remote tooling always sees the live session, not a stale one.
- **Worktrees stay consistent.** Inside a git worktree the file is written to both the worktree's session folder and the main repository's, so tooling watching the main repo stays in sync.

---

## Gotchas

- **Pings cost a little each.** A ping is a real model call. The feature exists to avoid a full cold rebuild, but if you are on a tight budget and away for a very long time, set a bounded duration (`/keepalive 30m`) rather than leaving it unlimited.
- **The provider decides the cache window.** The trigger default assumes a typical few-minute window. If your provider expires caches faster than your interval, you will see **Misses** in the popup and the feature will pause. Lower `CACHE_KEEPALIVE_TRIGGER` to match a short window.
- **Keep the trigger under the cache TTL.** If you set the interval longer than the provider's cache lifetime, every ping arrives after the cache has already gone and will always miss-report. The shipped 4:30 sits deliberately inside a typical 5:00 window.
- **A miss is not an error.** If the provider's cache expires between your trigger and the API round-trip, the ping simply will not read from cache. That is expected under load; `CACHE_KEEPALIVE_MAX_TRIES` controls how many misses are tolerated before the timer pauses.
- **`off` pauses, it does not destroy context.** Pausing lets the cache lapse naturally — the conversation is intact on disk. It just means your next message may pay for a rebuild. See [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md).
- **`enable`/`disable` versus `on`/`off`.** `enable`/`disable` start and stop the background timer outright (and `enable` resets the counters); `on`/`off` resume and pause a timer that already exists. For day-to-day use, `on` and `off` are what you will reach for.
- **Closing the popup is not turning it off.** `[X]` and `/keepalive hide` only hide the status panel; the timer keeps pinging. Use `/keepalive off` to pause it.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
- [Token Usage Observability](usage-observability.md)
- [Remote Agent Control and Fleet Management with SCORPIOX BOT](scorpiox-bot.md)
