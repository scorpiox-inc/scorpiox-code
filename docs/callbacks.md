# Scheduled Callbacks and Autonomous Agent Loops

An agent that stops when you stop is only half an agent. SCORPIOX CODE can keep a session moving on its own: it schedules a **callback** — a message plus a delay — and when the delay runs out, that message is delivered back into the conversation exactly as if you had typed it. The loop wakes up, does the next unit of work, and can schedule the one after that. Nobody has to type "continue".

That single mechanism is what powers **auto-pilot** (the agent carries a multi-step job forward turn after turn), **polling** (check a status every N seconds until it changes), and **long-running work** (fire a follow-up so a background job gets collected). A callback is not a background process and not a webhook: it is a message with a start time, delivered to the same agent loop you are already talking to.

You meet callbacks from two sides:

- **The agent** manages its own timers through the built-in `SetCallback` tool — scheduling, listing, and cancelling as it works.
- **You** watch and steer them from the terminal with the `/callbacks` command and its popup, and glance at the `[CB:n]` status-bar indicator.

Docs for SCORPIOX CODE @ `0cd528b`.

> **The whole idea in one line:** a callback is a self-addressed reminder — the agent writes down what to do next and when, goes idle, and the timer delivers the note back into the conversation at the right moment, which starts a fresh turn without you.

---

## The three moving parts

| Part | What happens |
|------|--------------|
| **Schedule** | A message, a delay, and a repeat count are stored in one of eight timer slots. |
| **Fire** | When the deadline passes and the agent is idle, the stored message enters the conversation as a normal user turn. The agent responds to it with the full tool set. |
| **Repeat** | After each fire the timer re-arms with the same delay and counts one repeat down. At zero it retires itself; `-1` means forever. |

Limits up front:

| Boundary | Value |
|----------|-------|
| Active timers | **8** (slot IDs 0–7). A ninth schedule is refused. |
| Delay per fire | **1–3600 seconds** — one second to one hour. |
| Message size | Up to **2048 characters**. |
| Repeat count | Default **20**; `-1` for infinite. |
| When it can fire | Only while the agent is **idle** and callbacks are **not paused**. |

Two properties are worth internalizing before you lean on this:

- **A fire is a real turn, not a notification.** It costs a normal request, and the agent can use tools in response. Budget repeats accordingly: 20 fires at 30 seconds is roughly ten minutes of polling, not a bargain.
- **The message is the whole instruction.** Whatever the timer carries is what the agent acts on, so make each message self-contained — what to check, what to do in each outcome, and when to cancel the timer. A vague message makes a repeating timer check nothing forever.

---

## The `SetCallback` tool

`SetCallback` is a built-in tool, on by default (`TOOL_SETCALLBACK=1`). One parameter, `action`, selects what it does; the remaining parameters depend on that choice.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | yes | `set`, `list`, or `cancel`. Omitted means `set`. |
| `message` | string | for `set` | The text delivered back into the conversation when the timer fires. |
| `delay_seconds` | integer | for `set` | Seconds to wait before firing. Must be between 1 and 3600. |
| `repeat_count` | integer | no | How many times to fire before the timer retires. Default **20**; `-1` fires forever. The timer re-schedules itself with the same delay after every fire. |
| `callback_id` | integer | for `cancel` | Which slot to cancel. **Omit to cancel all.** |

### `set` — schedule a timer

```json
{
  "action": "set",
  "message": "Check the deploy job. If it succeeded or failed, report and cancel this callback. Otherwise report nothing new.",
  "delay_seconds": 120,
  "repeat_count": 30
}
```

The confirmation spells out the cadence:

```
Callback scheduled: fires every 120s, repeats 30 times.
```

With `repeat_count: -1`:

```
Callback scheduled: fires every 120s, repeats infinitely.
```

Refusals are explicit, so the agent can correct itself mid-loop:

| Situation | Message |
|-----------|---------|
| No `message` | `Error: missing message parameter` |
| Delay out of range | `Error: delay_seconds must be between 1 and 3600 (got 7200)` |
| All 8 slots taken | `Error: all callback slots full (max 8)` |
| No timer support (browser build) | `Error: callback slots not available` |

### `list` — inspect what is running

```json
{ "action": "list" }
```

```
Active callbacks (2/8):
  [0] "Check the deploy job. If it succeeded or failed, report and cancel..." - fires in 0m12s (29 left)
  [3] "Check disk usage on /tmp" - fires in 1m00s (infinite)
```

The bracketed number is the slot ID — the same `callback_id` you would pass to `cancel`. With nothing scheduled it returns `No active callbacks.`

### `cancel` — stop timers

Cancel one slot by ID:

```json
{ "action": "cancel", "callback_id": 0 }
```

```
Cancelled callback slot 0: "Check the deploy job. If it succeeded or f..."
```

Or clear everything by omitting the ID:

```json
{ "action": "cancel" }
```

```
Cancelled 2 callbacks.
```

Bad IDs fail loudly rather than silently: `Error: callback_id 5 out of range (max 7)` for a slot that does not exist, and `Error: callback slot 3 is not active` for one that already fired out or was cancelled.

---

## How firing works

The scheduler is deliberately boring, and that is what makes it safe to lean on.

**Fires happen when the loop is idle.** The timer table is checked only when no agent run is in flight. If a deadline arrives while the agent is still working, the fire is deferred about five seconds and you see `[Callback] Deferred 5s — agent busy` in the transcript. Nothing is dropped; the timer waits for a free moment. Several timers coming due at once cannot stampede either — the busy check holds them back instead of racing.

**Overdue timers re-base instead of burst-firing.** When an agent run finishes, every timer whose deadline passed during the run is pushed forward by one full delay. A 30-second timer set mid-run does not fire instantly the moment the run ends — it fires a full 30 seconds later, and the corrected countdown is written back so anything watching the session sees the real next-fire time.

**Repeats re-arm from the moment they fire.** `delay_seconds: 120` with `repeat_count: 30` is 30 fires spaced 120 seconds apart, counted from each fire. When the count reaches zero the slot retires itself — the final fire still delivers its message.

**The fired message is visible as a turn.** The transcript shows `[Callback] Timer fired`, then the message itself as the next user turn, and the agent answers it like any other prompt.

**Everything an agent run does still applies.** A callback-driven run updates the usage readouts live (so a multi-hour auto-pilot round does not freeze the token figures) and handles context compaction the same way an interactive run does — a self-running session swaps into a fresh window instead of dying on a context-limit error.

**The other auto-continue mechanisms step aside.** The pending-task reminder that normally nudges a session back on is suppressed while a callback is active — the scheduled message *is* the resume, and both nudging at once would just keep the loop from ever idling. And when the opt-in loop-detection guard is enabled (`LOOP_DETECTION=1`), the warning an agent receives for repeating an identical command explicitly points at `SetCallback` as the right alternative: schedule a check-back instead of tight-looping.

---

## Managing timers from the terminal

`/callbacks` is your control surface — the way to see and manage timers without asking the model to do it for you.

| Command | Effect |
|---------|--------|
| `/callbacks` | Toggle the popup open or closed |
| `/callbacks on` | Show the popup |
| `/callbacks off` | Hide the popup |
| `/callbacks pause` | Freeze every timer — nothing fires until you resume |
| `/callbacks resume` | Unfreeze, and re-base overdue timers so nothing fires instantly |

The four sub-arguments are offered as you type, so you never have to memorize them. A stray argument gets `Usage: /callbacks [on|off|pause|resume]`.

### The popup

The popup is a small draggable overlay — grab its title bar to move it, click `[X]` to close. One line per active timer:

```
 Callbacks (2 active)
 [0] "Check the deploy job; report s..."  0m12s (29)
 [3] "Check disk usage on /tmp"           1m00s (inf)
```

Each line shows the slot index, a truncated message preview, the time until the next fire, and the fires remaining (`inf` when the timer repeats forever). An empty table reads `No active callbacks`. Pausing turns the title into `Callbacks (2 active) PAUSED`.

> **Pausing is not cancelling.** `pause` freezes the firing clock and `resume` starts it again with the schedule intact; only `cancel` removes a timer. Pause is a switch on the running session, so it does not survive a restart.

### The status bar and stats

Even with the popup closed, the status bar shows a compact indicator whenever timers are active: `[CB:2 2:14]` means two active callbacks with the next one due in 2 minutes 14 seconds (under a minute it reads `[CB:1 45s]`). The same numbers are mirrored to the session's `stats.json` under a `callbacks` block — `active_count`, `next_fire_ms` (epoch, `0` when none), and `remaining_s` — which is what scripts and dashboards should read. See [Token Usage Observability](usage-observability.md).

---

## Where timers live

Timer state is a small JSON document in the session folder — `.scorpiox/sessions/<session>/callbacks.json`:

```json
{
  "rev": 12,
  "updated_ms": 1759712400123,
  "paused": false,
  "slots": [
    {"id":0,"active":true,"message":"Check the deploy job; report and cancel if done.","delay_ms":120000,"repeat_count":29,"fire_at_ms":1759712520000}
  ]
}
```

Every writer — the agent's tool handler, the terminal tick, the embedded engine, a remote client — reads the file, modifies it, bumps `rev`, and atomically renames it into place. Every reader polls cheaply and re-adopts when `rev` moves. Timestamps are wall-clock epoch milliseconds, so the countdown is correct across processes and machines, and a torn write is never visible.

Four consequences fall out of that:

- **Timers survive a restart.** Resume a session and its timers come back with it; any that came due while you were away fire as soon as the loop is idle. Start a brand-new session and you start with an empty table.
- **Timers ride along across session swaps.** `/compact`, `/resume`, and `/clear` all rotate the session folder, and the timer file is re-bound to the new folder with the live schedule carried over — so an auto-pilot loop keeps its timers through a compaction instead of silently losing them. This is also what keeps the remote view honest: before this re-bind, the agent kept writing the *old* session's file after a swap, and the SCORPIO BOT Callbacks panel would show `No active callbacks` for slots that were alive and ticking.
- **Other processes can drive it.** Anything that can read and write that file — a remote bot API, a host application, your own script — can list, schedule, edit, or cancel timers, and the running session picks the changes up between turns. A timer scheduled remotely fires even though no tool call ever made it. See [SCORPIO BOT](scorpiox-bot.md).
- **Worktrees stay consistent.** Inside a git worktree the file is dual-written: once under the worktree's session folder and once under the main repository's, so tooling watching the main repo still sees the schedule. A worktree `/resume` can also pull in a session that only exists in the main checkout's store, and the timer file follows it there.

The embedded-engine form factor — host applications that load the agent as a library — uses the same file. A status thread checks it once a second, and a fired timer's message is queued through the exact path a host-supplied message takes, so a host cannot tell a callback from a user's own input.

---

## Recipes

**Poll a deployment without burning tokens on a tight loop.** Have the agent start the deploy, then schedule `delay_seconds: 120, repeat_count: 30` with a message like *"Check the deploy status; if it succeeded or failed, cancel this callback and report; otherwise say nothing new."* The agent checks once every two minutes instead of hammering the endpoint.

**Watchdog a long build.** `delay_seconds: 600, repeat_count: -1` with *"Confirm the build is still progressing; if it stalled, diagnose and cancel this callback."* Cancel it when the build is done, or let it run and watch the popup.

**Auto-pilot a work queue.** One timer, `delay_seconds: 60, repeat_count: -1`, message *"Pick the next item from TODO.md, do it, mark it done. If nothing is left, cancel this callback."* The agent works one item per round and shuts itself down when the queue empties.

**A single-shot nudge.** `repeat_count: 1` turns any timer into a one-shot: fire once, deliver the message, retire. Useful for "rerun the tests in 20 minutes and tell me if they are green."

**Pair it with background tasks.** A background command runs a process and returns output later, but the agent still has to be told to look at it. A callback *is* the telling: start the long job in the background, schedule a check-back, and the loop collects the result on its own.

---

## Gotchas

- **Eight slots is a hard limit.** Scheduling a ninth fails with `all callback slots full (max 8)`. For high-frequency work, run fewer longer-lived timers rather than many overlapping ones — and cancel what you no longer need (`/callbacks` shows which slots are idle).
- **The delay is capped at one hour.** For longer waits, fire a short timer and have the agent re-schedule the next hop. That also gives it a chance to look at real state between hops instead of sleeping blind.
- **`repeat_count` counts fires, not seconds.** Default 20. `-1` is forever; `1` is a one-shot. A timer retires itself when the count runs out, and its final fire still delivers.
- **Timers fire only while idle.** Never mid-turn. A deadline that lands during a run is deferred (about five seconds) or re-based to one full delay after the run finishes, so you never get a stampede of instant fires.
- **Disabling the tool does not disarm the timers.** `TOOL_SETCALLBACK=0` or `/disable_tool SetCallback` stops new scheduling; timers already on the books keep firing until they are cancelled, repeat out, or paused.
- **The message is the whole instruction.** The agent acts on it exactly as if you had typed it. Write what to check, what to do in each outcome, and when to cancel — otherwise a repeating timer happily checks nothing forever.
- **`/clear` does not cancel timers.** Clearing the conversation rotates the session folder and carries the live schedule into it; cancel explicitly, or use `/callbacks pause` while you reorganize.
- **Pause does not survive a restart.** It is a switch on the running session. Cancelled and fired timers, however, are recorded in the session file, so what you see after resuming is what was actually left running.
- **Cancelling by ID needs the right slot.** Slot IDs come from `SetCallback` with `action: "list"` or from the popup's `[n]` prefix. Cancelling without an ID clears every timer, which is rarely what you want mid-loop.
- **Nothing fires in the browser build.** The browser target has no timer slots; `SetCallback` reports `callback slots not available` there. Native and embedded runs are unaffected.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — where `TOOL_SETCALLBACK` sits in the configuration cascade, and how to disable the tool per project or per profile.
- [Token Usage Observability](usage-observability.md) — the `[CB:n]` status-bar indicator and the `callbacks` block in `stats.json`.
- [SCORPIO BOT](scorpiox-bot.md) — listing, scheduling, and cancelling the same timers remotely over the API and web dashboard.
- [Using the /keepalive Command](keepalive.md) — the other idle-time timer: it pings to keep the prompt cache warm rather than to wake the agent.
- [Conversation Compaction](conversation-compaction.md) — how a long auto-pilot loop survives its own context growth.
- [Folder-Based Event Hooks](hooks-system.md) — for deterministic reactions to lifecycle events; timers are the right tool for delayed and recurring work instead.
