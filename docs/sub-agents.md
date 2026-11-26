# Sub-Agents and Agent Swarms

One agent is a conversation. Two agents is a team. Ten agents working the same repository at once is a fleet — and the moment you get past two, the hard problems stop being about the models and start being about *plumbing*: how do you start them, how do you see what they are doing without interrupting them, how do you keep them from overwriting each other's files, and how do you stop them all when the job is done?

SCORPIOX CODE ships a built-in skill that answers exactly that. It is called **`preferred-sub-agents`**, and it is the house style for launching, peeking at, and orchestrating sub-agents and multi-agent swarms. It invents nothing new to learn — it is a short, precise set of instructions the agent follows, built on two things you already have: a terminal multiplexer to hold the sub-agent sessions, and a harness to run inside each one.

Docs for SCORPIOX CODE @ `902275b`.

> **The whole idea in one line:** every sub-agent is just an agent CLI running in its own multiplexer session — `scorpiox-multiplexer` (preferred, cross-platform) with `tmux` as the Unix fallback — and the `preferred-sub-agents` skill is the checklist that makes launching, peeking at, and cleaning up those sessions reliable instead of ad-hoc.

---

## Why a skill, and not a command

A sub-agent is not a special kind of process. It is a normal SCORPIOX CODE session (`sx`) — or any other agent CLI — running inside a terminal session that you can type into and read from without being attached to it. That is a multiplexer's job, and both the multiplexer and the harnesses already exist.

What goes wrong without a checklist is the *edge cases*:

- A session started at the default `80x24` wraps the agent's TUI output into unreadable ribbons.
- A compound shell command passed as a session startup argument is unreliable — on Windows it bypasses the shell entirely and is capped in length.
- Sending `C-m` instead of `Enter` to the multiplexer types the literal characters `C-m` into the agent.
- A long, quote-heavy brief gets mangled by PTY escaping.
- A worker gets killed while it is still thinking, and its report is lost.
- Two workers edit the same checkout and clobber each other's git index.

The `preferred-sub-agents` skill is the accumulated answer to those. It is loaded on demand like any other skill — the agent sees its name and one-line description, and reads the full instructions when it needs to spawn or supervise something. You can also read it yourself at any time:

```bash
scorpiox-builtin-skills show preferred-sub-agents
```

If that command is unfamiliar, the general skill machinery — where skills live, how project and user skills override built-ins, how a skill body is loaded into context — is covered in [Skills in SCORPIOX CODE](skills-system.md). This page is the sub-agent playbook on top of it.

---

## The two preferences, and the fallback

The skill makes exactly two choices for you, and states the fallback for each:

| Choice | Preferred | Fallback |
|---|---|---|
| **Multiplexer** | `scorpiox-multiplexer` (`sxmux`) — cross-platform, auto-starts its daemon on first command | `tmux`, on Linux and macOS only, and only if `scorpiox-multiplexer` is not installed |
| **Harness** | `sx` — SCORPIOX CODE itself, with config profiles | Any external harness when you ask for one or the task suits it: `claude`, `codex`, `agy` (Google Antigravity CLI), `opencode`, and so on |

`scorpiox-multiplexer` is the interesting half. It is the native session engine that `scorpiox-tmux` can drive as a backend, exposed here as a plain command-line tool for scripted use, and it works on **Linux, macOS, and Windows** (through ConPTY). You do not start a server: the first command you run starts the daemon for you.

`tmux` remains the graceful fallback on Unix, because every core operation has a drop-in tmux equivalent. Windows has no `tmux` binary at all, so on Windows it is `scorpiox-multiplexer` or nothing — which is exactly why it is the preferred choice everywhere.

> **Do not confuse the two layers.** This page is about driving the *multiplexer CLI* directly to hold sub-agent sessions. The `scorpiox-tmux` dashboard is a friendlier front end over the same engine, for interactive use and fleet monitoring. See [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md) for the dashboard, the launch modes, and the per-session model/profile pickers.

---

## Launching your first sub-agent

Every spawn is the same five moves. Nothing is attached to your terminal; you stay in control of the session you are already in.

### 1. Create a detached session, wide

```bash
scorpiox-multiplexer new -s sub-worker1 -d -c /path/to/repo -x 160 -y 48
```

`-s` names the session, `-d` makes it detached, `-c` sets the working directory the agent starts in, and `-x 160 -y 48` gives it a window big enough for a TUI. The dimensions matter more than they look: at the default `80x24` the agent's own interface wraps and you will be reading broken lines when you peek.

### 2. Tag it (optional, but do it for more than one)

```bash
scorpiox-multiplexer set-option -t sub-worker1 @role researcher
scorpiox-multiplexer set-option -t sub-worker1 @harness sx
```

Session options are free-form metadata. Name them `@role`, `@task`, `@harness` — whatever your swarm needs — and they show up in the JSON listing, so the listing becomes a status board instead of a list of names you have to remember.

On Windows PowerShell, quote the option name (`"@role"`), or PowerShell will try to interpret the leading `@`.

### 3. Start the harness inside the session

Two calls, never one. The text goes in literally, then `Enter` is sent separately:

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "sx -e ACTIVE_PROFILE=<profile>"
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

Drop the `-e ACTIVE_PROFILE=...` to run `sx` on whatever configuration it would normally pick up. This two-step form is not a style preference. On Windows a compound command handed to the multiplexer as a startup argument bypasses the shell entirely and is length-capped, so quoting and chains break; typing the command into the session's real shell handles both.

### 4. Confirm it booted

```bash
scorpiox-multiplexer capturep -t sub-worker1 -p
```

You want to see the agent's prompt — in `sx` that is a `>` — sitting idle. If you see a login prompt, an error, or a half-drawn screen, fix that before sending work.

### 5. Send the task

Covered in [Sending work](#sending-work) below.

---

## Choosing the harness

### `sx` with profiles: the default

The preferred harness is SCORPIOX CODE itself, because it brings config profiles to the party. A profile is a named bundle of overrides — provider, model, tool switches, start directory — that you can apply to one sub-agent without disturbing anything else on the machine. That is how you run a cheap fast worker next to an expensive careful one, or point one worker at a different provider entirely.

Profiles are `<name>.txt` files inside a `scorpiox-env` folder. The cascade, highest priority first, is the same one the rest of the product uses: the project's `.scorpiox/scorpiox-env/` (or `.claude/scorpiox-env/`), then `~/.claude/scorpiox-env/`, then the folder next to the installed binaries. Everything about how a profile is layered, and the difference between the saved and the temporary switch, is on [Configuration and Profiles](scorpiox-env.md) — do not re-derive it here.

To see what you already have:

```bash
# Linux / macOS
ls ~/.claude/scorpiox-env/

# Windows (PowerShell)
Get-ChildItem "$env:USERPROFILE\.claude\scorpiox-env" -Name
```

To see which one is active in the current config:

```bash
scorpiox-config --get ACTIVE_PROFILE
```

**A word on cost.** The skill is explicit that you should not move a sub-agent onto a paid or expensive profile without the user agreeing to it first. A swarm that silently spins up a dozen workers on the top-tier model is an expensive way to learn what a swarm costs. Pick the cheap profile by default and escalate deliberately.

### Profiles from frame zero: `-e ACTIVE_PROFILE`

The cleanest way to give a worker a profile is to give it at launch, so it never spends a single turn on the wrong model or provider:

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "sx -e ACTIVE_PROFILE=fast"
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

`-e KEY=VALUE` is a repeatable override applied before the provider initialises, and it stacks — you can pin a provider and a model and a tool switch in one launch if the task needs it.

### Switching a running worker: `/use`

Sometimes you decide mid-flight that this worker should change profile. Inside an idle `sx` session, `/use <profile>` switches the profile **for this session only** and writes nothing to disk. It is the temporary twin of `/profile`, which persists. (If you need the full rule, [Configuration and Profiles](scorpiox-env.md) spells out the one-line difference.)

The catch is mechanical: `/use` opens an argument-completion dropdown, so it needs **two** `Enter`s — the first accepts the completion, the second executes the command.

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "/use fast"
scorpiox-multiplexer send-keys -t sub-worker1 Enter
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

Then peek to confirm. You are looking for a confirmation line starting `Profile '<profile>' (session):`, followed by the model, the provider and the tool switch the session is now running. The `(session)` marker is how you know the switch stayed in this session instead of being written to disk:

```bash
scorpiox-multiplexer capturep -t sub-worker1 -p
```

### External harnesses

The multiplexer does not care what is running inside the session. If you or the user want a different agent CLI, the commands are identical — only the typed line changes:

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "claude"
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

| Harness | Line to type | What you give up |
|---|---|---|
| `sx` | `sx` or `sx -e ACTIVE_PROFILE=<name>` | nothing — this is the preferred path |
| `claude` | `claude` | SCORPIOX CODE config profiles and `-e` overrides |
| `codex` | `codex` | same |
| `agy` | `agy` | same |
| `opencode` | `opencode` | same |

External harnesses are a legitimate choice, not a compromise: sometimes the other CLI is already signed in, already configured, or simply the one the user trusts for a job. The supervision mechanics below — peeking, steering, killing, tagging — are entirely harness-agnostic.

---

## Sending work

Two rules cover almost every mistake people make here:

1. **Always send text with `-l` (literal), and send `Enter` as its own call.**
2. **Never send `C-m`.** To `scorpiox-multiplexer` it is not a carriage return; it types the literal characters `C-m` and the agent sits there.

A normal prompt needs exactly one `Enter` after the literal text.

### Short prompt

```bash
scorpiox-multiplexer send-keys -t sub-worker1 -l "Audit src/auth.c for race conditions and summarize findings."
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

### Long or multi-line brief

Do not paste a multi-paragraph brief through the PTY. Write it to a file, then send a one-line pointer:

```bash
# write /tmp/task-worker1.md with the goal, context, constraints, and the
# exact path the report must be written to, then:
scorpiox-multiplexer send-keys -t sub-worker1 -l "Read /tmp/task-worker1.md and complete it. Write your report to /tmp/report-worker1.md."
scorpiox-multiplexer send-keys -t sub-worker1 Enter
```

This is the single highest-leverage habit in the whole playbook. A self-contained task file carries exact paths, exact commands, and the acceptance criteria without a single escaping problem — and it leaves a durable record of what you asked for, which is invaluable when two workers disagree later.

---

## Peeking and managing

You should almost never attach to a sub-agent. Peeking gives you what you need, read-only, from wherever you are.

```bash
# every session, plus the @options you tagged, as JSON
scorpiox-multiplexer ls --json

# the current screen of one session
scorpiox-multiplexer capturep -t sub-worker1 -p
```

Two flags make peeking much more useful: `-S -100` pulls in scrollback, so you can read the agent's last several screens instead of just the tail; `-J` trims blank lines, which makes the output far easier to scan in a script or when piping it somewhere.

### Read the state before you type

`capturep` tells you whether a worker is free. If the screen shows it thinking, retrying, or in the middle of a tool call, **leave it alone** — typing into a busy agent corrupts its turn. Only send a follow-up when it is idle at its `>` prompt.

### Do not block with `sleep`

The instinct when waiting for a sub-agent is `sleep 120 && capturep`. Do not. Blocking burns the session you are in and gives you exactly one look. Instead, schedule a callback to wake you in a minute or three, peek, and continue — the callback re-enters your own loop like a typed line. The mechanics and the limits are on [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md).

### Steering and stopping

Sending a follow-up is the same literal-plus-`Enter` sequence. Stopping is one command:

```bash
scorpiox-multiplexer kill -t sub-worker1
```

Before you kill anything, two checks:

- **Never kill your own session.** Compare the name against `$SXMUX_SESSION` on Linux and macOS, `$env:SXMUX_SESSION` on Windows, or `$TMUX` under tmux. Racing to clean up the swarm and killing the session you are running in is a genuinely bad afternoon.
- **Confirm the work is saved.** Peek first. If the worker has not yet written its report or committed its branch, killing it throws that away.

A graceful exit first is often kinder than a kill, because it lets the agent flush what it was doing. The unambiguous way is to type `/exit` (literal) and then `Enter`; the skill also allows a `C-c` interrupt, which is a key name rather than literal text — so it is sent **without** `-l`. Either way, `kill` remains the backstop when a session is truly wedged.

---

## The swarm pattern

A single sub-agent is a convenience. The swarm is where the leverage is — and where the failure modes get expensive. Four rules keep it sane.

### 1. Give every editing worker its own worktree

Two agents editing one checkout will corrupt each other's files and git index; there is no clever way around it, only isolation. Each worker that *writes* gets a worktree and a branch of its own:

```bash
git worktree add ../repo-worker1 -b worker1
git worktree add ../repo-worker2 -b worker2
```

Then start each session with `-c` pointing at *its* worktree, not the shared repo. Workers that only read — research, review, search — can safely share the main checkout.

[Isolated Git Worktrees for Parallel Agent Sessions](git-worktrees.md) covers the isolation model in depth and the `scorpiox-tmux` dashboard's automatic equivalent for named sessions. The manual `git worktree add` above is the version you control when you are scripting the fan-out yourself.

### 2. Name every session with a consistent prefix

`sub-api`, `sub-ui`, `sub-tests` — a prefix turns a list of sessions into a swarm you can reason about at a glance. Combined with `set-option` tags, `ls --json` becomes a live status board: role, task, harness, and liveness in one command, with no attaching and no guessing.

### 3. Give every worker its own report file

Tell each worker, in its task file, exactly where to write. `/tmp/swarm-<role>-report.md` is fine. Two benefits: nothing collides, and when the swarm finishes you have N parallel findings documents to synthesise instead of N terminal screens to scroll back through before the scrollback is gone.

### 4. Fan out, then wait — do not babysit

Spawn every worker first. Then schedule a callback check (a minute to a few minutes out), peek at all of them, and only intervene where a worker is actually stuck or idle with its report unwritten. When they are done, read the reports and the diffs, synthesise the result, and clean up.

The temptation is to watch one worker closely while the others idle. Resist it: the swarm's whole value is parallelism, and a callback that wakes you every ninety seconds costs nothing.

---

## Windows notes

Most of the friction in this playbook is Windows-shaped, so it is worth naming once:

- **Always `scorpiox-multiplexer`.** There is no `tmux` binary on a stock Windows machine, and the fallback does not exist there.
- **Quote `@option` names in PowerShell.** `"@role"`, not `@role`, or PowerShell treats it as splatting syntax.
- **Never pass a compound command as a startup argument.** Two-step literal `send-keys` exists precisely because the Windows spawn path skips the shell and caps command length.
- **One runtime directory per visibility scope.** Session sockets live under a runtime directory; on Windows the default is per-logon, so a session created interactively is invisible to a service running under a different account. If a service (SCORPIOX BOT, for instance) must see desktop-started sessions, point both at one machine-wide directory with `SXMUX_RUNTIME_DIR`. See [Configuration and Profiles](scorpiox-env.md) for the cascade.
- **Wide sessions everywhere.** `-x 160 -y 48` is not a Unix-only nicety; wrapped TUI output is just as unreadable in ConPTY.

---

## The `tmux` fallback, line for line

On Linux and macOS, when `scorpiox-multiplexer` is not installed, every operation has a direct `tmux` equivalent. The skill keeps the two aligned; use whichever you have.

| Action | `scorpiox-multiplexer` (preferred) | `tmux` (fallback) |
|---|---|---|
| Create | `scorpiox-multiplexer new -s <n> -d -c <dir> -x 160 -y 48` | `tmux new-session -d -s <n> -c <dir> -x 160 -y 48` |
| List | `scorpiox-multiplexer ls --json` | `tmux list-sessions` |
| Send text | `scorpiox-multiplexer send-keys -t <n> -l "<txt>"` then `Enter` | `tmux send-keys -t <n> -l "<txt>"` then `Enter` |
| Peek | `scorpiox-multiplexer capturep -t <n> -p` | `tmux capture-pane -t <n> -p` |
| Kill | `scorpiox-multiplexer kill -t <n>` | `tmux kill-session -t <n>` |

The two invariants survive the fallback unchanged: literal send, then a separate `Enter`. Under `tmux` the "am I about to kill myself?" variable is `$TMUX` instead of `$SXMUX_SESSION`.

---

## Opting out, or writing your own

The skill is a default, not a policy. Teams and individuals with an established sub-agent workflow can turn it off or replace it, cleanly, at whatever scope they need. The toggle is **`SKILL_PREFERRED_SUB_AGENTS`**:

| Scope | How to set it |
|---|---|
| One project (git-tracked) | `SKILL_PREFERRED_SUB_AGENTS=0` in `<project>/.claude/scorpiox-env.txt`, or `<project>/.scorpiox/scorpiox-env.txt` (if both exist, `.scorpiox/` wins) |
| One user | `SKILL_PREFERRED_SUB_AGENTS=0` in `~/.claude/scorpiox-env.txt` |
| One profile | `SKILL_PREFERRED_SUB_AGENTS=0` inside that profile's `.txt` |
| One run | `sx -e SKILL_PREFERRED_SUB_AGENTS=0` |
| One shell and everything it launches | `export SKILL_PREFERRED_SUB_AGENTS=0` before starting `sx` |

Set to `0`, the skill is not loaded at all, so it costs nothing in context and the agent will not reach for it. Set to `1` (the shipped default) and it is available again.

Verify what is actually in force rather than what you think you wrote:

```bash
scorpiox-config --get SKILL_PREFERRED_SUB_AGENTS
```

That check matters because the cascade is layered. A real environment variable outranks every file; below it sits the active profile's `.txt`, then the project tier, then your user tier, then the global install next to the binaries. So a `0` written into a project file (or into the profile a project selects) beats a `1` you set at user level. [Configuration and Profiles](scorpiox-env.md) has the full ordering; the same tiering that governs `ACTIVE_PROFILE` governs this toggle.

### Overriding rather than disabling

If you want the skill's *name* to keep working but with your own instructions — your house naming scheme, your own harness, your own cleanup rules — drop a skill folder with the same name into your project or your home directory:

```
<project>/.claude/skills/preferred-sub-agents/SKILL.md
~/.claude/skills/preferred-sub-agents/SKILL.md
```

Project and user skills take priority over the built-in of the same name, so yours is what the agent reads and the built-in is shadowed. You do not need to disable anything; the override is enough. The priority order and the `.agents/skills` fallback are described in [Skills in SCORPIOX CODE](skills-system.md).

---

## Gotchas

- **`80x24` will ruin your peeks.** Start every session at `-x 160 -y 48`. Wrapped TUI output is the number one cause of "the agent looks broken" when it is fine.
- **`C-m` is not `Enter`.** It is typed literally. Always send `Enter` as its own `send-keys` call.
- **`-l` is not optional** for anything containing spaces or quotes. Without it, the multiplexer splits on spaces and eats your quoting.
- **Two `Enter`s for `/use`.** The first accepts the completion dropdown, the second runs the command. One `Enter` leaves the agent sitting in the picker.
- **Never type into a busy agent.** Peek first; only send when it is idle at `>`.
- **Never kill your own session.** Check `$SXMUX_SESSION` (`$env:SXMUX_SESSION` on Windows) or `$TMUX` before the kill.
- **Do not pass compound commands as a session startup argument.** It breaks on Windows. Create detached, then type the command in.
- **One checkout per writing worker.** Read-only workers may share; writers must not.
- **Do not put a swarm on expensive profiles by default.** Ask first, and prefer the cheap profile.
- **`SKILL_PREFERRED_SUB_AGENTS` is the on/off switch, not the instructions.** To change behaviour, override the skill; to remove it, set the key to `0`.

---

## Related

- [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md) — the interactive dashboard over the same multiplexer engine: session lifecycle, launch modes, peek versus watch, and per-session model and profile selection.
- [Isolated Git Worktrees for Parallel Agent Sessions](git-worktrees.md) — the isolation layer that makes parallel workers on one repository safe.
- [Configuration and Profiles](scorpiox-env.md) — the `scorpiox-env.txt` cascade, where profiles live, and the `/profile` versus `/use` distinction in full.
- [Skills in SCORPIOX CODE](skills-system.md) — how skills are discovered, loaded on demand, and overridden by project and user skills.
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md) — the non-blocking way to wait for a swarm instead of sleeping.

---

*Docs for SCORPIOX CODE @ `902275b`. Sub-agent supervision is built on the `preferred-sub-agents` built-in skill, enabled by default with `SKILL_PREFERRED_SUB_AGENTS=1`.*
