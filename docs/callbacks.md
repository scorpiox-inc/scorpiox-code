# Scheduled Callbacks and Autonomous Agent Loops

You told the agent to wait for a build to finish, or to keep checking a queue until a job shows up, and now you are staring at a terminal that has nothing to do. Or you want a long-running task to tick forward on its own — poll, retry, re-check — without you typing a nudge every few minutes. **Scheduled callbacks** in SCORPIOX CODE are that mechanism: a timer the agent sets for itself that later sends a message back into the conversation, waking the agent loop again so it can pick up where it left off. No user intervention, no lost context, no re-priming the session.

It is deliberately simple: the agent schedules a future message, the session waits, and when the time comes the message re-enters the conversation exactly as if you had typed it — and the agent responds. The same session, the same context, the same conversation. That is what turns a one-shot reply into an autonomous loop.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** the agent schedules a timed message, the session waits, and the message comes back into the conversation to re-awaken the agent — so work can poll, retry, and run ahead on its own.

---

## What it is (and what it is not)

- **A message back into the conversation, not a side channel.** When a callback fires, its text is delivered as an ordinary user message in the current session and the agent responds to it. It keeps the full conversation context — it is not a fresh session, not a webhook, not an out-of-band event you have to go read somewhere else.
- **Scheduled by the agent, for the agent.** You do not configure a list of timers by hand. The agent decides when to schedule one and what the message should say, using the built-in `SetCallback` tool. You only step in if you want to watch, pause, or cancel.
- **For work that genuinely needs to wait.** Builds, uploads, deployments, queues, rate limits, a service that is warming up. If the answer is not ready yet, the agent can stop, schedule a check-back a few minutes out, and do other useful work in the meantime instead of tight-polling.
- **Bounded and honest.** There is a hard cap on how many timers can be active at once, the delay is bounded, and every timer shows its slot, its remaining time, and how many repeats it has left, so you always know what is armed and when it will fire.
- **Pausable, and it survives the session.** Timers are persisted with the session, and you can pause them all at once and resume later without the paused ones firing in a burst the instant you resume.

---

## The `SetCallback` tool

This is the built-in tool the agent uses to manage its own timers. It has three actions: **`set`**, **`list`**, and **`cancel`**.

### Scheduling a callback (`set`)

To schedule, the agent provides:

- **`action`** — `set`. (This is also the default, so an omitted action schedules.)
- **`message`** — the text to send back into the conversation when the timer fires. This is the prompt the agent will act on. Make it specific: "check whether the build finished and report the result."
- **`delay_seconds`** — how long to wait, between **1 and 3600** seconds.
- **`repeat_count`** — how many times the callback fires before it is removed. Defaults to **20**. Use **`-1`** for infinite, in which case it keeps re-arming itself after every fire.

When it fires, the message is delivered to the conversation and the agent responds. If `repeat_count` is still greater than zero, the timer re-schedules itself with the same delay and fires again — that is how a polling loop works: schedule, wait, act, schedule again.

A session can hold up to **8 active callbacks** at once. If all slots are full, a new `set` is rejected until one frees up.

### Listing active callbacks (`list`)

`list` reports every active timer with its **slot id**, the message (truncated if long), the **time until it fires**, and how many **repeats remain** (or `infinite`). The agent uses this to keep track of what it has armed; you can ask for it at any time, and the `/callbacks` popup below shows the same state in the UI.

### Cancelling callbacks (`cancel`)

- Cancel one: pass the `callback_id` of the specific slot (the number you saw in `list` or the popup).
- Cancel all: omit `callback_id` and every active timer is removed.

---

## A typical autonomous loop

A realistic pattern, the kind the agent reaches for on its own:

1. **Do the first step.** Kick off the work — start a build, submit a job, fire the first check.
2. **If it is not ready, stop and schedule.** Instead of looping on the same command, the agent calls `SetCallback` with a sensible delay (for example 180 seconds) and a message like "re-check the build status now." It then does other useful work or simply waits.
3. **The message returns.** After the delay, the scheduled text re-enters the conversation and the agent acts on it — checking again, reporting success, or scheduling one more tick.
4. **It stops when done.** The agent either stops scheduling, or the repeat count runs out and the timer retires on its own.

This is what lets a single session babysit a long-running task: no re-typing, no context loss, no "why did it stop?" The agent that set the timer is the same agent that picks it up.

---

## The `/callbacks` command and its popup

Type **`/callbacks`** in the terminal to toggle the **Callbacks** popup over the conversation. It is a small, draggable window that shows every active timer in real time.

- **The title bar** reads the count of active callbacks, for example `Callbacks (3 active)`, and a `[X]` button closes the popup.
- **Each row** shows the slot id, a preview of the message (truncated with `...`), the remaining time until it fires (for example `2m05s`), and the repeats left (a number, or `inf` for infinite).
- **Empty state** reads `No active callbacks` when nothing is armed.
- **Paused state** shows `Callbacks (n active) PAUSED` in the title bar.

The command also takes subcommands:

- **`/callbacks`** — toggle the popup open or closed.
- **`/callbacks on`** / **`/callbacks off`** — explicitly show or hide the popup.
- **`/callbacks pause`** — pause all timers. They are frozen and will not fire while paused.
- **`/callbacks resume`** — resume them. Any timer whose deadline passed while paused is re-based to fire again after its original delay, rather than all firing at once the moment you resume.

While at least one callback is active, the status bar also shows a compact indicator such as **`[CB:2 04:10]`** — two active timers, the next one firing in four minutes ten seconds.

---

## What to keep in mind

- **Bounded, on purpose.** Up to 8 concurrent timers, delays from 1 to 3600 seconds, repeat counts up to 1000 (or `-1` for infinite). These limits keep an autonomous loop from spinning out of control.
- **It is a real wake-up, not a notification.** The scheduled message becomes a live user message that the agent acts on. If the session is busy with another run in flight, the firing is deferred briefly so the messages do not race each other, then it proceeds.
- **Watch it from the popup.** `/callbacks` is your dashboard: see what is armed, when it fires, how many repeats are left, and pause or resume the whole set without leaving the conversation.

---

*Related: the agent's polling guidance nudges it toward `SetCallback` when it detects it is tight-looping on a command whose result will not change until something external happens — one scheduled check-back beats many instant polls.*
