# Privacy Architecture and Zero Data Collection Guarantee

Most developer tools quietly build a second product out of you. Usage counters, feature adoption, crash dumps, "active user" dashboards. You can usually find a checkbox somewhere that says "opt out of telemetry," but the default is on, the data is already leaving your machine, and the only question is what happens to it afterwards.

SCORPIOX CODE takes the opposite position, and this page explains what that means in practice.

**The short version:** SCORPIOX CODE collects zero data, runs zero telemetry, and phones home to nobody. We literally do not know how many active users you, as a community, have.

---

## The proof: we cannot count our own users

This is the single most useful way to check any "private by default" claim, so here it is first.

OpenCode, Cursor, and the other mainstream agent tools know exactly how many users are active. Every one of them phones home. They track each user, each session, and each request, which is how they build the active-user counts and adoption charts their teams look at every morning. If a tool reports an active user count, it is counting *you*.

SCORPIOX CODE has no phone-home mechanism, no telemetry pings, and no user registration. There is no account to sign in to, no device ID, and no heartbeat. Which means:

- **We do not know how many people use SCORPIOX CODE.** Not approximately, not as an estimate — we have no channel through which that number could ever reach us.
- **We cannot know who you are**, what you build, or which models you run.
- **Turning it off is not a setting you have to remember.** There is nothing to turn off. The default *is* the guarantee.

If a vendor can tell you its monthly active users, it can tell them about your usage. That is the whole test.

---

## What this is not: a promise you have to configure

Some tools ship telemetry on by default and ask you to find a setting, flip a flag, and hope you understood what the flag disabled. SCORPIOX CODE does not ship a telemetry pipeline at all.

The only network traffic SCORPIOX CODE ever initiates is the traffic **you** explicitly pointed it at: the LLM endpoints you configured. Every other capability that a tool might use to "help" — usage reporting, session reporting, adoption tracking, error collection — is either absent or, where it exists as an opt-in capability, **disabled by default** and off unless you deliberately enable it yourself.

| Capability | Default | Notes |
|------------|---------|-------|
| Phone-home / heartbeat | **Never** | No mechanism exists. |
| User registration / accounts | **None** | No sign-in, no device ID. |
| Telemetry pings | **Off** | No background collection. |
| Token usage reporting | **Off** | Disabled by default; only active if you explicitly enable it. |
| Session event reporting | **Off** | Disabled by default; this one transmits real conversation content, so it stays off unless you ask. |

The defaults are the contract. You do not need to read this page to be private — you need to read it only if you want to understand *why* it is true.

---

## 100% local filesystem ownership

Everything SCORPIOX CODE does lives on your machine, in a plain folder you can open with any file tool.

Each session is a directory under `.scorpiox/sessions/`. Nothing about a session exists anywhere else. If you can read the folder, you have the whole story:

| File / folder | What it holds |
|---------------|---------------|
| `conversation.json` | The full conversation, verbatim. Every message, tool call, and result. |
| `commands.json` | The commands issued during the session. |
| `config-snapshot.txt` | The exact active configuration at the time, so the session is reproducible. |
| `events.jsonl` / `events/` | A machine-readable event stream: tool calls, results, state changes. |
| `messages/` | Individual message records. |
| `traffic/` | **Every** request and response the session made, captured raw — request bodies, response bodies, headers, and a per-call log with byte sizes. |
| `stats.json` | Local session statistics. |
| `meta.json` | Session metadata: id, start time, working directory, model, provider, profile. |

Three consequences fall out of this:

- **You own the record, completely.** No portion of your session is held by anyone but you. Delete the folder and it is gone. Copy it and it moves with you.
- **It is inspectable end to end.** The `traffic/` capture means you can open any call and read exactly what went out and what came back. There is no "trust us, it only sent X."
- **It is local by default in version control too.** SCORPIOX CODE adds `.scorpiox/sessions/` to `.gitignore` where applicable, so your session history does not get swept into a commit and pushed to a remote.

The same principle applies to how the agent works: sessions are filesystem-native data the agent reads and writes with ordinary file tools, not a server-side state you rent. See [Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md) for how this enables long-horizon work.

---

## Direct network control: only the endpoints you choose

The only outbound network calls SCORPIOX CODE makes are the model calls you configured. That is the entire network surface.

You point SCORPIOX CODE at the LLM endpoint you want, and that is all it talks to:

- **Local inference on your LAN.** Point the OpenAI-compatible base URL at a local runtime — llama.cpp, vLLM, LM Studio, Ollama — and the model calls stay inside your network. Your prompts and code never cross it.
- **Your chosen hosted endpoint.** Use a provider you already trust and a key you already own, with the URL you set.
- **Any other endpoint** you configure the same way.

There is no separate "SCORPIOX CODE cloud" your data must pass through, no proxy layer that sees your requests, and no fallback endpoint the tool phones when it is idle. If you can enumerate the endpoints in your configuration, you have enumerated the network. SCORPIOX CODE talks to the model you told it to talk to, and to nothing else.

> **The honest boundary.** SCORPIOX CODE cannot make the model vendor stop doing what the model vendor does. If you point it at a hosted endpoint, that provider receives your requests, because you are the one calling it and you are the one paying it. What SCORPIOX CODE guarantees is that *it* adds nothing to that picture — no extra endpoint, no copy, no side channel.

---

## Putting it together

| Question a vendor gets asked | OpenCode / Cursor-style tool | SCORPIOX CODE |
|------------------------------|------------------------------|---------------|
| Do you track active users? | Yes. | **We don't know how many there are.** |
| Is telemetry on by default? | Usually yes. | **No telemetry exists to turn on.** |
| Where does my session live? | Vendor's servers. | **Your filesystem, in a folder you can read.** |
| Who does it talk to on the network? | Model + its own backends. | **Only the endpoint you configured.** |
| How do I opt out? | Find the setting. | **There is nothing to opt out of.** |

That last row is the whole philosophy. Privacy in SCORPIOX CODE is not a preference you select in a menu; it is the shape the software has when it is not doing anything else. You are the only one who knows what you did, with what model, and the only copy of it is the one you already hold.

---

*Docs for SCORPIOX CODE @ `b59223a`.*
