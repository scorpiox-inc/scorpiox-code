# Configuration Cascade and Environment Profiles

SCORPIOX CODE resolves every configuration key through a **multi-tier cascade** with predictable precedence. Base defaults ship compiled into the binary; machine-wide globals, user preferences, per-repository overrides, and named profiles layer on top. The last writer wins — a key set in a higher tier silently overrides the same key from a lower tier.

Source of truth: `sx_config.h` and `sx_config.c` at commit `5fd054b`.

## The 5 tiers (lowest → highest)

| # | Tier | File / location | When it applies |
|---|------|-----------------|-----------------|
| 0 | **Default** | Compiled into the binary (`sx_config_defaults[]`) | Always — every key has a built-in fallback. |
| 1 | **Global** | `<exe_dir>/scorpiox-env.txt` | Next to the `sx` binary. Shared across all users on the machine. |
| 2 | **User** | `~/.claude/scorpiox-env.txt` | Per-user settings. On Windows `%USERPROFILE%\.claude\scorpiox-env.txt`. |
| 3 | **Project** | `CWD/.claude/scorpiox-env.txt` *or* `CWD/.scorpiox/scorpiox-env.txt` | Per-repository overrides. If both exist, `.scorpiox/` loads second and wins on conflict. `.claude/` is git-friendly (usually tracked); `.scorpiox/` is the traditional path. |
| 4 | **Profile** | `scorpiox-env/<name>.txt` (resolved from any tier's directory) | A named overlay activated by `ACTIVE_PROFILE`. Only the keys it explicitly defines override earlier tiers. |

**OS environment variables** are checked on every `sx_config_get()` call and beat all file-based tiers, including profiles. For example, `export PROVIDER=openai` in your shell overrides `PROVIDER` from every `scorpiox-env.txt` and from the active profile.

The cascade is loaded once at startup by `sx_config_init()` and can be reloaded at any time (e.g. after a `/profile` switch) via `sx_config_reload()`.

## How profiles work

A profile is a plain-text `KEY=VALUE` file stored in a `scorpiox-env/` directory at any cascade tier:

| Tier | Profile directory |
|------|-------------------|
| Global | `<exe_dir>/scorpiox-env/` |
| User | `~/.claude/scorpiox-env/` |
| Project | `.scorpiox/scorpiox-env/` |

Each file is named `<profile-name>.txt`. For example, `~/.claude/scorpiox-env/local-llama.txt`.

### Whole-file shadowing

When the same profile name exists at multiple tiers, the **highest tier wins entirely** — the file from the lower tier is ignored. Resolution order is project → user → global (`sx_config_profile_resolve`). This is whole-file shadowing, not key-level merging between two copies of the same profile.

### Sparse overlay

A profile file only needs the keys it wants to override. Everything else inherits from the base cascade (tiers 0–3). The profile tier (4) is loaded last, so its keys sit on top of everything except OS environment variables.

### Profile name rules

- Maximum 64 characters (`SX_CONFIG_PROFILE_NAME_MAX`).
- Must not contain path separators (`/`, `\`) or traversal sequences (`..`).
- The name maps directly to the filename: profile `foo` → `scorpiox-env/foo.txt`.

## Managing and switching profiles

### Setting the active profile

There are several ways to activate a profile:

**In `scorpiox-env.txt`** — set the key at any tier:

```
ACTIVE_PROFILE=local-llama
```

**As an OS environment variable** — overrides the file-based value:

```bash
export ACTIVE_PROFILE=local-llama
```

**In-session slash command** — writes `ACTIVE_PROFILE` to the user-tier file and hot-reloads:

```
/profile local-llama
```

**In-session session-only switch** — sets the environment variable without touching any file:

```
/use local-llama
```

### Deactivating a profile

```
/profile off
/use off
```

`/profile off` clears `ACTIVE_PROFILE` from the user-tier `scorpiox-env.txt`. `/use off` only clears the in-process environment variable and does not modify files.

### Interactive profile picker

Running `/profile` with no arguments launches the `scorpiox-profile` TUI picker (if the binary is present next to `sx`). The picker lists all discovered profiles across tiers with origin badges and a preview pane showing which keys each profile overrides. Select a profile with arrow keys and Enter; press Esc to cancel.

If the `scorpiox-profile` binary is not found, `/profile` falls back to printing the list inline in the chat.

The standalone CLI can also be invoked directly:

```bash
scorpiox-profile /tmp/output.txt
```

### Inspecting effective configuration

Use `scorpiox-config` in headless mode to see the resolved cascade:

```bash
# Flat dump of all resolved key=value pairs
scorpiox-config --dump

# Grouped by source tier with file paths
scorpiox-config --dump --show-source

# Single tier only (e.g. profile)
scorpiox-config --dump --level profile --show-source

# List which cascade files exist
scorpiox-config --dump --files

# Get a single key's resolved value
scorpiox-config --get PROVIDER

# Set a key at a specific level (global, user, or project)
scorpiox-config --set PROVIDER openai --level user
```

The TUI mode (`scorpiox-config` with no flags) also shows a source column — toggle it with the `s` key inside the TUI.

## Concrete examples

### Setting up a project-specific configuration

Create `.scorpiox/scorpiox-env.txt` (or `.claude/scorpiox-env.txt`) in your repository root:

```
# .scorpiox/scorpiox-env.txt
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_MODEL=qwen3-30b-a3b
TOOLS=1
TOOL_BASH=1
TOOL_WEBSEARCH=0
THINKING_BUDGET=16000
```

This overrides your user and global settings only when SCORPIOX CODE is launched from inside this repository. Other projects are unaffected.

### Creating a reusable local model profile

```
# ~/.claude/scorpiox-env/local-llama.txt
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080
OPENAI_API_KEY=
OPENAI_MODEL=qwen3-30b-a3b
OPENAI_TIMEOUT=3600
OPENAI_CHAT_TEMPLATE_KWARGS={"thinking_mode":"enabled"}
```

Activate it:

```bash
# Persistent (survives restarts)
echo 'ACTIVE_PROFILE=local-llama' >> ~/.claude/scorpiox-env.txt

# Or inside a running session (persistent — writes to user-tier file)
/profile local-llama

# Or session-only (no file change)
/use local-llama
```

### Multiple profiles for different providers

```
# ~/.claude/scorpiox-env/anthropic-direct.txt
PROVIDER=anthropic
ANTHROPIC_AUTH_PROVIDER=custom
ANTHROPIC_API_URL=https://api.anthropic.com/v1/messages
ANTHROPIC_API_KEY=sk-ant-...
MODEL=sonnet
```

```
# ~/.claude/scorpiox-env/copilot.txt
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
COPILOT_MODEL=claude-sonnet-5
```

Switch between them live:

```
/profile anthropic-direct
/profile copilot
/profile off
```

Each switch triggers a full config reload and provider re-creation — the new provider's model, auth, and API endpoint take effect immediately.

## Gotchas

- **Profile values are overlays, not baked in.** When `scorpiox-config` saves changes, it writes to the base cascade file (global, user, or project tier). Profile files are never auto-modified by the TUI. Edit profile files by hand.

- **Profile names cannot contain path separators or `..`.** `sx_config_profile_resolve` rejects any name with `/`, `\`, or `..` to prevent path traversal.

- **Provider changes trigger in-memory provider swap.** Switching profiles calls `sx_state_reload()`, which tears down the current provider and creates a new one with the reloaded config. If the new provider fails to initialize, SCORPIOX CODE automatically reverts to the previous profile.

- **`/profile` writes to the user-tier file; `/use` does not.** `/profile local-llama` persists the choice in `~/.claude/scorpiox-env.txt` so it survives restarts. `/use local-llama` only sets the `ACTIVE_PROFILE` environment variable in the current process — it is gone when the session ends.

- **OS env vars always win.** If `ACTIVE_PROFILE` is set as a real environment variable (e.g. in `.bashrc`), file-based `ACTIVE_PROFILE=` lines are ignored. The same applies to every other config key.

- **Both `.claude/` and `.scorpiox/` are scanned at the project tier.** `.claude/scorpiox-env.txt` loads first, then `.scorpiox/scorpiox-env.txt` loads and overwrites any conflicting keys. If you use only one, there is no conflict.

- **`STARTUP_DIR` interacts with profile switching.** If a profile sets `STARTUP_DIR`, SCORPIOX CODE will `chdir` to that directory and reload the project tier after switching. A missing or invalid directory is silently skipped — it does not revert the profile.

- **Cascade files are `KEY=VALUE` plain text.** Lines starting with `#` are comments. Empty lines are ignored. Keys are case-sensitive. Values are trimmed of leading/trailing whitespace. Maximum key length is 64 characters; maximum value length is 512 characters.
