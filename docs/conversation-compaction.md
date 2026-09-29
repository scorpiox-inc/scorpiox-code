# Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture

Agents do not finish in one turn. A real task — a failing pipeline, a cross-file refactor, a migration, a debugging hunt — runs for **hours** across **hundreds of turns**, and the context window keeps filling until the model runs out of room. Every serious agent harness has to answer the same question: *when the window is nearly full, what do you do with what you have already seen?*

Most harnesses answer it the same way — and the answer has a hidden cost. SCORPIOX CODE answers it differently. This page explains the long-horizon problem, why the common fix is lossy, and how SCORPIOX CODE's **filesystem-native session** design keeps everything verbatim on disk so the agent can always go back and read it.

Docs for SCORPIOX CODE @ `2b0bffd`.

---

## The fundamental challenge of long-horizon tasks

A single model call has a hard ceiling on how many tokens it can hold: the system prompt, your instructions, the files you have asked it to read, every tool call, and every tool result — all of it — must fit inside one context window.

Now imagine the work that actually takes an agent hours:

| Dimension | A short "demo" task | A real long-horizon task |
|-----------|--------------------|--------------------------|
| **Turns** | A handful | Hundreds, sometimes thousands |
| **Tool output** | A few small reads | Gigabytes of build logs, test suites, diffs, network traffic |
| **Working set** | One or two files | A whole subsystem, files you have not touched in 200 turns |
| **What you need to remember** | The obvious | The exact command that failed, the file you edited three hours ago, the assumption you agreed to with the user |

As the conversation grows, two things collide:

1. **The window fills up.** The model physically cannot accept more input. If nothing is done, the next turn simply fails.
2. **The signal gets buried.** Even while it still fits, the most important fact — the one precise error string, the one file path — may be 300 messages deep, past what the model weighs most heavily.

A long-horizon agent needs to be able to *compress the past without losing the past*, and to *start fresh without losing the ability to get it back*. That is the whole problem.

---

## The common fix: lossy in-memory summaries

The approach most harnesses take — **Claude Code**, **Codex**, **Pi**, **Hermes**, and **OpenCode** all use some version of it — is to **summarize the conversation into a smaller block and swap that block in for the original history**.

The idea: when the window nears its limit, ask the model to write a condensed note — "what the goal is, what's done, what's blocked, what comes next, which files matter" — and then **discard the original messages**, keeping only the note (plus, usually, the last few messages verbatim).

It works, and it is genuinely clever. But it is built on a trade-off that the design cannot escape: **the summary is irreversible and lossy.**

### What a typical summary actually keeps

The summary templates in these harnesses are strikingly similar. They are structured Markdown checkpoints:

- **Objective / Goal** — one or two sentences on what the user is trying to do.
- **Progress** — completed, in-progress, and blocked items.
- **Key decisions** — choices and their rationale.
- **Next steps** — an ordered list of what happens next.
- **Relevant files / critical context** — paths and references that matter.

The harnesses even tell the model to *"preserve exact file paths, symbols, commands, error strings, and identifiers."* That instruction is the tell: **a note is only written to preserve a handful of things it guesses are important, and everything it is not told to keep is gone.**

### What is permanently lost

Once the swap happens, the original conversation is out of the window. The model can no longer quote it, search it, or scroll to it. So the summary's guess becomes the ground truth for the rest of the run:

- **The exact command and its exact output** are reduced to a one-line bullet. The error string you were chasing is paraphrased — and a paraphrase of a compiler error is usually useless.
- **The nuance of a decision** ("we tried X, it failed because of Y, so we picked Z *and will revisit Y later*") collapses into "picked Z."
- **The tool results themselves are gone.** OpenCode, for example, truncates every retained tool output to a couple of kilobytes before summarizing; the rest is discarded.
- **What the model did not think to note is unrecoverable.** A summary can only carry what the summarizer decided mattered. There is no "let me go check what we said 200 turns ago," because there is nothing to go back to.

The result is that the *further* a long task goes, the *less* the agent can verify. Every compaction is a small, permanent amnesia. Stack several compactions over a long run and the model is working from a note about a note about the real conversation.

This is an honest design for a bounded session. It is the wrong foundation for an agent whose whole point is to run for hours and still get things exactly right.

---

## SCORPIOX CODE: filesystem-native sessions

SCORPIOX CODE inverts the premise. Instead of asking the model to compress its memory, it stops treating memory as something that lives inside the model. **A session is a folder on disk.** Everything the agent says, does, reads, and logs is written to the filesystem verbatim — and it stays there, in full, for as long as you want.

A session lives at:

```
.scorpiox/sessions/<session-name>/
```

The session name is a human-readable identifier (for example `2026_09_27_agile_wing`), so you can tell your sessions apart at a glance. Inside, the *complete* record is preserved — not a summary of it:

| File / folder | What it holds |
|---------------|---------------|
| `conversation.json` | The **full verbatim message transcript** — every user message, assistant reply, tool call, and tool result, in order, with timestamps. |
| `messages/` | The same transcript split into **one file per message**, so a single exchange can be opened, grepped, or diffed in isolation. |
| `events/` and `events.jsonl` | A structured, machine-readable event log of what happened and when. |
| `traffic/` | **Raw HTTP request and response data** — the literal bytes sent to and from the model. If you need to see exactly what was in the window, it is here. |
| `agent.log` / `session.log` | Agent-level and runtime logs. |
| `trace.jsonl` | Data-flow trace for the run. |
| `meta.json` | Session metadata: model, provider, start/end times, turn count, token totals, error count. |
| `config-snapshot.txt` | The frozen configuration the session started with, so you can always reconstruct *why* it behaved the way it did. |

Two consequences follow, and both are the point.

### The past is not compressed — it is on disk

Nothing is summarized into a lossy blob. The full transcript, the exact tool outputs, the raw traffic — all of it survives **byte-for-byte**. There is no paraphrase of a compiler error because there is no paraphrase at all: the error is still sitting in `traffic/` and `conversation.json`, exactly as it was.

### A fresh session starts at zero token bloat

When you compact (below), SCORPIOX CODE does not shrink the window by replacing history with a note. It **opens a brand-new session whose context is empty** — just the system prompt and a short pointer to the old folder. The model starts the next phase with a clean, small window and full headroom, not with a dense summary it has to reason around.

### The agent decides what to fetch, and it can always go back

This is the part the summary approach simply cannot offer. Because the old session is just files, **the agent can `grep`, `read`, and `ls` it freely** whenever it needs a historical detail. Need the exact command that failed 300 turns ago? It reads it out of the previous session. Need to check what was decided about a specific file? It opens that file. The model pulls in *precisely* what it needs, *when* it needs it, instead of pre-compressing a guess about what it might need later.

That is a fundamentally different relationship to memory: **retrieval instead of compression.** The cost model flips too — you pay tokens only for the specific facts you actually look up, not for a standing summary you never get back.

---

## How compaction works: `/compact`

`/compact` is the long-horizon control. When your context is getting heavy — or when the threshold is hit automatically — it does a **session swap**:

1. **The current session is saved to disk in full** (it already is — this just makes sure the latest messages are flushed).
2. **A new session is created.** The in-memory history and chat display are cleared; the model's window is reset to empty.
3. **The agent is handed a pointer, not a summary.** The new session's first instruction tells the agent where the old session lives and how to get back to the work:

   ```
   [CONTEXT COMPACTION - AUTOMATIC SESSION CONTINUATION]

   Your previous session was compacted due to context size limits.

   Previous session data is preserved at:
     .scorpiox/sessions/<old-session>/

   To understand what was being worked on:
   1. Read .scorpiox/sessions/<old-session>/conversation.json for full conversation history
   2. Read .scorpiox/sessions/<old-session>/traffic/ for raw HTTP request/response data
   3. Focus especially on the last ~20 messages, that is the most recent work

   Enter plan mode now. Create a detailed plan of:
   - What was being worked on
   - What has been completed
   - What remains to be done
   Then continue executing that plan.
   ```

4. The agent **re-orients from the filesystem** — reading the old transcript it needs, planning, and continuing — now running in a clean window.

So a compaction in SCORPIOX CODE is not "summarize and forget." It is **"checkpoint and re-enter."** The old session is untouched on disk; the new session is empty and fast; and the bridge between them is the filesystem, which the agent reads at will.

### Auto-compaction and the context threshold

Compaction is not only manual. SCORPIOX CODE watches the live token usage and, when the effective context size crosses a threshold, it **offers** you a choice before compacting:

- **Continue compaction** — start the fresh session with the plan, as above.
- **Reject compaction** — keep the current session as-is and continue.
- **Resize context window** — raise the threshold (for example to a larger token budget) so you can keep going without compacting at all.

The threshold is tunable. The defaults out of the box are:

| Setting | Default | Meaning |
|---------|---------|---------|
| `CONTEXT_CLEAR_THRESHOLD` | `190000` | Compact when effective context reaches ~190K tokens. |
| `CONTEXT_AUTO_COMPACT` | `1` | Auto-offer compaction when the threshold is hit (`0` to disable). |
| `CONTEXT_WARN_THRESHOLD` | `80` | Warn at 80% of the threshold before it fires. |
| `CONTEXT_COMPACT_PROMPT` | `1` | Ask before auto-compacting (`0` to compact without asking). |
| `CONTEXT_COMPACT_TIMEOUT` | `120` | Seconds to wait for your decision before defaulting to continue. |

You can also resize the window directly with `/context_resize <N>K`, which raises the threshold in place and lets a long run keep going in the same session.

---

## How resumption works: `/resume`

A session on disk is not just a backup — it is **re-attachable**. That is what makes long-horizon work and multi-tasking practical.

- **`/resume`** (no argument) opens a **picker** listing your saved sessions, newest first. Choose one and SCORPIOX CODE swaps into it the same way a compaction does: your current work is saved, and the chosen session becomes the active one.
- **`/resume <session-name>`** goes straight to a specific session by name — handy when you know exactly where you left off, or when an agent or script wants to jump back to a named checkpoint.

On resume, the agent again gets a pointer to the chosen session's folder on disk rather than a lossy summary, so it can read the full transcript it needs and continue:

```
[SESSION RESUME - CONTINUING FROM PREVIOUS SESSION]

You are resuming work from a previous session.

Previous session data is at:
  .scorpiox/sessions/<session-name>/

To understand what was being worked on:
1. Read .scorpiox/sessions/<session-name>/conversation.json for full conversation history
...
```

This is what lets one developer juggle several long-running efforts: you can leave a debugging session, start a refactor, come back an hour later with `/resume`, and pick the first one up with its entire verbatim history still intact on disk.

---

## Summary vs. filesystem sessions, side by side

| | **Lossy in-memory summary** (Claude Code, Codex, Pi, Hermes, OpenCode) | **Filesystem-native sessions** (SCORPIOX CODE) |
|---|---|---|
| **What happens at the limit** | Conversation is summarized into a structured note; the original is dropped. | A new empty session is opened; the old one is left intact on disk. |
| **The past** | A lossy, irreversible summary. Exact commands, outputs, and nuance are paraphrased or gone. | The **full verbatim** transcript, tool outputs, and raw traffic, byte-for-byte. |
| **Token cost of memory** | A standing summary you always carry, even when you don't need it. | Zero until you look — the agent reads only the specific facts it needs. |
| **Getting a detail back** | You cannot. If the summary didn't keep it, it is lost. | The agent `grep`s / `read`s the old session folder any time. |
| **Long runs** | Each compaction is a small permanent amnesia; accuracy drifts over time. | The record never degrades — retrieval stays exact no matter how far you go. |
| **Re-entering work** | Resuming reloads the (lossy) summary into the window. | `/resume [session-name]` re-attaches any session directly from disk. |
| **New phase headroom** | Starts with a dense summary to reason around. | Starts at **zero token bloat** with full window headroom. |

---

## When to use what

- **Let auto-compaction run (the default).** For a long task, let SCORPIOX CODE checkpoint at the threshold and re-enter fresh. You get clean headroom without thinking about it, and the old session is always recoverable.
- **`/compact` on purpose** when you know you are switching to a very different phase of work — a fresh window is the right move and the pointer handles the hand-off.
- **`/context_resize`** when you would rather keep everything in one session: raise the threshold and carry on instead of swapping.
- **`/resume [session-name]`** to juggle multiple efforts or to come back to a named checkpoint. The full history is waiting on disk.
- **Read the session folder yourself.** Because it is just files, you can inspect `conversation.json`, the per-message `messages/` files, and `traffic/` to audit exactly what the agent saw and did — no special tooling required.

---

## Gotchas

- **The old session is not deleted on compact or resume.** Compaction and resumption are *swaps*, not wipes: the previous session stays on disk under its own name. That is what makes recovery and audit possible — but it also means session folders accumulate. Clean up `.scorpiox/sessions/` when you no longer need a record.
- **The agent re-orients from disk, which costs a few tokens and a turn.** The first action after a compaction or resume is the agent reading the old transcript to plan. That is deliberate — it is the price of starting a phase with an empty window — and it is usually far cheaper than carrying a summary you don't need.
- **`/resume` swaps into the chosen session; your current unsaved turn is flushed first.** Save or finish a unit of work before hopping sessions so the two records stay clean.
- **The threshold is a token count, not a percentage of "the model's limit."** `CONTEXT_CLEAR_THRESHOLD` is an absolute token budget you tune to your provider and model. A larger budget defers compaction; a smaller one keeps windows lean.
- **Raw `traffic/` captures can be large.** The HTTP request/response logs are the most thorough record in the folder and the most disk-hungry. Keep them when you are debugging; prune them on sessions you have finished verifying.

---

## The bottom line

Summarizing a long conversation is the right answer to a question no one wants to keep asking: *what was important enough to remember?* Every lossy-summary harness is really pre-answering that question for you, once, irreversibly, and guessing at what matters.

SCORPIOX CODE doesn't guess. It keeps the whole conversation on the filesystem and lets the agent decide, turn by turn, exactly which past fact it needs — and read it back verbatim when it does. Long-horizon work becomes less about how much you can cram into the window and more about a model that always has the full record one `grep` away.
