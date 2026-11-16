# Plugins in SCORPIOX BOT

SCORPIOX BOT is a small, sharp tool until the day you want it to show you something it does not ship with — the GPU your job is saturating, the ports open on the box you are about to expose, the SQLite file your agent has been writing to. **Plugins** are how that happens without waiting for a release: a plugin is one native executable named `plugin_<name>` that lives on your node, and the bot runs it on demand and puts its page in front of you. A public catalogue, **BotMarket**, lists the plugins people have published, with their source and their checksums, and installs one onto your node in a click.

Docs for SCORPIOX CODE @ `ce3b5cd`.

> **The whole idea in one line:** a plugin is a single native binary on your node — metadata, a JSON API and a web UI in the same file — and BotMarket is the open catalogue that installs it, with the source and the SHA-256 published next to every artifact.

---

## What a plugin actually is

Be clear about this before installing anything.

- **A native executable, not a widget.** A plugin is a real binary, `plugin_<name>` (or `plugin_<name>.exe` on Windows), that the operating system runs with the permissions of the user who owns the node. It is not sandboxed, not interpreted, and not subject to a browser's security model. Everything it can read, it can read; everything it can run, it can run.
- **One binary, three modes.** The same file answers metadata, JSON and its own web page. There is no manifest server, no plugin runtime, no dependency to install.
- **Discovered on every request.** Drop the binary into a plugin directory and the bot finds it. There is no restart, no service to reload, no registry to edit.
- **Scoped to you and your node.** A plugin on your node is visible to you. Another account on the same web front end does not see it and cannot reach it.

That is the bargain: a plugin is as powerful as a shell command, which is why the market publishes the source and a digest for every artifact, and why the install dialog asks you to read before you click.

---

## Where plugins live

| Directory | When to use it |
|-----------|----------------|
| `/workspace/.scorpiox/plugins` | Preferred on BotCloud containers. `/workspace` survives a container restart, so an installed plugin does too. |
| `~/.scorpiox/plugins` | A workstation or a self-hosted node. On Windows this is `%LOCALAPPDATA%\scorpiox\plugins`. |
| `BOT_PLUGIN_DIR` | Set this environment variable to point the bot at a different directory. |

The bot creates these directories on startup, so they are usually already there. Copy a binary in, make it executable on Unix (`chmod +x`), and it shows up in the plugin list on the next scan. Remove the file and it is gone.

Keep helper binaries — anything a plugin shells out to — named *without* the `plugin_` prefix. The prefix is the only thing that marks a file as a plugin, and a helper that carries it will be listed as a UI plugin you can open.

---

## The three-mode contract

Every plugin in BotMarket is the same shape. One C file, no dependencies, one binary, three modes. This is the whole contract.

| Mode | Trigger | Response |
|------|---------|----------|
| Metadata | `--meta` argument, or `SX_PLUGIN_META=1` in the environment | `{"name":"...","desc":"..."}` as JSON |
| JSON API | `?action=...` or `?format=json` on the query string | `application/json` |
| Web UI | a plain `GET` | `text/html` — the tool's page |

Two rules that are easy to forget and expensive to debug:

- **Never emit `Content-Length`.** The server computes the exact payload length itself. A second `Content-Length` header makes the reverse proxy reject the response.
- **Honour `?embed=1`.** Your page is often rendered inside the chat surface or a settings panel. When `embed=1` is present, drop any standalone top bar so the tool fits the frame it was given.

A minimal plugin, complete:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(int argc, char **argv)
{
    const char *qs = getenv("QUERY_STRING");

    if ((argc > 1 && strcmp(argv[1], "--meta") == 0) || getenv("SX_PLUGIN_META")) {
        puts("{\"name\":\"Ports\",\"desc\":\"Listening TCP\"}");
        return 0;
    }

    if (qs && (strstr(qs, "action=") || strstr(qs, "format=json"))) {
        printf("Content-Type: application/json; charset=utf-8\r\n\r\n"
               "{\"ok\":true,\"items\":[]}\n");
        return 0;
    }

    printf("Content-Type: text/html; charset=utf-8\r\n\r\n"
           "<!DOCTYPE html><html><body><h1>Ports</h1></body></html>\n");
    return 0;
}
```

```bash
cc -O2 -Wall -Wextra -o ~/.scorpiox/plugins/plugin_ports plugin_ports.c
chmod +x ~/.scorpiox/plugins/plugin_ports
```

On a node with a password, read it from the `BOT_PASSWORD` environment variable and accept it either as the `X-Bot-Password` header or as the `pwd` parameter of the page URL — the chat frame appends it to the URL rather than setting a header, and a plugin that only checks one of the two will refuse the frame that renders it.

### Metadata style the market enforces

The catalogue is a grid of cards, and the card is only as useful as its two fields:

- **Short Title — one or two words.** `sysmon`, `docker`, `ports`, `sqlite`. Not "System Monitoring Dashboard".
- **Ultra-Concise Description — 15 characters or fewer.** `CPU & memory`, `Containers`, `Listening TCP`.

The longer `title` and `summary` are free text and belong in the detail view, not on the card.

---

## Browsing and installing from BotMarket

The public catalogue is **https://botmarket.scorpiox.net**. Inside the product, the same catalogue is a tab in your settings.

**Settings → Plugins** has three tabs:

- **Installed (N)** — every `plugin_*` executable this node reports, with its version, an **Open** link, a **Source** link, and — for anything BotMarket manages — **Update** and **Remove**. Each installed plugin also carries three placement chips: **Hub**, **Session Card** and **Chat**, plus a master switch. Those decide where the plugin appears: in the sessions hub's more menu, on a session card's menu, or as a floating window inside chat. A plugin you placed by hand still gets the chips; it simply has no version to update.
- **Marketplace (M)** — the catalogue merged with this node's state. Search, category chips, the plugin's `--meta` badge, the one-line summary, the artifact target this node would install, a **Source** link, an **Agent Prompt** copy button, and **Install** (or **Installed** when it already is). **Rescan** refreshes both the node scan and the catalogue.
- **Updates (K)** — only catalogue items with a newer version than this node runs. The tab stays hidden while its count is zero.

### What the install dialog tells you

Nothing installs the moment you click **Install**. The untrusted-native-binary dialog opens first, and it states the four things you should know:

1. **What code will run** — the plugin and its published capabilities.
2. **Which node** — the machine that will execute it, not the web front end you are looking at.
3. **Which commit** — the exact pinned source revision, with a link to the public repository.
4. **Which SHA-256** — the digest of the artifact, so you can verify the bytes yourself.

Then it offers three routes, and all three are legitimate:

- **Install Prebuilt** — download the artifact built for this node, verify it, put it in place.
- **Build from Source on Node** — clone the pinned source on the node and compile it there. Slower, needs a compiler, and leaves you with a binary you built yourself from source you can read.
- **Agent Prompt** — copy a ready-made prompt into SCORPIOX BOT chat and let your agent do the clone, audit, customise and compile for you. This is the route to take when you want the plugin *changed* for your setup, not just installed.

Deep link straight to a plugin's card:

```
https://bot.scorpiox.net/settings?sec=plugins&tab=market&install=<slug>
```

### What the installer verifies

The node does not trust the download blindly. In order:

1. The artifact URL must be on the published distribution host — anything else is refused unless the operator has explicitly allowed untrusted origins.
2. The bytes are hashed and compared against the stated SHA-256. A mismatch aborts and the temporary file is deleted.
3. The file's first bytes must be a real executable for this platform. This is what stops an HTML error page that was saved as a binary during an outage from being installed and run.
4. The binary is executed once with metadata mode requested, and its answer must be valid JSON with a name and a description. A crash or a non-JSON answer aborts the install.
5. Only then is the file moved into place and a small sidecar manifest written next to it, recording the slug, version, install method, target, digest and source commit.

**Remove** reverses it, and it will only ever delete a plugin from your own plugin directory. Operator-installed plugins are refused.

---

## The published artifacts

Every plugin ships six binaries under **https://dist.scorpiox.net/bot-plugins/<slug>/<version>/**, each with a digest in the manifest:

| Target | What it is |
|--------|------------|
| `linux-x86_64` | glibc x86_64 ELF |
| `linux-x86_64-musl` | static musl x86_64 ELF — the preferred Linux target |
| `linux-arm64-musl` | static musl aarch64 ELF |
| `macos-arm64` | Apple Silicon Mach-O |
| `macos-x86_64` | Intel macOS Mach-O |
| `windows-x86_64` | native Win64 static PE |

A Linux node asks the installer which target it should use and gets the static musl build as its first choice, because one static binary runs unchanged on Alpine, Debian, Ubuntu, Arch and Omarchy. It falls back to the glibc build only when there is a reason to.

The directory also carries `SHA256SUMS` and a `source.tar.gz` — an archive of the exact pinned commit. Everything needed to verify or rebuild a plugin is in one place, and it works offline once downloaded.

---

## The source-first track

Prebuilt binaries are the fast path. The other path is the one that makes this market worth having: **every plugin is public C99 source at a pinned commit.**

```
https://plugingit.scorpiox.net/scorpiox-bot-plugin-<slug>.git
```

Anonymous read is allowed — `git clone`, browse the history, download an archive, and read every line before a binary ever touches your machine. Each card in the catalogue links its repository and its commit.

Because the source is one dependency-free C file, an agent can do in seconds what used to be a packaging project. Paste this into SCORPIOX BOT chat:

> Clone `https://plugingit.scorpiox.net/scorpiox-bot-plugin-<slug>.git`, audit `src/plugin_<id>.c` for anything unsafe, customise it for my setup, compile it natively on this machine, verify `--meta`, and install it to `~/.scorpiox/plugins/`.

That is the whole workflow: clone, audit, edit, `cc -O2`, copy. No SDK, no version pin, no rebuild pipeline. If you would rather read the procedure yourself first:

- **https://botmarket.scorpiox.net/docs/llms.txt** — the agent-readable index: every plugin, its clone URL, its commit, plus the full contract.
- **https://botmarket.scorpiox.net/catalog/api?format=json** — the live catalogue as data.
- **https://dist.scorpiox.net/bot-plugins/index.json** — every published artifact and its digest.

---

## Writing your own

1. **Copy the starter.** The market publishes a complete single-binary plugin, a Makefile with native, musl, Win64 and macOS targets, and a Windows build script, at **https://botmarket.scorpiox.net/docs/sdk?file=plugin_starter.c**. Change three defines, run `make`.
2. **Name it `plugin_<name>`.** Lowercase, letters, digits, hyphens and underscores, up to 56 characters after the prefix.
3. **Answer `--meta` first.** The card in the marketplace is only as good as those two fields.
4. **Build the page for the frame.** Assume 390 pixels wide on a phone and a floating window inside chat. Honour `?embed=1`, and never emit `Content-Length`.
5. **Ship it where it can be found.** In the container, `/workspace/.scorpiox/plugins`. On a laptop, `~/.scorpiox/plugins`.

Then open **Settings → Plugins**, click **Rescan**, and switch on the placements you want. No restart.

If you want it in the public catalogue, the same rules apply that the market enforces on everything else: clean source, no secrets, no internal addresses, a pinned commit, a SHA-256 for every artifact, and honest metadata.

---

## Safety, in one place

- **A plugin is native code with your permissions.** Install from a source you have looked at, or one you trust. The dialog exists to make that decision possible, not to make it for you.
- **The digest is the check.** The installer verifies the SHA-256 and the executable format before it runs anything.
- **The source is the deeper check.** Every plugin's C99 source is public at a pinned commit, and the whole clone-audit-compile path is supported by design.
- **Plugins are per user, per node.** Discovery and execution are scoped to the account and the node; another user on the same front end gets nothing.
- **Remove means remove.** Deleting a plugin from your own plugin directory takes it off the menus immediately.

---

## Quick recipe: install one and open it

1. Open **Settings → Plugins → Marketplace**.
2. Find the plugin — for example `sysmon` — and read its summary, target and source link.
3. Click **Install**. In the dialog, confirm the node, the commit and the SHA-256, then choose **Install Prebuilt**.
4. Back on the **Installed** tab, click **Rescan**, then switch on the placements you want — **Hub**, **Session Card**, **Chat**.
5. Open **Chat** and pick the plugin from the more menu. The tool renders in a floating window next to your session.

To try the source route instead, choose **Agent Prompt** in step 3 and paste it into chat.

---

## Related

- [SCORPIOX BOT](/scorpiox-bot) — the chat surface, node selection, and the settings this page points into.
- [Docker](/docker) — running SCORPIOX BOT on a container node, where `/workspace/.scorpiox/plugins` keeps your plugins across restarts.
- [Configuration & Profiles](/scorpiox-env) — environment variables, including `BOT_PLUGIN_DIR`.
- [BotMarket](https://botmarket.scorpiox.net) — the public catalogue.
- [plugingit.scorpiox.net](https://plugingit.scorpiox.net) — public source for every published plugin.
