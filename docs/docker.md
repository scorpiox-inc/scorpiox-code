# Run SCORPIOX BOT in Docker

You have a machine that should host a **SCORPIOX BOT** node — a spare laptop, a VPS, a homelab box, a CI worker — and you want it online on the SCORPIOX BOT web with as little on that machine as possible. **Docker** gives you exactly that: one official image, one run command, and a node that survives reboots, upgrades, and restarts.

Everything this page needs is two named volumes and a single `docker run`. The image does the onboarding for you — login, node name, node password, starter workspace — the first time you start it, and quietly reconnects on every start after that.

Docs for SCORPIOX CODE @ `ade108f`.

> **The whole idea in one line:** `docker run` a `scorpioxinc/bot` image with a data volume and a codebases volume, answer three prompts the first time, and your machine shows up in the SCORPIOX BOT web as a node you can drive from anywhere.

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

The official image is **`scorpioxinc/bot`** on Docker Hub. It bundles `scorpiox-bot` (the node supervisor) plus a minimal Linux base with `git`, `curl`, and CA certificates — enough for the node and its onboarding.

| Tag | What it is | Use it for |
|-----|------------|------------|
| `edge` | Newest build on a temporary glibc base (Debian). **This is the working image today.** | Production and day-to-day running |
| `<date-version>`, e.g. `2026-10-13-0754` | Pinned release, dated | Rollback and reproducibility |
| `latest` | Newest release — **currently a preview** (Alpine base) | Preview only; it is being blocked by a glibc dependency until static musl bot builds land |

The tags change as releases land — `latest` will become the working tag when the Alpine image is unblocked. Until then, use `edge`.

---

## The one-line run command

```bash
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v scorpiox-code:/codebases \
  scorpioxinc/bot:edge
```

Every flag does exactly one job:

| Flag | What it does |
|------|--------------|
| `-it` | Interactive terminal. Required for the first run, because you type your account credentials and node password at prompts. |
| `--restart unless-stopped` | If the container or the host reboots, Docker brings the node back up on its own. |
| `-v scorpiox-data:/root` | Named volume for **everything the node owns**: your login credentials, the node identity and password, onboarding state, and logs. This is what makes the node persistent. |
| `-v scorpiox-code:/codebases` | Named volume for your **project folders** — the code the agents inside the node will work on. |

### What each folder holds

| Folder | Mount | Holds |
|--------|-------|-------|
| `/root` (the data volume) | `scorpiox-data` | `~/.scorpiox/` — login token, node settings (`bot-node.env`), onboarding state, logs. `~/.claude/scorpiox-env.txt` — the node's internal session settings, written by the image at first start. |
| `/codebases` (the code volume) | `scorpiox-code` | Your projects. The image's working directory is `/codebases`, and each subfolder is one project a session can work on. |

### Mounting a folder you already have

Named volumes are the simplest option, but if your projects already live in a directory on the host, bind-mount it instead of a named volume:

```bash
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v /home/<you>/projects:/codebases \
  scorpioxinc/bot:edge
```

Now `/codebases` inside the container is literally your `~/projects` folder — same files, both sides, no copying.

### Why no `-p` port publishing

The node talks to the mesh hub over a reverse tunnel — it dials out, nothing dials in. There is no port to publish and no firewall hole to make.

---

## First run: the onboarding walkthrough

The first time you start the container, the image walks you through onboarding in order. Each step is skipped automatically the moment its state file exists, so you only ever answer these once.

### 1. Starter workspace prompt

The image looks at `/codebases` first:

- **If it finds project folders already there**, it lists them and asks:
  `Also create a starter workspace 'bot-demo'? [y/N]` — the default is **no**.
- **If `/codebases` is empty**, it asks:
  `Create a starter workspace 'bot-demo'? [Y/n]` — the default is **yes**.

A *starter workspace* is a small, git-initialized folder `bot-demo` under `/codebases` with an `AGENTS.md` that tells the agents who they are. It is a safe default for a brand-new node: a place to poke at, without touching your real projects.

Your answer is remembered in the data volume, so it is asked **only once** — never again on restart, upgrade, or re-image. To change your mind later, delete the `demo=` line from the onboarding state file inside the data volume and re-run.

### 2. Login to your Scorpio+ account

If the node is not logged in yet, you are prompted for your account email and password (typed, not echoed). The credentials are stored in the **data volume** — `~/.scorpiox/auth.json` inside the container — so a restart, an upgrade, or even a fresh image never asks you to log in again.

> **Heads-up for non-interactive runs:** if stdin is not a terminal (CI, scripts, `docker exec` without `-it`), the image cannot prompt you. If it finds you are not logged in, it stops and tells you to re-run once with `-it`. So: do your first run interactively.

### 3. Node name and node password

The node gets a name for the SCORPIOX BOT web dashboard. The default is your user id with `-docker` appended (for example `12345-docker`); you can set your own with `SCORPIOX_NODE_NAME=my-box` in the run command.

Then you are prompted twice for a **node password**, echo off. This is the password you type when you open this node from the SCORPIOX BOT web — it is stored in the data volume, never in the image, never in the environment.

### 4. Connect and stay foreground

The image ends by connecting the node to the mesh hub and staying in the foreground. You should see the connection confirm, and your machine appear in the SCORPIOX BOT web as a node. `Ctrl+C` stops the container.

> **What you do next is remote, not local.** Once the node is online, you drive it from the SCORPIOX BOT web — sessions, prompts, live terminal, fleet view. See [Remote Agent Control & Fleet Management with SCORPIOX BOT](scorpiox-bot.md). Sessions themselves are managed with [scorpiox-tmux](scorpiox-tmux.md).

---

## Running in the background, restarting, and service life

The `--restart unless-stopped` flag is what turns the first interactive run into a service:

| You want to | Do this |
|-------------|---------|
| Stop the node (and keep it stopped) | `docker stop <container>` — restart policy respects this; it will not come back on its own. |
| Start it again | `docker start <container>` — login and node state are in the data volume; no prompts. |
| Watch it live | `docker logs -f <container>` |
| Re-run the onboarding (e.g. new account) | Remove the data volume: `docker volume rm scorpiox-data` — everything onboarding-related is re-prompted on the next start. |

The container's lifecycle is entirely Docker's: no systemd unit, no `tmux` pane to keep alive, no login on the host to maintain.

---

## Upgrading the node

The node's state is in the **volumes**, not in the image — so an upgrade is: pull the new image, recreate the container, same volumes. Login, node identity, and your projects all survive.

```bash
# 1. Pull the new image
docker pull scorpioxinc/bot:edge

# 2. Stop and remove the old container (volumes are not touched)
docker stop <container>
docker rm <container>

# 3. Start the new image with the same two volumes and flags
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v scorpiox-code:/codebases \
  scorpioxinc/bot:edge
```

No onboarding prompts — the state files are all already there in the data volume. The new image just connects the same node.

If you want a pinned tag for rollback (e.g. `2026-10-13-0754`), run the same three steps with that tag instead of `edge`.

---

## Where your data lives

Everything you care about is in the two named volumes. Nothing important lives in the image or in the container's writable layer:

| Data | Location (in-container) | Volume |
|------|-------------------------|--------|
| Login token (`auth.json`) | `~/.scorpiox/` | `scorpiox-data` |
| Node name and node password (`bot-node.env`) | `~/.scorpiox/` | `scorpiox-data` |
| Onboarding state (starter workspace answer) | `~/.scorpiox/` | `scorpiox-data` |
| Node logs | `~/.scorpiox/logs/` | `scorpiox-data` |
| Node session settings (`scorpiox-env.txt`) | `~/.claude/` | `scorpiox-data` |
| Your projects | `/codebases/` | `scorpiox-code` |
| Starter workspace (`bot-demo`, if you accepted it) | `/codebases/bot-demo` | `scorpiox-code` |

Inspect the data volume on the host with `docker volume inspect scorpiox-data`, or peek at a path inside a running container with `docker exec <container> ls -la /root/.scorpiox`.

---

## Knobs the image honours

Secrets (login, node password) are always typed at prompts — never passed as environment variables. A handful of *non-secret* settings are read from the environment if you want to set them up front:

| Variable | Effect |
|----------|--------|
| `SCORPIOX_NODE_NAME` | Node name shown in the SCORPIOX BOT web. Default: `<your user id>-docker`. |
| `SCORPIOX_CREATE_DEMO` | `yes` or `no` — answers the starter workspace prompt non-interactively. Useful in CI. |
| `SCORPIOX_WORKSPACE` | Starter workspace folder name under `/codebases`. Default: `bot-demo`. |

---

## A minimal day-to-day recipe

```bash
# First time, on the machine that will host the node
docker pull scorpioxinc/bot:edge
docker run -it --restart unless-stopped \
  -v scorpiox-data:/root \
  -v scorpiox-code:/codebases \
  scorpioxinc/bot:edge
# ... answer login, node name, node password, and the starter workspace prompt ...
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
