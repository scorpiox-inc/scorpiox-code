# Serving Sites with scorpiox-server

`scorpiox-server` is SCORPIOX CODE's built-in HTTP server. Point it at a folder — or at a git repository — and every file in it becomes a route: a Python script or a compiled native executable that answers a URL. There is no framework to bootstrap, no runtime to install beside it, and no separate web root to keep in sync. The routing *is* the file layout, and a deploy is a `git push`.

This page is the operator's guide: how routes resolve, what a script receives, how the HTML-plus-JSON contract works, how to deploy from git, every configuration key and default, JWT authentication, MCP mode, and the reverse-tunnel mesh that lets a node behind NAT join a hub without opening a port.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** one zero-dependency native binary serves Python and compiled handlers as web routes, deploys them by git push, enforces JWT auth in the server itself, and can expose the same scripts as MCP tools or tunnel them out through a hub — where the routing, the deployment, and the auth are the file layout, one flag, and one secret.

---

## What it is, and what it is not

`scorpiox-server` is a small, self-contained HTTP server. It listens on a port, accepts requests, and for each request decides which script should answer it, runs that script as a child process, and streams the script's output back as the HTTP response. A handler can be:

- a **Python script** — `name.py`, run with the system `python3` interpreter; or
- a **native executable** — `name` on Linux and macOS, `name.exe` on Windows.

Native handlers are checked first, then Python scripts. Because each handler is a process, a slow or crashed route never takes the server down with it, and one route's memory is not another's.

What it is **not**: it is not a reverse proxy, a load balancer, or a TLS terminator. It speaks plain HTTP and expects a TLS terminator to sit in front of it when a route crosses a network. It also does not ship a template language, ORM, or session store — those are the script's business. The server's job is routing, process execution, the request environment, and access control.

---

## Routes: the file layout is the routing table

A request path is mapped to a handler name under the route prefix. With the default prefix `/api/platform/websites/`:

| Request | Handler |
|---------|---------|
| `GET  {prefix}foo` | runs `foo` (native) or `foo.py` |
| `POST {prefix}foo` | same handler, with the request body available |
| `GET  {prefix}` | runs `index` or `index.py` (the default page) |
| any unmatched path | runs `_fallback` or `_fallback.py`, if present |
| `GET  /favicon.ico` | the configured favicon, if one is set |
| `GET  /api/ping` | health check — returns `ok` |
| `GET  /api/otp?a=NAME&s=SECRET` | generates a TOTP (JSON) |
| `POST /mcp` | MCP tools endpoint, when MCP mode is on |
| `GET  /ws/join`, `GET /mesh/nodes`, `/node/<id>/…` | the mesh hub, when running as a hub |

The route prefix is `SERVER_ROUTE_PREFIX`, default `/api/platform/websites/`. Set it to `/` for clean site URLs at the domain root. **In git deploy mode the prefix defaults to `/`** unless you configure one explicitly, so a pushed site answers at its own root out of the box.

Handler names may contain letters, digits, `-`, `_`, and `.`; a name containing `..` is refused as a traversal attempt. When several script directories are configured they are searched in order and the **first match wins**; if the same handler name exists in two directories the server prints a conflict warning at startup so you are never surprised about which one is live.

The HTTP method itself is delivered to the handler in `REQUEST_METHOD`, so a script can branch on it. The documented site convention is GET for reads and POST for writes. An `OPTIONS` request is handed to a handler named `options` (or to `_fallback`) when one exists — useful for CORS preflight — and answered directly with `204 No Content` when none does. Every response carries permissive CORS headers, so a browser client on another origin can call the site without extra configuration.

---

## The HTML + JSON contract

Every page a site serves should answer **two audiences through one URL**: a browser that wants HTML and a machine that wants structured data. The convention is a query parameter — add `?format=json` and the same route returns JSON instead of HTML.

This is not a server switch; it is a design requirement of the sites you build on the server. Query parameters arrive at the handler as environment variables named after the key, so `?format=json` reaches the script as the variable `format` with the value `json`, and the script decides what to emit. That is what makes a route usable by a human in a browser and a model in a loop without a second endpoint or a second handler.

The same pattern is used throughout SCORPIOX CODE's own web surfaces: the fleet dashboard and its API speak one dialect where every route accepts `?format=json`, so a browser and a bot can hit the identical URL. Treat it as mandatory for your own pages too — it costs one `if` in the handler and buys a machine-readable mode for free.

Two mechanics make it work cleanly:

- **Query parameters are exported as environment variables** named after the key (`?name=x` → `name=x`). Only safe names are exported: letters, digits, and underscores, not starting with a digit, containing at least one lowercase letter, and never a proxy variable or an existing variable. A request cannot use a query string to overwrite `PATH`, a proxy setting, or anything the operator set.
- **The handler emits CGI-style headers** at the top of its output — `Status:`, `Content-Type:`, and any others (`Location`, `Set-Cookie`) — followed by a blank line and the body. If no `Content-Type` is given, the response is `text/html`.

So a handler that serves a human page and a JSON API looks, in outline, like this:

```python
import os, json
fmt = os.environ.get("format", "html")
if fmt == "json":
    print("Content-Type: application/json")
    print()
    print(json.dumps({"status": "ok"}))
else:
    print("Content-Type: text/html")
    print()
    print("<h1>Hello</h1>")
```

---

## What a handler receives

A handler runs as a child process with the working directory set to its script directory, and with a deliberately narrow environment. That environment is where the request arrives.

| Variable | Meaning |
|----------|---------|
| `REQUEST_METHOD` | `GET`, `POST`, and so on |
| `QUERY_STRING` | the raw query string |
| `PATH_INFO` | the request path |
| `CONTENT_TYPE` | the request `Content-Type` |
| `CONTENT_LENGTH` | the request body length, when known |
| `HTTP_COOKIE` | the raw `Cookie` header |
| `HTTP_AUTHORIZATION` | the raw `Authorization` header |
| `HTTP_*` | every other request header, CGI-style (`X-Custom` → `HTTP_X_CUSTOM`) |
| *`key=value`* | each safe query parameter, named after its key |
| `POST_BODY_FILE` | path to a temp file holding a small POST body (see below) |
| `SX_STREAMING` | `1` when the response is being streamed (see below) |

**Request bodies.** A small POST body (under 512 KB) is written to a temporary file and passed as `POST_BODY_FILE`, so a handler reads it like any file and the body never sits in a shell. A large body — over 512 KB, or any chunked body — is streamed to the child's standard input in chunks and never buffered whole, so a big upload does not consume memory proportional to its size. On the streaming path there is no `POST_BODY_FILE`; the script reads standard input instead, and `SX_STREAMING=1` is set.

**Responses.** The server reads the handler's standard output, parses the leading headers, and forwards them with the body. If the handler streams a response (an MCP tool call, a large answer, a server-sent event stream), the server relays it as it is produced. A handler that emits `Content-Type: text/event-stream` gets live server-sent events relayed to the browser with caching disabled — no polling, no buffering.

**Timeouts.** A handler has an idle timeout — `SERVER_SCRIPT_TIMEOUT`, default **300 seconds** — measured as time with no output, not total runtime. A script that keeps producing output keeps running; one that goes quiet for the full window is reaped.

---

## Deploying from git

The fastest way to ship a site is to let the server own a clone of your repository:

```bash
scorpiox-server -r https://git.example.com/team/site.git
scorpiox-server -r https://git.example.com/team/site.git -b staging     # track a branch
scorpiox-server -r https://git.example.com/team/site.git --poll 5        # poll every 5s
```

At startup the server clones the repository into its cache and serves the working tree. It then polls in the background: it asks the remote what the branch head is, compares that to the local head, and when they differ it fetches and hard-resets to the new commit. **Your deploy is a `git push`** — the server picks up the change on the next poll.

| Key / flag | Default | Meaning |
|------------|---------|---------|
| `-r`, `--repo <url>` | *(off)* | Clone and serve from this repository |
| `-b`, `--branch <name>` | `main` | Branch to track |
| `--poll <seconds>` | `10` | Poll interval (`SERVER_GIT_POLL_INTERVAL`) |
| `SERVER_GIT_PAT` | *(empty)* | Personal access token for private clones/fetches |
| `SERVER_GIT_CACHE_DIR` | platform cache dir | Where the clone lives |

When `SERVER_GIT_PAT` is set it is embedded into HTTPS clone and fetch URLs so private repositories work without an interactive credential prompt. If the token would appear in a log line it is masked. The cache directory defaults to `$XDG_CACHE_HOME/scorpiox-server`, then `~/.cache/scorpiox-server`, and finally a private per-user path under the system temp directory; the clone itself lands in `<cache>/<repo-name>`. Clones are shallow — depth 1 — which keeps startup fast; the poll loop brings you forward, not backward.

Note the routing default in this mode: unless you set `SERVER_ROUTE_PREFIX` yourself, git deploy mode serves at `/`, so a repository whose top level holds `index.py`, `about.py`, and `style.css` answers as a normal website.

---

## Configuration reference

Every setting below is a `KEY=VALUE` pair resolved through the standard SCORPIOX CODE configuration cascade — defaults, then a global file, then your user file, then the project file, then the active profile, then the OS environment. See [Configuration and Profiles](scorpiox-env.md) for how the tiers combine and which file to edit.

### Server core

| Key | Default | Description |
|-----|---------|-------------|
| `SERVER_PORT` | `8080` | Listening port. The `-p` flag overrides it. |
| `SERVER_ROUTE_PREFIX` | `/api/platform/websites/` | Prefix stripped before a path becomes a handler name. Git deploy mode defaults to `/`. |
| `SERVER_SCRIPT_DIR` | `./scripts` | Comma-separated handler directories, up to 16. First match wins. |
| `SERVER_STATIC_ROOT` | *(off)* | Serve unmatched GET/HEAD paths as static files from here. Handlers always take priority. |
| `SERVER_MAX_REQUEST_MB` | `200` | Largest request body accepted, streamed rather than buffered. |
| `SERVER_MAX_RESPONSE_MB` | `200` | Largest buffered response body. |
| `SERVER_SCRIPT_TIMEOUT` | `300` | Handler idle timeout, in seconds with no output. |
| `SERVER_STREAM_SCRIPTS` | *(off)* | Comma-separated handler names whose responses are streamed with no size cap (idle timeout only). Unix only. |
| `SERVER_IP_WHITELIST` | *(off)* | Comma-separated IPs and CIDR ranges. Empty allows every client. |
| `SERVER_FAVICON` | *(off)* | A base64 favicon or a path to an image file, served at `/favicon.ico`. |

The server binds all interfaces and handles requests with a forked child per request on Linux and macOS, and a thread per request on Windows. The whitelist, when set, is checked against the first client address it can find — `X-Forwarded-For` first, then `X-Real-IP`, then the socket peer — so it works correctly behind a proxy that sets either header; once a header is present its address is authoritative, so a client cannot bypass the list by also connecting from a trusted peer. An entry is a plain address (`203.0.113.7`) or a range (`10.0.0.0/8`). A blocked request gets `403 Forbidden`.

Static serving, when `SERVER_STATIC_ROOT` is set, resolves directories to `index.html`, rejects any path containing `..` or resolving outside the root (including through a symlink), and sets `Cache-Control: no-store` so a git-pushed site never serves a stale asset. Handlers always win over static files, so turning this on cannot shadow an existing route.

### Git deploy

| Key | Default | Description |
|-----|---------|-------------|
| `SERVER_GIT_CACHE_DIR` | platform cache dir | Directory holding the clone. |
| `SERVER_GIT_POLL_INTERVAL` | `10` | Poll interval in seconds. |
| `SERVER_GIT_PAT` | *(empty)* | Token embedded into HTTPS clone and fetch URLs. |

### Authentication

| Key | Default | Description |
|-----|---------|-------------|
| `SERVER_JWT_SECRET` | *(empty)* | HMAC-SHA256 secret. A leading `/` or `./` is treated as a file path; anything else is an inline secret. Empty disables server-side JWT validation. |
| `SERVER_JWT_ISSUER` | *(empty)* | Expected `iss` claim. Empty skips the check. |
| `SERVER_JWT_AUDIENCE` | *(empty)* | Expected `aud` claim. Empty skips the check. |
| `SERVER_JWT_COOKIE` | *(empty)* | Cookie name to read a token from, after the `Authorization` header. Empty means header-only. |
| `SERVER_JWT_CLAIMS` | *(empty)* | Extra `claim=ENV_VAR` mappings, comma-separated. |
| `SERVER_JWT_PROTECT` | *(empty)* | Comma-separated route prefixes that require a valid token. Empty means no enforcement. |
| `SERVER_JWT_PUBLIC` | *(empty)* | Comma-separated prefixes exempt from `SERVER_JWT_PROTECT`. Matches whole path segments. |
| `SERVER_JWT_LOGIN_URL` | *(empty)* | Where a browser hitting a protected route is redirected when unauthenticated. |

### MCP server mode

| Key | Default | Description |
|-----|---------|-------------|
| `MCP_SERVER_NAME` | `scorpiox-server` | Name advertised in the MCP handshake. The `--name` flag overrides it. |
| `MCP_TOOL_EXCLUDE` | *(empty)* | Comma-separated glob patterns for handlers that must not become MCP tools. |
| `SERVER_SCRIPT_TIMEOUT` | `300` | Also the idle timeout for one tool call. |
| `SERVER_MAX_RESPONSE_MB` | `200` | Also caps captured tool output. |

### Mesh and hub

| Key | Default | Description |
|-----|---------|-------------|
| `SERVER_MESH_HUB` | `0` | Run as a mesh hub (equivalent to `--hub`). |
| `SERVER_MESH_CONNECT` | *(empty)* | Worker hub URL to dial (equivalent to `--connect`). |
| `SERVER_MESH_ID` | hostname | Node id advertised to the hub. |
| `SERVER_MESH_KEY` | *(empty)* | Pre-shared mesh key for a self-hosted hub. |
| `SERVER_MESH_TOKEN` | *(empty)* | Scorpio+ JWT for the worker. |
| `SERVER_MESH_CA_FILE` | *(system store)* | CA bundle for `wss://` hub URLs. |
| `SERVER_MESH_INSECURE` | `0` | Skip TLS certificate verification (equivalent to `--insecure`). |
| `SERVER_MESH_TIMEOUT` | `300` | Tunnel request timeout, in seconds. |

### Command-line reference

| Flag | Effect |
|------|--------|
| `-h`, `--help` | Print the built-in help. |
| `-p <port>` | Listen on this port. |
| `-e KEY=VALUE` | Override any configuration key for this run (repeatable). |
| `-r`, `--repo <url>` | Enable git deploy mode. |
| `-b`, `--branch <name>` | Track this branch (default `main`). |
| `--poll <seconds>` | Git poll interval. |
| `--mcp` | Serve handlers as MCP tools over HTTP. |
| `--name <name>` | MCP server name. |
| `--hub` | Run as a mesh hub. |
| `--connect`, `--join <url>` | Run as a mesh worker dialing this hub. |
| `--id <node-id>` | Mesh node id. |
| `--key <psk>` | Pre-shared mesh key. |
| `--token <sx_token>` | Scorpio+ JWT for the worker. |
| `--insecure` | Skip TLS verification for a `wss://` hub. |

A value that contains exactly two dots is treated as a JWT; anything else passed as `--token` is routed to the mesh key slot with a warning, so a mistyped pre-shared key never silently becomes a token. When no token is given explicitly, the worker falls back to `SX_TOKEN`, then to the stored SCORPIO+ credential at `~/.scorpiox/auth.json`.

---

## JWT authentication in the server

Set `SERVER_JWT_SECRET` and the server validates a signed JWT on each request, before any handler runs. Validation is HMAC-SHA256, the algorithm a browser login and a SCORPIO+ token both use. The secret is either an inline string or a path to a file — a file is read once at startup and trailing whitespace is stripped, which is the right shape for a secret mounted as a file in a container.

The token is read from the `Authorization: Bearer` header first, and from a cookie named by `SERVER_JWT_COOKIE` when configured. Issuer and audience checks run only when you set them; leaving `SERVER_JWT_ISSUER` or `SERVER_JWT_AUDIENCE` empty deliberately accepts any issuer or audience.

**What a handler learns about the caller.** A validated token becomes environment variables, so a handler never parses a token itself:

| Variable | Source |
|----------|--------|
| `X_AUTHENTICATED` | `1` when a valid token was presented, `0` otherwise |
| `X_USER_ID` | the `sub` claim (falling back to `nameid`) |
| `X_USER_EMAIL` | the `email` claim |
| `X_JWT_RAW` | the full decoded payload as JSON |
| *custom* | every claim named in `SERVER_JWT_CLAIMS` |

`SERVER_JWT_CLAIMS` maps claim names to variable names — `permissions=X_USER_PERMISSIONS,role=X_USER_ROLE,name=X_USER_NAME` — and array claims are joined with commas, so `["admin","git"]` arrives as `admin,git`. A handler can branch on that without a second lookup.

**Enforcement.** `SERVER_JWT_PROTECT` lists route prefixes that require a valid token (`/admin,/api/private,/dashboard`). `SERVER_JWT_PUBLIC` exempts prefixes from that requirement, matching whole path segments — `/assets` covers `/assets` and `/assets/x` but not `/assetsX` — so you can protect `/` while still serving static assets and a health check. A request to a protected route without a valid token gets JSON `{"error":"unauthorized"}` with status 401; if the client sent `Accept: text/html` and you set `SERVER_JWT_LOGIN_URL`, it gets a 302 redirect there instead. A *valid* token on a public route is still validated and still exported to the handler — it just is not required.

Enforcement is off by default. With no `SERVER_JWT_PROTECT`, the server validates and exports identity but lets every request through, and each script decides what to do about it.

---

## MCP mode: your scripts as tools

Add `--mcp` and the server also answers MCP JSON-RPC 2.0 on `POST /mcp`, exposing the handlers in its directories as tools:

```bash
scorpiox-server --mcp -p 8888                 # serve this folder's scripts as tools
scorpiox-server -r https://git.example.com/ops.git --mcp   # serve a repo's scripts as tools
scorpiox-server --mcp --name ops-server       # custom advertised name
```

Each handler becomes a tool named after its file. A handler can describe itself by answering a `--schema` probe: the first line is the tool description, and each following line declares a parameter as `name:type:description:required|optional`. At startup the server probes each handler with a short timeout, registers the ones that answer, and prints the resulting tool list, then serves MCP traffic:

- `initialize` replies with protocol version `2024-11-05`, a tools capability, and the advertised name (`MCP_SERVER_NAME`, or `--name`, default `scorpiox-server`).
- `tools/list` returns every registered tool with its input schema.
- `tools/call` runs the handler, passes the call arguments as JSON on standard input, captures standard output, and returns it as text content — with the error flag set when the handler exits non-zero.
- An unknown method gets a proper JSON-RPC error rather than a hang.

Tool names are restricted to letters, digits, `-`, and `_`, names beginning with `_` are skipped (so `_fallback` is never advertised), and `MCP_TOOL_EXCLUDE` takes comma-separated globs — `helper-*,setup,*.bak` — so auxiliary scripts never leak into a model's tool list.

Because a tool call runs a handler with the server's privileges and the endpoint answers whatever client can reach the port, treat MCP mode as production surface. Keep `SERVER_IP_WHITELIST` on, put a TLS terminator in front if it crosses a network, and exclude anything that is not meant to be called. This is the server side of SCORPIOX CODE's native MCP support; see [Native MCP 2.0 and OAuth 2.1](mcp.md) for the client side and the full comparison.

---

## The mesh: serving a machine behind NAT

A **hub** accepts outbound connections from **workers** and routes requests to them, so a worker needs no inbound port and no public address. This is the transport SCORPIOX BOT uses for fleet nodes behind NAT, and the same idea works for any site.

Run a hub:

```bash
scorpiox-server --hub -p 8080
```

Run a worker that dials it:

```bash
scorpiox-server --connect wss://hub.example.com/ws/join --id branch-office
scorpiox-server --join ws://hub.local:8080/ws/join --id lab-2 --key <psk>
```

A worker connects out through a WebSocket reverse tunnel and reports itself to the hub under its node id (the hostname by default). The hub then exposes three ingress surfaces:

| Surface | Purpose |
|---------|---------|
| `GET /ws/join?id=<id>&token=<token>` | worker handshake |
| `GET /mesh/nodes` | JSON registry of live nodes |
| `/node/<id>/<path>` | route a request to that node's own handlers |

Authentication on a hub is chosen automatically from its configuration, and can be combined:

- **SCORPIO+** — set `SERVER_JWT_SECRET` and `/ws/join`, `/mesh/nodes`, and `/node/<id>/` all require a valid signed token. Node ids are bound to the `sub` of the token that first claimed them, and one tenant cannot see or reach another's nodes; cross-tenant access is refused. This is the multi-tenant mode.
- **Pre-shared key** — `--key` or `SERVER_MESH_KEY` gates `/ws/join` (and nothing else), which is the right shape for a self-hosted, single-tenant hub.
- **Open** — neither is set; everything is reachable. Fine on a trusted LAN, wrong on the internet.

The hub strips any client-supplied `X-Mesh-*` header and injects its own `X-Mesh-Node`, `X-Mesh-User`, and `X-Mesh-Email` before handing the request to the node, so a handler sees trustworthy mesh identity as `HTTP_X_MESH_*`. Errors are consistent JSON with a status you can act on — `400` for a missing id, `401` unauthorized, `403` wrong key or forbidden node, `409` a node id owned by someone else, `502` a node offline, `504` a tunnel timeout.

A worker reconnects on its own with exponential backoff up to 30 seconds; an authentication failure jumps straight to the maximum delay, because retrying fast cannot fix a bad credential. The hub handles each tunnelled request on its own thread, so a long-lived stream on one node never blocks the registry or another node's traffic. Worker URLs may be `ws://`/`http://` for plaintext or `wss://`/`https://` for TLS, with `SERVER_MESH_CA_FILE` supplying a custom CA bundle and `--insecure` disabling verification for a test hub. The hub itself speaks plaintext — put a TLS terminator such as Caddy in front of it.

---

## How it compares to the mainstream servers

The stacks an operator would otherwise reach for are all capable, and all ask you to assemble a deployment around them. Here is the honest shape of the difference.

| | What it is | Runtime you install | Where routing lives | How you deploy | Built-in auth | Machine-readable output |
|--|-----------|--------------------|--------------------|----------------|---------------|-----------------------|
| **IIS** | Windows web server | IIS role and its feature modules | `web.config` and handler mappings | MSDeploy, file copy, or a pipeline | Windows auth / forms modules, separate config | Application's job |
| **Apache httpd** | Cross-platform web server | Apache plus modules (and mod_wsgi/mod_proxy to reach an app) | `.htaccess` and `httpd.conf` | File copy and reload | `mod_auth*` modules | Application's job |
| **nginx** | Edge server and reverse proxy | nginx, often plus an app server behind it | `nginx.conf` location blocks | File copy and reload | `auth_request` / external | Application's job |
| **ASP.NET Core Kestrel** | Application web server | .NET runtime and SDK tooling | Attribute/endpoint routing in compiled C# | `dotnet publish` and a service | ASP.NET authentication middleware | You write the JSON endpoints |
| **`scorpiox-server`** | Handler server | Nothing beyond the binary (and `python3` for Python handlers) | The file layout, one directory per route | `git push`, then poll | JWT enforced in the server | Contract on every page via `?format=json` |

Read the table as an operator, not a marketer:

- **Every stack above can serve a site.** The difference is what you must install and wire before the first route answers. IIS and Apache want modules and configuration languages; nginx usually wants an application server behind it; Kestrel wants a .NET runtime and toolchain. `scorpiox-server` wants a port and a folder.
- **Routing is the sharpest contrast.** In the mainstream stacks the routing table is configuration you maintain separately from the code. Here each route is a file, so adding a page is adding a file and deploying is pushing — nothing to keep in sync, no mapping to drift.
- **Deployment is the second sharpest.** A push-and-poll deploy removes the copy-and-reload step, the build step for native handlers, and the pipeline glue in between. You edit, you commit, you push, and the route is live within one poll interval.
- **Auth and the JSON contract are built in, not bolted on.** JWT validation, claim-to-variable export, and route protection live in the server; and the HTML-plus-JSON convention is a stated requirement of your pages rather than a second API you build beside them. Neither is a plugin you select and configure.

If you need upstream load balancing, HTTP/2 edge termination, or a mature module ecosystem, keep nginx or IIS at the edge and let `scorpiox-server` handle the application routes behind it. If you want a single native binary where the route is the file and the deploy is a push, that is exactly what this is.

---

## Gotchas

- **The prefix surprises people in both directions.** The default `/api/platform/websites/` is not what a public site wants; set `SERVER_ROUTE_PREFIX=/` for root URLs. In git deploy mode the default already flips to `/`, but an explicit prefix in your config wins.
- **Handlers always beat static files.** If a static asset and a handler share a name, the handler is what answers. Keep static assets under paths no handler claims.
- **A handler responds on any method it receives.** The method arrives in `REQUEST_METHOD`; the documented convention is GET and POST, but do not assume the server rejects other verbs for you — validate in the script if it matters.
- **Empty versus absent, again.** A key set to an empty value in a higher configuration tier still shadows a lower tier; an empty OS environment variable is ignored. See the configuration guide for the exact rule.
- **`SERVER_STREAM_SCRIPTS` is Unix only.** On Windows it is ignored with a warning, and large responses are still bounded by `SERVER_MAX_RESPONSE_MB`.
- **A streamed response has no size cap but an idle timeout.** It is bounded by `SERVER_SCRIPT_TIMEOUT` (no output), not by the response limit. A quiet stream is reaped.
- **JWT enforcement is opt-in.** Setting a secret alone validates and exports identity but blocks nothing. You must also set `SERVER_JWT_PROTECT` to close routes.
- **The mesh hub is plaintext.** Terminate TLS in front of it. Use `wss://` for the worker-to-hub hop and `SERVER_MESH_CA_FILE` for a private CA.
- **MCP mode executes scripts.** A tool call runs a handler with the server's privileges; whitelist the IPs, exclude auxiliaries, and do not expose `/mcp` to the open internet.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — the cascade every `SERVER_*` key resolves through, and where to put each one.
- [Remote Agent Control & Fleet Management with SCORPIO BOT](scorpiox-bot.md) — the fleet built on this server and its mesh, with the same `?format=json` dialect.
- [Native MCP 2.0 and OAuth 2.1](mcp.md) — the MCP client side, the `--mcp` server mode, and the head-to-head harness comparison.
- [Privacy Policy and Data Architecture](privacy.md) — how the self-hosted, zero-collection deployment of these surfaces is structured.
