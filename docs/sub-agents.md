# Sub-Agents and Agent Swarms

You want a second agent auditing the auth code while the first one refactors the UI — or six agents fanning out over six modules at once. You do not want to babysit terminal windows, and you do not want a swarm of agents clobbering each other's files.

SCORPIOX CODE ships this as a **built-in skill: `preferred-sub-agents`**. It is loaded automatically in every session and teaches the agent the exact recipes for spawning sub-agents, peeking at them while they work, orchestrating swarms, and cleaning up afterwards — using the tools you already have: `scorpiox-multiplexer` (a.k.a. `sxmux`) and the `sx` CLI. The agent follows it without you writing a single word of instructions.

Docs for SCORPIOX CODE @ `41b090b`.

> **The whole idea in one line:** one built-in skill that tells the agent how to launch a sub-agent in its own multiplexer session, send it a task, peek at its screen until it is done, and tear it down — alone or in a swarm.

---

## The two building blocks

Every sub-agent lives in a **multiplexer session** (an isolated terminal) and runs inside a **harness** (the agent CLI that actually does the work). SCORPIOX CODE has clear preferences for both:

| Building block | Preferred | Fallback / alternatives |
|---|---|---|
| Multiplexer | `scorpiox-multiplexer` (`sxmux`) — cross-platform: Linux, macOS, and Windows (ConPTY); its daemon auto-starts on first use | `tmux` on Linux/macOS only, if `scorpiox-multiplexer` is not installed |
| Harness | `sx` — supports config profiles via `-e ACTIVE_PROFILE=<name>` or live `/use <name>` | External harnesses: `claude`, `codex`, `agy`, `opencode` — same multiplexer commands, different command on the keyboard |

The built-in skill enforces these preferences by default, and the agent swaps in an external harness when you or the task ask for one.

---

## How the agent picks a profile

Profiles are named configuration sets — a mode that overrides only the keys it names (see [Configuration and Profiles](scorpiox-env.md)). For sub-agents they matter because a worker might need a different provider, model, or budget than your interactive session.

The agent discovers available profiles by listing the profile folder:

```bash
# Linux / macOS
ls ~/.claude/scorpiox-env/

# Windows (PowerShell)
Get-ChildItem "$env:USERPROFILE\.claude\scorpiox-env" -Name
```

It can check the current profile with `scorpiox-config --get ACTIVE_PROFILE`. **It will not switch a sub-agent onto a paid or expensive profile without your approval.**

---

## Spawning a sub-agent

The recipe is: one detached session at wide dimensions, then a **two-step literal `send-keys`** that launches the harness.

Why the two-step form? Two reasons baked into the skill:

- **Wide dimensions** (`-x 160 -y 48`): the default `80x24` wraps TUI output and mangles the agent's screen.
- **Literal text + separate `Enter`**: compound shell commands passed as positional arguments to `new` break on Windows (`CreateProcessA` skips the shell and caps command length), and literal sending avoids PTY escaping and wrapping issues.

```bash
# 1. Create a detached session in the target working directory
scorpiox-multiplexer new -s sub-worker1 -d -c /path/to/repo -x 160 -y 48

# 2. Optional: tag the session with metadata (visible in `ls --json`)
#    On Windows PowerShell, quote "@role" so PowerShell does not treat @ as splatting
scorpiox-multiplexer set-option -t sub-worker1 @role researcher
scorpiox-multiplexer set-option -t sub-worker1 @harness sx

# 3. Launch the harness from frame 0 (profile form preferred)
scorpiox-multiplexer send-keys -t sub-worker1 -l "sx -e ACTIVE_PROFILE=<profile>"
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

To launch with your default configuration, or to launch an external harness instead, the same two-step form works with any command:

```bash
# Default-config sx, or an external harness
scorpiox-multiplexer send-keys -t sub-worker1 -l "sx"
# scorpiox-multiplexer send-keys -t sub-worker1 -l "claude"
# scorpiox-multiplexer send-keys -t sub-worker1 -l "codex"
# scorpiox-multiplexer send-keys -t sub-worker1 -l "agy"
# scorpiox-multiplexer send-keys -t sub-worker1 -l "opencode"
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

Before sending any task, the agent verifies the harness booted and is idle at its `>` prompt:

```bash
scorpiox-multiplexer capturep -t sub-worker1 -p
```

---

## Switching profiles on a running sub-agent

No restart needed. Inside an already-running, idle `sx` session, `/use <profile>` opens an argument-completion dropdown, so the agent sends **two `Enter`s** — the first accepts the completion, the second executes:

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "/use <profile>"
scorpiox-multiplexer send-keys -t sub-worker1 Enter
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

It then peeks and confirms `Profile '<profile>' (session):` appeared on screen.

---

## Sending tasks

Three rules the skill enforces for every keystroke:

1. **Always `-l` (literal) for text**, and send `Enter` as a separate call.
2. **Never send `C-m` to `scorpiox-multiplexer`** — it is typed literally there; always use `Enter`.
3. Normal prompts need **one `Enter`**.

### Short prompt

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "Audit src/auth.c for race conditions and summarize findings."
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

### Long or multi-line brief (recommended for real work)

Write a self-contained task file — goal, context, constraints, and the exact report path — and send a one-line pointer. This sidesteps PTY escaping and wrapping entirely:

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "Read /tmp/task-worker1.md and complete it. Write your report to /tmp/report-worker1.md."
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

---

## Peeking and monitoring

```bash
# List all sessions and their @options as JSON
scorpiox-multiplexer ls --json

# Peek the current screen (add -S -100 for scrollback, -J to trim blank lines)
scorpiox-multiplexer capturep -t sub-worker1 -p
```

Two etiquette rules:

- **Do not interrupt working agents.** If `capturep` shows `Thinking...`, `Retrying...`, or an active tool call, leave it alone. Send follow-up input only when the agent is idle at the `>` prompt.
- **Do not block with `sleep` loops.** The agent schedules a callback (e.g. 60–240s) to wake up, peek, and continue — the main session stays responsive.

---

## Running a swarm

When you fan out multiple sub-agents in parallel, the skill follows four rules:

1. **Isolate code edits with `git worktree`.** Each editing worker gets its own worktree (`git worktree add ../repo-worker1 -b worker1`) so parallel agents never clobber each other's files or git index. Read-only research workers can share the repo directory. The full story is in the *git worktrees* page.
2. **Name and tag every session.** A consistent prefix (`sub-api`, `sub-ui`, `sub-tests`) plus `set-option` tags (`@role`, `@task`, `@harness`) turns `ls --json` into an instant swarm status board.
3. **A dedicated report file per worker.** Each worker writes to its own path (e.g. `/tmp/swarm-<role>-report.md`) and stops when done — you synthesize from files, not from six open terminals.
4. **Fan out, then wait.** Spawn all workers first, schedule a callback to check, inspect outputs and diffs when idle, synthesize the results, and clean up.

---

## Safe cleanup

- **Never kill your own session.** The agent checks `$SXMUX_SESSION` (`$env:SXMUX_SESSION` on Windows, `$TMUX` under the tmux fallback) before killing anything.
- **Confirm the worker finished** and its report and commits are saved, then terminate. A graceful exit first (`C-c` or `/exit`) is optional:

```bash
scorpiox-multiplexer kill -t sub-worker1
```

---

## Fallback: `tmux` (Unix only)

If `scorpiox-multiplexer` is not installed on Linux/macOS, the skill falls back to `tmux` with equivalent commands:

| Action | `scorpiox-multiplexer` (preferred) | `tmux` (fallback) |
|---|---|---|
| Create | `scorpiox-multiplexer new -s <n> -d -c <dir> -x 160 -y 48` | `tmux new-session -d -s <n> -c <dir> -x 160 -y 48` |
| List | `scorpiox-multiplexer ls --json` | `tmux list-sessions` |
| Send text | `scorpiox-multiplexer send-keys -t <n> -l "<txt>"` + `Enter` | `tmux send-keys -t <n> -l "<txt>"` + `Enter` |
| Peek | `scorpiox-multiplexer capturep -t <n> -p` | `tmux capture-pane -t <n> -p` |
| Kill | `scorpiox-multiplexer kill -t <n>` | `tmux kill-session -t <n>` |

The full session-manager story — attaching, watching, remote sessions — is in [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md).

---

## Opting out or overriding

If your project or workflow has its own sub-agent conventions, you can disable or replace the built-in skill cleanly. The config cascade (see [Configuration and Profiles](scorpiox-env.md)) decides which layer wins:

- **Disable per project (git-tracked):** add `SKILL_PREFERRED_SUB_AGENTS=0` to `<project>/.claude/scorpiox-env.txt` (or `<project>/.scorpiox/scorpiox-env.txt`). Every clone of the repo inherits it.
- **Disable per user, per profile, or per run:** add `SKILL_PREFERRED_SUB_AGENTS=0` to `~/.claude/scorpiox-env.txt`, to a profile `.txt` file, or pass `sx -e SKILL_PREFERRED_SUB_AGENTS=0` for a single launch.
- **Override the instructions:** drop a skill at `.claude/skills/preferred-sub-agents/SKILL.md` in the project (or in `~/.claude/skills/` for your user level). Project and user skills override built-ins — same name, higher layer wins. See the *skills system* page for how skill sources are resolved.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — how `ACTIVE_PROFILE`, profiles, and the `KEY=VALUE` cascade work
- [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md) — the session dashboard and its headless CLI
- *Isolated Git Worktrees for Parallel Agent Sessions* — worktree isolation for parallel agents
- *Skills System* — built-in, project, and user skills and their precedence
