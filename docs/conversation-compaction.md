# Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture

Every agent harness hits the same wall: an agent that works for hours across hundreds of turns will inevitably fill its context window. How you handle that moment — what you keep, what you throw away, and where it goes — is the difference between an agent that *remembers* and an agent that *forgets*.

SCORPIOX CODE takes a different approach to long-horizon tasks than any other harness we've examined. Instead of compressing your conversation into a lossy summary blob that permanently degrades, it treats the conversation as a **filesystem artifact** that the agent can explore at will.

Source of truth: `sx_session.h`, `sx_session.c`, `sx.c`, and `sx_agent.c` at commit `5fd054b`.

---

## The fundamental problem

An agent working on a real codebase doesn't finish in 20 turns. A non-trivial task — refactoring a module, debugging a flaky integration test, implementing a feature across multiple files with tests — can easily run for **hundreds of turns** and **hours of wall-clock time**. At some point, the context window fills up. Every token of conversation, every tool result, every code snippet you've shown the agent: all of it is now competing for the same finite space.

The moment the context window is full, the harness has a choice:

1. **Summarize and destroy** — ask the model to compress the entire conversation into a summary, then discard the originals. The summary becomes the new conversation.
2. **Swap sessions** — keep everything on disk, start fresh, and let the agent look back when it needs to.

Most harnesses do option 1. SCORPIOX CODE does option 2. The difference is not subtle.

---

## How other harnesses handle it

### OpenCode

OpenCode's `/compact` command calls a dedicated **summarizer model** with a prompt that says:

> *"Provide a detailed but concise summary of the conversation. Focus on: what was done, what is currently being worked on, which files are being modified, what needs to be done next."*

The resulting summary replaces the full conversation. The new session starts with that summary as its first message. The original tool outputs, exact code snippets, and command histories are gone.

### Pi / Hermes / Claude Code / Codex

These harnesses use the same fundamental pattern: an **LLM-based summarization pass** over the conversation history. The model reads the full transcript and produces a condensed summary. That summary is injected as the initial context of the new session. The original transcript is either discarded or, in some cases, retained internally but never surfaced to the agent in the new session.

The common failure mode across all of these approaches:

- **Exact code is lost.** A 200-line function body you showed the agent three hours ago gets reduced to *"the user shared a function that handles X."*
- **Command history is lost.** The exact `sed` command or `git diff` that produced a critical result is summarized, not preserved.
- **Nuance is lost.** "Don't do that because of Y" becomes "the user preferred a different approach."
- **The summary is only as good as the model.** If the model misreads intent, the error is baked into the new session and propagates forward.

This is **irreversible lossy compression**. Once the summary is written, the original detail is gone from the agent's working context.

---

## How SCORPIOX CODE handles it

### Sessions are folders on disk

Every SCORPIOX CODE session lives in a directory:

```
.scorpiox/sessions/<session-id>/
```

The session ID is human-readable — a date plus a fun adjective and noun, e.g. `2026_09_18_misty_euler`. Inside that folder:

| File | Contents |
|------|----------|
| `conversation.json` | **Full verbatim transcript** — every message, every tool call, every tool result. Nothing summarized, nothing truncated. |
| `events.jsonl` | Structured event log — session start/end, compaction events, tool usage, timestamps. |
| `agent.log` | Agent-level logging — API requests, responses, token counts. |
| `session.log` | General sx_log output (DEBUG/INFO/ERROR). |
| `trace.jsonl` | Data-flow trace of agent operations. |
| `config-snapshot.txt` | Frozen copy of the active configuration at session start. |
| `traffic/` | Raw HTTP request/response captures (when traffic capture is enabled). |
| `messages/` | Per-message emit files (`msg_0001`, `msg_0002`, …) for SDK consumers. |
| `meta.json` | Session metadata — model, provider, start time, summary. |
| `required_skills.txt` | Skills the session depended on; carried forward on compaction. |

The key file is `conversation.json`. It is the **complete, unmodified transcript** of everything that happened in that session. No summarization pass. No LLM rewriting. The exact words the agent said, the exact commands it ran, the exact code it wrote.

### Compaction is a session swap, not a compression

When the context window reaches its threshold (default **190K tokens**), SCORPIOX CODE does not ask a model to summarize. It does a **session swap**:

1. **Saves the current conversation** to `.scorpiox/sessions/<old-id>/conversation.json` (it's already being written there continuously, but this is an explicit flush).
2. **Creates a new session** with a fresh ID.
3. **Sends an instruction** to the agent in the new session telling it where the old data lives.
4. **The agent explores the old session files** using its normal filesystem tools — `grep`, `read`, `ls` — whenever it needs historical detail.

The instruction looks like this (simplified):

```
Your previous session was compacted due to context size limits.

Previous session data is preserved at:
  .scorpiox/sessions/2026_09_18_misty_euler/

To understand what was being worked on:
1. Read .scorpiox/sessions/2026_09_18_misty_euler/conversation.json
2. Focus especially on the last ~20 messages

Enter plan mode now. Create a detailed plan of:
- What was being worked on
- What has been completed
- What remains to be done

Then continue executing that plan.
```

The new session starts with **zero token bloat** from the old conversation. The agent's context window is clean. But the full history is one `read` command away.

### The agent decides what to look at

This is the critical difference. In a lossy summarization approach, the *harness* decides what to keep (whatever the model deemed important during summarization). In SCORPIOX CODE, **the agent decides what to look at** — and it can look at *anything* in the old session, at any granularity, as many times as it needs.

- Need the exact error message from 50 turns ago? `grep` the conversation file.
- Need to see what a specific command output? Read the `agent.log` or the `traffic/` capture.
- Need to understand why a decision was made? Read the surrounding conversation in `conversation.json`.

The old session is a **read-only archive the agent can query freely**, not a compressed artifact it has to work from.

---

## `/compact` — manual session swap

You can trigger compaction yourself at any time:

```
/compact
```

This does the same session swap that auto-compaction performs. The old session is saved to disk, a new session starts, and the agent is told where to find the old data.

> **Why do this manually?** If you feel the conversation has gotten long and you want to start fresh — for a different sub-task, or just to reset the context for clarity — `/compact` is your clean break. The old session isn't lost; it's archived.

### Auto-compact

By default, SCORPIOX CODE auto-compacts when the effective context usage hits the threshold. You can tune this:

| Config key | Default | Meaning |
|------------|---------|---------|
| `CONTEXT_AUTO_COMPACT` | `1` | Enable auto-compact (set `0` to disable). |
| `CONTEXT_CLEAR_THRESHOLD` | `190000` | Token count at which auto-compact triggers. |
| `CONTEXT_WARN_THRESHOLD` | `80` | Percentage of threshold at which a warning is shown (e.g. `80` = warn at 80% of threshold). |
| `CONTEXT_COMPACT_PROMPT` | `1` | If `1`, prompt the user before compacting (Continue / Reject / Resize). If `0`, compact silently. |
| `CONTEXT_COMPACT_TIMEOUT` | `120` | Seconds to wait for user response before auto-continuing. |

When auto-compact fires with `CONTEXT_COMPACT_PROMPT=1`, you get a choice:

| Option | What it does |
|--------|-------------|
| **Continue compaction** | Proceed with the session swap now. |
| **Reject compaction** | Stay in the current session. Auto-compact is suppressed for this turn. |
| **Resize context window** | Increase the threshold (doubles it by default, capped at 2000K) and continue in the current session. |

---

## `/resume` — restore any past session

`/resume` is the counterpart to `/compact`. Where `/compact` ends the current session and starts a new one, `/resume` **restores a previous session** from disk.

```
/resume
```

With no argument, a picker popup opens showing all sessions in `.scorpiox/sessions/`. You select one, and SCORPIOX CODE:

1. Saves the current session to disk.
2. Loads the selected session's `conversation.json` back into the agent's context.
3. Restores the working state from that session.

You can also resume a specific session directly by name:

```
/resume 2026_09_18_misty_euler
```

This is useful for:

- **Picking up old work.** You worked on a bug last week, context got compacted, and you want to go back to that conversation.
- **Branching.** You want to explore a different approach from a past checkpoint without losing the current session.
- **Audit.** You want to review what the agent did in a specific session.

### The difference between resume and compact

| | `/compact` | `/resume` |
|---|-----------|-----------|
| **Direction** | Old → New (forward) | New → Old (backward) |
| **What happens to current session** | Saved to disk, agent moves to a fresh session | Saved to disk, agent loads the old session's conversation |
| **Token cost** | New session starts clean (zero old context) | Old conversation is re-loaded into context |
| **When to use** | Context is too long, start fresh | You want to return to a previous conversation |

---

## Side-by-side: lossy summary vs filesystem session

| | Lossy summary (OpenCode, Pi, Hermes, Claude Code, Codex) | Filesystem session (SCORPIOX CODE) |
|---|---|---|
| **What's preserved** | A model-generated summary blob | Full verbatim transcript + logs + traffic + events |
| **Code snippets** | Paraphrased or dropped | Exact, on disk, `grep`-able |
| **Command history** | Summarized | Exact, in `conversation.json` and `agent.log` |
| **Nuance and intent** | Depends on model quality | Verbatim, no loss |
| **New session token cost** | Summary blob size (can be 5–20K tokens) | **Zero** — clean start |
| **Agent access to history** | Only what's in the summary | `grep`/`read` any old session file at any time |
| **Reversibility** | No — original is discarded | Yes — old session is a read-only archive |
| **Going back** | Not possible | `/resume <session-name>` |
| **Cross-session continuity** | None | `required_skills.txt` carries forward automatically |
| **Storage** | In-memory only (lost on restart) | On your filesystem, persistent |

---

## What this means in practice

### Long refactoring sessions

You're refactoring a module across 300 turns. Context fills up at turn 120. SCORPIOX CODE auto-compacts. The new session starts clean. The agent reads the last ~20 messages from the old session's `conversation.json`, enters plan mode, and continues. At turn 250, it needs to check what a specific function looked like before the refactor. It `grep`s the old session file. It has the exact code, not a summary of it.

### Multi-day projects

You work on a feature on Monday, compact twice. Tuesday, you come back and run `/resume monday_session_name`. The conversation is restored. You can see exactly where you left off, what was discussed, and what the agent concluded.

### Debugging across sessions

You hit a bug in session A, start investigating in session B. The root cause turns out to be in code that was modified in session A. The agent in session B reads `.scorpiox/sessions/<session-a-id>/conversation.json` and finds the exact diff and the exact error message. No information was lost.

---

## Configuration

All compaction behavior is controlled through the standard [configuration cascade](./scorpiox-env.md):

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `CONTEXT_AUTO_COMPACT` | `0`/`1` | `1` | Enable auto-compact on context threshold. |
| `CONTEXT_CLEAR_THRESHOLD` | int (tokens) | `190000` | Token threshold that triggers auto-compact. |
| `CONTEXT_WARN_THRESHOLD` | int (percent) | `80` | Show a warning at this percentage of the threshold. |
| `CONTEXT_COMPACT_PROMPT` | `0`/`1` | `1` | Prompt user before auto-compact (1 = prompt, 0 = silent). |
| `CONTEXT_COMPACT_TIMEOUT` | int (seconds) | `120` | Timeout for compact prompt before auto-continue. |

You can also resize the threshold live during a session:

```
/context_resize 300K
```

This is useful when you know you're entering a long, context-heavy task and want to defer compaction.

---

## The sessions directory

Everything is plain files on your filesystem:

```bash
# List all sessions
ls .scorpiox/sessions/

# See what a session was about
cat .scorpiox/sessions/<session-id>/meta.json

# Read the full conversation
cat .scorpiox/sessions/<session-id>/conversation.json

# Search across all sessions for a specific error
grep -r "ECONNREFUSED" .scorpiox/sessions/

# Delete a session entirely
rm -rf .scorpiox/sessions/<session-id>
```

No cloud sync. No server-side copy. If you delete the folder, the session is gone. If you copy the folder, you've backed up the session.
