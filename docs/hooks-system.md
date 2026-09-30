# Event Hooks in SCORPIOX CODE: Folder-Based Automation Architecture

You want your own scripts to fire when the agent does something — when a session starts, when a tool runs, when a run ends. Every agent harness offers some version of this. Some make you write a config file with a JSON schema, some make you register a callback in code, some make you wire up an external service.

SCORPIOX CODE does none of that. You drop a script into a folder, and it runs. That is the entire system.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** hooks are scripts you place in `.scorpiox/hooks/<event>/`. No config file, no registration step, no schema — the folder *is* the registration, and the filename decides whether it blocks.

---

## The layout

Everything lives under a single directory, `.scorpiox/hooks/`, with one subfolder per event:

```
.scorpiox/
└── hooks/
    ├── session_start/
    │   ├── 01-notify.sh
    │   └── sync-01-validate.sh
    ├── tool_use/
    │   └── log-tool.sh
    ├── agent_complete/
    │   └── 02-report.py
    └── ...one folder per event...
```

A hook is just a file inside the event folder that matches one of the supported types:

| Extension | How it runs |
|-----------|-------------|
| `.sh` | `/bin/sh <script>` (CRLF line endings are fixed automatically) |
| `.ps1` | `pwsh -NoProfile -File <script>` |
| `.py` | `python3 <script>` |
| `.bat` | `cmd.exe /c <script>` (Windows only) |
| other | executed directly (must have a shebang on Unix and be executable) |

No extension, no shebang, not executable, nothing to do. That is the whole discovery rule.

---

## The events

SCORPIOX CODE fires 18 lifecycle events through this system. The full list is what `scorpiox-hook events` prints:

| Event | Fires when |
|-------|------------|
| `session_start` | A new session begins |
| `session_end` | A session ends gracefully |
| `session_clear` | The user runs `/clear` |
| `user_message` | The user sends a message |
| `agent_run_start` | The agent begins processing |
| `agent_complete` | The agent finishes a run |
| `agent_idle` | The agent has fully stopped and is waiting at the prompt |
| `subagent_complete` | A sub-agent finishes a run |
| `agent_cancelled` | The user cancels the agent |
| `tool_use` | A tool is invoked |
| `mcp_call` | An MCP server tool is called |
| `api_response` | An API response is received |
| `api_error` | An API error occurs |
| `api_retry_wait` | A transient API error — the agent is waiting to resend |
| `api_retry_resume` | The agent is resending after the retry wait |
| `api_retry_exhausted` | The agent has used all its retries and the loop stops |
| `compact` | The conversation has been compacted |
| `hook_failed` | An agentless hook exited non-zero |

The event set is open, not closed. You can create a folder for any name you like — `emit` will silently no-op if there are no hooks in it, and it will run whatever is there if there are. The 18 above are the ones the agent itself emits; the rest of the surface is yours to define.

---

## Sync vs. async: the filename does the work

Every hook is either **sync** or **async**, and the decision is made entirely by the filename:

| Filename pattern | Mode | Behavior |
|------------------|------|----------|
| `01-notify.sh` | async (default) | Fire-and-forget; forked in parallel with the other async hooks |
| `sync-01-validate.sh` | sync | Runs in order, blocks until it exits, exit code is checked |
| `_disabled.sh` | skipped | Any name starting with `_` is not executed |
| `.hidden.sh` | skipped | Any name starting with `.` is not executed |

The execution contract per event is fixed:

1. **All sync hooks run first**, sorted by filename, sequentially, blocking.
2. **If any sync hook exits non-zero, the run aborts.** Async hooks for that event are skipped, and the failure is reported.
3. **All async hooks run after**, sorted by filename, forked in parallel.

This gives you two clean guarantees without any configuration:

- **Ordering.** Name your sync hooks `01-`, `02-`, `03-`, and they run in that order. There is no config field for priority; the filename is the priority.
- **Gating.** A failing sync hook cancels the async fan-out. This is how you build a guardrail: `sync-00-block-bad-input.sh` can inspect the payload and, if it decides to bail, prevent the notification hooks from firing.

Because async hooks run in parallel and their exit codes are not checked, use them for observability and side effects — logging, alerts, telemetry, notification. Use sync hooks for anything that should be able to stop the flow.

---

## What a hook receives

Every hook gets the same four positional arguments, in the same order, no matter which event it is:

```sh
$1 = event name       # "session_start"
$2 = session ID       # "2026_02_10_cool_newton" or "none"
$3 = ISO timestamp    # "2026-02-10T20:12:59Z"
$4 = JSON data        # '{"model":"opus"}' or '{}'
```

On top of that, four environment variables are set for the duration of the hook:

| Variable | Value |
|----------|-------|
| `SX_CWD` | Working directory the hook is running in |
| `SX_HOOKS_DIR` | `.scorpiox/hooks` |
| `SX_EVENT` | The event name (convenience mirror of `$1`) |
| `SX_SESSION_ID` | The session ID (convenience mirror of `$2`) |

The JSON payload in `$4` is the event-specific context — for `api_response` it includes token counts and stop reason, for `tool_use` it includes the tool name and ID, for `compact` it includes the before/after sizes. Each event carries a sample payload you can see with `scorpiox-hook events`.

A minimal shell hook looks like this:

```sh
#!/bin/sh
# .scorpiox/hooks/agent_complete/01-report.sh
echo "[hook] run finished: turns=$(echo "$4" | jq .turns) session=$2" >> ~/agent-runs.log
```

A sync hook that gates on a condition:

```sh
#!/bin/sh
# .scorpiox/hooks/session_start/sync-01-check-quota.sh
USAGE=$(jq -r .turns <<< "$4")
if [ -n "$USAGE" ] && [ "$USAGE" -gt 1000 ]; then
    echo "Quota exceeded" >&2
    exit 1   # blocks async hooks for this event
fi
exit 0
```

---

## The `scorpiox-hook` CLI

The CLI is a standalone binary that ships next to the main executable. It has six subcommands:

| Command | What it does |
|---------|--------------|
| `scorpiox-hook init` | Scaffolds all 18 event folders under `.scorpiox/hooks/` |
| `scorpiox-hook events` | Lists every event with a one-line description |
| `scorpiox-hook list [--event <name>]` | Lists installed hooks, with `[SYNC]` / `[ASYNC]` tags and a disabled count |
| `scorpiox-hook install <event> <script> [--sync]` | Copies a script into the event folder (adding `sync-` if requested), fixes CRLF, chmods `+x` |
| `scorpiox-hook test <event> [hook_name] [--data '{json}']` | Runs a hook with sample data in the foreground so you can see its output |
| `scorpiox-hook emit <event> [--data '{json}'] [--session <id>]` | Fires an event directly from the shell (same path the agent uses) |

A typical first-time setup is:

```sh
scorpiox-hook init                       # create the 18 folders
scorpiox-hook install agent_complete ~/hooks/report.sh
scorpiox-hook list                       # see what you have
scorpiox-hook test agent_complete        # try it with sample data
```

`test` is the debugging workhorse — it runs the hook synchronously in the foreground with the canonical sample payload for that event (or whatever you pass with `--data`), so you can watch the output without waiting for a real run to happen.

---

## Where the output goes

Every hook execution is logged. The log path depends on whether the hook fired inside a real session:

- **Inside a session:** `.scorpiox/sessions/<id>/hooks.log`
- **Outside a session** (e.g. from `scorpiox-hook emit` with no `--session`): `.scorpiox/hooks/logs/hook.log`

Each entry records the start time, the event, the hook name, whether it was sync or async, the captured stdout and stderr (for sync hooks; async hooks append directly to the log file), and the exit code with duration. This is the only place you need to look when a hook misbehaves — there is no separate tracing system.

---

## Disabling hooks

Three levels of control, all by convention:

1. **Per hook:** rename the file to start with `_` or `.`. The scanner skips it.
2. **Per event:** remove the event folder, or empty it. `emit` silently no-ops when a folder has no hooks.
3. **Global kill switch:** set `HOOKS_ENABLED` to `0`. No hooks fire at all, regardless of event. This is the escape hatch for production environments where you want the agent to run but the side-channel scripts to stay quiet.

There is no per-event config file, no allow-list, no policy layer. If you want a hook to stop running, you stop it by not having it there.

---

## How this compares to other agent harnesses

SCORPIOX CODE's design sits deliberately on one side of a line that most harnesses cross over. The line is: *do you configure a hook, or do you place a hook?*

### Claude Code

Claude Code hooks are defined in **JSON settings files** (`~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed policy files, plugin manifests). A hook is a nested structure: pick an event, add a matcher group that narrows which tool calls trigger it, and define one or more handlers (shell command, HTTP endpoint, MCP tool call, prompt, or agent). Handlers receive the event context as JSON on stdin and return a decision via exit code (`0` = no decision, `2` = block) or a structured JSON body.

This is a powerful system — the matcher + handler model gives fine-grained control, and the JSON-out contract lets hooks feed context back to the model. The cost is that you are writing a configuration document in a schema, not a script. The hook is not the thing that runs; the hook is the declaration of the thing that runs.

SCORPIOX CODE has no matcher layer. If a script is in the folder, it runs for that event. If you want to condition on the tool name, the tool name is in `$4` and you branch in your script. That is a smaller surface, but it also means the "hook" and the "code" are the same artifact, which is the whole point of the folder model.

### Codex

Codex's hook surface is smaller and more purpose-specific. The primary mechanism is a single `notify` command in `config.toml` — a shell command that Codex invokes with a JSON payload for notifications. There is also a lifecycle-hooks layer loaded from `hooks.json` or an inline `[hooks]` block, gated behind a feature flag, primarily oriented at managed/enterprise deployments (with an `allow_managed_hooks_only` policy in `requirements.toml`).

The `notify` path is close in spirit to SCORPIOX CODE's async hooks — a single command, JSON in, fire-and-forget — but it is one hook for one purpose, not a folder per event. The lifecycle layer is closer to an admin policy mechanism than a user-facing automation surface.

### Hermes

Hermes (Nous Research) runs four separate hook systems in parallel: **gateway hooks** (a `HOOK.yaml` + `handler.py` pair per hook, loaded from `~/.hermes/hooks/<name>/` at gateway startup), **plugin hooks** (registered in code via `ctx.register_hook("pre_tool_call", ...)`), **shell hooks** (declared in a `hooks:` block in `profile config.yaml`, pointing at shell scripts), and **outbound webhooks** (a `hooks.outbound:` list that pushes signed lifecycle events to external HTTP endpoints).

Gateway hooks are the closest cousin to SCORPIOX CODE's folder model — a directory per hook, a manifest file declaring which events it listens for, a handler that runs on match. The difference is that Hermes requires a manifest (`HOOK.yaml`) *and* a handler per hook, and the handler is a Python module imported in-process. SCORPIOX CODE collapses both into the file itself: the filename is the event subscription, the file is the handler, and the extension picks the interpreter. No manifest, no import, no registration.

### Pi

Pi (earendil-works) has no standalone hook system in the same sense. Its extension mechanism is **TypeScript modules** that call `pi.on("<event>", handler)` to subscribe to lifecycle events (`before_agent_start`, `tool_call`, `tool_result`, `message_end`, `turn_end`, `agent_end`, and others). Handlers run in-process in the Pi runtime, in registration order, and can transform data, replace results, or cancel operations depending on the event's declared result type.

This is the most powerful model of the four — handlers are real code with typed inputs and outputs, running in the same process as the agent — but it is also the most coupled. You are writing a module that Pi loads, not a script that sits on disk. There is no "place a file and it works" path; there is a loader, a runtime, and a type surface.

### The trade-off in one table

| Dimension | SCORPIOX CODE | Claude Code | Codex | Hermes | Pi |
|-----------|---------------|-------------|-------|--------|-----|
| Where hooks live | `.scorpiox/hooks/<event>/` folders | JSON settings files | `config.toml` + `hooks.json` | Four separate systems | TypeScript extension modules |
| Registration | Drop a file in the folder | Declare in JSON | Declare in config | Manifest + handler / in code | `pi.on()` in a module |
| Event selection | The folder name | Event + matcher | Fixed to `notify` / lifecycle events | `events:` list in manifest | `pi.on("<event>")` call |
| Payload | `$1`–`$4` args + env vars | JSON on stdin | JSON arg to `notify` | Keyword args / JSON | Typed event object |
| Sync vs. async | Filename prefix (`sync-`) | Per-handler config | N/A (notify is async) | Per-hook config / timeout | In-process, awaited |
| Blocking behavior | Non-zero exit aborts async phase | Exit code `2` blocks | N/A | `pre_tool_call` can block | Return value cancels |
| Script types | `.sh`, `.ps1`, `.py`, `.bat`, exec | Shell / HTTP / MCP / prompt / agent | Shell command | Shell / Python / HTTP | TypeScript only |
| Config file required | No | Yes | Yes | Yes (at least one) | Yes (module must be loaded) |

The common shape is clear: the other four harnesses all require you to describe the hook in some declarative form before it runs. SCORPIOX CODE's position is that the description is redundant — the file on disk *is* the description, and the filesystem is the registry. You trade a layer of expressive configuration for a guarantee you can verify with `ls`: if it is not in the folder, it does not run.

---

## When to use hooks vs. other mechanisms

Hooks are the right tool when you want **deterministic side effects on lifecycle events** — a log line on session start, a notification when a run completes, a guard that blocks a session from starting under certain conditions. They are not the right tool for:

- **Recurring work** (use scheduled callbacks instead — see the callbacks page).
- **Model-visible context injection** (hooks cannot feed content back to the model; for that, use the filesystem session architecture and let the agent read what it needs).
- **Interactive permission prompts** (the permission system handles that; a hook can observe a permission request, but it is not the gate).

A good rule of thumb: if you want to *react* to something the agent does, use a hook. If you want to *steer* what the agent does, use a different mechanism — hooks are observers with a narrow blocking capability, not controllers.

---

## Gotchas

- **The sync prefix is `sync-`, not `sync_` or `sync.`.** The scanner checks for the literal prefix `sync-`; anything else is treated as async.
- **A failing sync hook cancels all async hooks for that event.** It does not cancel other events, and it does not stop the agent. It only stops the rest of the fan-out for the current event.
- **Async hooks are fire-and-forget.** Their exit codes are logged but not checked, and they run in parallel. If two async hooks write to the same file, you get interleaved output.
- **CRLF is fixed automatically for `.sh` files**, but only on read. If you edit a `.sh` hook on Windows and commit it with CRLF line endings, the first run will rewrite it to LF. Subsequent runs see the LF version.
- **`emit` is silent for unknown events.** If there is no `.scorpiox/hooks/<event>/` folder, `emit` returns 0 and does nothing. This is intentional — it means you can add hooks for any event name without coordinating with the agent.
- **The `test` subcommand always runs hooks synchronously**, even if they are named as async. This is so you can see their output. It does not change how the hook runs in production.
- **Hooks run as the user running SCORPIOX CODE.** There is no sandbox, no permission boundary, no capability model. A hook can do anything the user can. Treat `.scorpiox/hooks/` as a trusted location — anyone who can write there can run code as you.

---

## Related

- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Data Privacy and Zero Data Collection](data-privacy.md)
