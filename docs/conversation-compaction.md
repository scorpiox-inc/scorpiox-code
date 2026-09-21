# Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture

Every agent harness eventually hits the same wall: an agent that works for hours across hundreds of turns will fill its context window. How you handle that moment — what you keep, what you throw away, and where it goes — is the difference between an agent that *remembers* and an agent that *forgets*.

SCORPIOX CODE takes a different approach to long-horizon tasks than the other harnesses we've examined (OpenCode, Pi, Hermes, Claude Code, Codex). Instead of compressing your conversation into a lossy summary blob that permanently degrades, it treats the conversation as a **filesystem artifact** that a fresh session can explore at will.

Source of truth: `sx_session.h`, `sx_session.c`, `sx.c`, and `sx_agent.c` at commit `24427d8`.

---

## The fundamental problem

An agent working on a real codebase doesn't finish in twenty turns. A non-trivial task — refactoring a module, chasing a flaky integration test, implementing a feature across several files plus tests — can run for **hundreds of turns** and **hours of wall-clock time**. At some point the context window fills up. Every token of conversation, every tool result, every code snippet the agent has seen is now competing for the same finite space.

The moment the window is full, the harness has a choice:

1. **Summarize and destroy** — ask the model to compress the entire conversation into a summary, then discard the originals. The summary becomes the new conversation.
2. **Swap sessions** — keep everything on disk, start fresh, and let the agent look back whenever it needs to.

Most harnesses do option 1. SCORPIOX CODE does option 2. The difference is not subtle.

---

## How the other harnesses handle it

### OpenCode

OpenCode's `/compact` calls a dedicated **summarizer model** with a prompt along the lines of:

> *"Provide a detailed but concise summary of the conversation. Focus on: what was done, what is currently being worked on, which files are being modified, what needs to be done next."*

The resulting summary replaces the full conversation. The new session starts with that summary as its first message. The original tool outputs, exact code snippets, and command history are gone.

### Pi / Hermes / Claude Code / Codex

These harnesses use the same fundamental pattern: an **LLM-based summarization pass** over the conversation history. The model reads the full transcript and produces a condensed summary; that summary is injected as the initial context of the new session. The original transcript is either discarded or, in some cases, retained internally but never surfaced to the agent in the new session.

The common failure mode across all of these approaches:

- **Exact code is lost.** A two-hundred-line function body you showed the agent three hours ago is reduced to *"the user shared a function that handles X."*
- **Command history is lost.** The exact `sed` command or `git diff` that produced a critical result is summarized, not preserved.
- **Nuance is lost.** "Don't do that because of Y" becomes "the user preferred a different approach."
- **The summary is only as good as the model.** If the model misreads intent, the error is baked into the new session and propagates forward.

This is **irreversible, lossy compression**. Once the summary is written, the original detail is gone from the agent's working context.

---

## How SCORPIOX CODE handles it

### Sessions are folders on disk

Every SCORPIOX CODE session lives in a directory:

```
.scorpiox/sessions/<session-id>/
```

The session ID is human-readable — a date plus an adjective and a noun, e.g. `2026_09_18_misty_euler`. Everything about that session is a plain file inside the folder:

| File | Contents |
|------|----------|
| `conversation.json` | **Full verbatim transcript** — every user and assistant message, every tool call and tool result. Nothing summarized, nothing truncated. |
| `events.jsonl` | Structured, machine-parseable event log — session start/end, compaction events, tool usage, timestamps. |
| `agent.log` | Agent-level log: API requests, responses, tool results. |
| `traffic/` | Raw HTTP request/response captures (when traffic logging is enabled). |
| `config-snapshot.txt` | The frozen configuration at the moment the session started. |
| `meta.json` | Session metadata plus a summary (model, provider, duration, total turns, token counts). |
| `trace.jsonl` | Data-flow trace. |
| `stats.json` | Live telemetry stats. |
| `thinking` | Empty presence flag — its existence means the TUI spinner is on. |

Because the transcript is a real file, nothing is ever summarized away. The full fidelity of the conversation — exact commands, exact code, exact error text — stays on disk for the entire lifetime of that folder.

### A fresh session with zero token bloat

When compaction happens, SCORPIOX CODE does **not** feed a summary into the new session. It starts a brand-new session whose context window is clean — **zero tokens** of the old conversation loaded in. The old session is simply saved to disk as an archive.

The new session is handed a short instruction pointing at the archive, for example:

```
[CONTEXT COMPACTION - AUTOMATIC SESSION CONTINUATION]

Your previous session was compacted due to context size limits.

Previous session data is preserved at:
  .scorpiox/sessions/2026_09_18_misty_euler/

To understand what was being worked on:
1. Read .scorpiox/sessions/2026_09_18_misty_euler/conversation.json for full conversation history
2. Read .scorpiox/sessions/2026_09_18_misty_euler/traffic/ for raw HTTP request/response data
3. Focus especially on the last ~20 messages, that is the most recent work

Enter plan mode now. Create a detailed plan of:
- What was being worked on
- What has been completed
- What remains to be done

Then continue executing that plan.
```

The new session starts clean. But the full history is one `read` command away.

### The agent decides what to look at

This is the critical difference. In a lossy summarization approach, the *harness* decides what to keep — whatever the model deemed important during summarization. In SCORPIOX CODE, **the agent decides what to look at**, and it can look at *anything* in the old session, at any granularity, as many times as it needs.

- Need the exact error message from fifty turns ago? `grep` the `conversation.json`.
- Need to see what a specific command output? Read `agent.log` or the `traffic/` capture.
- Need to understand why a decision was made? Read the surrounding conversation in `conversation.json`.

The old session is a **read-only archive the agent can query freely**, not a compressed artifact it has to work from. The agent can also carry forward its `required_skills.txt` into the new session automatically, so its learned context survives the swap.

---

## `/compact` — swap to a fresh session

You can trigger compaction yourself at any time:

```
/compact
```

This performs the same session swap that auto-compaction performs: the current session is saved to disk, a fresh session starts, and the agent is told where to find the old data.

> **Why do this manually?** If the conversation has grown long and you want a clean break — for a different sub-task, or just to reset the context for clarity — `/compact` is your clean break. The old session is not lost; it is archived.

### Auto-compact

By default, SCORPIOX CODE auto-compacts when effective context usage reaches the threshold. The trigger compares the larger of the provider's reported cached-context tokens and input tokens against the threshold, so it adapts to how each provider reports usage. You can tune it:

| Config key | Default | Meaning |
|------------|---------|---------|
| `CONTEXT_AUTO_COMPACT` | `1` | Enable auto-compact (set `0` to disable). |
| `CONTEXT_CLEAR_THRESHOLD` | `190000` | Token count at which auto-compact triggers. |
| `CONTEXT_WARN_THRESHOLD` | `80` | Percentage of the threshold at which a warning is shown (e.g. `80` warns at 80%). |
| `CONTEXT_COMPACT_PROMPT` | `1` | If `1`, ask the user before compacting (Continue / Reject / Resize). If `0`, compact silently. |
| `CONTEXT_COMPACT_TIMEOUT` | `120` | Seconds to wait for a user response before auto-continuing. |

When auto-compact fires with `CONTEXT_COMPACT_PROMPT=1`, you get a choice:

| Option | What it does |
|--------|-------------|
| **Continue compaction** | Proceed with the session swap now (fresh session, plan from the archive). |
| **Reject compaction** | Stay in the current session. Auto-compact is suppressed for this turn. |
| **Resize context window** | Increase the threshold and continue in the current session. |

You can also resize the threshold live at any point, not just when prompted:

```
/context_resize 500K
```

This is useful when you know you are entering a long, context-heavy task and want to defer compaction. The threshold is bounded between roughly 10K and 2000K tokens.

---

## `/resume` — restore any past session

`/resume` is the counterpart to `/compact`. Where `/compact` ends the current session and starts a new one, `/resume` takes you back to a previous session.

```
/resume
```

With no argument, a picker opens listing the sessions in `.scorpiox/sessions/` (newest first), each showing its name, a preview of the first message, the model, and message count. You select one with the arrow keys and press Enter. You can also resume a specific session directly by name:

```
/resume 2026_09_18_misty_euler
```

In both cases, SCORPIOX CODE saves the current session to disk and starts a fresh session pointed at the chosen archive — the new session reads the old `conversation.json` and `traffic/` data to rebuild its understanding, then re-enters plan mode and continues.

This is useful for:

- **Picking up old work.** You worked on a bug last week, the context got compacted, and you want to go back to that conversation.
- **Branching.** You want to explore a different approach from a past checkpoint without losing the current session.
- **Audit.** You want to review exactly what the agent did in a specific session.

### The difference between resume and compact

| | `/compact` | `/resume` |
|---|-----------|-----------|
| **Direction** | Forward (current → new) | Backward (current → a past session) |
| **What happens** | Current session saved, fresh session starts with a "continue from this archive" instruction | Current session saved, fresh session starts pointed at the chosen archive |
| **Token cost** | New session starts clean (zero old context) | New session starts clean; the agent re-reads the archive on demand |
| **When to use** | Context is too long; start fresh | You want to return to a specific past conversation |

---

## Side-by-side: lossy summary vs filesystem session

| | Lossy summary (OpenCode, Pi, Hermes, Claude Code, Codex) | Filesystem session (SCORPIOX CODE) |
|---|---|---|
| **What's preserved** | A model-generated summary blob | Full verbatim transcript plus logs, traffic, and events |
| **Code snippets** | Paraphrased or dropped | Exact, on disk, `grep`-able |
| **Command history** | Summarized | Exact, in `conversation.json` and `agent.log` |
| **Nuance and intent** | Depends on model quality | Verbatim, no loss |
| **New session token cost** | Summary blob size (often 5K–20K tokens) | **Zero** — clean start |
| **Agent access to history** | Only what made it into the summary | `grep` / `read` any old session file at any time |
| **Reversibility** | No — the original is discarded | Yes — the old session is a read-only archive |
| **Going back** | Not possible | `/resume <session-name>` |

---

## Working with the sessions directory

Everything is plain files on your filesystem, so the shell is your best friend:

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

No cloud sync, no server-side copy. If you delete the folder, the session is gone. If you copy the folder, you have backed up the session. That is the whole storage model — and it is exactly why long-horizon agents built on SCORPIOX CODE remember what they actually did, instead of a model's best guess about what they did.
