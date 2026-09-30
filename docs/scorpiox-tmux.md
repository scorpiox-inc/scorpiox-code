# Managing Agent Sessions with scorpiox-tmux

You can run several SCORPIOX CODE agents at once — one per repo, one per branch, one per task. But once they are off running in the background, where do you look to see what each one is doing, start another, or stop one that has gone off the rails?

**`scorpiox-tmux`** is the answer. It is the *Mission Control* for your agents: a terminal dashboard that shows every live agent session at a glance, and a command line (both inside the dashboard and as a headless CLI) to create, attach to, watch, poke, restart, and kill sessions. It is built to be driven from a small phone over SSH as easily as from a full terminal.

A session is a single SCORPIOX CODE agent running in its own isolated environment — a container, a sandbox, or just its own directory. `scorpiox-tmux` is the one place you start one, peek in on it without attaching, send it a keystroke, or shut it down.

It comes in two faces that share the same engine:

- **The TUI** — launch it with no arguments and you get a live dashboard: a list of active sessions up top, a `>` prompt at the bottom where you type slash commands, and a pane view for peeking and watching a session's screen.
- **The headless CLI** — the same operations as `--` flags, for scripts, automation, and "I just want to do this one thing without a dashboard."

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** every agent is a *session*; `scorpiox-tmux` lists them, starts them, lets you peek or watch their screen, sends them input, and tears them down — all from one command, in a TUI or from a script.

---

## Launching

```bash
scorpiox-tmux                 # launch the TUI (dashboard)
scorpiox-tmux --list          # headless: list sessions and exit
```

Run it with no arguments and you get the dashboard. Run it with any `--` flag and it does that one headless thing and exits. There is no mix-and-match — a flag puts it in CLI mode.

The dashboard, from top to bottom:

- **Header bar** — the title, a mode badge (a red `WATCH` or a blue `PEEK` when you are looking at a pane), and a clock on the right.
- **Active Sessions** — one row per session: a status dot (yellow when someone is attached), the session name, how long it has been running, an `(attached)` marker, and a `[worktree: project/session]` label when the session runs in its own git worktree.
- **Status line** — shows the result of your last command for a few seconds, then settles back to a hint.
- **Input bar** — a `>` prompt. This is where you type.

---

## The TUI command reference

Everything in the dashboard starts with `/` at the input bar. Press **Tab** to get autocomplete as you type — it completes command names, project names, launch modes, and session names, so you rarely type more than the first couple of letters.

| Command | What it does |
|---|---|
| `/new <project[:session[:profile]]> [mode\|distro] [@branch]` | Start a new session. Fails if a session with that name already exists. |
| `/enter <project[:session]> [mode] [@branch]` | **New + resume**: create the session if it is not running, then attach to it. If it already exists, it just attaches. |
| `/resume <session>` | Attach to an existing session. Takes over the whole terminal; when you detach, you are back in the dashboard. |
| `/restart <session> [mode] [@branch]` | Kill the session and recreate it (fresh state, same name). |
| `/kill <session>` | Kill a session. Removes its session record; the checkout and any worktree on disk are left alone. |
| `/list` | Refresh the session list now. |
| `/peek <session>` | Take a one-shot snapshot of the session's screen and show it. Press **Esc** to close. |
| `/watch <session>` | Live-poll the session's screen (updates about once a second). Press **Esc** to stop. |
| `/send <session> <command>` | Send a line of keystrokes (plus Enter) to the session. |
| `/projects` | List the projects `scorpiox-tmux` can see in the project base directories. |
| `/clear` | Clear the screen and return to a clean dashboard. |
| `/quit` or `/exit` | Leave `scorpiox-tmux`. |

### How the `<project[:session[:profile]]>` argument is read

- **`project`** — a directory under your project base path, or a direct path like `/tmp/scratch`, `~/work`, or `D:\work`. When you give just a project, the session is *named* after it.
- **`:session`** — a custom session name. Use this to run several sessions on the same project. With git, each named session gets its own **worktree** so it has an isolated working copy.
- **`:profile`** — a per-session profile name (see [Per-session model and profile](#per-session-model-and-profile)).

Examples:

```
/new myapp                     session "myapp", project "myapp"
/new myapp:fix-login           session "fix-login", project "myapp" (own worktree)
/new myapp:fix-login:work      same, but this one session runs under the "work" profile
/new /tmp/scratch:s1           direct directory, no worktree, session "s1"
```

Session names are normalized the way tmux itself normalizes them — dots become underscores — so the name you type is always the name the dashboard, `--list`, and the bot API will report.

### The optional second argument: mode, or a distro

The second word is interpreted in order:

1. If it is a **launch-mode keyword** — `unshare`, `podman`, `wsl`, or `native` — it sets how the session's environment is built.
2. Otherwise it is treated as a **distro / image name**, which means "run in `unshare` mode using that image."

So `/new myapp desktopx11-latest` runs the agent in a desktop image. See [Where and how a session runs](#where-and-how-a-session-runs) for what each of these means.

### The optional `@branch`

Append `@branch` to select **which SCORPIOX CODE build the session deploys** (for example `/new myapp podman@main` or `/new myapp @yyjson`). Leave it off to use the default build. This is about the build *inside* the session, not a git branch of your project.

---

## Watching a session: peek vs watch

You do not always need to attach to a session to see what it is doing.

- **`/peek <session>`** grabs the current contents of the pane once and displays it as a still frame. Good for "what is on the screen right now?"
- **`/watch <session>`** keeps polling the pane about once a second, so you get a live view without taking over the terminal. The header shows a `LIVE` indicator. Press **Esc** to stop and return to the dashboard.

In both views, the **Up** and **Down** arrow keys scroll the captured text, and the view auto-scrolls to the bottom so you always land on the newest output. In the headless CLI, the same operations are `--peek <session>` and `--watch <session>`, which print the pane contents to standard output (tmux captures the last screenful plus its recent scrollback).

> **Peek vs attach.** Peeking and watching are read-only — they never disturb the session. `/resume` (and `/enter` when it creates) *attaches*, which gives you full interactive control but occupies the terminal until you detach.

---

## Sending input without attaching

`/send <session> <command>` types a line into the session's pane and presses Enter. It is the quickest way to answer a prompt or run a command in a running agent without giving up your own terminal.

```bash
/send fix-login /profile work
```

The headless equivalent is `scorpiox-tmux --send <session> <message>`, where everything after the session name is joined into one line and sent literally — no shell interpretation on the way in, Enter appended at the end.

---

## Starting, restarting, and stopping

- **`/new`** starts a session. If the name is already taken it tells you and does nothing — use `/restart` to replace it, or `/enter` if you just want "get me to a running session with this name."
- **`/enter`** is the "just get me in" command: it creates the session if needed and attaches. This is usually the command you want when you are working interactively.
- **`/restart`** kills and recreates, in one step — the way to recover a session that has wedged, optionally switching mode or build at the same time.
- **`/kill`** tears the session down. It removes the session's record (the small file that maps the session name to its directory), but it **never deletes your checkout or the worktree tree** — files on disk are always left alone.

The dashboard refreshes its list automatically about every five seconds, and `/list` forces a refresh on demand.

---

## Discovering projects

`/projects` (and `scorpiox-tmux --projects`) lists the project directories `scorpiox-tmux` can start sessions in, read from the configured **project base paths**. The list also powers the autocomplete you get on the project argument of `/new` and `/enter`, so you can type the first few letters of a repo and hit **Tab**.

On Unix the base path defaults to `/codebases` (set `TMUX_REMOTE_BASE` to point elsewhere, or to a `:`-separated list of paths). On Windows, `TMUX_REMOTE_BASE` is **required** — there is no auto-detect — and is a `;`-separated list, e.g. `D:\codebases`.

---

## Where and how a session runs

A session is a SCORPIOX CODE agent inside an isolated environment. Three independent choices determine exactly *where* and *how* it runs:

### 1. What it runs in — the launch mode

| Mode | Runs in | Typical use |
|---|---|---|
| **`unshare`** | A `scorpiox-unshare` sandbox built from a **distro image** | The default on Linux. Light, isolated, fast to start. |
| **`podman`** | A Podman container | Where you want standard containers and your own image. |
| **`wsl`** | A `scorpiox-wsl` sandbox | Windows, when WSL is installed. |
| **`native`** | Nothing — the agent runs directly on the machine | The **only** mode that works on a plain Windows or macOS box with no container runtime. |

The default launch mode comes from `TMUX_LAUNCH_MODE`. On plain Windows and macOS the default is forced to `native` (there is no `scorpiox-unshare` there), unless you explicitly ask for a container.

### 2. Which image — the distro (for `unshare`)

In `unshare` mode the environment is built from a **distro image**. The two you are most likely to touch:

- **`agentcore-latest`** — headless, the default. No desktop.
- **`desktopx11-latest`** — adds a full desktop (Xvfb, XFCE, VNC, Chrome) for when the agent needs a GUI.

Pick the default image for a machine, or override it per session:

```bash
/new myapp desktopx11-latest          # one session, with a desktop
# or make it the machine default in scorpiox-env.txt:
TMUX_DEFAULT_DISTRO=desktopx11-latest
```

Any image name is passed through exactly as you typed it, and a path to a local rootfs archive works too.

### 3. Where it runs — local or remote

`TMUX_MODE` decides whether sessions run **on this machine** (`local`) or **on a remote host over SSH** (`remote`, using `TMUX_REMOTE_HOST`). Everything else — the dashboard, the commands, the list — works the same either way; you are just driving the agents on the other box. Worktree creation, project discovery, and existence checks all happen over SSH in remote mode.

> **Under the hood:** sessions are tracked through a pluggable backend — the classic `tmux` command line, or SCORPIOX CODE's own multiplexer (`sxmux`). `TMUX_BACKEND` selects it, or leave it to auto-detect. Windows always uses `sxmux` because there is no `tmux` binary. You do not need to think about this; it is only why the tool is called `scorpiox-tmux` but also works where tmux does not exist.

### Per-session model and profile

Two more per-session switches, both optional:

- **Model** — the CLI flags `-m` / `--model opus|sonnet|haiku` (on `--new`, `--restart`, and `--enter`) set the model for that one session.
- **Profile** — the third `:` segment of the target, `project:session:profile`, runs just that session under a named profile. The name is validated up front — an unknown profile is refused rather than silently falling back to your default. This is the same profile system as the rest of SCORPIOX CODE; see [Configuration and Profiles](scorpiox-env.md) for how profiles cascade and how `/use` vs `/profile` differ.

---

## The headless CLI

Every dashboard operation has a CLI form. The most common:

```bash
scorpiox-tmux --list [--output-json]              # list sessions
scorpiox-tmux --projects                          # list startable projects
scorpiox-tmux --new <project|/dir> [--mode M] [--branch B] [--name S] [-m MODEL]
scorpiox-tmux --enter <project> [--mode M] [--branch B] [--name S]
scorpiox-tmux --resume <session>                  # attach
scorpiox-tmux --restart <session> [--mode M] [--branch B]
scorpiox-tmux --kill <session>
scorpiox-tmux --peek <session> [--output-json]    # one-shot screen capture
scorpiox-tmux --watch <session> [--output-json]   # full pane capture
scorpiox-tmux --send <session> <message>
scorpiox-tmux --help
```

`--output-json` on `--list`, `--peek`, and `--watch` gives machine-readable output. For `--list`, each session carries `name`, `created` (Unix timestamp), `attached`, and — whenever the session's directory is known — `worktree` (the session's working directory, for every kind of session), `kind` (`worktree`, `direct`, or `main`), and `profile` when one is active. A missing session on `--peek`/`--watch` JSON answers `{"error":"session not found",...}`.

### Session naming from the CLI

```bash
scorpiox-tmux --new myapp --name fix-login     # project + custom session name
scorpiox-tmux --new /tmp/scratch               # any folder; session named "scratch"
scorpiox-tmux --new ~/work --name w1
scorpiox-tmux --new D:\work\app                # Windows path, used as-is
```

A direct directory (anything starting with `/`, `~`, `./`, or a drive letter) is used exactly as it is — no worktree is created, even if the folder is not a git repo. The session is still recorded so `--list` and the bot API can find where it lives.

### Running a one-off command: `--cmd`

For automation, `--new` can run an arbitrary command instead of launching the interactive agent, and report back when it finishes:

```bash
scorpiox-tmux --new work --name job \
  --cmd "make build" \
  --env CI=true \
  --on-exit keep \
  --harness codex \
  --wait --timeout 600 --output-json
```

- `--cmd "<command>"` — run this command in the session instead of the interactive agent.
- `--env KEY=VALUE` — set an environment variable (repeatable).
- `--on-exit keep|kill|banner` — what happens when the command finishes: leave the session alive at a shell prompt (default), auto-kill it, or print a finish banner (with the command's exit code and a UTC timestamp) before dropping to a shell.
- `--harness NAME` — tag the session (for example `codex`) so tooling can tell what kind of agent is running in it; the tag is cleared when the command exits. Names are 1–32 characters of letters, digits, `_`, or `-`.
- `--wait` — block until the command finishes and print its exit code (times out with code 124).
- `--timeout <seconds>` — maximum runtime; with `--wait` it stops waiting, without `--wait` it leaves a background watchdog behind that kills the session when the time is up.
- `--output-json` — machine-readable result, including the exit code and whether it timed out.

A `--cmd` session whose command starts with `cd <dir>` records that directory, so bot-API lookups by session name resolve for it the same way they do for any other session.

This is the surface scripts and other agents use to drive a worker session and get a structured answer back.

---

## Keys in the dashboard

| Key | Effect |
|---|---|
| **Tab** | Show / accept autocomplete (commands, projects, modes, session names). |
| **Up / Down** | Move through the autocomplete popup; or scroll the pane in peek/watch. |
| **Enter** | Run the command you typed. |
| **Left / Right / Home / End** | Move the cursor in the input. |
| **Backspace / Delete** | Edit the input. |
| **Esc** | Dismiss the autocomplete popup; or leave peek/watch and return to the dashboard. |
| **Ctrl-C** | Quit `scorpiox-tmux`. |

The input bar hints that you can start anything with `/` — and the autocomplete will fill in the rest.

---

## Configuration

`scorpiox-tmux` reads its settings from the same `scorpiox-env.txt` cascade as the rest of SCORPIOX CODE (see [Configuration and Profiles](scorpiox-env.md) for tiers, precedence, and profiles). The keys that matter for it all start with `TMUX_`:

| Key | Meaning |
|---|---|
| `TMUX_MODE` | `local` or `remote` — run sessions on this machine or over SSH. |
| `TMUX_REMOTE_HOST` | The SSH host, when `TMUX_MODE=remote`. |
| `TMUX_REMOTE_BASE` | Where projects live. `:`-separated on Unix, `;`-separated on Windows. Defaults to `/codebases` on Unix; **required** on Windows. |
| `TMUX_WORKTREE_BASE` | Where per-session git worktrees are created. Default: in a `.worktrees` folder beside the repo. |
| `TMUX_BACKEND` | `tmux` or `sxmux` (or auto-detect). |
| `TMUX_LAUNCH_MODE` | Default launch mode: `unshare`, `podman`, `wsl`, or `native`. |
| `TMUX_DEFAULT_DISTRO` | Default `unshare` image. `agentcore-latest` (headless) or `desktopx11-latest` (desktop). |
| `TMUX_PODMAN_*` | Registry, image, TLS, volume mount, and extra args for `podman` sessions. |
| `TMUX_UNSHARE_*` | Extra args, network mode (`host` to share host networking), and volume mount for `unshare` sessions. |
| `TMUX_BIND_HOST_BINS` | Bind the host's SCORPIOX CODE binaries into the session instead of re-downloading (default on). |
| `TMUX_BIND_USER_CONFIG` | Bind your host agent config into the session read-only (default on). |
| `TMUX_BIND_TOOLS` + `TMUX_TOOLS_PATH` | Expose a host tool directory inside the session at `/mnt/apps` (off by default). |
| `TMUX_GUI_*` | Desktop/VNC publishing for `desktopx11-latest` sessions (off by default). |
| `SXMUX_RUNTIME_DIR` | Where `sxmux` session state lives. Must be machine-wide if a service also needs to see the sessions. |

A minimal `scorpiox-env.txt` for a local machine:

```ini
TMUX_MODE=local
TMUX_LAUNCH_MODE=unshare
TMUX_DEFAULT_DISTRO=agentcore-latest
```

Everything else falls back to safe defaults. Check what a machine is actually using with `scorpiox-config --json`.

### Windows notes

- `TMUX_REMOTE_BASE` is mandatory. Set it in `%LOCALAPPDATA%\scorpiox\scorpiox-env.txt` (e.g. `TMUX_REMOTE_BASE=D:\Workspace`) or with `setx TMUX_REMOTE_BASE D:\Workspace`, then relaunch. Without it, `--projects` is empty and project lookups fail.
- Set `SXMUX_RUNTIME_DIR` to a **machine-wide** directory such as `C:\ProgramData\sxmux` if a Windows service (which has no user profile) must see sessions created from your desktop logon — the default temp location is per-logon, so the two halves would otherwise not see the same sessions.
- `native` is the launch mode you can rely on there; the agent binary is located by absolute path so it works even when the session's shell has no user PATH.

---

## Desktop sessions (optional)

If you pick a desktop image for a session — `/new myapp desktopx11-latest` — the agent gets a full graphical desktop. Two settings control the extras:

- `TMUX_GUI_FOREGROUND=1` (default) keeps the agent in the foreground of the pane so its output actually appears; leave it on.
- `TMUX_GUI_VNC_PORT=<port>` publishes the desktop's VNC port so you can watch it with a VNC viewer. The first free port at or above the one you set is used, so parallel sessions do not collide. `TMUX_GUI_VNC_BIND=127.0.0.1` keeps the desktop on loopback only — strongly recommended, because these images run VNC with no password; reach it from elsewhere with an SSH tunnel (`ssh -L 6901:127.0.0.1:6901 user@host`).

---

## A typical workflow

```bash
scorpiox-tmux                          # open Mission Control
/new myapp:fix-login                   # start a session on a private worktree
/watch fix-login                       # watch it work (Esc to stop)
/send fix-login /profile work          # nudge it without attaching
/enter myapp:fix-login                 # ...or just jump in when it needs you
/restart fix-login                     # ...and recover it if it wedges
/kill fix-login                        # clean up when it is done
```

The headless versions of the same steps are `--new`, `--watch`, `--send`, `--enter`, `--restart`, and `--kill`.

---

## Gotchas

- **`/new` refuses duplicate names.** If a session with that name exists you get an error, not a silent replacement. Use `/restart` to replace it, or `/enter` to simply get to it.
- **`/resume` and `/enter` take over the terminal.** They attach to the session; you are back in the dashboard only when you detach. Use `/peek` or `/watch` if you just want to look.
- **`/kill` never deletes files.** It removes the session record and kills the process; the checkout and any worktree stay on disk for you to reuse or clean up yourself.
- **`@branch` is not a git branch.** It selects the SCORPIOX CODE build deployed into the session.
- **The model flag is `-m` / `--model` on the CLI**, and only on `--new`, `--restart`, and `--enter` — not on `--resume`, which just attaches to what is already running.
- **`TMUX_REMOTE_BASE` is mandatory on Windows.** Without it, `--projects` is empty and project lookups fail; there is no auto-detect there.
- **`peek` and `watch` are read-only.** They capture the pane; they never send anything. Only `/send` (and `--send`) write to a session.
- **Sessions survive service restarts on Linux.** When the first session of a batch is started from inside a system service, the session host is deliberately placed outside that service's control group, so restarting the service does not kill every running agent. Set `SCORPIOX_TMUX_NO_CGROUP_ESCAPE=1` if you want the old behavior.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Remote Agent Control and Fleet Management with SCORPIOX BOT](scorpiox-bot.md)
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
