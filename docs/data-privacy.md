# Privacy Architecture and Zero Data Collection Guarantee

Most AI coding tools ship a privacy policy. SCORPIOX CODE ships an architecture that makes the policy a non-issue.

There is no account to create, no server that knows you exist, and no background process that phones home. The reason we can guarantee **zero data collection and zero telemetry** is not that we promise to delete what we collect — it is that there is no collection mechanism to delete. The data never leaves the machine you run it on, except for the model tokens you deliberately send to an endpoint you chose.

This page explains what that guarantee means in practice, where your data actually lives, and how to verify the whole thing yourself.

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** your prompts, your code, and your tool output go exactly one place you point them — the model endpoint in your config — and everything else (sessions, logs, history) is a plain folder on your disk that you own outright.

---

## What "zero data collection" actually means

Three things that come standard in a commercial tool are absent from SCORPIOX CODE **by construction, not by configuration**:

1. **No phone-home.** There is no background heartbeat, no periodic check-in, and no "is this user still alive" ping. If you run SCORPIOX CODE with the network cable unplugged, it does the same thing it does online (against a local endpoint) — it never notices the difference.
2. **No telemetry pings.** There is no metrics endpoint, no crash-reporting service, and no anonymous usage counter. The binary has nowhere to send a number, so it never sends one.
3. **No user registration.** You do not sign up. There is no login, no email, no license activation, and no identity tied to your machine. The binary runs because you have it on disk, full stop.

None of these are settings you can flip off. There is nothing to flip — the code path does not exist. That distinction matters: a setting you can disable can be silently re-enabled, a default can drift, a checkbox can be reworded in a terms update. A capability that is not present cannot be switched on later without shipping new code you can inspect.

---

## The proof: we do not know how many users you are

The cleanest way to understand the difference is the one number every commercial AI coding tool tracks and SCORPIOX CODE fundamentally cannot: **active users.**

Tools like OpenCode and Cursor operate on a connected model. Every session, every prompt, every request is routed through or reported to vendor infrastructure, so they can answer "how many people ran the tool today" with a precise count. That number is the backbone of their analytics, their usage dashboards, and their growth metrics. It is also, necessarily, proof that they know *you* exist and *what* you did.

SCORPIOX CODE has no such mechanism. **We literally do not know how many active users there are.** We cannot produce an "active users today" figure because the only thing that touches the network is your chosen model endpoint, and that endpoint is yours to point anywhere — including a model running on your own LAN. There is no intermediary, no counter, and no record of you.

That is not a marketing claim. It is an architectural fact, and it is the single strongest privacy property a tool can have: **if we did not have to design a way to count you, we never built one.**

---

## 100% local filesystem ownership

Everything SCORPIOX CODE remembers about you lives in a single place: a folder on your local filesystem at `.scorpiox/sessions/`.

Every session, every prompt, every tool execution, and every log line is a plain file in a folder you can open with any text editor, copy, move, or delete. There is no database you cannot read, no encrypted blob you cannot access, and no copy stored somewhere else.

| What lives there | What it is |
|------------------|------------|
| `conversation.json` | The full verbatim transcript — every user message, assistant reply, tool call, and tool result, in order, with timestamps. |
| `messages/` | The same transcript split into one file per message, so a single exchange can be opened, grepped, or diffed in isolation. |
| `events/` and `events.jsonl` | A structured, machine-readable event log of what happened and when. |
| `traffic/` | Raw HTTP request and response data — the literal bytes sent to and from the model. |
| `agent.log` / `session.log` | Agent-level and runtime logs. |
| `stats.json` | Live session statistics, written locally for the status bar you are already looking at. |

Because it is just files, the ownership is total:

- **Read it** — open it in any editor or pager. No proprietary format, no "export" button you have to know exists.
- **Move it** — copy the folder to another machine and the session comes with it.
- **Delete it** — remove the folder and the data is gone. There is no server-side copy to request the deletion of, because there never was one.
- **Back it up** — the folder *is* the backup.

This is the same storage model that powers long-horizon agents on SCORPIOX CODE — see [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md) for how the sessions folder doubles as a queryable archive. The privacy point and the capability point are the same point: **the data is a file you own, not a record we hold.**

---

## Direct network control: your content goes where you point it

The only outbound traffic that carries your content goes to the **LLM endpoint you explicitly configure**. Nothing else. That endpoint is entirely yours to choose, which means you control where your code and prompts physically travel.

| Endpoint you configure | Where your content goes |
|------------------------|-------------------------|
| **Local inference** (`llama.cpp`, `vLLM`, or any OpenAI-compatible server on your LAN) | Nowhere off your network. Your code and prompts never leave your machine or your LAN. This is the strongest configuration: literally zero external content egress. |
| **A self-hosted or on-prem endpoint** | Only to infrastructure you operate. You decide who can see it and how it is logged. |
| **A vendor API you choose** | Only to the endpoint and vendor you selected, for the request you sent. You opted in to that specific transfer, for that specific call. |

The through-line is control: in every case, the destination is a line in your configuration file, not a hard-coded address the tool reaches for on its own. If you do not point the tool at a remote endpoint, there is no remote endpoint to reach. The full cascade of how that endpoint — and every other setting — gets resolved and layered across your machine, project, and session is documented in the configuration reference.

> **Rule of thumb:** the most private configuration is local inference. Run the model on your own hardware and the "network" your content touches is your own machine.

---

## The two switches that prove it is off

For the two features that *could* have carried information off-machine, SCORPIOX CODE ships them **disabled by default** and makes them strictly opt-in. You can see them resolved in your configuration:

| Setting | Default | What it does when you enable it |
|---------|---------|---------------------------------|
| `USAGE_TRACKING` | `0` (off) | Would report per-call token usage to a usage endpoint. Off by default — nothing is reported unless you turn it on. |
| `EMIT_SESSION_TRACKING` | `0` (off) | Would stream full conversation events to a session endpoint. Off by default — and it is disabled out of the box *specifically because it would transmit actual conversation content.* |

Both are `0` out of the box, and neither sends anything until you explicitly set it to `1`. That is the entire telemetry surface of the product, and it is closed by default. If you want a configuration that can never phone home, leave both at their defaults and point your endpoint at local inference — at that point there is nothing to turn off because there is nothing left to send.

---

## How the rest of the field compares

| Dimension | SCORPIOX CODE | Typical connected AI coding tools (OpenCode, Cursor, and the like) |
|-----------|---------------|---------------------------------------------------------------------|
| **User registration** | None. The binary runs because you have it. | An account is the entry point; your identity is the record key. |
| **Do we know you exist?** | No. No registration, no device ID, no anonymous counter. | Yes — the account *is* the tracking record. |
| **Active-user count** | Unknown by design. There is no counter. | Known and reported; it is the core of their analytics. |
| **Telemetry / usage pings** | Off by default, opt-in, destination of your choosing. | On by default, reported to vendor infrastructure. |
| **Where your content lives** | A plain folder on your disk you can read, move, and delete. | Vendor storage you can only see through an export. |

No account, no counter, no phone-home, no active-user number — because there is no mechanism that would let us count you. What you get instead is the only guarantee a tool can make without asking for your trust: **everything you can see is everything there is.**

---

## Related

- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md)
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
