# Direct Anthropic API & Custom Endpoints in SCORPIOX CODE

Already have an Anthropic API key, or a proxy that speaks the Anthropic Messages API? Point SCORPIOX CODE straight at it. Set `PROVIDER=anthropic`, drop in your key (or your proxy's endpoint), pick a model, and go — no OAuth login, no subscription, no router.

This page covers every configuration key for the **Anthropic provider**, explains the four authentication modes, how model aliases and version overrides work, and how this provider differs from the Claude Code subscription provider and the Google Cloud Claude provider.

Source of truth: `sx_provider_anthropic.c`, `sx_provider_claude_code.c`, `sx_models_generated.h`, `scorpiox-config.c`, and `scorpiox-env.txt` at commit `5fd054b`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have an Anthropic API key** | Pay-as-you-go tokens from [console.anthropic.com](https://console.anthropic.com/settings/keys) |
| **You run a proxy** | A gateway, gateway-for-teams, or LLM router that exposes the Anthropic `/v1/messages` format |
| **You use a third-party Claude backend** | Z.AI or another Anthropic-compatible endpoint behind a custom URL |
| **You want full control of the endpoint** | Pin the exact `ANTHROPIC_API_URL` your server expects, including on-prem or air-gapped hosts |

If you pay for a Claude / Claude Code **subscription** and want to draw on that allowance instead of API tokens, use `PROVIDER=claude_code` instead. If you run Claude models through a **Google** Cloud Code Assist sign-in, use `PROVIDER=google_claude`. The Anthropic provider is the one for **API keys and custom endpoints**.

> **API key, not subscription.** The Anthropic provider authenticates with a bearer API key (`ANTHROPIC_API_KEY`) and bills pay-per-token against the key or the endpoint it points at. It does **not** do the OAuth session login that the Claude Code provider uses, and it does **not** draw on a Claude subscription's 5-hour / weekly windows.

---

## The one key you must set

The only thing that is strictly required is `PROVIDER=anthropic`. Everything else has a sensible default, but `ANTHROPIC_API_KEY` is the one you'll almost always fill in:

```
PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
```

That's a working, default-endpoint, `sonnet` setup. The rest of this page is the knobs you reach for when the default is not what you want.

> **Never commit or paste a real key into documentation, profiles you share, or the repository.** Keys are secrets. Use a placeholder (`sk-ant-...`) anywhere you're writing things down.

---

## Configuration keys

All keys can be set in `scorpiox-env.txt` at any cascade tier, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and profile-switching semantics.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `anthropic` to activate this provider. |
| `ANTHROPIC_AUTH_PROVIDER` | choice | `official` | Auth mode: `official`, `antigravity`, `zai`, or `custom`. |
| `ANTHROPIC_API_KEY` | text | *(empty)* | Bearer key for `official` and `custom`; also the fallback key for `antigravity` and `zai`. Read from the OS environment variable of the same name as well. |
| `ANTHROPIC_API_URL` | text | `https://api.anthropic.com/v1/messages` | Endpoint used in `custom` mode. Only read when `ANTHROPIC_AUTH_PROVIDER=custom`. |
| `ANTHROPIC_ANTIGRAVITY_KEY` | text | *(empty)* | Key for `antigravity` mode (falls back to `ANTHROPIC_API_KEY` if empty). |
| `ANTHROPIC_ANTIGRAVITY_URL` | text | *(empty — required)* | Endpoint for `antigravity` mode. **Must be set** when this mode is active. |
| `ANTHROPIC_ZAI_KEY` | text | *(empty)* | Key for `zai` mode (falls back to `ANTHROPIC_API_KEY` if empty). |
| `ANTHROPIC_ZAI_URL` | text | `https://api.z.ai/api/anthropic/v1/messages` | Endpoint for `zai` mode. |
| `MODEL` | choice | `sonnet` | Model to use. Short alias `opus`, `sonnet`, or `haiku`, or a full model ID. |
| `CLAUDE_MODEL_OPUS_ID_OVERRIDE` | text | *(empty)* | Pins the concrete model ID that the `opus` alias resolves to. |
| `CLAUDE_MODEL_SONNET_ID_OVERRIDE` | text | *(empty)* | Pins the concrete model ID that the `sonnet` alias resolves to. |
| `CLAUDE_MODEL_HAIKU_ID_OVERRIDE` | text | *(empty)* | Pins the concrete model ID that the `haiku` alias resolves to. |

---

## The four authentication modes

`ANTHROPIC_AUTH_PROVIDER` selects which key and which endpoint the provider uses. Each mode maps one key onto one URL; the table shows the effective pair.

| `ANTHROPIC_AUTH_PROVIDER` | Key used | Endpoint used | Model handling |
|---------------------------|----------|---------------|----------------|
| `official` *(default)* | `ANTHROPIC_API_KEY` | `https://api.anthropic.com/v1/messages` | Alias resolves via the override keys, else compiled defaults |
| `custom` | `ANTHROPIC_API_KEY` | `ANTHROPIC_API_URL` (default: `https://api.anthropic.com/v1/messages`) | Alias resolves via the override keys, else compiled defaults |
| `antigravity` | `ANTHROPIC_ANTIGRAVITY_KEY` (fallback `ANTHROPIC_API_KEY`) | `ANTHROPIC_ANTIGRAVITY_URL` **(required)** | Aliases remapped to `*-thinking` IDs; full IDs pass through |
| `zai` | `ANTHROPIC_ZAI_KEY` (fallback `ANTHROPIC_API_KEY`) | `ANTHROPIC_ZAI_URL` (default: `https://api.z.ai/api/anthropic/v1/messages`) | All models routed to the backend's single model |

### `official` — direct Anthropic API

The default. Your key is sent as a bearer to Anthropic's own endpoint. If `ANTHROPIC_API_KEY` is empty, the OS environment variable `ANTHROPIC_API_KEY` is checked as a fallback, so you can keep the key out of the config file entirely:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

Then the config only needs:

```
PROVIDER=anthropic
```

### `custom` — your own Anthropic-compatible endpoint

Use this when a proxy, gateway, or on-prem server speaks the Anthropic Messages format but lives somewhere other than `api.anthropic.com`. Point `ANTHROPIC_API_URL` at the full messages endpoint and keep using your API key:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=custom
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_API_URL=https://your-gateway.example.com/v1/messages
```

A few rules for this mode:

- **`ANTHROPIC_API_URL` is the full endpoint, including the path.** The provider does not append `/v1/messages` — whatever you set is exactly what it calls. Default to `https://api.anthropic.com/v1/messages` only if you genuinely want the public API.
- **`ANTHROPIC_API_URL` is ignored in every other mode.** In `official`, `antigravity`, and `zai` it has no effect; only the mode-specific URL wins. Set `ANTHROPIC_AUTH_PROVIDER=custom` to make it take effect.

### `antigravity` — the Antigravity Claude proxy

Runs Claude models through the Antigravity backend. This mode has its own key and its own endpoint, and it **requires** `ANTHROPIC_ANTIGRAVITY_URL` — if it is empty the provider errors out rather than guessing:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=antigravity
ANTHROPIC_ANTIGRAVITY_KEY=...
ANTHROPIC_ANTIGRAVITY_URL=https://antigravity-endpoint/...
```

In this mode the short aliases are remapped to the backend's thinking-model IDs (e.g. `opus` → `claude-opus-4-5-thinking`). A full model ID you set in `MODEL` passes through unchanged.

### `zai` — the Z.AI proxy

Runs through Z.AI's Anthropic-compatible endpoint. All model selections are routed to the backend's single served model, so the alias you pick only matters for bookkeeping, not for which underlying weights run:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=zai
ANTHROPIC_ZAI_KEY=...
ANTHROPIC_ZAI_URL=https://api.z.ai/api/anthropic/v1/messages
```

If `ANTHROPIC_ZAI_KEY` is empty it falls back to `ANTHROPIC_API_KEY`.

---

## Choosing a model

The model is controlled by `MODEL`. You can use either a **short alias** or a **full model ID**.

### Short aliases

`opus`, `sonnet`, and `haiku` are stable shorthand that track the model tier rather than a specific release. They are also what the `/model` command offers and what tab-completion suggests. The default is `sonnet`.

### Full model IDs

Any full model ID works and is sent to the endpoint as-is. This is how you pin an exact release, or how you request a model the aliases don't cover:

```
MODEL=claude-sonnet-4-6
```

### Pinning an alias with the override keys

Each alias resolves to a concrete model ID. By default that is the compiled-in default for the tier; the override keys let you pin a specific version without giving up the convenience of the alias:

| Override key | Pins the alias | Compiled default |
|--------------|----------------|------------------|
| `CLAUDE_MODEL_OPUS_ID_OVERRIDE` | `opus` | `claude-opus-4-6` |
| `CLAUDE_MODEL_SONNET_ID_OVERRIDE` | `sonnet` | `claude-sonnet-4-6` |
| `CLAUDE_MODEL_HAIKU_ID_OVERRIDE` | `haiku` | `claude-haiku-4-5-20251001` |

Leave an override empty and the compiled default applies. Set it to pin the exact ID you want behind the alias:

```
PROVIDER=anthropic
MODEL=sonnet
CLAUDE_MODEL_SONNET_ID_OVERRIDE=claude-sonnet-4-6
```

A full model ID in `MODEL` (anything that is not exactly `opus`/`sonnet`/`haiku`) is not subject to overrides and is passed through verbatim.

> **The aliases are not the same in every mode.** In `antigravity` and `zai` the aliases are remapped by the backend (see the mode table above). The override keys and the compiled defaults apply to `official` and `custom`, where the provider sends the model straight to Anthropic.

### Switching the model at runtime

You don't have to restart to change models. Use the `/model` command in a session, or set `MODEL` in a profile and switch profiles:

```
/model sonnet
```

`/profile` and `/use` both trigger a live provider reload, so a new `MODEL`, key, or endpoint takes effect immediately — and SCORPIOX CODE reverts automatically if the new profile fails to initialize. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the switching semantics.

---

## Worked examples

### Direct Anthropic API key (simplest)

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=official
ANTHROPIC_API_KEY=sk-ant-...
MODEL=sonnet
```

### API key kept in the environment, not the file

```
# scorpiox-env.txt
PROVIDER=anthropic
MODEL=opus
```
```bash
# shell
export ANTHROPIC_API_KEY=sk-ant-...
```

### Team gateway / custom endpoint

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=custom
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_API_URL=https://gateway.example.com/v1/messages
MODEL=claude-sonnet-4-6
```

### Named profile so you can flip between API key and subscription

Create `~/.claude/scorpiox-env/anthropic-direct.txt`:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=official
ANTHROPIC_API_KEY=sk-ant-...
MODEL=sonnet
```

Then activate it persistently or per-session:

```bash
/profile anthropic-direct      # persists (writes ACTIVE_PROFILE)
/use anthropic-direct          # session-only
```

---

## How it compares to the other Claude providers

It's easy to mix these up because they all run Anthropic models. Here is the difference:

| | **Anthropic provider** (this page) | **Claude Code provider** | **Google Cloud Claude provider** |
|---|---|---|---|
| `PROVIDER` value | `anthropic` | `claude_code` | `google_claude` |
| **Authentication** | `ANTHROPIC_API_KEY` bearer (or per-mode key) | OAuth session login (`scorpiox-claudecode-login`) | Google account OAuth via Antigravity sign-in |
| **Billing** | Pay-per-token against the API key or your endpoint | Your Claude / Claude Code subscription allowance | Your Google account's usage tier |
| **Endpoint** | Any Anthropic endpoint (`ANTHROPIC_API_URL` in `custom` mode) | `api.anthropic.com/v1/messages?beta=true` | Google Cloud Code Assist backend |
| **Model control** | `MODEL` aliases + ID overrides | `MODEL` aliases + ID overrides | `GOOGLE_CLAUDE_MODEL` choice |
| **Best for** | API keys, proxies, custom / on-prem endpoints | People who already pay for Claude / Claude Code | Running Claude through a Google sign-in |

Rule of thumb:

- **You have an API key or a custom endpoint** → `PROVIDER=anthropic` (this page).
- **You have a Claude / Claude Code subscription and no API key** → `PROVIDER=claude_code` — see [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md).
- **You want Claude through a Google Cloud / Antigravity sign-in** → `PROVIDER=google_claude` — see [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md).

---

## Gotchas

- **`ANTHROPIC_API_URL` only does something in `custom` mode.** In `official`, `antigravity`, and `zai` the mode-specific endpoint is used instead. If you set a URL and it seems to be ignored, check that `ANTHROPIC_AUTH_PROVIDER=custom`.

- **`ANTHROPIC_API_URL` is the full endpoint, not a base URL.** Unlike the OpenAI provider (which appends `/v1/chat/completions`), the Anthropic provider calls the URL exactly as written. Include the path.

- **`antigravity` mode fails without a URL.** `ANTHROPIC_ANTIGRAVITY_URL` is required; an empty value is a hard error, not a fallback to the default.

- **The API key has an environment-variable fallback.** `ANTHROPIC_API_KEY` is read from the OS environment as well as from the config file, so a key exported in your shell works even when the config value is empty.

- **Aliases are mode-dependent.** `opus`/`sonnet`/`haiku` resolve to the override key or compiled default in `official` and `custom`, but are remapped in `antigravity` and `zai`. If your model name looks wrong in a proxy mode, set the full model ID in `MODEL` instead.

- **Full model IDs bypass the overrides.** Only the exact aliases `opus`, `sonnet`, and `haiku` go through the override keys. Anything else in `MODEL` is sent to the endpoint verbatim.

- **Profile switches are live.** `/profile` and `/use` tear down and rebuild the provider, so a new key, URL, or model applies immediately — and reverts automatically if the new profile can't initialize.
