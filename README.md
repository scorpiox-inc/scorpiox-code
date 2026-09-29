# SCORPIOX CODE

High-performance native coding assistant with 100% local filesystem sessions.

Pure C99 systems architecture. 463K lines of hand-engineered C. 111 standalone binaries. Zero dependencies. Zero cloud lock-in.

---

## Why SCORPIOX CODE

- **Pure C99** — No runtime, no interpreter, no JIT. Compiles to a single statically linked binary.
- **Sub-millisecond startup** — No cold-start tax. The binary is always ready.
- **100% local** — All sessions, transcripts, and state live on your filesystem. Nothing leaves your machine unless you explicitly send it to an LLM API.
- **Zero dependencies** — One binary. No `node_modules`, no `pip`, no `npm`. It just works.
- **Offline-auditable** — Every interaction is logged locally. Full conversation compaction, traffic logging, and session replay without a network round-trip.

---

## Installation

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

One command. One binary. No package manager required.

---

## Supported AI Providers

SCORPIOX CODE works with any major LLM backend. Each provider guide covers authentication, configuration, and advanced options.

| Provider | Documentation |
|---|---|
| OpenAI Codex & ChatGPT Subscription | [docs/codex-provider.md](docs/codex-provider.md) |
| Claude Code CLI Subscription | [docs/claude-code-provider.md](docs/claude-code-provider.md) |
| GitHub Copilot CLI Subscription | [docs/copilot-provider.md](docs/copilot-provider.md) |
| xAI Grok Build & SuperGrok Subscription | [docs/grok-provider.md](docs/grok-provider.md) |
| Google Antigravity CLI Subscription | [docs/antigravity-provider.md](docs/antigravity-provider.md) |
| OpenAI Provider & Local GPU Backends | [docs/openai-provider.md](docs/openai-provider.md) |

---

## Configuration

SCORPIOX CODE uses a deterministic 5-tier configuration cascade. Environment variables, project-level settings, user settings, system defaults, and compiled-in fallbacks are resolved in a fixed priority order.

Full reference: [docs/project-instructions.md](docs/project-instructions.md)

---

## Advanced Features

| Feature | Documentation |
|---|---|
| Scheduled Callbacks & Autonomous Agent Loops | [docs/callbacks.md](docs/callbacks.md) |
| Skills System | [docs/skills-system.md](docs/skills-system.md) |
| Conversation Compaction | [docs/conversation-compaction.md](docs/conversation-compaction.md) |
| File Editing | [docs/file-editing.md](docs/file-editing.md) |
| Keepalive | [docs/keepalive.md](docs/keepalive.md) |
| Traffic Logging | [docs/traffic-logging.md](docs/traffic-logging.md) |
| Data Privacy | [docs/data-privacy.md](docs/data-privacy.md) |

---

## Architecture

```
scorpiox-code/
  src/              Pure C99 source (463K lines)
  docs/             All documentation lives here
  .claude/          Agent skills and project configuration
```

- **111 standalone binaries** — Each capability is a separate, self-contained executable.
- **No shared libraries** — Static linking throughout. The binary you install is the binary that runs.
- **C99 standard** — No extensions, no non-portable constructs. Builds cleanly on any POSIX or MSVC toolchain.

---

## License

Proprietary. All rights reserved.
