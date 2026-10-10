# Native Linux Container Runtime for AI Agents

You want an AI coding agent running somewhere it cannot touch your real machine — but you do not want to pay for it with a hypervisor, a Docker Desktop license, or a daemon that has to be up and healthy before anything can start.

**`scorpiox-unshare`** is SCORPIOX CODE's native Linux container runtime. It is a single small binary written in pure C with zero external dependencies, and it builds isolation out of the kernel itself — Linux **user namespaces** — rather than out of a virtual machine, a background daemon, or a second operating system. Hand it an image and a command and it hands back a fast, rootless, isolated Linux environment that is ready in a fraction of a second and cleaned up the moment the command exits.

There is no daemon to install, no `dockerd` to keep running, no VM image to boot, and no network stack to configure. One binary, one command, one container.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** `scorpiox-unshare <image> <command>` runs any agent or command inside a real Linux user namespace — no Docker, no podman, no daemon, no VM — and it is the default environment under every SCORPIOX CODE agent session.

---

## The two ways to run it

`scorpiox-unshare` has one command shape and two ways to put it to work.

### 1. The SCORPIOX CODE path (recommended default)

This is the one you will actually use. The SCORPIOX CODE session manager, [`scorpiox-tmux`](scorpiox-tmux.md), builds every agent session on top of `scorpiox-unshare` by default. You never type the runtime's name: you start a session, and the runtime is already underneath it.

```bash
# Inside the scorpiox-tmux dashboard, at the > prompt:
/new myapp
```

That one command picks the default image, downloads and unpacks it once, mounts your project in, and runs the agent inside the namespace. The runtime is invisible plumbing — which is exactly the point. You can still name the mode or the image explicitly when you want to:

```bash
/new myapp unshare               # force unshare mode
/new myapp agentcore-latest      # unshare, with this image
/new myapp desktopx11-latest     # unshare, with a desktop
```

Everything about how that path behaves — which image, which mounts, which extra flags — is driven by a small set of `TMUX_` keys in `scorpiox-env.txt`, covered in [Configuration](#configuration-the-tmux_keys).

### 2. Direct: run any agent or command

`scorpiox-unshare` is not locked to SCORPIOX CODE. It is harness-agnostic: `<image> <command>` runs *any* command — any agent, any script, any shell — inside the container. That makes it a general-purpose rootless runtime you can reach for from anywhere.

```bash
scorpiox-unshare agentcore-latest                  # interactive shell
scorpiox-unshare agentcore-latest "uname -a"       # one-off command
scorpiox-unshare --bind ~/proj agentcore-latest    # your project at /workspace
```

The first form drops you into an interactive shell; the second runs the command and exits with its status. The image is resolved, cached, and mounted identically either way.

A third consumer ships in the box: the SCORPIOX CODE agent runner (`scorpiox-sdk --agent`) launches each agent through the same runtime, binding its task directory at `/workspace` and a build cache at `/persist`. You get the same isolation whether an agent is started by hand, by the session manager, or by automation.

---

## What the isolation actually is

Every container is a real kernel construct, not a chroot and not an emulation layer.

- **Mount namespace + `pivot_root`.** The image is mounted as the container's root filesystem, and the container cannot see your host mounts at all. The only host paths inside are the ones you explicitly mounted.
- **PID namespace.** The container has its own process tree and its own process numbering.
- **UTS namespace.** The container gets its own hostname, taken from the image name.
- **Network namespace (default).** By default the container gets its own network stack, wired to the outside through a small userspace network helper. See [Networking and ports](#networking-and-ports). `--net host` opts out.
- **User namespace.** This is the core trick, and the reason nothing here needs root. Your unprivileged host user is mapped to `root` *inside* the container — so you are the administrator of the container and an ordinary user on your own machine at the same time. If the host has subordinate UID/GID ranges configured (`/etc/subuid` and `/etc/subgid`), the container gets a full range of identities so services that need real group handling work; without them it falls back to a single-identity map. When you run as actual root on the host, the user namespace is skipped — there is nothing left to gain from it.
- **Filesystem layers.** On kernels that support overlayfs inside user namespaces (5.11 and newer), the read-only image is overlaid with a private writable layer, so the image itself is never modified. On older kernels it falls back to a bind mount of the image.
- **Syscall and capability trimming.** Inside the container, dangerous kernel entry points are blocked with a seccomp filter and memory-locking capability is dropped from the bounding set before anything runs.

Two consequences worth internalizing:

- **The container filesystem is ephemeral.** Writes land in the private layer and are deleted when the container exits. Anything you want to keep must be mounted in — see `--bind`, `-v`, and `--persist` below.
- **You are not root on the host.** The `root` you see inside the container maps back to your own user. That is the boundary the whole runtime is built around, and it is why there is no daemon and no permission prompt.

---

## Images

A container is built from an **image**: a `.tar` of a root filesystem, or an OCI image layout. `scorpiox-unshare` resolves the `<image>` argument in this order and passes any name through verbatim:

1. **A local path** — anything starting with `/` or `.` is treated as a path to a `.tar` on disk.
2. **A cached image** — if `<image>.tar` is already in the image cache, it is used as-is.
3. **A download** — otherwise it is fetched from the image base URL and cached for next time.

Two images cover almost everything you will do with SCORPIOX CODE:

| Image | What it is |
|-------|------------|
| **`agentcore-latest`** | The headless default. No desktop. This is what `/new myapp` uses out of the box. |
| **`desktopx11-latest`** | Adds a full desktop — Xvfb, XFCE, VNC, Chrome — for agents that need a GUI. |

A few things worth knowing about image handling:

- **The cache lives under `~/.scorpiox`.** Downloaded archives are kept in `~/.scorpiox/images/`, unpacked root filesystems in `~/.scorpiox/rootfs/<image>/`, and the throwaway writable layer in `~/.scorpiox/work/`. Set `SCORPIOX_HOME` (or pass `--home`) to move all of it.
- **First launch is slow, every launch after is fast.** Download and unpack happen once. After that, a boot reuses the cached root filesystem and reports its own timing — boot timing output is on by default, and `--no-perf` (or `SCORPIOX_PERF=0`) turns it off.
- **Updates are noticed, not automatic.** When a cached image is used, the runtime compares it against the remote copy and, if the sizes differ, prints an update notice. Apply it when you want to:

```bash
scorpiox-unshare --list                        # see what is available, with sizes and what is cached
scorpiox-unshare --update agentcore-latest     # re-download and re-extract one image
scorpiox-unshare --clean                      # remove all unpacked root filesystems and work dirs
```

- **OCI layouts just work.** If the archive is an OCI image layout rather than a flat root filesystem, the layers are unwrapped automatically into a flat root filesystem.
- **Own images are fine.** Any `.tar` root filesystem — yours or one built elsewhere — works by path: `scorpiox-unshare /path/to/my-rootfs.tar "ls /"`. There is no build step inside the runtime; you bring the filesystem, it runs it.

---

## Flags

| Flag | What it does |
|------|--------------|
| `--bind <dir>` | Bind-mount a host directory at `/workspace` (and start the shell there). |
| `--persist <dir>` | Bind-mount a host directory at `/persist`, for anything that must outlive the container. |
| `-v H:C[:ro]` | Mount host path `H` at container path `C`, optionally read-only. Repeatable. |
| `-p H:C` | Publish container port `C` on host port `H`. `-p H+` picks the first free host port at or above `H`. Repeatable. |
| `-e KEY=VALUE` | Set an environment variable inside the container. Repeatable. `-e KEY` with no `=` copies the value from your host environment. |
| `--memory <limit>` | Set a memory ceiling (e.g. `40G`, `512M`) via a cgroup v2 limit. |
| `--gpu` | Pass host NVIDIA GPU (and DRI) devices through into the container. |
| `--net host` | Use host networking instead of the isolated default. |
| `--gui-foreground` | On desktop images, run your command in the foreground so it owns the terminal; the desktop starts detached. |
| `--no-overlay` | Force bind-mount mode instead of overlayfs. |
| `--privileged` | Run with real root and no user namespace. Requires running as root; the runtime prints a loud warning. |
| `--home <path>` | Override the runtime home directory (normally from `SCORPIOX_HOME`, default `~/.scorpiox`). |
| `--no-perf` | Disable the boot-timing output (or set `SCORPIOX_PERF=0`). |
| `--verbose` | Show debug output. |
| `--list` / `-l` | List available images. |
| `--update <image>` | Re-download and re-extract an image. |
| `--clean` | Remove unpacked root filesystems and leftover work directories. |
| `-v` / `--version` | Print the version. |
| `-h` / `--help` | Show help. |

Four of these deserve a sentence each:

- **`--bind` vs `-v`.** `--bind` is the one purpose-built for "get my project in": it lands at `/workspace`, which is also where the shell starts. `-v` is the general tool for mounting anything anywhere, including read-only, at a path you choose.
- **`--persist`.** Since the container filesystem is thrown away on exit, anything that must survive — a package cache, a build artifact, a download — belongs under `/persist`, where it is just an ordinary host directory.
- **`--net host`.** Isolated networking is the default and the safe choice. `--net host` gives the container your host's network interfaces directly: faster and simpler, but no longer network-isolated. Treat it as a deliberate decision, not a default.
- **`--privileged`.** This disables the user-namespace boundary and hands the container real root with full capabilities. It exists for the rare image that genuinely needs it, or for debugging. The runtime refuses it unless you are already root and warns loudly when it engages, because you are opting out of the isolation that is the whole point.

Limits to be aware of: up to **16** port mappings, **16** volume mounts, and **32** environment variables per container. That is plenty for an agent workspace and keeps the runtime small.

---

## Networking and ports

The default is **isolated networking**: the container gets its own network stack with its own interfaces, and reaches the outside world through a small userspace network process. The container sees a normal-looking interface and a resolver pointed at the network gateway; host loopback is not reachable from inside, which is the isolation you want.

That network process is **bundled**. SCORPIOX CODE ships `scorpiox-slirp4netns`, a pure-libc, zero-dependency drop-in for the slirp4netns subset the runtime uses, placed next to the runtime binary. This means isolated networking works with nothing installed on the host — the runtime is genuinely self-contained. Which helper it picks is controlled by one variable, `SX_NET_BACKEND`:

| `SX_NET_BACKEND` | Which helper is used |
|---|---|
| `auto` (default) | The system `slirp4netns` if it is installed, otherwise the bundled `scorpiox-slirp4netns`. |
| `scorpiox` | Always the bundled `scorpiox-slirp4netns` (next to the runtime binary, then on `PATH`). |
| `system` | Always the system `slirp4netns` from `PATH`. |

If no helper can be found at all — a stripped-down host with neither the bundled binary nor a system `slirp4netns` — the runtime fails fast with a clear message rather than starting a container that cannot reach the network.

`-p H:C` maps host port `H` to container port `C`; `-p H` alone maps the same port on both sides. The `-p H+` form is the useful one for automation: it asks for the first free host port at or above `H`, so parallel containers never fight over a port — the desktop images use exactly this to publish VNC.

By default a published port binds on all interfaces. Set `SX_PORT_BIND_ADDR=127.0.0.1` to keep it on loopback only — strongly recommended for anything unauthenticated, and reachable from elsewhere with a plain SSH tunnel.

To reach a service *inside* the container from *outside* it, publish a port:

```bash
scorpiox-unshare -p 8080:8080 agentcore-latest "python -m http.server 8080"
```

`--net host` skips all of the above and shares the host's network namespace: the container sees your interfaces, your resolver, and everything listening on your machine. It also removes the need for the network helper entirely, which is why some minimal hosts use it — but it is a real reduction in isolation.

---

## Inside the container

A few things are set up for you before your command runs:

- **Your agent config comes with you.** On the session-manager path, if `~/.claude` exists on the host it is mounted read-only at `/root/.claude`. Your agent identity, settings, and credentials are visible inside the container without being modifiable by it.
- **Host binaries can be reused.** When launched by SCORPIOX CODE, the install directory is mounted read-only at `/opt/host-bin` and its binaries are linked into `/usr/local/bin`, so the container runs the same build you run on the host instead of downloading another copy. `TMUX_BIND_HOST_BINS` controls this on the session-manager path; the runtime itself does the mounting whenever it is given the directory.
- **A clean, predictable environment.** Inside the container you get a standard `PATH`, `HOME=/root`, `USER=root`, `LANG=C.UTF-8`, your host's `TERM`, and a `container` variable set to `scorpiox-unshare` that scripts can use to detect they are inside the sandbox. Anything you pass with `-e` is applied on top.
- **Working `/proc` and `/dev`.** A real `/proc` and `/sys`, device nodes for `null`, `zero`, `random`, `urandom` and `tty`, a private pseudo-terminal set, and a writable `/tmp` — enough that shells, package managers, and compilers behave as they do on the host. A synthetic random-entry substitute is provided when a kernel hides those files from user namespaces, so shell profiles do not complain.
- **Package bootstrap, opt-in.** The `CONTAINER_PACKAGES` environment variable (a comma-separated list) is installed at first start with whatever package manager the image uses, with common names mapped per distribution, and the result is cached so it only ever runs once. Shipped as `none` — the images SCORPIOX CODE publishes already carry what agents need — so leave it alone unless you are using your own image.
- **Image entrypoints are honored.** If the image ships an entrypoint, it runs; your command is passed through it. Otherwise your command runs directly, and an interactive session gets a login shell. GUI images that ship a desktop entrypoint bring the desktop up and run your command against it — with `--gui-foreground` your command owns the terminal instead of being backgrounded by the desktop startup.

Everything is torn down again on exit: the writable layer, the temporary mounts, and the per-run work directory are removed, and the container's exit status is passed straight back to your shell. Interruptions (Ctrl-C, `SIGTERM`) clean up too.

---

## Configuration (the `TMUX_` keys)

When you run through the SCORPIOX CODE path you do not pass these flags by hand — you set keys in the same `scorpiox-env.txt` cascade the rest of the product uses (see [Configuration and Profiles](scorpiox-env.md) for tiers, precedence, and profiles). The keys that matter for unshare sessions all start with `TMUX_`:

| Key | Meaning |
|-----|---------|
| `TMUX_LAUNCH_MODE` | `unshare` (default), `podman`, `wsl`, or `native`. Sets how sessions are built. |
| `TMUX_DEFAULT_DISTRO` | The default image for unshare sessions. Shipped as `agentcore-latest`. |
| `TMUX_UNSHARE_EXTRA_ARGS` | Extra CLI flags passed straight to `scorpiox-unshare` (e.g. `"--persist /data -p 8080:8080"`). |
| `TMUX_UNSHARE_NET_MODE` | Empty means isolated networking (the default); `host` shares the host network. |
| `TMUX_UNSHARE_VOLUME_MOUNT` | Explicit volume mount (`host:container`). Leave empty to auto-mount the project base paths. |
| `TMUX_BIND_HOST_BINS` | Bind-mount the host's binaries into the container (default on — skips a download). |
| `TMUX_BIND_USER_CONFIG` | Bind-mount `~/.claude` into the container read-only (default on). |
| `TMUX_BIND_TOOLS` | Bind-mount a host tools directory at `/mnt/apps` read-only (default off). |
| `TMUX_TOOLS_PATH` | Host directory bound at `/mnt/apps` when `TMUX_BIND_TOOLS=1` (default `/root/tools`). |
| `TMUX_GUI_FOREGROUND` | On desktop images, run the agent in the foreground so it owns the pane (default on). |
| `TMUX_GUI_VNC_PORT` | Publish the desktop's VNC port to a host port so you can watch it (default `0` = off). |
| `TMUX_GUI_VNC_BIND` | Host address the published VNC port binds to (default all interfaces; set `127.0.0.1` on a shared network). |

Volume mounts deserve one note: with `TMUX_UNSHARE_VOLUME_MOUNT` empty, the session manager mounts your project base paths into the container at the same paths they have on the host, so agent and host see the project in the same place. Set the key to a `host:container` pair to control it yourself.

A minimal `scorpiox-env.txt` that says "unshare, headless, on this machine":

```ini
TMUX_MODE=local
TMUX_LAUNCH_MODE=unshare
TMUX_DEFAULT_DISTRO=agentcore-latest
```

Everything else falls back to the shipped defaults, and every one of these keys also works as an OS environment variable or inside a named profile, exactly like the rest of the configuration cascade.

### Environment variables the runtime itself reads

A handful of variables are read directly by `scorpiox-unshare`, whether it was launched by SCORPIOX CODE or by hand:

| Variable | Meaning |
|----------|---------|
| `SCORPIOX_HOME` | Runtime home directory (default `~/.scorpiox`). `--home` overrides it. |
| `IMAGE_BASE_URL` | Where images are downloaded from (default `https://dist.scorpiox.net/container-images/`). |
| `SX_NET_BACKEND` | `auto` (default), `system`, or `scorpiox` — which userspace network helper to use. |
| `SX_PORT_BIND_ADDR` | Address a published port binds to (default `0.0.0.0`; set `127.0.0.1` to keep it on loopback). |
| `CONTAINER_PACKAGES` | Comma-separated packages to install on first start (default `none`). |
| `SCORPIOX_PERF` | `0` disables boot-timing output. |

---

## How it compares to Docker and podman

If you already know `docker` or `podman`, here is where `scorpiox-unshare` stands relative to them — and where it does not.

| | `scorpiox-unshare` | Docker / Docker Desktop | podman |
|---|---|---|---|
| **Daemon** | None — one process per container | A daemon is required and must be running | Rootless mode has no daemon, but the tooling around it is heavier |
| **Requires root** | No — rootless by design | Docker Desktop: no, but it runs a Linux VM underneath; plain `dockerd`: yes | No, in rootless mode |
| **Virtual machine** | No — kernel namespaces only | Docker Desktop on macOS and Windows: yes | No — kernel namespaces |
| **External dependencies** | None — pure C, with its own bundled userspace network helper | A large daemon, a container runtime, and a registry client | podman plus buildah, skopeo, and a container runtime |
| **Image format** | `.tar` root filesystem / OCI layout | OCI / registry images | OCI / registry images |
| **Startup** | A fraction of a second from the cached root filesystem | Fast, but daemon and image pull add up | Comparable to unshare in rootless mode |
| **Footprint** | One small binary | Large — daemon, VM on non-Linux, registry client | Moderate |
| **Build / multi-stage / compose / network objects** | None of these | Yes, extensively | Yes, extensively |
| **Best at** | Disposable isolated Linux for an agent or a command | General container platform | General container platform, daemonless |

**Where it is superior.** For the specific job of "give an AI agent a fast, isolated, disposable Linux environment," `scorpiox-unshare` does more with less. There is no daemon to install, babysit, or restart after a reboot; no VM image to boot, patch, or license; no registry client to configure; nothing resident between uses. The thing you are actually optimizing for — a real isolation boundary with near-zero setup and near-instant start — is exactly what a namespace runtime is good at and exactly where a full container stack is overkill. Starting a container becomes the boring, reliable, sub-second part of the experience rather than a thing you troubleshoot.

**The honest limits.** It is not a general container platform, and it does not pretend to be. There is no `Dockerfile` builder and no multi-stage builds; images are pulled as root filesystems from a base URL rather than built and published through the registry ecosystem; there is no `compose`, no named networks, no volume objects — you get port publishing and bind mounts, and that is the whole surface. A `--memory` limit needs cgroup v2 with a writable cgroup tree on the host. GPU passthrough only exposes devices that exist on the host. If your real job is "run a containerized service with a build pipeline and a compose stack," use Docker or podman — that is what they are for. `scorpiox-unshare` is for the one thing it does exceptionally well: a fast, isolated, rootless Linux box for a command, with nothing else attached.

The networking story is fully self-contained: isolated networking uses SCORPIOX CODE's own bundled userspace helper (`scorpiox-slirp4netns`), so there is nothing extra to install. If you prefer the system `slirp4netns` — or you have a minimal host where it is already present — set `SX_NET_BACKEND=system` to use it, or leave it on `auto` and let the runtime pick.

---

## A typical session, end to end

```bash
# In the scorpiox-tmux dashboard:
/new myapp                          # build the unshare session (downloads the image on first run)
/watch myapp                        # watch it work (Esc to stop)
/send myapp "git status"            # poke it without taking over the pane
/enter myapp                        # jump in when it needs you
/kill myapp                         # tear it down when it is done
```

Directly, without SCORPIOX CODE:

```bash
scorpiox-unshare --bind ~/myapp agentcore-latest "make test"
# or drop into a shell:
scorpiox-unshare --bind ~/myapp agentcore-latest
```

Either way the container is built from the image, your project is mounted in, the command runs, and the ephemeral state is cleaned up on exit. What survives is what you explicitly mounted with `--bind`, `-v`, or `--persist`.

---

## Gotchas

- **The first launch is slow, the rest are fast.** The image is downloaded and unpacked once and cached under `~/.scorpiox`. If a session feels slow only on its very first run, that is the cache filling — not a problem.
- **`-v` is overloaded by design.** `-v H:C[:ro]` is a volume mount, but `-v` alone (or `--version`) prints the version. The runtime tells the two apart by whether the next argument contains a `:`. Do not type `-v` followed by a path with no colon and expect a mount.
- **`--net host` removes network isolation.** It is a deliberate escape hatch, not a default. If you set `TMUX_UNSHARE_NET_MODE=host`, you have opted out of the network boundary.
- **`--privileged` is a security off-switch.** It disables the user namespace and grants real root. The runtime warns loudly. Treat it as "I have read the warning and I need it," not a tuning knob.
- **The container filesystem is ephemeral.** Writes to `/` do not persist across launches. Use `--persist`, `--bind`, or `-v` for anything you want to keep.
- **Isolated networking is self-contained by default.** The bundled `scorpiox-slirp4netns` helper ships next to the runtime, so nothing needs to be installed. The only way isolated networking fails is on a host where neither the bundled helper nor a system `slirp4netns` is present; the runtime fails fast with a clear message rather than starting a container that cannot reach the network.
- **User namespaces must be enabled on the host.** Most modern kernels allow unprivileged user namespaces by default. If the runtime reports they are unavailable, the host has disabled them (for example an older `kernel.unprivileged_userns_clone` sysctl) and they need to be turned on; the runtime prints the exact sysctl to set.
- **Runs as real root? Then no user namespace.** If you start the runtime as root it skips the user-namespace mapping — there is nothing for it to map. That is the correct, documented behavior, but it means a root-launched container is not rootless.
- **It is Linux-only.** There is no `scorpiox-unshare` for plain Windows or macOS — there are no Linux namespaces there. On those systems SCORPIOX CODE falls back to `native` mode (see [`scorpiox-tmux`](scorpiox-tmux.md)); on Windows, `wsl` mode when WSL is installed.

---

## The bottom line

`scorpiox-unshare` is the reason SCORPIOX CODE can hand every agent an isolated Linux environment without asking you to install or run anything. It is a pure-C, rootless container runtime that leans on the kernel's namespaces for isolation, pulls a rootfs image when it needs one, ships its own userspace network helper, and starts a container in a fraction of a second with no daemon, no VM, and no external machinery. It is the default under your SCORPIOX CODE sessions, and it is also a standalone tool: point `<image> <command>` at anything and you get a disposable, isolated Linux box to run it in.

---

## Related

- [Managing Agent Sessions with scorpiox-tmux](scorpiox-tmux.md) — the dashboard that launches every session on this runtime.
- [Configuration and Profiles](scorpiox-env.md) — the cascade the `TMUX_*` keys live in.
- [Remote Agent Control and Fleet Management with SCORPIO BOT](scorpiox-bot.md) — drive the same sessions from anywhere.
- [Privacy Policy & Data Architecture](privacy.md) — what stays on your machine, including inside these containers.
