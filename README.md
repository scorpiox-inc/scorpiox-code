# SCORPIOX CODE

High-performance native coding assistant with 100% local filesystem sessions.

Pure C99 architecture engineered for sub-millisecond CLI startup, complete offline auditability, and zero cloud lock-in.

---

## Architecture & Systems Engineering Truth

- **Pure C**: 458,000 lines of hand-crafted C99. No interpreted layers. No runtime.
- **111 Standalone Native Binaries**: Dedicated Unix-philosophy helper binaries executing focused operational tasks.
- **Zero Dependencies**: No Node.js, no Python, no Electron, no package manager. One binary. Full system.
- **100% Local Filesystem Sessions**: Transcripts and state stored as raw, transparent JSON directly on disk for complete offline auditability.
- **Wire-Level Transparency**: Verbatim on-disk HTTP traffic recording for full cryptographic and API auditability.

---

## Quick Install

Single-line installation. No package manager required.

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

| Provider | Architecture | Guide |
|---|---|---|
| **Claude Code CLI** | Native OAuth session login, local background token daemon | [Guide](docs/claude-code-provider.md) |
| **GitHub Copilot CLI** | Official device-code login, no API key meter limits | [Guide](docs/copilot-provider.md) |
| **xAI Grok Build & SuperGrok** | Grok Build and SuperGrok subscription auth | [Guide](docs/grok-provider.md) |
| **Google Antigravity CLI** | Gemini & Claude Sonnet over Google Cloud Code Assist | [Guide](docs/antigravity-provider.md) |

---

## Documentation

Technical reference for every subsystem:

### Configuration & Architecture

- [Project Instructions](docs/project-instructions.md): Context injection, instruction hierarchy, and prompt cache stability.
- [Scheduled Callbacks](docs/callbacks.md): Autonomous agent loops, scheduled callbacks, and background execution.

### Core Operations

- [Keepalive & Cache Warming](docs/keepalive.md): Live popup UI, prompt cache warming, and background pings.
- [Conversation Compaction](docs/conversation-compaction.md): Filesystem-native sessions, deterministic compaction, and raw session retention.
- [File Editing & Autonomy](docs/file-editing.md): Deterministic line-based file editing tools without container lock-in.
- [Hooks System](docs/hooks-system.md): Folder-based lifecycle event hooks with synchronous and asynchronous execution.

### Privacy & Auditing

- [Data Privacy](docs/data-privacy.md): Zero telemetry, zero external tracking, 100% local filesystem sessions.

---

## Ecosystem

- **Website**: [code.scorpiox.net](https://code.scorpiox.net)
- **Install**: [get.scorpiox.net](https://get.scorpiox.net)
- **Repository**: [github.com/scorpiox-inc/scorpiox-code](https://github.com/scorpiox-inc/scorpiox-code)
