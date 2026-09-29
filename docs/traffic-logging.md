# API Traffic Logging and Complete Remote Call Transparency

When SCORPIOX CODE talks to a remote model, most tools treat that conversation as a black box. You see a prompt go in, you see an answer come back, and the only way to know what actually happened in between is to trust the tool. SCORPIOX CODE does not ask for that trust. Every single outgoing request and every single incoming response is written to disk, verbatim, on your machine, the moment it happens — in plain files you can open in any text editor.

This page explains what is captured, exactly where it lives, what each file contains, and how to use it to verify, to the byte, what left your machine at any given turn.

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** nothing about your network traffic is a secret, a summary, or an after-the-fact report. The exact bytes you sent and the exact bytes you received are saved as raw files on your disk, per session, so the audit trail is immutable, local, and yours to inspect.

---

## Why every call is recorded

A remote call to a model is the one place where information actually leaves your machine. Everything else in SCORPIOX CODE is local. Because the model endpoint is the only outbound path, that single path is where transparency matters most.

Recording it completely changes the security model:

- **No black-box network activity.** If a request was sent, you can see it. There is no hidden metadata, no silently stripped context, no background channel you cannot see. What you can inspect is all that was sent.
- **An immutable, filesystem-native trail.** The record is a set of plain files. There is no database to reverse-engineer, no proprietary format to decode, and no service that holds the only copy. It is on your disk, in a folder you own outright.
- **Verifiability over trust.** You do not have to believe the tool is only sending what it claims to be sending. You open the file and read what actually went out.

This is the same philosophy behind the zero-data-collection architecture: the guarantee is not a promise, it is something you can check with your own eyes.

---

## Where the traffic lives

Each session has its own traffic directory. For a session with identifier `<session>`, every recorded exchange is written under:

```
.scorpiox/sessions/<session>/traffic/
```

The directory is created up front when the session starts, so it exists even before the first request. Everything inside it is per-session: one session's traffic never mixes with another's. If you run multiple sessions, each gets its own isolated folder under `.scorpiox/sessions/`.

A typical traffic directory looks like this:

```
.scorpiox/sessions/<session>/traffic/
  traffic.log           # human-readable chronological summary of every exchange
  raw/                  # the verbatim bytes
    001-req-body.json   # exact request body sent on call #001
    001-req-headers.txt # exact request line + headers for call #001
    001-resp-200.json   # exact response body returned on call #001
    002-req-body.json
    002-req-headers.txt
    002-resp-200.json
    ...
```

The numbering is a simple per-session sequence: call `001`, then `002`, and so on. The response file carries the HTTP status code in its name (`resp-200.json`, `resp-429.json`, ...), so you can tell at a glance which calls succeeded and which hit an error, a retry, or a rate limit.

---

## What is captured in each file

The record is verbatim, not a summary. Here is what you can read out of each file.

### `traffic.log` — the timeline

`traffic.log` is an append-only, chronological log of the session's network activity. One line per event, in the order it happened, each stamped with a local time. It is the fastest way to get the shape of a session: how many calls were made, roughly how big each one was, and where things went wrong.

### `raw/*-req-body.json` — exactly what you sent

This is the request body, byte for byte, in the raw JSON it was transmitted as. Read it and you see the exact prompt, the full conversation context, every tool definition, and any tool calls that were part of the request. If something you did not expect was included in a prompt, it will be sitting in this file. Nothing is paraphrased, truncated, or summarized here.

### `raw/*-req-headers.txt` — the endpoint and the headers

Each request file pairs with a headers file containing the exact request line and the HTTP headers that were sent, including the full endpoint URL and the request method. This is where you verify *where* a call went, not just *what* it contained.

One deliberate exception: the API key is not written to disk. The `x-api-key` header is masked in the recorded headers, so your credentials never land in the traffic log. Every other header is recorded as sent.

### `raw/*-resp-<code>.json` — exactly what came back

The response body, verbatim, named after the HTTP status code that produced it. This is the raw payload the model returned: the completed message, any tool calls it requested, and the usage figures for that call. Because the status code is in the filename, a `429` (rate limited) or a `5xx` (provider error) is immediately visible and separate from a clean `200`.

### Timestamps and token usage

Because every file is written at the moment the call happens and `traffic.log` is time-stamped line by line, the trail carries a real timeline. And because the response bodies are stored whole, the per-call token usage the provider returned — input, output, and any cache-read or cache-creation counts — is right there in the raw response for any call where the provider included it. You never have to rely on an in-memory number that disappears when the session ends.

---

## Reading a session's traffic yourself

You do not need a special tool. Everything is plain text on disk.

To list the calls in a session and their sizes:

```bash
cat .scorpiox/sessions/<session>/traffic/traffic.log
```

To read exactly what was sent on the first call:

```bash
cat .scorpiox/sessions/<session>/traffic/raw/001-req-body.json
```

To see the endpoint and headers for that same call:

```bash
cat .scorpiox/sessions/<session>/traffic/raw/001-req-headers.txt
```

And to read what the model returned:

```bash
cat .scorpiox/sessions/<session>/traffic/raw/001-resp-200.json
```

That is the entire audit. Three files per call, all readable, all local, all yours. If you want to script against it — diff two sessions, grep a prompt across history, feed it to a log viewer — it is plain text on the filesystem, so standard tools work unchanged.

---

## What this guarantees, and what it does not

Be precise about the guarantee, because it is strong but bounded:

- **It is complete for what left the machine.** The outbound request body is the exact data that was transmitted. If a file shows a prompt, that prompt went out. There is no hidden payload you cannot see.
- **It is verbatim, not interpretive.** These are the bytes, not a description of the bytes. You are never reading a summary of what you sent — you are reading what you sent.
- **It is local and under your control.** The trail is on your disk. You can read it, copy it, move it, or delete it. Nothing about it requires a server, an account, or a vendor to access.
- **It does not reach back.** Recording traffic is a local write. Saving these files sends nothing anywhere. The files exist on your machine and nowhere else unless you put them there.

In short: the record proves what went out, it proves it byte for byte, and it lives only where you can look.

---

## How this differs from typical tools

| Dimension | SCORPIOX CODE | Typical AI coding tools |
|-----------|---------------|-------------------------|
| **Outbound requests** | Saved verbatim to disk, per session. | Not exposed; you infer what was sent from the UI. |
| **Endpoint & headers** | Recorded, including the exact URL. | Hidden inside the tool. |
| **Response payloads** | Saved raw, status-coded, readable. | Shown in the UI only, not persisted as raw data. |
| **Audit trail** | Plain files on your filesystem, immutable and local. | Vendor-side logs or none at all. |
| **Credentials in the log** | API key masked; never written to disk. | N/A — there is no log to inspect. |
| **How you verify it** | Open the file and read the bytes. | Trust the tool. |

The difference is not a feature flag. It is whether the network path is observable at all. In SCORPIOX CODE it is, completely, and the evidence sits in a folder you can open right now.

---

## Related

- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md)
- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md)
