# Native MCP 2.0 & OAuth 2.1 in SCORPIOX CODE: Architecture vs Other Harnesses

You want the agent to reach beyond the tools it ships with — your mail server, your DevKit, a hosted search API, your own scripts on a box in another rack. The Model Context Protocol (MCP) is how that happens in practice, and every agent harness you can name has some answer to it. Most of those answers look the same from the outside and are very different underneath: a Node subprocess tree, a Python virtual environment, a browser-bound login flow, a config file that quietly spawns processes you never inspect.

SCORPIOX CODE takes a different road. MCP is implemented **natively, in the same language as the product** — a JSON-RPC 2.0 client over both stdio and Streamable HTTP, a complete OAuth 2.1 PKCE authorization flow that runs in a terminal, and an MCP server mode that turns a folder of your own scripts into tools any MCP client can call. There is no runtime to install, no SDK to pin, no `node_modules` to audit, and nothing in the auth path that assumes you have a desktop.

Docs for SCORPIOX CODE @ `655c38f`.

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

- **The protocol negotiation is explicit.** Both the client and the server mode declare the MCP protocol version during `initialize`, and both reply with their own `serverInfo` name and version, so you always know who you are talking to.
- **The client is stateless per command.** Every `scorpiox-mcp-httpclient` invocation runs initialize, does its work, and exits. There is no daemon to babysit and no half-open connection to time out on you.
- **Session continuity is handled the way the protocol says.** If the server issues an `Mcp-Session-Id`, the client captures it from the response headers and replays it on every subsequent request in that command.

---

## The four CLIs, in one table

| Command | Mode | Key subcommands |
|---------|------|-----------------|
| `scorpiox-mcp` | stdio client | `discover`, `call`, `list`, `info`, `test` |
| `scorpiox-mcp-httpclient` | Streamable HTTP client | `list`, `call`, `info`, `test` |
| `scorpiox-mcp-login` | OAuth 2.1 PKCE client | full flow, `--force`, `--refresh`, `--status`, `--token`, `--list` |
| `scorpiox-server --mcp` | Streamable HTTP server | `--name`, `-p`, `-r <repo-url>` |

Each one is a standalone binary. That is not a marketing line; it is the reason they behave the same on a laptop, in a container, and on a headless server with nothing on it but SSH.

---

## Local servers: stdio, configured once

MCP over stdio is the original transport, and SCORPIOX CODE speaks it directly — no SDK, no wrapper process, no interpreter.

```
MCP=1
MCP_SERVER=devkit:/root/.claude/devkit-linux-x64
MCP_SERVER=devkit:/root/.claude/devkit-linux-x64;fs:npx:-y,@anthropic/mcp-filesystem,/home
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

That one line runs the entire MCP authentication specification (2025-03-26) end to end:

1. **Probe the endpoint.** A `POST` to the MCP endpoint without credentials. A `401` plus a `WWW-Authenticate` header tells the client the auth is required and points it at the authorization servers.
2. **Discover the resource metadata.** `GET /.well-known/oauth-protected-resource` yields the resource identifier and the list of authorization servers.
3. **Discover the server metadata.** `GET /.well-known/oauth-authorization-server` yields the authorization, token, and registration endpoints, the supported scopes, and whether `S256` PKCE is advertised (it is checked, and a warning is raised if the server does not list it, though the flow continues because most servers accept it anyway).
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
scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp call websearch --file params.json
echo '{"query":"llama.cpp"}' | scorpiox-mcp-httpclient https://mcp.scorpiox.net/mcp call websearch
```

Each command performs the full handshake itself: `initialize`, the initialized notification, then your request, then exit. The `test` command is the fastest way to check a server — it reports the negotiated protocol version, the server's own name and version, the session id if one was issued, and the tool count. Requests accept plain JSON, `key=value` named arguments, a `--file` payload for large or nested inputs, or piped stdin. Responses arrive as plain JSON or as an event stream; both are parsed, so you do not have to know which one a given server speaks.

---

## Serving your own tools: `scorpiox-server --mcp`

Most harnesses can only *consume* MCP. SCORPIOX CODE can also *produce* it. Point `scorpiox-server` at a folder of scripts — or at a git repository — and it runs an MCP server over Streamable HTTP:

```bash
scorpiox-server --mcp -p 8888                    # serve ./scripts (and nested dirs)
scorpiox-server -r https://git.example.com/me/tools.git --mcp -p 8888
scorpiox-server --mcp --name myserver -p 8888
```

At startup it walks the script directories and, for every native binary or Python script it finds, runs the script with a `--schema` flag (five-second timeout) and parses the resulting JSON into a proper MCP tool definition — name, description, and parameter schema. Anything without a schema still registers as a tool with a generic description.

The wire protocol it serves is exactly what any MCP client expects:

- `initialize` replies with the protocol version, a tools capability, and the configured server name (`MCP_SERVER_NAME`, or `--name`, defaulting to `scorpiox-server`).
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

## Head to head: SCORPIOX CODE vs the other harnesses

This is the comparison that matters operationally. The feature list looks convergent — everyone speaks MCP — but the implementations are not.

| Capability | SCORPIOX CODE | Claude Code | Cursor | OpenCode | Hermes Agent |
|---|---|---|---|---|---|
| **Transport** | Streamable HTTP (JSON-RPC 2.0) + stdio | stdio subprocesses | stdio subprocesses | stdio (Bun subprocess) | stdio subprocesses |
| **Remote MCP over HTTP** | Native client, no SDK | Via SDK / wrappers | Via SDK / wrappers | Via SDK / wrappers | Via SDK / wrappers |
| **Auth** | Native OAuth 2.1 PKCE (RFC 8414 metadata, dynamic registration, token cache + refresh) | Browser-based OAuth / API keys | Browser-based / API keys | Browser-based / API keys | API keys / manual |
| **Runtime for MCP** | Pure C99, statically linked, zero dependencies | Node.js + `@modelcontextprotocol/sdk` | Node.js + SDK (Electron app) | Bun runtime + SDK | Node.js / Python |
| **Install surface** | Binaries already present | `node_modules` tree + SDK version matrix | App bundle + SDK tree | Bun install + package tree | Interpreter + packages |
| **Headless / SSH login** | Yes — paste-mode fallback, no GUI required | Requires a reachable browser | Requires a GUI browser | Requires a reachable browser | Depends on backend |
| **Token lifecycle** | Auto-detected expiry, auto-refresh, single-file credential store | SDK-managed | SDK-managed | SDK-managed | Manual |
| **MCP server mode** | Built-in: `scorpiox-server --mcp`, scripts become tools | Not first-class | Not first-class | Not first-class | Not first-class |
| **Tool filtering** | Per-server glob allow/deny, per-tool `.mcp` class/method filters | Tool list in config | UI toggles | Tool list in config | Config-based |
| **Cold start of a tool call** | Process start, connect, request | Interpreter boot + subprocess spawn | Interpreter boot + subprocess spawn | Runtime boot + subprocess spawn | Interpreter boot + subprocess spawn |

### Why the architecture differences are not academic

**Zero runtime bloat.** A C99 binary that implements the protocol itself has no transitive dependency tree, no SDK version pin, and no package manager to coordinate across machines. The failure modes that dominate Node and Python MCP setups in production — a transitive dependency bump that breaks the handshake, a `node_modules` tree that differs between the dev laptop and the CI box, a venv that was rebuilt with a different interpreter — disappear because there is nothing to rebuild.

**Streamable HTTP is the transport the protocol actually moved to.** The old stdio-only story works when the server is a process you own on the same machine. Enterprise tool servers — hosted search, hosted DevKits, shared internal APIs — are HTTP endpoints, usually authenticated, usually multi-tenant. A stdio-only harness needs a local proxy process per tool, which means more processes to supervise, more env vars to thread, and a local process as the trust boundary. SCORPIOX CODE talks to the endpoint directly with a bearer token, so the trust boundary sits where it belongs.

**OAuth 2.1 PKCE, not pasted API keys.** The login flow performs metadata discovery, dynamic client registration, PKCE `S256`, state verification, and refresh — the full spec, in one command. It has no secret to store (a public client with `none` auth at the token endpoint), and the only artifact on disk is the credential file with its `0700` directory. The paste-mode fallback means a bare SSH session is a complete environment: no desktop, no forwarded browser, no GUI process.

**Dual role.** Consuming tools is half the job. `scorpiox-server --mcp` makes SCORPIOX CODE the other half of the ecosystem: your scripts become tools for *any* MCP client, including the other harnesses in this table. That is a position none of the Node-based harnesses can take, because they cannot serve what they cannot speak.

**Granular security.** `MCP_ALLOW` / `MCP_DENY` with per-server glob patterns, plus the `.mcp` class-and-method filters, mean the model sees exactly the surface you intend. All-or-nothing tool lists are fine for a personal laptop; they do not survive contact with a production tool server that exposes shutdown verbs.

**Latency and cold boot.** A static binary executes its first request as soon as it is running. An interpreter-backed client pays for its interpreter boot, its module graph, and its subprocess spawn on every cold invocation. Sub-millisecond against a couple of seconds is not a benchmark artifact; it is the difference between a tool call that feels like a keystroke and one that feels like a startup.

---

## Quick reference

| Goal | Command / setting |
|------|-------------------|
| Enable MCP in the agent | `MCP=1` in `scorpiox-env.txt` |
| Add a local stdio server | `MCP_SERVER=name:command:arg1,arg2` |
| Import standard MCP config | `MCP_CONFIG=/path/to/mcp_servers.json` |
| Per-agent servers | `.mcp` file in the agent directory |
| Discover local tools | `scorpiox-mcp discover` |
| Call a local tool | `scorpiox-mcp call <server> <tool> <json>` |
| Log in to a remote MCP server | `scorpiox-mcp-login https://mcp.scorpiox.net` |
| Check token status | `scorpiox-mcp-login <url> --status` |
| Refresh / print token | `scorpiox-mcp-login <url> --refresh` / `--token` |
| List saved credentials | `scorpiox-mcp-login --list` |
| Test a remote server | `scorpiox-mcp-httpclient <url> test` |
| List / call remote tools | `scorpiox-mcp-httpclient <url> list` / `call <tool> <json>` |
| Serve local scripts as MCP | `scorpiox-server --mcp -p 8888` |
| Filter tools | `MCP_ALLOW=server:glob` / `MCP_DENY=server:glob` |
| Exclude scripts from server mode | `MCP_TOOL_EXCLUDE=helper-*,setup` |
| Restrict the server mode | `SERVER_IP_WHITELIST=...` |
| Block one tool in a session | `/disable_tool mcp__<server>__<tool>` |

---

## See also

- [Anthropic Provider](anthropic-provider.md)
- [Claude Code Provider](claude-code-provider.md)
- [Docker](docker.md)
- [Hooks System](hooks-system.md)
