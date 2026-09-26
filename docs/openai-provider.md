# Using the OpenAI Provider

SCORPIOX CODE can talk to any server that exposes an OpenAI-compatible `/v1/chat/completions` endpoint. Set `PROVIDER=openai` and point `OPENAI_BASE_URL` at your server — local or remote. This page covers every configuration key, explains the translation layer, and gives verified launch examples for the three major self-hosted inference engines: **llama.cpp**, **vLLM**, and **SGLang**.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **Local inference** | llama.cpp, vLLM, or SGLang running on your workstation or LAN |
| **Remote OpenAI-compatible API** | OpenAI, Azure OpenAI, Together, Groq, xAI (API key), or any `/v1/chat/completions` proxy |
| **Air-gapped / on-prem** | Models served behind a firewall with no internet access |

If you are connecting to Anthropic, GitHub Copilot, Google, Grok, or another first-party provider, use the dedicated `PROVIDER` value for that service instead (for example `anthropic`, `copilot`, `google_gemini`, `grok`). The OpenAI provider is the right fit when you have an OpenAI **API key** or a **self-hosted OpenAI-compatible server**.

> **API keys only.** The Grok subscription (OAuth login) and Codex paths are separate providers. To call xAI with an *API key* (`XAI_API_KEY`), use this provider with `OPENAI_BASE_URL=https://api.x.ai` — that is the billing path that accepts a key, distinct from the Grok subscription provider.

---

## Configuration keys

All keys can be set in `scorpiox-env.txt` at any cascade tier, in a named profile, or as OS environment variables. Environment variables override the file values. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | — | Set to `openai` to activate this provider. |
| `OPENAI_BASE_URL` | text | *(empty)* | Base URL of the OpenAI-compatible server. SCORPIOX CODE appends `/v1/chat/completions` automatically — set only the host, **do not** include the path. There is no built-in default at this commit, so you must set this for every deployment. |
| `OPENAI_API_BASE` | text | *(empty)* | Legacy alias for `OPENAI_BASE_URL`. Used only when `OPENAI_BASE_URL` is empty. |
| `OPENAI_API_KEY` | text | *(empty)* | Bearer token sent in the `Authorization` header. Leave empty for local servers that do not require authentication. |
| `OPENAI_MODEL` | text | `default` | Model name passed to the server in the `model` field. If you leave it unset, the literal `default` is sent. The server decides which model it actually runs, so match it to a model your server serves. |
| `OPENAI_TIMEOUT` | text | `1800` | HTTP request timeout in seconds (30 minutes). Increase for very large models or slow hardware. |
| `OPENAI_CHAT_TEMPLATE_KWARGS` | text | *(empty)* | JSON object of extra Jinja template keyword arguments forwarded to the server (for example `{"enable_thinking":"true"}`). Used by llama.cpp and SGLang to control chat-template behavior. |
| `OPENAI_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort level: `low`, `medium`, `high`, or `max`. When set, it is added to the request as `reasoning_effort`. Empty means it is not sent. Useful for reasoning-capable models (for example OpenAI o-series). |

> **You own the URL.** The shipped configuration ships `OPENAI_BASE_URL` **empty** at this commit. There is no built-in endpoint to fall back to, so a blank value produces a `OPENAI_BASE_URL not configured` error the first time a request is attempted. Point it at your own server explicitly — `http://localhost:8080` for llama.cpp, `http://localhost:8000` for vLLM, `http://localhost:30000` for SGLang.

> **Set the host only.** Because the `/v1/chat/completions` suffix is appended for you, `OPENAI_BASE_URL` should be the scheme and host (optionally with a port) and nothing else. `http://localhost:8080` is correct; `http://localhost:8080/v1/chat/completions` is not.

---

## Setting the keys

### In scorpiox-env.txt (any cascade tier)

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
```

### In a named profile

Create a file such as `~/.claude/scorpiox-env/local-llama.txt`:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
OPENAI_TIMEOUT=3600
OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking":"true"}
```

Activate it persistently or for a single session:

```bash
# Persistent (survives restarts)
/profile local-llama

# Session-only (no file change)
/use local-llama
```

Both commands trigger a live provider reload, so the new endpoint and model take effect immediately.

---

## How the translation works

The OpenAI provider is a thin translation layer. Every turn, it converts the internal Anthropic-format request into an OpenAI Chat Completions request, sends it to your server, and converts the response back.

1. **Outbound:** the internal request is translated into an OpenAI Chat Completions request and POSTed to `{OPENAI_BASE_URL}/v1/chat/completions`.
2. **Inbound:** the OpenAI response is translated back and returned.

This happens transparently — you interact with SCORPIOX CODE the same way regardless of provider.

### Field translations

| Internal (Anthropic) | OpenAI (wire) |
|----------------------|---------------|
| `model` | `model` |
| `messages` (role + content) | `messages` (role + content) |
| `system` | `system` (merged into the messages where the server expects it) |
| `tools` | `tools` |
| `max_tokens` | `max_tokens` |
| — | `chat_template_kwargs` (injected from `OPENAI_CHAT_TEMPLATE_KWARGS`) |
| — | `reasoning_effort` (injected from `OPENAI_REASONING_EFFORT`) |

### Thinking and reasoning content

The layer maps reasoning content between formats. When the server returns a `reasoning_content` or `reasoning` field (as llama.cpp and vLLM do), it is surfaced as a thinking block in the UI. Both field names are handled automatically.

**Reasoning token accounting.** The status bar shows a reasoning-token counter for local models using a two-step strategy:

1. **Authoritative:** if the backend returns `usage.completion_tokens_details.reasoning_tokens` (OpenAI o-series style), that value is used directly.
2. **Estimated fallback:** when the backend returns thinking text but no `reasoning_tokens` (common with llama.cpp and vLLM), the count is estimated from the thinking text's word count. This keeps the display useful even when the backend does not track reasoning tokens natively.

### Retry behavior

Failed requests are retried up to **5 times** with exponential backoff (starting at 1 second, capped at 30 seconds). Truncated or malformed responses (for example a tool-call `arguments` string cut off at `finish_reason=length`) are retried a few times with a short backoff before an error is surfaced.

---

## Verified launch examples

> Model used in all three examples: **Qwen/Qwen3-30B-A3B**. Swap in any model your server serves.

### llama.cpp

llama.cpp's built-in server listens on **`127.0.0.1:8080`** by default and exposes `/v1/chat/completions`.

**Start the server:**

```bash
./llama-server \
    --model /path/to/model.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    --ctx-size 16384 \
    --n-gpu-layers 99 \
    --jinja \
    --reasoning-format deepseek \
    --flash-attn
```

- `--jinja` enables the Jinja chat-template engine. It is on by default for the server, but pass it explicitly on older builds — it is required for `chat_template_kwargs` passthrough and for models that ship Jinja templates (Qwen3, DeepSeek, and most modern models).
- `--reasoning-format deepseek` puts the model's thinking into `message.reasoning_content` so SCORPIOX CODE can surface it cleanly. Other values: `none` (leave thoughts in content) and `deepseek-legacy` (keep the tags in content while also populating `reasoning_content`). Default is `auto`.
- `--n-gpu-layers 99` offloads all layers to the GPU. Lower it for partial offload.
- `--flash-attn` enables Flash Attention (recommended when your GPU supports it).

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking":"true"}
```

> **Tip:** For Qwen3 thinking models, set `OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking":"true"}` so the Jinja template activates the model's built-in thinking mode. Set it to `{"enable_thinking":"false"}` to turn thinking off.

---

### vLLM

vLLM listens on **`8000`** by default and exposes an OpenAI-compatible API.

**Start the server:**

```bash
vllm serve Qwen/Qwen3-30B-A3B \
    --host 0.0.0.0 \
    --port 8000 \
    --reasoning-parser qwen3 \
    --served-model-name qwen3-30b-a3b
```

- `--reasoning-parser` tells vLLM how to extract thinking from the model's output. Common values use underscores: `qwen3`, `deepseek_r1`, `deepseek_v3`.
- `--served-model-name` sets the model name returned by `/v1/models` and expected in requests. If omitted, vLLM uses the HuggingFace model ID.

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8000
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
```

> **Note:** vLLM returns reasoning content in the `reasoning_content` field (or `reasoning` on some versions). The translation layer handles both field names automatically.

---

### SGLang

SGLang listens on **`30000`** by default and exposes an OpenAI-compatible API.

**Start the server:**

```bash
python -m sglang.launch_server \
    --model Qwen/Qwen3-30B-A3B \
    --host 0.0.0.0 \
    --port 30000 \
    --reasoning-parser qwen3
```

- `--reasoning-parser` works the same way as in vLLM, but SGLang names use dashes. Common values: `qwen3`, `qwen3-thinking`, `deepseek-r1`, `deepseek-v3`.
- SGLang accepts `chat_template_kwargs` in the request body, so `OPENAI_CHAT_TEMPLATE_KWARGS` works here too.
- Alternatively, pin a server-wide default at launch with `--default-chat-template-kwargs '{"enable_thinking": true}'` to apply thinking to every request. A per-request `chat_template_kwargs` (from `OPENAI_CHAT_TEMPLATE_KWARGS`) takes precedence over the server default.

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:30000
OPENAI_API_KEY=
OPENAI_MODEL=Qwen/Qwen3-30B-A3B
```

---

### Remote OpenAI-compatible endpoint

For remote APIs (OpenAI, Azure OpenAI, Together, Groq, xAI, or any compatible proxy):

```
PROVIDER=openai
OPENAI_BASE_URL=https://api.openai.com
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
```

For Azure OpenAI, point `OPENAI_BASE_URL` at your deployment's base URL (for example `https://my-resource.openai.azure.com/openai/deployments/my-deployment`) and set the API key accordingly. For xAI with an API key, use `OPENAI_BASE_URL=https://api.x.ai` and your `XAI_API_KEY`.

---

## Discovering and monitoring models

A set of companion utilities ships alongside the main binary and the OpenAI translation proxy.

### `/models` — model picker

Inside a session, the `/models` command opens a live popup that polls your server's `/v1/models` endpoint and lists the available models. With no argument it defaults to `OPENAI_BASE_URL`; pass a URL to inspect a specific server.

```
/models                          # use OPENAI_BASE_URL
/models http://localhost:8000    # inspect a specific server
```

### `/model` — show or switch model

`/model` shows the active model and lets you switch it for the session.

### Companion CLI tools

These are useful from a shell to verify a server is up and to watch throughput.

| Tool | What it does | Example |
|------|--------------|---------|
| `scorpiox-openai-models` | Queries `{base_url}/v1/models` and prints a compact list. Strips a trailing `/v1` or `/models` for you, so both a plain base URL and a pasted `/v1/models` URL work. | `scorpiox-openai-models http://localhost:8080` |
| `scorpiox-llamacpp-slots` | Polls a llama.cpp server's `/slots` endpoint and reports token generation and prompt processing speed. | `scorpiox-llamacpp-slots http://localhost:8080` |
| `scorpiox-vllm-metrics` | Polls a vLLM server's Prometheus `/metrics` endpoint and reports generation and prompt speed from the counters. | `scorpiox-vllm-metrics http://localhost:8000` |

---

## Gotchas

- **You must set `OPENAI_BASE_URL`.** At this commit the default is empty and there is no built-in endpoint. A blank value fails on the first request with a `OPENAI_BASE_URL not configured` error. Point it at your own server — `http://localhost:8080` (llama.cpp), `http://localhost:8000` (vLLM), or `http://localhost:30000` (SGLang).

- **Do not include the path in the URL.** SCORPIOX CODE appends `/v1/chat/completions` automatically. Set only the host — `http://localhost:8080`, not `http://localhost:8080/v1/chat/completions`. (The `/models` helper and `scorpiox-openai-models` are more forgiving and strip a trailing `/v1`, but the provider itself does not.)

- **Match `OPENAI_MODEL` to what your server serves.** The value is passed through verbatim; if you leave it unset the literal string `default` is sent, and a server that has no model named `default` will reject it. Use the same name the server returns from `/v1/models`.

- **`OPENAI_API_KEY` is optional for local servers.** Leave it empty when llama.cpp, vLLM, or SGLang runs without authentication. Set it for OpenAI, Azure, Together, Groq, or any remote key-protected endpoint.

- **llama.cpp thinking needs `--jinja` and the right `--reasoning-format`.** Without `--jinja`, `OPENAI_CHAT_TEMPLATE_KWARGS` like `{"enable_thinking":"true"}` has no effect. Use `--reasoning-format deepseek` so the thinking lands in `reasoning_content` where SCORPIOX CODE reads it.

- **Reasoning parser names differ between vLLM and SGLang.** vLLM uses underscores (`qwen3`, `deepseek_r1`); SGLang uses dashes (`qwen3`, `deepseek-r1`). Set the parser that matches your engine and model, or the thinking will not be split out of the content.
