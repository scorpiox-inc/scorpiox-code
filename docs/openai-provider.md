# Using the OpenAI Provider

SCORPIOX CODE can talk to **any** server that exposes an OpenAI-compatible `/v1/chat/completions` endpoint. Set `PROVIDER=openai`, point `OPENAI_BASE_URL` at your server — local or remote — and you are running. The provider translates the wire format for you, so a single configuration drives llama.cpp, vLLM, SGLang, Ollama, LM Studio, a real OpenAI API key, Azure, or any other compatible endpoint.

This page is the full how-to: the keys that make it work, a **verified launch command** for each of the three modern self-hosted engines (llama.cpp, vLLM, SGLang), how to pick and switch a model, how to tune thinking, reasoning effort, and timeouts, and the one URL gotcha that trips almost everyone up.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** with `PROVIDER=openai` and `OPENAI_BASE_URL` set to your server origin, SCORPIOX CODE translates every request to the OpenAI Chat Completions format and runs the full tool loop against it — llama.cpp, vLLM, SGLang, or the OpenAI cloud, all the same way.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You run your own model server** | llama.cpp, vLLM, or SGLang on your workstation or a GPU node |
| **You want OpenAI API access with a key** | `api.openai.com` or any OpenAI-compatible cloud endpoint |
| **You want a custom / private endpoint** | Azure OpenAI, Ollama, LM Studio, or an internal gateway |
| **Air-gapped / on-prem** | Models served behind a firewall with no internet access |

If you are instead **signing in with a subscription** rather than a key or a local server, that is a different provider: the [Codex provider](codex-provider.md) (OpenAI Codex plan) or the [Copilot provider](copilot-provider.md) (GitHub Copilot) draw on your account allowance. The OpenAI provider is for **endpoints you address directly** — a key, or your own server.

> **No login flow.** Unlike the Codex and Copilot providers, there is no OAuth device-code sign-in here. You either supply a key, or you point at a local server that needs none. Configuration is a handful of keys, not a login.

---

## The headline: a few keys

Everything is a `KEY=VALUE` pair in the [configuration cascade](scorpiox-env.md). For a local server you only ever need:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
MODEL=your-model-name
```

That is the whole thing for a local server. `OPENAI_API_KEY` is **optional** — local engines do not check it, so leave it empty (it is only sent as a `Bearer` header when non-empty). For a real OpenAI-compatible cloud endpoint, add your key.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `openai` to activate this provider. |
| `OPENAI_BASE_URL` | text | *(empty)* | The server origin, e.g. `http://localhost:8080`. **Required in practice.** See [The one gotcha](#the-one-gotcha-openai_base_url-is-an-origin-not-a-path) below. `OPENAI_API_BASE` is accepted as a fallback if this is empty. |
| `OPENAI_API_BASE` | text | *(empty)* | Legacy alias for `OPENAI_BASE_URL`. Used only when `OPENAI_BASE_URL` is empty. |
| `OPENAI_API_KEY` | text | *(empty)* | Bearer token. Optional for local servers; required for a real API. |
| `MODEL` | text | `sonnet` | Which model to run — the name sent to the server. This is the generic model slot and **wins over `OPENAI_MODEL`** when both are set. Set it to a name the server advertises (see [Choosing and switching a model](#choosing-and-switching-a-model)). |
| `OPENAI_MODEL` | text | `default` | Provider-specific model name. Used only when the generic `MODEL` slot is not set. In practice, set `MODEL`. |
| `OPENAI_TIMEOUT` | text | `1800` | HTTP request timeout in seconds (30 minutes). Raise it for very large models or slow prefill. |
| `OPENAI_STREAM` | bool | `0` | `1` streams internally (SSE) and reassembles the body so time-to-first-token can be measured. `0` (default) uses a single non-streamed response. |
| `OPENAI_CHAT_TEMPLATE_KWARGS` | text | *(empty)* | A JSON object merged into every request's `chat_template_kwargs` — the llama.cpp / vLLM / SGLang chat-template channel (e.g. `{"enable_thinking": true}`). |
| `OPENAI_REASONING_EFFORT` | choice | *(empty)* | `low` / `medium` / `high` / `max`. Injected into the request as `reasoning_effort`; only engines that read it act on it. Leave empty to not send it. |
| `OPENAI_EXTRA_HEADERS` | text | `X-Sx-Session-Id:{session_guid},X-Sx-Thread-Id:{thread_guid}` | Identity headers added to every outgoing request. See [Identity headers](#identity-headers-on-every-request). |

> **You must point `OPENAI_BASE_URL` at your server.** There is no built-in server address, so if the URL is empty you get `OPENAI_BASE_URL not configured` and the session will not start.

---

## The one gotcha: `OPENAI_BASE_URL` is an origin, not a path

SCORPIOX CODE appends `/v1/chat/completions` to whatever you put in `OPENAI_BASE_URL`. So set **only the origin**:

```
OPENAI_BASE_URL=http://localhost:8080          # correct
OPENAI_BASE_URL=http://localhost:8080/v1       # wrong — becomes .../v1/v1/chat/completions
```

- **Do not include** `/v1`, `/v1/chat`, or `/v1/chat/completions` in the URL.
- A `404` after a long prompt usually means the base URL is wrong (extra path, or the wrong port). Confirm the endpoint answers with `scorpiox-openai-models <url>` before touching anything else.

The helper utilities are lenient about it: `/models`, `/slots`, and `/metrics` strip a trailing `/v1` or `/models` for you when they build their own request, so a slightly messy `OPENAI_BASE_URL` still works for *inspecting* the server even if it would 404 for chat.

---

## Guided setup

If you would rather not hand-edit a profile, there is a wizard that does it for you:

```bash
scorpiox-openai-login
```

It asks for the base URL, an optional API key, the model (it fetches `/v1/models` from the endpoint and lets you pick — no more model-id typos), and whether to stream, then runs one tiny live completion through the exact final configuration and writes the profile.

```bash
scorpiox-openai-login --name local-llama     # write profile local-llama.txt
scorpiox-openai-login --status               # machine-readable status
```

Then switch to it with `/profile local-llama`. The wizard only talks to your endpoint for the operations you are actively performing (listing models and the confirmation completion) — nothing else is called.

---

## Running llama.cpp

llama.cpp's `llama-server` is the lightest way to serve a local model. Its OpenAI-compatible endpoint is at `/v1/chat/completions`, and current builds listen on **`127.0.0.1:9931`** by default (older builds used `8080`). Set the port explicitly so your base URL always matches:

```bash
llama-server \
  --model your-model.gguf \
  --host 127.0.0.1 \
  --port 8080 \
  --ctx-size 32768
```

| Flag | What it does |
|------|--------------|
| `-m, --model` | The GGUF model file to load. |
| `--host` / `--port` | Bind address. Defaults `127.0.0.1:9931`. |
| `-c, --ctx-size` | Maximum context window (tokens). `0` loads the size recorded in the model file. Size it to fit the model — see the note below. |

llama.cpp's Jinja chat-template engine is **enabled by default** on the server, so it accepts `chat_template_kwargs` from the request — including `enable_thinking` for reasoning models. Point the provider at it:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
MODEL=your-model
```

To set a standing thinking behavior for every request, add the template kwargs:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
MODEL=your-model
OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking": true}
```

Flip the value to `false` to turn thinking off for a model that defaults it on. On the server side you can do the same thing at launch with `--chat-template-kwargs '{"enable_thinking": true}'`, `--reasoning on` / `--reasoning off`, or `--reasoning-effort <level>`.

> **Check your slot, not just the port.** llama.cpp can serve multiple concurrent slots. Use the `/slots` slash command in SCORPIOX CODE (it reads `OPENAI_BASE_URL` if you don't pass a URL) to see live slot occupancy, or run `scorpiox-llamacpp-slots http://localhost:8080`.

> **Size the context to the model.** If you launch the server with a small `--ctx-size` and the conversation outgrows it, llama.cpp rejects the oversized prompt. SCORPIOX CODE no longer wedges on this: it reads the server's true context cap from the error, lowers its auto-compact threshold just under that cap, and triggers a compact so the session continues. Still, pick a `--ctx-size` the model actually supports — recovery keeps you going, it does not buy you a bigger window.

---

## Running vLLM

vLLM is the go-to for high-throughput serving, especially across a fleet of GPUs. Its `vllm serve` command exposes the OpenAI-compatible API on **port `8000`** by default.

```bash
vllm serve your-org/your-model \
  --host 127.0.0.1 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser llama3_json \
  --default-chat-template-kwargs '{"enable_thinking": false}'
```

| Flag | What it does |
|------|--------------|
| `serve <model>` | Positional model path (Hugging Face repo id or local path). This is the name that appears in `/v1/models` — set it as `MODEL`. |
| `--host` / `--port` | Bind address. Port defaults to `8000`. |
| `--enable-auto-tool-choice` | **Required for tool calling.** Without it the server does not parse tool calls, so the tool loop breaks. |
| `--tool-call-parser` | Which parser to use. **Required with `--enable-auto-tool-choice`.** Pick the one matching your model — common values include `llama3_json`, `llama4_json`, `hermes`, `mistral`, `deepseek_v3`, `qwen3_coder`, and `glm47`. |
| `--default-chat-template-kwargs` | A JSON object merged into every request's template kwargs (request-level values win). This is how you set a standing `enable_thinking` for a Qwen3 / DeepSeek-style reasoning model. |

vLLM merges `--default-chat-template-kwargs` with any request-level `chat_template_kwargs`, so you can set a server-wide default and still override per-session with `OPENAI_CHAT_TEMPLATE_KWARGS`.

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8000
MODEL=your-org/your-model
```

> **Tool calling is opt-in in vLLM.** This is the number-one reason a vLLM backend "doesn't call tools" under SCORPIOX CODE. You must pass **both** `--enable-auto-tool-choice` **and** a matching `--tool-call-parser`. If you set one without the other, vLLM errors at startup.

> **Server metrics.** vLLM exposes a Prometheus `/metrics` endpoint. Use the `/metrics` slash command in SCORPIOX CODE (reads `OPENAI_BASE_URL` by default) to pull live throughput and KV-cache numbers, or run `scorpiox-vllm-metrics <url>` from a shell.

---

## Running SGLang

SGLang is a strong choice for high-throughput and agentic workloads. Its `sglang serve` command is OpenAI-compatible and listens on **`127.0.0.1:30000`** by default.

```bash
sglang serve \
  --model-path your-org/your-model \
  --host 127.0.0.1 \
  --port 30000 \
  --tool-call-parser qwen3_coder \
  --reasoning-parser qwen3
```

| Flag | What it does |
|------|--------------|
| `--model-path` (`--model`) | Model path (Hugging Face repo id or local path). This is the name in `/v1/models` — set it as `MODEL`. |
| `--host` / `--port` | Bind address. Defaults `127.0.0.1:30000`. |
| `--tool-call-parser` | Selects the tool-call parser. Use `auto` to detect it from the chat template, or a model-matched name such as `qwen3_coder`, `llama3`, `hermes`, `deepseekv3`, or `mistral`. |
| `--reasoning-parser` | Selects the reasoning parser so thinking is surfaced cleanly; `auto` detects it from the chat template, or use a model-matched name (for example `qwen3`, `deepseek-r1`, `glm45`, `kimi_k2`). |
| `--default-chat-template-kwargs` | A JSON object applied to every request when not overridden per-request (keys such as `enable_thinking`, `thinking`, `reasoning_effort`). |

SGLang reads `chat_template_kwargs` from the request, including `enable_thinking`, so the SCORPIOX CODE knob reaches it directly — request-level values take precedence over the server-wide default:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:30000
MODEL=your-org/your-model
OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking": false}
```

> **Note the port.** SGLang's default port is `30000`, not `8080` or `8000`. A `404` or `connection refused` after a long prompt usually means the base URL is pointing at the wrong port. Confirm with `scorpiox-openai-models http://localhost:30000`.

> **Server metrics.** SGLang exposes Prometheus metrics plus a server-info endpoint. Run `scorpiox-sglang-metrics <url>` from a shell to pull token throughput and limits as clean JSON.

---

## Pointing at a real OpenAI-compatible API

For the actual OpenAI API (or Azure / any OpenAI-compatible cloud endpoint), the shape is identical — you just add the key and use the provider's base URL:

```
PROVIDER=openai
OPENAI_BASE_URL=https://api.openai.com
OPENAI_API_KEY=sk-...
MODEL=gpt-4o
```

Again, **no `/v1`** in the base URL — the path is appended for you. The key is only sent as a `Bearer` header when it is non-empty, so keep it out of your local profiles and only put it where a real endpoint expects it.

---

## Choosing and switching a model

- **List what the server offers:** the `/models` slash command (or `scorpiox-openai-models <url>` from a shell) prints the `/v1/models` list. The values in `models[].id` are your candidates for `MODEL`.
- **In-session switch:** `/model <name>` switches the active model without editing any file. This is the fastest way to A/B two models.
- **Persist a choice:** write `MODEL` into your user or project `scorpiox-env.txt`, or into a named profile.
- **Reasoning effort:** `/reasoning_effort <low|medium|high|max|off>` sets it in-session; `OPENAI_REASONING_EFFORT` is the config key. It is injected into the request as `reasoning_effort` and only has an effect on engines that read it (see [Reasoning Effort Control](reasoning-effort.md)).

---

## Identity headers on every request

Every request the OpenAI provider sends can carry a small set of **identity headers** — a per-session GUID and a per-conversation-lineage GUID — plus anything extra you choose to add. They travel in one config key, `OPENAI_EXTRA_HEADERS`, and the shipped default is:

```
OPENAI_EXTRA_HEADERS=X-Sx-Session-Id:{session_guid},X-Sx-Thread-Id:{thread_guid}
```

Both values are UUIDs generated on your machine; the placeholders are substituted per request, and adding your own pairs (a tenant tag, a correlation id) is a comma away. Set the key **empty** to send nothing at all. The full story — the two IDs, the placeholders, the opt-out, and the validation that refuses `Content-Type`/`Authorization` and CR/LF injection — is in [Session Identity Headers](identity-headers.md).

---

## How a request flows

1. You start a session with `PROVIDER=openai` and your base URL set.
2. For each turn, SCORPIOX CODE builds the request and sends it to `OPENAI_BASE_URL` + `/v1/chat/completions`.
3. The server replies in OpenAI Chat Completions format. SCORPIOX CODE translates it back and drives the tool loop.
4. When the model returns a tool call, SCORPIOX CODE executes the tool and sends the result back in the next request — repeating until the model finishes.

Because the translation is endpoint-agnostic, the exact same session works against llama.cpp, vLLM, SGLang, or a cloud API — the only thing that changes is `OPENAI_BASE_URL` (and the key, when one is needed).

If the endpoint returns a non-200, SCORPIOX CODE now surfaces the **backend's own error message** (OpenAI's `error.message`, or the top-level `message` that vLLM and SGLang emit) instead of a bare retry count, and passes the HTTP status through so retryable failures are told apart from permanent ones.

---

## Tuning for slow or large models

- **`OPENAI_TIMEOUT`** — raise it (for example `3600`) when prefill on a large MoE model exceeds the 30-minute default.
- **`OPENAI_STREAM`** — set `1` on servers that do not return per-request timing (vLLM, SGLang) so SCORPIOX CODE can stream internally, reassemble the body, and still measure time-to-first-token for `~pp` / `~tg`. Servers that ignore streaming fall back transparently.
- **`OPENAI_CHAT_TEMPLATE_KWARGS`** — the lever for per-engine behavior (thinking on/off, JSON mode, etc.) that is a chat-template concern rather than a generic API field.
- **`OPENAI_REASONING_EFFORT`** — how hard a reasoning model thinks before answering.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `OPENAI_BASE_URL not configured` | The key is empty | Set `OPENAI_BASE_URL` to your server origin. |
| `connection refused` | Wrong port, or server not running | Confirm the port (llama.cpp `9931`, vLLM `8000`, SGLang `30000`) and that the server is up. |
| `404` | Extra `/v1` in the URL, or wrong path | Set the origin only — see [The one gotcha](#the-one-gotcha-openai_base_url-is-an-origin-not-a-path). |
| Server does not call tools (vLLM) | Tool calling not enabled | Add `--enable-auto-tool-choice` **and** `--tool-call-parser`. |
| Model name rejected | `MODEL` does not match the server | List names with `/models` or `scorpiox-openai-models <url>` and use one of `models[].id`. |
| No thinking / reasoning | Engine not told to think | Set `OPENAI_CHAT_TEMPLATE_KWARGS` (and optionally `OPENAI_REASONING_EFFORT`). |
| Prompt rejected as too large (llama.cpp) | Context smaller than the conversation | SCORPIOX CODE learns the cap and compacts automatically; size `--ctx-size` to the model if it keeps recurring. |
| A cryptic upstream failure | The endpoint refused the request | Read the surfaced backend message (OpenAI / vLLM / SGLang error text) for the real reason. |

---

## Reference

- **Provider:** `PROVIDER=openai`
- **Base URL:** `OPENAI_BASE_URL` (origin only — `/v1/chat/completions` is appended); `OPENAI_API_BASE` is the fallback alias
- **Model keys:** `MODEL` (wins) / `OPENAI_MODEL` (fallback, defaults to `default`)
- **Auth:** `OPENAI_API_KEY` (optional for local servers)
- **Timeout:** `OPENAI_TIMEOUT` (seconds, default `1800`)
- **Streaming:** `OPENAI_STREAM` (`0`/`1`, default `0`)
- **Template behavior:** `OPENAI_CHAT_TEMPLATE_KWARGS` (JSON)
- **Reasoning:** `OPENAI_REASONING_EFFORT` (`low`/`medium`/`high`/`max`, or leave empty)
- **Identity headers:** `OPENAI_EXTRA_HEADERS` (default sends `X-Sx-Session-Id` / `X-Sx-Thread-Id`)
- **Helper commands:** `/models`, `/slots` (llama.cpp), `/metrics` (vLLM), `/model <name>`, `/reasoning_effort <level>`
- **Standalone tools:** `scorpiox-openai-login` (guided setup), `scorpiox-openai-models <url>`, `scorpiox-llamacpp-slots <url>`, `scorpiox-vllm-metrics <url>`, `scorpiox-sglang-metrics <url>`
- **See also:** [Configuration and Profiles](scorpiox-env.md), [Session Identity Headers](identity-headers.md), [Reasoning Effort Control](reasoning-effort.md), [API Traffic Logging](traffic-logging.md)
