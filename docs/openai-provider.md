# Using the OpenAI Provider

SCORPIOX CODE can talk to any server that exposes an OpenAI-compatible `/v1/chat/completions` endpoint. Set `PROVIDER=openai` and point `OPENAI_BASE_URL` at your server — local or remote. This page covers every configuration key, explains the translation layer, and gives verified launch examples for the three major self-hosted inference engines.

Source of truth: `scorpiox-openai.c`, `sx_provider_openai.c`, `sx_config.c`, and `scorpiox-config.c` at commit `5fd054b`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **Local inference** | llama.cpp, vLLM, or SGLang running on your workstation or LAN |
| **Remote OpenAI-compatible API** | OpenAI, Azure OpenAI, Together, Groq, any `/v1/chat/completions` proxy |
| **Air-gapped / on-prem** | Models served behind a firewall with no internet access |

If you are connecting to Anthropic, GitHub Copilot, Google, or another first-party provider, use the dedicated `PROVIDER` value for that service instead (e.g. `anthropic`, `copilot`, `google_gemini`).

---

## Configuration keys

All keys can be set in `scorpiox-env.txt` at any cascade tier, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for full cascade details.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `openai` to activate this provider. |
| `OPENAI_BASE_URL` | text | *(empty)* | **Required.** Base URL of the OpenAI-compatible server (e.g. `http://localhost:8080`). The proxy appends `/v1/chat/completions` automatically — do **not** include the path. |
| `OPENAI_API_BASE` | text | *(empty)* | Legacy alias for `OPENAI_BASE_URL`. Used only when `OPENAI_BASE_URL` is empty. |
| `OPENAI_API_KEY` | text | *(empty)* | Bearer token sent in the `Authorization` header. Leave empty for local servers that do not require authentication. |
| `OPENAI_MODEL` | text | *(empty)* | Model name passed to the server in the `model` field. If empty, `default` is sent. The server decides what model to run. |
| `OPENAI_TIMEOUT` | text | `1800` | HTTP request timeout in seconds (30 minutes). Increase for very large models or slow hardware. |
| `OPENAI_CHAT_TEMPLATE_KWARGS` | text | *(empty)* | JSON object of extra Jinja template keyword arguments forwarded to the server (e.g. `{"thinking_mode":"enabled"}`). Used by llama.cpp and SGLang to control chat template behavior. |
| `OPENAI_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort level: `low`, `medium`, `high`, or `max`. When set, added to the request as `"reasoning_effort"`. Useful for reasoning-capable models (e.g. OpenAI o-series). |

> **No hardcoded default URL.** Unlike some other providers, the OpenAI provider ships with an empty `OPENAI_BASE_URL`. You must set it explicitly or you will get an error: `error: OPENAI_BASE_URL not configured`.

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
OPENAI_CHAT_TEMPLATE_KWARGS={"thinking_mode":"enabled"}
```

Activate it persistently or per-session:

```bash
# Persistent (survives restarts — writes to user-tier scorpiox-env.txt)
/profile local-llama

# Session-only (no file change)
/use local-llama
```

### As OS environment variables

```bash
export PROVIDER=openai
export OPENAI_BASE_URL=http://localhost:8080
export OPENAI_MODEL=qwen3-30b-a3b
```

OS environment variables override all file-based tiers, including the active profile.

---

## How it works under the hood

SCORPIOX CODE internally speaks the Anthropic Messages API format. When `PROVIDER=openai` is active, every request is piped through the **`scorpiox-openai`** helper binary, which acts as a stateless translation proxy:

1. SCORPIOX CODE builds an Anthropic-format JSON request.
2. `scorpiox-openai` translates it to an OpenAI Chat Completions request and POSTs it to `{OPENAI_BASE_URL}/v1/chat/completions`.
3. The OpenAI-format response is translated back to Anthropic format and returned.

This happens transparently — you interact with SCORPIOX CODE the same way regardless of provider.

### Thinking / reasoning content

The proxy maps reasoning content between formats:

- **OpenAI → Anthropic:** The `reasoning_content` field (or `reasoning` for vLLM) in the response is translated to an Anthropic `thinking` content block.
- **llama.cpp `chat_template_kwargs`:** The `OPENAI_CHAT_TEMPLATE_KWARGS` value is forwarded verbatim in the request body so llama.cpp's Jinja engine can enable thinking mode (e.g. `{"thinking_mode":"enabled"}` for Qwen3).
- **`OPENAI_REASONING_EFFORT`:** Forwarded as `"reasoning_effort"` in the request for servers that support it.

### Retry behavior

The proxy retries failed requests up to 5 times with exponential backoff (1 s initial, 30 s max).

---

## Verified launch examples

### llama.cpp

llama.cpp's built-in server listens on port **8080** by default and exposes `/v1/chat/completions`.

**Start the server:**

```bash
./llama-server \
    --model /path/to/model.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    --ctx-size 16384 \
    --n-gpu-layers 99 \
    --jinja \
    --flash-attn
```

- `--jinja` is required for models that use Jinja chat templates (most modern models including Qwen3, DeepSeek, etc.) and for `chat_template_kwargs` passthrough.
- `--flash-attn` enables Flash Attention (recommended when supported by your GPU).
- `--n-gpu-layers 99` offloads all layers to GPU. Adjust for partial offload.

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
OPENAI_CHAT_TEMPLATE_KWARGS={"thinking_mode":"enabled"}
```

> **Tip:** For Qwen3 thinking models, set `OPENAI_CHAT_TEMPLATE_KWARGS={"thinking_mode":"enabled"}` so the Jinja template activates the model's built-in thinking mode.

---

### vLLM

vLLM listens on port **8000** by default and exposes an OpenAI-compatible API.

**Start the server:**

```bash
vllm serve Qwen/Qwen3-30B-A3B \
    --host 0.0.0.0 \
    --port 8000 \
    --reasoning-parser qwen3 \
    --served-model-name qwen3-30b-a3b
```

- `--reasoning-parser` tells vLLM how to extract thinking/reasoning from the model's output. Common values: `qwen3`, `deepseek_r1`, `deepseek_v3`.
- `--served-model-name` sets the model name returned by `/v1/models` and expected in requests. If omitted, vLLM uses the HuggingFace model ID.

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8000
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
```

> **Note:** vLLM returns reasoning content in the `reasoning_content` field (or `reasoning` on some versions). The `scorpiox-openai` proxy handles both field names automatically.

---

### SGLang

SGLang listens on port **30000** by default and exposes an OpenAI-compatible API.

**Start the server:**

```bash
python -m sglang.launch_server \
    --model Qwen/Qwen3-30B-A3B \
    --host 0.0.0.0 \
    --port 30000 \
    --reasoning-parser qwen3
```

- `--reasoning-parser` works similarly to vLLM. Common values: `qwen3`, `deepseek-r1`, `deepseek-v3`.
- SGLang supports `chat_template_kwargs` in the request body, so `OPENAI_CHAT_TEMPLATE_KWARGS` works here too.
- You can also pass `--default-chat-template-kwargs '{"enable_thinking": true}'` at server launch to apply thinking by default to all requests.

**Configure SCORPIOX CODE:**

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:30000
OPENAI_API_KEY=
OPENAI_MODEL=Qwen/Qwen3-30B-A3B
```

---

### Remote OpenAI-compatible endpoint

For remote APIs (OpenAI, Azure OpenAI, Together, Groq, or any compatible proxy):

```
PROVIDER=openai
OPENAI_BASE_URL=https://api.openai.com
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
```

For Azure OpenAI, point `OPENAI_BASE_URL` at your deployment's base URL (e.g. `https://my-resource.openai.azure.com/openai/deployments/my-deployment`) and set the API key accordingly.

---

## Helper binaries

Two companion binaries ship alongside `sx` and `scorpiox-openai`:

### scorpiox-openai

The Anthropic-to-OpenAI translation proxy. You do not invoke it directly — SCORPIOX CODE forks it automatically for each API call when `PROVIDER=openai` is active.

It reads the following environment variables at runtime:

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | Bearer token for the `Authorization` header |
| `OPENAI_BASE_URL` | Server base URL (falls back to `OPENAI_API_BASE`) |
| `OPENAI_TIMEOUT` | HTTP timeout in seconds (default: 1800) |
| `OPENAI_CHAT_TEMPLATE_KWARGS` | JSON kwargs forwarded in the request body |
| `OPENAI_REASONING_EFFORT` | Reasoning effort level forwarded in the request |
| `OPENAI_TRAFFIC_DIR` | Directory for raw traffic logging (set automatically by SCORPIOX CODE) |

### scorpiox-openai-models

Queries `{base_url}/v1/models` and prints a compact JSON list of available models. Useful for verifying that your server is running and checking which models it serves.

```bash
# List models from a local llama.cpp server
scorpiox-openai-models http://localhost:8080

# List models from a vLLM server
scorpiox-openai-models http://localhost:8000

# The /v1 suffix is also accepted
scorpiox-openai-models http://localhost:8080/v1
```

### scorpiox-llamacpp-slots

Polls a llama.cpp server's `/slots` endpoint and reports token generation speed and prompt processing speed. Useful for monitoring a local llama.cpp instance:

```bash
scorpiox-llamacpp-slots http://localhost:8080
```

---

## Gotchas

- **No hardcoded default URL.** `OPENAI_BASE_URL` defaults to empty. You must set it or `scorpiox-openai` will exit with an error. This is intentional — the provider does not assume any particular server address.

- **Empty `OPENAI_API_KEY` is valid.** Local servers (llama.cpp, vLLM, SGLang) typically do not require authentication. Leave the key empty or omit it entirely. When the key is empty, no `Authorization` header is sent.

- **`OPENAI_API_BASE` is a legacy alias.** It is checked only when `OPENAI_BASE_URL` is empty. If both are set, `OPENAI_BASE_URL` wins.

- **Do not include `/v1/chat/completions` in the URL.** The proxy appends the path automatically. Set only the base (e.g. `http://localhost:8080`, not `http://localhost:8080/v1/chat/completions`).

- **The default timeout is 30 minutes (1800 s).** Large MoE models with long prefill times may need more. Set `OPENAI_TIMEOUT=3600` (or higher) for slow models.

- **`OPENAI_CHAT_TEMPLATE_KWARGS` must be valid JSON.** It is injected verbatim into the request body. Malformed JSON will cause the server to reject the request. Example: `{"thinking_mode":"enabled"}`.

- **Streaming is not supported.** The OpenAI provider currently uses synchronous (non-streaming) requests regardless of the `STREAMING` setting. This is a known limitation.

- **Model name pass-through.** The value in `OPENAI_MODEL` is sent as-is in the `model` field of the request. The server decides what to do with it. If your server auto-detects the model (like llama.cpp with a single loaded model), the model name in the request is informational only.

- **Profile switching is live.** Switching profiles with `/profile` or `/use` triggers a full provider reload. The new `OPENAI_BASE_URL`, model, and all other keys take effect immediately without restarting SCORPIOX CODE.
