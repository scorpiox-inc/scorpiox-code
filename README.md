# SCORPIOX CODE

High-performance native coding assistant with 100% local filesystem sessions.

Pure C99 architecture engineered for sub-millisecond CLI startup, complete offline auditability, and zero cloud lock-in.

---

## Architecture & Systems Engineering Truth

- **Pure C99**: 93,000 lines of hand-crafted C99. No interpreted layers.
- **78 Standalone Native C Binaries**: Dedicated Unix-philosophy helper binaries executing focused operational tasks.
- **Zero Dependencies**: No Node.js runtime, no Python daemon, no Electron, no npm packages.
- **100% Local Filesystem Sessions**: Transcripts and state stored as raw, transparent JSON directly on disk for complete offline auditability.
- **Wire-Level Transparency**: Verbatim on-disk HTTP traffic recording for full cryptographic and API auditability.

---

## Quick Install

Single-line installation scripts for all major operating systems:

### Windows (PowerShell)
```powershell
iwr -useb https://get.scorpiox.net | iex
```

### Linux
```bash
curl -fsSL "https://get.scorpiox.net?platform=linux" | bash
```

### macOS
```bash
curl -fsSL "https://get.scorpiox.net?platform=mac" | bash
```

---

## Supported AI Providers

SCORPIOX CODE connects natively to official subscription CLI tools and raw endpoints without token metering or vendor markups:

| Provider | Authentication & Architecture | Guide |
|---|---|---|
| **OpenAI Codex / ChatGPT** | OAuth device code authentication and automatic token refresh | [Codex Provider Guide](docs/codex-provider.md) |
| **Claude Code CLI** | Native OAuth session login and local background token daemon | [Claude Code Provider Guide](docs/claude-code-provider.md) |
| **GitHub Copilot CLI** | Official Copilot device-code login without API key meter limits | [Copilot Provider Guide](docs/copilot-provider.md) |
| **xAI Grok Build & SuperGrok** | Grok Build and SuperGrok subscription authentication | [Grok Provider Guide](docs/grok-provider.md) |
| **Google Antigravity CLI** | Gemini & Claude Sonnet over Google Cloud Code Assist | [Antigravity Provider Guide](docs/antigravity-provider.md) |
| **OpenAI Provider** | Direct support for local engines: llama.cpp, vLLM, SGLang, Ollama | [OpenAI Provider Guide](docs/openai-provider.md) |
| **Anthropic Direct** | Native Anthropic Messages API support | [Anthropic Provider Guide](docs/anthropic-provider.md) |

---

## Documentation Hub

Comprehensive, technical how-to guides for every subsystem:

### Configuration & Architecture
- [5-Tier Configuration Cascade](docs/scorpiox-env.md): Hierarchical cascade from DEFAULT to GLOBAL, USER, PROJECT, and PROFILE.
- [Project Instructions](docs/project-instructions.md): Context injection, instruction hierarchy, and prompt cache stability.
- [Scheduled Callbacks](docs/callbacks.md): Autonomous agent loops, scheduled callbacks, and background execution.
- [Keepalive & Cache Warming](docs/keepalive.md): Draggable live popup UI, prompt cache warming, and background pings.
- [Conversation Compaction](docs/conversation-compaction.md): Filesystem-native sessions, deterministic compaction vs lossy summarization, and raw session retention.
- [File Editing & Autonomy](docs/file-editing.md): Deterministic line-based file editing tools without container or sandbox lock-in.

### Fleet & System Integration
- [Skills System](docs/skills-system.md): Dynamic skill discovery, 4-tier cascade, and modular capabilities.
- [Hooks System](docs/hooks-system.md): Folder-based lifecycle event hooks with synchronous and asynchronous execution.
- [Model Context Protocol (MCP)](docs/mcp.md): Native pure C MCP client implementation.
- [SCORPIOX BOT](docs/scorpiox-bot.md): Remote fleet orchestration, headless C API, and web interface.

### Privacy & Auditing
- [Data Privacy](docs/data-privacy.md): Zero telemetry, zero external tracking, 100% local filesystem sessions.
- [Traffic Logging](docs/traffic-logging.md): Wire-level transparent HTTP traffic recording.
- [Privacy Policy](docs/privacy.md): Privacy guarantees across SCORPIOX CODE and SCORPIO BOT operational tiers.

---

## Ecosystem Links

- **Official Website**: [https://code.scorpiox.net](https://code.scorpiox.net)
- **Installer Service**: [https://get.scorpiox.net](https://get.scorpiox.net)
- **Remote Fleet UI**: [https://bot.scorpiox.net](https://bot.scorpiox.net)
