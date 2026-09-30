# Sending Messages to a Running Agent

An agent that is in the middle of a turn is not frozen. You can still reach it — stop it, steer it, or leave a note for it to read when it is done. SCORPIOX CODE gives you one shared submission path for all of it, and it works the same whether you are typing at the keyboard, dropping a file on disk, calling an HTTP endpoint, or embedding the agent in your own application.

This page covers the three submit modes, exactly how each behaves in the terminal, the drop-in session inbox, remote control through SCORPIO BOT, and the public embedding API.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** every message to a running agent is classified as `auto`, `interrupt`, or `queue` — a *timing* decision, not a transport one — and that same classification runs whether the message came from your keyboard, a file, an HTTP route, or a function call in your own program.

---

## The three submit modes

Every message you send to a running agent is classified as one of three modes. The mode is not a transport choice — it is a *timing* choice: when should the agent read this.

| Mode | When the agent is idle | When the agent is busy | Use it for |
|------|------------------------|------------------------|------------|
| **`auto`** | Starts a new turn. | Behaves like **`interrupt`**. | The default. "Just send it; do the sensible thing." |
| **`interrupt`** | Starts a new turn. | Injected into the current run — the agent reads it at the next tool-call boundary, mid-turn. | Steering: "keep going, but now do X instead." |
| **`queue`** | Starts a new turn. | Stashed and held. Delivered as a fresh run once the current turn finishes. | "Don't bother it now — run this next." |

Two facts fall out of the table immediately:

- **`auto` is the workhorse.** It resolves to "new turn" when nothing is running and to "interrupt" when something is. That is exactly what pressing **Enter** does in the terminal — there is no separate auto/interrupt/queue key in the TUI, just Enter and a `~` prefix (below).
- **`interrupt` and `queue` only differ while the agent is busy.** When the agent is idle, all three modes do the same thing: they start a new turn.

`interrupt` messages *accumulate*. If you interrupt twice before the agent reaches a tool-call boundary, both notes are joined and delivered together at the next boundary. `queue` messages likewise accumulate — every queued message is joined and sent as one follow-up run when the turn ends.

---

## In the TUI

The terminal gives you the three modes with two keys and one prefix.

### Enter

Pressing **Enter** submits with `auto`. So:

- Agent **idle** → your text starts a new turn immediately.
- Agent **busy** → your text becomes an **interrupt**. The transcript echoes it as `[Interrupt] …` and the agent picks it up at the next tool-call boundary, mid-turn.

There is no "are you sure the agent is busy?" prompt. Enter-while-busy is a feature, not a mistake — it is the fastest way to redirect a running turn.

### The `~` prefix: queue it

If you type a message while the agent is busy and you want it to wait its turn rather than cut in, prefix it with a tilde:

```
~ once this test passes, update the changelog
```

The leading `~` (and any spaces after it) are stripped, and the remainder is submitted as **`queue`**. The agent keeps running its current turn, and when that turn ends, SCORPIOX CODE sends the queued text as a fresh run. You will see a `[Queued N] …` line confirming it is waiting, and a `[Sending queued message(s)]` line when it is finally dispatched.

Queue multiple notes by sending several `~`-prefixed messages in a row — they are all held and delivered together when the turn completes. Queued messages also survive a cancel: if you press **ESC** to stop the current turn, anything already queued is still delivered afterward.

### ESC: pure cancel

The **ESC** key is not a submit mode. Pressing **ESC** while the agent is running cancels the agentic loop entirely — the turn stops, and if a shell command is executing it is killed on the spot. Nothing is sent in the turn's place. Use it to *stop*, not to *say*.

| You want… | In the TUI |
|-----------|------------|
| Stop it dead, say nothing | **ESC** |
| Keep it going, but redirect | Just type and press **Enter** (interrupt) |
| Let it finish, then do this | Type `~ your note` and press **Enter** (queue) |

### Slash and bang commands share the same rules

The same submission path classifies slash commands and `!` shell lines too, and each has its own busy policy:

- **`!command` while idle** runs as a shell line in the built-in terminal. While busy, it is treated like any other text — an interrupt or a queued note.
- **UI-only commands** (`/help`, `/models`, `/usage`, `/callbacks`, `/exit` and friends) still execute while the agent is busy. They only touch the interface, so there is nothing to protect.
- **State-changing commands** (`/clear`, `/compact`, `/model`, `/resume`, `/cd` and friends) are refused while busy with a clear `Agent is busy — /x not run` message, so they cannot mutate a session out from under a running turn.
- **Custom commands and scripts** (`/my-command`, `/.sh` scripts) inject their text as an interrupt — they become part of the conversation, so a busy agent reads them at the next boundary.

---

## The drop-in session inbox

The TUI, the API, and any external program all share one other input channel: a folder on disk. Every session has an inbox directory, and the agent polls it as part of its normal loop.

```
.scorpiox/sessions/<session-id>/inbox/
    <stem>.json          ← a message the agent has not read yet
    <stem>.png           ← optional image sidecar (same stem)
    done/
        <stem>.json      ← messages the agent has already consumed
```

A message is a small JSON file. The agent looks for the **`text`** field (or **`content`**, the accepted alias) and an optional **`mode`** field:

```json
{ "mode": "queue", "text": "Once the build is green, run the full test suite." }
```

`mode` accepts `auto`, `interrupt`, or `queue`. If you omit it — or spell it wrong — it defaults to **`auto`**.

### The handshake: write `.tmp`, then rename

The inbox is a coordination point between two processes, so the delivery is made atomic. You do **not** write `<stem>.json` directly. You write the fully-formed message to a temporary name and rename it into place:

```
<stem>.json.tmp   →   <stem>.json
```

The agent only ever sees complete files, never a half-written one. Use any stem you like — a timestamp, a sequence number, a UUID — as long as it is a safe filename: letters, digits, `_`, `-`, `.` only, no `..`, no path separators, up to 64 characters. The agent sorts pending stems and processes **one per poll pass**, in name order, so the stem doubles as a priority/sort key. The poll runs continuously — every frame of the session loop, whether the agent is idle or mid-turn — so delivery latency is effectively instant, and an interrupt dropped while the agent is busy is consumed before its next tool-call boundary.

Malformed files do not wedge the channel: a JSON file that is empty, unreadable, oversized, or not valid JSON is moved straight to `done/` and skipped, and the poller moves on to the next stem.

### Images

A message can carry an image. Put an image file in the inbox with the **same stem** as its JSON and one of these extensions: `png`, `jpg`, `jpeg`, `gif`, `webp`. The agent reads it in alongside the text — a text-only drop with a sidecar works, and an image-only drop (no `text` field) works too. There is a hard cap — an image larger than 8 MB is ignored, and a JSON larger than 64 KB is skipped — so keep payloads in the message, not in the file.

### Consumption

Once the agent has read a message, it moves the file (and any image sidecar) into `inbox/done/`. The `done/` folder is your audit trail of what the agent has actually acted on. An empty `inbox/` is a no-op, so it is safe to point the poller at a session that is about to start.

### The `SCORPIOX_INBOX` gate

The whole inbox channel is behind a single config flag, **`SCORPIOX_INBOX`**, which is **on by default**. Set it to `0` to disable polling entirely. As with every other setting, the value follows the standard configuration cascade — built-in default, then global install, user, project, profile, then the real environment variable. See [Configuration and Profiles](scorpiox-env.md) for the cascade and where each file lives.

Because the inbox is just files, it is the most portable input channel: a build script, a cron job, another machine, or another agent can reach a running session by dropping one file.

### Sessions in git worktrees have two inboxes

If a session runs inside a git worktree, SCORPIOX CODE dual-writes its session state to two places: the live directory under the worktree, and a mirror under the main repository. **Both** inbox directories are created and **both** are drained, in that order, so it does not matter which of the two a writer picked — a drop into either one reaches the agent. This closes an easy-to-miss trap: a remote writer resolving a session by its ID alone used to land on the mirror, where the message sat unread while both sides believed the write had succeeded.

---

## From SCORPIO BOT

[SCORPIO BOT](scorpiox-bot.md) is the layer that drives a session without a keyboard. To send a message to a session that is already alive, you hit the same inbox over HTTP:

```
POST /inbox?id=<session-id>&mode=auto
Content-Type: text/plain

Fix the null check in auth.c and run the tests
```

The `mode` query parameter is the same `auto` / `interrupt` / `queue` set as everywhere else. The **body is plain text** — the SCORPIO BOT layer writes it into the session inbox for you, so the atomic-rename handshake is handled for you rather than something you have to do by hand. A tilde prefix on the body is read as **queue**, which is the same `~` shorthand the TUI uses.

For the *stop* case there is the escape/interrupt path: a signal that cancels the active turn, the remote equivalent of pressing **ESC**. And when you are driving the *terminal* rather than the agent — typing into a shell, sending raw keystrokes — that is the separate `POST /pty_input` route. The rule of thumb: **inbox for the agent, pty_input for the shell.**

For the exact request/response shapes, streaming endpoints, and the rest of the remote surface, see [Remote Agent Control & Fleet Management with SCORPIO BOT](scorpiox-bot.md).

---

## The embedding API

When you embed the agent in your own application through the public DLL, the same three modes are three exported functions. These are the public entry points — you call them from your host (C# P/Invoke, Swift, Java via JNI, or any FFI host) and the agent handles the rest.

| Export | What it does |
|--------|--------------|
| `sx_interrupt()` | Cancels the agent immediately — the programmatic equivalent of pressing **ESC** in the TUI. Returns `0` on success; a no-op (still `0`) if the agent is not busy. |
| `sx_interrupt_message(message)` | Injects a message into the running turn. The agent reads it at the next tool-call boundary — a steer, not a stop. Returns `-1` if the agent is not initialized or not busy. |
| `sx_enqueue(message)` | Queues a message for after the current turn — the same end-of-turn delivery the TUI's `~` prefix uses. If the agent is already idle, it is sent immediately as a new turn. Returns `0` on success. |

```csharp
[DllImport("sx.dll")] static extern int sx_interrupt();
[DllImport("sx.dll")] static extern int sx_interrupt_message(string message);
[DllImport("sx.dll")] static extern int sx_enqueue(string message);
```

`sx_interrupt()` and `sx_interrupt_message()` are the two halves of steering: the first says *stop*, the second says *stop and say this first*. `sx_enqueue()` is the "do this next" path.

Three things worth knowing when you host the agent yourself:

- **`sx_send()` is the plain channel, and it is single-slot.** While the agent is busy, a new `sx_send()` is rejected rather than buffered — that is what `sx_enqueue()` is for. `sx_is_busy()` tells you which call to make, and `sx_send_image()` follows the same rule with an image attached.
- **Interrupts accumulate under your control.** Repeated `sx_interrupt_message()` calls are joined into one injected note; when the buffer is full the call fails loudly instead of silently dropping your text.
- **The wrapper libraries mirror the exports.** The generated C# wrapper exposes `Interrupt()`, `InterruptMessage()`, and `Enqueue()`; the Swift bindings expose `interrupt()`, `interruptMessage()`, and `enqueue()`; the JNI bridge maps `sxInterrupt`, `sxInterruptMessage`, and `sxEnqueue`. Each `sx_interrupt()` produces an `info` event followed by the usual `done` event once the cancelled run unwinds, so a host loop keyed on `done` behaves the same whether a turn finished or was cancelled.

The browser/WASM build carries `sx_interrupt()` only — cancel is available, but steering and queueing are native-build features.

---

## How this compares to other harnesses

The question every agent harness has to answer is: *what happens to a message the user sends while the agent is already working?* The honest answers differ in one dimension — **where the waiting message lives.**

### OpenCode v2

OpenCode's v2 model is the most developed of the public alternatives. It separates **steer** (a note the agent should fold into the current run) from **queue** (a note to run after the current run), exposed as the `delivery` field on its "send message" API route. The wording matters: prompts are *durably admitted* — each submitted message is written into a database table (`session_input`) with an admission sequence number before any agent work happens, and delivery is the act of *promoting* an admitted row into the conversation at a turn boundary. Steering promotes all pending steers into the next request of the running loop; queueing promotes one queued message plus any pending steers when the current run ends. A per-session run coordinator serializes execution, coalesces the wake-ups that follow new admissions, and owns a separate **interrupt** call that stops the active run — bound to **Escape** in its terminal UI, with a dedicated queue-management view (`<leader>q`, "Manage queued prompts").

The consequence of putting the queue in a database: the pending state is inspectable and durable. A client can list what is waiting, the state survives process restarts, and two writers cannot race each other into double-delivery.

### SCORPIOX CODE

SCORPIOX CODE takes the opposite trade. The interrupt and queue state is **in-memory**: two small, process-local buffers — a 4 KB interrupt buffer for the next tool-call boundary and an 8 KB queue buffer for end-of-turn — drained by the agent's own loop. There is no separate coordinator process and no database. What you get in exchange:

- **No moving parts.** Delivery is the agent polling its own buffers and its own inbox directory. Nothing to deploy, nothing to keep in sync, nothing to restart.
- **A durable channel without a database.** The part that must survive across processes — "a message from another program or machine" — is the **filesystem inbox**, not a queue table. The inbox is the durable surface; the in-memory buffers are just the fast path between polls.
- **One submit path everywhere.** TUI Enter, the `~` prefix, a dropped file, the SCORPIO BOT HTTP route, and the DLL exports all funnel into the same classification. There is no per-surface queue to reason about.

| Dimension | OpenCode v2 | SCORPIOX CODE |
|-----------|-------------|---------------|
| Where waiting messages live | Durable database table (`session_input`) | In-memory buffers + filesystem inbox |
| Coordination | Per-session run coordinator with coalesced wake-ups | The agent's own loop polls buffers + inbox |
| Inspecting the pending set | Read the table / the queued-prompts view | List `inbox/` — the files *are* the queue |
| "Stop it" | Dedicated interrupt route / **Escape** keybind | **ESC** in the TUI, `sx_interrupt()` in the API |
| Durability of live notes | Survives restart (database) | Lost if the process dies; inbox files survive |
| Per-surface queues | Central queue | None — one shared submit path |

The consequence of the trade: SCORPIOX CODE's *live* interrupt/queue state is volatile — it lives as long as the process. If the process dies, in-flight notes in the buffers are lost, but anything already written to `inbox/` is not, because the filesystem is the source of truth for cross-process delivery. OpenCode's database-backed queue makes the live state durable and inspectable at the cost of a persistent store and a coordinator to manage it. If you want durability out of SCORPIOX CODE, write to the inbox — that is what it is for.

---

## Choosing a mode

- **Default to `auto`** (or just press **Enter**). It does the right thing in both states and there is almost never a reason to think harder.
- **Interrupt** when the agent is going down the wrong road and you want to redirect *now* — mid-turn, at the next tool call. In the TUI that is just Enter; in the API that is `mode=interrupt` or `sx_interrupt_message`.
- **Queue** when the current work is on track and you have a follow-up. In the TUI that is `~ note`; in the API that is `mode=queue` or `sx_enqueue`.
- **Cancel** (**ESC**, or `sx_interrupt()`) when you do not want the current turn to finish at all — it stops the turn, kills a running shell command, and sends nothing in its place.

---

## Related

- [Remote Agent Control & Fleet Management with SCORPIO BOT](scorpiox-bot.md)
- [Configuration and Profiles](scorpiox-env.md)
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
- [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md)
