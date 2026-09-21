# Privacy Architecture and Zero Data Collection Guarantee

Most AI coding tools ship a privacy policy. SCORPIOX CODE ships an architecture that makes the policy a non-issue.

There is no account to create, no server that knows you exist, and no background process that phones home. The reason we can guarantee **zero data collection and zero telemetry** is not that we promise to delete what we collect — it is that there is no collection mechanism to delete. The data never leaves the machine you run it on, except for the model tokens you deliberately send to an endpoint you chose.

This page explains what that guarantee means in practice, where your data actually lives, and how to verify the whole thing yourself.

> **The whole idea in one line:** your prompts, your code, and your tool output go exactly one place you point them — the model endpoint in your config — and everything else (sessions, logs, history) is a plain folder on your disk that you own outright.

---

## What "zero data collection" actually means

Three things that come standard in a commercial tool are absent from SCORPIOX CODE by construction, not by configuration:

1. **No phone-home.** There is no background heartbeat, no periodic check-in, and no "is this user still alive" ping. If you run SCORPIOX CODE with the network cable unplugged, it does the same thing it does online (against a local endpoint) — it never notices the difference.
2. **No telemetry pings.** There is no metrics endpoint, no crash-reporting service, and no anonymous usage counter. The binary has nowhere to send a number, so it never sends one.
3. **No user registration.** You do not sign up. There is no login, no email, no license activation, and no identity tied to your machine. The binary runs because you have it on disk, full stop.

None of these are settings you can flip off. There is nothing to flip — the code path does not exist. That distinction matters: a setting you can disable can be silently re-enabled, a default can drift, a checkbox can be reworded in a terms update. A capability that is not present cannot be switched on later without shipping new code you can inspect.

---

## The proof: we do not know how many users you are

The cleanest way to understand the difference is the one number every commercial AI coding tool tracks and SCORPIOX CODE fundamentally cannot: **active users.**

Tools like OpenCode and Cursor operate on a connected model. Every session, every prompt, every request is routed through or reported to vendor infrastructure, so they can answer "how many people ran the tool today" with a precise count. That number is the backbone of their analytics, their usage dashboards, and their growth metrics. It is also, necessarily, proof that they know *you* exist and *what* you did.

SCORPIOX CODE has no such mechanism. We literally do not know how many active users there are. We cannot produce an "active users today" figure because the only thing that touches the network is your chosen model endpoint, and that endpoint is yours to point anywhere — including a model running on your own LAN. There is no intermediary, no counter, and no record of you.

That is not a marketing claim. It is an architectural fact, and it is the single strongest privacy property a tool can have: **if we did not have to design a way to count you, we never built one.**

---

## 100% local filesystem ownership

Everything SCORPIOX CODE remembers about you lives in a single place: a folder on your local filesystem at `.scorpiox/sessions/`.

Every session, every prompt, every tool execution, and every log line is a plain file in a folder you can open with any text editor, copy, move, or delete. There is no database you cannot read, no encrypted blob you cannot access, and no copy stored somewhere else.

| What lives there | What it is |
|------------------|------------|
| **The full conversation** | Every user and assistant message, every tool call and tool result — verbatim, nothing summarized or truncated. |
| **Structured events** | A machine-readable event log: session start and end, tool usage, timestamps. |
| **Agent log** | The agent-level record of requests, responses, and tool results. |
| **Traffic captures** | Raw request/response data for the model endpoint, when traffic logging is enabled. |
| **Session metadata** | Model, provider, duration, turn count, token totals. |

Because the session is a real folder of plain files, ownership is total and unambiguous:

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

The through-line is control: in every case, the destination is a line in your configuration file, not a hard-coded address the tool reaches for on its own. If you do not point the tool at a remote endpoint, there is no remote endpoint to reach.

> **Rule of thumb:** the most private configuration is local inference. Run the model on your own hardware and the "network" your content touches is your own machine.

---

## The two switches that prove it is off

For the two features that *could* have carried information off-machine, SCORPIOX CODE ships them **disabled by default** and makes them strictly opt-in. You can see them resolved in your configuration:

| Setting | Default | What it does when you enable it |
|---------|---------|---------------------------------|
| `USAGE_TRACKING` | `0` (off) | Would report per-call token usage to a usage endpoint. Off by default — nothing is reported unless you turn it on. |
| `EMIT_SESSION_TRACKING` | `0` (off) | Would stream session events to a session endpoint. Off by default — nothing is streamed unless you turn it on. |

Two things to notice about how these are built:

- **Opt-in, not opt-out.** The default value in the shipped configuration is `0`. You must actively set the value to `1` to enable either one. A feature that is off unless you ask is categorically different from one that is on unless you object.
- **You choose the destination.** Even when enabled, both are pointed at an endpoint you configure. There is no fixed vendor sink baked in that you cannot redirect.

If you want the guarantee to be *yours*, not ours, this is the place to look. Resolve your effective configuration and confirm both keys are `0`. When they are, the only network path with your content in it is the model endpoint you set.

---

## How the rest of the field compares

| Dimension | SCORPIOX CODE | Typical connected AI coding tools (OpenCode, Cursor, and the like) |
|-----------|---------------|---------------------------------------------------------------------|
| **User registration** | None. The binary runs because you have it. | An account is the entry point; your identity is the record key. |
| **Do we know you exist?** | No. No registration, no device ID, no anonymous counter. | Yes — the account *is* the tracking record. |
| **Active-user count** | Unknown by design. There is no counter. | Known and reported; it is the core of their analytics. |
| **Telemetry / usage pings** | Off by default, opt-in, destination of your choosing. | On by default, reported to vendor infrastructure. |
| **Phone-home / heartbeat** | None. Offline operation is indistinguishable from online. | Present; the tool checks in on a schedule. |
| **Where your content goes** | Only the endpoint in your config (can be your own LAN). | Routed through or reported to vendor infrastructure. |
| **Session storage** | A plain folder on your disk you fully own. | Vendor-side, subject to their retention policy. |

The pattern is the same in every row: SCORPIOX CODE is built around the assumption that *you* are the endpoint owner, while the connected tools are built around the assumption that the *vendor* is one.

---

## How to verify it yourself

You do not have to take the guarantee on faith. Every claim above is checkable on your own machine:

1. **Confirm the switches are off.** Resolve your effective configuration and check that `USAGE_TRACKING` and `EMIT_SESSION_TRACKING` are `0`. That is the entire telemetry surface, and it is closed by default.

2. **Watch the network, not the docs.** Run a session against a **local** endpoint and capture outbound traffic (a firewall log, a packet capture, or your router's logs). The only content-bearing connection should be to the local model endpoint you configured. There is nothing else to see, because there is nothing else to see.

3. **Run fully offline.** Point the endpoint at a local model, unplug the network, and work. The tool behaves exactly as it does online. A tool that needs a vendor server to "phone home to" would not — the fact that it does not is the point.

4. **Own the folder.** Open `.scorpiox/sessions/`. Every byte of your session history is a readable file. Copy the folder to verify you can carry the data anywhere; delete a folder to verify nothing is retained server-side (there was never a server-side copy).

---

## What this means in practice

The guarantee has a cost, and it is worth naming it honestly:

- **No cloud sync.** Because there is no server holding a copy, there is no "your session on another machine" to sign into. To move work to another machine, you move the folder.
- **No server-side recovery.** If you delete a session folder, it is gone — permanently, with no vendor backup to restore it from. That is not a limitation of the design; it is the design. You own the data, which means you own its deletion.
- **No vendor-side audit trail.** There is no central log of "what users did," because there is no central place to log it. Accountability for the data is yours, not ours.

These are the trade-offs of putting ownership where it belongs. For a tool that handles your actual source code, the exchange is usually worth it: **the strongest privacy guarantee is the one where there is no one to leak, nothing to subpoena from a vendor, and no data to be breached — because the data is, and only is, on the machine you control.**
