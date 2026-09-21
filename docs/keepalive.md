# Using the /keepalive Command

Claude's prompt cache gives SCORPIOX CODE a 5-minute window where the conversation context stays warm on the provider side. After that window, the cache expires and your next message pays full input tokens again — slower, more expensive, and it interrupts any in-flight reasoning that depends on the warm prefix.

The **cache keep-alive** feature solves this. When you are stepping away from the terminal but do not want to lose the cache, SCORPIOX CODE automatically sends a lightweight ping message at a configurable interval before the TTL runs out. Because the ping goes through the normal agent path with the exact same provider settings, tools, and thinking budget, the cache fingerprint matches perfectly and the response is a guaranteed cache read. Your context stays warm, the cache clock resets, and when you come back the agent is ready to pick up where you left off — no cold start, no re-embedding, no extra cost.

Source of truth: `sx_cache_keepalive.c`, `sxui_keepalive.c`, `sx_slashcmd.c`, and `scorpiox-env.txt` at commit `24427d8`.

---

## What keep-alive does, in one paragraph

A background timer watches the gap since the last API response. Once that gap reaches the trigger threshold (default 270 seconds — 4 minutes 30 seconds into the 5-minute TTL, leaving a 30-second safety margin), SCORPIOX CODE types a short message into the conversation and dispatches it through the normal agent pipeline. The response comes back with `cache_read_tokens > 0`, the timer resets, and the cycle repeats until the total keep-alive duration elapses or a miss streak triggers a pause.

The keep-alive thread never touches the provider, history, or conversation state directly. It is purely a timer that sets a flag; the main loop performs the actual send.

---

## States

Keep-alive moves through five states. The popup and status bar reflect the current state at all times.

| State | Meaning |
|-------|---------|
| **disabled** | Feature is off (config `CACHE_KEEPALIVE=0` or explicitly disabled at runtime). |
| **idle** | Armed and waiting. No cache exists yet — the feature auto-activates the moment the first cached response arrives. |
| **active** | Monitoring the timer. A ping will be dispatched once the trigger threshold is reached. |
| **pinging** | The main loop is in the middle of sending the keep-alive message. |
| **paused** | Consecutive cache misses hit the `MAX_TRIES` limit. The cache is gone; keep-alive stops to avoid wasted pings. Use `/keepalive on` to resume. |

---

## The /keepalive slash command

Type `/keepalive` in the chat input to open the keep-alive popup. With no arguments it simply toggles the popup window on and off.

### Quick reference

```
/keepalive                          Toggle popup visibility
/keepalive on                       Enable (or resume) keep-alive
/keepalive off                      Disable keep-alive
/keepalive <duration>               Set total keep-alive duration and arm
/keepalive interval <duration>      Change the ping interval
/keepalive message <text>           Set the ping message
/keepalive show                     Show the popup
/keepalive hide                     Hide the popup
```

`<duration>` accepts compound time values:

| Example | Meaning |
|---------|---------|
| `45m` | 45 minutes |
| `2h` | 2 hours |
| `1h30m` | 1 hour 30 minutes |
| `270s` | 270 seconds |
| `270` | 270 (raw seconds) |

### Typical session

```text
> /keepalive 1h30m
Keep-alive for 1h 30m (18 pings every 270s). Armed - starts after your next message creates a cache.

> (send any normal message; the response creates the cache)

> /keepalive show
(popup appears in the top-right corner)
```

The "Armed" suffix appears when keep-alive is in the **idle** state. It means the feature is on but has not yet seen a cached response, so the first real message you send will activate the timer.

### The /goal shortcut

`/goal <text>` is a shorthand for `/keepalive message <text>`. It sets the ping message and auto-enables keep-alive if it was disabled. It also shows the popup.

```text
> /goal check build status
Keep-alive goal: "check build status"
Cache keep-alive auto-enabled.
```

---

## The popup UI

The keep-alive popup is a small draggable window that renders in the top-right corner of the terminal by default. You can drag it anywhere with the mouse. Click the `[X]` button in the title bar, or type `/keepalive hide`, to dismiss it.

### Layout

```
┌─ Keep-Alive (active)              [X] ─┐
│ Interval:  4m 30s                     │
│ Next ping: 3m 12s                     │
│ Pings:     3 / 18                     │
│ Hits: 3        Misses: 0              │
│ Max tries: 1 (streak: 0)             │
│ Message:  keepalive, respond ok       │
└───────────────────────────────────────┘
```

| Row | What it shows |
|-----|---------------|
| **Title** | Current state in color: green = active, purple = pinging, amber = paused, grey = disabled/idle. |
| **Interval** | The configured ping interval (`trigger_sec`). |
| **Next ping** | Countdown to the next scheduled ping. Shows "sending..." while a ping is in flight. |
| **Pings** | Total pings sent so far, divided by the max allowed (if `MAX_PINGS > 0`). |
| **Hits / Misses** | Cache hits (green) and cache misses (red) across all pings. |
| **Max tries** | The miss-streak threshold and the current consecutive-miss streak. |
| **Message** | The text that will be sent as the next ping. |

### Status bar indicator

Even with the popup hidden, the status bar shows a small keep-alive indicator with the current state and total ping count, so you can glance at it without opening the popup.

---

## Configuration keys (scorpiox-env.txt)

All keep-alive settings live under the **Cache Keep-Alive Settings** section of `scorpiox-env.txt`. Every key can also be changed at runtime via the slash command.

| Key | Default | Description |
|-----|---------|-------------|
| `CACHE_KEEPALIVE` | `0` | Master enable. `0` = off, `1` = on. Can be overridden at runtime with `/keepalive on` or `/keepalive off`. |
| `CACHE_KEEPALIVE_TRIGGER` | `270` | Seconds of idle time before a ping is dispatched. Default 270 (4:30) leaves a 30-second safety margin before the 5:00 TTL expires. |
| `CACHE_KEEPALIVE_MAX_TRIES` | `1` | Maximum consecutive cache misses before the feature pauses. Default 1 means a single miss stops the pings (the cache is already gone). |
| `CACHE_KEEPALIVE_MAX_PINGS` | `2` | Maximum total pings before auto-stop. `0` means unlimited. Default 2 gives roughly 9 minutes of coverage. |
| `CACHE_KEEPALIVE_MESSAGE` | *(empty)* | Custom ping message. Empty uses the built-in default: `keepalive, respond ok`. Can be changed at runtime with `/keepalive message <text>` or `/goal <text>`. |

### Tuning the interval

The trigger value should be less than the provider's cache TTL (5 minutes for Claude). A shorter trigger means pings fire earlier, giving more margin but consuming more tokens per hour. A longer trigger saves tokens but leaves less room for network latency or processing delays.

The default of 270 seconds is a safe balance. If your provider or network is particularly fast, you can push it to 280 or 285 seconds. If you are on a high-latency connection, pull it back to 240.

```text
> /keepalive interval 4m
Keep-alive interval set to 240s.
```

### Tuning the total duration

When you specify a total duration (e.g. `/keepalive 2h`), SCORPIOX CODE calculates how many pings that implies at the current interval and sets the max-pings limit accordingly. Once that many pings have been sent, keep-alive auto-stops.

```text
> /keepalive 1h
Keep-alive for 1h (12 pings every 270s). Armed - starts after your next message creates a cache.
```

### Tuning the message

The ping message is sent as a normal user message. It should be short and unambiguous so the agent responds with a minimal turn. The default `keepalive, respond ok` is fine for most cases. If you want the agent to do something slightly useful during the ping (e.g. re-check a variable), you can set a custom message:

```text
> /keepalive message "ping: confirm cache is warm"
Keep-alive message: "ping: confirm cache is warm"
```

> **Keep it short.** The message is appended to the conversation history. A long or complex message increases the token cost of every ping and may cause the agent to perform work you did not intend.

---

## How it fits with the rest of the system

- **No separate API call.** The ping goes through the exact same agent thread as a user-typed message. Same provider, same tools, same thinking budget, same `max_tokens`. There is no special "keep-alive endpoint" or reduced-cost path — it is a real message, and the cache hit is what makes it cheap.

- **Interrupts nothing.** The ping is only dispatched when the agent is idle (no active run in progress). If the agent is mid-turn, the ping is held until the agent finishes.

- **Miss detection is automatic.** If a ping comes back with `cache_read_tokens == 0`, the cache is gone. SCORPIOX CODE counts the miss, and if the streak hits `MAX_TRIES`, it pauses. This prevents a cascade of wasted pings against an already-expired cache.

- **Resets on new conversation.** Starting a new session (or using `/reset`) clears the keep-alive state. The feature re-arms to idle and waits for the first cached response.

---

## Gotchas

- **Keep-alive is disabled by default.** You must either set `CACHE_KEEPALIVE=1` in `scorpiox-env.txt` or type `/keepalive on` in a session before pings will fire.

- **"Armed" does not mean "running."** In the idle state the feature is on but the timer has not started yet. It activates the moment a cached response is observed. If you see "Armed" in the status message, just send your next normal message — the timer kicks in automatically.

- **One miss = paused by default.** `MAX_TRIES` defaults to 1, so a single cache miss pauses the feature. This is intentional: if the cache is gone, there is nothing left to keep alive. To make it more tolerant, raise `CACHE_KEEPALIVE_MAX_TRIES` or type `/keepalive on` to resume.

- **The ping is a real turn.** The agent sees and responds to the keep-alive message. If the agent is in the middle of a multi-step task, the ping will not interrupt it (pings are held while the agent is busy), but it will consume the next idle moment.

- **Max pings is a safety valve, not a timer.** When you specify a total duration like `/keepalive 45m`, SCORPIOX CODE sets the max-pings count to roughly `duration / interval`. It does not track wall-clock time independently — it counts pings. If the agent is busy for a while, pings are delayed, and the total wall-clock coverage may exceed the nominal duration.

- **WASM builds do not support keep-alive.** The browser/WASM build has no background threads, so the keep-alive timer is a no-op. The slash command and popup still work, but no pings are dispatched.
