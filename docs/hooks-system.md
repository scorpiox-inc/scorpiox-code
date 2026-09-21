# Event Hooks in SCORPIOX CODE: Folder-Based Automation Architecture

SCORPIOX CODE ships with a **folder-based event hook system** that lets you drop a script into the right directory and it just runs — no config file, no JSON, no plugin registry, no build step. If a hook exists in a folder, it fires. If no folder exists, nothing happens.

The `scorpiox-hook` CLI handles everything: scaffolding, installing, testing, and listing. The runtime discovers hooks by scanning `.scorpiox/hooks/<event>/` each time an event fires, so you can add or remove hooks between sessions without restarting anything.

Source of truth: `scorpiox-hook.c` at commit `24427d8`.

---

## Why folders, not config files

Most agent harnesses bolt hooks onto the system as a configuration layer: a JSON file listing event → command pairs, a YAML plugin manifest, or a CLI flag that reads a separate script bundle. That works, but it means there are now *two* things to keep in sync — the config file that says *what* should run, and the scripts that *actually* run — and one of them will drift.

SCORPIOX CODE collapses the two into one. **The folder is the config.** Drop `01-notify.sh` into `.scorpiox/hooks/session_start/` and it runs at the start of every session. Delete it and it stops running. No registration step, no serialization format, no version mismatch between a manifest and its payload.

That design trades a small amount of expressiveness (you can't express "run this only in CI" without writing a shell conditional) for a big one: the system is trivially inspectable with `ls`, trivially diffable in git, and impossible to misconfigure in ways that produce silent no-ops.

---

## The layout

```
.scorpiox/
├── hooks/
│   ├── session_start/
│   │   ├── 01-notify.sh          (async — default)
│   │   └── sync-01-validate.sh   (sync — blocking, exit code checked)
│   ├── session_end/
│   ├── agent_complete/
│   ├── tool_use/
│   └── ... one folder per event
├── sessions/
│   └── <session_id>/
│       └── hooks.log             (per-session log, if session-scoped)
└── hooks/
    └── logs/
        └── hook.log              (fallback log when no session)
```

One folder per event. Scripts inside a folder are sorted by filename and run in order. That's the entire mental model.

---

## The 15 lifecycle events

| Event | When it fires |
|-------|---------------|
| `session_start` | A new session begins. |
| `session_end` | A session ends gracefully. |
| `session_clear` | The user runs `/clear`. |
| `user_message` | The user sends a message. |
| `agent_run_start` | The agent begins processing a message. |
| `agent_complete` | The agent finishes a run. |
| `agent_idle` | The agent has fully stopped and is waiting for user input. |
| `subagent_complete` | A sub-agent finishes a run. |
| `agent_cancelled` | The user cancels the agent. |
| `tool_use` | A tool is invoked. |
| `mcp_call` | An MCP server tool is called. |
| `api_response` | An API response is received. |
| `api_error` | An API error occurred (rate limits, timeouts, etc.). |
| `compact` | The conversation was compacted. |
| `hook_failed` | A sync hook exited non-zero and aborted the pipeline. |

Unknown event names are silently accepted. You can create a folder for any arbitrary event name and emit it with `scorpiox-hook emit my_custom_event --data '...'` — the system will run any scripts it finds there.

---

## Sync vs async hooks

Every script in an event folder is either **sync** or **async**, decided by its filename:

| Prefix / rule | Meaning |
|---------------|---------|
| `sync-<name>.sh` | **Sync** — runs to completion before the agent proceeds. Exit code is checked. A non-zero exit **aborts the pipeline and skips all async hooks for that event.** |
| `<name>.sh` (no prefix) | **Async** — forked in parallel, fire-and-forget. Output is redirected to the session log. Exit code is not checked. |

So within a single event:

1. All sync hooks run first, sequentially, in filename order.
2. If any sync hook fails, the async phase is skipped entirely.
3. Otherwise, all async hooks run in parallel.

This gives you a clean pattern: use sync hooks for validation or gating (block the agent if a pre-condition fails), and async hooks for notifications, logging, metrics, and any work that should not delay the main flow.

---

## Disabling a hook without deleting it

Prefix the filename with an underscore to disable it. The scanner skips any file starting with `_` or `.`:

```
.scorpiox/hooks/session_start/_01-notify.sh    # disabled — will not run
.scorpiox/hooks/session_start/.old-notify.sh   # disabled (dotfile)
.scorpiox/hooks/session_start/01-notify.sh     # active
```

Rename to toggle. No config change, no restart.

---

## What your hook receives

Every hook is invoked with four positional arguments:

| Arg | Value | Example |
|-----|-------|---------|
| `$1` | Event name | `session_start` |
| `$2` | Session ID (or `none`) | `2026_02_10_cool_newton` |
| `$3` | ISO 8601 timestamp | `2026-02-10T20:12:59Z` |
| `$4` | JSON payload | `{"model":"opus","provider":"claude_code"}` |

In addition, four environment variables are set in the hook's process:

| Variable | Value |
|----------|-------|
| `SX_CWD` | The working directory where the session is running. |
| `SX_HOOKS_DIR` | The path to `.scorpiox/hooks`. |
| `SX_EVENT` | The event name (convenience, same as `$1`). |
| `SX_SESSION_ID` | The session ID (convenience, same as `$2`). |

A minimal bash hook:

```bash
#!/bin/sh
# .scorpiox/hooks/session_start/01-log.sh
echo "[$3] session $2 started (event: $1)"
echo "data: $4"
```

---

## Supported script types

The runner detects the interpreter from the file extension:

| Extension | Invoked as |
|-----------|-----------|
| `.sh` | `/bin/sh <script> <args>` — CRLF line endings are auto-fixed on every run. |
| `.ps1` | `pwsh -NoProfile -File <script> <args>` |
| `.py` | `python3 <script> <args>` |
| `.bat` / `.cmd` | `cmd.exe /c <script> <args>` (Windows only) |
| anything else | Executed directly. On Unix, the file must have a shebang and the execute bit set. On Windows, the file must have a recognized extension. |

There is no per-hook interpreter config. The extension is the config.

---

## Per-session logging

Every hook execution is logged. The log path is resolved dynamically:

- **With a session ID** — logs go to `.scorpiox/sessions/<id>/hooks.log`. This keeps hook output co-located with the rest of that session's artifacts.
- **Without a session** (e.g. you ran `scorpiox-hook emit` by hand) — logs go to `.scorpiox/hooks/logs/hook.log`.

Each execution block records the event name, hook name, sync/async mode, all stdout/stderr captured from the hook, the exit code, and the wall-clock duration. Sync hooks are fully captured (the runner reads their output before proceeding). Async hooks have their output redirected straight to the log file in the grandchild process.

---

## The `scorpiox-hook` CLI

| Command | What it does |
|---------|-------------|
| `scorpiox-hook init` | Creates the `.scorpiox/hooks/` tree with one directory per known event. |
| `scorpiox-hook events` | Prints the built-in event list with descriptions. |
| `scorpiox-hook list [--event <name>]` | Lists installed hooks, their sync/async status, and per-event counts. |
| `scorpiox-hook install <event> <file> [--sync]` | Copies a script into the event folder. `--sync` adds the `sync-` prefix automatically. |
| `scorpiox-hook test <event> [name] [--data '{json}']` | Runs hooks in the event folder with sample data (or your own) without firing a real session. |
| `scorpiox-hook emit <event> [--data '{json}'] [--session <id>]` | Fires an event right now. Useful for wiring custom events from outside the agent, or for testing. |

### Install and test workflow

```bash
# Scaffold the folder structure
scorpiox-hook init

# Drop a hook in place
scorpiox-hook install session_start ./notify.sh

# Or make it sync (blocks the agent, exit code checked)
scorpiox-hook install session_start ./validate.sh --sync

# Dry-run without starting a session
scorpiox-hook test session_start

# Or with custom data
scorpiox-hook test session_start --data '{"model":"opus","log_level":"DEBUG"}'

# List what's installed
scorpiox-hook list
```

---

## How this compares to other agent harnesses

### Claude Code

Claude Code's hook system is configuration-driven. You write a `hooks.json` (or equivalent) inside `.claude/` that maps event names to command handlers, and Claude Code reads that config to decide what to run and with what input. The events cover a similar lifecycle surface — session start/end, tool pre/post, prompt submit, stop, subagent start/stop — plus some more fine-grained ones like `PermissionRequest` and `PostToolUseFailure`.

The difference is structural. In Claude Code you maintain a JSON mapping *and* the scripts it points at; in SCORPIOX CODE the scripts themselves are the registration. A Claude Code hook config can get out of sync with the scripts it references (stale entry, wrong path, schema drift); a SCORPIOX CODE hook folder can't, because there is no separate registry to drift from. Claude Code also supports HTTP endpoints, MCP tool calls, and LLM prompts as hook handlers; SCORPIOX CODE hooks are local scripts only.

### Codex

Codex (OpenAI's CLI) has no first-class hook or lifecycle-event mechanism in its current public configuration. Automation is typically achieved by wrapping the `codex` CLI in a shell script or CI step that inspects output files or exit codes after each run. There is no event bus, no per-tool-call interception, and no notion of a hook firing *during* a turn.

SCORPIOX CODE's model is fundamentally more integrated: events fire from inside the agent loop, at well-defined points, with structured JSON context. You don't have to parse CLI output to find out that a tool call happened — a `tool_use` hook fires with the tool name and ID in `$4`.

### Pi

Pi (by badlogic) takes a minimalist approach: it exposes extension points through a small TypeScript API surface rather than a filesystem convention. You write a module that registers listeners against the Pi event emitter, and Pi loads those modules at startup. There is no "drop a file in a folder and it runs" guarantee — the extension must be registered in Pi's config.

The trade-off: Pi's API gives you typed access to the full session state, whereas SCORPIOX CODE hooks are decoupled subprocesses that receive a fixed set of arguments and environment variables. For most automation (notify on completion, log to a file, update a dashboard), the subprocess model is simpler and doesn't require a build step or a language runtime that Pi's API would demand.

### Hermes

Hermes does not ship a documented, stable hook system in its current release. Automation is handled through the agent's own tool calls or through external orchestrators that shell out to Hermes. There is no equivalent of a per-event hook folder.

The practical effect: if you need "run X when the agent finishes a turn," in Hermes you either prompt the agent to run X as its last action, or you wrap the Hermes invocation in a script that fires X on exit. Neither gives you mid-run events like `tool_use` or `api_error` without instrumenting the agent loop yourself. SCORPIOX CODE's hook system gives you those events out of the box.

---

## When to use hooks vs other mechanisms

| Need | Mechanism |
|------|-----------|
| Notify (Slack, email, terminal bell) on completion | Async hook on `agent_complete` |
| Block the agent until a pre-condition is met | Sync hook on `session_start` or `user_message` |
| Log every tool call to a central store | Async hook on `tool_use` |
| React to rate-limit errors | Async hook on `api_error` |
| Run a custom script outside the agent | `scorpiox-hook emit my_event --data '...'` |
| Gate a sub-agent result before the parent continues | Sync hook on `subagent_complete` |

Hooks are the right tool when the trigger is a **lifecycle event** and the action is **local and fast**. For anything that requires network round-trips, long-running work, or cross-session state, use the [scheduled callback system](/24427d8/en/callbacks) instead — callbacks are designed for asynchronous, agent-driven loops, while hooks are designed for synchronous, event-driven side effects.
