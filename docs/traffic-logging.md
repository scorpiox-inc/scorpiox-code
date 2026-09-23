# API Traffic Logging and Complete Remote Call Transparency

Every time SCORPIOX CODE talks to a remote model, it makes a network call: a request carrying your prompt, your code, and the full conversation, and a response carrying the model's reply. Most tools treat that exchange as a black box. The request goes out, a reply comes back, and the details are gone — you are left trusting the tool that it sent only what it said it sent and received only what it said it received.

SCORPIOX CODE does not work that way. **Every single request it sends and every single response it receives from a remote LLM endpoint is written to disk, verbatim, before anything else happens.** Not sampled. Not summarized. Not "if something goes wrong." Every call, every time, to a plain folder on your own filesystem.

That is the whole point of this page: you can open that folder, read every byte that left your machine and every byte that came back, and confirm with your own eyes that the network activity was exactly what you expected. There is no hidden telemetry, no silent background call, no request you did not approve. If it went over the wire, it is a file you can read. If you cannot find a file for a call, that call never happened.

Source of truth: `sx_provider_openai.c`, `sx_provider_anthropic.c`, `sx_provider_copilot.c`, `sx_provider_codex.c`, `sx_provider_claude_code.c`, `sx_provider_grok.c`, `sx_provider_gemini_vertex.c`, `sx_claude.c`, `sx_state.c`, and `sx_session.c` at commit `6c70ad6`.

---

## What "100% transparency" actually means

A black-box agent is one where you cannot tell what it did unless it tells you, and it has no obligation to tell you the truth. The network layer is where the danger hides, because a tool that looks innocent in the terminal can still quietly send your source code, your environment, or your session history somewhere you never chose.

SCORPIOX CODE removes that class of risk at the root. For each remote provider API call it captures:

- **The exact request body** — the prompt, the full conversation context, the system instructions, the tool definitions, and the tool calls, exactly as serialized and sent. If the model saw your file contents, you can read them here.
- **The exact endpoint and headers** — the full URL, the HTTP method, and the request headers, so you can verify *where* a call went, not just *what* was in it.
- **The exact response** — the raw response payload the model returned, the HTTP status code, and the response headers.
- **Timing and usage** — a timestamp for each direction and the token-usage figures the provider reported for that call.

Because all of it is captured on the way out *and* on the way back, the record is complete and internally consistent. You are not being shown a friendly summary the tool composed; you are being shown the wire data.

---

## Where your traffic lives

Traffic captures live inside the session folder, right beside the conversation and the event log, under the session's `traffic/` directory:

```
.scorpiox/sessions/<session>/traffic/
├── traffic.log          # One line per direction: timestamp, call number, provider, size
├── raw/                 # Verbatim bytes, one file per direction per call
│   ├── 001-req-headers.txt   # Method, full URL, and request headers
│   ├── 001-req-body.json     # Exact request body as sent
│   ├── 001-res-body.json     # Exact response body as received
│   ├── 002-req-headers.txt
│   ├── 002-req-body.json
│   └── 002-res-body.json
├── raw_messages/        # Request/response bodies mirrored in the provider's message shape
│   ├── 001-req.json
│   └── 001-res.json
└── raw_usage/           # Token-usage figures extracted from each response
    └── 001-usage.json
```

A few things about the layout:

- **`traffic.log` is the index.** It is a plain append-only text log. Each line records the time, the call number, the provider, and the direction (`<-` for the request going out, `->` for the response coming back) and the payload size in bytes. It is the fastest way to see, at a glance, exactly how many remote calls a session made and roughly how large each one was.
- **`raw/` is the ground truth.** The numbered files are the actual bytes. The request headers file holds the method, the full endpoint URL, and the headers; the request-body and response-body files hold the verbatim payloads. The three-digit number is the call sequence within the session, so `004-req-body.json` and `004-res-body.json` are the request and response of the same fourth call.
- **`raw_messages/` and `raw_usage/`** are convenience mirrors of the same data in the shape the provider uses, plus the parsed token usage, so you can answer "how many tokens did this cost" without grepping the raw response.

The naming is consistent across providers — OpenAI, Anthropic, Copilot, Codex, Claude Code, Grok, Gemini Vertex, and the rest all write into the same session `traffic/` tree with the same shape. You learn the layout once and it holds everywhere.

---

## How to read a call

To verify a specific turn, start with the index and drill in:

1. **Open the index.** `traffic.log` tells you how many calls the session made and, by their sizes, which ones were large (a big request body usually means a lot of context was in flight).
2. **Pick a call number.** Say you want to understand the fourth exchange.
3. **Read what left.** Open `raw/004-req-headers.txt` to confirm the endpoint and method, then `raw/004-req-body.json` to see the exact prompt, tools, and context that were sent.
4. **Read what came back.** Open `raw/004-res-body.json` for the model's response and `raw_usage/004-usage.json` for the tokens it consumed.

That is a complete, self-contained audit of one remote call. Repeat it for any call number, or for the whole session, and you have a full accounting of everything that crossed the network boundary. Nothing in a SCORPIOX CODE session can reach a remote endpoint without leaving exactly this trail.

---

## What you can do with it

The record is plain text and plain JSON in a real folder, so any tool you already have works on it:

- **Audit a single turn.** Confirm the exact prompt and context the model saw, and the exact reply it returned.
- **Replay a request.** The endpoint, method, headers, and body are all present, so you can reconstruct the call and reproduce it against the same or a different endpoint.
- **Diff two runs.** Capture the same task twice and compare the `traffic/` folders to see precisely what changed in what was sent and what came back.
- **Trace token cost.** `raw_usage/` gives you the provider-reported usage per call, so you can line up spend with the exact request that incurred it.
- **Hand it to someone.** The folder is self-contained. Copy it and a reviewer can inspect every remote call without touching your machine.

Because the files are append-only and written as the calls happen, the record is an immutable, filesystem-native audit trail. You do not need to export anything or enable a debug mode; the evidence is on disk the moment each call completes.

---

## Where this fits

Traffic logging is the network half of a single design: everything a session does is a file you own, and nothing reaches the network without leaving a readable trace.

- For the broader guarantee that your content goes only to the endpoint you configure and nowhere else, see [Data Privacy](data-privacy.md).
- For how the same session folder doubles as the long-horizon conversation archive that traffic captures sit beside, see [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md).
- For the provider-specific details of which endpoints and credentials each backend uses, see the individual provider pages such as [OpenAI Provider](openai-provider.md) and [Claude Code Provider](claude-code-provider.md).

---

## Gotchas

- **The record is verbatim, not redacted.** The files hold the actual bytes that crossed the wire. That is the point of the transparency — but it also means your prompts and code are stored there in plain form, exactly as the conversation itself is. Treat the `traffic/` folder with the same care as the rest of the session.
- **It is a log, not a cache.** The captures exist for inspection and verification. They are not replayed automatically and do not change what the model does. Reading them changes nothing about the session.
- **Big sessions make big folders.** Each call writes full request and response bodies, and long sessions with lots of context send large payloads every turn. The folder grows with the session; that is the cost of keeping a complete, unsummarized record, and it is the same reason the session folder is the natural place for it.
- **Every call is captured, including the ones you did not ask for.** That is the feature. If a tool call or a background exchange you did not expect appears in `traffic/`, you have just found it — which is precisely the zero-black-box guarantee this page is about.
