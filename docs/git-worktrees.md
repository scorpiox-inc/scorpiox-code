# Isolated Git Worktrees for Parallel Agent Sessions

You want to run several SCORPIOX CODE agents on the *same repository* at once — one per task, one per branch — but you do not want them fighting over the same files. Two agents sharing one working tree is how you end up with a half-written `main` overwritten by a feature branch, or a second agent's `.scorpiox/sessions` clobbering the first agent's, so that "which session is this?" becomes a guess.

SCORPIOX CODE's session manager, `scorpiox-tmux`, solves this for you automatically: **every *named* session on a git repository gets its own isolated checkout and its own branch.** You do not create the worktree, you do not pick a path, and you do not hand the agent a `git worktree add` command. You just name the session, and the isolation is there.

A worktree is git's native way of checking out the same repository into a second directory that shares the history but has its own working files and its own branch. That is exactly the unit of isolation an agent needs: a real, separate working directory that never touches the one the other agents are using.

Docs for SCORPIOX CODE @ `62c4a90`.

> **The whole idea in one line:** name a session, and SCORPIOX CODE gives it a private git worktree — its own directory, its own branch, its own session store — so parallel agents on one repo never collide, and a session can still find the sessions it needs from the main checkout later.

---

## When you get a worktree

A session gets its own worktree in exactly one case: **the session name differs from the project name.** The session name is the second word in `project:session` (the part after the colon), or the `--name` you pass on the command line.

```
/new myapp              session "myapp"  == project "myapp"  ->  main checkout, no worktree
/new myapp:fix-login    session "fix-login" != project "myapp" ->  own worktree + own branch
```

So:

- **`/new myapp`** (or `scorpiox-tmux --new myapp`) — the session is named after the project itself, which is the "one agent per repo" case. It runs in the project's main checkout. No worktree.
- **`/new myapp:fix-login`** (or `scorpiox-tmux --new myapp --name fix-login`) — a named session. SCORPIOX CODE creates a git worktree, checks it out on a fresh branch named `fix-login`, and runs the agent inside *that* directory. The main checkout stays exactly as it was.

This is the rule you can rely on: *named* sessions are isolated, the *default* session (named after the project) shares the main checkout. If you want isolation, give the session a name.

A session running in a worktree is marked everywhere you would look. In the dashboard it carries a `[worktree: myapp/fix-login]` label; `scorpiox-tmux --list` prints the same label; and the JSON form of `--list` reports the session's working directory in `worktree` plus a `kind` of `worktree`.

---

## The three kinds of session

There are exactly three ways a session can land on disk, and it is useful to know which one you are in because the cleanup and the "where is my work?" questions differ:

| Kind | How you get it | Where the agent runs |
|---|---|---|
| **own worktree** | `/new myapp:fix-login` / `--new myapp --name fix-login` | a private worktree with its own branch |
| **direct folder** | `/new /tmp/scratch:s1` / `--new /tmp/scratch --name s1` | that folder, used as-is |
| **main checkout** | `/new myapp` (no separate name), or `--allow-main` | the project's main checkout |

### 1. Own worktree (the isolated case)

The common case. A named session on a git repo gets a worktree.

### 2. Direct folder (no worktree, by design)

If you point a session at a folder with an absolute path — anything starting with `/`, `~`, `./`, or a Windows drive letter — SCORPIOX CODE uses that folder exactly as it is. **No worktree is created, even if the folder happens to be a git repository.** A plain folder was always meant to be run in place, so that is the intended behaviour, not a bug. The session is still recorded so `--list` and the bot API can find where it lives.

```
/new /tmp/scratch:s1        # direct folder, no worktree
scorpiox-tmux --new /tmp/scratch --name s1
```

### 3. Main checkout (the shared case)

Two ways a session runs in the main checkout rather than a worktree:

- **The default session** — `/new myapp`, where the session name equals the project name. This is the "single agent on this repo" case, and running in place is what you want.
- **`--allow-main`** — an explicit opt-in you pass when you *know* the worktree cannot be created and you are willing to accept the shared-checkout risk (covered below).

---

## The worktree layout

Where the worktree lands is deterministic, and you can see it:

- **Default location:** a `.worktrees` folder *beside the repository*. For a repo at `/codebases/myapp`, a session `fix-login` becomes:

  ```
  /codebases/myapp/.worktrees/myapp-fix-login
  ```

- **The branch** is named after the session (`fix-login`), so each worktree is on its own branch and the worktrees never fight over a branch checkout.

- **A `.sessions` index** beside the worktrees records the mapping from each session name to its project path, its working directory, its host, and its kind. This is how `--list`, the bot API, and `--kill` all know where a session lives. It is bookkeeping for the tooling, not something you normally edit by hand.

- **Inside the worktree** the agent writes its own `.scorpiox/` tree — sessions, logs, traffic, config — into the worktree, so each session's state lives next to its own files and not in a shared place the other agents can touch.

You can redirect where worktrees are created. Set `TMUX_WORKTREE_BASE` in `scorpiox-env.txt` to a directory and every worktree lands at `<TMUX_WORKTREE_BASE>/<project>-<session>` instead. This is the knob for pointing worktrees at a dedicated volume, or at a path that is convenient for your machine. An explicit `TMUX_WORKTREE_BASE` always wins over the default "beside the repo" location. See [Configuration and Profiles](scorpiox-env.md) for the cascade and where to set it.

---

## What happens on creation

When a named session on a git repo is started, SCORPIOX CODE:

1. Updates the repo (fetch and rebase) so the new branch starts from the latest `main`. Failures here are non-fatal — branching from a slightly stale `main` is better than refusing to start.
2. Creates the worktree on a fresh branch (`git worktree add <path> -b <session>`).
3. If the branch already exists from a previous session of the same name, it **reuses that branch** in the new worktree rather than failing. This means restarting a session with a name you used before is not an error — it re-attaches to the existing branch.
4. Records the session-to-directory mapping so the tooling can find it.

Creation is idempotent in the parts you feel: if the worktree already exists and is populated, it is reused rather than recreated, so restarting a session does not clobber a checkout you have been working in.

---

## Failing loudly, and the explicit opt-out

The one failure that matters most is the quiet one: a git repo whose worktree *should* be created but cannot be. In that case two named sessions would silently land in the same main checkout, sharing one working tree and one `.scorpiox/sessions`, and the bot API would resolve both names to whichever agent was most active. That is a real collision, and it is the footgun SCORPIOX CODE refuses to hide.

So the default is **fail loudly**: if the project is a git repository but its worktree cannot be created, the session does not start. You get a clear error naming the session and the project, telling you it would collide with any other session on that repo, and pointing you at the fix:

```
Worktree failed for 'fix-login' on 'myapp': <reason>
  Refusing to start on the main checkout - it would collide with
  any other session on this repo (shared working tree and shared
  .scorpiox/sessions).
  Fix git worktree, or pass --allow-main to run on main anyway.
```

**`--allow-main`** is the explicit opt-out. Pass it (on `--new` / `--enter`) when you have deliberately decided the shared main checkout is acceptable for this one session:

```
scorpiox-tmux --new myapp --name fix-login --allow-main
```

With `--allow-main` on a git repo, the session runs on the main checkout and tells you it is doing so. Without it, the session aborts rather than collide.

The rule is asymmetric, on purpose:

- A **plain folder** (not a git repo, or no `git` on the machine) always falls back to running in place, quietly. Running in place was always the intent, so there is nothing to fail about.
- A **git repo whose worktree failed** does not. That is the collision case, and it aborts unless you opt in with `--allow-main`.

---

## Remote worktrees

When your sessions run on a remote host (`TMUX_MODE=remote` with `TMUX_REMOTE_HOST` set), the same worktree logic runs *over SSH*. The worktree is created on the remote machine, beside the repo there, and the session mapping records which host it lives on. Nothing changes for you: `/new myapp:fix-login` behaves the same way whether the agent is on this box or on the other end of the network, and the worktree is on the host where the code lives.

---

## Finding sessions again later

This is the part that makes worktrees safe to use as the default. A worktree is a disposable directory — you can delete it at any time — and a session running in one is not *inside* the main checkout, so a naive "list the sessions in the main checkout" would miss it. SCORPIOX CODE handles this in two directions:

- **Dual-write of session state.** While a session runs inside its worktree, its session data (conversation, logs, traffic, stats, and so on) is written to **both** the worktree's `.scorpiox/sessions/` and the main checkout's `.scorpiox/sessions/`. So the main checkout always has a copy of every session's state, even the ones running in worktrees.
- **`/resume` sees both stores.** When you run `/resume` from a worktree, the session list is the union of the worktree's own store **and** the main checkout's store, with the worktree's copy winning when the same session appears in both. From the main checkout you see your own sessions plus all the mirrored worktree sessions. Either way, a session started in a worktree can be found, reattached, and resumed later — you do not have to remember which directory it was in.

In practice: start a named session, let it run in its worktree, walk away, and come back to it with `/resume <session>` (or `--resume <session>`) from wherever you are. It is found.

> **Why this matters.** Without the main-checkout mirror, a worktree session's state would live only in a throwaway directory, and the moment that directory was gone the session would be unfindable. The mirror is what makes "parallel, isolated, and still resumable" all hold at once.

---

## Killing a session vs. deleting a worktree

These are two different things, and the tooling keeps them separate:

- **`/kill <session>`** (or `--kill <session>`) removes the session's record and stops its process. It **never deletes the worktree or any files on disk.** The checkout and the branch are left for you to inspect, reuse, or clean up yourself. This is the "stop this agent, keep my work" command.
- **`git worktree remove <path>`** (you run it yourself) is the command that actually deletes a worktree directory. Run it when you are done with that session's checkout entirely. The branch survives the removal; only the working directory goes.

So a worktree is a working directory you can throw away freely, while the session's state is preserved in the main checkout regardless. You can delete the worktree and still resume the session from the main checkout.

---

## The day-to-day workflow

```
# one agent per repo, shared main checkout (the simple case)
/new myapp

# two agents on the same repo, fully isolated from each other
/new myapp:fix-login
/new myapp:perf-scan

# a named session that must reuse an existing branch from a past run
scorpiox-tmux --new myapp --name fix-login        # branch reused if it already exists

# a named session you are willing to run on main (worktree creation will collide)
scorpiox-tmux --new myapp --name hotfix --allow-main

# a scratch folder, no repo, no worktree, used as-is
/new /tmp/scratch:s1
```

Check which session is in which worktree at any time with `scorpiox-tmux --list` — each worktree session carries its `[worktree: project/session]` label.

---

## Cross-references

- [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md) — the full session lifecycle: launching, watching, sending input, restarting, killing, and the dashboard. This page covers only the git worktree isolation layer that `scorpiox-tmux` provides for named sessions.
- [Configuration and Profiles](scorpiox-env.md) — the `scorpiox-env.txt` cascade, where `TMUX_WORKTREE_BASE` and the other `TMUX_` keys live, and how profiles apply to per-session settings.

---

*Docs for SCORPIOX CODE @ `62c4a90`. Session lifecycle and command reference are covered in [Managing Agent Sessions](scorpiox-tmux.md).*
