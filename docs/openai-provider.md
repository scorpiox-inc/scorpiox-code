# Using the OpenAI Provider

You have a local or self-hosted model — Qwen on a workstation, Llama on a cluster, a reasoning MoE on your GPU box — and you want SCORPIOX CODE to run against it instead of a hosted API. Or you want your agent traffic to stay on your network: fully local, fully private, no per-token cost.

If your model server speaks the **OpenAI Chat Completions API** — and the modern ones do — SCORPIOX CODE can drive it directly. `llama.cpp`, `vLLM`, and `SGLang` are the three engines most people self-host, and this page has a verified launch command for each, plus the configuration keys, model switching, and the failure modes you will actually hit.

Docs for SCORPIOX CODE @ `41b090b`.

> **The whole idea in one line:** point `PROVIDER=openai` at a running OpenAI-compatible server, name the model, and go. Tool calling, thinking blocks, and all the protocol translation between what SCORPIOX CODE speaks and what the server speaks is handled for you — you only ever configure the endpoint.

---

## How it works

SCORPIOX CODE speaks its own internal agent protocol (messages, tools, thinking). Your server speaks OpenAI Chat Completions. The openai provider sits in the middle and translates both directions:

- **Outbound:** each request is converted to `POST {OPENAI_BASE_URL}/chat/completions` with your model name, tools, and — when set — the optional extras (`reasoning_effort`, `chat_template_kwargs`).
- **Inbound:** tool calls come back as tool calls, and `reasoning_content` in the response becomes a thinking block, so reasoning models "think" visibly in your session instead of dumping raw tags into the answer.

You never write an adapter or a wrapper. One provider, one config block, any compatible server.

---

## The five keys that matter

Everything lives in your `scorpiox-env.txt` (see [Configuration and Profiles](scorpiox-env.md) for where that file lives and what wins). The keys the openai provider reads:

| Key | Required | What it does |
|---|---|---|
| `PROVIDER=openai` | Yes | Selects the openai provider. |
| `OPENAI_BASE_URL` | Yes | Your server's root, e.g. `http://localhost:8000/v1`. `OPENAI_API_BASE` is accepted as an alias. |
| `OPENAI_MODEL` | Yes in practice | The model name your server expects. This is passed through verbatim, so it must match `GET /v1/models` on your server. |
| `OPENAI_API_KEY` | Cloud only | Sent as a Bearer token. Not needed for local servers; required for OpenAI cloud or hosted endpoints. |
| `OPENAI_TIMEOUT` | No | Request timeout in seconds, default `1800` (30 min). Raise it for large MoE models with slow prefill. |

The rest are opt-in:

| Key | Default | When you want it |
|---|---|---|
| `OPENAI_STREAM` | `0` | Set `1` to stream internally (SSE) and reassemble. Lets you measure time-to-first-token on servers that do not return timings (vLLM, SGLang). |
| `OPENAI_CHAT_TEMPLATE_KWARGS` | empty | A JSON object forwarded as `chat_template_kwargs` — how you talk to a llama.cpp Jinja template, e.g. `{"enable_thinking": false}`. |
| `OPENAI_REASONING_EFFORT` | empty | `low` / `medium` / `high` / `max`. Forwarded as `reasoning_effort` for OpenAI-style reasoning models. |
| `OPENAI_SYSTEM_PROMPT` | empty | Per-provider system prompt override; empty means the global `SYSTEM_PROMPT` is used. |
| `OPENAI_EXTRA_HEADERS` | session headers | Extra HTTP headers on every request. Defaults carry session/thread IDs; set empty to opt out. |

Environment variables `OPENAI_API_KEY`, `OPENAI_BASE_URL`, and `OPENAI_API_BASE` override the config file when set in your shell — useful for CI and one-off runs.

---

## Quick start

```bash
# In your scorpiox-env.txt (or a profile — see below)
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8000/v1
OPENAI_MODEL=qwen3-30b
# OPENAI_API_KEY=            # leave empty for local servers

# Launch
sx
```

That is the entire setup. Inside the session:

- `/model <name>` switches the model for the session without touching any file.
- `/models [url]` opens a popup that lists whatever your server advertises at `GET /v1/models`, so you can copy the exact model id without guessing.
- From a shell, `scorpiox-openai-models http://localhost:8000` prints the model list as JSON — handy in scripts.

---

## Launching llama.cpp

llama.cpp is the natural choice for single-GPU and CPU workstations. Its OpenAI-compatible server is the `llama-server` binary; tool calling requires the Jinja chat template engine (`--jinja`), which is **enabled by default**, so a stock launch already works with tools.

```bash
llama-server \
  -m /models/qwen3-30b-q4_k_m.gguf \
  --jinja \
  -np 2 \
  -a qwen3-30b \
  --port 9931
```

Then:

```
PROVIDER=openai
OPENAI_BASE_URL=http://127.0.0.1:9931/v1
OPENAI_MODEL=qwen3-30b
```

| Flag | What it does |
|---|---|
| `-np 2` | Two parallel slots, so two agents (e.g. a sub-agent swarm) can talk to it at once. |
| `-a qwen3-30b` | Alias: the model id the API reports and you name in `OPENAI_MODEL`. Without it, the id is the file path. |
| `--port 9931` | The server's default port. Pick your own if 9931 is taken; use it verbatim in `OPENAI_BASE_URL`. |
| `--reasoning-budget N` | Cap thinking tokens for reasoning models (`-1` = unlimited, `0` = never think). |
| `--reasoning-effort LEVEL` | Server-side reasoning effort for templates that render it. |

Thinking on/off per request: llama.cpp Jinja templates accept `chat_template_kwargs`. For a Qwen3-style template, turn thinking off for a session with:

```
OPENAI_CHAT_TEMPLATE_KWARGS={"enable_thinking": false}
```

---

## Launching vLLM

vLLM is the choice for production GPU serving: continuous batching, PagedAttention, tensor parallelism. One important difference from the others: **tool calling is off by default** and must be enabled per model family, or the agent's tool calls will not round-trip.

```bash
vllm serve Qwen/Qwen3-30B-A3B-Instruct \
  --served-model-name qwen3-30b \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser qwen3 \
  --reasoning-parser qwen3
```

Then:

```
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8000/v1
OPENAI_MODEL=qwen3-30b
```

| Flag | What it does |
|---|---|
| `--enable-auto-tool-choice` | Required. Lets the server recognize and emit tool calls. |
| `--tool-call-parser <family>` | Required. The parser matching your model family (`qwen3`, `hermes`, `deepseek_v3`, `openai`, ...). Wrong parser = broken tool calls. |
| `--reasoning-parser <family>` | Enables `reasoning_content` in responses so thinking shows up properly in your session. |
| `--served-model-name` | The id to use in `OPENAI_MODEL`, decoupled from the Hugging Face path. |
| `--max-model-len` | Raise it for long-context work; the default is taken from the model config. |
| `--port 8000` | vLLM's default port. |

If your model is not in the built-in parser list, check vLLM's tool calling docs for the correct `--tool-call-parser` value — it is model-specific, not a guess.

---

## Launching SGLang

SGLang is built for high-throughput serving with RadixAttention prefix caching — a strong fit when many agents share long, similar system prompts (it reuses the cached prefix instead of re-prefilling it).

```bash
python -m sglang.launch_server \
  --model-path Qwen/Qwen3-30B-A3B-Instruct \
  --port 30000 \
  --trust-remote-code \
  --tp 1
```

Then:

```
PROVIDER=openai
OPENAI_BASE_URL=http://127.0.0.1:30000/v1
OPENAI_MODEL=Qwen/Qwen3-30B-A3B-Instruct
```

| Flag | What it does |
|---|---|
| `--port 30000` | SGLang's default HTTP port. Note it is **not** 8000 or 9931 — use the real port in `OPENAI_BASE_URL`. |
| `--tp N` | Tensor parallelism across N GPUs. |
| `--trust-remote-code` | Needed by some model repos whose tokenizer ships custom code. |
| `--dp N` | Data-parallel replicas for extra throughput. |

SGLang serves the model path as the model id by default, so `OPENAI_MODEL` can be the full Hugging Face path. Tool calling and reasoning content work for supported families with no extra flags; if your model's chat template is not in its tokenizer, pass one with `--hf-chat-template-name`.

---

## Profiles: one command per backend

You will have more than one backend — the local dev box, the GPU server, maybe OpenAI cloud. That is exactly what [profiles](scorpiox-env.md) are for. A profile file in `scorpiox-env/` overrides only the keys it names:

```bash
# ~/.claude/scorpiox-env/llamacpp.txt
PROVIDER=openai
OPENAI_BASE_URL=http://127.0.0.1:9931/v1
OPENAI_MODEL=qwen3-30b

# ~/.claude/scorpiox-env/vllm.txt
PROVIDER=openai
OPENAI_BASE_URL=http://gpu-server:8000/v1
OPENAI_MODEL=qwen3-30b
```

Switch without restarting:

```
/profile llamacpp      # load the profile
/use vllm              # session-only, no file written
/profile              # back to your base config
```

A profile can also set `STARTUP_DIR`, so each backend profile can carry its own working directory — one command gets you into the right mode, the right model, and the right repo.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `OPENAI_BASE_URL not configured` | The provider cannot find an endpoint. Set `OPENAI_BASE_URL` (or the `OPENAI_API_BASE` alias) in your config or environment. The URL is the root including `/v1`, e.g. `http://localhost:8000/v1`. |
| `model ... not found` from the server | `OPENAI_MODEL` does not match the server's id. Check `GET {base}/v1/models` (or `/models` in-session) and use the exact id — for llama.cpp that is your `--alias`, for vLLM your `--served-model-name`. |
| Tools not executing / agent stuck in a loop | The server is not emitting tool calls. vLLM: add `--enable-auto-tool-choice --tool-call-parser <family>`. llama.cpp: `--jinja` must be on (it is by default). |
| Thinking shows up as raw tags in the answer | The server is not populating `reasoning_content`. vLLM: add `--reasoning-parser <family>`; llama.cpp: set `--reasoning-format deepseek`. |
| Request times out on long prompts | Raise `OPENAI_TIMEOUT`. Large MoE models have slow prefill; `1800` seconds is the default, set more if needed. |
| `401` from a local server | Some servers enable auth. Either set `OPENAI_API_KEY` to match, or launch the server without it. |
| Port confusion between engines | The defaults differ: llama.cpp `9931`, vLLM `8000`, SGLang `30000`. Always copy the port from your server's startup log into `OPENAI_BASE_URL`. |

---

## What you get

One config block, any OpenAI-compatible server. Tool calling round-trips, reasoning models think visibly, `/model` and `/models` give you live control, and profiles make every backend one command away. The protocol is the one thing you do not have to think about anymore.

*Docs for SCORPIOX CODE @ `41b090b`.*
