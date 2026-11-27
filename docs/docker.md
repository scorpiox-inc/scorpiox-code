# Run SCORPIOX BOT in Docker

You have a machine that should host a **SCORPIOX BOT** node — a spare laptop, a VPS, a homelab box, a CI worker — and you want it online in the SCORPIOX BOT web with as little on that machine as possible. **Docker** gives you exactly that: one official image, one run command, and a node that survives reboots, upgrades, and restarts.

Everything this page needs is two named volumes and a single `docker run`. The image does the onboarding for you — starter workspace, login, node name, node password — the first time you start it, and quietly reconnects on every start after that.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** `docker run` a `scorpioxinc/bot` image with a data volume and a codebases volume, answer the prompts the first time, and your machine shows up in the SCORPIOX BOT web as a node you can drive from anywhere.

---

## Before you start: install Docker

This guide assumes you can run `docker run`. If you cannot, install Docker Engine or Docker Desktop and come back:

| Platform | Official install docs |
|----------|-----------------------|
| Linux | https://docs.docker.com/engine/install/ |
| macOS | https://docs.docker.com/desktop/install/mac-install/ |
| Windows | https://docs.docker.com/desktop/install/windows-install/ |

Docker itself is out of scope for this page — no daemons to babysit, no VMs to configure, once it works.

---

## Which image, and which tag

The official image is **`scorpioxinc/bot`** on Docker Hub. It bundles the SCORPIOX CODE core suite (`scorpiox`, `scorpiox-bot`, and the bot API and web pieces) plus a minimal Linux base with `bash`, `git`, `curl`, CA certificates, time zones, and process tools — enough for the node, its onboarding, and the agent workspaces it hosts.

| Tag | What it is | Use it for |
|-----|------------|------------|
| `latest` | Alpine (musl) image. Every binary in it is static or verified to load at build time — the build fails if any executable would not run. **This is the working image today; use this tag.** | Production and day-to-day running |
| `edge` | Newest untested build on a temporary Debian (glibc) base. It works, and is kept as a fallback. | Fallback only |
| `<scorpiox version>`, e.g. `2026-10-13-0754` | Pinned release, dated | Rollback and reproducibility |
| `arch`, `agentcore`, `desktop` | Placeholder images, not the bot node | Nothing yet |

Two honest caveats about `latest` today:

- **It is x86_64-only for now.** The arm64 build is blocked until the bot API and web pieces publish arm64 builds — until then, run the image on x86_64 hardware (or an x86_64 emulation layer).
- The in-container `/help` topic lookup is not available in this image (its binary has no Linux build for the musl base); everything else works.

---

## The one-line run command

```bash
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v scorpiox-code:/codebases \
  scorpioxinc/bot:latest
```

Every flag does exactly one job:

| Flag | What it does |
|------|--------------|
| `-it` | Interactive terminal. Required for the first run, because you answer the onboarding prompts — login and node password are typed, not passed as variables. |
| `--restart unless-stopped` | If the container or the host reboots, Docker brings the node back up on its own. |
| `-v scorpiox-data:/root` | Named volume for **everything the node owns**: your login credentials, the node identity and password, onboarding state, and logs. This is what makes the node persistent. |
| `-v scorpiox-code:/codebases` | Named volume for your **project folders** — the code the agents inside the node will work on. |

### What each folder holds

| Folder | Mount | Holds |
|--------|-------|-------|
| `/root` (the data volume) | `scorpiox-data` | `~/.scorpiox/` — login token (`auth.json`), node settings (`bot-node.env`), onboarding state (`onboard.state`), logs. `~/.claude/scorpiox-env.txt` — the node's internal session settings, written by the image at first start. |
| `/codebases` (the code volume) | `scorpiox-code` | Your projects. The image's working directory is `/codebases`, and each subfolder is one project a session can work on. |

### Mounting a folder you already have

Named volumes are the simplest option, but if your projects already live in a directory on the host, bind-mount it instead of a named volume:

```bash
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v /home/<you>/projects:/codebases \
  scorpioxinc/bot:latest
```

Now `/codebases` inside the container is literally your `~/projects` folder — same files, both sides, no copying.

### Why no `-p` port publishing

The node joins the mesh hub over a reverse tunnel — it dials out, nothing dials in. There is no port to publish and no firewall hole to make.

---

## First run: the onboarding walkthrough

The first time you start the container, the image walks you through onboarding in order. Every step is skipped automatically the moment its state file exists in the data volume, so you only ever answer these once.

### 1. Starter workspace prompt

The image looks at `/codebases` first:

- **If it finds project folders already there**, it lists them and asks:
  `Also create a starter workspace 'bot-demo'? [y/N]` — the default is **no**.
- **If `/codebases` is empty**, it asks:
  `Create a starter workspace 'bot-demo'? [Y/n]` — the default is **yes**.

A *starter workspace* is a small, git-initialized folder `bot-demo` under `/codebases` with an `AGENTS.md` that tells the agents who they are. It is a safe default for a brand-new node: a place to poke at, without touching your real projects.

Your answer is remembered in the data volume, so it is asked **only once** — never again on restart, upgrade, or re-image. To change your mind later, delete the `demo=` line from `~/.scorpiox/onboard.state` inside the data volume and re-run.

### 2. Login to your Scorpio+ account

If the node is not logged in yet, you are prompted for your account email, then your password (echo off). The credentials are stored in the **data volume** — `~/.scorpiox/auth.json` inside the container — so a restart, an upgrade, or even a fresh image never asks you to log in again.

> **Heads-up for non-interactive runs:** if stdin is not a terminal (CI, scripts, `docker exec` without `-it`), the image cannot prompt you. If it finds you are not logged in, it stops and tells you to re-run once with `-it`. So: do your first run interactively.

### 3. Node name and node password

The node gets a name for the SCORPIOX BOT web dashboard. The default is your user id with `-docker` appended (for example `12345-docker`); you can set your own with `SCORPIOX_NODE_NAME=my-box` in the run command.

Then you are prompted twice for a **node password**, echo off. This is the password you type when you open this node from the SCORPIOX BOT web — it is stored in the data volume, never in the image, never in the environment.

### 4. Connect and stay foreground

The image ends by connecting the node to the mesh hub and staying in the foreground. Your machine appears in the SCORPIOX BOT web as a node. `Ctrl+C` stops the container.

> **What you do next is remote, not local.** Once the node is online, you drive it from the SCORPIOX BOT web — sessions, prompts, live terminal, fleet view. See [Remote Agent Control & Fleet Management with SCORPIOX BOT](scorpiox-bot.md). Sessions themselves are managed with [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md).

---

## Running in the background, restarting, and service life

The `--restart unless-stopped` flag is what turns the first interactive run into a service:

| You want to | Do this |
|-------------|---------|
| Stop the node (and keep it stopped) | `docker stop <container>` — the restart policy respects this; it will not come back on its own. |
| Start it again | `docker start <container>` — login and node state are in the data volume; no prompts. |
| Watch it live | `docker logs -f <container>` |
| Re-run the onboarding (e.g. a new account) | Remove the data volume: `docker volume rm scorpiox-data` — everything onboarding-related is re-prompted on the next start. |

The container's lifecycle is entirely Docker's: no systemd unit, no `tmux` pane to keep alive, no login on the host to maintain.

---

## Upgrading the node

The node's state is in the **volumes**, not in the image — so an upgrade is: pull the new image, recreate the container, same volumes. Login, node identity, and your projects all survive.

```bash
# 1. Pull the new image
docker pull scorpioxinc/bot:latest

# 2. Stop and remove the old container (volumes are not touched)
docker stop <container>
docker rm <container>

# 3. Start the new image with the same two volumes and flags
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v scorpiox-code:/codebases \
  scorpioxinc/bot:latest
```

No onboarding prompts — the state files are all already there in the data volume. The new image just connects the same node.

If you want a pinned tag for rollback (e.g. `2026-10-13-0754`), run the same three steps with that tag instead of `latest`.

---

## Where your data lives

Everything you care about is in the two named volumes. Nothing important lives in the image or in the container's writable layer:

| Data | Location (in-container) | Volume |
|------|-------------------------|--------|
| Login token (`auth.json`) | `~/.scorpiox/` | `scorpiox-data` |
| Node name and node password (`bot-node.env`) | `~/.scorpiox/` | `scorpiox-data` |
| Onboarding state (starter workspace answer, `onboard.state`) | `~/.scorpiox/` | `scorpiox-data` |
| Node logs | `~/.scorpiox/logs/` | `scorpiox-data` |
| Node session settings (`scorpiox-env.txt`) | `~/.claude/` | `scorpiox-data` |
| Your projects | `/codebases/` | `scorpiox-code` |
| Starter workspace (`bot-demo`, if you accepted it) | `/codebases/bot-demo` | `scorpiox-code` |

Inspect the data volume on the host with `docker volume inspect scorpiox-data`, or peek at a path inside a running container with `docker exec <container> ls -la /root/.scorpiox`.

---

## Knobs the image honours

Secrets (login, node password) are always typed at prompts — never passed as environment variables, never as arguments. A handful of *non-secret* settings are read from the environment if you want to set them up front:

| Variable | Effect |
|----------|--------|
| `SCORPIOX_NODE_NAME` | Node name shown in the SCORPIOX BOT web. Default: `<your user id>-docker`. |
| `SCORPIOX_CREATE_DEMO` | `yes` or `no` — answers the starter workspace prompt non-interactively. Useful in CI. |
| `SCORPIOX_WORKSPACE` | Starter workspace folder name under `/codebases`. Default: `bot-demo`. |

Passing a command to the image (`docker run scorpioxinc/bot:latest <command>`) runs that command inside the image instead of onboarding — this is what `scorpiox-unshare <image> <command>` uses to give an agent a disposable shell inside the same image. A bare login shell (`bash -l` or `bash -i`) is treated as interactive and still onboards.

---

## A minimal day-to-day recipe

```bash
# First time, on the machine that will host the node
docker pull scorpioxinc/bot:latest
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v scorpiox-code:/codebases \
  scorpioxinc/bot:latest
# ... answer the starter workspace prompt, log in, set the node name and node password ...
# ... open the SCORPIOX BOT web, find your node, drive it ...

# Any later day, after a reboot or an upgrade
docker start <container>          # and it is back, same node, same login
docker logs -f <container>        # to watch it
```

That is the whole surface. One image, two volumes, one command, and a node that lives in the SCORPIOX BOT web and survives everything short of deleting the data volume.

---

## See also

- [Remote Agent Control & Fleet Management with SCORPIOX BOT](scorpiox-bot.md) — the web dashboard, the headless API, and how you actually drive this node once it is online.
- [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md) — session lifecycle: list, attach, watch, steer, kill.
