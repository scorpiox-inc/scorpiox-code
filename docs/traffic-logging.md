# API Traffic Logging and Complete Remote Call Transparency

Every remote LLM provider call SCORPIOX CODE makes is saved to disk, verbatim, the moment it happens. The exact prompt it sent, the exact endpoint it hit, the exact headers it used, and the exact response that came back — all written to a plain folder on your machine that you can open with any editor.

There is no "trust us" layer. If a call was made, there is a file for it. If a file is on disk, you can read it. This page explains where those files live, what is captured in them, and how to use them to audit what left your machine at any given turn.

Docs for SCORPIOX CODE @ `b59223a`.

---

## Why this exists

Most agent tools are a black box around the network. They talk to an API, you get an answer, and the wire traffic in between is gone. When something looks off — a surprising token bill, a model behaving oddly, a context that seems to have changed — you have no record to check against. You are asked to take the tool's word for what it sent.

SCORPIOX CODE removes that gap entirely. **Every outgoing request and every incoming response to any remote LLM endpoint is written to disk before the conversation continues.** No sampling, no "recent N only", no opt-in flag. The capture is part of the normal request path, so what you see on disk is exactly what was sent and received — nothing more, nothing less.

The result is a **100% transparent, fully auditable** record of the session's remote activity:

- **The exact data sent** — the full request body, including the assembled prompt, every tool definition, every message in context, and any tool calls.
- **The exact endpoint and headers** — the URL the request was posted to and the HTTP headers it carried.
- **The exact response** — the full response body that came back, plus the HTTP status and any usage accounting the provider reported.
- **A per-call timeline** — a single log that lists, in order, every request and response with a timestamp and a byte count, so you can follow the session call by call.

You can answer, for any turn, the questions "what exactly did SCORPIOX CODE send to the provider?" and "what did the provider return?". The answer is on your disk, in plain files, in the order they happened.

---

## Where the captures live

Each SCORPIOX CODE session has its own folder under `.scorpiox/sessions/`, and traffic captures are stored inside the session, in a `traffic/` subdirectory:

```
.scorpiox/sessions/<session>/
  ├── conversation.json      # Full conversation history
  ├── session.log            # Session-level log output
  ├── meta.json              # Session metadata (model, provider, profile, ...)
  └── traffic/               # All remote API traffic for this session
```

Because the traffic folder lives *inside* the session directory, the network record for a session sits right next to the conversation it produced. Open the session, and both the dialogue and the wire traffic that drove it are in the same place.

The session folder name is a human-readable, date-anchored identifier (for example `2026_09_26_happy_euler`), so you can tell at a glance which day a capture belongs to. The full set of files that make up a session is documented in [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md).

---

## What is captured

Every API call is recorded as a pair of files — one for the request, one for the response — numbered in the order the calls were made. A call is a **request** (what you sent) followed by a **response** (what came back). For the *n*th call in the session you will find:

| File | What it holds |
|------|---------------|
| `NNN-req-headers.txt` | The request line (method and URL) plus the HTTP headers sent with the call. |
| `NNN-req-body.json` | The complete request body, verbatim: model, full message/context array, tools, and parameters. |
| `NNN-res-headers.txt` | The response status line and response headers. |
| `NNN-res-body.json` / `NNN-res-body.txt` | The complete response body, verbatim — JSON for providers that return JSON, raw stream text for providers that stream. |
| `NNN-usage.json` | The token-usage object the provider reported for the call, pulled out of the response on its own. |

`NNN` is a zero-padded sequence number, so the files sort in the order the calls happened: `001-req-body.json`, `002-req-body.json`, and so on. The same numbering appears in the per-call log, which makes it trivial to jump from a line in the log to the matching request and response files.

### The per-call log

Alongside the per-call files, each session keeps a single append-only log that records every request and response in order, with a wall-clock timestamp and a byte size. Each line shows which provider handled the call, the direction (outgoing request or incoming response), the endpoint or status, and the payload size.

Read the log top to bottom and you have the session's complete network timeline — every round trip, in order, with the size of each payload. When you find a line you want to inspect, the sequence number in the line points straight at the matching request and response files in `raw/`.

### Token usage, captured per call

When a provider reports token usage in its response, SCORPIOX CODE extracts that usage object and saves it as its own small file for the call, alongside the request and response. So for any call you can read the model's account of how many tokens it consumed without parsing the full response body by hand. This is the same data the provider uses for billing, captured locally so you can reconcile a session against your own records.

### What is redacted

The captures are faithful, but they are not credential dumps. Authentication material in the headers is masked before it is written to disk — bearer tokens and API keys do not end up in your traffic files in plaintext. Everything else — the URL, the other headers, the full request and response bodies — is captured verbatim.

---

## Inspecting a session's traffic

Because the capture is filesystem-native, you inspect it with the tools you already use. There is no viewer to install and no server to query.

A typical workflow:

1. **Find the session folder.**
   ```bash
   ls -t .scorpiox/sessions/
   ```
   The newest first, each named by date and a short identifier.

2. **Read the timeline.**
   ```bash
   cat .scorpiox/sessions/<session>/traffic/*.log
   ```
   This is the call-by-call record: provider, direction, endpoint or status, and payload size, in order. It is the fastest way to see how many calls a session made and roughly how big each was.

3. **Open a specific call.** Pick a sequence number from the log, then read its files.
   ```bash
   cat .scorpiox/sessions/<session>/traffic/raw/001-req-body.json
   cat .scorpiox/sessions/<session>/traffic/raw/001-res-body.json
   ```
   The request file shows exactly what left the machine; the response file shows exactly what came back.

4. **Check what was sent.** The request body is the assembled prompt — every message, every tool, every parameter — so it answers "what did SCORPIOX CODE actually send the provider?" with no reconstruction and no guesswork.

5. **Check the cost.** Open the usage file for the call to see the token counts the provider reported.
   ```bash
   cat .scorpiox/sessions/<session>/traffic/*usage*.json
   ```

Nothing about this requires network access, a running session, or a privileged mode. The files are plain text and JSON, so `jq`, `grep`, and your editor all work on them directly.

---

## Zero black-box activity

The property worth keeping in mind is the negative one: **there is no network activity that is not captured.** Every remote LLM call goes through the same request path that writes these files, so the on-disk record is complete by construction, not by sampling.

That completeness, combined with the fact that the record lives in a folder you own, gives you an **immutable, filesystem-native audit trail**. You can:

- **Verify what left the machine.** Open any request file and read, word for word, the prompt, tools, and context that were sent.
- **Reconstruct the conversation on the wire.** The ordered request bodies, read in sequence, are the exact progression of context the provider saw across the session.
- **Reconcile usage.** The per-call usage files let you line up a session's token consumption against a provider's billing statement.
- **Replay or diff.** Because the bodies are saved verbatim, you can diff two sessions, two models, or two prompts against each other to see precisely what changed.
- **Own the record completely.** The files are yours, on your disk. Delete the session folder and the record is gone; copy it and it travels with you.

This is the traffic-side complement to the session's local-first design. The conversation lives in the session folder, and now the network traffic that produced it lives right beside it. Together they make a session fully reproducible and fully inspectable, with no part of the picture held on someone else's server.

For the broader guarantee — no telemetry, no phone-home, local ownership of every session artifact — see [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md).

---

## Gotchas

- **The traffic folder is part of the session, so it is as local as the session itself.** It lives under `.scorpiox/sessions/<session>/`, and that directory is added to `.gitignore` where applicable, so captures are not swept into a commit and pushed to a remote. They stay on your machine unless you move them.
- **Auth material is masked, the rest is verbatim.** Bearer tokens and API keys are redacted in the captured headers, so you will not find a live credential in a traffic file. But the URLs, the other headers, and the full request and response bodies are captured exactly as they were sent and received.
- **Response file extension follows the payload.** Providers that return JSON give a `.json` response body; providers that stream give a raw `.txt` body of the stream. Both are the complete response, just different shapes.
- **Captures are append-only and per-session.** A session's traffic is written as the calls happen and is not rewritten or truncated while the session runs. The sequence numbers make the ordering explicit and unambiguous.
- **This is the API traffic, not the agent's internal logging.** The session also keeps its own log and event streams (the conversation, agent log, events). Traffic captures specifically record the remote HTTP calls — what crossed the network to the provider and back.
- **Credential safety is on you, too.** Because request bodies contain your full prompt and context verbatim, treat the traffic folder the same way you treat the session folder: it is sensitive data. Do not commit or share it carelessly.

---

*Docs for SCORPIOX CODE @ `b59223a`.*
