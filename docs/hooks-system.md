# Event Hooks in SCORPIOX CODE: Folder-Based Automation Architecture

You want SCORPIOX CODE to do something every time a specific thing happens — log a line when a session starts, run a linter after a tool fires, post a message the moment the agent goes idle, or gate the whole run on a validation script that must pass before any API call. That is what the **event hook system** is for. It watches the agent's lifecycle and runs *your* scripts at the moments you pick, without you touching the agent's code and without writing a single line of config.

The whole mechanism is a folder. Hooks live in `.scorpiox/hooks/<event>/`. A script sitting in that directory runs when the matching event fires. That is the entire registration step. There is no manifest, no JSON to validate, no TOML to keep in sync, no matcher syntax to learn. The folder is the wiring.

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** drop a script into `.scorpiox/hooks/<event>/` and it runs the next time that event happens — the filesystem *is* the hook registry, and the filename tells you how it behaves.

---

## The core principle: the folder is the registry

Most agent harnesses make you *declare* hooks in a config file — a JSON object, a TOML section, a manifest entry — and the harness reads that file to learn what to run and when. SCORPIOX CODE inverts this. It does not read a list you write. It looks at the disk.

Concretely:

1. Each lifecycle event has a directory: `.scorpiox/hooks/session_start/`, `.scorpiox/hooks/tool_use/`, and so on.
2. Every eligible file inside an event directory is a hook for that event.
3. **Add a file** = add a hook. **Delete a file** = remove a hook. **Rename a file** to prefix it with an underscore = disable it without deleting it.
4. There is no separate registry to update, and no schema to keep valid. A file you cannot parse is simply skipped, never a crash.

The practical consequence is that hooks behave the way you would expect from a well-behaved shell: they are ordinary, inspectable files you can open, `git` track, diff, and copy between machines. They are not config blobs bound to one harness's internal format.

---

## The lifecycle events

The system knows a set of lifecycle events. Each one is a directory you can put scripts in. The full list (run `scorpiox-hook events` to print it):

| Event | When it fires | Typical data |
|-------|---------------|--------------|
| `session_start` | A new session begins | model, provider, log level |
| `session_end` | A session ends gracefully | — |
| `session_clear` | The user clears the session | — |
| `user_message` | The user sends a message | message length |
| `agent_run_start` | The agent begins a run | message length |
| `agent_complete` | The agent finishes a run | turn count, history length |
| `agent_idle` | The agent is stopped and waiting at the prompt | — |
| `subagent_complete` | A sub-agent finishes a run | turn count, history length |
| `agent_cancelled` | The user cancels the agent | reason, turn count |
| `tool_use` | A tool is invoked | tool name, tool id |
| `mcp_call` | An MCP server tool is called | server, tool |
| `api_response` | An API response is received | turn, stop reason, token counts |
| `api_error` | An API error occurs | error, status code |
| `api_retry_wait` | A transient API error, before resending | status, attempt, delay |
| `api_retry_resume` | The agent resends after a retry wait | attempt, turn |
| `api_retry_exhausted` | Retries are used up | status, attempt count, turn |
| `compact` | The conversation is compacted | from/to message counts |
| `hook_failed` | A hook exits non-zero | exit code, hook name |

Two details worth knowing up front:

- **The events are open, not closed.** The list above is the set the harness emits. But the runner happily accepts *any* event name — if you `scorpiox-hook emit my_custom_event` and there is a `.scorpiox/hooks/my_custom_event/` directory, your scripts run. You are not limited to the built-ins.
- **Hooks are enabled by default.** A single kill switch (the `HOOKS_ENABLED` setting) turns the whole system on or off. When it is off, no scripts run and nothing is logged.

---

## Sync vs. async, decided by the filename

This is the part that replaces an entire config schema with a naming convention. The **prefix of the filename** tells the runner how to treat the hook.

| Filename pattern | Behaviour |
|------------------|-----------|
| `01-notify.sh` | **Async** (default). Fire-and-forget; the agent does not wait for it. |
| `sync-01-validate.sh` | **Sync**. The runner blocks until it exits, and checks the exit code. |
| `_disabled.sh` | **Skipped.** An underscore prefix disables the hook without deleting it. |
| `.hidden.sh` | **Skipped.** A dot prefix disables the hook without deleting it. |

The numeric prefix (`01-`, `02-`) does not just look tidy — it sets the **order**. Hooks always run in sorted filename order, so `01-` runs before `02-` before `03-`. Deterministic, every run, no extra ordering key to maintain.

### What sync buys you

An **async** hook is detached: the runner launches it and moves on. Use it for anything that is a side effect — logging, notifications, metrics, a message to Slack. The agent does not care about the result, and a slow or failing async hook can never hold up a turn.

A **sync** hook is a gate. The runner waits for it to finish and reads its exit code. If *any* sync hook exits non-zero, the runner aborts and **skips all async hooks for that event**. That is how you turn a hook into a checkpoint: a `sync-` validation script that fails on a bad state stops the event's async work in its tracks.

In the agentless / headless path this is load-bearing: the `session_start` sync hooks run before the agent loop starts, and a non-zero exit is captured and surfaced as the run's result. A sync `session_start` hook is effectively a "do not proceed unless this passes" guard.

---

## What your script receives

Every hook, regardless of event or sync/async, is handed the same four positional arguments:

| Argument | Meaning | Example |
|----------|---------|---------|
| `$1` | Event name | `session_start` |
| `$2` | Session id | `2026_09_29_prickly_hamilton` |
| `$3` | ISO timestamp (UTC) | `2026-09-29T04:30:00Z` |
| `$4` | The event data as JSON | `{"model":"opus","provider":"claude_code"}` |

The same values are also exported as environment variables, so a script can read them positionally *or* by name — whichever is clearer:

| Variable | Value |
|----------|-------|
| `SX_EVENT` | The event name (same as `$1`) |
| `SX_SESSION_ID` | The session id (same as `$2`) |
| `SX_CWD` | The working directory |
| `SX_HOOKS_DIR` | The `.scorpiox/hooks` path |

`$4` is the part that makes hooks composable across events: it is the JSON payload the harness attached to that specific event, so a `tool_use` hook sees `{"tool":"Bash","id":"..."}` while an `api_response` hook sees token counts and the stop reason. The same script shape can branch on `$1` and read the payload from `$4`.

---

## Supported script types

Any of the following extensions run out of the box; the runner picks the right interpreter from the extension, so you do not need the executable bit set:

| Extension | How it runs |
|-----------|-------------|
| `.sh` | `/bin/sh <script>` (CRLF line endings are auto-fixed before running) |
| `.py` | `python3 <script>` |
| `.ps1` | `pwsh -NoProfile -File <script>` |
| `.bat` | `cmd.exe /c <script>` (Windows only) |
| anything else | Executed directly; on Unix it must carry a shebang and the executable bit |

The CRLF auto-fix is a quiet reliability detail: a `.sh` hook written on one machine and checked out on another runs cleanly instead of failing on stray carriage returns.

---

## Execution order: the contract

When an event fires, the runner does exactly this, in this order:

1. **Discover** every eligible hook in the event directory and sort it by filename.
2. **Run all sync hooks first**, one at a time, sequentially, blocking.
3. **If any sync hook exits non-zero**, stop and **skip the async phase entirely**.
4. **Otherwise run all async hooks**, each detached and in parallel.

So the mental model is: *sync hooks are the gate, async hooks are the side effects, and a failed gate silences the side effects.* You never have to reason about two hooks racing over a shared file — the order is the sort order, and sync strictly precedes async.

---

## Per-session logging

Every hook run is logged, per session, so "did my hook actually run, and what did it print" has a one-file answer:

| Location | When |
|----------|------|
| `.scorpiox/sessions/<id>/hooks.log` | The default for any hook that fired inside a named session |
| `.scorpiox/hooks/logs/hook.log` | The fallback when there is no session context |

Each entry records the event, the hook name, whether it was sync or async, the hook's combined stdout/stderr, its exit code, and how long it took. Async hooks write straight to that log; sync hooks echo to the terminal *and* the log. You can reconstruct a session's entire hook behaviour from a single file.

---

## The `scorpiox-hook` CLI

The runner is also a small command-line tool for managing and debugging hooks. It is standalone (no library dependencies) and is what the agent harness calls internally to fire each event.

| Command | What it does |
|---------|--------------|
| `scorpiox-hook events` | Print the list of known events and their descriptions. |
| `scorpiox-hook init` | Scaffold every known event directory under `.scorpiox/hooks/`. |
| `scorpiox-hook list [--event <name>]` | List installed hooks per event, tagged as sync or async, with disabled ones counted. |
| `scorpiox-hook install <event> <file> [--sync]` | Copy a script into the event's directory. `--sync` prefixes the name with `sync-`. |
| `scorpiox-hook test <event> [name] [--data '{json}']` | Run the event's hooks once with sample data and report pass/fail per hook. |
| `scorpiox-hook emit <event> [--data '{json}'] [--session <id>]` | Fire an event on demand (this is what the harness does internally). |

The two you will reach for most often: `init` once to lay out the tree, and `test` to prove a hook behaves before you let it run in a real session. `test` feeds each hook the same four arguments and sample JSON, prints its output, and reports a pass/fail per hook so you can iterate without starting a full conversation.

### A minimal end-to-end example

```sh
# 1. Lay out the event directories
scorpiox-hook init

# 2. A side-effect hook: log every session start
mkdir -p .scorpiox/hooks/session_start
cat > .scorpiox/hooks/session_start/01-announce.sh <<'EOF'
echo "session started: $SX_SESSION_ID ($1) at $3"
EOF

# 3. A gate hook: fail the run before it starts if the tree is dirty
cat > .scorpiox/hooks/session_start/sync-00-check-clean.sh <<'EOF'
[ -z "$(git status --porcelain 2>/dev/null)" ] || {
  echo "working tree is dirty" >&2
  exit 1
}
EOF

# 4. Prove it without a live session
scorpiox-hook test session_start
```

The `sync-00-` hook runs first (it sorts before `01-`), and because it is sync, a non-zero exit aborts the event and the async `01-announce.sh` never runs. The gate and the side effect, in two files.

---

## Disabling, and the kill switch

- **One hook:** rename it with a `_` or `.` prefix. It is ignored until you rename it back. No config edit.
- **One event:** remove (or prefix) everything in that event's directory.
- **Everything:** set `HOOKS_ENABLED` to `0`. The whole system goes quiet — no scripts, no hook logging — until you set it back to `1`. It is on by default.

---

## How this compares to the other harnesses

The comparison that matters is *registration model*, because that is what you live with every time you add, disable, or reorder a hook. SCORPIOX CODE registers hooks by their presence on disk; the others register them in a configuration file they parse.

| Harness | How hooks are registered | Where a hook lives | How you disable one | What you write |
|---------|--------------------------|--------------------|---------------------|----------------|
| **SCORPIOX CODE** | A script in `.scorpiox/hooks/<event>/`. Filesystem is the registry. | A plain file in a per-event folder. | Rename to a `_` / `.` prefix. | A script. The filename encodes sync/async and order. |
| **Claude Code** | A JSON object in `settings.json` mapping lifecycle events (e.g. `PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `Stop`, `Notification`) to arrays of `{matcher, command}` entries. | In the settings file as JSON. | Edit the JSON and remove the entry. | JSON, with a matcher string per entry. |
| **Codex** | A `notify` hook declared in `~/.codex/config.toml` — a command the CLI invokes. | In the TOML config. | Edit the TOML. | A TOML command entry. |
| **Pi** | Wired through the agent's configuration rather than a drop-in tree of scripts. | In the agent config. | Edit the config. | Config entries bound to the harness's event model. |
| **Hermes** | Declared in the agent's configuration / manifest. | In the config or manifest. | Edit the config. | Config entries bound to the harness's event model. |

Read that table and the pattern is the same one that shows up everywhere else in this product:

| | **Config-file harnesses** | **SCORPIOX CODE** |
|---|---|---|
| **Registration** | Declare the hook in a file the harness parses. | The hook *is* a file in a folder. |
| **Schema** | Must be valid JSON / TOML; a typo can break parsing. | No schema. A bad file is skipped, not fatal. |
| **Ordering** | An explicit order key or array position you must maintain. | The filename sort order. |
| **Disabling** | Edit and save the config. | Rename the file. |
| **Inspecting** | Read and mentally parse a config blob. | Open, diff, and `git` track a script. |
| **Portability** | Bound to that harness's config format. | Copy the folder; it runs anywhere. |

The last row is the one the others cannot copy without changing their model. A hook you wrote for SCORPIOX CODE is a plain script. You can copy it into another project, run it by hand from a shell, or point it at a different event by moving it into a different directory. It is not a fragment of someone else's config file.

None of the other approaches are wrong — JSON hooks with matchers are more expressive for per-tool matching, and TOML `notify` is fine for "send me one message when X happens." The trade-off is that SCORPIOX CODE optimizes for the case where **the hook is a real program you already have**, not a config value. If your automation is a script, the folder model keeps it a script.

---

## When to use what

- **Async hooks** for pure side effects: logging, notifications, metrics, posting to a channel. They cannot block or break a turn.
- **Sync hooks** for gates: validation, preconditions, anything where "must pass before we continue" is the point. A non-zero exit stops the event.
- **The numeric prefix** to pin an order you depend on (`00-` for the gate, `10-` for setup, `20-` for the rest).
- **`_` / `.` renaming** to keep a broken or experimental hook around without running it.
- **`scorpiox-hook test`** before trusting a hook in a live session — it is the cheapest way to see exactly what a hook will do with representative data.

---

## Gotchas

- **Sync failure silences async.** If even one sync hook exits non-zero, the async phase for that event is skipped entirely. That is the gate working as designed, but it means a flaky sync hook can quietly suppress your notifications. Keep gates deterministic.
- **Order is the filename sort, not your mental order.** If two hooks must run in a specific sequence, encode it in the names (`01-`, `02-`). Bare names sort too, but the sort is the contract, so make it explicit.
- **Hooks are on by default and per-event, not global.** A script in `.scorpiox/hooks/tool_use/` only runs on `tool_use`. There is no "runs on every event" directory — put the script in each event directory that needs it, or fire a custom event yourself.
- **Async output goes to the log, not the terminal.** An async hook's stdout/stderr lands in the session's `hooks.log`. If you are debugging and see nothing on screen, check the log — the hook ran, it just did not echo to your console.
- **`$4` is JSON, parse it as JSON.** The payload shape differs by event. Reading a field that is absent in some events returns nothing, so branch on `$1` before reaching into `$4`.
- **Unknown events are legal.** If you emit an event with no directory, nothing happens (no error). If you add a directory for a custom event, your scripts run. The system is open on both sides, which means a mistyped event name is a silent no-op rather than a loud failure.

---

## The bottom line

Event hooks in SCORPIOX CODE are a folder, not a config file. A script in `.scorpiox/hooks/<event>/` runs when that event fires; the filename decides whether it gates the run or fires and forgets, and in what order; the four arguments and four environment variables hand it everything it needs; and a per-session log records every run with its exit code and duration. Add a file to add a hook, rename it to disable one, and keep the automation as plain, portable scripts you can diff and version — no schema to satisfy, no manifest to keep in sync.

---

## Related

- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md)
- [Deterministic File Editing & Developer Autonomy in SCORPIOX CODE](file-editing.md)
- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md)
