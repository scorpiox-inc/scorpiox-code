# API Traffic Logging and Complete Remote Call Transparency

SCORPIOX CODE never hides what it sends. Every HTTP request it makes to a remote LLM endpoint — and every response that comes back — is written to disk, byte for byte, as plain files on your machine. No black-box network activity. No "trust us, it's just talking to the model." You can open the file and read exactly what left your machine, exactly what came back, and how many tokens each call used.

This page explains what gets recorded, where it lives, and how to read it.

Source of truth: the per-provider network layers and the session layout at commit `5fd054b`.

---

## The one-line promise

**There is no remote call SCORPIOX CODE makes that you cannot see on your own disk.**

Every outgoing request body, the endpoint it went to, the headers it carried, the raw response it received, and the token usage the model reported — all of it is captured verbatim into a per-session folder that only you control. If it's not on your filesystem, SCORPIOX CODE didn't send it.

> **Why does this matter to you?** Most AI tools treat their network traffic as a closed pipe. You can't see what they prompt, what context they leak, or what they receive. That's a black box. SCORPIOX CODE turns the box into an audit log you can open in any text editor. What you can't inspect, you can't verify. This is inspectable.

---

## Where it all lives

Everything for one session lives in one place:

```
.scorpiox/sessions/<session>/traffic/
```

That directory is created automatically when a session starts, so it's always there — even for sessions where no calls were made yet. The traffic folder is local-only: it's never uploaded, never synced, and it's automatically added to `.gitignore` so you never accidentally commit it.

> **Local by default, git-ignored by default.** The traffic directory is part of your session, not your repo. Deleting `.scorpiox/sessions/<session>/traffic/` deletes that session's audit trail, and nothing else.

---

## What gets written

Each remote call is one numbered exchange. SCORPIOX CODE keeps a running counter per session (`001`, `002`, `003`, …) so requests and their responses stay paired. You'll find a small set of files per exchange:

| File | What it is |
|------|------------|
| `traffic.log` | A running, timestamped summary line for every request and response — the quickest way to scan a whole session. |
| `raw/…-req…` | The exact bytes sent to the endpoint: the request body (your prompt, tool calls, full context) plus, where applicable, the request headers. |
| `raw/…-res…` / `raw/…-resp-<code>.json` | The exact bytes the endpoint returned, along with its HTTP status code. |
| `raw_messages/…-req…` / `…-res…` | The same request and response bodies, kept in a dedicated folder for easy message-level review. |
| `raw_usage/…-usage.json` | The token usage object the model reported for that call (input, output, cache reads, cache writes). |

### `traffic.log` — the session timeline

A single append-only file, one line per direction. It's the fastest way to see the shape of a session at a glance:

```
[14:22:31] #001 openai <- (18234 bytes)
[14:22:35] #001 openai -> (4102 bytes)
[14:22:35] #002 openai <- (20117 bytes)
```

- `#001` — the exchange number, so you can jump straight to the matching files in `raw/`.
- `openai <-` — a request *leaving* your machine (the `<-` points toward the provider).
- `openai ->` — a response *coming back*.
- The byte count tells you the size of each payload without opening it.

### `raw/` — the verbatim payloads

The actual content. For a request you get the body exactly as it was sent — the full prompt, any tool calls, the whole conversation context the model saw. Where the provider layer records headers, the request headers are saved too, alongside the body. For a response you get the body exactly as it arrived, with the HTTP status code baked into the filename.

Because these are the raw bytes, "what did the model see?" and "what did the model say?" have a single, unambiguous answer: open the file.

### `raw_usage/` — token accounting

Each call's usage block is extracted and stored as its own JSON file. That's the token math for that call — prompt tokens in, completion tokens out, plus cache read and cache write counts. Useful for cost, budgeting, and spotting context that's ballooning turn over turn.

---

## What you can verify, end to end

Because the capture is complete and plain-text, these are all one `cat` away:

```bash
# Where is my session's traffic?
ls .scorpiox/sessions/

# The whole session's timeline, fastest first
cat .scorpiox/sessions/<session>/traffic/traffic.log

# Exactly what request #001 sent (prompt + tool calls + context)
cat .scorpiox/sessions/<session>/traffic/raw/001-req-body.json

# Exactly what the model returned for it
cat .scorpiox/sessions/<session>/traffic/raw/001-res-body.json

# Just the token usage for a call
cat .scorpiox/sessions/<session>/traffic/raw_usage/001-usage.json
```

That's the entire audit trail. No server to query, no account to check, no "contact support." The filesystem is the source of truth, and it's yours.

---

## Secrets stay masked, not missing

Transparency and safety are not in tension here. SCORPIOX CODE does **not** write your credentials into the log. Sensitive headers — API keys, bearer tokens — are redacted to `***MASKED***` when request headers are recorded, so the log shows *what* went out and *to where* without ever becoming a copy of your key.

You get the audit value (what was sent, where, when, how big, what came back) without turning your traffic log into a credential dump.

> **Rule of thumb:** the log answers "what left my machine?" completely, but it deliberately withholds the one thing you don't want sitting in a plaintext file — your secret.

---

## Every provider, one layout

The capture is uniform across the supported remote providers — Anthropic, OpenAI-compatible, GitHub Copilot, OpenAI Codex, Grok, Google Gemini / Vertex, Google Claude, Claude Code, and the rest. Whichever endpoint you point SCORPIOX CODE at, the same `.scorpiox/sessions/<session>/traffic/` structure is filled in.

That means the inspection workflow above doesn't change when you change providers. Point at a local llama.cpp server, point at a remote API, point at Copilot — the audit trail looks the same and lives in the same place.

---

## A standalone capture tool, for anything

Beyond its own provider calls, SCORPIOX CODE ships a general-purpose traffic capture you can wrap around *any* command — handy for inspecting a tool's HTTP behavior, not just SCORPIOX CODE's. It runs a local capture and writes the same kind of artifacts, plus a few analysis-friendly aggregates:

| Output | Purpose |
|--------|---------|
| `summary.json` | A quick overview of every captured request. |
| `conversation.jsonl` | The whole capture as a sequential, line-oriented log. |
| `all.har` | HTTP Archive format — drop it straight into Chrome DevTools. |
| `curl/` | Each request as an executable `curl` command, for replay or diffing. |

Same idea, broader net: raw, local, and yours to inspect.

---

## Verify it yourself

You don't have to take this on faith:

1. **Start a session** and make a couple of calls to whatever provider you use.
2. **Open the folder.** `ls .scorpiox/sessions/<session>/traffic/` — you'll see `traffic.log` and the `raw/`, `raw_messages/`, and `raw_usage/` folders filling up.
3. **Read a request and its response side by side.** The prompt you typed, the tool calls it triggered, and the model's reply are all in there, byte for byte.
4. **Check the key is gone.** Search the headers for your API key — you'll find `***MASKED***`, not the secret.

If any of that surprises you — a file that doesn't match what you expected to send, a key that *isn't* masked — that's a bug worth reporting, not expected behavior.

---

## TL;DR

- **No black-box network activity.** Every outgoing request and incoming response to any remote LLM endpoint is saved verbatim to disk.
- **One place to look:** `.scorpiox/sessions/<session>/traffic/` — `traffic.log` for the timeline, `raw/` for the exact bytes, `raw_usage/` for token counts.
- **Full visibility:** exact data sent (prompts, tool calls, context), exact endpoint and headers, timestamps, response payloads, and token usage.
- **Secrets stay safe:** API keys are masked in the log, never written out.
- **Uniform across providers** and **local-only** — the filesystem is the audit trail, and it's yours to open, diff, or delete.

You can read exactly what SCORPIOX CODE sent, to where, and what came back — for every single call, in every single session.
