# API Traffic Logging and Complete Remote Call Transparency

Every time SCORPIOX CODE talks to a remote model, it makes a network call: a request carrying your prompt, your code, and the full conversation, and a response carrying the model's reply. Most tools treat that exchange as a black box. The request goes out, a reply comes back, and the details are gone — you are left trusting the tool that it sent only what it said it sent and received only what it said it received.

SCORPIOX CODE does not work that way. **Every single request it sends and every single response it receives from a remote LLM endpoint is written to disk, verbatim, the moment it happens.** Not sampled. Not summarized. Not "if something goes wrong." Every call, every time, to a plain folder you own on your own filesystem.

That is the whole point of this page: you can open that folder, read every byte that left your machine and every byte that came back, and confirm with your own eyes that the network activity was exactly what you expected. There is no hidden telemetry, no silent background call, no request you did not approve. If it went over the wire, it is a file you can read. If you cannot find a file for a call, that call never happened.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** every outgoing HTTP request and every incoming response SCORPIOX CODE sends to a remote model endpoint is saved verbatim to `.scorpiox/sessions/<session>/traffic/` — the full request body, the endpoint URL, the headers, the response payload, and the token usage — so you can inspect, verify, and reproduce exactly what left your machine at any turn.

---

## Why this matters

An agent is only as trustworthy as your ability to check what it did. "Trust us" is not a transparency model. Three concrete problems a local, verbatim traffic capture solves:

| Problem | What a black box does | What SCORPIOX CODE does |
|---------|----------------------|-------------------------|
| **Cost surprise** | Token count is a number you cannot verify against the actual payload. | The exact request body and the provider's returned usage are both on disk. Open them and check the math yourself. |
| **Wrong behavior** | "The model did that" is not falsifiable. | The exact request that produced the behavior is preserved, with the full context and tool definitions, so you can see *why* it did what it did. |
| **Data leak** | You cannot prove what context was included in a request. | The full prompt, system instructions, and tool calls are captured verbatim. If something left your machine, it is in these files and you can see it. |

The guarantee is precise: **there is zero black-box network activity.** If SCORPIOX CODE sent it, it is in `traffic/`. If it is not in `traffic/`, SCORPIOX CODE did not send it. That is what "100% transparency" means in practice — the audit trail is complete by construction, not by promise.

---

## What "100% transparency" actually means

For each remote provider API call, SCORPIOX CODE captures:

- **The exact request body** — the prompt, the full conversation context, the system instructions, the tool definitions, and the tool calls, exactly as serialized and sent. If the model saw your file contents, you can read them here.
- **The exact endpoint and headers** — the full URL, the HTTP method, and the request headers, so you can verify *where* a call went, not just *what* was in it. Credential headers are masked before they are written (see the format notes below), so a capture is safe to share.
- **The exact response** — the raw response payload the model returned, the HTTP status code, and the response headers.
- **Timing and usage** — a timestamp for each direction and the token-usage figures the provider reported for that call.

Because all of it is captured on the way out *and* on the way back, the record is complete and internally consistent. You are not being shown a friendly summary the tool composed; you are being shown the wire data.

---

## Where your traffic lives

Every session gets its own traffic directory, created automatically the moment the session starts — before the first call is made, so the folder is always waiting:

```
.scorpiox/
└── sessions/
    └── <session-id>/
        └── traffic/
            ├── traffic.log                 # chronological one-line index of every call
            ├── raw/                        # verbatim bytes, one file per direction per call
            │   ├── 001-req-headers.txt     # method, full URL, and request headers
            │   ├── 001-req-body.json       # exact request body as sent
            │   ├── 001-res-body.json       # exact response body as received
            │   ├── 002-req-body.json
            │   ├── 002-res-body.json
            │   └── ...
            ├── raw_messages/               # the same request/response in the provider's message shape
            │   ├── 001-req.json
            │   ├── 001-res.json
            │   └── ...
            └── raw_usage/                  # token-usage objects extracted from each response
                ├── 001-usage.json
                └── ...
```

A few things about the layout:

- **`traffic.log` is the index.** It is a plain append-only text log. Each line records the time, the call number, the provider, the direction — `<-` for the request going out, `->` for the response coming back — and the payload size in bytes. It is the fastest way to see, at a glance, exactly how many remote calls a session made and roughly how large each one was.
- **`raw/` is the ground truth.** The numbered files are the actual bytes. The request-headers file holds the method, the full endpoint URL, and the headers; the request-body and response-body files hold the verbatim payloads. The three-digit number is the call sequence within the session, so `004-req-body.json` and `004-res-body.json` are the two halves of the same fourth call.
- **`raw_messages/`** is a convenience mirror of the same request and response in the provider's message-oriented shape — handy when you are reasoning about the conversation rather than the wire format.
- **`raw_usage/`** holds the provider's parsed token-usage object, pulled out of each response and written on its own so you can sum, diff, or audit cost independently of the rest of the payload.

### Exact file names vary by provider

The three-digit call number and the `raw/` / `raw_messages/` / `raw_usage/` split are consistent everywhere, but the individual file names differ slightly by provider because each wire protocol names its pieces differently. In practice you will see:

| Provider path | Request | Response | Notes |
|---------------|---------|----------|-------|
| OpenAI-compatible | `raw/001-req-body.json`, `raw_messages/001-req.json` | `raw/001-res-body.json`, `raw_messages/001-res.json` | Also writes `raw/001-openai-req.json`, `raw/001-openai-req-url.txt`, and `raw/001-openai-res-headers.txt` for the subprocess transport, so the URL and status are readable as plain text. |
| Anthropic Messages | `raw/001-req-body.json` | `raw/001-resp-<status>.json` | The HTTP status is embedded in the response file name, so a `200` and a `429` are never confused. |
| Copilot, Codex, Grok | `raw/001-req-body.json` | `raw/001-res-body.txt` | Streaming providers keep the response as raw event text, so what you see is what crossed the wire. |
| Claude Code | `raw/001-req-body.json` | `raw/001-res-body.json` | A request that fails local validation is written as `raw/001-INVALID-req-body.json`, so a malformed call is preserved rather than dropped. |
| Google (Gemini / Claude via OAuth) | `raw/001-<label>.json` | `raw/001-<label>.json` | Labeled pairs identify the request and response halves. |

You learn the layout once and it holds everywhere. A session that mixes providers in a single run still produces one coherent, sequential record. If a provider has no session directory yet, it falls back to a timestamped folder under `.scorpiox/traffics/providers/<provider>/`, so no call is ever silently uncaptured.

---

## How to read a call

To verify a specific turn, start with the index and drill in:

```bash
# 1. See the whole session at a glance
cat .scorpiox/sessions/<session-id>/traffic/traffic.log

# 2. Read the exact prompt and context that left your machine
jq . .scorpiox/sessions/<session-id>/traffic/raw/004-req-body.json

# 3. Read what came back
jq . .scorpiox/sessions/<session-id>/traffic/raw/004-res-body.json

# 4. Confirm the endpoint and method
cat .scorpiox/sessions/<session-id>/traffic/raw/004-req-headers.txt
```

That is a complete, self-contained audit of one remote call. Repeat it for any call number, or for the whole session, and you have a full accounting of everything that crossed the network boundary.

Three more patterns worth knowing:

```bash
# Sum the token usage for the entire session
jq -s 'map(.input_tokens) | add' \
   .scorpiox/sessions/<session-id>/traffic/raw_usage/*.json

# Find a leak — search every request body for a string that should not have left
grep -l "SECRET_VALUE" .scorpiox/sessions/<session-id>/traffic/raw/*-req*.json

# Diff two runs of the same task to see exactly what changed
diff -ru run-a/traffic run-b/traffic
```

If the string is not in any request file, it was not sent. That is the audit: the filesystem is the source of truth for what left your machine.

---

## Format notes you should know

- **Credential headers are masked.** API keys and authorization headers are redacted before they are written, so a capture is safe to hand to a reviewer or attach to a bug report. The request *body* is captured verbatim — that is the point — so treat a `traffic/` folder with the same care you would treat the conversation itself.
- **Nothing needs enabling.** There is no debug flag, no "start capture" button, and no export step. The trail is written as each call completes, and the session folder is the export.
- **The directory follows the session.** When you compact, clear, or resume a session, traffic logging re-points to the new session folder, so records never leak into a previous session's directory. The call sequence starts fresh with each session.
- **Worktrees are covered.** When you run a session from a git worktree, the same capture is mirrored into the main checkout's session folder, so the traffic record is visible from either location.
- **The folder is private by default.** Session and traffic files are created with owner-only permissions, so a capture on a shared machine is not readable by other users.

---

## What you can do with it

The record is plain text and plain JSON in a real folder, so any tool you already have works on it:

- **Audit a single turn.** Confirm the exact prompt and context the model saw, and the exact reply it returned.
- **Reproduce a request.** The endpoint, method, headers, and body are all present, so the call can be reconstructed. The `scorpiox-executecurl` utility browses your session traffic and re-issues a captured call for you, which turns "I saw what happened" into "I can make it happen again."
- **Diff two runs.** Capture the same task twice and compare the `traffic/` folders to see precisely what changed in what was sent and what came back.
- **Trace token cost.** `raw_usage/` gives you the provider-reported usage per call, so you can line up spend with the exact request that incurred it.
- **Hand it to someone.** The folder is self-contained. Copy it and a reviewer can inspect every remote call without touching your machine.

Because the files are written as the calls happen, the record is an immutable, filesystem-native audit trail. The evidence is on disk the moment each call completes.

---

## Capturing traffic for tools outside SCORPIOX CODE

The session trail above covers every call SCORPIOX CODE itself makes. When you need the same visibility into *another* program that is not going through SCORPIOX CODE — a vendor CLI, a script, a one-off `curl` — the standalone `scorpiox-traffic` utility wraps any command in a local capture proxy:

```bash
scorpiox-traffic <command> [args...]
# e.g.
scorpiox-traffic curl -s https://api.example.com/v1/models
```

It writes a self-contained capture folder for the run, with the same verbatim request/response files plus ready-made artifacts: a request index, a summary, a conversation view, an HTTP Archive (`.har`) you can open in any browser devtools, and a replayable `curl` script per call. This is the escape hatch for "I want to see exactly what *that* binary sent," and it is what makes the transparency story hold beyond the agent's own traffic.

> **One honest caveat.** To intercept HTTPS, `scorpiox-traffic` runs a local man-in-the-middle proxy and points the wrapped command at it through proxy environment variables. That is exactly how it is able to read encrypted traffic, and it is also why it must disable TLS verification for the proxied process. Use it deliberately, on commands you trust, and treat the resulting capture folder as sensitive. The always-on session trail described above has no such caveat — it is written by SCORPIOX CODE itself, in-tree, with no proxy.

---

## What "100% transparency" guarantees — and what it does not

Honest about the edges, the same way the [Data Privacy and Zero Data Collection](privacy.md) page is:

- **The model endpoint sees the model traffic.** If you point SCORPIOX CODE at a hosted provider, *that provider* receives the prompts and returns the responses — that is inherent to using a hosted model. The traffic capture records what SCORPIOX CODE sent to and received from that endpoint; it does not, and cannot, see what the provider does internally with the request. Running local inference removes even that boundary.
- **The capture is as complete as the call.** The guarantee is that every call SCORPIOX CODE makes is recorded. It is not a promise about what the provider's server does after it receives the bytes — that is the provider's business, and the capture is your record of the boundary between your machine and theirs.
- **Local files are your responsibility.** Because the capture lives on your filesystem, deleting a session removes its traffic record, and nothing is kept on a server to restore it. That is the flip side of ownership, and it is the same trade as every other session file.

The guarantee, precisely: **by construction, every outgoing request and incoming response SCORPIOX CODE produces for a remote model call is written verbatim to `.scorpiox/sessions/<session>/traffic/` before the session moves on.** No call is hidden, summarized away, or sent without a local copy. If you can read your traffic capture, you can verify exactly what left your machine — and the only way a byte leaves the machine is for it to be in those files.

---

## Related

- [Data Privacy and Zero Data Collection](privacy.md) — the local-ownership guarantee this capture is evidence for.
- [Configuration and Profiles](scorpiox-env.md) — how providers, models, and endpoints are selected through the cascade.
- [Session Identity Headers](identity-headers.md) — the headers that travel on every request and appear in the captured request-headers files.
- [Using the OpenAI Provider](openai-provider.md) — the provider whose calls fill most local traffic captures, with its own URL and status files.
