# Native MCP 2.0 & OAuth 2.1 in SCORPIOX CODE: Architecture vs Other Harnesses

Most agent harnesses treat MCP as an afterthought: bolt a Node.js subprocess onto the side, pipe JSON through `stdin`/`stdout`, and call it a day. SCORPIOX CODE does it differently. It speaks **MCP 2.0 over Streamable HTTP** natively, ships a **zero-dependency OAuth 2.1 PKCE** login, and can even *serve* your own scripts as an MCP server — all from small, self-contained C99 binaries with no runtime to install.

This page walks through how the whole system works, why the architecture matters, and how it compares head-to-head with Claude Code, Cursor, OpenCode, and Hermes Agent.

Source of truth: `scorpiox-mcp.c`, `scorpiox-mcp-httpclient.c`, `scorpiox-mcp-login.c`, `libsxnet/sx_mcp.c`, and `scorpiox-server.c` at commit `5fd054b`.

> **Not a Node.js toy.** SCORPIOX CODE's MCP is not the legacy "spawn a Node or Python process and talk to it over stdio" design. It is a pure C99, Streamable HTTP, JSON-RPC 2.0 client and server, with first-class OAuth 2.1 PKCE. That single choice removes an interpreter, a dependency tree, and a whole class of cold-start latency.

---

## The three pieces

SCORPIOX CODE's MCP is one capability split across three small binaries. You rarely need all three, but knowing what each does is the whole ballgame:

| Binary | Role | What it talks to |
|--------|------|------------------|
| `scorpiox-mcp` | Local tool manager | Local MCP server subprocesses (stdio, JSON-RPC 2.0) |
| `scorpiox-mcp-httpclient` | Remote tool client | Remote MCP servers (Streamable HTTP, JSON-RPC 2.0) |
| `scorpiox-mcp-login` | OAuth 2.1 PKCE client | Any remote MCP server that needs authorization |
| `scorpiox-server --mcp` | MCP **server** | Your local scripts / a git repo, exposed as MCP tools |

The engine itself (`libsxnet/sx_mcp.c`) is a thin shim: when `MCP=1`, it discovers tools and presents them to the model as `mcp__<server>__<tool>`. So from the model's point of view, an MCP tool looks exactly like a built-in tool — no special handling.

---

## Why the architecture is fundamentally different

### 1. Zero runtime bloat

The other harnesses reach for MCP by running an *interpreter*. Claude Code and Cursor lean on Node.js and `npx`; OpenCode is built on Bun; a Python-backed MCP server needs a venv. Each of those pulls in a runtime, a package manager, and a dependency tree that can quietly break on the next update.

SCORPIOX CODE ships MCP as small native C99 binaries. No `node_modules`, no `venv`, no `npm install` to run. A binary that is tens of kilobytes, not hundreds of megabytes. The security and supply-chain surface area of "did a transitive npm package just break" simply does not exist here.

### 2. Streamable HTTP, not just local pipes

The MCP spec's **Streamable HTTP** transport means a single `POST` of JSON-RPC 2.0 to an endpoint — stateless per command, session-tracked via an `Mcp-Session-Id` header. That is what lets you:

- Call remote, enterprise-hosted tools with no local process to spawn.
- Run each command independently: `initialize` → action → exit. No long-lived local daemon you have to babysit.
- Point at any server that speaks MCP 2.0 over HTTP, including the built-in `scorpiox-server --mcp`.

Local stdio-only harnesses can only ever talk to a process they started themselves on the same machine.

### 3. Native OAuth 2.1 with PKCE

`scorpiox-mcp-login` implements the full MCP authorization flow, end to end, with **no external OAuth library and no browser lock-in**:

1. Probe the endpoint → get a `401` + `WWW-Authenticate` challenge.
2. Discover `authorization_servers` via `/.well-known/oauth-protected-resource`.
3. Fetch server metadata (endpoints, supported scopes, PKCE S256) via `/.well-known/oauth-authorization-server`.
4. **Dynamic client registration** — `POST /register` returns a `client_id` on the fly. No pre-provisioned app.
5. Run the PKCE authorize flow (S256 code challenge).
6. Exchange the code → `access_token` + `refresh_token` via `POST /token`.
7. Cache the result to `~/.mcp/credentials/<sha256-of-url>.json`.

The token is cached per-URL and auto-refreshed, so a second `scorpiox-mcp-httpclient` call against the same server picks up the bearer token automatically — you log in once and it stays fresh.

### 4. Headless & SSH-native

The PKCE flow is designed for the terminal. It opens a browser on a local callback port when one is available, and falls back to a **manual paste** path when it is not. That is the whole point of a headless OAuth: you can authenticate against a remote MCP server from a bare SSH session on a Linux box, with no desktop, no GUI, no browser required. This is the kind of thing a desktop-bound harness simply cannot do.

### 5. Dual role: client *and* server

Most harnesses are MCP **consumers only**. SCORPIOX CODE can also be the provider. `scorpiox-server --mcp` turns a folder of scripts (or a git repo) into an MCP 2.0 server, where each script file becomes a tool:

```bash
# Serve a folder of scripts as MCP tools on port 8888
scorpiox-server --mcp -p 8888

# Serve scripts from a git repo instead of a local folder
scorpiox-server -r <repo-url> --mcp

# Give the server a custom name (used in the initialize handshake)
scorpiox-server --mcp --name myserver
```

Each script is self-describing via a `--schema` flag: line 1 is the tool description, subsequent lines are `param_name:type:description:required|optional`. The endpoint is `POST /mcp` over Streamable HTTP — connect to it from SCORPIOX CODE's own `scorpiox-mcp-httpclient`, or from any other Streamable HTTP MCP client.

### 6. Granular security: allow/deny glob filtering

Instead of "expose every tool the server has," you control exactly what the model sees, per server, with glob patterns in `scorpiox-env.txt`:

```
MCP=1
MCP_SERVER=devkit:/root/.claude/devkit-linux-x64;fs:npx:-y,@anthropic/mcp-filesystem,/home
MCP_ALLOW=devkit:mailing_tool_*,git_smart_commit_*;fs:*
MCP_DENY=devkit:*shutdown*,*delete*
```

`MCP_ALLOW` whitelists by server and tool pattern; `MCP_DENY` overrides allow and removes tools. Blanket tool exposure is the default in many harnesses — this is not the default here.

### 7. Performance & cold boot

A native binary starts in sub-millisecond time. There is no interpreter to boot, no module graph to load, no JIT to warm up. For a harness that fans out to many tool calls, that cold-start difference (sub-millisecond vs. the 2–5 seconds a Node.js interpreter can take on a cold start) adds up quickly across a session.

---

## Quick start

### Consume a remote MCP server

```bash
# 1. Authenticate (one-time; caches + auto-refreshes the token)
scorpiox-mcp-login https://mcp.scorpiox.net

# 2. See what tools it offers
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp list

# 3. Inspect a single tool's schema
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp info <tool>

# 4. Call a tool
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp call <tool> '{"key":"value"}'

# 5. Health-check connectivity
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp test
```

Each command is stateless: it initializes the session, performs the action, and exits. The bearer token from step 1 is loaded automatically for steps 2–5.

### Manage tokens

```bash
scorpiox-mcp-login --list                 # all saved credentials
scorpiox-mcp-login <url> --status         # token status for one server
scorpiox-mcp-login <url> --refresh        # force a refresh
scorpiox-mcp-login <url> --force          # re-auth even if a token exists
scorpiox-mcp-login <url> --token          # print the bearer token to stdout
```

### Serve your own tools

```bash
scorpiox-server --mcp -p 8888
# then, from SCORPIOX CODE or any Streamable HTTP MCP client:
scorpiox-mcp-httpclient http://localhost:8888/mcp list
```

### Enable MCP inside the engine

Turn on the master switch and register your servers in `scorpiox-env.txt`:

```
MCP=1
MCP_SERVER=devkit:/root/.claude/devkit-linux-x64
# optionally restrict what the model can see:
MCP_ALLOW=devkit:mailing_tool_*
MCP_DENY=devkit:*shutdown*
```

With `MCP=1`, discovered tools appear to the model as `mcp__<server>__<tool>` and are invoked through the normal tool path.

---

## Comparison matrix

How SCORPIOX CODE's MCP stacks up against the other major agent harnesses:

| Capability | SCORPIOX CODE | Claude Code | Cursor | OpenCode | Hermes Agent |
|------------|---------------|-------------|--------|----------|--------------|
| **Protocol** | Streamable HTTP (MCP 2.0) + local stdio | stdio subprocess | stdio subprocess (Electron) | stdio subprocess (Bun) | stdio subprocess |
| **Remote tools over HTTP** | Native, first-class | Not native (local only) | Not native (local only) | Not native (local only) | Not native (local only) |
| **Auth** | Native OAuth 2.1 PKCE + dynamic client registration | Manual API keys / none | Manual API keys | Browser-based auth | Manual / none |
| **Token management** | Auto-cache + refresh (`~/.mcp/credentials/<hash>.json`) | Manual | Manual | Manual | Manual |
| **Runtime / dependencies** | Pure C99, zero runtime deps | Node.js + npm | Electron + Node.js | Bun | Node.js |
| **Binary footprint** | Tens of KB native binary | `node_modules` (hundreds of MB) | Electron app bundle | Bun runtime | npm tree |
| **Cold start** | Sub-millisecond | ~2–5 s interpreter | Heavy (Electron) | Faster (Bun) but still JS | ~2–5 s interpreter |
| **MCP server capability** | Built-in C HTTP server (`scorpiox-server --mcp`) | N/A (consumer only) | N/A (consumer only) | N/A (consumer only) | N/A (consumer only) |
| **Headless / SSH support** | Native (manual-paste PKCE fallback) | Requires local browser | Requires GUI browser | Requires browser | Requires local browser |
| **Tool filtering** | Per-server glob `MCP_ALLOW` / `MCP_DENY` | All-or-nothing | All-or-nothing | All-or-nothing | All-or-nothing |
| **Supply-chain surface** | None (no packages to install) | npm transitive deps | npm transitive deps | Bun/npm deps | npm transitive deps |

The short version: every other harness *consumes* MCP by spawning an interpreted local process and leans on you to paste API keys. SCORPIOX CODE *speaks* MCP 2.0 over HTTP natively, authenticates with a real OAuth 2.1 PKCE flow, filters tools per server, and can flip the roles to *serve* your own tools.

---

## How tool names work

When you enable MCP in the engine, each discovered tool is exposed to the model with a namespaced name:

```
mcp__<server>__<tool>
```

For example, a tool `send_email` on the `devkit` server becomes `mcp__devkit__send_email`. This keeps tools from different servers from colliding and makes it obvious, in a transcript, where a tool call came from.

---

## Configuration reference

All MCP settings live in `scorpiox-env.txt`:

| Key | Purpose |
|-----|---------|
| `MCP` | Master switch. `0` = off (default), `1` = on. |
| `MCP_SERVER` | Semicolon-separated `name:command:arg1,arg2,...` server definitions. |
| `MCP_CONFIG` | Path to an external `mcp_servers.json`. Takes precedence over `MCP_SERVER`. |
| `MCP_FILE` | Path to a `.mcp` file (set automatically for agent tasks). |
| `MCP_ALLOW` | Per-server glob allow-list. Empty = allow everything. |
| `MCP_DENY` | Per-server glob deny-list. Overrides `MCP_ALLOW`. |
| `MCP_SERVER_NAME` | Name reported in the initialize handshake (default `scorpiox-server`). |
| `MCP_TOOL_EXCLUDE` | Comma-separated globs of scripts to skip when serving via `scorpiox-server --mcp`. |

Example:

```
MCP=1
MCP_SERVER=devkit:/root/.claude/devkit-linux-x64;fs:npx:-y,@anthropic/mcp-filesystem,/home
MCP_ALLOW=devkit:mailing_tool_*,git_smart_commit_*;fs:*
MCP_DENY=devkit:*shutdown*,*delete*
MCP_TOOL_EXCLUDE=helper-*,setup,*.bak
```

---

## Gotchas

- **It is off by default.** `MCP=0` out of the box. Nothing is discovered or exposed until you set `MCP=1` and define servers.

- **`MCP_DENY` beats `MCP_ALLOW`.** A tool that matches both an allow and a deny pattern is removed. Use deny for the sharp edges (`*shutdown*`, `*delete*`).

- **Login is per-URL.** Credentials are cached keyed on the SHA-256 of the server URL, in `~/.mcp/credentials/<hash>.json`. Pointing at a different host means a different (and fresh) token.

- **The headless fallback is intentional.** Over SSH with no browser, the PKCE flow offers a manual-paste path rather than failing — that is the feature, not a workaround.

- **`--mcp` server mode is experimental.** Treat the built-in server as a way to publish scripts to MCP clients, not as a hardened public API endpoint — protect it with your network boundaries and the JWT/IP-whitelist options already available to `scorpiox-server`.

- **The engine's `scorpiox-mcp` is stdio; the HTTP client is separate.** Local subprocess servers go through `scorpiox-mcp`; remote Streamable HTTP servers go through `scorpiox-mcp-httpclient`. They share the same `mcp__<server>__<tool>` naming in the model's view.
