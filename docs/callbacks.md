# Scheduled Callbacks and Autonomous Agent Loops

You want SCORPIOX CODE to keep working after it finishes a turn — to check back on something, poll a result, or drive a long multi-step job without you typing "continue" every time. A **scheduled callback** does exactly that: it sets a timer, and when the timer fires the scheduled message is injected back into the conversation as if you had just typed it. That re-awakens the agent loop on its own, with no user in front of the terminal.

That single mechanism is what powers **auto-pilot** (the agent keeps itself going turn after turn), **polling** (check a status every N seconds until it changes), and **long-running work** (fire a follow-up a few seconds later so a background task can be picked up). A callback is not a background process or a webhook. It is a *message with a start time*, delivered to the same agent loop you are already talking to.

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** a callback schedules a message to be sent back to the agent after a delay, and the agent treats it exactly like a line you typed — so a short "keep going" note can loop the agent forward without any human input.

---

## How a callback works

There are three moving parts, all already built in:

1. **The timer.** You ask SCORPIOX CODE to schedule a message with a delay (in seconds) and a repeat count. The delay and the message are stored in a slot.
2. **The fire.** When the clock reaches the scheduled time, the stored message is injected into the conversation as a normal user message. The agent sees it, responds to it, and the loop runs — all on its own.
3. **The repeat.** After a callback fires, it re-schedules itself with the same delay and counts one repeat down. When the repeat count reaches zero, the slot is released. Set the repeat count to infinite and it fires forever until you stop it.

Because the message re-enters the loop like a typed line, the agent can use a callback to hand work back to itself. "Poll the build and report when it's done" becomes: run the build, set a 30-second callback that says "check the build again", and let the agent repeat the check until it decides the job is finished. You do not have to babysit the loop.

### Limits you should know up front

| Boundary | Value |
|----------|-------|
| Delay | **1 to 3600 seconds** (one second to one hour) |
| Active callbacks | **Up to 8 at a time** |
| Repeats | **0–1000**, or **infinite** |
| Message length | Up to 2048 characters |

If all eight slots are in use, a new schedule is rejected until one frees up. Keep your loop count small: one or two well-placed callbacks beat ten tiny ones.

---

## The SetCallback tool

The built-in **`SetCallback`** tool is how the agent (or you, when steering it) manages these timers. It takes a single `action` and a few optional fields.

### Actions

| `action` | What it does |
|----------|--------------|
| `set` | Schedule a callback (the default if you omit `action`). Requires a `message` and a `delay_seconds`. |
| `list` | Show every active callback with its slot ID, time remaining, and repeats left. |
| `cancel` | Cancel one callback by `callback_id`, or **all** of them if `callback_id` is omitted. |

### Parameters

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| `action` | string | yes | `set`, `list`, or `cancel` |
| `message` | string | for `set` | The text sent back into the conversation when the timer fires |
| `delay_seconds` | integer | for `set` | Seconds to wait, **1–3600** |
| `repeat_count` | integer | no | Fires before auto-cancel. Default **20**. Use **`-1`** for infinite. The callback re-schedules itself with the same delay each time. |
| `callback_id` | integer | no | Slot ID to cancel (from a `list`). Omit to cancel all. |

### A typical schedule

The agent calls the tool, and the tool confirms what it has registered:

```
SetCallback: "Check the deploy status and report" in 30s
```

```
Callback scheduled: fires every 30s, repeats 20 times.
```

### Listing what is active

A `list` action prints each live timer with its countdown and repeat budget:

```
Active callbacks (1/8):
  [0] "Check the deploy status and report" - fires in 0m24s (19 left)
```

The leading number in brackets is the **slot ID** — the `callback_id` you would pass to `cancel`.

### Cancelling

```
SetCallback: cancel #0
Cancelled 1 callback.
```

Or drop the ID entirely to clear every active timer at once.

---

## The /callbacks command and the popup

The `SetCallback` tool is how the *agent* manages timers. The **`/callbacks`** slash command is how *you* see and manage them from the terminal, without asking the model to do it for you.

Type `/callbacks` on its own to toggle the **callback list popup** — a small, draggable overlay that shows every active timer at a glance.

```
/callbacks
```

The popup lists each active slot with its ID, a preview of the message, the time remaining, and the repeats left. Its title shows the live count, for example `Callbacks (1 active)`, and switches to `Callbacks (1 active) PAUSED` when the timers are held. Drag the box by its title bar to move it; close it with the `[X]` or by toggling again.

### Subcommands

| Command | Effect |
|---------|--------|
| `/callbacks` | Toggle the popup visible / hidden |
| `/callbacks on` | Show the popup |
| `/callbacks off` | Hide the popup |
| `/callbacks pause` | **Pause** all timers. They hold their remaining time and do not fire. |
| `/callbacks resume` | **Resume** paused timers. Timers that would have fired while paused are re-based so they do not fire instantly. |

Pausing is the clean way to freeze an auto-pilot mid-flight — for example, to read what it has done so far, make a manual edit, and then let the loop continue from where it stopped rather than having a backlog of timers fire at once.

---

## Using callbacks in practice

### Auto-pilot a multi-step job

Tell the agent to work through a checklist and keep itself going:

```
Migrate the database, then run the full test suite, then fix any failures.
Keep going until everything is green.
```

The agent completes a step, then schedules a short callback ("continue: run the test suite") so the loop advances without you typing anything. Each callback is the nudge that carries the next step forward.

### Poll until something changes

```
Watch the CI pipeline. Check every 60 seconds and tell me the moment it
turns green or fails.
```

The agent sets a 60-second repeating callback, checks the status each fire, and cancels the callback the moment the state it is waiting for appears. The loop stops itself when the condition is met.

### Defer a follow-up

```
Run the build in the background and check on it in a couple of minutes.
```

A single one-shot callback (repeat count of 1, or the default) fires later, re-enters the conversation, and lets the agent read the result.

### Stopping it

From the terminal, `/callbacks pause` holds the loop and `/callbacks` shows you what is pending. Let the agent cancel a specific timer, or cancel everything to bring a runaway loop to a stop immediately.

---

## How this differs from a background task

A callback is not the same as a background command. A background task runs a process and returns output later; the agent still has to be told to look at it. A callback *is* the telling: it delivers a message straight back into the agent loop, which is what keeps the loop alive between your messages. The two are complementary — start a long process in the background, then use a callback to come back and collect the result.

---

## Gotchas

- **Delay is capped at one hour.** A callback of more than 3600 seconds is rejected. For longer waits, chain shorter callbacks or let the agent re-schedule itself each round.
- **Eight slots is the ceiling.** If every slot is active, a new schedule fails until one is cancelled or expires. Keep auto-pilot loops to one or two timers.
- **A repeating callback re-arms itself.** Every fire consumes one repeat and resets the same delay. With the default of 20, a 30-second poll runs about ten minutes on its own — pick the repeat count to match how long you actually want the loop to run, or use `-1` and cancel it when you are done.
- **Pause re-bases timers.** Resuming does not replay the time you paused for; held timers do not fire in a burst. That is intentional.
- **The tool and the command manage the same slots.** A `list` from the agent and the `/callbacks` popup read the identical set of timers, so what you cancel from the terminal is the same thing the agent sees.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Antigravity provider](antigravity-provider.md)
