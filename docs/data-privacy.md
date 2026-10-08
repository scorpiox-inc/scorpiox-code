# Privacy Architecture and Zero Data Collection Guarantee

Most AI coding tools ship a privacy policy. A policy is a promise about what happens to your data *after* it has been collected, written by the company doing the collecting. SCORPIOX CODE takes a different position: instead of promising to behave well with data it gathers, it ships an architecture in which there is nothing to gather. There is no account to create, no server that knows you exist, no background process that phones home, and no counter that increments when you launch the binary.

That is not a compliance posture. It is how the product is built, and every part of it is verifiable on your own machine with a terminal and a packet capture.

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** your prompts, your code, and your tool output go exactly one place you point them — the model endpoint in your config — and everything else (sessions, logs, history, traffic captures) is a plain folder on your own disk that you own outright.

---

## The one number we cannot produce

Every commercial AI coding tool can answer a question SCORPIOX CODE cannot: *how many people used it today?*

OpenCode, Cursor, and most tools in this category operate on a connected model. They know their active-user counts because they count them — a launch ping here, an account sign-in there, an analytics beacon riding along on every request. That number is the backbone of their dashboards, their usage analytics, and their growth metrics. It is also, unavoidably, proof that a pipe runs from your machine to their servers, and that everything moving through it is legible to them.

SCORPIOX CODE has no such pipe. **We literally do not know how many active users we have.** There is no sign-up, no device fingerprint, no install counter, and no "is this user still alive" heartbeat. If you run SCORPIOX CODE right now, nothing on the other end of any wire learns that you did. We cannot produce an "active users this month" figure — not because we are being modest, but because the mechanism that produces that number does not exist in the product.

This matters to you for a concrete reason. A tool that cannot count its users cannot profile them, cannot aggregate their behaviour, cannot leak their records, and cannot be compelled to hand them over. **What is never sent cannot be logged, aggregated, breached, or subpoenaed.**

---

## What "zero data collection" actually means

Three things that come standard in a commercial developer tool are absent from SCORPIOX CODE **by construction, not by configuration**:

1. **No phone-home.** There is no launch heartbeat, no periodic check-in, no update-check beacon, and no "is this installation still alive" ping. Run it with the network cable unplugged, pointed at a local model, and it behaves exactly as it does online — it never notices the difference.
2. **No telemetry pings.** There is no metrics endpoint, no crash-reporting service, no anonymous usage counter, and no device ID. The binary has nowhere to send a number, so it never sends one.
3. **No user registration.** You do not sign up. There is no login, no email, no license activation, and no identity tied to your machine. The binary runs because it is on your disk, full stop.

None of these are settings you can flip off. There is nothing to flip — the code path does not exist. That distinction is the whole point: a setting you can disable can be silently re-enabled, a default can drift between releases, and a checkbox can be reworded in a terms update. A capability that is not present cannot be switched on later without shipping new code that you can inspect.

---

## The complete inventory of outbound traffic

Zero-collection claims are cheap to make and hard to check, so here is the full list of every path a network byte can leave your machine, and the state each one ships in.

| Surface | What it sends | Ships as | Destination |
|---------|---------------|----------|-------------|
| **Model requests** | Your prompts, context, and tool definitions — the working payload of the agent | Always on, because it is the product | **Only** the endpoint in your config (see below) |
| **Usage tracking** (`USAGE_TRACKING`) | Token counts per model and rate-limit utilization, plus machine metadata | **Off** — disabled in every tier of the config cascade | A URL you set (`USAGE_API_URL`) |
| **Session event emission** (`EMIT_SESSION_TRACKING`) | Per-message conversation events, including your prompts and the assistant's replies | **Off** — disabled out of the box *specifically because it would transmit actual conversation content* | A URL you set (`EMIT_SESSION_API_URL`) |
| **Usage popups** (`/usage`) | A read-only query against the provider you already pay, using your own credential | Off until you open the popup | The provider's own usage API |
| **Server probing** (`/slots`, `/metrics`, `/models`) | Empty probe requests | Off until you invoke them | The `OPENAI_BASE_URL` you configured |
| **Voice transcription** (`/voice`) | The recording you just made | Per-invocation, only when you run it | A transcription endpoint you can override (`VOICE_WHISPER_URL`) |
| **MCP tools** | Whatever the tool call needs | Only for servers you configured | Your MCP servers (local subprocess or a URL you set) |
| **Web search / fetch** | The query or URL the agent chose to look up | Only when the agent runs those commands | The public engines and pages they name |
| **Distribution downloads** | Nothing of yours — plain GETs for an installer or a container image | Only when you install or launch a container | A distribution server, no identifiers attached |

Three things to notice about that table.

**The two reporting switches are the entire telemetry surface, and both are closed.** `USAGE_TRACKING=0` and `EMIT_SESSION_TRACKING=0` ship as the default in the shipped environment file, in the compiled-in defaults, and in the built-in defaults — three independent places, all zero. Nothing else in the product reports anything anywhere, so with both keys left alone there is no reporting path active at all.

**The gates are checked twice, not once.** The agent never even starts the reporting helper unless the switch reads `1`, and the helper itself re-checks the switch and exits immediately if it is not set. Flipping the switch back to `0` in any config tier takes effect on the next invocation, with no restart required.

**Nothing here has a default destination that is a SCORPIOX CODE collector.** An empty `USAGE_API_URL` falls back to a built-in endpoint rather than nowhere, so if you ever do turn tracking on, set a real URL of your own instead of blanking it — keep the data in-house, where it belongs.

```ini
# Shipped defaults — telemetry off, nothing sent anywhere
USAGE_TRACKING=0
EMIT_SESSION_TRACKING=0
```

---

## 100% local filesystem ownership

Everything SCORPIOX CODE remembers about a session lives in one folder on your disk:

```
.scorpiox/sessions/<session-id>/
```

Every session, every prompt, every tool execution, and every log line is a plain file in a directory you can open with any text editor. Inside a session folder you will find:

| File / folder | What it holds |
|---------------|---------------|
| `conversation.json` | The full verbatim transcript — every user message, assistant reply, tool call, and tool result |
| `messages/` | One numbered file per message and event, for SDK and headless consumers |
| `events/`, `events.jsonl` | The structured, machine-readable record of what happened and when |
| `traffic/` | The raw HTTP request and response bodies for every model call the session made |
| `agent.log`, `session.log` | The agent-level narrative and the runtime log |
| `trace.jsonl`, `stats.json` | The data-flow trace and the live usage snapshot, refreshed about once a second |
| `meta.json`, `config-snapshot.txt` | Session identity and the frozen configuration the session started with |
| `callbacks.json`, `required_skills.txt`, `inbox/` | The session's timers, skill contract, and drop-in input queue |

Because these are files and not database rows, the ownership is total:

- **Read it.** No proprietary format, no "export your data" button you have to go looking for.
- **Move it.** Copy the folder to another machine and the session comes with it intact.
- **Delete it.** Remove the folder and the data is gone. There is no server-side copy to request deletion of, because there never was one.
- **Back it up.** The folder *is* the backup.

Two details worth knowing, because they are the kind of thing other tools never mention:

- **Sessions never leak into your commits.** In a git repository, the product adds `.scorpiox/sessions/` to your `.gitignore` on first run if it is not already there, so your conversation history cannot end up committed, pushed, or mirrored by accident.
- **The session's config snapshot redacts secrets.** The frozen configuration written into each session folder masks every value whose key looks like a credential — anything containing key, pass, secret, token, or credential — so the snapshot is safe to read, share, or attach to a bug report.

Local retention is also yours to set. `SESSION_RETENTION_DAYS` (default `7`, set `0` to keep everything forever) prunes old session folders at startup. That cleanup runs against your own disk — there is no remote retention policy, no legal-hold copy, and no "deleted sessions are retained for X days for quality purposes" footnote, because there is no remote.

Worktree users get the same guarantee twice over: a session inside a git worktree keeps a mirror copy in the main repository's `.scorpiox/` so both trees can see it, and both copies are on your filesystem. No third location exists.

See [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md) for how this same folder doubles as a queryable archive, and [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) for the capture layout in detail.

---

## Direct network control: your content goes where you point it

The only outbound traffic that carries your content goes to the **LLM endpoint you explicitly configure**. Which endpoint that is — and therefore who physically receives your prompts and code — is a line in your configuration, not a hard-coded address:

| What you configure | Where your content goes |
|--------------------|-------------------------|
| **Local inference** — `llama.cpp`, `vLLM`, `ollama`, or LM Studio on `localhost` or your LAN (`PROVIDER=openai` + `OPENAI_BASE_URL`) | Nowhere off your network. This is the strongest configuration: literally zero external content egress. |
| **A self-hosted or on-prem endpoint** you operate | Only to infrastructure you run, under your logging and your rules. |
| **A vendor API you chose** — Anthropic, OpenAI, Google, xAI, GitHub Copilot, and the subscription providers | Only to the endpoint and vendor you selected, for the call you made. You opted into that specific transfer. |

There is no intermediary relay and no SCORPIOX CODE proxy sitting between you and the model by default. If you do not point the tool at a remote endpoint, there is no remote endpoint to reach.

The same principle covers credentials. Where your model token comes from is your choice too: the default reads the credential file your own login wrote, entirely offline, and the fleet-oriented sources — an HTTP endpoint, an SSH host, or a raw TCP relay — exist only for teams that want one credential served to many machines, and each one names its own endpoint in your config. On a personal machine, `CLAUDE_CODE_TOKEN_SOURCE=local` keeps every credential read on your disk and off the network.

> **Rule of thumb:** the most private configuration is local inference. Run the model on your own hardware and the only "network" your content touches is your own machine — the tool needs nothing else to function.

One related surface is worth naming because it is the opposite of a tracker: with an OpenAI-compatible endpoint you can attach session and thread identity headers to your requests (see [Session Identity Headers](identity-headers.md)). The values are UUIDs generated on your machine, they travel only to the endpoint you chose, and blanking the setting sends nothing. It is identity you hand out, not identity harvested from you.

---

## Verify it yourself

A guarantee you cannot check is marketing. Every claim on this page can be tested in a few minutes:

1. **Watch the network.** Capture traffic during a normal session (`tcpdump`, Wireshark, or your firewall's log). The only content-bearing connection is the model endpoint in your config. With local inference, the only traffic is on your LAN.
2. **Run it air-gapped.** Point the tool at a local model server and disconnect from the internet entirely. It works exactly as before — because it was never designed to need anything else.
3. **Confirm the defaults.** Check `USAGE_TRACKING` and `EMIT_SESSION_TRACKING` in your resolved configuration (the `/config` editor shows both, or grep your environment file). Both read `0` unless you changed them.
4. **Read your own traffic.** Open `.scorpiox/sessions/<session-id>/traffic/` — the verbatim request and response bodies of every model call are there, written by your machine, for your eyes. Nothing in that folder is uploaded anywhere.
5. **Check the gitignore.** In any git repository, confirm `.scorpiox/sessions/` appears in `.gitignore` after your first session.

If any of those steps surprises you — an unexpected host, a file outside your own filesystem — that would be a bug worth reporting, not expected behaviour.

---

## Honest edges

A privacy page that only lists guarantees is not credible. These are the boundaries of the claim, stated plainly:

- **Your model endpoint sees your model traffic.** Point SCORPIOX CODE at a hosted provider and that provider receives your prompts and returns the responses — that is inherent to using a hosted model, not a property of this tool. Run local inference and even that disappears.
- **The opt-in switches exist.** If you deliberately enable usage tracking or session-event emission, data flows to the URL you configured. The guarantee is that both are off by default, double-gated, and pointed wherever you say — not that no such path exists in the code.
- **Downloads are downloads.** Installing the product or pulling a container image fetches files from a distribution server. Those are plain GETs you initiated, carrying no session content and no identifier beyond what your network stack reveals — but like any web server, the distribution server sees the request.
- **Features you deliberately connect connect.** Enable the fleet supervisor's hub, a relay, or a messaging bridge and those services are part of your setup, on endpoints you named. The zero-collection guarantee covers what the product does on its own; it does not cover infrastructure you chose to attach.
- **Local files are your responsibility.** Deleting a session folder removes it permanently, with nothing server-side to restore from. That is the flip side of ownership, and it is the design rather than a limitation of it.

---

## How the rest of the field compares

| Dimension | SCORPIOX CODE | Typical connected AI coding tools (OpenCode, Cursor, and the like) |
|-----------|---------------|---------------------------------------------------------------------|
| **User registration** | None. The binary runs because you have it. | An account is the entry point; your identity is the record key. |
| **Do we know you exist?** | No. No registration, no device ID, no anonymous counter. | Yes — the account *is* the tracking record. |
| **Active-user count** | Unknown by design. There is no counter to read. | Known and reported; it is the core of their analytics. |
| **Telemetry / usage pings** | Off by default, opt-in, destination of your choosing. | On by default, reported to vendor infrastructure. |
| **Where your data lives** | A folder on your disk that you own and can delete. | Vendor infrastructure you do not control. |
| **Can you audit what was sent?** | Yes — verbatim request/response capture on your own disk. | Generally not. |

---

## The bottom line

- **No account, no phone-home, no telemetry pings** — we genuinely cannot count our active users, and that is the strongest privacy property a tool can have.
- **Both reporting switches ship off**, are double-gated, and point only at URLs you set.
- **All session data stays on your local filesystem** under `.scorpiox/sessions/`, in plain files you can read, move, or delete — with secrets redacted and sessions kept out of git.
- **The only content-bearing network calls go to the LLM endpoints you configure** — local, LAN, or a provider you chose — and identity headers are yours to add or withhold.

You can run SCORPIOX CODE fully air-gapped against your own inference and it will behave exactly as it does on the open internet — because it was built to have nothing to send in the first place.

---

## Related

- [Token Usage Observability](usage-observability.md) — the local surfaces that show your usage, and the opt-in tracker behind `USAGE_TRACKING`.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim on-disk record of every request and response.
- [Configuration and Profiles](scorpiox-env.md) — the cascade that resolves every switch above, and how to override per project or per profile.
- [Session Identity Headers](identity-headers.md) — the locally generated session and thread GUIDs, and the one key that turns them off.
- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md) — how sessions-as-folders power long runs without a server in the loop.
- [Serving Sites with scorpiox-server](scorpiox-server.md) — what the companion server writes to disk, and what never leaves your machine.
