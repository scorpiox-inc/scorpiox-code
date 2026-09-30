# API Traffic Logging and Complete Remote Call Transparency

Every agent you have used has made network calls on your behalf. Most of them do it in a way you cannot see, cannot replay, and cannot prove. The prompt you typed, the system instructions, the tool calls, the full context window, the exact endpoint, the exact headers, the exact response, the exact token count — all of it is assembled in memory, sent over the wire, and gone. When something goes wrong — a wrong model, a truncated context, a surprise cost, a context leak you did not expect — you are left arguing about what a tool did with your data and you have no way to check.

SCORPIOX CODE is built the other way. **Every single remote call it makes is written to disk, verbatim, the moment it happens** — the request before it leaves, the response when it arrives, in plain files under your session directory, in a structure you can open with any editor, `cat`, `jq`, or `grep`. There is no database, no proprietary format, no "contact support to export your data." It is an **immutable, filesystem-native audit trail** of every byte that left or entered your machine on the agent's behalf.

This page is the full how-to: where the files live, what each one contains, how to read them, how to replay a captured call, and the exact guarantees that make this a transparency tool rather than a logging feature bolted onto a black box.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** every outgoing HTTP request and every incoming response SCORPIOX CODE sends to a remote model endpoint is saved verbatim to `.scorpiox/sessions/<session>/traffic/` — the full request body, the endpoint URL, the headers, the response payload, and the token usage — so you can inspect, verify, and replay exactly what left your machine at any turn.

---

## Why this matters

An agent is only as trustworthy as your ability to check what it did. "Trust us" is not a transparency model. The three concrete problems a local, verbatim traffic capture solves:

| Problem | What a black box does | What SCORPIOX CODE does |
|---------|----------------------|-------------------------|
| **Cost surprise** | Token count is a number you cannot verify against the actual payload. | The exact request body and the provider's returned usage are both on disk. Open them and check the math yourself. |
| **Wrong behavior** | "The model did that" is not falsifiable. | The exact request that produced the behavior is preserved, with the full context and tool definitions, so you can see *why* it did what it did. |
| **Data leak** | You cannot prove what context was included in a request. | The full prompt, system instructions, and tool calls are captured verbatim. If something left your machine, it is in these files and you can see it. |

The guarantee is precise: **there is zero black-box network activity.** If SCORPIOX CODE sent it, it is in `traffic/`. If it is not in `traffic/`, SCORPIOX CODE did not send it. That is what "100% transparency" means in practice — the audit trail is complete by construction, not by promise.

---

## Where the capture lives

Every session gets its own traffic directory, created automatically when the session starts:

```
.scorpiox/
└── sessions/
    └── <session-id>/
        └── traffic/
            ├── traffic.log                 # chronological summary of every call
            ├── raw/                        # one file per request or response
            │   ├── 001-req-body.json
            │   ├── 001-res-body.json
            │   ├── 002-req-body.json
            │   ├── 002-res-body.json
            │   └── ...
            ├── raw_messages/               # message-oriented copy (per-endpoint)
            │   ├── 001-req.json
            │   ├── 001-res.json
            │   └── ...
            └── raw_usage/                  # extracted token-usage objects
                ├── 001-usage.json
                ├── 002-usage.json
                └── ...
```

A few things about that layout worth knowing up front:

- **It is per-session.** Each session writes to its own `traffic/` directory, so sessions never overwrite each other. A session is a set of files you own — inspect, copy, or delete them at will.
- **It is local.** Nothing in the traffic capture is uploaded anywhere. It is written to your filesystem, by your machine, for your eyes. (This is the same local-ownership guarantee described in [Data Privacy](data-privacy.md) — the traffic capture is the *evidence* for that guarantee.)
- **The number is the sequence.** `001`, `002`, `003` … is the request number within the session, in the order calls were made. The request and its response share a number, so `001-req-body.json` and `001-res-body.json` are one call.

---

## What each file contains

### `traffic.log` — the timeline

A single, append-only, human-readable log of every remote call in the session, one line per event, oldest first:

```
[09:41:02] #001 openai <- (48211 bytes)
[09:41:09] #001 openai -> (1204 bytes)
[09:41:09] #002 openai <- (48990 bytes)
[09:41:18] #002 openai -> (2876 bytes)
```

Each line carries a **timestamp**, the **sequence number** (`#001`), the **provider**, the **direction** (`<-` request going out, `->` response coming back), and the **byte size** of the payload. It is the fastest way to answer "how many calls did this session make, to whom, and how big were they" without opening a single file. Pair it with the `raw/` files when you want the actual content of a line.

### `raw/` — the verbatim payloads

This is the heart of the capture. One file per direction, per call, holding the **exact bytes** that were sent or received:

- **`NNN-req-body.json`** — the complete outgoing request body: the full prompt, every message in the context, the system instructions, the tool definitions, the model name, and the sampling parameters. This is the literal data that left your machine.
- **`NNN-res-body.json`** — the complete incoming response body: the model's reply, any tool calls it issued, and the provider's usage block. This is the literal data that came back.

For providers that translate between wire formats (for example, an Anthropic-style request being converted to a Google endpoint), the intermediate forms are captured too — you can see both the request as the agent assembled it and the request as it was actually put on the wire. There is no hidden translation step between "what the agent thought it sent" and "what the network actually carried."

The **endpoint URL and headers** for each call are recorded alongside the body, so you can see exactly which host, path, and headers a request used — not just the payload.

### `raw_messages/` — the message-oriented copy

A parallel copy of the request and response focused on the conversation itself (`NNN-req.json` / `NNN-res.json`). It is the same content as `raw/`, kept in a per-endpoint form that is convenient when you are reasoning about the conversation rather than the wire format.

### `raw_usage/` — the token accounting

For each response, the provider's **usage object** is extracted and saved on its own:

```json
{
  "input_tokens": 48211,
  "output_tokens": 312,
  "cache_read_input_tokens": 0,
  "cache_creation_input_tokens": 0
}
```

This is the number you would be billed on, pulled out of the response and written to a file of its own so you can sum, diff, or audit it independently of the rest of the payload. It is the difference between "the tool told me this is what it cost" and "I can add up the raw usage files and confirm it myself."

---

## Reading a capture

The files are plain JSON on disk. You do not need a special viewer, a plugin, or network access. A few common patterns:

**See what a session sent and received at a glance:**

```bash
cat .scorpiox/sessions/<session-id>/traffic/traffic.log
```

**Inspect the exact prompt of a specific call:**

```bash
jq . .scorpiox/sessions/<session-id>/traffic/raw/001-req-body.json
```

**Check the token usage for the whole session:**

```bash
jq -s 'map(.input_tokens) | add' \
   .scorpiox/sessions/<session-id>/traffic/raw_usage/*.json
```

**Find a leak** — search every request body for a string you did not expect to leave the machine:

```bash
grep -l "SECRET_VALUE" .scorpiox/sessions/<session-id>/traffic/raw/*-req*.json
```

If the string is not in any request file, it was not sent. That is the audit: the filesystem is the source of truth for what left your machine.

---

## Replaying a captured call

Inspecting a capture is one thing; being able to **reproduce** the exact call is what makes it a real debugging tool. SCORPIOX CODE ships an interactive helper, `scorpiox-executecurl`, that browses your traffic captures and re-issues any of them:

```bash
scorpiox-executecurl                 # list available captures
scorpiox-executecurl <session-id>    # jump straight to one session
```

It walks the session traffic directories (and, for older layouts, the legacy capture locations) and lets you pick a call to replay. This is the difference between "I saw what happened" and "I can make it happen again" — essential when you are chasing a provider-side quirk, a context-length edge case, or a request that only fails under specific conditions.

---

## Which calls are covered

The capture is **provider-agnostic by design**. Every remote model endpoint SCORPIOX CODE can be pointed at — local inference servers, OpenAI-compatible hosts, and the various hosted provider integrations — writes its calls into the same `traffic/` structure for the active session. The shape of the wire format differs per provider, so the intermediate files carry provider-specific labels, but the layout, the sequence numbering, the `traffic.log` timeline, and the `raw_usage/` extraction are the same everywhere.

That uniformity is the point: you learn the layout once, and it applies to whichever endpoint your session used. A session that mixes providers in a single run still produces one coherent, sequential, verbatim record.

---

## What "100% transparency" guarantees — and what it does not

Honest about the edges, the same way the [Data Privacy](data-privacy.md) page is:

- **The model endpoint sees the model traffic.** If you point SCORPIOX CODE at a hosted provider, *that provider* receives the prompts and returns the responses — that is inherent to using a hosted model. The traffic capture records what SCORPIOX CODE sent to and received from that endpoint; it does not, and cannot, see what the provider does internally with the request. Running local inference removes even that.
- **The capture is as complete as the call.** The guarantee is that every call SCORPIOX CODE makes is recorded. It is not a promise about what the provider's server does after it receives the bytes — that is the provider's business, and the capture is your record of the boundary between your machine and theirs.
- **Local files are your responsibility.** Because the capture lives on your filesystem, deleting a session removes its traffic record — and nothing is kept on a server to restore it. That is the flip side of ownership, and it is the same trade as every other session file.

The guarantee, precisely: **by construction, every outgoing request and incoming response SCORPIOX CODE produces for a remote model call is written verbatim to `.scorpiox/sessions/<session>/traffic/` before the session moves on.** No call is hidden, summarized away, or sent without a local copy. If you can read your traffic capture, you can verify exactly what left your machine — and the only way a byte leaves the machine is for it to be in those files.

---

## Related

- [Data Privacy and Zero Data Collection](data-privacy.md) — the local-ownership guarantee this capture is evidence for.
- [Configuration and Profiles](scorpiox-env.md) — how providers, models, and endpoints are selected.
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md) — how the context window is managed, which is what the request bodies show you.
