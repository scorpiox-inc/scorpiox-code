# Using the OpenAI Provider

SCORPIOX CODE can talk to **any** server that exposes an OpenAI-compatible `/v1/chat/completions` endpoint. Set `PROVIDER=openai`, point `OPENAI_BASE_URL` at your server — local or remote — and you are running. The provider translates the request wire format for you, so a single configuration drives llama.cpp, vLLM, SGLang, Ollama, LM Studio, a real OpenAI API key, Azure, or any other compatible endpoint.

This page is the full how-to: the keys that make it work, a **verified launch command** for each of the three modern self-hosted engines (llama.cpp, vLLM, SGLang), how to pick and switch a model, how to tune thinking, reasoning effort, and timeouts, and the one URL gotcha that trips almost everyone up.

Docs for SCORPIOX CODE @ `13253cf`.

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

## Setting the keys

### In `scorpiox-env.txt` (any cascade tier)

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
MODEL=your-model-name
```

### In a named profile

Create a file such as `~/.claude/scorpiox-env/local-llama.txt`:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
MODEL=your-model-name
OPENAI_TIMEOUT=3600
OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking": true}
```

Activate it persistently or for a single session:

```bash
# Persistent (survives restarts — written to the user tier)
/profile local-llama

# Session-only (no file change)
/use local-llama
```

---

## Running llama.cpp

llama.cpp's `llama-server` is the lightest way to serve a local model. Its OpenAI-compatible endpoint is at `/v1/chat/completions`, and it listens on **`127.0.0.1:8080`** by default.

```bash
llama-server \
  --model your-model.gguf \
  --host 127.0.0.1 \
  --port 8080 \
  --ctx-size 32768
```

| Flag | What it does |
|------|--------------|
| `--model` | The GGUF model file to load. |
| `--host` / `--port` | Bind address. Defaults `127.0.0.1:8080`. |
| `--ctx-size` | Maximum context window (tokens). `0` loads the size recorded in the model file. Size it to fit the model — see the note below. |

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

Flip the value to `false` to turn thinking off for a model that defaults it on.

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
| `--tool-call-parser` | Which parser to use. **Required with `--enable-auto-tool-choice`.** Pick the one matching your model — common values include `llama3_json`, `llama4_json`, `hermes`, `mistral`, `deepseek_v3`, and `qwen3_coder`. |
| `--default-chat-template-kwargs` | A JSON object merged into every request's template kwargs (request-level values win). This is how you set a standing `enable_thinking` for a Qwen3 / DeepSeek-style reasoning model. |

vLLM merges `--default-chat-template-kwargs` with any request-level `chat_template_kwargs`, so you can set a server-wide default and still override per-session with `OPENAI_CHAT_TEMPLATE_KWARGS`.

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8000
MODEL=your-org/your-model
```

> **Tool calling is opt-in in vLLM.** This is the number-one reason a vLLM backend "doesn't call tools" under SCORPIOX CODE. You must pass **both** `--enable-auto-tool-choice` **and** a matching `--tool-call-parser`. If you set one without the other, vLLM errors at startup.

> **Server metrics.** vLLM exposes a Prometheus `/metrics` endpoint. Use the `/metrics` slash command in SCORPIOX CODE (reads `OPENAI_BASE_URL` by default) to pull live throughput and KV-cache numbers.

---

## Running SGLang

SGLang is a strong choice for high-throughput and agentic workloads. Its `sglang serve` command is OpenAI-compatible and listens on **`127.0.0.1:30000`** by default.

```bash
sglang serve \
  --model-path your-org/your-model \
  --host 127.0.0.1 \
  --port 30000
```

| Flag | What it does |
|------|--------------|
| `--model-path` | Model path (Hugging Face repo id or local path). This is the name in `/v1/models` — set it as `MODEL`. |
| `--host` / `--port` | Bind address. Defaults `127.0.0.1:30000`. |

SGLang reads `chat_template_kwargs` from the request, including `enable_thinking`, so the SCORPIOX CODE knob reaches it directly. Tool calling is handled by a `tool_call_parser` that you point at a model-matched parser (it is `None`/auto by default, so the model's own template drives tool-call extraction):

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:30000
MODEL=your-org/your-model
OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking": false}
```

> **Note the port.** SGLang's default port is `30000`, not `8080` or `8000`. A `404` or `connection refused` after a long prompt usually means the base URL is pointing at the wrong port. Confirm with `scorpiox-openai-models http://localhost:30000`.

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
- **Reasoning effort:** `/reasoning_effort <low|medium|high|max|off>` sets it in-session; `OPENAI_REASONING_EFFORT` is the config key. It is injected into the request and only has an effect on engines that read it.

---

## How a request flows

1. You start a session with `PROVIDER=openai` and your base URL set.
2. For each turn, SCORPIOX CODE builds the request and sends it to `OPENAI_BASE_URL` + `/v1/chat/completions`.
3. The server replies in OpenAI Chat Completions format. SCORPIOX CODE translates it back and drives the tool loop.
4. When the model returns a tool call, SCORPIOX CODE executes the tool and sends the result back in the next request — repeating until the model finishes.

Because the translation is endpoint-agnostic, the exact same session works against llama.cpp, vLLM, SGLang, or a cloud API — the only thing that changes is `OPENAI_BASE_URL` (and the key, when one is needed).

---

## Tuning for slow or large models

- **`OPENAI_TIMEOUT`** — raise it (for example `3600`) when prefill on a large MoE model exceeds the 30-minute default.
- **`OPENAI_STREAM`** — set `1` on servers that do not return per-request timing (vLLM, SGLang) so SCORPIOX CODE can stream internally, reassemble the body, and still measure time-to-first-token for `~pp` / `~tg`. Servers that ignore streaming fall back transparently.
- **`OPENAI_CHAT_TEMPLATE_KWARGS`** — the lever for per-engine behavior (thinking on/off, JSON mode, etc.) that is a chat-template concern rather than a generic API field.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `OPENAI_BASE_URL not configured` | The key is empty | Set `OPENAI_BASE_URL` to your server origin. |
| `connection refused` | Wrong port, or server not running | Confirm the port (llama.cpp `8080`, vLLM `8000`, SGLang `30000`) and that the server is up. |
| `404` | Extra `/v1` in the URL, or wrong path | Set the origin only — see [The one gotcha](#the-one-gotcha-openai_base_url-is-an-origin-not-a-path). |
| Server does not call tools (vLLM) | Tool calling not enabled | Add `--enable-auto-tool-choice` **and** `--tool-call-parser`. |
| Model name rejected | `MODEL` does not match the server | List names with `/models` or `scorpiox-openai-models <url>` and use one of `models[].id`. |
| No thinking / reasoning | Engine not told to think | Set `OPENAI_CHAT_TEMPLATE_KWARGS` (and optionally `OPENAI_REASONING_EFFORT`). |
| Prompt rejected as too large (llama.cpp) | Context smaller than the conversation | SCORPIOX CODE learns the cap and compacts automatically; size `--ctx-size` to the model if it keeps recurring. |

---

## Reference

- **Provider:** `PROVIDER=openai`
- **Base URL:** `OPENAI_BASE_URL` (origin only — `/v1/chat/completions` is appended)
- **Model keys:** `MODEL` (wins) / `OPENAI_MODEL` (fallback, defaults to `default`)
- **Auth:** `OPENAI_API_KEY` (optional for local servers)
- **Timeout:** `OPENAI_TIMEOUT` (seconds, default `1800`)
- **Streaming:** `OPENAI_STREAM` (`0`/`1`, default `0`)
- **Template behavior:** `OPENAI_CHAT_TEMPLATE_KWARGS` (JSON)
- **Reasoning:** `OPENAI_REASONING_EFFORT` (`low`/`medium`/`high`/`max`, or leave empty)
- **Helper commands:** `/models`, `/slots` (llama.cpp), `/metrics` (vLLM), `/model <name>`, `/reasoning_effort <level>`
- **Standalone tools:** `scorpiox-openai-models <url>`, `scorpiox-llamacpp-slots <url>`, `scorpiox-vllm-metrics`
