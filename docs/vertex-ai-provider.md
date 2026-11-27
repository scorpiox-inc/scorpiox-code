# Enterprise Google Cloud Vertex AI in SCORPIOX CODE

You have a Google Cloud project, a billing account, and a Google Cloud API key, and you want SCORPIOX CODE to run Gemini models through **Vertex AI** — pay-per-use against your own GCP project with its own dedicated quotas. Set `PROVIDER=gemini_vertex`, put your key in `VERTEX_API_KEY`, and name the model you want with `VERTEX_MODEL`. This page walks through the publisher endpoint, the two model keys, multi-account rotation with profiles, and how to inspect the models your key can reach.

Docs for SCORPIOX CODE @ `b978da9`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Google Cloud project and billing account** | Direct GCP pay-per-use, your own quota |
| **You want dedicated, predictable quotas** | No shared consumer-pool contention |
| **You run multiple GCP accounts / projects** | Rotate keys per team or environment via profiles |
| **You already hold a Google Cloud API key** | A key for the AI Platform / Vertex AI endpoint |

If you do **not** have a Google Cloud project or API key and instead use the free **Antigravity** subscription that ships with a Google account, use `PROVIDER=google_gemini` or `PROVIDER=google_claude` instead — see [Using Google Antigravity CLI Subscription](antigravity-provider.md).

> **API key, not OAuth.** This provider authenticates with a Google Cloud API key and bills per token against your GCP billing account. It does **not** use the Antigravity OAuth sign-in and does **not** draw on the consumer subscription quota.

---

## The configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `gemini_vertex` to activate this provider. |
| `VERTEX_API_KEY` | text | *(empty)* | **Required.** Your Google Cloud API key. Requests fail without it. |
| `VERTEX_MODEL` | text | `gemini-3.1-pro-preview` | The concrete Gemini model ID sent to the Vertex AI publisher endpoint. Accepts a full `gemini-*` ID (e.g. `gemini-3.7-flash`, `gemini-3.8-flash`) or a short tier alias. |
| `MODEL` | choice | `opus` | The model SCORPIOX CODE resolves and displays. Set it to the same tier alias as `VERTEX_MODEL` for reliable selection. |

> **Set both `MODEL` and `VERTEX_MODEL`.** `VERTEX_MODEL` is what actually gets sent to the publisher endpoint, while `MODEL` drives the internal resolver and the model shown in the UI. Keeping the two in agreement — for example `MODEL=sonnet` with `VERTEX_MODEL=gemini-3.7-flash` — prevents the display and the request from drifting apart, which is the most common cause of "wrong model" confusion.

### Short tier aliases

When `VERTEX_MODEL` (or `MODEL`) is a short alias, it resolves to a concrete model:

| Alias | Resolves to |
|-------|-------------|
| `opus`, `pro` | `gemini-3.1-pro-preview` |
| `sonnet`, `flash` | `gemini-3-flash-preview` |
| `haiku` | `gemini-3.1-flash-lite-preview` |

Any value that already starts with `gemini-` is passed through unchanged — so `gemini-3.7-flash` and `gemini-3.8-flash` reach the publisher endpoint exactly as written.

---

## Setting the keys

### In scorpiox-env.txt (any cascade tier)

```
PROVIDER=gemini_vertex
VERTEX_API_KEY=your-google-cloud-api-key
MODEL=sonnet
VERTEX_MODEL=gemini-3.7-flash
```

### In a named profile (recommended for multi-account)

Create a file such as `~/.claude/scorpiox-env/gemini-vertex-flash-a.txt`:

```
PROVIDER=gemini_vertex
VERTEX_API_KEY=your-google-cloud-api-key
MODEL=sonnet
VERTEX_MODEL=gemini-3.7-flash
```

Activate it persistently or for the session only:

```bash
# Persistent (survives restarts — writes to the user-tier scorpiox-env.txt)
/profile gemini-vertex-flash-a

# Session-only (no file change)
/use gemini-vertex-flash-a
```

### As OS environment variables

```bash
export PROVIDER=gemini_vertex
export VERTEX_API_KEY=your-google-cloud-api-key
export VERTEX_MODEL=gemini-3.7-flash
```

OS environment variables override all file-based tiers, including the active profile.

---

## Multi-account rotation with profiles

Vertex AI is a natural fit for rotating several Google Cloud accounts — one per team, environment, or billing project. Give each account its own named profile (the `gemini-vertex-flash-*` naming convention keeps them easy to scan), then hop between them live:

```
# ~/.claude/scorpiox-env/gemini-vertex-flash-a.txt
PROVIDER=gemini_vertex
VERTEX_API_KEY=key-for-account-a
MODEL=sonnet
VERTEX_MODEL=gemini-3.7-flash

# ~/.claude/scorpiox-env/gemini-vertex-flash-b.txt
PROVIDER=gemini_vertex
VERTEX_API_KEY=key-for-account-b
MODEL=sonnet
VERTEX_MODEL=gemini-3.8-flash
```

Switch at runtime:

```bash
/profile gemini-vertex-flash-a   # persist
/profile gemini-vertex-flash-b   # switch account, hot-reloads
/use gemini-vertex-flash-a       # or session-only, no file change
```

Each switch reloads the provider in place. If the new profile cannot initialize (for example, a bad or missing key), the switch reverts automatically and your working session is not interrupted.

---

## Inspecting available models

The `scorpiox-vertex-models` CLI lists the Gemini models your key can reach. It reads `VERTEX_API_KEY` from the environment or from your SCORPIOX CODE config and prints raw JSON to stdout.

```bash
# List all available models
scorpiox-vertex-models

# Inspect a single model
scorpiox-vertex-models gemini-3.7-flash

# Pipe through jq to pick out names only
scorpiox-vertex-models | jq -r '.models[].name'
```

Use this to confirm a model ID before you put it in `VERTEX_MODEL`, and to check that the account you rotated to actually has access to it.

---

## Enterprise GCP billing vs. consumer Antigravity quota

Both paths reach Gemini models, so it helps to know which one you are on:

| | **Vertex AI** (this page) | **Antigravity** (consumer) |
|---|---|---|
| **`PROVIDER`** | `gemini_vertex` | `google_gemini` / `google_claude` |
| **Authentication** | Google Cloud API key (`VERTEX_API_KEY`) | OAuth PKCE (`scorpiox-antigravity-login`) |
| **Billing** | Pay-per-use against your GCP billing account | Your Antigravity subscription tier |
| **Endpoint** | Vertex AI publisher (`aiplatform.googleapis.com`) | Cloud Code Assist (subscription backend) |
| **Quotas** | Dedicated quotas on your GCP project | Shared consumer quota (the `supergoogle-*` / Antigravity profiles) |
| **Models** | Gemini via Vertex AI | Gemini *and* Claude-via-Google |
| **Best for** | GCP workloads already on a billing project | People with a Google / Antigravity subscription and no GCP project |

Rule of thumb: **you have a Google Cloud project and API key → `gemini_vertex` (this page). You only have a Google / Antigravity subscription → `google_gemini` / `google_claude`.**

The practical difference is quota isolation. The `supergoogle-*` profiles from an Antigravity login draw on a shared consumer pool, whereas `gemini_vertex` bills and meters against your own project, so enterprise throughput and rate limits are yours alone.

---

## Gotchas

- **`VERTEX_API_KEY` is required.** Without it every request fails with an authentication error. The `scorpiox-vertex-models` CLI reports the same when the key is unset.
- **Set `MODEL` and `VERTEX_MODEL` together.** They are read by different parts of the stack. Leaving one at a default while the other is custom is what produces the "the status bar says one model but the request went to another" situation.
- **Full `gemini-*` IDs pass through unchanged.** Only short aliases (`opus`, `sonnet`, `haiku`, `pro`, `flash`) get resolved. If you type a `gemini-…` ID, that exact string is sent to the publisher endpoint.
- **`VERTEX_API_KEY` is a secret.** Keep it out of shared config tiers and committed files. Per-account profiles under `~/.claude/scorpiox-env/` are the safest place to keep them.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile cannot initialize — no restart, no lost session.
