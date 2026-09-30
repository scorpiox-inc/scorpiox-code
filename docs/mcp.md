# Native MCP 2.0 & OAuth 2.1 in SCORPIOX CODE: Architecture vs Other Harnesses

You want the agent to reach beyond the tools it ships with — your mail server, your DevKit, a hosted search API, your own scripts on a box in another rack. The Model Context Protocol (MCP) is how that happens in practice, and every agent harness you can name has some answer to it. Most of those answers look the same from the outside and are very different underneath: a Node subprocess tree, a Python virtual environment, a browser-bound login flow, a config file that quietly spawns processes you never inspect.

SCORPIOX CODE takes a different road. MCP is implemented **natively, in the same language as the product** — a JSON-RPC 2.0 client over both stdio and Streamable HTTP, a complete OAuth 2.1 PKCE authorization flow that runs in a terminal, and an MCP server mode that turns a folder of your own scripts into tools any MCP client can call. There is no runtime to install, no SDK to pin, no `node_modules` to audit, and nothing in the auth path that assumes you have a desktop.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** SCORPIOX CODE speaks MCP 2.0 natively — local servers over stdio, remote servers over Streamable HTTP with a full OAuth 2.1 PKCE login (`scorpiox-mcp-login`) that works headless over SSH, token caching and refresh included — and it can *serve* your own scripts as MCP tools with `scorpiox-server --mcp`, all in pure C with zero runtime dependencies.

---

## What "native MCP 2.0" means here

MCP is a JSON-RPC 2.0 protocol with two common transports. A **stdio** transport talks to a local server process over pipes. A **Streamable HTTP** transport sends each JSON-RPC request as an HTTP POST and reads the response back — plain JSON or an event stream — which is what makes remote, authenticated, multi-tenant tool servers possible at all.

SCORPIOX CODE implements both transports itself, plus the authorization layer the remote transport actually requires in the field:

| Piece | What it is | What it talks to |
|-------|------------|------------------|
| `scorpiox-mcp` | Local MCP client over stdio (JSON-RPC 2.0, newline framing) | Server processes you configure, spawned and managed per call |
| `scorpiox-mcp-httpclient` | Remote MCP client over Streamable HTTP | Any MCP 2.0 HTTP endpoint, with or without OAuth |
| `scorpiox-mcp-login` | OAuth 2.1 PKCE authorization client | Any MCP server that authenticates with OAuth |
| `scorpiox-server --mcp` | MCP server mode (Streamable HTTP) | Any MCP client, including other agents and the llama.cpp Web UI |

Three things to hold onto before the details:

- **The protocol negotiation is explicit.** Both the client and the server mode declare MCP protocol version `2024-11-05` during `initialize`, and both reply with their own `serverInfo` name and version, so you always know who you are talking to.
- **The client is stateless per command.** Every `scorpiox-mcp-httpclient` invocation runs initialize, does its work, and exits. There is no daemon to babysit and no half-open connection to time out on you.
- **Session continuity is handled the way the protocol says.** If the server issues an `Mcp-Session-Id`, the client captures it and replays it on every subsequent request in that command.

---

## The four CLIs, in one table

| Command | Mode | Key subcommands |
|---------|------|-----------------|
| `scorpiox-mcp` | stdio client | `discover`, `call`, `list`, `info`, `test` |
| `scorpiox-mcp-httpclient` | Streamable HTTP client | `list`, `call`, `info`, `test` |
| `scorpiox-mcp-login` | OAuth 2.1 PKCE client | full flow, `--force`, `--refresh`, `--status`, `--token`, `--list` |
| `scorpiox-server --mcp` | Streamable HTTP server | `--name`, `-p`, `-r <repo-url>` |

Each one is a standalone static binary. That is not a marketing line; it is the reason they behave the same on a laptop, in a container, and on a headless server with nothing on it but SSH.

---

## Local servers: stdio, configured once

Local MCP servers are defined in `scorpiox-env.txt`, which follows the same configuration cascade as everything else in SCORPIOX CODE — machine defaults, then your project override. A server definition is a name, a command, and its arguments:

```
MCP=1
MCP_SERVER=devkit:/root/devkit/DevKitMCP.Client;fs:npx:-y,@modelcontextprotocol/server-filesystem,/home/you
```

`MCP=1` is the master switch. Then:

```bash
scorpiox-mcp list                     # servers as configured
scorpiox-mcp discover                 # every tool across every server
scorpiox-mcp info devkit send_email   # one tool's JSON schema
scorpiox-mcp test devkit              # start, handshake, tools/list, report
scorpiox-mcp call devkit send_email '{"to":"ops@example.com","body":"build 42 is green"}'
```

`discover` connects to each server in turn, sends `initialize` plus the `notifications/initialized` notification, calls `tools/list`, and prints the result per server. `call` opens a session, sends `tools/call` with your JSON as `arguments`, concatenates the returned text blocks, and propagates the server's `isError` flag as a non-zero exit code so shell scripts and the agent both see the failure.

Three properties worth knowing before you rely on this:

- **The tool contract is enforced, not assumed.** Every tool name is validated against the API's rules (letters, digits, underscore, hyphen, at most 64 characters). Names that are too long are shortened deterministically — a short hash plus the tail of the original name, which is where the meaningful verb usually lives — so a server with sprawling auto-generated names does not silently lose tools.
- **The command never goes through a shell.** The server command is executed directly, and any configured command containing shell metacharacters is rejected up front. Defense in depth, not decoration.
- **Server responses get a generous window.** A stdio server has up to 30 minutes to answer a request, because real tools do slow things sometimes — a long render, a heavy query, a batch job.

### Two config formats, one behavior

Besides `MCP_SERVER`, two other forms are accepted, and they exist because the rest of the ecosystem does not agree on one format:

| Key | Format | Use it when |
|-----|--------|-------------|
| `MCP_SERVER` | `name:command:arg1,arg2` (semicolon-separated for several servers) | You want everything in `scorpiox-env.txt` |
| `MCP_CONFIG` | Standard `mcpServers` JSON file | You already maintain the JSON format other tools use |
| `MCP_FILE` | Line-based `.mcp` file | You are running a named agent that ships its own servers |

The `.mcp` format is the most interesting of the three, because it does per-server work that would otherwise need a second config file:

```
devkit=/root/devkit/DevKitMCP.Client APIKEY__sk-xyz|REGION__eu,MailingTool(send_email:list_recent);GitTool
```

Read that as: server `devkit`, binary path, two environment variables injected into the server's environment (`APIKEY`, `REGION`), and a tool filter — only `send_email` and `list_recent` from the `MailingTool` class, and everything from `GitTool`. Class names are matched case-insensitively against snake_case tool names, so a server that advertises `MailingToolSendEmail` still matches `send_email`.

### How this lands in a session

When you start SCORPIOX CODE with MCP enabled, tools are discovered once at startup and injected into the tool list next to the built-ins. From the model's point of view there is no special category: an MCP tool is a tool, named `mcp__<server>__<tool>` — for example `mcp__devkit__send_email`. The agent calls it, the result comes back through the normal tool-result path, and the call is recorded in the session event stream like any other tool use.

You can manage these tools like any other:

```bash
/disable_tool mcp__devkit__send_email   # blocked for this session, still listed
/enable_tool  mcp__devkit__send_email   # back again
```

Named agent instances get this for free. When an agent directory carries a `.mcp` file, the agent writes `MCP=1` and `MCP_FILE=<path>` into that instance's own project config — instance-scoped MCP without touching your global settings.

---

## Remote servers: OAuth 2.1 with PKCE, designed for terminals

This is the part that most harnesses handle with a browser, a GUI, or a vendored SDK. SCORPIOX CODE does it with one command that runs anywhere a terminal exists.

```bash
scorpiox-mcp-login https://mcp.scorpiox.net
```

The flow follows the MCP authentication specification end to end, seven steps, printed as it goes:

1. **Probe the endpoint.** A minimal `initialize` is POSTed; a `401` response with a `WWW-Authenticate` header carrying a `resource_metadata` URL is the expected, healthy answer. A `200` means the server needs no auth at all, and the command says so and stops.
2. **Read the protected-resource metadata.** This names the authorization server(s) and the scopes that apply to this resource.
3. **Read the authorization-server metadata.** Endpoints, scopes, whether dynamic client registration is offered, whether the `none` token auth method is supported, and whether `S256` PKCE is advertised (you get a clear warning if it is not).
4. **Register a client dynamically.** A `POST /register` with the product name, both loopback redirect URIs (`127.0.0.1` and `localhost`), the authorization-code and refresh-token grant types, and `token_endpoint_auth_method: none` — a public client, no secret to leak. If the server does not offer registration, a default client identifier is used instead.
5. **Generate the PKCE challenge.** 32 bytes from a cryptographically seeded random generator, base64url-encoded into a verifier, SHA-256 hashed into the `S256` challenge, plus a random `state` value.
6. **Authorize.** The URL is printed for you to open in a browser on any device. The client binds a local listener on `http://127.0.0.1:19457/mcp/callback` and waits for the redirect; the `state` parameter is verified on arrival. If the port cannot be bound — a common situation on a remote box with no forwarded loopback — the command says so and drops to **paste mode**: you paste either the code or the full callback URL, and the code is extracted for you. This fallback is what makes the flow work over SSH without any local browser.
7. **Exchange and store.** The code plus the verifier are swapped for tokens at the token endpoint, and the result is saved.

The saved file lives at `~/.mcp/credentials/<sha256-of-url>.json` and contains the URL, client id, access token, refresh token, granted scopes, token endpoint, revocation endpoint, and an absolute expiry time. The directory is created with `0700`.

```bash
scorpiox-mcp-login https://mcp.scorpiox.net --status    # validity, remaining seconds, scopes, file path
scorpiox-mcp-login https://mcp.scorpiox.net --refresh   # new access token, keeps or rotates the refresh token
scorpiox-mcp-login https://mcp.scorpiox.net --token     # print the bearer token to stdout (scripts)
scorpiox-mcp-login --list                               # every saved credential on this machine
scorpiox-mcp-login https://mcp.scorpiox.net --force     # re-authenticate, replacing what is stored
```

Two behaviors make this genuinely low-maintenance rather than merely present:

- **Refresh is automatic in use.** When `scorpiox-mcp-httpclient` receives a `401` with a token attached, it invokes the refresh flow itself, reloads the new token, and retries the request exactly once. You find out only if refresh genuinely failed.
- **Expiry is checked before the request, not after the failure.** A stored token that has already expired is flagged when it is loaded, with the exact command to run, instead of surfacing as an opaque server error later.

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

One built-in skill ships with the product and teaches the agent exactly this: prefer `scorpiox-mcp` for local stdio servers, `scorpiox-mcp-login` plus `scorpiox-mcp-httpclient` for remote ones, and named arguments over hand-built JSON. In practice this means you can ask the agent to "list the tools on that server and call the geocoder on Auckland" and it will reach for the right commands by itself.

---

## Serving your own tools: `scorpiox-server --mcp`

The dual role is the part most harnesses simply do not have. SCORPIOX CODE consumes MCP servers, and it **is** one:

```bash
scorpiox-server --mcp -p 8888                 # serve scripts from the configured folder
scorpiox-server -r https://git.example.com/org/tools.git --mcp   # serve a git-deployed repo
scorpiox-server --mcp --name myserver         # custom serverInfo name
```

Every executable and `.py` script in the served folder becomes a tool; the filename is the tool name. A script that supports a `--schema` flag describes itself in two lines, and the server builds a real JSON schema from it:

```
#!/usr/bin/env python3
# deploy_report --schema prints:
#
# Build a deployment report for an environment.
# environment:string:staging or production:required
# since:string:ISO date, e.g. 2026-09-01:optional
```

At startup the server probes each script with a short timeout, registers what answers, prints the tool list, and then answers MCP traffic on `POST /mcp`:

- `initialize` replies with protocol version `2024-11-05`, a tools capability, and the configured server name (`MCP_SERVER_NAME`, or `--name`, defaulting to `scorpiox-server`).
- `tools/list` returns every registered tool with its schema.
- `tools/call` forks the script, passes the arguments as JSON on stdin, captures stdout, and returns it as text content — with `isError` set when the exit code is non-zero.
- Unknown methods get a proper JSON-RPC error, not a hang.

This is what makes the llama.cpp Web UI integration a one-liner: point it at `http://host:8888/mcp` and it discovers your tools on its own.

Because this server also runs unattended in production, the knobs around it matter, and they are the same ones that govern its script-serving role:

| Setting | Effect |
|---------|--------|
| `SERVER_IP_WHITELIST` | Restrict `/mcp` (and everything else) to listed IPs or CIDR ranges |
| `SERVER_SCRIPT_TIMEOUT` | Idle timeout for a tool call's script, 300 s by default |
| `SERVER_MAX_RESPONSE_MB` | Cap on captured tool output, 200 MB by default |
| `MCP_TOOL_EXCLUDE` | Glob patterns for scripts that must not become tools — `helper-*,setup,*.bak` |

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

The practical pattern: allow-list the useful surface, deny-list the destructive verbs, and do both per server so one noisy server cannot widen another's exposure. Combined with the per-tool `.mcp` filters above, you can express "this server, only these classes, only these methods" in one line.

There is also the session-level lever: `/disable_tool` accepts `mcp__` names verbatim, so you can take a single tool out of a running session without editing config, then bring it back the same way.

---

## What this costs you: the runtime question

This is the part where the comparison stops being about features and starts being about operations.

| | SCORPIOX CODE | Typical Node/Python harness |
|---|---|---|
| Language of the MCP client | Pure C99, statically linked | JavaScript (Node/Bun) or Python |
| Third-party SDK for MCP | None — the protocol is implemented in the product | Official MCP SDK (`@modelcontextprotocol/sdk`, `mcp`, `rmcp`) as a versioned dependency |
| Install surface for MCP | The binaries you already have | An SDK plus its transitive dependency tree |
| Runtime to keep patched | None beyond the binaries | An interpreter plus packages, updated per release |
| Cold start of a tool call | Immediate — process start, connect, request | Interpreter startup before the first byte |
| Protocol upgrades | Rebuild the binary | Bump the SDK, re-test the dependency matrix |

Two concrete risks disappear with the second column. First, **supply chain**: every SDK release you pin is a set of transitive packages you inherit and must watch for advisories; a native implementation has no such list. Second, **environment drift**: a Python MCP stack means virtual environments, interpreter versions, and a package manager that must all agree on the same box — including the remote one you are SSH'd into at 2 a.m. A static binary sidesteps the entire class of problems.

The performance story is the same direction, stated carefully. A stdio tool call is a process start and a pipe round-trip; a Streamable HTTP call is one HTTP POST. Neither pays an interpreter's startup cost first. On long sessions with many tool calls, that difference compounds; on a cold start, it is the difference between the first tool running immediately and waiting for a runtime to boot.

---

## How this compares to other harnesses

The short version: **stdio-local MCP is table stakes** — everyone in this table has it. The differentiators are the transport (who can reach *remote* servers natively), the authorization model (who can complete an OAuth 2.1 flow without a GUI or a heavyweight SDK), whether the harness can *serve* tools as well as consume them, and how precisely you can filter the tool surface. Here is the honest comparison.

| Harness | Protocol support | Auth for remote servers | Runtime / dependencies | Serves MCP tools | Headless / SSH friendliness | Tool filtering |
|---------|------------------|--------------------------|------------------------|-------------------|------------------------------|----------------|
| **SCORPIOX CODE** | stdio client **and** Streamable HTTP client, MCP protocol version `2024-11-05`; built-in MCP **server mode** over Streamable HTTP | **Native OAuth 2.1 PKCE client**: RFC-style discovery, dynamic client registration, loopback callback **or paste mode**, cached tokens, automatic refresh on `401` | Pure C99, statically linked, no runtime, no MCP SDK | Yes — `scorpiox-server --mcp`, scripts become tools, schema via `--schema` | Designed for it: paste-mode auth, no browser or desktop required, static binaries | Per-server glob allow **and** deny lists, plus per-server class/method filters in `.mcp`, plus session-level `/disable_tool` |
| **Claude Code** | stdio (`.mcp.json` `command`/`args`), plus `sse`, `http`, and `ws` server types | OAuth handled for `sse`/`http` types (browser prompt on first use, tokens managed, refresh automatic); token headers for the rest | Node.js CLI; MCP servers are typically `npx` child processes | Yes — `claude mcp serve` runs Claude Code itself as an MCP server | CLI login with a `--no-browser` stdin redirect exists for SSH, but the day-to-day surface is a GUI/IDE flow | **Server-level** managed allow/deny policy lists (`allowedMcpServers` / `deniedMcpServers`); per-tool globs are not the model |
| **Cursor** | stdio servers spawned by the editor; remote HTTP endpoints configurable in settings | Manual: API keys and header values entered by hand in the editor UI | Electron editor; MCP servers run as child processes of the editor's Node runtime | No — an editor, not a tool server | Weak by construction: it is a desktop IDE | Server-level enable/disable in settings UI |
| **OpenCode** | stdio, Streamable HTTP, and SSE — via the official TypeScript MCP SDK | OAuth via the same SDK: dynamic registration, PKCE, browser redirect to a local callback server (default port `19876`) | Bun/TypeScript, SDK plus its dependency tree | Its `serve` mode is a control-plane HTTP API for the harness, **not** an MCP tools server | CLI commands exist (`mcp add/auth/logout`), but auth expects a browser on or near the machine | Per-server `enabled` switch; no per-tool glob filtering |
| **Hermes Agent** | stdio (with managed `npx`/`uvx` launchers) and Streamable HTTP/SSE, via the official Python MCP SDK | OAuth 2.1 with PKCE through the SDK, browser plus loopback callback, tokens cached per server | Python, SDK `mcp` as a pinned extra, plus a large dependency tree | Partial — it can expose a curated subset of its own tools to another agent over stdio | CLI-first and SSH-tolerant, with browser-open fallbacks | **Per-server** `include`/`exclude` globs — the closest peer to SCORPIOX CODE's model |
| **Goose** | stdio and Streamable HTTP, via the official Rust MCP SDK | OAuth flow with browser open and loopback callback through the SDK | Rust core, but the shipped desktop app is Electron (Node) | Server-side SDK capability exists in the ecosystem; the product's role is primarily a client | Desktop-first | Server-level enable/disable; per-extension filtering rather than glob rules |

Read that table the way an operator would, not a marketer:

- **Every harness in this table can use a local stdio MCP server.** If a vendor's pitch stops there, it is telling you about parity, not advantage.
- **The remote story diverges immediately.** SCORPIOX CODE and Hermes authenticate remote servers with a real OAuth 2.1 PKCE flow; Cursor leans on hand-pasted keys; Claude Code automates OAuth but binds it to its own GUI-era flow. And SCORPIOX CODE is the only one here whose OAuth client is part of the product itself rather than a vendored SDK's behavior you have to reason about from release notes.
- **Serving is rare.** Claude Code can put itself on the other side of the protocol. SCORPIOX CODE can serve *your* scripts — a folder or a git repo becomes a tool server any MCP client can consume. OpenCode's server mode is an API for controlling OpenCode, which is a different job.
- **Filtering granularity differs by an order of magnitude.** Server-level enable/disable (Cursor, OpenCode, Goose) versus per-server glob allow/deny (SCORPIOX CODE, Hermes) versus server-level policy lists (Claude Code). If you care about least privilege, this row decides the comparison more than any feature checkbox.
- **The runtime row is not cosmetic.** Two of these harnesses ship a second language runtime just to speak MCP, and one more ships an editor. Every CVE in those dependency trees is something you inherit on the day it is disclosed. SCORPIOX CODE's answer to "what does MCP depend on?" is "nothing you do not already ship."

Where the others genuinely lead: Claude Code's OAuth automation for its `sse`/`http` types is polished and its managed policy lists are the strongest *organization-level* control here; Hermes's per-server `include`/`exclude` globs match SCORPIOX CODE's filtering model and it adds a curated install catalog on top; OpenCode's SDK-based transport matrix covers SSE explicitly. These are real capabilities. What they have in common is that all of them arrive as someone else's library, versioned and inherited, while the SCORPIOX CODE implementation is part of the product you are already shipping.

---

## Gotchas worth knowing before you deploy

- **stdio MCP is not yet available on Windows.** The stdio client reports this explicitly rather than failing mysteriously; the Streamable HTTP client and the OAuth login are cross-platform from the same release. If your fleet is Windows-first, plan around the HTTP path.
- **Server mode answers `application/json`, not an event stream.** That is fully within the Streamable HTTP contract for request/response traffic, and every MCP client here handles it — but do not expect server-sent streaming from `scorpiox-server --mcp` today.
- **The `Mcp-Session-Id` is per command, not per machine.** Because each client invocation runs its own handshake, servers that issue session ids see a fresh session per call. Servers that require long-lived session affinity are the edge case to test first.
- **Large tool outputs are capped.** Remote responses are capped at 4 MB per call, and server-mode tool output is bounded by `SERVER_MAX_RESPONSE_MB`. Design tools that summarize rather than dump.
- **A `401` with no stored token is a hard stop, by design.** Auto-refresh only fires when there is a token to refresh. Run `scorpiox-mcp-login <url>` once per server per machine and everything after that is automatic.
- **Dynamic registration is not universal.** If a server does not advertise a registration endpoint, the login falls back to a default client identifier; servers that require pre-registered clients with secrets will need that arranged with their operator.
- **The `.mcp` tool filter is strict in one direction.** If a server entry lists tools, only tools matching that list are discovered; a wrong class name or method name silently narrows the surface to nothing, and discovery reports zero tools for that server. A server entry with no tool list exposes everything it has.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — the cascade `MCP`, `MCP_SERVER`, `MCP_ALLOW`, and `MCP_DENY` live in.
- [Event Hooks in SCORPIOX CODE](hooks-system.md) — deterministic side effects around the same lifecycle.
- [Skills in SCORPIOX CODE](skills-system.md) — including the built-in skill that teaches the agent to prefer these MCP commands.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — what model-provider traffic looks like on disk, and why tool calls deserve the same scrutiny.
- [Data Privacy and Zero Data Collection](data-privacy.md) — where credentials and sessions live, and what never leaves your machine.
