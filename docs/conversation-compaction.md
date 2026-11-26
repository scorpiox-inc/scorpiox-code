# Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture

Long-horizon agent tasks are the ones that matter: a migration that spans forty files, a research session that reads two hundred sources, a build that you babysit for the better part of an afternoon. The agent keeps working, and the conversation keeps growing. Every message, tool call, tool result, and line of reasoning is real context that the model has to re-read on the next turn. Given enough turns, every harness eventually hits the wall: the context window fills up.

What you do about that wall is the whole design question. Most agent harnesses answer it by **compressing the conversation into a lossy summary** and stuffing that summary back into the context. SCORPIOX CODE answers it differently: it never compresses anything away. The entire conversation is written to disk as a session folder, and when the context fills up it **opens a fresh, empty session** and lets the agent read what it needs back off the filesystem. The old session is not summarised; it is preserved, verbatim, exactly as it happened.

Docs for SCORPIOX CODE @ `3f5b938`.

> **The whole idea in one line:** other harnesses shrink your conversation into a summary and throw the detail away; SCORPIOX CODE writes the full conversation to a disk folder and, when it fills up, starts fresh and lets the agent re-read the originals — nothing is ever lost.

---

## The fundamental challenge: long-horizon tasks hit a hard ceiling

A model has a fixed context window. It is not a soft limit you can ask nicely to grow — it is the maximum number of tokens that fit in one request. When the running conversation (system prompt, every message so far, every tool result, the skills in play) crosses the threshold, the next turn cannot be sent. The task is not "slow," it is blocked.

The longer the task, the more inevitable this is. A short "rename this function" turn fits with room to spare. A task that runs for hours across hundreds of turns, each of them adding tool output to the pile, will fill the window long before the work is done. Every serious agent harness has to handle this, and the difference between them is *how*.

There are two strategies in the wild, and they are the two poles of this page:

| | Lossy in-memory summary | Filesystem-native session |
|---|---|---|
| **What is stored** | A compressed summary injected into the context | The full, verbatim conversation on disk |
| **What is kept** | Whatever the summariser thought was important | Everything — every message, tool call, tool result, log line |
| **Can you recover dropped detail** | No — it is gone from the model's view | Yes — read it back off disk, byte for byte |
| **Cost of a fresh start** | Re-reads the (lossy) summary every turn | Zero — a fresh session starts empty; the agent fetches what it needs on demand |
| **Who decides what matters** | A summary prompt, once, up front | The agent, continuously, as it works |

The second column is the design SCORPIOX CODE is built around.

---

## Lossy in-memory summaries: how the other harnesses do it

Most harnesses face the context ceiling the same way: ask the model to write a summary of the conversation, and hand that summary forward as the new context. The original messages are dropped from the active window.

This works well for short, self-contained tasks. It has a structural weakness for long-horizon work, and it is worth naming it precisely:

- **It is irreversible.** The summary is a single compressed blob. The model had to decide, in one pass, which code snippets, which exact commands, which nuances, were worth carrying forward. Whatever it did not write down is gone from the model's view. You cannot later ask it to "show me the exact `curl` command you ran in turn 47" if the summariser paraphrased it or left it out.
- **It is a snapshot, not a record.** The summary is a frozen interpretation. If the work later pivots, or a detail that seemed minor becomes critical, the summary does not update to capture it — you are stuck with the one interpretation that was produced at the moment of compaction.
- **It re-reads the summary on every subsequent turn.** The compressed blob lives in the context window, so every turn after compaction pays for it again, even for turns that have nothing to do with the compacted material.

You can see this plainly in the public harnesses. Codex's compaction prompt literally asks the model to "create a handoff summary for another LLM that will resume the task," keeping progress, key decisions, and "any critical data, examples, or references needed to continue." OpenCode keeps a summary plus a tail of recent messages. Claude Code's `/compact` produces a compaction summary that replaces the earlier conversation. These are all the same family: **a lossy distillation, injected into the window, and the original is no longer on the model's desk.**

None of this is a bug. It is the right trade for a 100-turn conversation that never leaves the window. The problem is that it is the *only* trade — and for a task measured in thousands of turns, "irreversible" is a word you want out of your vocabulary.

---

## Filesystem-native sessions: how SCORPIOX CODE does it

SCORPIOX CODE treats the conversation as a **document on the filesystem**, not as a pile of tokens in a buffer. Every session is a real folder:

```
.scorpiox/sessions/<session-name>/
```

The name is a human-friendly `yyyy_MM_dd_adjective_noun` — a date, a random adjective, and the surname of a notable scientist or inventor, the same convention container runtimes use to name their instances. For example `2026_11_24_sleepy_faraday`, or `2026_11_24_wistful_wright`. You can tell at a glance when a session ran, and its sessions are easy to tell apart in a list. Inside the folder, the conversation is stored in full:

```
.scorpiox/sessions/2026_11_24_sleepy_faraday/
  conversation.json        # the complete message transcript, verbatim
  messages/               # one file per message: user, assistant, thinking,
  |  msg_0008_user.txt             tool calls, tool results, usage, events
  |  msg_0013_assistant.txt        ... numbered in arrival order
  |  msg_0016_tool_call.txt
  |  msg_0017_tool_result.txt
  events/                 # one JSON file per event, plus events.jsonl
  traffic/                # raw HTTP request/response bodies for every provider call
  |  raw/    raw_messages/    raw_usage/    traffic.log
  session.log             # timestamped activity log
  agent.log               # agent-level log
  trace.jsonl             # tool/turn trace
  meta.json               # id, start time, cwd, model, provider, profile, lineage GUIDs
  config-snapshot.txt     # the exact configuration in effect for this session
  required_skills.txt     # the skills this session relied on (carried to the next)
  keepalive.json  commands.json  infobox.json  stats.json  inbox/
```

Three consequences follow, and each one is the point:

**The transcript is complete and verbatim.** `conversation.json` holds every message with its full content and timestamp, and `messages/` splits the same material into per-message files so any single message can be read in isolation. There is no summary standing between you and what the agent actually said, ran, or returned. If you want the exact command, the exact output, or the exact reasoning from turn 212, it is a file read away.

**The raw wire traffic is preserved too.** Every request to and from the model provider is captured under `traffic/` — request body, response, headers. This is the [API traffic logging](traffic-logging.md) story applied per session: the full remote call is on disk, not just the parts the harness chose to summarise.

**The session is durable.** It is a folder on disk, not RAM. Close the terminal, reboot the machine, come back the next day — the folder is still there, and so is the conversation.

This is what "filesystem-native" means: the conversation has the same persistence, searchability, and addressability as any other set of files you already trust.

---

## Why a fresh session costs nothing

In the summary model, the moment you compact you pay a permanent price: the summary is in the window and rides along on every future turn. SCORPIOX CODE's fresh session has no such overhead. When it starts a new session, the context begins **empty** — just the system prompt, the current configuration, and the skills in play. There is no 12,000-token summary blob being re-read on every turn.

Instead, the old session is available as a data source. The agent gets a short, precise instruction pointing it at the folder it just left:

```
[CONTEXT COMPACTION - AUTOMATIC SESSION CONTINUATION]

Your previous session was compacted due to context size limits.

Previous session data is preserved at:
  .scorpiox/sessions/2026_11_24_sleepy_faraday/

To understand what was being worked on:
1. Read .../conversation.json for full conversation history
2. Read .../traffic/ for raw HTTP request/response data
3. Focus especially on the last ~20 messages, that is the most recent work

Enter plan mode now. Create a detailed plan of:
- What was being worked on
- What has been completed
- What remains to be done

Then continue executing that plan.
```

Notice what this is *not*: it is not a summary the agent must trust. It is a **pointer to the ground truth, plus a method.** The agent reads the folder, and if its first read turns out to be the wrong slice, it reads again — it can `grep` the transcript for the exact string it needs, open the specific message file, or inspect the raw traffic. The cost of getting a detail wrong is a second read, not a lost detail.

This is the design win in a single sentence: **the agent decides what to re-read, and it can always re-read more.** In the summary model, the summariser decided, and it can never be asked for more.

---

## `/compact`: swap sessions without losing a line

When the context fills up, you (or the agent, automatically) trigger a compaction. `/compact` performs a session swap:

1. **Save the current conversation** to the session's `conversation.json`.
2. **End the current session** — it is closed out cleanly and left intact on disk.
3. **Create a fresh session** — a new `yyyy_MM_dd_adjective_noun` folder, empty context.
4. **Carry the lineage forward.** The new session inherits the old session's conversation-lineage identifier, so the work is still recognisable as the same thread even though it is now a new session. The skills the old session relied on are copied across too.
5. **Re-aim the logs.** The session log, trace, and required-skills file now point at the new folder, so the fresh session records into its own files.
6. **Send the continuation instruction** shown above, so the agent orients itself against the preserved folder and continues the work.

The net effect: your context is empty again, and your work is untouched. You did not summarise anything. You rotated to a clean session and left the full record behind, addressable, on disk.

### Automatic compaction and your three choices

You do not have to catch the ceiling yourself. When a turn lands the context at or past its threshold, the agent pauses and asks you what to do — because there are genuinely different answers to "we are out of room":

```
Context limit reached (174K / 200K tokens).
Tip: You can resize anytime via /context_resize <size>
     (e.g. /context_resize 500K or /context_resize 500000).

Please choose an action:
[1] Continue compaction (start fresh session with plan)
[2] Reject compaction (keep current session)
[3] Resize context window (e.g. 500K or 500000)
```

- **Continue compaction** — do the session swap above and carry on in a fresh session.
- **Reject compaction** — keep the current session as-is (useful if you have a way to free up context, or you are about to resize).
- **Resize context window** — grow the threshold instead of rotating. `/context_resize` accepts presets (`100K`, `180K`, `200K`, `500K`, `1000K`, `2000K`) or a raw token count, clamped between 10K and 2000K. Picking the "resize" option with no explicit size doubles the current threshold (capped at 2000K).

That is the whole philosophy in a prompt: when you hit the wall, you choose — rotate to a clean session, hold the line, or move the wall. The record is never at stake.

---

## `/resume`: pick up any session, from disk

Compaction rotates the *current* session forward. `/resume` lets you come back to **any** previous session, directly from the folder, whenever you want:

- **`/resume <session-name>`** — name a session (it is fuzzy-matched against your saved sessions) and SCORPIOX CODE swaps into a fresh session seeded with an instruction pointing at that folder. The same "read the transcript, read the traffic, focus on the last ~20 messages, enter plan mode, then continue" method applies — the agent orients itself against the preserved record and resumes the work.
- **`/resume`** (no argument) — opens a picker listing your sessions. Each entry shows what you need to choose: the session name, a preview of the first user message, when it started, how many messages it holds, and which model and provider it used.

Because every session is a folder, "resuming" is not a special, fragile operation that reconstructs state from a summary. It is: *open the folder, point the agent at it, let it read what it needs.* You can resume a session from three weeks ago, fork off an old line of work, or compare two sessions side by side — all of it is just files.

---

## Side by side

| | Lossy in-memory summary (Codex, OpenCode, Claude Code, ...) | Filesystem-native session (SCORPIOX CODE) |
|---|---|---|
| **Storage** | Summary blob in the context window | Full conversation in a disk folder |
| **Detail fidelity** | Only what the summariser kept | 100% of what happened, verbatim |
| **Recovery of dropped detail** | Not possible | Read it back off disk, on demand |
| **Fresh-start cost** | Re-reads the summary every turn | Zero — fresh session is empty |
| **Who decides what to recall** | The summariser, once | The agent, continuously |
| **Raw provider traffic** | Usually not retained | Captured under `traffic/` |
| **Durable across restart** | Depends on the harness | Yes — it is a folder |
| **Resume any past session** | Limited by what was summarised | Yes — name a folder and resume it |

The summary approach is a reasonable fit for short tasks. For long-horizon work — exactly the tasks that survive long enough to hit the context ceiling — the filesystem-native design is the one where "the agent needs to remember something from turn 90" is never a problem.

---

## Quick reference

| Task | How |
|---|---|
| See where a session lives | `.scorpiox/sessions/<yyyy_MM_dd_adjective_noun>/` |
| Read the full transcript | open `conversation.json` in the session folder |
| Read one message | open its file under `messages/` |
| Read raw provider traffic | open the files under `traffic/` |
| Rotate to a fresh session | `/compact` |
| Grow the window instead | `/context_resize 500K` (10K–2000K) |
| Resume a named session | `/resume <session-name>` |
| Browse and pick a session | `/resume` (picker) |
| The three auto-compact choices | Continue compaction · Reject compaction · Resize context window |

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — where the context threshold, model, and profile for a session are set
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the per-session `traffic/` capture described here
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md) — keeping a long task alive between turns while a session is open
