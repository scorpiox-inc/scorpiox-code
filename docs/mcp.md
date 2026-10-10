# Native MCP 2.0 & OAuth 2.1 in SCORPIOX CODE: Architecture vs Other Harnesses

Most agent harnesses treat MCP as an afterthought: bolt a Node.js subprocess onto the side, pipe JSON through `stdin`/`stdout`, and call it a day. SCORPIOX CODE does it differently. It speaks **MCP 2.0 over Streamable HTTP** natively, ships a **zero-dependency OAuth 2.1 PKCE** login, and can even *serve* your own scripts as an MCP server — all from small, self-contained C99 binaries with no runtime to install.

This page walks through how the whole system works, why the architecture matters, and how it compares head-to-head with Claude Code, Cursor, OpenCode, and Hermes Agent.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** SCORPIOX CODE speaks MCP 2.0 with its own lean, native binaries — a JSON-RPC 2.0 client over both stdio (local) and Streamable HTTP (remote), a complete OAuth 2.1 PKCE login (`scorpiox-mcp-login`) that works headless over SSH, and a server mode (`scorpiox-server --mcp`) that turns a folder of your own scripts into tools any MCP client can call. No interpreter, no SDK, no `node_modules`.

---

## The four pieces

SCORPIOX CODE's MCP is one capability split across a handful of small binaries. You rarely need all of them, but knowing what each does is the whole ballgame:

| Binary | Role | What it talks to |
|--------|------|------------------|
| `scorpiox-mcp` | Local MCP client over stdio | Server processes you configure, spawned and managed per call |
| `scorpiox-mcp-httpclient` | Remote MCP client over Streamable HTTP | Any MCP 2.0 HTTP endpoint, with or without OAuth |
| `scorpiox-mcp-login` | OAuth 2.1 PKCE authorization client | Any MCP server that authenticates with OAuth |
| `scorpiox-server --mcp` | MCP server mode (Streamable HTTP) | Any MCP client, including other agents and the llama.cpp Web UI |
| `scorpiox-login` | Dispatch entry point | Runs any provider's login tool verbatim, including the MCP one |

`scorpiox-login` is the umbrella that arrived alongside this generation of the product: one command that finds the matching per-provider login tool and runs it with your arguments. `scorpiox-login mcp <url>` is a straight passthrough to `scorpiox-mcp-login`, so the OAuth flow below is unchanged — you just have one place to remember.

Three things to hold onto before the details:

- **The protocol negotiation is explicit.** Both the client and the server mode declare MCP protocol version `2024-11-05` during `initialize`, and both reply with their own server name and version, so you always know who you are talking to.
- **The client is stateless per command.** Every `scorpiox-mcp-httpclient` invocation runs initialize, does its work, and exits. There is no daemon to babysit and no half-open connection to time out on you.
- **Session continuity is handled the way the protocol says.** If the server issues an `Mcp-Session-Id`, the client captures it and replays it on every subsequent request in that command.

---

## Why the architecture is fundamentally different

### 1. Zero runtime bloat

The other harnesses reach for MCP by running an *interpreter*. Claude Code and Cursor lean on Node.js and `npx`; OpenCode is built on Bun; a Python-backed MCP server needs a venv. Each of those pulls in a runtime, a package manager, and a dependency tree that can quietly break on the next update.

SCORPIOX CODE ships MCP as small native C99 binaries. No `node_modules`, no `venv`, no `npm install` to run. A binary that is tens of kilobytes, not hundreds of megabytes. The security and supply-chain surface area of "did a transitive npm package just break" simply does not exist here.

### 2. Streamable HTTP, not just local pipes

The MCP spec's **Streamable HTTP** transport means a single `POST` of JSON-RPC 2.0 to an endpoint — stateless per command, session-tracked via an `Mcp-Session-Id` header. That is what lets you:

- Call remote, enterprise-hosted tools with no local process to spawn.
- Run each command independently: `initialize`, action, then exit. No long-lived local daemon you have to babysit.
- Point at any server that speaks MCP 2.0 over HTTP, including the built-in `scorpiox-server --mcp`.

Local stdio-only harnesses can only ever talk to a process they started themselves on the same machine.

### 3. Native OAuth 2.1 with PKCE

`scorpiox-mcp-login` implements the full MCP authorization flow, end to end, with **no external OAuth library and no browser lock-in**:

1. Probe the endpoint and get a `401` plus a `WWW-Authenticate` challenge that names the resource metadata URL.
2. Discover the authorization server from that metadata document.
3. Fetch server metadata (endpoints, supported scopes, PKCE S256 support).
4. **Dynamic client registration** — a registration request returns a `client_id` on the fly, as a public client. No pre-provisioned app and no secret to leak.
5. Generate the PKCE challenge: a random verifier, SHA-256 hashed into the `S256` challenge, plus a random `state`.
6. Run the authorization flow (more on the two modes below).
7. Exchange the code for an `access_token` and a `refresh_token`, then save them.

The saved credential file lives under your home directory, keyed by a hash of the server URL, in a directory created with owner-only permissions. It stores the URL, client id, access token, refresh token, granted scopes, token endpoint, revocation endpoint, and an absolute expiry time. Nothing else on the machine can read it.

```bash
scorpiox-mcp-login https://mcp.scorpiox.net                 # full flow (also: scorpiox-login mcp <url>)
scorpiox-mcp-login https://mcp.scorpiox.net --status        # validity, scopes, remaining time, file path
scorpiox-mcp-login https://mcp.scorpiox.net --refresh       # new access token, keeps or rotates the refresh token
scorpiox-mcp-login https://mcp.scorpiox.net --token         # print the bearer token to stdout (scripts)
scorpiox-mcp-login --list                                   # every saved credential on this machine
scorpiox-mcp-login https://mcp.scorpiox.net --force         # re-authenticate, replacing what is stored
```

### 4. Headless and SSH-native

The authorization flow supports **two modes**, and the second one is what makes it usable on a remote Linux box with no desktop:

- **Loopback callback.** The client binds a local listener on `127.0.0.1:19457` and waits for the redirect. The authorize URL is printed for you to open in a browser on any device, and the `state` value is checked when the redirect arrives.
- **Paste mode.** If the loopback port cannot be bound — the normal situation over SSH with no forwarded port — the command drops to paste mode: you paste either the authorization code or the full callback URL, and the code is extracted for you. No browser required on the server, no GUI dependency.

Two behaviors make this genuinely low-maintenance rather than merely present:

- **Refresh is automatic in use.** When the HTTP client receives a `401` with a token attached, it invokes the refresh flow itself, reloads the new token, and retries the request exactly once. You only hear about it if refresh genuinely failed.
- **Expiry is checked before the request, not after the failure.** A stored token that has already expired is flagged when it is loaded, with the exact command to run, instead of surfacing as an opaque server error later.

---

## The four CLIs, in one table

| Command | Mode | Key subcommands |
|---------|------|-----------------|
| `scorpiox-mcp` | stdio client | `discover`, `call`, `list`, `info`, `test` |
| `scorpiox-mcp-httpclient` | Streamable HTTP client | `list`, `call`, `info`, `test` |
| `scorpiox-mcp-login` | OAuth 2.1 PKCE client | full flow, `--force`, `--refresh`, `--status`, `--token`, `--list` |
| `scorpiox-server --mcp` | Streamable HTTP server | `--name`, `-p`, `-r <repo-url>` |
| `scorpiox-login` | Dispatcher | `<provider>` passthrough, `list`, `--status` |

Each one is a standalone static binary. That is not a marketing line; it is the reason they behave the same on a laptop, in a container, and on a headless server with nothing on it but SSH.

---

## Local servers: stdio, configured once

Local MCP servers are declared in `scorpiox-env.txt` and discovered at startup. Tools come back to the model as `mcp__<server>__<tool>` — the same shape as a built-in tool, so nothing downstream needs special handling.

```
# MCP master switch
MCP=1

# Serve definitions: name:command:arg1,arg2 — semicolon-separated for several
MCP_SERVER=devkit:/root/.claude/devkit-linux-x64
MCP_SERVER=devkit:/root/.claude/devkit-linux-x64;fs:npx:-y,@anthropic/mcp-filesystem,/home

# External server list (standard mcp_servers.json), takes precedence when set
MCP_CONFIG=

# Session-scoped .mcp file (set by the launcher for agent tasks)
MCP_FILE=
```

Then drive it from the terminal the same way the agent would:

```bash
scorpiox-mcp list                 # configured servers
scorpiox-mcp discover             # every tool from every server
scorpiox-mcp discover --json      # machine-readable
scorpiox-mcp info devkit send_email
scorpiox-mcp call devkit send_email '{"to":"ops@example.com","subject":"hi"}'
scorpiox-mcp test devkit
```

The client is intentionally small and explicit: it shells out to each configured server process, speaks newline-framed JSON-RPC 2.0, and reports what it finds. Nothing is cached behind your back.

---

## Remote servers: Streamable HTTP with OAuth

This is where the remote story stops being "a subprocess on my laptop" and becomes "any authenticated tool endpoint on the network."

### Logging in once

```bash
scorpiox-mcp-login https://mcp.scorpiox.net
# or, through the umbrella:
scorpiox-login mcp https://mcp.scorpiox.net
```

The flow walks discovery, dynamic registration, PKCE, and token exchange as described above, then stores the credential. From that point on, the HTTP client finds it automatically — keyed by the server URL — and injects it as a bearer token on every request. Expiry is enforced, and an expired token triggers an automatic refresh.

### Calling remote tools

```bash
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp test
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp list
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp info geocode
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp call geocode '{"address":"Auckland"}'
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp call geocode address="Auckland"
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp call websearch --file params.json
echo '{"query":"llama.cpp"}' | scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp call websearch
```

Each command performs the full handshake itself: `initialize`, the initialized notification, then your request, then exit. The `test` command is the fastest way to check a server — it reports the negotiated protocol version, the server's own name and version, the session id if one was issued, and the tool count. Requests accept plain JSON, `key=value` named arguments, a `--file` payload for large or nested inputs, or piped stdin. Responses arrive as plain JSON or as an event stream; both are parsed, so you do not have to know which one a given server speaks.

### The agent path, taught by a built-in skill

A built-in skill ships in the box to steer the agent toward these commands rather than raw `curl` or ad-hoc subprocesses: use the stdio client for local servers, and the login plus HTTP client for remote ones, preferring named `key="value"` arguments and `--file` for nested payloads. The effect is that MCP work in a session looks the same as any other tool call, with no bespoke glue.

---

## Serving your own scripts: `scorpiox-server --mcp`

SCORPIOX CODE is not only an MCP client. It can expose a folder of scripts or a git repo as an MCP 2.0 server over Streamable HTTP:

```bash
scorpiox-server --mcp -p 8888              # serve the current folder's scripts as tools
scorpiox-server -r https://git.example/ops-scripts.git --mcp   # serve a git repo's scripts
scorpiox-server --mcp --name ops-server    # custom advertised server name
```

Every executable in the served folder becomes a tool named after its file. Each script can describe itself by answering a `--schema` probe: the first line is the tool description, and following lines declare parameters as `name:type:description:required|optional`. At startup the server probes each script with a short timeout, registers what answers, prints the tool list, and then answers MCP traffic on `POST /mcp`:

- `initialize` replies with protocol version `2024-11-05`, a tools capability, and the configured server name (`MCP_SERVER_NAME`, or `--name`, defaulting to `scorpiox-server`).
- `tools/list` returns every registered tool with its schema.
- `tools/call` runs the script, passes the arguments as JSON on stdin, captures stdout, and returns it as text content — with the error flag set when the exit code is non-zero.
- Unknown methods get a proper JSON-RPC error, not a hang.

This is what makes the llama.cpp Web UI integration a one-liner: add `http://host:8888/mcp` under its MCP servers and it discovers your tools on its own.

Because this server also runs unattended in production, the knobs around it matter, and they are the same ones that govern its script-serving role:

| Setting | Effect |
|---------|--------|
| `SERVER_IP_WHITELIST` | Restrict `/mcp` (and everything else) to listed IPs or CIDR ranges |
| `SERVER_SCRIPT_TIMEOUT` | Idle timeout for a tool call's script, 300 s by default |
| `SERVER_MAX_RESPONSE_MB` | Cap on captured tool output, 200 MB by default |
| `MCP_TOOL_EXCLUDE` | Glob patterns for scripts that must not become tools — `helper-*,setup,*.bak` |
| `MCP_SERVER_NAME` | The name advertised in the handshake response |

Be deliberate here: a tool call executes a script with your privileges, and the endpoint answers whatever client can reach the port. Put the whitelist on, put a TLS terminator in front if it crosses a network, and use `MCP_TOOL_EXCLUDE` so nothing auxiliary and dangerous gets advertised as a tool by accident.

---

## Tool filtering: allow and deny, per server, with globs

Discovery is filtered at the source, so a denied tool never reaches the model's tool list at all — it cannot be called by accident because it does not exist as far as the session is concerned.

```
MCP_ALLOW=devkit:mailing_tool_*,git_smart_commit_*;fs:read_*
MCP_DENY=devkit:*shutdown*,*delete*;*:*admin*
```

The semantics are worth reading once, carefully:

- **Both keys take `server:pattern1,pattern2` groups**, semicolon-separated, and the server name may be `*` to match every server.
- **Deny wins.** A tool matching any deny pattern for its server is refused, whatever the allow list says.
- **Allow lists narrow; their absence does not.** If no allow pattern exists for a server, all of its tools pass. If at least one does, a tool must match one of them.
- **Globs are `*` (any run) and `?` (one character)**, evaluated per tool name.

The practical pattern: allow-list the useful surface, deny-list the destructive verbs, and do both per server so one noisy server cannot widen another's exposure. Combined with the per-server class and method filters available in an `.mcp` file, you can express "this server, only these classes, only these methods" in one line.

---

## Comparison matrix

| Harness | Protocol support | Auth for remote servers | Runtime / dependencies | Serves MCP tools | Headless / SSH friendliness | Tool filtering |
|---------|------------------|--------------------------|------------------------|-------------------|------------------------------|----------------|
| **SCORPIOX CODE** | stdio client **and** Streamable HTTP client, MCP protocol version `2024-11-05`; built-in MCP **server mode** over Streamable HTTP | **Native OAuth 2.1 PKCE client**: metadata discovery, dynamic client registration, loopback callback **or paste mode**, cached tokens, automatic refresh on `401` | Pure C99, statically linked, no runtime, no MCP SDK | Yes — `scorpiox-server --mcp`, scripts become tools, schema via `--schema` | Designed for it: paste-mode auth, no browser or desktop required, static binaries | Per-server glob allow **and** deny lists, plus per-server class/method filters in `.mcp` |
| **Claude Code** | stdio, plus `sse`, `http`, and `ws` server types | OAuth for `sse`/`http` types (browser prompt on first use, managed tokens, refresh); token headers for the rest | Node.js CLI; MCP servers are typically `npx` child processes | Yes — a `serve` subcommand runs Claude Code itself as an MCP server | A CLI login with a no-browser stdin redirect exists for SSH, but the day-to-day surface is GUI/IDE-driven | Server-level managed allow/deny policy lists; per-tool globs are not the model |
| **Cursor** | stdio servers spawned by the editor; remote HTTP endpoints configurable in settings | Manual: API keys and header values entered by hand in the editor UI | Electron editor; MCP servers run as child processes of the editor's Node runtime | No — an editor, not a tool server | Weak by construction: it is a desktop IDE | Server-level enable/disable in settings |
| **OpenCode** | stdio, Streamable HTTP, and SSE — via the official TypeScript MCP SDK | OAuth via the same SDK: dynamic registration, PKCE, browser redirect to a local callback server | Bun/TypeScript, SDK plus its dependency tree | Its serve mode is a control-plane HTTP API for the harness, **not** an MCP tools server | CLI commands exist, but auth expects a browser on or near the machine | Per-server enabled switch; no per-tool glob filtering |
| **Hermes Agent** | stdio (with managed `npx`/`uvx` launchers) and Streamable HTTP/SSE, via the official Python MCP SDK | OAuth 2.1 with PKCE through the SDK, browser plus loopback callback, tokens cached per server | Python, SDK as a pinned extra, plus a large dependency tree | Partial — it can expose a curated subset of its own tools to another agent over stdio | CLI-first and SSH-tolerant, with browser-open fallbacks | Per-server `include`/`exclude` globs — the closest peer to SCORPIOX CODE's model |

Read that table the way an operator would, not a marketer:

- **Every harness here can use a local stdio MCP server.** If a vendor's pitch stops there, it is telling you about parity, not advantage.
- **The remote story diverges immediately.** SCORPIOX CODE and Hermes authenticate remote servers with a real OAuth 2.1 PKCE flow; Cursor leans on hand-pasted keys; Claude Code automates OAuth but binds it to its own GUI-era flow. And SCORPIOX CODE is the only one here whose OAuth client is part of the product itself rather than a vendored SDK's behavior you have to reason about from release notes.
- **The runtime row is not cosmetic.** Two of these harnesses ship a second language runtime just to speak MCP, and one more ships an editor. Every CVE in those dependency trees is something you inherit on the day it is disclosed. SCORPIOX CODE's answer to "what does MCP depend on?" is "nothing you do not already ship."

Where the others genuinely lead: Claude Code's managed policy lists are the strongest organization-level control here, and its OAuth automation for its `sse`/`http` types is polished; Hermes's per-server `include`/`exclude` globs match SCORPIOX CODE's filtering model and it adds a curated install catalog on top; OpenCode's SDK-based transport matrix covers SSE explicitly. These are real capabilities. What they have in common is that all of them arrive as someone else's library, versioned and inherited, while the SCORPIOX CODE implementation is part of the product you are already shipping.

---

## Gotchas worth knowing before you deploy

- **stdio MCP is not yet available on Windows.** The stdio client reports this explicitly rather than failing mysteriously; the Streamable HTTP client and the OAuth login are cross-platform from the same release. If your fleet is Windows-first, plan around the HTTP path.
- **Server mode answers `application/json`, not an event stream.** That is fully within the Streamable HTTP contract for request/response traffic, and every MCP client here handles it — but do not expect server-sent streaming from `scorpiox-server --mcp` today.
- **The session id is per command, not per machine.** Because each client invocation runs its own handshake, servers that issue session ids see a fresh session per call. Servers that require long-lived session affinity are the edge case to test first.
- **Large tool outputs are capped.** Remote responses are capped at 4 MB per call, and server-mode tool output is bounded by `SERVER_MAX_RESPONSE_MB`. Design tools that summarize rather than dump.
- **A `401` with no stored token is a hard stop, by design.** Auto-refresh only fires when there is a token to refresh. Run the login once per server per machine and everything after that is automatic.
- **Dynamic registration is not universal.** If a server does not advertise a registration endpoint, the login falls back to a default client identifier; servers that require pre-registered clients with secrets will need that arranged with their operator.
- **The `.mcp` tool filter is strict in one direction.** If a server entry lists tools, only tools matching that list are discovered; a wrong class name or method name silently narrows the surface to nothing, and discovery reports zero tools for that server. A server entry with no tool list exposes everything it has.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — the cascade where `MCP`, `MCP_SERVER`, `MCP_ALLOW`, and `MCP_DENY` live.
- [Skills in SCORPIOX CODE](skills-system.md) — including the built-in skill that teaches the agent to prefer these MCP commands.
- [Event Hooks in SCORPIOX CODE](hooks-system.md) — deterministic side effects around the same lifecycle.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — what model-provider traffic looks like on disk, and why tool calls deserve the same scrutiny.
- [Data Privacy and Zero Data Collection](data-privacy.md) — where credentials and sessions live, and what never leaves your machine.
