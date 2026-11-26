# Event Hooks in SCORPIOX CODE: Folder-Based Automation Architecture

Every agent harness has some answer to the same question: *how do I run my own code when something happens inside the agent?* A session starts. A tool runs. A run finishes. Some tools answer with a JSON schema in a settings file, some with a plugin API you compile against, some with a webhook endpoint you have to stand up and secure.

SCORPIOX CODE answers with a folder. You put a script in `.scorpiox/hooks/<event>/`, and when that event fires, the script runs. There is no registration step, no manifest to validate, no daemon to restart — the directory listing *is* the configuration, and the filename decides everything else: whether the hook blocks, whether it is disabled, and in what order it runs.

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** hooks are scripts you drop into `.scorpiox/hooks/<event>/`; the folder name subscribes you, the filename sets the mode, the arguments and environment carry the event payload, and every execution leaves a line in a per-session log you can read with `cat`.

---

## The layout

Everything lives under one directory tree, with one subfolder per event:

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

A hook is any file inside an event folder whose extension tells SCORPIOX CODE how to run it:

| Extension | How it runs |
|-----------|-------------|
| `.sh` | `/bin/sh <script>` on Unix, `bash <script>` on Windows (CRLF line endings are repaired automatically before the first run) |
| `.ps1` | `pwsh -NoProfile -File <script>` |
| `.py` | `python3 <script>` on Unix, `python <script>` on Windows |
| `.bat` / `.cmd` | `cmd.exe /c <script>` (Windows only) |
| anything else | executed directly, which means it needs a shebang and the executable bit on Unix |

That is the entire discovery rule. No extension, no shebang, not executable — nothing happens, and the file is simply skipped. There is no error to chase because there is nothing registered in the first place.

Under the hood the scripts are launched as a proper argument vector, not interpolated into a shell command line. That detail is worth knowing because it means an apostrophe inside the JSON payload of an event cannot break out of quoting and run something you did not write.

---

## The lifecycle events

SCORPIOX CODE emits **18 events** through this system. `scorpiox-hook events` prints exactly this list:

| Event | Fires when |
|-------|------------|
| `session_start` | A new session begins |
| `session_end` | A session ends gracefully |
| `session_clear` | The user runs `/clear` |
| `user_message` | The user sends a message |
| `agent_run_start` | The agent begins processing a run |
| `agent_complete` | The agent finishes a run |
| `agent_idle` | The agent has fully stopped and is waiting at the prompt |
| `subagent_complete` | A sub-agent finishes a run |
| `agent_cancelled` | The user cancels the agent |
| `tool_use` | A tool is invoked |
| `mcp_call` | An MCP server tool is called |
| `api_response` | An API response is received |
| `api_error` | An API error occurs |
| `api_retry_wait` | A transient API error — the agent is waiting before resending |
| `api_retry_resume` | The agent is resending after the retry wait |
| `api_retry_exhausted` | All retry attempts are used up and the loop stops |
| `compact` | The conversation has been compacted |
| `hook_failed` | An agentless hook exited non-zero |

The event set is **open, not closed**. If you create a folder for a name the agent never emits, nothing breaks: firing an event with no folder, or with an empty folder, is a silent no-op. Conversely, if you want your own event namespace — `nightly_backup/`, `release_gate/` — you can emit it yourself with `scorpiox-hook emit` and wire scripts to it exactly the same way.

Most events carry a small JSON payload in the fourth argument. The ones the agent emits look like this:

| Event | Example payload |
|-------|-----------------|
| `session_start` | `{"model":"opus","provider":"claude_code","log_level":"DEBUG"}` |
| `user_message` | `{"len":42}` |
| `agent_run_start` | `{"msg_len":42}` |
| `agent_complete` | `{"turns":3,"history":8}` |
| `agent_cancelled` | `{"turns":2}` |
| `tool_use` | `{"tool":"Bash","input_len":128}` |
| `api_response` | `{"turn":1,"content_count":2,"stop_reason":"end_turn","in_tokens":3721,"out_tokens":331}` |
| `api_error` | `{"error":"rate_limited","turn":4}` |
| `api_retry_wait` | `{"http_code":429,"attempt":1,"max_attempts":10,"delay_ms":30000,"reason":"HTTP 429 RESOURCE_EXHAUSTED","turn":36}` |

Two payloads deserve a second look. `api_response` carries the token counts for the turn, which is why it is the natural place for a usage metering script. The retry family turns the provider's transient failures into a scriptable timeline — you can log every backoff, or page someone when `api_retry_exhausted` finally lands.

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

That contract gives you two guarantees with zero configuration:

- **Ordering.** Name your sync hooks `01-`, `02-`, `03-` and they run in that order. There is no priority field anywhere; the filename is the priority.
- **Gating.** A failing sync hook cancels the async fan-out for that event. This is how you build a guardrail: `sync-00-block-bad-input.sh` inspects the payload and, if it decides to bail, the notification hooks behind it never fire.

Async hooks are fire-and-forget by design — their output is appended to the log and their exit codes are recorded, but nothing waits for them and nothing acts on the result. Use them for observability and side effects: logging, alerts, telemetry, notifications. Use sync hooks whenever the next step should depend on the hook agreeing to proceed.

A practical consequence of the two-phase design: an async hook cannot veto anything, and a sync hook cannot run concurrently with anything. If you need both, write two scripts.

---

## What a hook receives

Every hook gets the same four positional arguments, in the same order, no matter which event fired:

```sh
$1 = event name       # "session_start"
$2 = session ID       # "2026_02_10_cool_newton" or "none"
$3 = ISO timestamp    # "2026-02-10T20:12:59Z"
$4 = JSON data        # '{"model":"opus"}' or '{}'
```

On top of that, four environment variables are exported for the duration of the hook:

| Variable | Value |
|----------|-------|
| `SX_CWD` | Working directory the hook runs in |
| `SX_HOOKS_DIR` | `.scorpiox/hooks` |
| `SX_EVENT` | The event name, again, for scripts that prefer reading the environment |
| `SX_SESSION_ID` | The session ID, or `none` |

The JSON payload is deliberately not parsed for you. There is no schema to satisfy and no conversion step to fail; the raw text is handed over and you reach for whatever you already have — `jq` in a shell script, `json.loads` in Python, `ConvertFrom-Json` in PowerShell.

A minimal async hook that records every finished run:

```sh
#!/bin/sh
# .scorpiox/hooks/agent_complete/01-report.sh
echo "[hook] run finished: turns=$(echo "$4" | jq .turns) session=$2" >> ~/agent-runs.log
```

A sync hook that gates on a condition:

```sh
#!/bin/sh
# .scorpiox/hooks/session_start/sync-01-check-quota.sh
TURNS=$(echo "$4" | jq -r .turns)
if [ -n "$TURNS" ] && [ "$TURNS" -gt 1000 ]; then
    echo "Quota exceeded" >&2
    exit 1   # blocks every async hook for this event
fi
exit 0
```

And a Python hook, because the interpreter is chosen by extension rather than shebang:

```python
#!/usr/bin/env python3
# .scorpiox/hooks/api_response/01-usage.py
import json, os, sys
event, session, ts, data = sys.argv[1], sys.argv[2], sys.argv[3], json.loads(sys.argv[4])
print(f"{ts} {event} in={data.get('in_tokens')} out={data.get('out_tokens')}")
```

---

## The `scorpiox-hook` CLI

The hook system ships as its own standalone binary, `scorpiox-hook`, placed next to the main executable. It has six subcommands:

| Command | What it does |
|---------|--------------|
| `scorpiox-hook init` | Scaffolds all 18 event folders under `.scorpiox/hooks/` |
| `scorpiox-hook events` | Lists every event with a one-line description |
| `scorpiox-hook list [--event <name>]` | Lists installed hooks with `[SYNC]` / `[ASYNC]` tags and a disabled count |
| `scorpiox-hook install <event> <script> [--sync]` | Copies a script into the event folder, adds the `sync-` prefix if asked, repairs CRLF, and sets the executable bit |
| `scorpiox-hook test <event> [hook_name] [--data '{json}']` | Runs hooks in the foreground with sample data so you can see their output |
| `scorpiox-hook emit <event> [--data '{json}'] [--session <id>]` | Fires an event directly from the shell, through the same path the agent uses |

A first-time setup looks like this:

```sh
scorpiox-hook init                          # create the 18 folders
scorpiox-hook install agent_complete ~/hooks/report.sh
scorpiox-hook list                          # see what you have
scorpiox-hook test agent_complete           # try it with sample data
```

`test` is the debugging workhorse. It always runs the hook synchronously in the foreground — even hooks named for async — and feeds it the canonical sample payload for that event, or whatever you pass with `--data`. You watch the output, fix the script, run it again. Nothing about `test` changes how the hook behaves in a real session.

`emit` is the other half of the workflow: it is the same entry point the agent itself calls, which means you can drive your hook scripts from cron, from CI, or from another tool, and they behave exactly as they do when the agent fires them. Its exit status is non-zero if any sync hook failed, so a pipeline can fail loudly on a rejected event.

---

## Where the output goes

Every hook execution is logged. The log location depends on whether the hook fired inside a real session:

- **Inside a session:** `.scorpiox/sessions/<id>/hooks.log`
- **Outside a session** (for example `scorpiox-hook emit` with no `--session`): `.scorpiox/hooks/logs/hook.log`

Each entry looks like this:

```
═══ [2026-10-02 14:22:31] agent_complete/01-report.sh (sync) ═══
[hook] run finished: turns=3 session=2026_10_02_quiet_fermat
── EXIT=0 (0.08s) ──
```

A start line with the timestamp, the event, the hook name, and the mode; the hook's captured stdout and stderr (sync hooks are captured through a pipe, async hooks append straight into the same file); and an exit line with the code and the duration. That file is the only place you need to look when a hook misbehaves — there is no separate tracing system and no level to switch on.

This is also why hooks fire with a session ID at all: the per-session log means a session folder on disk carries not just the transcript and the traffic capture but the complete record of what your own automation did during that session.

---

## Wiring hooks from an agent pack

Agent packs — the packaged agent definitions the fleet runner clones and launches — can declare hooks in a `.hooks` file sitting next to the pack's other dotfiles:

```
session_start:echo "session $2 started on $(hostname)" >> /var/log/agents.log
agent_complete:./notify-slack.sh "$4"
```

The format is one `event:command` pair per line, and multiple commands for the same event are combined into a single generated script in that task's `.scorpiox/hooks/<event>/` folder. One rule is applied automatically: **hooks declared for `session_start` are installed with the `sync-` prefix**, so they block before the first model call rather than racing it — the natural place for a pre-flight check that must finish before any tokens are spent.

Everything else about those hooks is identical to hand-placed ones: same arguments, same environment, same log file, same `list` and `test` subcommands.

---

## Turning hooks off

Three levels of control, all by convention or a single key:

1. **Per hook:** rename the file to start with `_` or `.`. The scanner skips it; `scorpiox-hook list` counts it as disabled.
2. **Per event:** empty or remove the event folder. Firing an event with no hooks is a silent no-op.
3. **Global kill switch:** set `HOOKS_ENABLED=0`. No hooks fire at all, in any session, in any mode. This is the escape hatch for environments where the agent should run but side-channel scripts must stay quiet.

Sync hooks also have a documented timeout setting, `HOOKS_TIMEOUT` (seconds, default 30, async hooks have none). Treat it as a courtesy bound rather than a hard guarantee: the reliable protection is to keep sync hooks fast and deterministic, and to put anything slow or flaky behind an async hook where a hang cannot hold up the agent.

---

## How this compares to other agent harnesses

SCORPIOX CODE sits deliberately on one side of a line that most harnesses cross: *do you configure a hook, or do you place a hook?* The four tools below all configure. Here is what each one costs you, in its own terms.

### Claude Code

Claude Code defines hooks in **JSON settings files** — `~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed policy settings, plugin manifests, and even skill front matter, each with a different scope. A hook is a nested structure: pick an event from roughly thirty, add a matcher (exact strings or JavaScript regular expressions) that narrows which tool calls trigger it, then declare one or more handlers of several types — shell command, HTTP endpoint, MCP tool call, an LLM-evaluated prompt, or a subagent that verifies a condition with tools.

Handlers receive a JSON object on **stdin** — `session_id`, `transcript_path`, `cwd`, `permission_mode`, plus event-specific fields — and talk back through exit codes and stdout: `0` means success (with stdout parsed as JSON if it looks like an object), `2` blocks the action, where the event supports blocking at all. Timeouts default to 600 seconds for command handlers, with per-event overrides. Async execution is opt-in per handler, and everything runs behind a workspace-trust dialog that holds hooks back until the folder is accepted.

It is a deep and genuinely powerful system — matchers, decision control, and model-visible context injection are things SCORPIOX CODE's hooks do not attempt. The cost is that the hook is a declaration in a schema, verified by reading a settings file, and the artifact that runs is buried two levels down in a JSON tree. In SCORPIOX CODE the hook and the code are the same file, and "is this hook active?" is answered with `ls`.

### Codex

Codex has two surfaces. The older one is a single `notify` command in `config.toml`: one shell command that Codex invokes with a JSON payload for notifications. That is close in spirit to an async hook — JSON in, fire-and-forget — but it is one hook for one purpose, not a folder per event.

The newer one is a lifecycle-hooks layer: a `hooks.json` file or an inline `[hooks]` table, organized as event, then matcher group (a regex over tool names or session sources), then handlers — currently command and MCP-tool handlers, gated behind a feature flag. Handlers get JSON on stdin, can return JSON that injects context or rewrites a tool call, run concurrently, and default to 600-second timeouts with an eight-at-a-time cap for background hooks. Non-managed hooks must be individually reviewed and trusted before they will run at all — trust is keyed to a hash of the hook definition, so editing a hook marks it for review again.

The trust machinery is a real security posture for organizations, and the matchers are more expressive than a folder name. But the same review flow means a hook you wrote five minutes ago is not running until you approve it, and a hook that is running is a definition inside a config layer rather than a file you can see and touch. `notify` remains the simplest path — and it is one slot.

### Hermes

Hermes (Nous Research) runs **four hook systems in parallel**, and its own documentation opens with that table: **gateway hooks** (a `HOOK.yaml` manifest plus a `handler.py` per hook directory, loaded at gateway startup), **plugin hooks** (registered in Python via `ctx.register_hook(...)`), **shell hooks** (a `hooks:` block in profile `config.yaml` pointing at scripts), and **outbound webhooks** (a list of HTTP endpoints that receive signed JSON payloads for lifecycle events).

The shell-hook system is the closest cousin to SCORPIOX CODE's: single-file scripts, any language, subprocess isolation, JSON on stdin. But it is declared in YAML, filtered by regex matchers, consent-gated through a first-use prompt per `(event, command)` pair persisted to an allowlist file, and bounded by a 60-second timeout capped at 300. Gateway hooks are trusted by directory placement but only load in the gateway process; plugin hooks run in-process and require authoring a Python module.

Hermes' design reflects its footprint — a gateway that serves chat platforms needs consent records and signed outbound pushes. If what you want is "run this script when the agent does something," you are choosing between four mechanisms and a config file in all of them. SCORPIOX CODE has one mechanism and no config file.

### Pi

Pi (earendil-works) has no hook system in the file-based sense at all. Its extension mechanism is **TypeScript modules** loaded into the agent process, which subscribe to lifecycle events with `pi.on("<event>", handler)` — `before_agent_start`, `tool_call`, `tool_result`, `message_end`, `turn_end`, `agent_end`, and others. Handlers run in-process, in registration order, and depending on the event they can transform data, replace results, or cancel an operation outright.

This is the most powerful model of the four: real code, typed inputs and outputs, running inside the agent with full access to session state. It is also the most coupled. An extension runs with the same operating-system permissions as the agent, it is loaded through a runtime rather than discovered on disk, and there is no path that starts with "create a file." If you want a script to append a line to a log when a run finishes, you are writing a TypeScript module and registering a handler for it.

### The trade-off in one table

| Dimension | SCORPIOX CODE | Claude Code | Codex | Hermes | Pi |
|-----------|---------------|-------------|-------|--------|-----|
| Where hooks live | `.scorpiox/hooks/<event>/` folders | JSON settings files + plugin/skill front matter | `config.toml`, `hooks.json` | Four separate systems | TypeScript extension modules |
| Registration | Drop a file in the folder | Declare in JSON | Declare in config, then trust it | Manifest + handler / in code / consent prompt | `pi.on()` in a module |
| Event selection | The folder name | Event + matcher (exact or regex) | Event + regex matcher group | `events:` list, matchers on tool events | `pi.on("<event>")` |
| Payload | `$1`–`$4` arguments + env vars | JSON on stdin | JSON on stdin (`notify`: JSON argument) | JSON on stdin / keyword args | Typed event object |
| Sync vs. async | Filename prefix (`sync-`) | Per-handler `async` flag | Per-handler `async` flag | Per-hook config and timeout | In-process, awaited |
| Blocking | Non-zero exit aborts the async phase | Exit `2` blocks (per event) | Decisions in JSON, exit `2` on some events | `pre_tool_call` can block, `fail_closed` option | Handler return value cancels |
| Script types | `.sh`, `.ps1`, `.py`, `.bat`, executables | Shell / HTTP / MCP / prompt / agent | Shell / MCP tool | Shell / Python / HTTP webhook | TypeScript only |
| Trust gate | None — the directory is the trust boundary | Workspace-trust dialog | Per-definition hash review | First-use consent prompts | Trusted source only |
| Config file required | No | Yes | Yes | Yes | Yes |

The common shape is hard to miss: every one of the four requires you to describe the hook in some declarative or programmatic form before it runs. SCORPIOX CODE's position is that the description is redundant — the file on disk *is* the description, and the filesystem is the registry. You give up matchers, decision objects, and typed handler APIs, and in exchange you get a guarantee you can verify with `ls`: **if it is not in the folder, it does not run.** There is also nothing to approve, because the thing you would approve is the file you just wrote.

---

## What hooks are not for

Hooks are the right tool for **deterministic side effects on lifecycle events** — a log line on session start, a notification when a run completes, a guard that stops the async fan-out when a condition fails. They are the wrong tool for:

- **Recurring or delayed work.** Use scheduled callbacks: the agent can set and manage its own timers, and the mechanism is built for firing while the session is idle.
- **Model-visible context injection.** A hook cannot feed content back into the model's context. The agent reads what it needs from the session folder on disk; put the material somewhere it looks.
- **Interactive permission decisions.** The permission system owns that gate. A hook can observe that a run is starting, but it is not the approval dialog.

A useful rule of thumb: if you want to **react** to something the agent did, write a hook. If you want to **steer** what the agent does next, hooks are the wrong surface — they are observers with a narrow blocking capability, not controllers.

---

## Gotchas

- **The sync prefix is `sync-`, not `sync_` or `sync.`.** The check is for the literal prefix; anything else runs as async.
- **A failing sync hook cancels the async hooks for that event only.** It does not cancel other events, and it does not stop the agent. The agent keeps running; only the rest of that event's fan-out is skipped.
- **Async hooks are fire-and-forget.** Exit codes are logged, not acted on, and the hooks run in parallel. Two async hooks writing to the same file will interleave.
- **CRLF is repaired for `.sh` files before each run.** A hook edited on Windows and committed with CRLF is rewritten to LF on first execution; from then on the file on disk is LF.
- **`emit` is silent for unknown events.** No folder, no work, exit 0. This is what makes custom event names possible — and it also means a typo in an event name fails quietly. `scorpiox-hook list` is how you confirm the folder you meant to create exists.
- **`test` always runs synchronously.** Async hooks behave as sync under `test` so their output is visible. Production behavior is unchanged.
- **Session-less emissions log to the fallback file.** `emit` without `--session`, or with a session ID that has no folder on disk, writes to `.scorpiox/hooks/logs/hook.log` rather than a per-session log. If you are looking in the wrong one, the log is not empty — it is elsewhere.
- **Hooks run as you, with your permissions.** There is no sandbox and no capability model. Anyone who can write into `.scorpiox/hooks/` can run code as your user, which is the same trust you already extend to shell scripts in your repository — treat the folder accordingly, and review hook scripts the way you review any other code you execute.
- **Up to 64 hooks per event folder.** The scanner stops counting past that; keep the folder for scripts that genuinely share an event, and split distinct jobs across events.

---

## Related

- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md) — the timer mechanism for recurring work, and why it is not a hook.
- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md) — the session folder that `hooks.log` lives inside.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the other per-session record, for what the agent said to the provider.
- [Configuration and Profiles](scorpiox-env.md) — where `HOOKS_ENABLED` and `HOOKS_TIMEOUT` sit in the configuration cascade.
- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md) — hook output never leaves your disk either.
