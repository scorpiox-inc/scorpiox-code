# Direct Anthropic API & Custom Endpoints in SCORPIOX CODE

Have an Anthropic API key and want to pay per token instead of running through a subscription? Or you operate an Anthropic-compatible gateway and need SCORPIOX CODE to talk to something other than Anthropic's own servers? The **Anthropic provider** (`PROVIDER=anthropic`) does both. It speaks the Anthropic Messages API directly with a plain `x-api-key` header, and it lets you retarget the endpoint to almost anything that speaks the same protocol: the official API, a self-hosted gateway, a corporate relay, or a third-party proxy with its own key and URL.

This page is the full how-to: the four authentication modes (`official`, `custom`, `zai`, `antigravity`), the keys that make each one work, how model aliases and version overrides resolve, what actually goes on the wire, and where this provider sits relative to the Claude Code subscription provider and the Google Cloud Claude provider.

Docs for SCORPIOX CODE @ `0cd528b`.

> **The whole idea in one line:** set `PROVIDER=anthropic`, hand over an API key, and SCORPIOX CODE sends ordinary Anthropic Messages API requests — to `api.anthropic.com` by default, or to whatever URL you name.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have an Anthropic API key** | A key from the Anthropic Console, billed per token |
| **You run an Anthropic-compatible endpoint** | A gateway, relay, or self-hosted server that exposes `/v1/messages` |
| **You route through a third-party proxy** | Z.AI or an Antigravity-style relay, each with its own key and URL |
| **You want the exact model ID on the wire** | Pass a full `claude-*` ID and it is sent as-is |

If you are instead signing in with a **subscription** rather than a key, that is a different provider: the [Claude Code provider](claude-code-provider.md) (`PROVIDER=claude_code`, OAuth session login) or the Google Cloud Claude provider (`PROVIDER=google_claude`). Those draw on an account allowance. The Anthropic provider is for **endpoints you address directly with a key**.

> **Key, not subscription.** The Anthropic provider authenticates with an `ANTHROPIC_API_KEY` sent as an `x-api-key` header and bills per token against that key. It does not run an OAuth login, and it does not draw on a Claude or Claude Code seat.

---

## The headline: the keys

Everything is a `KEY=VALUE` pair in the [configuration cascade](scorpiox-env.md). To run against the official Anthropic API you need exactly two lines:

```
PROVIDER=anthropic
ANTHROPIC_API_KEY=sk-ant-...
```

That is the whole thing for `api.anthropic.com`. Add `ANTHROPIC_API_URL` when you point at a custom endpoint, and switch `ANTHROPIC_AUTH_PROVIDER` when you use a proxy mode.

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `anthropic` to activate this provider. |
| `ANTHROPIC_AUTH_PROVIDER` | choice | `official` | `official`, `custom`, `zai`, or `antigravity`. Selects which key and URL are used (see [Auth provider modes](#auth-provider-modes)). An unrecognized value falls back to `official`. |
| `ANTHROPIC_API_KEY` | text | *(empty)* | The key for `official` and `custom` modes, and the **fallback key** in `zai` and `antigravity` modes when their dedicated key is empty. Also read from the OS environment variable of the same name, which beats every file in the cascade. **Required** unless a proxy key is set. |
| `ANTHROPIC_API_URL` | text | `https://api.anthropic.com/v1/messages` | The **full** Messages endpoint, path included. Used in `custom` mode. |
| `ANTHROPIC_ZAI_KEY` | text | *(empty)* | Key for `zai` mode. Falls back to `ANTHROPIC_API_KEY` when empty. |
| `ANTHROPIC_ZAI_URL` | text | `https://api.z.ai/api/anthropic/v1/messages` | Endpoint for `zai` mode. |
| `ANTHROPIC_ANTIGRAVITY_KEY` | text | *(empty)* | Key for `antigravity` mode. Falls back to `ANTHROPIC_API_KEY` when empty. |
| `ANTHROPIC_ANTIGRAVITY_URL` | text | *(none)* | Endpoint for `antigravity` mode. **No default** — you must set it or the mode refuses to start. |
| `MODEL` | text | `sonnet` | The model to run: a short alias (`opus`, `sonnet`, `haiku`) or a full model ID. See [Choosing a model](#choosing-a-model). |
| `CLAUDE_MODEL_OPUS_ID_OVERRIDE` / `CLAUDE_MODEL_SONNET_ID_OVERRIDE` / `CLAUDE_MODEL_HAIKU_ID_OVERRIDE` | text | *(empty)* | Pin a specific model version to an alias. See [Model overrides](#model-overrides). |
| `STREAMING` | bool | `0` | `0` = one complete response per turn, `1` = stream server-sent events. |
| `THINKING` / `THINKING_BUDGET` | bool / number | `1` / `10000` | Extended-thinking toggle and its token budget. |
| `USAGE_SPEED_ESTIMATE` | bool | `0` | Opt-in client-side speed estimate in the status bar. See [Streaming and speed estimates](#streaming-and-speed-estimates). |

> **No login step.** Unlike the Claude Code and Grok providers there is no `scorpiox-*-login` helper here. There is nothing to sign in to — you supply a key, or you point at a proxy that handles the billing. Configuration is a handful of keys, not a login.

In the config editor (`/config` in a session, or `scorpiox-config` from a shell) these keys live in the **Provider & Authentication** section and appear once `PROVIDER` is set to `anthropic`. `ANTHROPIC_API_URL` only shows up when `ANTHROPIC_AUTH_PROVIDER` is `custom`.

---

## Auth provider modes

`ANTHROPIC_AUTH_PROVIDER` decides which key is used and which URL is hit. The default is `official`, and an empty or unrecognized value falls back to `official` too.

| Mode | Key used | Endpoint used | Model behavior |
|------|----------|---------------|----------------|
| `official` (default) | `ANTHROPIC_API_KEY` | The built-in Anthropic Messages URL (`https://api.anthropic.com/v1/messages`) | Aliases resolve to the current Claude IDs |
| `custom` | `ANTHROPIC_API_KEY` | `ANTHROPIC_API_URL` — your endpoint, your rules | Aliases resolve to the current Claude IDs |
| `zai` | `ANTHROPIC_ZAI_KEY`, falling back to `ANTHROPIC_API_KEY` | `ANTHROPIC_ZAI_URL`, defaulting to `https://api.z.ai/api/anthropic/v1/messages` | **Every request is rewritten to `glm-5`** |
| `antigravity` | `ANTHROPIC_ANTIGRAVITY_KEY`, falling back to `ANTHROPIC_API_KEY` | `ANTHROPIC_ANTIGRAVITY_URL` (**must be set**) | Aliases map to `*thinking` variants |

### `official` — the Anthropic API

The default. Set nothing extra beyond the key:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=official
ANTHROPIC_API_KEY=sk-ant-...
MODEL=sonnet
```

The endpoint is Anthropic's Messages API. Each request carries `content-type: application/json`, `accept: application/json`, an `anthropic-version` header, and your key as `x-api-key`. There are no CLI-emulating headers and no subscription beta flags — this is the plain API surface, which is exactly what makes an ordinary API key the right credential for it.

### `custom` — your own endpoint

Point at any Anthropic-compatible URL:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=custom
ANTHROPIC_API_KEY=your-gateway-key
ANTHROPIC_API_URL=https://llm-gateway.internal.example.com/v1/messages
MODEL=sonnet
```

Three things to get right:

- **The URL is the full endpoint, path included.** SCORPIOX CODE sends it as-is and appends nothing. `https://llm-gateway.internal.example.com/v1/messages` is correct; `https://llm-gateway.internal.example.com` is not.
- **The key is whatever your endpoint expects.** It still goes out as `x-api-key`, so a gateway that checks a different header needs to be the thing that translates.
- **Any server that accepts Anthropic Messages JSON works.** The request body is standard: `model`, `max_tokens`, `stream`, an optional `thinking` block, a `system` array, `messages`, and `tools` when tools are on.

### `zai` — the Z.AI proxy

Z.AI exposes an Anthropic-compatible surface with its own key and endpoint:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=zai
ANTHROPIC_ZAI_KEY=your-zai-key
MODEL=sonnet
```

`ANTHROPIC_ZAI_URL` already has a working default (`https://api.z.ai/api/anthropic/v1/messages`), so you normally leave it alone and set only the key. In this mode every request is rewritten to the model `glm-5` regardless of the alias you set — the alias still records your *intent*, but the ID on the wire is fixed.

### `antigravity` — the Antigravity proxy

Antigravity uses its own key and endpoint and a different family of model IDs. There is **no default URL** for this mode, so you must set it:

```
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=antigravity
ANTHROPIC_ANTIGRAVITY_KEY=your-key
ANTHROPIC_ANTIGRAVITY_URL=https://your-antigravity-endpoint/v1/messages
MODEL=sonnet
```

If the Antigravity URL is empty the provider stops and reports that it is not configured rather than guessing a default. The aliases map onto the proxy's thinking-style model IDs — see [Model behavior by mode](#model-behavior-by-mode).

---

## Choosing a model

`MODEL` accepts either a short alias or a full model ID. In `official` and `custom` modes the aliases resolve to these IDs at this commit:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-4-6` (the provider default `sonnet`) |
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251001` |
| any other value | passed through as-is |

Anything that is not one of the three aliases — for example `claude-opus-4-6` or a newer ID your account has access to — is sent to the endpoint unchanged. That is how you run a model version the built-in defaults do not know about without editing anything.

### Model overrides

Pin a specific version to each alias with the override keys:

- `CLAUDE_MODEL_OPUS_ID_OVERRIDE`
- `CLAUDE_MODEL_SONNET_ID_OVERRIDE`
- `CLAUDE_MODEL_HAIKU_ID_OVERRIDE`

When set, an override replaces the built-in ID for that alias — so `MODEL=opus` resolves to your pinned version instead of the default one. Full model IDs written directly in `MODEL` are never overridden; they pass through as-is. Leave an override empty to keep the built-in default.

### Model behavior by mode

The alias-to-ID mapping depends on the auth provider:

| Mode | `opus` | `sonnet` | `haiku` | Full ID in `MODEL` |
|------|--------|----------|---------|--------------------|
| `official` / `custom` | `claude-opus-4-6` | `claude-sonnet-4-6` | `claude-haiku-4-5-20251001` | passed through |
| `antigravity` | `claude-opus-4-5-thinking` | `claude-sonnet-4-5-thinking` | `claude-sonnet-4-5-thinking` | passed through |
| `zai` | `glm-5` | `glm-5` | `glm-5` | `glm-5` (every request) |

Switch models at runtime without editing any file with the in-session `/model` command, or set a standing default in your profile. `/model` takes the same values as `MODEL` — aliases and full IDs both work, and the switch lasts for the current session; put `MODEL` in a profile or config file to make it stick. You can also pin a model for a single run with `sx -m opus`, or override any key for that run only with `sx -e ANTHROPIC_AUTH_PROVIDER=zai`.

---

## What a request looks like

For each turn SCORPIOX CODE builds one Messages API request:

1. **Model** — the resolved ID from [Choosing a model](#choosing-a-model).
2. **System prompt** — sent as two text blocks, each marked as ephemeral cache control so the provider can reuse the prefix between turns.
3. **Messages** — the conversation, with consecutive same-role entries grouped, and a cache breakpoint on the last block of each role.
4. **Thinking** — an extended-thinking block with `THINKING_BUDGET` tokens when `THINKING=1`.
5. **Tools** — the tool definitions when `TOOLS=1`, with the same cache treatment on the first turn.

Each request is sent with a ten-minute timeout, so long agentic turns are not cut off mid-flight. The response comes back as content blocks with a stop reason and a usage object. Token counts are shown in the session status: input, output, and the cache read and cache write figures — exactly what you need to check what a turn actually cost. See [Token Usage Observability](usage-observability.md) for where those numbers appear and what feeds them.

Note that the subscription-rate indicators (`5h:` / `7d:` in the status bar) come from the subscription providers' response headers. The Anthropic provider sends plain API requests, so those indicators stay empty here; your cost signal is the token count itself.

---

## Streaming and speed estimates

`STREAMING=0` (the default) sends one request and parses one complete response. `STREAMING=1` asks the endpoint for server-sent events instead; the response is parsed the same way either way, so the only practical difference is how the endpoint delivers the body.

Pair streaming with the opt-in speed estimate to get live figures in the status bar:

```
STREAMING=1
USAGE_SPEED_ESTIMATE=1
```

With both on, SCORPIOX CODE times each request locally and shows prompt and generation rates (`~pp` / `~tg`) even when the endpoint reports no timings of its own; when no first-token timestamp can be picked out of the stream you get a single end-to-end figure (`~e2e`) instead. Estimates include network latency, and any timings the server does report always win.

---

## Retries and transient errors

The provider retries transient failures on its own, up to ten attempts with an exponential backoff that doubles from one second and caps at a minute, plus jitter so many agents do not retry in lockstep. These statuses are treated as transient:

| Status | Meaning |
|--------|---------|
| `429` | Rate limited |
| `500` | Internal server error |
| `502` | Bad gateway |
| `503` | Service unavailable |
| `529` | Site overloaded |
| `403` | Forbidden, when it is transient auth or proxy trouble |

Anything else is reported immediately as an error with the status and the response body, so a bad key or an unknown model fails fast instead of burning ten retries on it. Above the provider retries, the agent loop has its own wider retry policy for long runs — see [Configuration and Profiles](scorpiox-env.md) for the `AGENT_RETRY_*` keys if you need to tune that.

---

## Where every request is recorded

Every request and response is written verbatim to disk: the full endpoint URL, all headers (your key masked), the complete request body, and the response payload with its status code. When session logging is active the capture lands in that session's `traffic/` folder; otherwise it goes to a timestamped folder under `.scorpiox/traffics/providers/anthropic/`. A request that fails internal validation before it is sent is saved too, marked invalid, so nothing disappears silently.

This is the same zero-black-box stance as the rest of SCORPIOX CODE — see [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) for the file layout and how to replay a captured call.

---

## Anthropic provider vs. the other Claude paths

It is easy to confuse the three because they all run Claude models. Here is the difference:

| | **Anthropic provider** (this page) | **Claude Code provider** | **Google Cloud Claude provider** |
|---|---|---|---|
| `PROVIDER` value | `anthropic` | `claude_code` | `google_claude` |
| **Authentication** | `ANTHROPIC_API_KEY` sent as `x-api-key` | OAuth session login (`scorpiox-claudecode-login`) | Google account OAuth via Google's Claude backend |
| **Billing** | Pay-per-token API usage on your key | Your Claude / Claude Code subscription allowance | Your Google AI subscription allowance |
| **Endpoint** | `ANTHROPIC_API_URL` (default `api.anthropic.com/v1/messages`) | `api.anthropic.com/v1/messages?beta=true` (or the SCORPIOX proxy) | Google's Claude endpoint |
| **Model key** | `MODEL` | `MODEL` | `GOOGLE_CLAUDE_MODEL` |
| **Redirectable to your own URL** | Yes — `custom` mode, plus `zai` and `antigravity` proxy modes | No — the Claude Code surface only | No |
| **Wire format** | Plain Messages API, no CLI-emulating headers, no beta flags | Full Claude Code CLI emulation (user-agent, `x-stainless-*`, beta flags) | Translated to Google's request shape |

Rule of thumb: **API key or custom endpoint, use `anthropic`. A Claude / Claude Code seat, use `claude_code`. A Google AI plan for Claude models, use `google_claude`.**

The Anthropic and Claude Code providers both reach Anthropic's Messages API, but with different credentials, different headers, and different billing. Only the Anthropic provider can be pointed at a non-Anthropic URL.

---

## Gotchas

- **`ANTHROPIC_API_URL` is a full endpoint, not an origin.** It includes the path — the reference value is `https://api.anthropic.com/v1/messages`. This is the opposite convention from the OpenAI provider's `OPENAI_BASE_URL`, which is an origin with no path. A missing `/v1/messages` is the most common custom-endpoint failure. See [Using the OpenAI Provider](openai-provider.md).
- **`ANTHROPIC_API_URL` only applies in `custom` mode.** Setting it while `ANTHROPIC_AUTH_PROVIDER` is `official` does nothing — official mode always uses the built-in Anthropic URL. Switch the mode to `custom` for your URL to be honored.
- **`antigravity` has no default URL.** You must set `ANTHROPIC_ANTIGRAVITY_URL`. If it is empty the provider reports the URL as unconfigured instead of guessing.
- **`antigravity` aliases map to `*thinking` variants.** `sonnet` and `haiku` both become `claude-sonnet-4-5-thinking`; only `opus` becomes `claude-opus-4-5-thinking`.
- **`MODEL` defaults to `sonnet`.** Leave it unset and requests ask for `claude-sonnet-4-6`. Set it to a model the endpoint actually serves, or pass a full model ID.
- **The config editor offers three models.** `MODEL` is a three-way choice (`opus` / `sonnet` / `haiku`) in the editor UI. Full model IDs still work — set them in a config file, a profile, or pass them to `/model` or `sx -m` instead.
- **The OS environment variable wins.** An `ANTHROPIC_API_KEY` exported in your shell beats any value in a file — useful for one-off runs, easy to forget an hour later. The same holds for every other key in this table.
- **No `5h:` / `7d:` indicators here.** Those come from subscription providers' rate-limit headers; this provider sends plain API requests, so the status bar shows token counts only.
- **Traffic is logged locally.** Requests and responses are written under the session's `traffic/` directory (or a timestamped folder under `.scorpiox/traffics/providers/anthropic/`) for inspection. See [Traffic Logging](traffic-logging.md).

---

## Quick reference

| Goal | What to set |
|------|-------------|
| Official Anthropic API | `PROVIDER=anthropic` + `ANTHROPIC_API_KEY` |
| Custom endpoint | `+ ANTHROPIC_AUTH_PROVIDER=custom` + `ANTHROPIC_API_URL` |
| Z.AI proxy | `+ ANTHROPIC_AUTH_PROVIDER=zai` + `ANTHROPIC_ZAI_KEY` |
| Antigravity proxy | `+ ANTHROPIC_AUTH_PROVIDER=antigravity` + `ANTHROPIC_ANTIGRAVITY_KEY` + `ANTHROPIC_ANTIGRAVITY_URL` |
| Pick a model | `MODEL=sonnet` (or `opus` / `haiku` / a full ID) |
| Pin a version | `CLAUDE_MODEL_*_ID_OVERRIDE` |
| Switch mid-session | `/model <name>` |
| Status-bar speed figures | `STREAMING=1` + `USAGE_SPEED_ESTIMATE=1` |
| One-off run override | `sx -e ANTHROPIC_API_KEY=... -p "..."` |

Pick your mode, set the key (and the URL where there is no default), choose a model, and go.

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md) — the subscription path to the same models.
- [Token Usage Observability](usage-observability.md) — where the token counts and speed figures show up.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim on-disk record of every request.
- [Using the OpenAI Provider](openai-provider.md) — the other key-addressable provider, and its opposite URL convention.
