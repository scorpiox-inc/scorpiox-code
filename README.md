# SCORPIOX CODE

High-performance native coding assistant with 100% local filesystem sessions.

---

## Architecture & Systems Engineering Truth

- **93,000 lines of pure hand-crafted C99.** No runtime, no interpreter, no JIT.
- **78 standalone native C binaries.** Each capability compiles to its own self-contained executable target.
- **Zero dependencies.** No Node.js, no Python, no Electron, no `npm` packages, no package manager.
- **Sub-millisecond CLI startup.** No cold-start tax, no garbage collector, no VM warm-up.
- **100% local filesystem sessions.** Raw JSON transcripts on disk. Complete offline auditability, replay, and wire-level traffic logging.
- **5-tier configuration cascade.** `DEFAULT -> GLOBAL -> USER -> PROJECT -> PROFILE`, with deterministic override semantics.
- **Zero telemetry.** No usage tracking, no remote logging. Privacy enforced by architecture, not by policy.

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

SCORPIOX CODE talks to major LLM backends directly. Each guide covers authentication, configuration, and advanced options.

| Provider | Documentation |
|---|---|
| OpenAI Codex & ChatGPT Subscription (OAuth device-code login, token refresh) | [docs/codex-provider.md](docs/codex-provider.md) |
| Claude Code CLI Subscription (OAuth session login, local token daemon) | [docs/claude-code-provider.md](docs/claude-code-provider.md) |
| GitHub Copilot CLI Subscription (OAuth device-code login, no API-key metering) | [docs/copilot-provider.md](docs/copilot-provider.md) |
| xAI Grok Build & SuperGrok Subscription (CLI authentication) | [docs/grok-provider.md](docs/grok-provider.md) |
| Google Antigravity CLI Subscription (Gemini & Claude Sonnet on Cloud Code Assist) | [docs/antigravity-provider.md](docs/antigravity-provider.md) |
| OpenAI Provider (compatible with llama.cpp, vLLM, SGLang, Ollama) | [docs/openai-provider.md](docs/openai-provider.md) |
| Anthropic Direct Messages API & Custom Endpoints | [docs/anthropic-provider.md](docs/anthropic-provider.md) |
| OpenCode Zen Subscription (OAuth device-code login, no separate API key) | [docs/opencode-provider.md](docs/opencode-provider.md) |
| Vertex AI Provider (Enterprise Google Cloud Model Garden) | [docs/vertex-ai-provider.md](docs/vertex-ai-provider.md) |

---

## Documentation Hub

Direct links to every guide shipped in `docs/`.

| Guide | Description |
|---|---|
| [docs/scorpiox-env.md](docs/scorpiox-env.md) | 5-tier configuration cascade and profiles |
| [docs/callbacks.md](docs/callbacks.md) | Scheduled callbacks and autonomous agent loops |
| [docs/conversation-compaction.md](docs/conversation-compaction.md) | Deterministic compaction for long-horizon tasks |
| [docs/skills-system.md](docs/skills-system.md) | Dynamic on-demand skills cascade |
| [docs/hooks-system.md](docs/hooks-system.md) | Folder-based lifecycle event hooks |
| [docs/mcp.md](docs/mcp.md) | Native Model Context Protocol client |
| [docs/keepalive.md](docs/keepalive.md) | Draggable live popup and cache warming |
| [docs/file-editing.md](docs/file-editing.md) | Deterministic line-based file editing |
| [docs/data-privacy.md](docs/data-privacy.md) | Zero telemetry, zero external tracking |
| [docs/traffic-logging.md](docs/traffic-logging.md) | Wire-level transparent HTTP traffic recording |
| [docs/scorpiox-bot.md](docs/scorpiox-bot.md) | Headless fleet orchestration |
| [docs/project-instructions.md](docs/project-instructions.md) | `CLAUDE.md` / `AGENTS.md` instruction cascade |
| [docs/privacy.md](docs/privacy.md) | Privacy policy and zero-collection guarantee |

Extended guides:

| Guide | Description |
|---|---|
| [docs/git-worktrees.md](docs/git-worktrees.md) | Isolated git worktrees for parallel agent sessions |
| [docs/reasoning-effort.md](docs/reasoning-effort.md) | Reasoning effort control |
| [docs/lazy-skill-loading.md](docs/lazy-skill-loading.md) | Lazy skill loading |
| [docs/identity-headers.md](docs/identity-headers.md) | Session identity headers |
| [docs/sub-agents.md](docs/sub-agents.md) | Sub-agents and agent swarms |
| [docs/scorpiox-server.md](docs/scorpiox-server.md) | Serving sites with scorpiox-server |
| [docs/scorpiox-tmux.md](docs/scorpiox-tmux.md) | Managing agent sessions with scorpiox-tmux |
| [docs/send-messages.md](docs/send-messages.md) | Sending messages to a running agent |
| [docs/unshare.md](docs/unshare.md) | Native Linux container runtime for AI agents |
| [docs/usage-observability.md](docs/usage-observability.md) | Token usage observability |
| [docs/docker.md](docs/docker.md) | Run SCORPIOX BOT in Docker |
| [docs/bot-drop.md](docs/bot-drop.md) | Private file drop synced across devices |
| [docs/bot-notepad.md](docs/bot-notepad.md) | Private notepad synced across devices |
| [docs/bot-plugins.md](docs/bot-plugins.md) | Plugins in SCORPIOX BOT (BotMarket) |
| [docs/bot-preview.md](docs/bot-preview.md) | Live artifact preview in SCORPIOX BOT chat |
| [docs/bot-vnc.md](docs/bot-vnc.md) | Live VNC desktops in SCORPIOX BOT chat |

---

## Ecosystem

- Official Website: [https://code.scorpiox.net](https://code.scorpiox.net)
- Remote Fleet UI: [https://bot.scorpiox.net](https://bot.scorpiox.net)
