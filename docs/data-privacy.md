# Privacy Architecture and Zero Data Collection Guarantee

SCORPIOX CODE is built on a simple, radical premise: **it never phones home, never pings a telemetry endpoint, and never requires you to register an account.** Every session, prompt, tool execution, and log stays on your own filesystem. The only network traffic it ever makes is the one you explicitly pointed it at.

This page explains what that guarantee actually means, where your data lives, and how to verify none of it leaves your machine.

Source of truth: `sx.c`, `scorpiox-emit-session.c`, and the config cascade at commit `5fd054b`.

---

## The one-line proof

We **literally do not know how many active users we have.**

Most AI coding tools — OpenCode, Cursor, and most others in this category — track every single user. They phone home on launch, count active installations, and report usage back to a central server. That means they can tell you "1.2 million users this month."

SCORPIOX CODE can't do that. It has **no phone-home mechanism, no telemetry pings, and no user registration.** There is no backend that receives a "hello, I'm running" from your machine, so there is no dashboard counting heads. If we can't count you, we can't track you. That isn't a marketing claim — it's an architectural constraint: the code simply has no channel to report.

> **Why does this matter to you?** Because the tools that count their users have, by design, a pipe from your machine to their servers. SCORPIOX CODE does not open that pipe. What isn't sent can't be logged, aggregated, or leaked.

---

## What "zero data collection" means here

| Capability | SCORPIOX CODE |
|------------|---------------|
| Mandatory account / sign-in | **None** — you can run it with no account at all |
| Telemetry pings on launch | **None** — no phone-home, no install counters |
| Usage analytics sent to vendor | **None** |
| Prompt / conversation upload | **None** — conversations stay local |
| Code content upload | **None** — your files are read locally, not shipped |
| Network calls | **Only** to the LLM endpoint *you* configured |
| Session storage | **Only** your local filesystem |

The distinction that matters: SCORPIOX CODE can *optionally* send session event data to a URL, but that feature is **off by default** and requires you to deliberately enable it. Out of the box, there is no such traffic at all.

---

## 100% local filesystem ownership

Everything SCORPIOX CODE records for a session lives in a directory under your local filesystem:

```
.scorpiox/sessions/<session-id>/
```

Within it you'll find the conversation history, per-message event files, status/telemetry-of-local-state dumps, and (only when you turn on traffic capture) the raw HTTP request/response pairs. These files are:

- **Written to your disk, not to a server.** They are local artifacts, not uploads.
- **Yours to inspect.** Open them, diff them, or delete them — they're plain files.
- **Yours to delete.** Removing `.scorpiox/sessions/<id>/` removes that session's record entirely.

There is no shadow copy, no "synced to the cloud," no backup we hold. If you wipe the directory, the data is gone.

```bash
# List your local sessions
ls .scorpiox/sessions/

# Inspect one session's conversation
cat .scorpiox/sessions/<session-id>/conversation.json

# Delete a session entirely (local-only, irreversible)
rm -rf .scorpiox/sessions/<session-id>
```

Because there is no server-side copy, "data deletion" is trivial and total: delete the directory.

---

## You own the network: direct endpoint control

The **only** outbound network connections SCORPIOX CODE makes are to the LLM endpoints you explicitly configure. No default endpoint is baked in for your benefit to "just work" with a hosted service.

That means you decide, completely, where requests go:

- **Local inference on your machine** — point it at a llama.cpp or vLLM server running on `localhost`.
- **Inference on your LAN** — a dedicated GPU box, an internal vLLM cluster, a private SGLang deployment.
- **A provider you choose** — any OpenAI-compatible, Anthropic, Gemini, or other endpoint you configure.

Nothing else leaves. There's no secondary "telemetry" host, no update-check beacon, no usage-reporting service riding along in the background. The network surface of the tool is exactly the endpoints you wrote into your config.

> See [Using the OpenAI Provider](openai-provider.md) and [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how endpoints and profiles are set.

---

## The one opt-in switch (off by default)

For transparency, here is the *only* place a session-event network call is possible: the session event telemetry feature, gated by the `EMIT_SESSION_TRACKING` setting.

- **Default: `0` (disabled).** Out of the box, this feature does nothing and sends nothing.
- **Enabled: `1`.** Only when you set it to `1` does it send session event data to `EMIT_SESSION_API_URL` (a URL you control).

This is the inverse of how most tools behave. Their telemetry is on by default and buried in the ToS; this one is off by default and requires an explicit setting to turn on. If you never set it, there is no such network traffic.

```
# Default (shipped) — telemetry off, nothing sent
EMIT_SESSION_TRACKING=0
```

---

## Verify it yourself

You don't have to take this on faith. A few things you can check on your own machine:

1. **Watch the network.** Run SCORPIOX CODE with a packet capture (e.g. `tcpdump` on your machine) and confirm the only outbound connections are to the endpoint you configured.
2. **Look at the disk.** Confirm sessions appear under `.scorpiox/sessions/` and nowhere else.
3. **Confirm the default.** The config default for `EMIT_SESSION_TRACKING` is `0`. If you haven't set it, no session telemetry is active.
4. **Run air-gapped.** Point it at a local inference server, disconnect from the internet entirely, and use it. It works, because it was never designed to need anything else.

If any of those steps surprise you — an unexpected host, a file outside your filesystem — that would be a bug worth reporting, not expected behavior.

---

## TL;DR

- **No account, no phone-home, no telemetry pings** — we genuinely can't count our active users.
- **All session data stays on your local filesystem** (`.scorpiox/sessions/`) and is yours to inspect or delete.
- **The only network calls are to the LLM endpoints you configure** — local, LAN, or a provider you choose.
- **The one optional telemetry switch is off by default** and must be explicitly enabled.

You can run SCORPIOX CODE fully air-gapped, against your own inference, and it will behave exactly as it does on the open internet — because it was built to have nothing to send in the first place.
