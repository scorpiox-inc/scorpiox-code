# SCORPIOX CODE

High-performance native coding assistant with 100% local filesystem sessions.

---

## Architecture & Systems Engineering Truth

Measured from the source tree at commit `d100592`:

- **Pure C** — 204,078 lines of hand-crafted C across 222 source files (228,462 lines including 97 headers). No runtime, no interpreter, no JIT.
- **119 standalone native C binaries** — Each capability compiles to its own self-contained executable target.
- **Zero dependencies** — No Node.js, no Python, no Electron, no `npm` packages, no package manager.
- **Sub-millisecond CLI startup** — No cold-start tax, no garbage collector, no VM warm-up.
- **100% local filesystem sessions** — Raw JSON transcripts on disk. Full offline auditability, replay, and wire-level traffic logging.
- **5-tier configuration cascade** — `DEFAULT -> GLOBAL -> USER -> PROJECT -> PROFILE`, with deterministic override semantics.
- **Zero telemetry** — No usage tracking, no remote logging. Privacy enforced by architecture, not by policy.

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
| Google Antigravity CLI Subscription (Gemini & Claude access via Cloud Code Assist) | [docs/antigravity-provider.md](docs/antigravity-provider.md) |
| OpenAI API & Subscription (compatible with llama.cpp, vLLM, SGLang, Ollama) | [docs/openai-provider.md](docs/openai-provider.md) |
| OpenCode Zen Subscription (OAuth device-code login, no separate API key) | [docs/opencode-provider.md](docs/opencode-provider.md) |
| Vertex AI Provider (Enterprise Google Cloud Model Garden) | [docs/vertex-ai-provider.md](docs/vertex-ai-provider.md) |
| Anthropic Direct Messages API & Custom Endpoints | [docs/anthropic-provider.md](docs/anthropic-provider.md) |

---

## Documentation Hub

| Guide | Description |
|---|---|
| [docs/scorpiox-env.md](docs/scorpiox-env.md) | 5-tier configuration cascade and profiles |
| [docs/callbacks.md](docs/callbacks.md) | Scheduled callbacks and autonomous agent loops |
| [docs/conversation-compaction.md](docs/conversation-compaction.md) | Deterministic compaction for long-horizon tasks |
| [docs/skills-system.md](docs/skills-system.md) | On-demand skills cascade |
| [docs/hooks-system.md](docs/hooks-system.md) | Folder-based lifecycle event hooks |
| [docs/mcp.md](docs/mcp.md) | Native MCP 2.0 & OAuth 2.1 client |
| [docs/keepalive.md](docs/keepalive.md) | Draggable live popup and cache warming |
| [docs/file-editing.md](docs/file-editing.md) | Deterministic line-based file editing |
| [docs/git-worktrees.md](docs/git-worktrees.md) | Isolated git worktrees for parallel agent sessions |
| [docs/project-instructions.md](docs/project-instructions.md) | `CLAUDE.md` / `AGENTS.md` instruction cascade |
| [docs/reasoning-effort.md](docs/reasoning-effort.md) | Reasoning effort control |
| [docs/lazy-skill-loading.md](docs/lazy-skill-loading.md) | Lazy skill loading |
| [docs/identity-headers.md](docs/identity-headers.md) | Session identity headers |
| [docs/data-privacy.md](docs/data-privacy.md) | Privacy architecture and zero data collection |
| [docs/traffic-logging.md](docs/traffic-logging.md) | Wire-level transparent HTTP traffic recording |
| [docs/usage-observability.md](docs/usage-observability.md) | Token usage observability |
| [docs/privacy.md](docs/privacy.md) | Privacy policy |
| [docs/scorpiox-bot.md](docs/scorpiox-bot.md) | Remote agent control and fleet management |
| [docs/bot-drop.md](docs/bot-drop.md) | Private file drop synced across devices |
| [docs/bot-notepad.md](docs/bot-notepad.md) | Private notepad synced across devices |
| [docs/bot-preview.md](docs/bot-preview.md) | Live artifact preview in bot chat |
| [docs/bot-vnc.md](docs/bot-vnc.md) | Live VNC desktops in bot chat |
| [docs/bot-plugins.md](docs/bot-plugins.md) | Plugins in SCORPIOX BOT |
| [docs/send-messages.md](docs/send-messages.md) | Sending messages to a running agent |
| [docs/scorpiox-tmux.md](docs/scorpiox-tmux.md) | Managing agent sessions with scorpiox-tmux |
| [docs/scorpiox-server.md](docs/scorpiox-server.md) | Serving sites with scorpiox-server |
| [docs/unshare.md](docs/unshare.md) | Native Linux container runtime for AI agents |
| [docs/docker.md](docs/docker.md) | Run SCORPIOX BOT in Docker |
