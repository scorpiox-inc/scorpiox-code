# SCORPIOX CODE

High-performance native coding assistant with 100% local filesystem sessions.

---

## Architecture & Systems Engineering Truth

- **Pure C99** — 93,000 lines of hand-crafted C99. No runtime, no interpreter, no JIT.
- **78 standalone native C binaries** — Each capability compiles to its own self-contained executable.
- **Zero dependencies** — No Node.js, no Python, no Electron, no `npm` packages. One binary, statically linked.
- **Sub-millisecond CLI startup** — No cold-start tax. The binary is always ready.
- **100% local filesystem sessions** — Raw JSON transcripts on disk. Full offline auditability, replay, and wire-level traffic logging.

---

## Quick Install

One command. One binary. No package manager required.

**Windows (PowerShell)**

```powershell
iwr -useb https://get.scorpiox.net | iex
```

**Linux**

```bash
curl -fsSL "https://get.scorpiox.net?platform=linux" | bash
```

**macOS**

```bash
curl -fsSL "https://get.scorpiox.net?platform=mac" | bash
```

---

## Supported AI Providers

SCORPIOX CODE works with any major LLM backend. Each guide covers authentication, configuration, and advanced options.

| Provider | Documentation |
|---|---|
| OpenAI Codex & ChatGPT Subscription (OAuth device-code login, token refresh) | [docs/codex-provider.md](docs/codex-provider.md) |
| Claude Code CLI Subscription (OAuth session login, local token daemon) | [docs/claude-code-provider.md](docs/claude-code-provider.md) |
| GitHub Copilot CLI Subscription (OAuth device-code login, no API-key metering) | [docs/copilot-provider.md](docs/copilot-provider.md) |
| xAI Grok Build & SuperGrok Subscription | [docs/grok-provider.md](docs/grok-provider.md) |
| Google Antigravity CLI Subscription (Gemini & Claude Sonnet on Cloud Code Assist) | [docs/antigravity-provider.md](docs/antigravity-provider.md) |
| OpenAI Provider (compatible with llama.cpp, vLLM, SGLang, Ollama) | [docs/openai-provider.md](docs/openai-provider.md) |
| Anthropic Direct Messages API | [docs/anthropic-provider.md](docs/anthropic-provider.md) |

---

## Configuration

SCORPIOX CODE uses a deterministic 5-tier configuration cascade: environment variables, project-level settings, user settings, system defaults, and compiled-in fallbacks — resolved in a fixed priority order.

Full reference: [docs/scorpiox-env.md](docs/scorpiox-env.md)

---

## Documentation Hub

| Guide | Description |
|---|---|
| [docs/scorpiox-env.md](docs/scorpiox-env.md) | 5-tier configuration cascade |
| [docs/callbacks.md](docs/callbacks.md) | Scheduled callbacks and autonomous agent loops |
| [docs/conversation-compaction.md](docs/conversation-compaction.md) | Deterministic compaction vs lossy summarization |
| [docs/skills-system.md](docs/skills-system.md) | Dynamic skills cascade |
| [docs/hooks-system.md](docs/hooks-system.md) | Lifecycle event hooks |
| [docs/mcp.md](docs/mcp.md) | Model Context Protocol native client |
| [docs/keepalive.md](docs/keepalive.md) | Draggable live popup & cache warming |
| [docs/file-editing.md](docs/file-editing.md) | Deterministic line-based file editing |
| [docs/data-privacy.md](docs/data-privacy.md) | Zero telemetry, zero external tracking |
| [docs/traffic-logging.md](docs/traffic-logging.md) | Wire-level transparent HTTP traffic recording |
| [docs/scorpiox-bot.md](docs/scorpiox-bot.md) | Headless fleet orchestration |
| [docs/project-instructions.md](docs/project-instructions.md) | Instruction cascade |
| [docs/privacy.md](docs/privacy.md) | Privacy policy |

---

## Ecosystem

- Official Website: [https://code.scorpiox.net](https://code.scorpiox.net)
- Remote Fleet UI (SCORPIO BOT): [https://bot.scorpiox.net](https://bot.scorpiox.net)

---

## License

Proprietary. All rights reserved.
