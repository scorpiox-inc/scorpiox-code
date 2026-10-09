# Enterprise Google Cloud Vertex AI in SCORPIOX CODE

You have a Google Cloud project, a billing account, and a Google Cloud API key — and you want SCORPIOX CODE running Gemini models against your own project, on your own quotas, billed per token to you instead of drawn down from a consumer subscription. Set `PROVIDER=gemini_vertex`, put the key in `VERTEX_API_KEY`, and name the model in both `VERTEX_MODEL` and `MODEL`. After that it is ordinary SCORPIOX CODE: the same agent loop, the same tools, the same local traffic capture.

This page walks through the configuration keys and the way the two model slots interact, multi-account rotation with profiles (`/profile` and `/use`), the helper that lists the models your key can reach, and how direct enterprise billing differs from the consumer Antigravity OAuth quota.

Docs for SCORPIOX CODE @ `ad926d7`.

> **The whole idea in one line:** a Google Cloud API key in `VERTEX_API_KEY` plus `PROVIDER=gemini_vertex` turns SCORPIOX CODE into a Vertex AI client against the publisher endpoint — pay-per-use against your project, no OAuth login, no token refresh, no subscription quota to share with anyone.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Google Cloud project and a billing account** | Direct pay-per-use; every request bills to your project |
| **You need dedicated, predictable quotas** | Your project's rate limits are yours alone — no shared consumer pool |
| **You run several GCP accounts or projects** | One key per team or environment, rotated with profiles |
| **You already hold a Google Cloud API key** | The key is the whole credential — no login flow to run |

If you do **not** have a Google Cloud project and key, and you instead use a Google / Antigravity subscription, use `PROVIDER=google_gemini` or `PROVIDER=google_claude` — see [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md). The two paths run overlapping Gemini models on completely different billing and quota models; [Enterprise GCP billing vs. consumer Antigravity quota](#enterprise-gcp-billing-vs-consumer-antigravity-quota) below compares them.

> **API key, not OAuth.** This provider authenticates with a Google Cloud API key and bills per token against your GCP billing account. It does not use the Antigravity sign-in and does not touch consumer subscription quota.

---

## The headline: three keys

Everything is a `KEY=VALUE` pair in the [configuration cascade](scorpiox-env.md). The minimal working set is:

```
PROVIDER=gemini_vertex
VERTEX_API_KEY=your-google-cloud-api-key
MODEL=gemini-3.7-flash
VERTEX_MODEL=gemini-3.7-flash
```

`PROVIDER` selects the Vertex AI path, `VERTEX_API_KEY` carries the credential, and the two model keys name the model. This page assumes you already have the project, billing, and the key — that setup happens on the Google side, and the key is the only thing SCORPIOX CODE needs back from it.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(install default)* | Set to `gemini_vertex` to activate this provider. |
| `VERTEX_API_KEY` | text | *(empty)* | **Required.** Your Google Cloud API key. Every request fails without it. |
| `VERTEX_MODEL` | text | `gemini-3.1-pro-preview` | The provider's own model slot. Accepts a full `gemini-*` ID (for example `gemini-3.7-flash`, `gemini-3.8-flash`) or a short alias. |
| `MODEL` | text | `sonnet` | The generic model slot shared by every provider. **This is the value that ultimately names the model on the wire.** Set it to the same value as `VERTEX_MODEL`. |

### Why set both `MODEL` and `VERTEX_MODEL`

The two keys are not duplicates; they are read at different points and they answer different questions.

- `MODEL` is the generic slot every provider shares. It is loaded at startup and re-applied to the active provider whenever settings change, so it is always the last word on which model a request names.
- `VERTEX_MODEL` is the provider's own slot. It is what a bare Vertex AI session would use, and it is the value the provider resolves into a concrete Gemini model when nothing else overrides it.

Because `MODEL` is applied last, **a mismatch silently wins in its favor**. `MODEL=opus` with `VERTEX_MODEL=gemini-3.8-flash` sends the `opus` mapping, not the flash model you thought you pinned. The reliable habit is to set both to the same full ID:

```
MODEL=gemini-3.8-flash
VERTEX_MODEL=gemini-3.8-flash
```

That guarantees the request that goes out, the model shown in the UI, and the model you wrote down all agree. It is the single most common source of "wrong model" confusion on this provider.

### Short aliases and pass-through

When a model value is a short alias, it resolves to a concrete model:

| Alias | Resolves to |
|-------|-------------|
| `opus`, `pro` | `gemini-3.1-pro-preview` |
| `sonnet`, `flash` | `gemini-3-flash-preview` |
| `haiku` | `gemini-3.1-flash-lite-preview` |

Any value that already starts with `gemini-` is passed through unchanged — so `gemini-3.7-flash` and `gemini-3.8-flash` reach the publisher endpoint exactly as written. Anything that is neither a known alias nor a `gemini-*` ID falls back to the default `gemini-3.1-pro-preview`, with no error.

---

## Setting the keys

### In `scorpiox-env.txt` (any cascade tier)

```
PROVIDER=gemini_vertex
VERTEX_API_KEY=your-google-cloud-api-key
MODEL=gemini-3.7-flash
VERTEX_MODEL=gemini-3.7-flash
```

### In a named profile (recommended)

Profiles keep a key out of any file you might commit, and give you one switch per account. Create `~/.claude/scorpiox-env/gemini-vertex-flash-a.txt`:

```
PROVIDER=gemini_vertex
VERTEX_API_KEY=your-google-cloud-api-key
MODEL=gemini-3.7-flash
VERTEX_MODEL=gemini-3.7-flash
```

Activate it persistently or for a single session:

```bash
# Persistent (survives restarts — written to the user tier)
/profile gemini-vertex-flash-a

# Session-only (no file change)
/use gemini-vertex-flash-a
```

See [Multi-account rotation with profiles](#multi-account-rotation-with-profiles) for the full pattern.

### As OS environment variables

```bash
export PROVIDER=gemini_vertex
export VERTEX_API_KEY=your-google-cloud-api-key
export MODEL=gemini-3.7-flash
export VERTEX_MODEL=gemini-3.7-flash
```

Environment variables sit above every file tier, including the active profile. That makes them the right tool for one-off runs and CI, and the classic trap everywhere else — an exported `VERTEX_API_KEY` from last week's experiment silently outranks the profile you are looking at. When a value "will not stick", check the environment before blaming the file.

### Confirm what is actually configured

```bash
scorpiox-config            # interactive config editor; the Vertex keys appear when this provider is selected
scorpiox-config --verbose  # resolved values (PROVIDER, MODEL, ACTIVE_PROFILE, ...) with the tier each came from
```

`--verbose` shows the resolved provider and model with their source tier, so you know which file won before a long run. The Vertex-specific keys only appear in the editor once `PROVIDER=gemini_vertex` is set, which is itself a useful check that the provider value took.

---

## Multi-account rotation with profiles

Vertex AI is the natural fit for running several Google Cloud accounts side by side — one per team, one per environment, one per billing project. Give each account its own named profile and hop between them live; the `gemini-vertex-flash-*` naming convention keeps them easy to scan in a picker:

```
# ~/.claude/scorpiox-env/gemini-vertex-flash-a.txt
PROVIDER=gemini_vertex
VERTEX_API_KEY=your-google-cloud-api-key
MODEL=gemini-3.7-flash
VERTEX_MODEL=gemini-3.7-flash
```

```
# ~/.claude/scorpiox-env/gemini-vertex-flash-b.txt
PROVIDER=gemini_vertex
VERTEX_API_KEY=your-other-google-cloud-api-key
MODEL=gemini-3.8-flash
VERTEX_MODEL=gemini-3.8-flash
```

Switch at runtime:

```text
/profile gemini-vertex-flash-a    # persist it — survives restarts
/profile gemini-vertex-flash-b    # switch account and model, hot-reloads
/use    gemini-vertex-flash-a     # this session only, nothing written
/profile off                      # drop back to the base cascade
/model  gemini-3.8-flash          # swap models in-session without touching files
```

Both `/profile` and `/use` reload the provider in place — new key, new model, no restart, no lost session. If a profile cannot initialize (a missing or rejected key, for instance), the switch reverts to what was running and tells you, so you are never left half-switched. The difference between the two commands is scope only: **`/profile` writes `ACTIVE_PROFILE` and survives restarts; `/use` is an in-session switch that disappears when the session ends.** See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

A few habits that make rotation painless:

- **One profile per billing account.** The profile is the unit of "who pays for this run". Keep a `gemini-vertex-flash-*` profile per account, and the answer to "what did this session bill against" is always one `/profile` away.
- **Keep keys in the user tier.** `~/.claude/scorpiox-env/` is your machine, not your repository. A project-tier profile with a live key in it is a credential you have committed.
- **Run `scorpiox-vertex-models` after each switch.** Confirm the account you just rotated to can actually see the model you pinned before you lean on it.

---

## Inspecting available models

The `scorpiox-vertex-models` CLI lists the Gemini models your key can reach. It reads `VERTEX_API_KEY` from the environment or from your SCORPIOX CODE configuration, prints raw JSON to stdout, and reports errors on stderr:

```bash
# List every model the key can see
scorpiox-vertex-models

# Details for a single model
scorpiox-vertex-models gemini-3.7-flash

# Pick out just the names
scorpiox-vertex-models | jq -r '.models[].name'
```

With no key configured it fails fast and tells you:

```
error: VERTEX_API_KEY not set
Set via environment or in scorpiox-env.txt
```

Use it to confirm a model ID exists before you pin it in `VERTEX_MODEL`, and to check that a rotated profile's key still sees the models you expect. The listing comes from Google's model-listing endpoint, so treat its output as the candidate list: a model that appears there is worth trying, and the last word on what works is a real turn against the publisher endpoint.

---

## What a request looks like

For each turn SCORPIOX CODE builds one request and sends it to the Vertex AI publisher endpoint for the model you named, translating between SCORPIOX CODE's wire format and Gemini's request and response shapes for you. Nothing to configure; the consequences are what you can see:

- **Model** — the resolved ID from [Short aliases and pass-through](#short-aliases-and-pass-through). A full `gemini-*` ID is sent exactly as written.
- **System prompt, conversation, and tools** — translated in one step, so the agent loop, tool calls, and tool results behave exactly as they do on any other provider.
- **Thinking is on by default.** Every request asks the model to include its reasoning, and reasoning summaries come back as their own block, separate from the answer text.
- **Images work.** Images you attach, or that the agent reads, are sent as native inline image data — a screenshot you paste is visible to the model.
- **Usage is reported per turn.** Prompt, candidate, and cached token counts map onto the usual input / output / cache-read / cache-write figures, so the status bar shows `in:` / `R:` / `W:` / `out:` exactly as it does elsewhere. See [Token Usage Observability](usage-observability.md).
- **Streaming is not used on this path.** Each turn is one request and one complete response; `STREAMING=1` has no effect here. If you want throughput figures in the status bar, `USAGE_SPEED_ESTIMATE=1` gives you an end-to-end estimate rather than a first-token split.
- **Transient failures are retried for you.** Rate limits and server-side hiccups (`429`, `500`, `503`, `529`) and network errors are retried up to five times with an exponential backoff that starts at one second, doubles, and caps at thirty seconds, plus jitter so parallel agents do not retry in lockstep. Each attempt has a two-minute timeout. Anything else — a bad key, an unknown model — fails fast with the status code and the response body in the message.
- **Every call is captured locally.** The request and the response land in the session's traffic capture, verbatim, like every other provider. See [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md).

---

## Enterprise GCP billing vs. consumer Antigravity quota

Both paths reach Gemini models, and mixing them up is easy because the model names overlap. The difference is who pays and which pool the request draws from:

| | **Vertex AI** (this page) | **Antigravity providers** (consumer) |
|---|---|---|
| `PROVIDER` value | `gemini_vertex` | `google_gemini` / `google_claude` |
| Credential | Google Cloud API key (`VERTEX_API_KEY`) | OAuth sign-in, stored refresh token |
| Billing | Per token, on your GCP billing account | Subscription allowance |
| Quotas | Dedicated to your GCP project | Shared consumer pool |
| Token refresh | None — the key is static | Automatic from the stored refresh token |
| Profiles | `gemini-vertex-flash-*` rotated with `/profile` | `supergoogle-*` accounts written by the login |

The practical difference is quota isolation. The consumer Antigravity profiles the OAuth login writes (the `supergoogle-*` family — `antigravity` and `antigravity-claude` at this commit) draw on a shared pool where a busy window is everyone's busy window, and `scorpiox-google-quota` tells you how much of it is left. `gemini_vertex` bills and meters against your own project instead: the throughput ceiling and the rate limits are yours alone, and the cost lands on your invoice rather than on a subscription allowance.

Choose `gemini_vertex` when the bill, the quota, or the data path has to belong to an organization; choose the Antigravity providers when you want Gemini on a Google account with no project and no key.

---

## Gotchas

- **`VERTEX_API_KEY` is required.** With no key, every request fails immediately with an authentication error, and `scorpiox-vertex-models` reports the same. Set the key first; nothing else works without it.
- **`MODEL` wins over `VERTEX_MODEL`.** The generic slot is applied last, so it is the last word. `MODEL=opus` with `VERTEX_MODEL=gemini-3.8-flash` sends the `opus` mapping, not the model you named. Set both to the same full ID.
- **Unknown values do not error — they fall back.** Anything that is neither a `gemini-*` ID nor a known alias becomes `gemini-3.1-pro-preview`. A typo like `gemini 3.7 flash` (spaces) runs the pro default without a word of complaint. Copy IDs out of `scorpiox-vertex-models` instead of typing them.
- **A generic alias is a moving target.** `MODEL=sonnet` is convenient and vague: it maps to a specific preview model at this commit and possibly a different one later. If reproducibility matters — billing analysis, benchmarking, a fleet of agents that must behave identically — pin the full `gemini-*` ID.
- **Environment variables beat profiles too.** An exported `VERTEX_API_KEY`, `VERTEX_MODEL`, or `MODEL` outranks the active profile and every file tier. Indispensable in CI; silently confusing everywhere else.
- **Keys are secrets.** Keep them out of project tiers and committed files. Per-account profiles under `~/.claude/scorpiox-env/` stay on your machine, which is exactly where a credential belongs.
- **The in-browser build does not run this provider.** The web build has no native path for Vertex AI; use a native install for `PROVIDER=gemini_vertex`.
- **Billing is per token, on your account.** A runaway agent loop is your invoice. The status bar counters and the session traffic capture are the two honest ways to see what a run actually consumed.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile cannot initialize — no restart, no lost session.
- **Model listings are a candidate list.** A model that `scorpiox-vertex-models` shows is available to the key for listing; confirm the exact ID with a real turn before you pin it to a fleet.

---

## Reference

- **Provider:** `PROVIDER=gemini_vertex`
- **Endpoint:** the Vertex AI publisher endpoint (`aiplatform.googleapis.com`), called with your key
- **Auth:** `VERTEX_API_KEY` — required, static, no refresh
- **Provider model slot:** `VERTEX_MODEL` (default `gemini-3.1-pro-preview`)
- **Wire model slot:** `MODEL` — applied last, wins on the wire; set both to the same value
- **Aliases:** `opus` / `pro` → `gemini-3.1-pro-preview`, `sonnet` / `flash` → `gemini-3-flash-preview`, `haiku` → `gemini-3.1-flash-lite-preview`, any `gemini-*` ID passes through
- **Retries:** up to 5 attempts on `429` / `500` / `503` / `529` and network errors, 1 s to 30 s exponential backoff with jitter, 120 s per attempt
- **Streaming:** not used on this path; `USAGE_SPEED_ESTIMATE=1` gives an end-to-end figure
- **Helper commands:** `/profile`, `/use`, `/model <name>`, `scorpiox-config --verbose`
- **Standalone tool:** `scorpiox-vertex-models [model-id]`

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md)
- [Token Usage Observability](usage-observability.md)
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md)
