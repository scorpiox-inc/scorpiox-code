# Using the /keepalive Command

When you work with a long-running conversation, the model's **prompt cache** is your friend. It avoids re-processing the entire context on every turn, cutting both latency and token cost. But caches are ephemeral: after a fixed TTL the provider drops them and the next turn pays the full re-computation price.

SCORPIOX CODE includes a built-in **cache keep-alive** feature that solves this. When the conversation goes idle, SCORPIOX CODE automatically sends a lightweight ping message through the normal agent path before the cache expires, keeping the fingerprint warm so the next real turn hits the cache instead of missing it.

You control everything through the `/keepalive` slash command and its **draggable popup**, and tune the defaults in `scorpiox-env.txt`.

Docs for SCORPIOX CODE @ `b59223a`.

---

## How cache keep-alive works

The prompt cache the provider maintains for your conversation has a **5-minute TTL**. SCORPIOX CODE's keep-alive timer fires a ping at **4:30** (270 s) of inactivity — 30 s before the cache would expire. The ping is a real user message routed through the same agent path as anything you type, so:

- It carries the same provider, model, tools, and thinking settings as your conversation.
- Its cache fingerprint matches the existing cache exactly, producing a guaranteed `cache_read` hit.
- The timer resets on every response, so the cadence is always anchored to the last API round-trip, not to when you issued a command.

The keep-alive thread itself never talks to the provider or touches the conversation history. It is a timer that raises a flag; the main loop performs the actual send.

### States

| State | Meaning |
|-------|---------|
| **disabled** | Feature turned off (config or `/keepalive disable`). No timer runs. |
| **idle** | Armed but no cache exists yet. Auto-promotes to **active** on the first cached response. |
| **active** | Monitoring the idle timer. Will send a ping when the trigger point is reached. |
| **pinging** | The main loop is currently dispatching the keep-alive message. |
| **paused** | Consecutive cache misses reached `CACHE_KEEPALIVE_MAX_TRIES`. Stopped until you resume. |

---

## The `/keepalive` slash command

`/keepalive` is your control surface. Type it in the input line to manage the feature without asking the agent.

| Command | What it does |
|---------|--------------|
| `/keepalive` | Toggle the popup open/closed. |
| `/keepalive on` | Enable keep-alive (or resume if paused). |
| `/keepalive off` | Pause keep-alive (timer stops; does not destroy state). |
| `/keepalive enable` | Enable and start the background timer thread. |
| `/keepalive disable` | Disable and stop the thread entirely. |
| `/keepalive show` | Show the popup. |
| `/keepalive hide` | Hide the popup. |
| `/keepalive message <text>` | Set the ping message (max 256 chars). |
| `/keepalive interval <duration>` | Set the idle interval before pinging (e.g. `5m`, `270s`). |
| `/keepalive <duration>` | Set a **total** keep-alive window. SCORPIOX CODE computes the number of pings needed (e.g. `2h`, `1h30m`, `45m`). |

### Duration syntax

Both `interval` and the bare duration argument accept compound durations:

| Example | Value |
|---------|-------|
| `45m` | 45 minutes |
| `2h` | 2 hours |
| `1h30m` | 1 hour 30 minutes |
| `270s` | 270 seconds |
| `270` | 270 seconds (bare number) |

### `/goal` shortcut

`/goal <text>` is an alias for `/keepalive message <text>`. It sets the keep-alive message, auto-enables the feature if it was off, and opens the popup. Useful when you want a more descriptive ping than the default.

---

## The popup

The popup is a **draggable** overlay that auto-positions in the top-right of the terminal. Drag its title bar to move it anywhere.

```
┌──────────────────────────────────────────┐
│ Keep-Alive (active)                   [X] │
├──────────────────────────────────────────┤
│ Interval:       4m 30s                   │
│ Next ping:      2m 15s                   │
│ Pings:          3 / 12                   │
│ Hits: 3       Misses: 0                  │
│ Max tries:      1 (streak: 0)           │
│ Message:        keepalive, respond ok    │
└──────────────────────────────────────────┘
```

| Row | What it shows |
|-----|---------------|
| **Title** | Current state in the title bar, colour-coded: green for active, purple for pinging, orange for paused, dim for idle/disabled. The `[X]` button closes the popup. |
| **Interval** | The idle duration before the next ping (from `CACHE_KEEPALIVE_TRIGGER` or `/keepalive interval`). |
| **Next ping** | Count-down to the next ping. Shows `sending...` while a ping is in flight, `--` when not active. |
| **Pings** | Total pings sent this session, with the cap (from `CACHE_KEEPALIVE_MAX_PINGS`). |
| **Hits / Misses** | Pings that got a cache-read hit vs. pings that missed. |
| **Max tries** | Consecutive-miss threshold and current streak. |
| **Message** | The ping text (truncated to fit the box). |

### Status bar indicator

The bottom status bar shows a compact indicator so you can glance without opening the popup:

| Indicator | Colour | State |
|-----------|--------|-------|
| `KA` | green | Active |
| `KA:<n>` | green | Active, *n* pings sent |
| `KA*` | yellow | Pinging (in flight) |
| `KA\|\|` | orange | Paused |
| `KA-` | dim | Idle (armed, waiting for cache) |
| *(blank)* | — | Disabled |

---

## Configuration (`scorpiox-env.txt`)

All keep-alive settings live under the **Cache Keep-Alive Settings** section of `scorpiox-env.txt`. Runtime slash commands override these for the current session.

| Key | Default | Description |
|-----|---------|-------------|
| `CACHE_KEEPALIVE` | `0` | Master switch. `0` = disabled, `1` = enabled at startup. Use `/keepalive` to toggle at runtime. |
| `CACHE_KEEPALIVE_TRIGGER` | `270` | Seconds of idle time before the ping fires. The prompt cache TTL is 300 s, so 270 s leaves a 30 s safety margin. |
| `CACHE_KEEPALIVE_MAX_TRIES` | `1` | Consecutive cache misses before keep-alive pauses. Default 1 means "pause after the first miss — the cache is already gone." |
| `CACHE_KEEPALIVE_MAX_PINGS` | `2` | Maximum total pings before auto-stop. `0` = unlimited. Default 2 is conservative; raise for longer sessions. |
| `CACHE_KEEPALIVE_MESSAGE` | *(empty)* | The ping text sent through the agent path. Empty uses the built-in default `keepalive, respond ok`. Override at runtime with `/keepalive message <text>` or `/goal <text>`. |

### Example

```ini
# Keep the prompt cache alive for up to 1 hour with a 5-minute interval
CACHE_KEEPALIVE=1
CACHE_KEEPALIVE_TRIGGER=270
CACHE_KEEPALIVE_MAX_TRIES=1
CACHE_KEEPALIVE_MAX_PINGS=12
CACHE_KEEPALIVE_MESSAGE=keepalive, respond ok
```

---

## A typical session

1. **Start working.** You type a task; the agent responds. The provider caches the prompt.
2. **Idle.** You step away. After 270 s of no API activity, the timer fires.
3. **Ping.** SCORPIOX CODE sends `keepalive, respond ok` through the normal agent path. The provider returns a `cache_read` hit, confirming the cache is still warm. The timer resets.
4. **Repeat.** Every 4:30 the cycle repeats until you come back, or `CACHE_KEEPALIVE_MAX_PINGS` is reached, or a ping misses (cache already gone).
5. **Return.** You type your next message. Because the cache survived, the turn is fast and cheap — the provider only processes the new delta.

---

## Gotchas

- **Disabled by default.** `CACHE_KEEPALIVE` is `0` in the shipped config. You must set it to `1` or run `/keepalive on` to activate it.

- **It only works if there is a cache to keep alive.** In the **idle** state, keep-alive waits for the first response that contains cache tokens before it arms. On a brand-new session with zero messages, there is nothing to keep alive yet.

- **One miss pauses it.** With the default `CACHE_KEEPALIVE_MAX_TRIES=1`, a single cache miss stops the timer. This is intentional: if the cache is already gone, further pings would just create a new one (wasting tokens) without guaranteeing the next real turn hits it. Resume with `/keepalive on`.

- **`disable` is stronger than `off`.** `off` pauses the timer (state becomes **paused**); `disable` stops the background thread entirely (state becomes **disabled**). Use `off` if you might resume later in the same session.

- **The ping costs tokens.** Each ping is a real agent turn. At the default 270 s interval with a 1-hour window that is roughly 12 pings. Keep `CACHE_KEEPALIVE_MAX_PINGS` proportional to how long you actually expect to be away.

- **Duration commands override the max-pings cap.** `/keepalive 2h` recalculates the ping budget from the interval and the total duration, resetting the counter. Use it when you know exactly how long you will be away.

- **State is per-session.** Restarting SCORPIOX CODE resets the keep-alive context. Config in `scorpiox-env.txt` persists; the runtime state (pings sent, current streak) does not.

- **The message should be short and self-contained.** It is sent as a real user turn. Keep it under ~256 characters and avoid anything that would cause the agent to take a long action — you just want a fast ack to refresh the cache fingerprint.

---

Docs for SCORPIOX CODE @ `b59223a`.
