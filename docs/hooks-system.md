# Event Hooks in SCORPIOX CODE: Folder-Based Automation Architecture

You want something to happen every time a session starts, or every time the agent finishes a run, or every time an API error fires — but you do not want to write a config file, define a JSON schema, register a matcher, or review a trust hash before anything runs.

**Folder-based event hooks** in SCORPIOX CODE are that mechanism: you drop a script file into the right folder and it runs. No config file, no YAML, no JSON schema, no matcher expressions, no review-and-trust ceremony. The folder is the configuration. The filename is the routing rule. That is the whole system.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** put a script in `.scorpiox/hooks/<event>/` and it runs automatically when that event fires — sync if the name starts with `sync-`, async (fire-and-forget) by default, disabled if the name starts with `_` or `.`.

---

## When you get a hook

A hook fires when a lifecycle event occurs. There is no registration step, no subscription, no config block to maintain. The moment a script appears in the correct event folder, the next time that event fires, the script runs.

```
.scorpiox/hooks/
  session_start/
    sync-01-validate.sh      <- sync: blocks, exit code checked
    02-notify.sh             <- async: fire-and-forget (default)
    _03-legacy.sh            <- disabled: underscore prefix, skipped
  session_end/
    01-cleanup.sh
  agent_complete/
    01-report.py
  api_error/
    01-log-error.sh
  ...
```

---

## The events

Every event that fires in a SCORPIOX CODE session has a corresponding folder. When the event fires, all enabled scripts in that folder are discovered and executed.

| Event | Fires when | Sample data |
|---|---|---|
| `session_start` | New session begins | `{"model":"opus","provider":"claude_code","log_level":"DEBUG"}` |
| `session_end` | Session ends gracefully | `{}` |
| `session_clear` | User runs `/clear` | `{}` |
| `user_message` | User sends a message | `{"len":42}` |
| `agent_run_start` | Agent begins processing | `{"msg_len":42}` |
| `agent_complete` | Agent finishes a run | `{"turns":3,"history":8}` |
| `agent_idle` | Agent fully stopped, waiting for input | `{}` |
| `subagent_complete` | Sub-agent finishes a run | `{"turns":3,"history":8}` |
| `agent_cancelled` | User cancels the agent | `{"reason":"user","turns":2}` |
| `tool_use` | Tool invocation | `{"tool":"Bash","id":"toolu_01abc"}` |
| `mcp_call` | MCP server tool call | `{"server":"filesystem","tool":"read_file"}` |
| `api_response` | API response received | `{"turn":1,"content_count":2,"stop_reason":"end_turn","in_tokens":3721,"out_tokens":331}` |
| `api_error` | API error occurred | `{"error":"rate_limited","status":429}` |
| `api_retry_wait` | Transient API error, agent waiting to resend | `{"http_code":429,"attempt":1,"max_attempts":10,"delay_ms":30000,"reason":"HTTP 429","turn":36}` |
| `api_retry_resume` | Agent resending after retry wait | `{"attempt":1,"turn":36}` |
| `api_retry_exhausted` | Agent retries used up, loop stops | `{"http_code":429,"attempts":10,"reason":"HTTP 429","turn":36}` |
| `compact` | Conversation compacted | `{"from":50,"to":12}` |
| `hook_failed` | A sync hook exited non-zero | `{"exit_code":1,"hook":"session_start"}` |

The event list is not closed. `scorpiox-hook emit <any-name>` accepts arbitrary event names, so you can add your own events without touching the source.

---

## Sync vs async: the naming rule

The execution model is determined by the filename, not by a config key.

| Filename prefix | Mode | Behaviour |
|---|---|---|
| `sync-` (e.g. `sync-01-validate.sh`) | **Sync** | Blocks the caller. Exit code is checked. If any sync hook exits non-zero, all remaining sync hooks still run (they are not short-circuited), but the async phase is skipped entirely. |
| anything else (e.g. `02-notify.sh`) | **Async** (default) | Fire-and-forget. The script runs in a detached background process. Exit code is recorded in the log but does not affect the caller. |
| `_` prefix (e.g. `_03-legacy.sh`) | **Disabled** | Skipped. The script is not executed. Rename to remove the prefix to re-enable. |
| `.` prefix (e.g. `.hidden.sh`) | **Disabled** | Same as underscore. Useful for temporarily stashing a hook. |

Numbering at the front (`01-`, `02-`, etc.) controls sort order. Sync hooks run in sorted order, sequentially, one after another. Async hooks are sorted, then all forked in parallel — they do not wait for each other.

**Why this design matters:** sync hooks are for work that must complete before the agent proceeds — validation, precondition checks, environment setup. The agentless mode (headless, CI, scripted runs) relies on this: the `session_start` sync hook gates the entire session. If it exits non-zero, the session does not begin and the failure is logged as a `hook_failed` event.

---

## What your script receives

Every hook script receives four positional arguments:

| Position | Value | Example |
|---|---|---|
| `$1` | Event name | `session_start` |
| `$2` | Session ID | `2026_02_10_cool_newton` or `none` |
| `$3` | ISO 8601 UTC timestamp | `2026-02-10T20:12:59Z` |
| `$4` | JSON data (or `{}`) | `'{"model":"opus"}'` |

In addition, four environment variables are set for every hook invocation:

| Variable | Value |
|---|---|
| `SX_CWD` | Current working directory |
| `SX_HOOKS_DIR` | Path to `.scorpiox/hooks` |
| `SX_EVENT` | Event name (convenience, same as `$1`) |
| `SX_SESSION_ID` | Session ID (convenience, same as `$2`) |

A minimal sync hook that checks an environment precondition:

```sh
#!/bin/sh
# sync-01-check-env.sh
# Blocks the session if a required env var is missing.

if [ -z "$DATABASE_URL" ]; then
  echo "DATABASE_URL is not set. Session will not start." >&2
  exit 1
fi
exit 0
```

An async hook that sends a notification on agent completion:

```sh
#!/bin/sh
# 01-report.sh
# Fires after agent_complete. Runs in background; exit code does not matter.
echo "Agent finished: $1 session=$2 at $3" >> /tmp/agent-complete.log
```

---

## Supported script types

The interpreter is chosen by file extension. You do not need a shebang for the common cases.

| Extension | Interpreter | Notes |
|---|---|---|
| `.sh` | `/bin/sh` | CRLF line endings are auto-fixed in-place before execution. No `+x` permission needed. |
| `.ps1` | `pwsh -NoProfile -File` | PowerShell. |
| `.py` | `python3` | Python 3. |
| `.bat` / `.cmd` | `cmd.exe /c` | Windows only. |
| other (no extension) | direct `exec` | Must have a shebang on Unix. `+x` permission required. |

The CRLF auto-fix means a `.sh` file edited on Windows and copied to a Unix host runs without the `bad interpreter` error. The fix is logged to stderr.

---

## Per-session logging

Every hook invocation is logged. The log file is per-session when a session ID is present, so you can audit exactly which hooks ran for which session and what they output.

- **Per-session log:** `.scorpiox/sessions/<session-id>/hooks.log`
- **Fallback (no session):** `.scorpiox/hooks/logs/hook.log`

The log records, for each hook: the timestamp, event name, script name, sync/async mode, the full stdout and stderr output, the exit code, and the wall-clock duration in seconds.

```
═══ [2026-02-10 20:12:59] session_start/sync-01-validate.sh (sync) ═══
DATABASE_URL OK
── EXIT=0 (0.02s) ──
```

---

## The `scorpiox-hook` CLI

The hook runtime is a standalone binary (pure C + POSIX, no library dependencies) that ships with SCORPIOX CODE. It is called `scorpiox-hook` and sits next to the main binary.

| Command | Purpose |
|---|---|
| `scorpiox-hook init` | Scaffold all event directories under `.scorpiox/hooks/` |
| `scorpiox-hook events` | List all known events with descriptions |
| `scorpiox-hook list` | Show all installed hooks, their sync/async status, and disabled count |
| `scorpiox-hook list --event <name>` | Filter to one event |
| `scorpiox-hook test <event> [hook_name]` | Run hooks for an event with sample data (or a specific hook only). Does not require a live session. |
| `scorpiox-hook install <event> <script_path> [--sync]` | Copy a script into the event folder. `--sync` adds the `sync-` prefix. |
| `scorpiox-hook emit <event> [--data '{json}'] [--session <id>]` | Fire an event manually. Useful for testing or from external scripts. |

`scorpiox-hook test` is the fastest way to verify a hook before committing it. It uses the built-in sample data for known events, so you do not need to construct the JSON by hand.

```
$ scorpiox-hook init
$ scorpiox-hook install session_start ./check-env.sh --sync
$ scorpiox-hook test session_start
Testing session_start hooks with data: {"model":"opus","provider":"claude_code","log_level":"DEBUG"}

  sync-01-check-env.sh  [SYNC]  OK
```

---

## Kill switch and timeout

Two config keys control hook behaviour. They live in `scorpiox-env.txt`:

| Key | Default | Purpose |
|---|---|---|
| `HOOKS_ENABLED` | `1` | Set to `0` to disable all hook execution. The kill switch. |
| `HOOKS_TIMEOUT` | `30` | Timeout in seconds for sync hooks. Async hooks have no timeout (they are fire-and-forget). |

---

## Comparison: how SCORPIOX CODE hooks differ

### Claude Code

Claude Code hooks are defined in `settings.json` files (`~/.claude/settings.json` for user scope, `.claude/settings.json` for project scope, plus managed policy settings and plugin hooks). Each hook requires:

- An **event** key (e.g. `PreToolUse`, `SessionStart`, `Stop`).
- A **matcher** expression that filters which tool calls or contexts trigger the hook.
- A **handler** of one of five types: `command`, `http`, `mcp_tool`, `prompt`, or `agent`.

The matcher system is powerful but adds a second layer of configuration between the event and the script. Handler types beyond shell commands (HTTP endpoints, MCP tool calls, LLM prompt hooks, subagent hooks) each have their own schema and timeout rules. Async hooks are a separate opt-in per handler. There is no per-session log file; you debug with the `/hooks` menu or by inspecting the transcript.

**What SCORPIOX CODE does differently:** no config file, no matcher, no handler-type taxonomy. A script in the folder runs. The event name is the routing rule. The filename prefix is the sync/async decision. Per-session log files are built in.

### Codex

Codex hooks are defined in `hooks.json` files at four scope levels (system, user, project, plugin). Each hook entry requires a `command` or MCP tool reference. Before any non-managed hook runs for the first time, Codex requires you to **review and trust** it: the hook's hash is recorded, and any change to the script re-triggers the review. Use `/hooks` in the CLI to inspect, trust, or disable. `allow_managed_hooks_only` in `requirements.toml` lets administrators restrict which hooks can run.

**What SCORPIOX CODE does differently:** no trust-hash ceremony. A script that appears in the folder runs on the next event. There is no review step, no hash, no `/hooks` menu. The folder is the trust boundary: if the script is there, it runs.

### Pi

Pi (badrapup/pi-mono) is a minimal TypeScript-based agent harness. Its extension model is module-based: you write a TypeScript extension module that implements a lifecycle hook interface, and the harness calls it. Extensions are npm packages or local modules loaded at session start. There is no folder-based script system; the unit of extension is a compiled module, not a script file.

**What SCORPIOX CODE does differently:** no build step, no package manager, no module system. A shell script is a hook. A Python script is a hook. A PowerShell script is a hook. The script language is determined by the file extension, and the interpreter is auto-selected.

### Hermes

Hermes (the harness referenced in SCORPIOX CODE's file-editing documentation) uses fuzzy search/replace blocks for file editing and has a different extension model. Its hook mechanism is not a folder-based script system; hooks are registered through the harness's own configuration.

**What SCORPIOX CODE does differently:** the folder is the configuration. You do not register hooks in a config file. You place scripts in folders. The system discovers them at event time.

---

## The design principle

The consistent thread across all of these comparisons is the same: **SCORPIOX CODE's hook system has no configuration surface.** There is no file to edit, no schema to validate, no trust hash to review, no matcher to write, no handler type to choose. The folder path is the event. The filename prefix is the mode. The script is the behaviour.

That is deliberate. Hooks are a systems-level concern: "when this happens, do that." The unit of automation in a Unix-like system is a script in a directory. SCORPIOX CODE treats hooks the same way: scripts in directories. Everything else — the event names, the arguments, the environment variables, the logging — is provided by the harness. You write the script. The harness handles the rest.

---

## Tips

- **Use `sync-` for gates, plain names for side effects.** If a hook's job is to prevent something from proceeding (validation, precondition check, environment setup), make it sync. If its job is to report, log, notify, or trigger a side process, make it async. The exit code of a sync hook is meaningful; the exit code of an async hook is informational.
- **Test before you commit.** `scorpiox-hook test <event>` runs your hook with realistic sample data and shows you the output and exit code without starting a session.
- **Check the per-session log.** When a hook misbehaves, the per-session log at `.scorpiox/sessions/<id>/hooks.log` has the full stdout/stderr, the exit code, and the duration.
- **Disable without deleting.** Rename with a `_` prefix to disable. Rename back to re-enable. Useful for temporarily stashing a hook during debugging.
- **Agentless mode.** In headless or scripted runs, the `session_start` sync hook is the gate. If it exits non-zero, the session does not begin. This is how CI pipelines use hooks as preconditions: validate the environment before the agent starts consuming tokens.
- **CRLF is handled.** If you edit a `.sh` file on Windows and copy it to a Unix host, the CRLF line endings are auto-fixed in-place before execution. You do not need to run `dos2unix` first.

---

*Related: [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md), [Isolated Git Worktrees for Parallel Agent Sessions](git-worktrees.md), [Deterministic File Editing and Developer Autonomy in SCORPIOX CODE](file-editing.md)*
