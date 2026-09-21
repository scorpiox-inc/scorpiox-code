# Scheduled Callbacks and Autonomous Agent Loops

SCORPIOX CODE can run **autonomously**. A *scheduled callback* is a timer the agent sets for itself: after a delay, SCORPIOX CODE drops a message back into the conversation and re-awakens the agent loop **without any user typing**. That is what powers auto-pilot loops, status polling, retries, and long-running background work.

You manage callbacks in two places:

- **The agent**, through the built-in `SetCallback` tool (it schedules, lists, and cancels timers on its own).
- **You**, through the `/callbacks` slash command and its popup window (view, pause, resume, or clear the active timers).

Source of truth: `sx_tools.c`, `sx_agent.c`, `sx.c`, and `sxui_callbacks.c` at commit `24427d8`.

---

## What a callback actually does

Think of it as a self-triggering reminder. When a timer fires, SCORPIOX CODE injects the callback's `message` **as if you had typed it** and starts a fresh agent run from that point. Because the agent decides when and what to schedule, it can chain steps together indefinitely — do a unit of work, schedule a check-in 30 s later, do the next unit of work, and so on.

| Property | Value |
|----------|-------|
| **Max concurrent timers** | 8 (slots 0–7). Trying to schedule a ninth returns an error. |
| **Delay range** | 1–3600 seconds per fire. |
| **Message size** | Up to 2048 characters per callback. |
| **Repeat count** | Number of times the timer fires before it auto-cancels. Default **20**. Use **-1** to run forever. |
| **Re-schedule behavior** | After each fire the timer re-arms itself with the same delay, so it fires on a steady cadence. |
| **When it can fire** | Only while the agent is **idle** (not mid-run) and callbacks are **not paused**. |

> **One fire at a time.** The main loop processes at most one callback per frame, so multiple timers never collide in a single tick.

> **Timers never fire "late and instantly."** If a timer's deadline passes while the agent is still busy, it is re-based to fire one full delay after the agent finishes — a 30 s timer set mid-run will not burst-fire the instant you go idle.

---

## The `SetCallback` tool

This is the tool the agent itself uses. It is **enabled by default** (`TOOL_SETCALLBACK=1`). All three actions share one parameter, `action`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | yes | `set`, `list`, or `cancel`. If omitted, `set` is assumed. |
| `message` | string | for `set` | The text injected back into the conversation when the timer fires. |
| `delay_seconds` | integer | for `set` | Seconds to wait before firing. Must be 1–3600. |
| `repeat_count` | integer | no | Fires before auto-cancel. Default **20**; use **-1** for infinite. |
| `callback_id` | integer | for `cancel` | Slot to cancel. **Omit to cancel all.** |

### `set` — schedule a timer

```json
{
  "action": "set",
  "message": "Poll the deploy job; report when it finishes or fails.",
  "delay_seconds": 30,
  "repeat_count": 40
}
```

On success the tool reports, for example:

```
Callback scheduled: fires every 30s, repeats 40 times.
```

With `repeat_count: -1` it reads:

```
Callback scheduled: fires every 30s, repeats infinitely.
```

If all 8 slots are already in use you get `Error: all callback slots full (max 8)` — cancel something first (see below), then retry.

### `list` — inspect active timers

```json
{ "action": "list" }
```

Returns a summary of every active slot — its index, a message preview, time until the next fire, and repeats remaining:

```
Active callbacks (2/8):
  [0] "Poll the deploy job; report when it finishes or fails." - fires in 0m12s (38 left)
  [3] "Check disk usage on /tmp" - fires in 1m00s (infinite)
```

If nothing is active it returns `No active callbacks.`

### `cancel` — stop timers

Cancel a single slot by its id:

```json
{ "action": "cancel", "callback_id": 0 }
```

```
Cancelled callback slot 0: "Poll the deploy job; report when it finishes or fails."
```

Or **cancel everything** by omitting `callback_id`:

```json
{ "action": "cancel" }
```

```
Cancelled 2 callbacks.
```

---

## The `/callbacks` slash command

`/callbacks` is **your** control surface — a quick way to see and manage timers without asking the agent.

| Command | What it does |
|---------|--------------|
| `/callbacks` | Toggle the popup open/closed. |
| `/callbacks on` | Show the popup. |
| `/callbacks off` | Hide the popup. |
| `/callbacks pause` | Freeze all timers (they keep counting down but do **not** fire). |
| `/callbacks resume` | Unfreeze and re-base every active timer so none fire instantly. |

### The popup window

The popup is a **draggable** overlay (drag its title bar to move it). It shows one line per active timer:

```
 Callbacks (2 active)
 [0] "Poll the deploy job..."  0m12s (38)
 [3] "Check disk usage on /tmp" 1m00s (inf)
```

Each line shows the slot index, a truncated message preview, **time until the next fire**, and the **repeats remaining** (`inf` when infinite). When you pause with `/callbacks pause`, the title changes to:

```
 Callbacks (2 active) PAUSED
```

> **Pausing ≠ cancelling.** `pause`/`resume` freeze the firing clock; `cancel` (via the tool) actually removes a timer. You can pause a batch, look around, and resume with the full schedule intact.

The status bar also keeps a small live indicator showing how many callbacks are active and when the next one is due, so you can glance at it without opening the popup.

---

## A typical autonomous loop

Here is the shape of a self-running job, as the agent would drive it:

1. **Kick off the work** and immediately call `SetCallback` with `action: "set"` to schedule its own follow-up.
2. **Go idle.** The timer counts down; SCORPIOX CODE waits without user input.
3. **Timer fires.** The `message` is injected as a new turn and the agent wakes up, does the next unit of work, and schedules the *next* timer.
4. **Stop when done** — either by calling `SetCallback` with `action: "cancel"` (or by letting `repeat_count` run out), or by you pressing `/callbacks off` / `pause` from the command line.

This is how SCORPIOX CODE polls a job, retries a flaky step, or babysits a long-running build with no human in the loop.

---

## Gotchas

- **8 is a hard limit.** `SX_CALLBACK_MAX` slots, indexes 0–7. A ninth `set` fails with "all callback slots full (max 8)". For very high-frequency work, use fewer, longer-lived timers rather than many overlapping ones.

- **Delay is capped at 3600 s (1 hour).** For longer waits, fire a short timer and have the agent re-schedule the next hop — this also lets it inspect state between hops.

- **`repeat_count` is "fires remaining," not "seconds."** Default is 20. A value of `-1` means forever; `0`/`1` effectively means one shot. When the count reaches 0 the slot deactivates itself.

- **Timers only fire while idle.** A timer set during an active run is held (and re-based) until the agent finishes, so it never fires in the middle of another turn.

- **`pause` survives nothing but the process.** Paused, fired, and active state all live for the lifetime of the running session — restarting SCORPIOX CODE clears the timer table.

- **The message is what wakes the agent.** Make the `message` self-contained (what to do, what to check, how to stop), because the agent acts on it exactly as if a human had typed it.
