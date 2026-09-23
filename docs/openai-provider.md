# Using the OpenAI Provider in SCORPIOX CODE

SCORPIOX CODE can talk to **any** server that exposes an OpenAI-compatible `/v1/chat/completions` endpoint. Set `PROVIDER=openai`, point `OPENAI_BASE_URL` at your server, and you are done — the provider handles the rest. This page walks through the configuration, how the request translation works, and gives **verified launch examples** for the three major self-hosted inference engines: **llama.cpp**, **vLLM**, and **SGLang**.

Source of truth: `sx_provider_openai.c`, `scorpiox-openai.c`, `scorpiox-openai-models.c`, `scorpiox-llamacpp-slots.c`, `scorpiox-vllm-metrics.c`, `sx_config_embedded.c`, `sx_slashcmd.c`, and `scorpiox-env.txt` at commit `6c70ad6`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **Local inference** | llama.cpp, vLLM, or SGLang running on your workstation or on the LAN |
| **Remote OpenAI-compatible API** | OpenAI, Azure OpenAI, Groq, Together, or any `/v1/chat/completions` proxy |
| **Air-gapped / on-prem** | Models served behind a firewall with no internet access |

If you are connecting to Anthropic, GitHub Copilot, Grok, Google, or another first-party service, use the dedicated `PROVIDER` value for that service instead (e.g. `anthropic`, `copilot`, `grok`, `google_gemini`). The OpenAI provider is the general-purpose path for everything that speaks the OpenAI Chat Completions wire format.

---

## Step 1 — Turn on the OpenAI provider

Set two keys and you are running:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
```

That is the entire minimum. `OPENAI_API_KEY` is **optional** for local servers that do not require authentication, and `OPENAI_MODEL` is **optional** — when it is empty the request sends `default` and lets the server decide which model to serve.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `openai` to activate this provider. |
| `OPENAI_BASE_URL` | text | `https://opus.scorpiox.net` | **Required in practice.** Base URL of the OpenAI-compatible server. The proxy appends `/v1/chat/completions` automatically — do **not** include that path. Override the embedded default to point at your own server. |
| `OPENAI_API_BASE` | text | *(empty)* | Legacy alias for `OPENAI_BASE_URL`. Used only when `OPENAI_BASE_URL` is empty. |
| `OPENAI_API_KEY` | text | *(empty)* | Bearer token sent in the `Authorization` header. Leave empty for local servers without authentication. |
| `OPENAI_MODEL` | text | *(empty → `default`)* | Model name passed in the `model` field of the request. The server decides what to run with it. |
| `OPENAI_TIMEOUT` | text | `1800` | HTTP request timeout in seconds (30 minutes). Raise this for very large models or slow hardware. |
| `OPENAI_CHAT_TEMPLATE_KWARGS` | text | *(empty)* | JSON object of extra Jinja chat-template keyword arguments, forwarded verbatim to the server (e.g. `{"thinking_mode":"enabled"}`). Used by llama.cpp and SGLang to control thinking. |
| `OPENAI_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, `high`, or `max`. When set, added to the request as `"reasoning_effort"`. |
| `REASONING_EFFORT` | choice | *(empty)* | Generic reasoning-effort key, same value set. |

> **You must point `OPENAI_BASE_URL` at your own server.** The embedded default is the SCORPIOX router (`https://opus.scorpiox.net`). For local inference, set it to your engine's address — the launch examples below show the exact value for each engine.

### Where the keys can live

**Any cascade tier of `scorpiox-env.txt`:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_MODEL=qwen3-30b-a3b
```

**A named profile** (create a file, e.g. `~/.scorpiox/scorpiox-env/local-llama.txt`):

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
OPENAI_TIMEOUT=3600
OPENAI_CHAT_TEMPLATE_KWARGS={"thinking_mode":"enabled"}
```

Activate it persistently or for the current session:

```bash
# Persistent (survives restarts — writes to the user-tier config)
/profile local-llama

# Session-only (no file change)
/use local-llama
```

**OS environment variables:**

```bash
export PROVIDER=openai
export OPENAI_BASE_URL=http://localhost:8080
export OPENAI_MODEL=qwen3-30b-a3b
```

OS environment variables override all file-based tiers, including the active profile.

Inspect what is actually resolved and where each value came from with:

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, OPENAI_BASE_URL, OPENAI_MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## How it works

SCORPIOX CODE internally speaks the Anthropic Messages API format. When `PROVIDER=openai` is active, every request is routed through the **`scorpiox-openai`** helper, which acts as a stateless translation proxy:

1. **Outbound** — the Anthropic Messages JSON is translated to an OpenAI Chat Completions request and `POST`ed to `OPENAI_BASE_URL/v1/chat/completions`.
2. **Inbound** — the OpenAI response is translated back to Anthropic Messages format.

The translation is what makes the same tool-calling, multi-turn, and image pipeline work unchanged across every OpenAI-compatible backend.

### Field translations worth knowing

- **Reasoning text.** The `reasoning_content` field in the response (or `reasoning`, which vLLM emits) is translated to an Anthropic `thinking` content block. Both field names are handled automatically.
- **`OPENAI_CHAT_TEMPLATE_KWARGS`.** Forwarded verbatim in the request body so llama.cpp's and SGLang's Jinja engines can enable thinking mode (e.g. `{"thinking_mode":"enabled"}` for Qwen3).
- **`OPENAI_REASONING_EFFORT`.** Forwarded as `"reasoning_effort"` for servers that support it.
- **Model pass-through.** The value in `OPENAI_MODEL` is sent as-is in the `model` field. If the server auto-detects a single loaded model (common with llama.cpp), the name in the request is informational only.

### Reasoning token accounting

The proxy reports `reasoning_tokens` to the status bar so the reasoning counter appears for local models. It uses a two-step strategy:

1. **Authoritative** — if the backend returns `usage.completion_tokens_details.reasoning_tokens` (OpenAI o-series style), that value is used directly.
2. **Estimated fallback** — when the backend returns thinking text but no `reasoning_tokens` (common with llama.cpp and vLLM), the count is estimated from the thinking text so the display stays useful.

### Retry behavior

Failed requests are retried with exponential backoff (1 s initial, 30 s max), and truncated or malformed responses are retried a few more times with a short backoff before surfacing an error.

---

## Verified launch examples

Each example below starts an engine and gives the exact SCORPIOX CODE keys to point at it. All three expose an OpenAI-compatible `/v1/chat/completions` endpoint.

### llama.cpp

llama.cpp's `llama-server` listens on **port 8080** by default (host `127.0.0.1`).

**Start the server:**

```bash
./llama-server \
    --model /path/to/model.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -c 16384 \
    -ngl 99 \
    -fa on \
    --jinja
```

- `--host 0.0.0.0` binds to all interfaces (the default is `127.0.0.1`, local only).
- `--port 8080` is the default, shown for clarity.
- `-c 16384` sets the context window (default is loaded from the model).
- `-ngl 99` offloads all layers to the GPU (aliases: `--gpu-layers`, `--n-gpu-layers`; default is `auto`).
- `-fa on` enables Flash Attention (`on`, `off`, or `auto`; default is `auto`).
- `--jinja` enables the Jinja chat-template engine. It is **enabled by default** in current builds, so this is only needed on older builds or to be explicit.

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
```

> **Tip:** for Qwen3 thinking models, add `OPENAI_CHAT_TEMPLATE_KWARGS={"thinking_mode":"enabled"}` so the Jinja template activates the model's built-in thinking mode.

**Watch throughput** with the `/slots` command (reads llama.cpp's `/slots` endpoint and reports prompt-processing and generation speed):

```
/slots              # uses OPENAI_BASE_URL
/slots http://localhost:8080
```

---

### vLLM

vLLM's `vllm serve` listens on **port 8000** by default.

**Start the server:**

```bash
vllm serve Qwen/Qwen3-30B-A3B \
    --host 0.0.0.0 \
    --port 8000 \
    --served-model-name qwen3-30b-a3b \
    --reasoning-parser qwen3
```

- `--host 0.0.0.0` and `--port 8000` bind the server (8000 is the default).
- `--served-model-name` sets the name returned by `/v1/models` and expected in requests. If omitted, vLLM uses the Hugging Face model ID.
- `--reasoning-parser` tells vLLM how to split thinking from the model's output. Common values: `qwen3`, `deepseek_r1`, `deepseek_v3`. (vLLM uses underscore naming.)

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8000
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
```

> **Note:** vLLM returns reasoning content in the `reasoning` field. The `scorpiox-openai` proxy handles both `reasoning` and `reasoning_content` automatically, so no extra configuration is needed.

**Watch throughput** with the `/metrics` command (reads vLLM's Prometheus `/metrics` endpoint):

```
/metrics              # uses OPENAI_BASE_URL
/metrics http://localhost:8000
```

---

### SGLang

SGLang's server listens on **port 30000** by default.

**Start the server:**

```bash
python -m sglang.launch_server \
    --model Qwen/Qwen3-30B-A3B \
    --host 0.0.0.0 \
    --port 30000 \
    --reasoning-parser qwen3
```

- `--model` (an alias for `--model-path`) is the Hugging Face repo ID or a local weights path.
- `--port 30000` is the default, shown for clarity.
- `--reasoning-parser` works the same as vLLM. Common values: `qwen3`, `qwen3-thinking`, `deepseek-r1`, `deepseek-v3`. (SGLang uses hyphen naming.)
- You can also bake thinking into every request at launch with `--default-chat-template-kwargs '{"enable_thinking": true}'`. Per-request `OPENAI_CHAT_TEMPLATE_KWARGS` still takes precedence.

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:30000
OPENAI_API_KEY=
OPENAI_MODEL=Qwen/Qwen3-30B-A3B
```

---

### Remote OpenAI-compatible endpoint

For remote APIs (OpenAI, Azure OpenAI, Groq, Together, or any compatible proxy):

```
PROVIDER=openai
OPENAI_BASE_URL=https://api.openai.com
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
```

For Azure OpenAI, point `OPENAI_BASE_URL` at your deployment's base URL and set the API key accordingly.

---

## Inspecting a running server

A few helper commands and binaries let you see what a server is actually doing without leaving the session:

| What | Command | What it reads |
|------|---------|---------------|
| List models | `/models` (or `/models <url>`) | the server's `/v1/models` endpoint |
| llama.cpp speed | `/slots` (or `/slots <url>`) | llama.cpp's `/slots` endpoint (prompt-processing + generation speed) |
| vLLM speed | `/metrics` (or `/metrics <url>`) | vLLM's Prometheus `/metrics` endpoint |

When a command is given no URL, it falls back to `OPENAI_BASE_URL`. Each also has a standalone binary you can run from a shell:

```bash
scorpiox-openai-models http://localhost:8080     # list /v1/models
scorpiox-llamacpp-slots http://localhost:8080    # llama.cpp slot speed
scorpiox-vllm-metrics http://localhost:8000      # vLLM /metrics speed
```

---

## Gotchas

- **Do not include `/v1/chat/completions` in the URL.** `OPENAI_BASE_URL` is a base; the provider appends the path. Point it at `http://localhost:8080`, not `http://localhost:8080/v1/chat/completions`.
- **`OPENAI_MODEL` is pass-through, not a catalog lookup.** Whatever you put there is sent to the server. If the server does not recognize it, the server's error is what you get. Match the name the server serves (see `/models` or `--served-model-name`).
- **`OPENAI_API_KEY` is optional for local engines, required for cloud endpoints.** Leaving it empty for a server that needs auth produces a 401.
- **Reasoning-parser naming differs per engine.** vLLM uses underscores (`deepseek_r1`); SGLang uses hyphens (`deepseek-r1`). Both accept `qwen3`. Check your engine's `--help` if a parser name is rejected.
- **The thinking kwargs key name differs per engine too.** llama.cpp typically expects `thinking_mode`, while SGLang's Qwen3 template often expects `enable_thinking`. Match the template your model ships.
- **Local servers bind to `127.0.0.1` by default.** If SCORPIOX CODE and the engine are on different machines, start the engine with `--host 0.0.0.0` and use the machine's LAN IP in `OPENAI_BASE_URL`.
- **The provider is stateless.** Each request is an independent translation; there is no per-server session state to keep warm on the provider side (see [the keepalive guide](keepalive.md) for cloud cache TTL, which does not apply to local engines).
