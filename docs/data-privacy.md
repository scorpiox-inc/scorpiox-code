# Privacy Architecture and Zero Data Collection Guarantee

You hand an agent your code, your prompts, your credentials, and the raw HTTP traffic of your work — and then you are left with a question that no marketing page wants to answer honestly: *where does all of that go?* Most agent tools answer with a "we never store your code" checkbox and a privacy policy you will never read. The real question is narrower and harder: does the tool *know* you are using it, and can it report you back to its makers?

SCORPIOX CODE is built so that the honest answer is **no**. There is no phone-home, no telemetry by default, and no user registration. This page explains the architecture that makes that true, what "zero data collection" actually means in practice, and the one number that proves it.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** SCORPIOX CODE runs entirely on your machine, talks only to the model endpoints you configure, writes every session to your own filesystem, and ships with both telemetry switches turned **off** — so by default it sends nothing anywhere that is not a model you pointed it at.

---

## The number that proves it

Here is the most concrete thing a privacy guarantee can produce:

> **We do not know how many people use SCORPIOX CODE.**

We genuinely cannot tell you. There is no sign-in, no device fingerprint, no install counter, and no analytics ping. If you run SCORPIOX CODE today, there is nothing on the other end of the wire that learns you ran it.

That is the opposite of how the mainstream agent tools work. OpenCode, Cursor, and the rest track every user: they know their active-user counts, their session durations, and what models and providers are being exercised. They can tell you the number of active users because they **count them** — and counting a user means collecting something about that user.

SCORPIOX CODE has no such count, because it has no mechanism to count. "We cannot tell you how many active users we have" is not a humble brag; it is the direct, observable consequence of having no telemetry. A tool that does not track you cannot report you. That is the guarantee, and the architecture below is what makes it hold.

---

## The architecture

Zero data collection is not a policy promise; it is a property of how the product is built. Three decisions do the work.

### 1. It ships with no telemetry, and the switches are off

A common "trust me" pattern is to build a telemetry pipeline and call it "off by default" while leaving a hidden beacon running. SCORPIOX CODE does not take that risk. The only two reporting paths in the product — **usage tracking** (token counts) and **session-event emission** (per-message events) — both start **disabled** out of the box.

| Mechanism | What it would report | Default |
|-----------|----------------------|---------|
| **Usage tracking** | Token counts per model, rate-limit utilization | **Off** |
| **Session-event emission** | Per-message session events | **Off** |
| **Everything else** | — | No network call at all |

Nothing in the normal agent loop phones home. The model calls you make are the only thing that leaves the machine, and they go to the endpoint you configured (see the next section). If you do not turn the two switches on, there is no reporting path active — and there is nothing to report *to*, because there is no account to attach it to.

These two are **opt-in, not opt-out**. You must deliberately enable either one, and you control exactly where it points (`USAGE_API_URL`, `EMIT_SESSION_API_URL`). That is the inverse of the usual "privacy policy with an opt-out link" pattern.

### 2. Every session lives on your filesystem

Nothing about a session — the prompt, the model's responses, the tool calls, the logs, even the raw request/response traffic — is sent to SCORPIOX CODE servers. It is written to your local filesystem under `.scorpiox/sessions/`, in a directory per session:

```
.scorpiox/
└── sessions/
    └── <session-id>/
        ├── conversation.json     # the full transcript
        ├── events/               # per-event records
        ├── messages/             # per-message files
        ├── agent.log             # agent activity log
        ├── stats.json            # live local stats (stays local)
        └── traffic/              # raw HTTP request/response capture
```

Two things matter here. First, **your data never leaves your machine** unless you move it. The session you just had is a set of files you own, in a location you control, that you can inspect, copy, or delete at will. Second, the agent is built to *read back* these files — its own design treats `.scorpiox/sessions/<id>/` as the source of truth, so the data has a local home rather than a remote one.

That is the **100% local filesystem ownership** guarantee: sessions are not records in someone else's database; they are files on your disk.

### 3. The only network traffic goes where you point it

SCORPIOX CODE does not have its own API that your code "calls home" to. The network calls the product makes on your behalf go **exclusively** to the LLM endpoints you have explicitly configured:

- **Local inference** — a `llama.cpp` or `vLLM` server on your LAN, so the model itself never leaves your network.
- **Your chosen provider endpoint** — the API base URL and key you supply for the provider you signed up with.

There is no intermediary relay and no SCORPIOX CODE proxy sitting between you and the model. The traffic-capture feature (which logs every request and response) writes to `.scorpiox/sessions/<id>/traffic/` **locally** so you can audit exactly what was sent and received — not so a vendor can see it. If you can read your traffic capture, the data is yours; if it were going to a third party, the capture would not be the whole story.

---

## What this looks like in practice

| Question you should ask any agent tool | SCORPIOX CODE |
|----------------------------------------|---------------|
| Do I need to create an account to use it? | No. There is no SCORPIOX CODE account. |
| Does it report usage when I run it? | Not by default. Both reporting switches ship **off**. |
| Where does my conversation go? | `.scorpiox/sessions/<id>/`, on your disk. |
| Where do my model calls go? | Only to the endpoint you configured (local or your provider). |
| Can the vendor see my prompts or code? | No. They are never sent to a SCORPIOX CODE server. |
| How do they know I'm a user? | They do not — and that is the point. |
| Can I audit every byte sent/received? | Yes, the local traffic capture logs it all. |

---

## What "zero data collection" is *not*

Honest about the edges:

- **Your model endpoint sees the model traffic.** If you point SCORPIOX CODE at a hosted provider, *that provider* receives the prompts and returns the responses — that is inherent to using a hosted model, not a property of this product. Running local inference removes even that.
- **The opt-in switches exist.** If you deliberately enable usage tracking or session-event emission, data flows to the URL you configure. The guarantee is that this is **off by default and yours to direct**, not that no such path exists in the code.
- **Local files are your responsibility.** Because sessions live on your filesystem, deleting a session removes it from your machine — and nothing is kept on a server to restore it. That is the flip side of ownership.

The guarantee is precise: by default, on a normal run, SCORPIOX CODE collects nothing and sends nothing except the model calls you explicitly made to the endpoints you explicitly chose.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Traffic Logging](traffic-logging.md)
