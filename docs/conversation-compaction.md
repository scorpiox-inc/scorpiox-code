# Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture

Agents that run for hours across hundreds of turns run into a hard wall: the context window. Every message, tool call, and result the agent has produced stays resident in the model's working set, and once you hit the limit you have to decide what to do with all of that history.

Most agent harnesses solve this the same way: they **summarize** the conversation into a shorter text and drop the original. SCORPIOX CODE solves it differently — it treats the conversation as **data that lives on your filesystem**, and lets the agent go back and read it whenever it wants.

This page explains the long-horizon problem, the two families of solutions, and how SCORPIOX CODE's filesystem-native sessions (`/compact` and `/resume`) work in practice.

---

## The fundamental challenge: long-horizon agent tasks

A "long-horizon" task is one that does not fit in a single context window. Migrations across many files, large refactors, multi-stage builds and deploys, research that reads dozens of sources, or an agent that polls a job and keeps working overnight — all of these cross hundreds of turns.

Every turn adds tokens to the context window:

- The user's messages.
- The agent's responses.
- Every tool call the agent made and its arguments.
- Every tool result the agent read back (file contents, command output, HTTP responses).

Tool results are the biggest contributors. A single `grep` over a repo or a `cat` of a generated file can push in tens of thousands of tokens. After a few hundred turns the window is full, and the model can no longer see its earlier work. At that point the harness must either stop, or it must shrink what it is sending.

The question is *how* it shrinks.

---

## The mainstream answer: lossy in-memory summaries

Nearly every mainstream harness — OpenCode, Codex, Claude Code, and the open agent projects like Pi and Hermes — handles the wall the same way. When the context window nears its limit, they call the model (or a cheaper "summarizer" model) and ask it to **compress the whole conversation into a summary**. That summary is then injected back into the context as if it were a new message, and the original messages are discarded from the window.

The shape is the same everywhere. OpenCode, for example, runs a dedicated summarizer prompt against the message list, takes the returned text, stores it as a `SummaryMessageID`, and from that point only the messages *after* the summary marker are re-sent. The old messages are still on disk, but the model is no longer shown them.

The problem with this approach is that a summary is **lossy and irreversible**:

- **Exact content is lost.** The summary says "edited the parser to handle edge cases." It does not preserve the actual diff, the exact line numbers, or the failing test output. When the agent needs those specifics later, they are gone.
- **Commands and paths are paraphrased.** The precise command that was run, the exact flag, the temporary path — these get generalized into prose.
- **Nuance collapses.** Decisions, "we tried X and it failed because Y" reasoning, and the *why* behind choices are the first things a model drops when told to be concise.
- **It is one-directional.** Once the original messages are out of the window, the agent cannot recover them by asking. It has to work from the summary.

For short tasks this is fine. For a long-horizon task where, two hundred turns later, the agent needs the exact error message it saw at turn forty, the summary has already forgotten it.

---

## The SCORPIOX CODE answer: filesystem-native sessions

SCORPIOX CODE takes a different position. A session is not a blob of in-memory text that you occasionally compress. **A session is a persistent folder on your filesystem**, and the full, verbatim record of everything that happened lives in that folder.

Every session gets its own directory under `.scorpiox/sessions/<session-name>/`. The session name is human-readable and dated (for example `2026_09_26_quiet_lantern`), so you can tell sessions apart at a glance.

When the context window fills up, SCORPIOX CODE does not summarize and discard. It **closes the current session — preserving everything — and starts a fresh one with an empty context**. The old session is still sitting on disk, in full, and the agent is told exactly where to find it.

| | Lossy in-memory summary | Filesystem-native session |
|---|---|---|
| **What happens at the limit** | Conversation is compressed into a summary; originals dropped from context | Session is sealed to disk; a fresh, empty session starts |
| **Original messages** | Lost from context, paraphrased into prose | Preserved **verbatim** on disk |
| **Exact code, commands, errors** | Paraphrased or dropped | Preserved exactly, readable on demand |
| **How the agent gets history back** | It cannot — the summary is all it has | It reads the old session files with its normal tools |
| **Token cost of a fresh session** | Carries the summary blob | **Zero** — starts clean |
| **Recoverability** | One-way; detail is permanently gone | Two-way; any session can be re-opened anytime |

The key insight: **the model's context window is not the archive.** The archive is the disk. The context window is just the working set you are currently looking at, and the filesystem is where the complete, lossless record lives.

---

## What a session folder actually contains

Each session is a self-contained folder. The full conversation transcript, every tool call and result, the raw network traffic, logs, and a small machine-readable metadata file all live side by side:

| Path in the session folder | What it holds |
|---|---|
| `conversation.json` | The **complete, verbatim** conversation — every user message, assistant response, tool call, and tool result, in order. Nothing is summarized. |
| `traffic/` | Raw HTTP request/response data for the provider calls, for when you need to inspect exactly what went over the wire. |
| `events.jsonl` | A structured event stream (one JSON line per event) for tooling and external consumers. |
| `session.log`, `agent.log` | Human-readable logs for the session and the agent loop. |
| `trace.jsonl` | Step-by-step trace of the agent's execution. |
| `meta.json` | Session metadata — start time, working directory, model, and provider — written at session start. |
| `stats.json` | Live telemetry (token usage, timing), updated as the session runs. |
| `required_skills.txt` | Skills the session needed, carried forward automatically when you compact or resume. |
| `config-snapshot.txt` | The active configuration at the time, so a session is reproducible. |

None of this is a summary. `conversation.json` is the conversation, in full. That is what makes the rest of this design possible: because the record is complete and on disk, the agent can go back to it with the same tools it uses on any other file.

> **Sessions are git-ignored by default.** SCORPIOX CODE adds `.scorpiox/sessions/` to `.gitignore` when applicable, so your session history is local by default and does not get swept into a commit.

---

## The agent decides, and explores freely

This is the part that separates the two approaches in everyday use.

When a session is compacted (manually or automatically), the fresh session starts with **zero token bloat** — no summary blob, no carried-over baggage. The only thing injected is a short continuation note telling the agent where the previous session's files are and asking it to plan: what was being worked on, what is done, and what remains.

From there, the agent is a free agent. If it needs a detail from earlier, it does not have to ask you and hope you remember. It uses the tools it already has — `grep`, `read`, `bash` — on the old session folder. It can:

- `grep` the old `conversation.json` for an exact error message, a variable name, or a decision it made.
- Read the last ~20 messages of the previous conversation to get up to speed on the most recent work.
- Inspect `traffic/` to see a raw response it was uncertain about.
- Skim `events.jsonl` to reconstruct the sequence of tool calls.

The context window stays lean because the agent only pulls in the specific historical detail it needs for the step it is on, and nothing more. The full history is always one file-read away, but it does not have to be *in* the window.

---

## `/compact` — swap into a fresh session

`/compact` seals the current session to disk and starts a new one. It is the manual version of what happens automatically when the context limit is reached.

```
/compact
```

What happens:

1. The current conversation is saved in full to `.scorpiox/sessions/<old-session>/conversation.json`.
2. A `session_compact` event is recorded and the old session is closed.
3. A new, empty session is created.
4. The `required_skills.txt` from the old session is carried into the new one.
5. A continuation instruction is sent to the agent: it is told where the old session lives, asked to read `conversation.json` (focusing on the last ~20 messages), to enter plan mode, write out what is done and what remains, and then continue.

You will see a confirmation like `compact: 2026_09_26_quiet_lantern -> 2026_09_26_brave_compass` in the chat. The old session is untouched on disk; the new one is where you are now.

### Automatic compaction

Compaction is also automatic. By default, SCORPIOX CODE watches the effective context size (the larger of the cached-read tokens and the input tokens) and, when it reaches the threshold, it offers to compact into a fresh session. The defaults at this commit:

| Setting | Default | Meaning |
|---|---|---|
| `CONTEXT_AUTO_COMPACT` | `1` | Auto-compact is on. |
| `CONTEXT_CLEAR_THRESHOLD` | `190000` | Compact when effective context reaches ~190K tokens. |
| `CONTEXT_WARN_THRESHOLD` | `80` | Warn at 80% of the threshold. |
| `CONTEXT_COMPACT_PROMPT` | `1` | Ask you before compacting. |
| `CONTEXT_COMPACT_TIMEOUT` | `120` | Seconds to wait for a decision before defaulting. |

When the automatic trigger fires, you are offered a choice: **continue compaction**, **reject it** (and keep going in the current session), or **resize the context window** to a larger threshold and continue. The keys are plain config keys you set through the normal configuration cascade — see [Configuration and Profiles in SCORPIOX CODE](scorpiox-env.md).

---

## `/resume` — pick up any session from disk

Because every session is a folder on disk, you can come back to *any* of them later, not just the most recent.

```
/resume <session-name>
```

- **With a name**, SCORPIOX CODE finds the matching session and resumes it: it saves the current work, swaps to a fresh session, carries forward `required_skills.txt`, and sends the same continuation instruction as compact — point the agent at the old folder, ask it to plan, then continue.
- **Without an argument**, a **picker** pops up listing your previous sessions so you can choose one to resume.

```
/resume
```

This is what makes long-horizon work resumable across days. Walk away in the middle of a migration; come back hours or a day later and `/resume` the session you were in. The full transcript is still on disk, so the agent re-reads exactly what happened and picks up from there.

> **Compact and resume are two ends of the same mechanism.** `/compact` is "this session is full, start clean but keep everything on disk." `/resume` is "bring a session back from disk into the active view." Both rely on the same fact: the session folder *is* the record.

---

## A typical long-horizon flow

1. **Start.** A session is created, named, and its folder laid down under `.scorpiox/sessions/`.
2. **Work.** The agent runs for hours, reading files, running tools. Every turn is written to `conversation.json` as it happens.
3. **Approaching the limit.** At ~80% of the threshold, a warning appears.
4. **Hit the limit.** Auto-compact offers to continue. You accept (or run `/compact` yourself).
5. **Fresh start.** A new empty session begins. The agent reads the last ~20 messages of the sealed session, plans, and continues.
6. **Needs history?** At any point the agent `grep`s or reads the old session files for exact details, without bloating the current window.
7. **Later.** Walk away. Come back and `/resume` the session (or pick from the picker). Full fidelity, every time.

---

## Gotchas

- **The context window is the working set, not the archive.** A fresh session starts clean, but the agent can always read the sealed session on disk. If a detail seems "lost," it is almost certainly in an old session's `conversation.json`.
- **Compaction preserves everything; it does not delete.** The old session stays on disk until it is cleaned up. Sessions are local and git-ignored by default — do not assume they are backed up elsewhere if persistence across machines matters.
- **`/resume` without an argument opens the picker, not an error.** If you want a specific session, pass its name (e.g. `/resume 2026_09_26_quiet_lantern`).
- **`required_skills.txt` carries forward on both compact and resume.** Skills the session depended on are copied into the new session automatically so the agent does not lose them.
- **The continuation instruction tells the agent to plan first.** After a compact or resume the agent is asked to enter plan mode, write what is done and what remains, and then continue — so a resumed session re-grounds itself before acting.
- **The automatic threshold is large by default (~190K tokens).** If you are on a smaller context window, lower `CONTEXT_CLEAR_THRESHOLD` so compaction happens before the model runs out, rather than after.

---

Docs for SCORPIOX CODE @ `b59223a`.
