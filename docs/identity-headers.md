# Session Identity Headers

Every request SCORPIOX CODE sends to an OpenAI-compatible endpoint can carry a small set of **identity headers**: one for the session you are in, one for the conversation lineage that session belongs to, plus anything extra you choose to add. Providers use them for request grouping, session affinity, cache correlation, and their own analytics — and because you control exactly what goes out, nothing is hidden and nothing is forced.

The whole feature is one config key: **`OPENAI_EXTRA_HEADERS`**. It ships with a sensible default (two `X-Sx-*` headers naming your session and thread), it accepts any additional headers you want, and it is the single switch for turning identity headers off completely.

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** with `PROVIDER=openai`, SCORPIOX CODE stamps every outgoing request with `X-Sx-Session-Id` and `X-Sx-Thread-Id` — a per-session GUID and a per-conversation-lineage GUID — and lets you add, rename, or remove all of it through one key.

---

## What gets sent by default

Out of the box, an OpenAI-compatible request leaves your machine with these headers attached:

```
X-Sx-Session-Id: 8f14e45f-ceea-467f-a1c5-5f0e2d3f9a1b
X-Sx-Thread-Id:  c4ca4238-a0b9-4e2a-a7c9-33e0f0d5a1d2
```

Both values are **UUIDs generated on your machine**. Nothing is fetched, nothing is registered, and the values have no meaning outside your own session folder and whatever the receiving endpoint does with them.

| Header | Where the value comes from | How long it stays the same |
|--------|----------------------------|----------------------------|
| `X-Sx-Session-Id` | A UUIDv4 generated when the session is created, stored in that session's `meta.json` | One session — a new session folder means a new value |
| `X-Sx-Thread-Id` | A UUIDv4 naming the *conversation lineage* the session belongs to | The whole lineage — see [Session vs. thread](#session-vs-thread-why-there-are-two-ids) |

Both GUIDs live in the session's `meta.json` next to the model, provider, and profile, so a session resumed later carries the same identity it had when it was created. If you open a session that predates this feature, SCORPIOX CODE generates GUIDs for it once and writes them back, so older sessions get stable identity too.

---

## The one key: `OPENAI_EXTRA_HEADERS`

`OPENAI_EXTRA_HEADERS` is a comma-separated list of `Name:value` pairs. It applies to the **OpenAI provider only** (`PROVIDER=openai` — the provider for llama.cpp, vLLM, SGLang, Ollama, LM Studio, Azure, and any other OpenAI-compatible endpoint; see [Using the OpenAI Provider](openai-provider.md)). No other provider reads it.

```
OPENAI_EXTRA_HEADERS=X-Sx-Session-Id:{session_guid},X-Sx-Thread-Id:{thread_guid}
```

That is the shipped default: two headers, named exactly as above, with their values filled in per request. Change the names, add more pairs, or empty the key — the format stays the same either way.

### Placeholders

Three placeholders are supported, each substituted **per request**, just before the request is sent. Anything else in braces passes through untouched:

| Placeholder | Substitutes to | Notes |
|-------------|----------------|-------|
| `{session_guid}` | The UUIDv4 for the current session | Always present for a valid session. This is the value behind `X-Sx-Session-Id`. |
| `{thread_guid}` | The UUIDv4 of the current conversation lineage | Always present for a valid session. This is the value behind `X-Sx-Thread-Id`. |
| `{session_id}` | The session's human-readable ID, the same name as the session folder (for example `2026_10_02_quiet_tesla`) | Populated when session file logging has been initialized for the run. |
| *(unknown tokens)* | Left as literal text | `{anything_else}` is not a known placeholder and is sent verbatim. |

A pair whose value ends up empty is dropped, so a header you configured to a placeholder that resolves to nothing is not sent as `Header:` with an empty value — the pair disappears quietly.

> **Literal-token trap.** Unknown placeholders are passed through as literal text. If you use `{session_id}` in an environment where the session ID has not been populated, the header is sent with the literal string `{session_id}` as its value — recognizable at the receiving end, but probably not what you intended. Prefer `{session_guid}` and `{thread_guid}`, which are always populated for a valid session.

### Custom headers

Add anything you like after the defaults — static values or placeholder-bearing values:

```
OPENAI_EXTRA_HEADERS=X-Sx-Session-Id:{session_guid},X-Sx-Thread-Id:{thread_guid},X-Team:platform,X-Cost-Center:{session_guid}
```

Common reasons to add a pair:

- **A tenant or routing tag** your gateway reads to steer requests (`X-Team:platform`).
- **A second copy of an identity value** under a name your infrastructure already expects (`X-Correlation-Id:{session_guid}`).
- **Endpoint-specific flags** a self-hosted engine or middleware layer wants on every call.

The list is processed left to right. Whitespace around a pair is tolerated (`Name: value` works), so both `X-Team:platform` and `X-Team: platform` are fine.

---

## Session vs. thread: why there are two IDs

Two identifiers sound redundant until you watch a long task run. They answer different questions:

- **The session GUID answers "which session folder is this?"** It is generated fresh for every new session — `/clear`, a fresh launch, an automatic compaction, all produce a new one. It matches the session's entry in `meta.json` and the folder under `.scorpiox/sessions/`.
- **The thread GUID answers "which conversation lineage is this?"** It is the continuity anchor. A brand-new session starts (roots) a new thread, and a session that has been compacted or resumed **keeps the same thread GUID** as the session it continued from, so every request across that whole run reports one consistent thread. Legacy sessions without a thread GUID fall back to their session GUID when the lineage is carried forward.

The practical difference shows up in what a provider sees:

| Event | `X-Sx-Session-Id` | `X-Sx-Thread-Id` |
|-------|--------------------|-------------------|
| New session started | New UUID | New UUID (new lineage) |
| Auto- or manual compaction (`/compact`) | New UUID (new session folder) | **Unchanged** — the lineage carries over |
| `/resume` of an earlier session | New UUID for the resumed session | **Unchanged** — the lineage carries over |
| `/clear` | New UUID | New UUID (fresh lineage) |

So a provider that groups by session sees many short sessions; a provider that groups by thread sees one continuous work stream per task, even when the session underneath it has been swapped out several times. That is what makes the thread GUID the right key for per-task correlation, and the session GUID the right key for per-session bookkeeping.

Both values are plain UUIDs generated locally with a cryptographically-seeded generator. They contain no username, no machine name, no path, and no project name.

---

## Hardening: what the header path refuses to do

Every header in the list passes through validation before it is attached. The checks are strict on purpose:

- **CR/LF injection is rejected.** A name or a value containing a carriage return or newline is dropped, and the header never reaches the wire. This closes the classic header-injection hole where a crafted value could smuggle in extra headers or split the request.
- **Header names may not contain whitespace.** A name with an embedded space or tab is dropped along with its value.
- **`Content-Type` and `Authorization` cannot be overridden.** These two are set by the request path itself (`Content-Type: application/json`, and `Authorization: Bearer ...` when a key is configured). An attempt to set either through `OPENAI_EXTRA_HEADERS` is ignored with a notice on the diagnostic stream, so your auth header and body framing always stay intact.
- **Empty segments are skipped.** Trailing commas or doubled commas in the list are tolerated rather than sent as broken headers.
- **The list is size-bounded.** The resolved header list is capped at 2048 characters and each pair at 512, and the request's header table holds a fixed maximum (32) — a list that would overflow is stopped with a notice rather than silently truncated.

The behavior is fail-safe in the boring direction: anything questionable is dropped or refused, and the request still goes out with the headers that did validate.

---

## Opting out: sending no identity headers at all

Setting the key to empty disables the feature entirely — no `X-Sx-Session-Id`, no `X-Sx-Thread-Id`, no custom headers. The request goes out with exactly what the endpoint requires and nothing more.

### The quick way

Add one line to your project config:

```ini
# .scorpiox/scorpiox-env.txt
OPENAI_EXTRA_HEADERS=
```

An empty value means "send nothing extra" — the key's comment in `scorpiox-env.txt` says exactly that: *Empty = send nothing (opt out of session tracking).*

### The durable way

Put the same line in the tier that matches how long you want the opt-out to last. See [Configuration and Profiles](scorpiox-env.md) for the full cascade — highest tier wins per key:

| You want… | Put `OPENAI_EXTRA_HEADERS=` in |
|-----------|-------------------------------|
| This project only | `.scorpiox/scorpiox-env.txt` (or `.claude/scorpiox-env.txt`) in the project |
| Every project on this machine, for you | `~/.claude/scorpiox-env.txt` |
| Every user of this install | `scorpiox-env.txt` next to the installed binaries |
| A named profile you flip on and off | `scorpiox-env/<name>.txt` profile overlay |

You can also set it as a real OS environment variable, which outranks every file:

```bash
OPENAI_EXTRA_HEADERS= ./scorpiox
```

And you can edit the key interactively with `scorpiox-config` (the TUI editor) or non-interactively:

```bash
scorpiox-config --set OPENAI_EXTRA_HEADERS ""            # writes to the project tier
scorpiox-config --set OPENAI_EXTRA_HEADERS "" --level user
```

Profile switching is the tidy pattern for temporary opt-outs: keep a profile whose only delta is `OPENAI_EXTRA_HEADERS=`, and flip to it with `/profile <name>` (persistent) or `/use <name>` (this session only). Switch back the same way. See [Configuration and Profiles](scorpiox-env.md).

### What changes at the provider when you opt out

- **Your requests stop carrying `X-Sx-*` headers.** A provider that groups or correlates by those headers now sees requests with no session or thread marker from SCORPIOX CODE. Per-session grouping, per-task rollups, and any affinity built on them stop working at that provider.
- **Local behavior does not change.** Sessions, compaction, resumes, traffic logging, and everything on disk work exactly as before — the GUIDs still exist in `meta.json`; they are simply not attached to requests.
- **Other identity headers, if any, are provider-owned.** The opt-out affects only what `OPENAI_EXTRA_HEADERS` adds. Providers that add their own session headers do so regardless — see the comparison below.

> **A profile (or any higher tier) can silently re-enable it.** If you set the key empty in your project file but an active profile sets it to something non-empty, the profile wins and the headers go out. When headers "won't turn off," run `scorpiox-config --verbose` and check which tier is supplying the value.

---

## What a request looks like on the wire

For a session running with the default configuration:

```http
POST /v1/chat/completions HTTP/1.1
Host: localhost:8080
Content-Type: application/json
Authorization: Bearer <your key, when configured>
X-Sx-Session-Id: 8f14e45f-ceea-467f-a1c5-5f0e2d3f9a1b
X-Sx-Thread-Id: c4ca4238-a0b9-4e2a-a7c9-33e0f0d5a1d2
```

The identity headers travel alongside the standard ones, after them, and are yours to inspect: point `OPENAI_BASE_URL` at any endpoint that echoes request headers (a local debug server, or a logging reverse proxy in front of your engine) and the two `X-Sx-*` lines appear in the captured request with the UUIDs your session's `meta.json` holds.

---

## How other providers handle identity

The openai provider is the only one that takes its headers from `OPENAI_EXTRA_HEADERS`. Other providers manage identity themselves — fixed header names, values generated or configured per provider:

| Provider | Identity sent with every request | Who controls it |
|----------|----------------------------------|-----------------|
| **openai** | `X-Sx-Session-Id:{session_guid}`, `X-Sx-Thread-Id:{thread_guid}` by default; anything else you configure | **You** — via `OPENAI_EXTRA_HEADERS`, including the opt-out |
| **grok** | Fixed session-affinity headers: `x-grok-session-id` and `x-grok-conv-id` (the same generated UUID, held for the provider's lifetime) plus `x-grok-turn-idx`, and the same UUID as the `prompt_cache_key` field in the body to pin requests onto a warm KV cache | The provider — fixed names and values, no config surface |
| **opencode** | Required session headers: `x-opencode-session` (required on inference; auto-generated `ses_...` id, or your `OPENCODE_SESSION_ID` override), `x-opencode-request` (per-call), `x-opencode-client: cli`, `x-opencode-project: global` | The provider, with one override key (`OPENCODE_SESSION_ID`) |

Two things are worth taking from that table:

- **Grok's headers exist for the cache, not for you.** The session/conversation pair is what keeps consecutive turns landing on a warm prompt cache (measured on a 15.6K-token prompt: no session key means nothing is cached turn to turn; with the key, the full prompt is cached from the second turn). Those names cannot be renamed and cannot be suppressed from `OPENAI_EXTRA_HEADERS` — they are part of how the Grok provider talks to its endpoint. See [Using Grok Build Subscription](grok-provider.md).
- **OpenCode's session header is a requirement, not a decoration.** The endpoint rejects inference requests without `x-opencode-session`, which is why that provider generates one (or takes your `OPENCODE_SESSION_ID`) and sends it on every call. See [Using OpenCode Zen Subscription](opencode-provider.md).

If you need uniform identity headers across providers — the same two names on every backend — that is what `OPENAI_EXTRA_HEADERS` gives you on the openai provider, and why the default names are deliberately neutral (`X-Sx-*`) rather than tied to one vendor.

---

## Configuration reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `OPENAI_EXTRA_HEADERS` | text | `X-Sx-Session-Id:{session_guid},X-Sx-Thread-Id:{thread_guid}` | Extra HTTP headers attached to every outgoing request when `PROVIDER=openai`. Comma-separated `Name:value` pairs. Placeholders: `{session_guid}`, `{thread_guid}`, `{session_id}`. Empty = send nothing (opt out of session tracking). |

It lives in the **Provider & Authentication** section of the config editor and is visible when `PROVIDER=openai`. As with every key, it can be set in any cascade tier, in a profile overlay, or as an OS environment variable — the highest tier that sets it wins.

---

## Gotchas

- **It is one line, not a header file.** Everything is a single comma-separated value. A pair may not contain a comma in its value (the comma splits pairs), so keep values comma-free — UUIDs are.
- **Whitespace is trimmed around pairs, not inside names.** `X-Team: platform` is fine; a space inside the name is not.
- **Empty beats absent, and highest tier wins.** An empty value in a higher tier disables the headers even though a lower tier sets them; a non-empty value in a higher tier overrides an empty lower-tier value. Check the active profile first when behavior surprises you.
- **Unknown placeholders go out literally.** `{session_guid}` and `{thread_guid}` are always populated for a valid session; `{session_id}` depends on session file logging being initialized, and anything else is sent as-is.
- **Reserved headers are ignored, not an error.** Attempting to set `Content-Type` or `Authorization` produces a notice on the diagnostic stream and the pair is dropped; the request proceeds normally.
- **The key only affects `PROVIDER=openai`.** If you switch profiles to grok, opencode, or a subscription provider, their own identity headers apply and `OPENAI_EXTRA_HEADERS` is dormant until you switch back.

---

## Related

- [Using the OpenAI Provider](openai-provider.md) — the provider these headers travel on.
- [Configuration and Profiles](scorpiox-env.md) — the cascade, tiers, and `/profile` vs `/use`.
- [Privacy Architecture and Zero Data Collection](data-privacy.md) — where session data lives and what leaves your machine.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim capture of every request.
- [Using Grok Build Subscription](grok-provider.md) — session affinity for prompt-cache hits on the Grok provider.
- [Using OpenCode Zen Subscription](opencode-provider.md) — the required session header on OpenCode.
