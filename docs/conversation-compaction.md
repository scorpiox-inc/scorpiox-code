# Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture

A long-horizon agent task — migrating a module across versions, refactoring a service over a weekend, driving a multi-day integration test loop — runs for **hours** across **hundreds of turns**. The model's context window is a finite buffer. No matter how clever the agent is, the transcript of everything it has read, done, and said will eventually exceed what the provider will accept in a single request.

Every agent harness has to solve this. Some of them solve it by throwing information away. SCORPIOX CODE solves it by refusing to throw anything away.

Source of truth: `sx.c`, `sx_agent.c`, `sx_session.c`, `sx_slashcmd.c`, and `sxui_resume.c` at commit `6c70ad6`.

---

## The problem: context is a finite, expensive, and lossy buffer

Three facts make long-horizon agent work hard:

1. **The window is finite.** Each provider has a hard input limit (typically 200K to 1M tokens). Once the prompt — system prompt + tools + full conversation — exceeds it, the request is rejected. There is no "just keep going."
2. **Tokens are money and time.** Re-sending the full transcript every turn multiplies cost and latency linearly with conversation length. A 300-turn session that keeps the full history in-context is paying for 300 copies of the history, forever.
3. **The agent needs the past.** A competent agent mid-task needs to remember what it already tried, which files it already changed, what error strings it has seen. If the past disappears, the agent re-explores, re-reads, re-derives, and eventually re-does work.

A naive fix — just make the window bigger — only delays the problem. A better fix has to satisfy all three: stop paying for the full history in-context, stop being rejected by the provider, and let the agent still reach the full history when it needs it.

Different harnesses satisfy these constraints in very different ways.

---

## How other harnesses do it: lossy in-memory summaries

Most popular coding agents — OpenCode, Codex, Claude Code, Pi, Hermes — converge on the same design when context gets close to the limit:

1. **Detect overflow.** Track token usage. When the count approaches the usable limit, mark the session for compaction.
2. **Call the model to summarize the conversation.** A dedicated compaction prompt is sent with the current transcript. The model returns a structured Markdown summary — objectives, work state, files touched, blockers, next steps.
3. **Replace the history with the summary.** The transcript is discarded from the in-memory prompt. The summary is injected as the new "first message" so the model can keep going.
4. **Repeat.** Every time the limit is hit again, the model is asked to *merge* the old summary with the new conversation, producing a new, still-lossy summary.

OpenCode and Codex implement this almost identically: a compaction summary part is produced by a "compaction agent" running the same model, the prior summary is discarded and re-absorbed into the next, and the previous messages are dropped from the live context. Claude Code, Pi, and Hermes follow the same summarizer-pass pattern with the same structural outcome.

The result is **lossy**. The summary is whatever the model chose to keep. Exact error strings, exact shell commands, exact diff hunks, the user's precise phrasing from turn 47 — any of these that the summarizer did not write down are **gone**. The next compaction can only summarize what the previous compaction already lost. After two or three compactions, the agent is operating on a compressed copy of a compressed copy.

That is not a bug. It is the fundamental constraint of the design: the only place the "real" conversation ever existed was in the provider's request body, and the provider did not keep it for you.

---

## How SCORPIOX CODE does it: sessions are files, and files do not lie

SCORPIOX CODE makes a different architectural choice: **a session is a folder on disk, and the conversation is just a file in that folder.**

Every session lives in a directory under `.scorpiox/sessions/`. The session ID is human-readable — a date plus a short adjective and noun, for example `2026_09_21_quiet_euler`. Inside that folder:

| File or directory | Contents |
|------|----------|
| `conversation.json` | **The full, verbatim transcript** — every user message, every assistant reply, every tool call with its complete input, every tool result with its complete output, including the model's thinking blocks. Written to disk after every turn. |
| `events.jsonl` | Structured, append-only event log — session start, session swap, compaction, resume, and task outcomes. |
| `events/` | Individual event files, numbered in sequence (`000001.json`, ...), mirroring the event log. |
| `meta.json` | Session metadata — model, provider, start time, and a summary written at session end. |
| `agent.log` | Agent-level log — the running narrative of the session. |
| `session.log` | General debug/info/error log output. |
| `trace.jsonl` | Data-flow trace of agent operations. |
| `config-snapshot.txt` | A frozen copy of the active configuration at session start, so the session is reproducible. |
| `traffic/` | Raw HTTP request/response captures (when traffic logging is enabled). |
| `messages/` | Per-message emit files (`msg_0001_info.txt`, `msg_0004_tools.txt`, ...) for SDK and tool consumers. |
| `inbox/` | Inbound message queue for the session. |
| `required_skills.txt` | Skills the session depended on; carried forward automatically on compaction. |

The key file is `conversation.json`. It is the **complete, unmodified** record of everything that happened. No summarization pass, no LLM rewriting, no truncation. The exact words the agent said, the exact commands it ran, the exact code it wrote, the exact error text the provider returned.

This single architectural choice changes everything about long-horizon tasks:

| | Lossy in-memory summary | Filesystem-native session |
|---|---|---|
| **Where the transcript lives** | Only in the provider request body | On your disk, in your repo's `.scorpiox/` folder |
| **Survives process exit** | No | Yes — forever |
| **Survives compaction** | No — replaced by a summary | Yes — the file is untouched |
| **Survives restart** | No | Yes — `/resume` reads it back |
| **Searchable** | No (it is gone) | Yes — `grep`, `jq`, or open in any editor |
| **Verifiable** | No (the model's claim about its own past) | Yes — the bytes are the bytes |
| **Auditable** | No | Yes — `events.jsonl` and `agent.log` are first-class |
| **Cost per turn** | Full transcript re-sent every turn | Only what the agent chooses to read back |

The transcript is not "a cache of the context." The transcript **is** the record, and the context window is just a working scratchpad on top of it.

---

## Compaction in SCORPIOX CODE: a session swap, not a summary

When SCORPIOX CODE's context usage crosses the threshold, it does **not** ask the model to summarize the conversation. It does something structurally different:

1. **Detect the threshold.** The agent tracks live usage. The effective token count is the *larger* of the provider's cached-read and raw input token counts, so the check adapts to how each provider reports usage. When that count reaches the threshold, a warning is raised.
2. **Prompt the user** (optional). If the prompt is enabled, SCORPIOX CODE asks what to do:
   - **Continue compaction** — start a fresh session with a plan.
   - **Reject compaction** — keep the current session as-is.
   - **Resize the context window** — raise the threshold (for example to `500K`) and keep going instead of compacting. The default resize doubles the current threshold, capped at 2000K. You can also resize at any time with `/context_resize 500K`.
3. **Save the current session to disk.** The current `conversation.json` is flushed. The session folder is now a complete, self-contained artifact.
4. **Start a fresh session.** A new session folder is created (for example `2026_09_21_quiet_euler` becomes `2026_09_21_warm_hamilton`), and the agent is pointed at the old folder:

   > "Your previous session was compacted due to context size limits.
   > Previous session data is preserved at `.scorpiox/sessions/<old>/`.
   > To understand what was being worked on: read `.scorpiox/sessions/<old>/conversation.json` for full conversation history, and `.scorpiox/sessions/<old>/traffic/` for raw request/response data. Focus especially on the last ~20 messages.
   > Enter plan mode now. Create a detailed plan of what was being worked on, what has been completed, and what remains. Then continue."

5. **The agent reads what it needs.** The fresh session starts with **zero token bloat** — the new context window is essentially empty. The agent decides, using the same shell tools it would use on any other filesystem, what to read back. Usually that is a `grep` on the old `conversation.json`, a `head` or `tail` of the last N messages, or a targeted `jq` for the file paths it already touched.

The compaction is **lossless** because nothing was summarized. The "compact" step is just *moving the working set from the in-memory prompt to the disk*, where it was already, in full, the whole time.

The key insight: **the context window is not the source of truth.** The filesystem is. The agent's job is to decide what to keep in the window, not to lose what is not.

---

## Resumption: `/resume` restores any session, directly from disk

The `/resume` command is the other half of the architecture.

- **`/resume`** (no arguments) opens a picker overlay. SCORPIOX CODE scans `.scorpiox/sessions/`, reads each session's metadata, and shows a scrollable list — session name, model, provider, first user message preview, and timestamp — newest first.
- **`/resume <session-name>`** skips the picker and resumes that session directly. The name can be an exact session ID, an exact short name, or a substring — `zen` matches `2026_09_21_zen_johnson`. Exact IDs win, then exact names, then substring matches, newest first on ties.

Resume works the same way as compact, in reverse: the current session is saved, a new session is created, and the agent is instructed to read the chosen old session from disk and continue — again entering plan mode and rebuilding its working model of the task from the file rather than from a model-generated summary. Because every session is a self-contained folder, you can resume *any* session from *any* point in time — including sessions from days ago, or a parallel session you were running in a different worktree.

This is not "continue where we left off" in the in-memory sense. It is "open a different file."

---

## The agent freely explores its own history

Because sessions are just files, the agent can do anything it can do on a filesystem. During a long task it might:

- `grep -R "TODO" .scorpiox/sessions/2026_09_18_quiet_euler/` to find work it flagged earlier and never finished.
- `jq '.messages[-20:]' .scorpiox/sessions/<old>/conversation.json` to re-read the last 20 turns of a previous run.
- `cat .scorpiox/sessions/<old>/events.jsonl | grep task_result` to see the structured outcome of an earlier task.
- `ls .scorpiox/sessions/` to see how many attempts it has made at a hard problem across sessions.

The agent is the one who decides when history matters. There is no hidden compaction layer making that decision for it, and no information is ever out of reach because it fell out of the window.

---

## Gotchas

- **Compaction is a session swap, not an in-place edit.** The old session folder is left in place; a new one is created. If you expected the context to "shrink" in the same session, that is not what happens — the session *changes*.
- **The fresh session starts with a plan, not a summary.** After `/compact`, the agent is instructed to enter plan mode and write out what was being worked on, what is complete, and what remains. That plan is its working model of the task, built from reading the old `conversation.json`, not from a model-generated summary.
- **The picker only shows sessions under the current directory's `.scorpiox/`.** If you are in a git worktree, sessions are mirrored to the main checkout's `.scorpiox/`, so the picker sees both the worktree's and the main repo's sessions.
- **`/resume` is not `/rewind`.** `/rewind` goes back to an earlier checkpoint *within the current session*. `/resume` opens a *different* session folder entirely.
- **The threshold is a soft limit, not a hard one.** The provider can still reject a request below the threshold if the model's output is unusually long. The threshold is a heuristic for "when to proactively compact," not a guarantee.
- **Sessions are your audit log.** `events.jsonl` is machine-readable. Pipe it into `jq` to get a timeline of every task result, tool call, and session transition.
- **Sessions are plain, unencrypted files on your filesystem.** If you commit them to a public repository, you are publishing the full transcript, including any secrets the agent saw. SCORPIOX CODE adds `.scorpiox/sessions/` to `.gitignore` automatically; keep it that way.

---

*Docs for SCORPIOX CODE @ 6c70ad6*
