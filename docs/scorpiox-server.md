# Serving Sites with scorpiox-server

You want to put a site or an API on a box. The traditional answer is a stack: a web server that only proxies, an application runtime it hands work to, an auth middleware package, a deployment pipeline, and a reverse-proxy config so TLS terminates somewhere sensible. Each layer is a product with its own config language, and every new site means wiring all of them together again.

SCORPIOX CODE ships a different answer: **`scorpiox-server`**, a single native binary that is the web server *and* the application runtime *and* the auth gate *and* the deployment mechanism. The routing is your file layout. A `git push` deploys. JWT validation is a config key. And every page you build serves two audiences at once — HTML for the human in the browser, JSON for the machine in the loop.

This is the same server that powers the product's own public website, and the same one SCORPIO BOT supervises as its API tier. It runs sites in production every day.

Docs for SCORPIOX CODE @ `ad926d7`.

> **The whole idea in one line:** drop `scorpiox-server` next to a folder of Python scripts or native executables, and the file names become the routes — with a mandatory HTML + JSON contract on every page, git-push deployment, built-in JWT authentication, optional static-file serving, per-script response streaming, an MCP mode that turns your scripts into agent tools, and a reverse-tunnel mesh for sites that live behind NAT — all in one zero-dependency binary.

---

## The mental model: routing is the file layout

There is no route table to maintain. Give the server a script directory and a URL prefix, and every executable or `.py` file in that directory becomes a route whose name is the file name:

```
/var/site/
  index.py        ->  GET  /                     (the front page)
  install.py      ->  GET  /install
  api-reference.py -> GET  /api-reference
  _fallback.py    ->  *    (everything else)
```

Hit `/{prefix}{name}` and the server finds a handler named `name`: a **native executable** first (a file with the execute bit set on Unix, `name.exe` on Windows), then a **`name.py`** script run by `python3`. With multiple directories configured, they are searched in order and the first match wins — useful for a shared library of scripts shadowed by site-specific overrides. Name conflicts between directories are detected and reported at startup.

A few conventions make the whole system work:

- **`index` is the default page.** A request to the bare prefix (with or without the trailing slash) runs `index` or `index.py`.
- **`_fallback` is the catch-all.** When no script matches, the server runs `_fallback` or `_fallback.py` — your 404 page, your dynamic router, whatever you want. The product's own website uses a single `_fallback.py` that routes `/tools/{name}` pages and renders 404s.
- **The route namespace is flat.** Names may contain letters, digits, `-`, `_`, and `.` — but not `/`. There are no subdirectories in routing. Scripts that need internal paths (like a docs tree) read `PATH_INFO` and route inside themselves.
- **Traversal is blocked twice.** Names containing `..` are rejected, and every resolved file is verified to actually live inside its script directory before anything executes.

If no script matches, there is no fallback, and no static file is found, the request gets a clean `404` with a JSON error body.

---

## Start it

```bash
scorpiox-server                     # default/configured port (8080)
scorpiox-server -p 3000             # specific port
scorpiox-server -e SERVER_SCRIPT_DIR=/var/site -e SERVER_ROUTE_PREFIX=/
scorpiox-server -r https://git.example.com/org/site.git   # git deploy mode
scorpiox-server -h                  # full help
```

`-e KEY=VAL` overrides any configuration key for this process — repeatable, and values for sensitive keys (secrets, tokens) are masked in the startup log. Configuration otherwise comes from the same `scorpiox-env.txt` cascade the rest of SCORPIOX CODE uses (see [Configuration and Profiles](scorpiox-env.md)), so a server and your agent share one configuration story.

On startup the server prints its banner: port, PID, request/response limits, the route prefix, and each script directory with an accessibility check. Missing directories are warned about; a server with no accessible directories refuses to start rather than serve 404s.

---

## The dual contract: HTML for humans, JSON for machines

This is a requirement of sites built on the server, not a nice-to-have: **every page must answer a browser and a machine from the same URL.** Humans get HTML. Automation, agents, monitors, and other tools get JSON they can consume without scraping. The HTML view is what a person reads; the JSON view is what makes the same page actionable in a loop — one URL, both audiences, no scraping layer in between.

The product's own site does exactly this. The landing page serves rendered HTML to a browser and a plain-text installer script to PowerShell (user-agent sniffing at the CGI layer). The `/version` endpoint answers JSON. A Swagger UI route serves interactive HTML at `/swagger` and the same API surface as a machine-readable OpenAPI document at `/swagger/v1/swagger.json`. SCORPIO BOT's API takes it further with an explicit `?format=json` switch on every route, so a human in the dashboard and a script in a loop use identical URLs.

Your scripts implement the contract however suits the page — content negotiation on the `Accept` header, a `?format=json` query parameter, a parallel `/api/...` route — but build it in from the first commit. Retrofitting "make this page machine-readable" onto an HTML-only site is the mistake this contract exists to prevent. An agent that can read your site is an agent that can act on it.

---

## What a script receives: the CGI environment

Scripts run in a full CGI environment. The server sets everything a standard CGI program expects, plus its own additions:

| Variable | What it carries |
|----------|-----------------|
| `REQUEST_METHOD` | The HTTP method, passed through as-is (`GET`, `POST`, and any other method) |
| `CONTENT_TYPE` / `CONTENT_LENGTH` | The request body's type and size |
| `QUERY_STRING` | The raw query string, URL-decoded |
| `PATH_INFO` | The full request path |
| `HTTP_COOKIE` / `HTTP_AUTHORIZATION` | Those headers, verbatim |
| `HTTP_*` | Every other request header, uppercased with `-` → `_` (so `User-Agent` becomes `HTTP_USER_AGENT`) |
| `POST_BODY_FILE` | Path to a temp file holding the POST body (buffered mode) |
| `SX_STREAMING` | Set to `1` when the body is being streamed (see below) |
| query parameters | **Each safely named query parameter becomes its own environment variable** — `?platform=linux` arrives as `platform=linux` |

The query-parameter-to-environment mapping is the workhorse: a script reads `os.environ.get('platform')` and never parses a query string. The product's own install router does exactly this — one `?platform=` parameter, read straight out of the environment.

**Query parameters are filtered, and never overwrite anything.** A request must not be able to redefine the environment a handler runs in, so only parameters whose names are `[A-Za-z0-9_]`, start with a non-digit, and contain at least one lowercase letter are exported — and only when the name is not already set. Real parameters (`name`, `ref`, `token`, `dataBase64`) pass; all-uppercase names like `PATH`, `LD_PRELOAD`, `GIT_ORG`, `HTTP_AUTHORIZATION`, `REQUEST_METHOD`, or `POST_BODY_FILE` are dropped, as are the lowercase proxy variables (`http_proxy`, `https_proxy`, and friends). An operator's own `-e` value can never be clobbered by the request. If you need to pass an all-uppercase identifier through, send it in a POST body instead of the query string.

POST bodies are handled in one of two ways:

- **Buffered (bodies up to 512 KB):** the body is written to a temp file and exported as `POST_BODY_FILE`, and also piped to the script's stdin.
- **Streamed (larger bodies, or chunked requests):** the body streams straight from the socket to the script's stdin with no temp file and no buffering, with `SX_STREAMING=1` set. A 1 MB upload round-trips through this path byte-for-byte — the mesh test suite verifies exactly that with a SHA256 equality check.

The script's working directory is its own directory. stdout and stderr are merged. The script's stdout becomes the HTTP response.

---

## What a script sends back

Script output is CGI: optional headers, a blank line, then the body.

```
Content-Type: text/html; charset=utf-8
Status: 200 OK

<html>...
```

- `Status:` sets the HTTP status code (values outside 100–599 fall back to 200).
- `Content-Type:` sets the response type (default `text/html`).
- Everything else — `Location:`, `Set-Cookie:`, your own headers — is forwarded to the client. Redirects and cookies are pure script territory.
- If the output has no recognizable CGI header block, the whole output is served as `text/html` with status 200. A script that just prints HTML works with zero ceremony.

Responses also carry permissive CORS headers on every reply (`Access-Control-Allow-Origin: *`, all methods, all headers, credentials allowed), so browser-based clients on other origins work without extra configuration. Tighten this at a TLS-terminating proxy if your deployment needs stricter origins.

**Streaming (SSE):** if a script declares `Content-Type: text/event-stream`, the server switches from buffering to streaming — each `data:` chunk is forwarded the moment the script flushes it, on all platforms. Server-sent events, live logs, progress feeds, token-by-token output: write a normal script that flushes, and the client gets it live. SSE streams are subject to the idle timeout (below), not a total-duration cap — a stream that keeps producing can run as long as it keeps producing.

---

## Limits and lifecycle

These keys govern resource behavior:

| Key | Default | Meaning |
|-----|---------|---------|
| `SERVER_MAX_REQUEST_MB` | `200` | Maximum request body size |
| `SERVER_MAX_RESPONSE_MB` | `200` | Maximum buffered script output (a buffer cap, not a streaming cap) |
| `SERVER_SCRIPT_TIMEOUT` | `300` | **Idle** timeout — seconds with no stdout activity |
| `SERVER_STREAM_SCRIPTS` | *(empty)* | Script base names whose responses are streamed instead of buffered (Unix only) |

The timeout is idle-based, not wall-clock: the deadline resets on every chunk the script produces. A report that streams progress for an hour is fine; a script that hangs silently for 300 seconds is killed (`SIGTERM`, then `SIGKILL` after a 100 ms grace) and whatever output was captured is returned. Bump `SERVER_SCRIPT_TIMEOUT` for scripts with long silent phases — SCORPIO BOT raises it to 900 seconds for exactly that reason.

Concurrency is a process per request on Unix (fork) and a thread per request on Windows. That is the entire concurrency story — no worker pool to size, no event loop to starve. The listen backlog is 256.

### When a response is too big to buffer: `SERVER_STREAM_SCRIPTS`

The normal path captures a script's whole output in memory up to `SERVER_MAX_RESPONSE_MB`, sets `Content-Length`, and sends it. That is wrong for a script that produces an unbounded response — a `git clone` of a multi-gigabyte repository, a database dump, a large download. The request is tiny, so the size-based streaming heuristic never fires, and the response is truncated at the cap.

`SERVER_STREAM_SCRIPTS` names the scripts whose **response** should be relayed as it is produced, regardless of request size:

```ini
SERVER_STREAM_SCRIPTS=_fallback,clone,export
```

For a listed script the server relays stdout straight to the client:

- **No `SERVER_MAX_RESPONSE_MB` cap, no full-body buffering.** Memory stays flat while gigabytes flow.
- **Idle timeout only** — the deadline resets on every read and write. A client that stops reading is bounded by a socket send timeout so it cannot pin the child forever.
- **Correct HTTP framing.** If the script sets `Content-Length`, it is honored. Otherwise the response is chunked for HTTP/1.1 — and the terminating zero-chunk is only written when the script exits cleanly, so a script that fails midway shows up as a truncated transfer rather than a silently short body. HTTP/1.0 clients (which cannot parse chunks) get a close-delimited body instead.
- **Clean child handling.** A client disconnect turns into `EPIPE` in the child, which is then sent `SIGTERM` and `SIGKILL` and reaped — no orphaned processes.

The flag is a comma-separated list of script base names (up to 16). It applies to `_fallback` like any other script, so a catch-all download endpoint streams without extra wiring. **It is Unix-only:** on Windows the setting is ignored with a warning, because response streaming there would require a different relay path.

---

## Static files: `SERVER_STATIC_ROOT`

Not every route is a script. When `SERVER_STATIC_ROOT` is set, GET/HEAD requests that match no script route (no `index`, no `{name}`, no `_fallback`) are served as files from that directory before the final 404:

```ini
SERVER_STATIC_ROOT=/var/www/assets
```

```
/var/www/assets/
  index.html      ->  GET  /
  app.js          ->  GET  /app.js
  img/logo.svg    ->  GET  /img/logo.svg
```

How it behaves:

- **Scripts always win.** Static serving is the last resort, after every script route has been tried — existing sites are unaffected, and a file can never shadow a script of the same name.
- **GET and HEAD only.** Other methods fall through to the normal 404 path.
- **Directories resolve to `index.html`.** A request for a directory (or a path with no file) appends `index.html`.
- **MIME types are chosen from the extension** — HTML, JS, CSS, JSON, images (PNG/JPEG/GIF/SVG/WebP), ICO, PDF, WASM, fonts (WOFF/WOFF2), audio/video (MP3/MP4), XML, text, ZIP/GZIP, and binaries; anything unknown is served as `application/octet-stream`.
- **HEAD returns headers only.**
- **Hardened.** A `..` in the path is refused up front, and the resolved file is verified to actually live inside the root (catching symlinks and encoded escapes). Files are opened read-only, streamed in 64 KB chunks, and sent with `Cache-Control: no-store, no-cache, must-revalidate`.

This is how you host a pure-static site — a landing page, a docs tree, build artifacts, a Web UI's assets — on the same engine, and it pairs naturally with git deploy mode: point `-r` at a repo and set `SERVER_STATIC_ROOT` into the clone, and a `git push` updates your static assets too. Empty (the default) means off.

---

## Built-in endpoints

Two health/utility endpoints ship in the binary itself, plus optional favicon serving:

| Route | Behavior |
|-------|----------|
| `GET /api/ping` | Returns `ok` as plain text. Always unauthenticated — this is your load-balancer and uptime-probe target. |
| `GET /api/otp?a=ACCOUNT&s=SECRET` | Generates a TOTP code by invoking the `scorpiox-otp` CLI (which must be installed). Returns JSON. Both parameters are validated against a strict character set and rejected with `400` otherwise. Like `/api/ping`, this endpoint is matched before JWT protection — it is never gated. |
| `GET /favicon.ico` | Serves `SERVER_FAVICON` — either a base64-encoded icon inline in config or a file path (max 1 MB), sent as `image/x-icon`. Unset means no favicon route. |

`OPTIONS` requests get a real answer too: `204 No Content` by default, unless a script named `options` (or the fallback) exists, in which case the request is routed to it — CORS preflight handling you can customize in script.

---

## Git deploy mode: push to deploy

This is the deployment story, and it needs no pipeline:

```bash
scorpiox-server -r https://git.example.com/org/site.git
scorpiox-server -r https://git.example.com/org/site.git -b staging --poll 30
```

What happens:

1. The server clones the repository (`--depth 1`) into its cache directory and serves **the clone** as the script directory.
2. In git mode the route prefix defaults to `/` — a site repository is the site, no prefix ceremony.
3. A background poll thread compares the local `HEAD` with the remote branch head (`git ls-remote`) every `SERVER_GIT_POLL_INTERVAL` seconds (default 10). On a change it pulls (`fetch` + `reset --hard FETCH_HEAD`) and the new code is live on the next request.

Configuration for the mode:

| Key | Default | Meaning |
|-----|---------|---------|
| `SERVER_GIT_CACHE_DIR` | `~/.cache/scorpiox-server` (honors `XDG_CACHE_HOME`; `%LOCALAPPDATA%\scorpiox-server` on Windows; UID-qualified `/tmp` fallback created `0700`) | Where clones live |
| `SERVER_GIT_POLL_INTERVAL` | `10` | Seconds between remote-head checks |
| `SERVER_GIT_PAT` | *(empty)* | Personal access token, injected into HTTPS URLs for private repositories |

Operational details worth knowing: an existing clone is pulled rather than re-cloned on restart; a failed pull logs a warning and keeps serving the previously working code rather than taking the site down; a failed initial clone aborts startup. Git operations never touch a shell — they are spawned as argument vectors directly, eliminating the command-injection class of bug. And the obvious security note: deploying whatever the branch contains is exactly as trustworthy as whoever can push to that branch. Treat write access to the deploy branch as production access.

The loop is deliberately simple — no webhooks to receive, no runners to register. `git push` is the deploy button, and the site follows the branch within one poll interval. For most sites that is the entire CI/CD story.

---

## Authentication: built-in JWT

Set one key and every route can require authentication:

```ini
SERVER_JWT_SECRET=<hmac-secret>        # or a file path: /etc/scorpiox/jwt.key
SERVER_JWT_PROTECT=/admin,/api/private
SERVER_JWT_LOGIN_URL=https://login.example.com
SERVER_JWT_COOKIE=sx_token
```

The server validates **HMAC-SHA256** JWTs itself — no library, no middleware package. Tokens arrive as an `Authorization: Bearer` header or, if `SERVER_JWT_COOKIE` is set, from the named cookie (the header is checked first). Verification is constant-time, expired tokens (`exp`) are rejected, and optional `iss`/`aud` checks pin tokens to your issuer and audience. An empty `SERVER_JWT_SECRET` disables validation entirely — scripts then decide their own authentication, which is a legitimate mode for sites that handle auth internally.

**Who gets blocked, and how.** Only routes whose path starts with a prefix in `SERVER_JWT_PROTECT` require a token. An unauthenticated browser (`Accept: text/html`) hitting a protected route is redirected to `SERVER_JWT_LOGIN_URL` when that is set; anything else gets `401` with a JSON error body. `SERVER_JWT_PUBLIC` carves exemptions back out of a protected prefix — a site can protect `/` yet leave `/assets` and `/api/health` open. The two match differently: `PROTECT` is a raw prefix match (`/admin` also covers `/adminX`), while `PUBLIC` matches whole path segments (`/assets` covers `/assets` and `/assets/x`, but not `/assetsX`). A trailing slash in either value is trimmed. A valid token on a public route is still validated and exported — public means *not required*, not *ignored*.

**Identity reaches your scripts.** Every request exports what it knows about the caller as environment variables:

| Variable | Source |
|----------|--------|
| `X_AUTHENTICATED` | `1` or `0` — always set when a secret is configured |
| `X_USER_ID` | The token's `sub` claim (falls back to `nameid`) |
| `X_USER_EMAIL` | The token's `email` claim |
| `X_JWT_RAW` | The decoded payload JSON |
| your mappings | `SERVER_JWT_CLAIMS=role=X_ROLE,perm=X_PERMS` exports any claims you name; array claims arrive comma-joined |

So `SERVER_JWT_CLAIMS` is your authorization seam: mint tokens with a `permissions` claim, map it to an environment variable, and have scripts check it. Roles, tiers, scopes — the server handles verification and transport; your script decides what the identity may do.

---

## Network exposure controls

Two more layers ship in the binary:

- **IP allowlist:** `SERVER_IP_WHITELIST=203.0.113.7,10.0.0.0/8` restricts script routes and built-in endpoints to listed IPs or CIDR ranges. Empty means allow all. Behind a reverse proxy, the check consults `X-Forwarded-For` first, then `X-Real-IP`, then the socket peer — the first available proxy header wins, with no fallback to the peer address when a proxy header is present. Non-matching clients get `403 Forbidden`. Note the boundary: mesh endpoints (`/ws/join`, `/mesh/nodes`, `/node/<id>/`) are intercepted at the accept loop *before* the allowlist runs, which is exactly why the mesh has its own JWT/PSK authentication layer — the allowlist protects your routes, the mesh auth protects the tunnel.
- **TLS:** the server speaks plain HTTP and expects TLS termination in front of it — Caddy, nginx, IIS, a cloud load balancer, whatever your stack already runs. This is the same pattern the product's own infrastructure uses, and it keeps the binary free of certificate management. Terminate TLS at the front, point it at `scorpiox-server`, done.

The two compose into a sensible default posture: TLS terminator in front, JWT on the sensitive prefixes, IP allowlist when the audience is known. Everything is config keys — no policy files, no middleware wiring.

---

## MCP mode: your scripts become agent tools

With `--mcp`, the served folder gains a second interface: every script becomes an **MCP tool** callable by any MCP client — including other SCORPIO CODE instances and the llama.cpp Web UI.

```bash
scorpiox-server --mcp -p 8888
scorpiox-server -r https://git.example.com/org/tools.git --mcp
scorpiox-server --mcp --name myserver
```

The endpoint is `POST /mcp`, speaking MCP JSON-RPC 2.0 over Streamable HTTP: `initialize` (protocol version `2024-11-05`, your `serverInfo` name from `--name` or `MCP_SERVER_NAME`), `tools/list`, `tools/call`, and proper JSON-RPC errors for unknown methods. A tool call forks the script, passes the arguments as JSON on stdin, and returns stdout as the tool result — with `isError` flagged when the exit code is non-zero.

Scripts can self-describe with a `--schema` flag:

```
# deploy_report --schema prints:
#
# Build a deployment report for an environment.
# environment:string:staging or production:required
# since:string:ISO date, e.g. 2026-09-01:optional
```

Line one is the tool description; each following line is a parameter as `name:type:description:required|optional`. At startup the server probes each script with this flag (5-second timeout), builds real JSON schemas from what answers, and registers up to 128 tools. Tool names must be alphanumeric plus `_` and `-`, names starting with `_` are skipped (internal helpers stay internal), and `MCP_TOOL_EXCLUDE=helper-*,setup,*.bak` filters out anything that should never be advertised as a tool.

The same deployment modes apply: MCP mode over a git-deployed repo means your tool folder gets updates by `git push`, same as a website. For the client side of MCP — connecting *to* other servers, OAuth flows, allow/deny lists — see [Native MCP 2.0 and OAuth 2.1](mcp.md).

One operational caution, stated plainly: a tool call executes a script with your privileges, and `/mcp` answers whatever client can reach the port. Combine `SERVER_IP_WHITELIST`, JWT protection via your TLS terminator's auth layer, and `MCP_TOOL_EXCLUDE` so nothing dangerous gets advertised by accident.

---

## The mesh: sites behind NAT, without open ports

`scorpiox-server` also speaks an **inverted transport** — a reverse-tunnel mesh that solves the oldest deployment problem there is: how do you serve a site on a machine that has no open inbound ports?

**A hub** accepts worker connections and routes public traffic to them:

```bash
scorpiox-server --hub -p 8080
```

**A worker** dials out to the hub and serves traffic through that connection:

```bash
scorpiox-server --connect wss://mesh.example.com/ws/join --id build-node
```

Run the worker with no `-p` flag and it opens **zero inbound ports** — it only dials out. (Give it a port and it also listens locally, which is how a machine can be both a mesh worker and a normally-reachable server.) The hub receives `GET /node/<id>/<path>`, forwards it down the worker's existing tunnel, the worker executes it through its normal routing (same scripts, same CGI environment, same JWT identity variables), and the response streams back — including POST bodies and SSE. A worker behind home NAT, a container, or a corporate firewall becomes a reachable site as long as it can make *outbound* connections.

Hub endpoints, all intercepted alongside its normal local routes:

| Endpoint | Purpose |
|----------|---------|
| `GET /ws/join` | Worker registration handshake |
| `GET /mesh/nodes` | JSON registry of connected nodes: `mode`, node `id`s, `connected_at`, `last_seen` |
| `GET|POST /node/<id>/<path>` | Route traffic through a worker's tunnel |

Connection hygiene is handled: 30-second keepalive pings, automatic reconnection with exponential backoff (1 s doubling to a 30 s ceiling, auth failures jumping straight to the ceiling), and a dead worker answers `502 node offline` while its registration — and its owner — are retained for when it returns.

**Three authentication modes, selected automatically by configuration:**

| Mode | Trigger | Effect |
|------|---------|--------|
| **SCORPIO+** (JWT) | `SERVER_JWT_SECRET` set on the hub | `/ws/join`, `/mesh/nodes`, and `/node/<id>/` all require a valid token. Every node id is bound to the `sub` of the token that first registered it — user A cannot reach, hijack, or even list user B's nodes. The `admin` permission bypasses ownership checks and sees owner metadata in the registry. |
| **PSK** | `--key <secret>` / `SERVER_MESH_KEY` | A pre-shared key gates `/ws/join` only; node routes and the registry stay open. Simple LAN trust. |
| **Open** | neither | Everything open. Dev machines and trusted LANs only. |

JWT and PSK combine — the token provides identity, the key is an extra gate. Client-supplied `X-Mesh-*` headers are stripped at the hub and replaced with the real authenticated identity (`X-Mesh-Node`, `X-Mesh-User`, `X-Mesh-Email`), so a script behind a worker sees trustworthy caller identity it cannot be spoofed into believing.

Workers find their token in order: `--token`, `SERVER_MESH_TOKEN`, the `SX_TOKEN` environment variable, then the `~/.scorpiox/auth.json` file written by `scorpiox-bot login` — so a machine that has already logged into SCORPIO+ needs no token plumbing at all. `wss://` is supported with certificate verification against the system CA store (or `SERVER_MESH_CA_FILE`); `--insecure` skips verification, loudly. The hub itself always speaks plaintext with TLS terminated in front of it — the product's own hub at `mesh.scorpiox.net` runs exactly this way behind Caddy.

The cleanest way to run all of this: `scorpiox-bot --connect` supervises a mesh worker for you — wiring the token, the node id, the parent-death shutdown so an orphaned tunnel can never outlive its supervisor — and shows up in the SCORPIO BOT fleet dashboard. See [Remote Agent Control and Fleet Management with SCORPIOX BOT](scorpiox-bot.md).

---

## The shared-library form factor

The same source also builds **`scorpiox-server-dll`**, a shared library exposing `web_server_start`, `web_server_stop`, `web_server_status`, and `web_server_get_config` for embedding in another application via P/Invoke or FFI. Point it at a JSON config (`root_dir`, `route_prefix`, `port`) and your .NET (or anything-with-FFI) application hosts a full script-serving web server in-process — no side process, no port juggling between apps.

---

## How this compares to the mainstream stacks

Here is what an operator would otherwise assemble, and what each piece costs:

| | **IIS** | **Apache httpd** | **nginx** | **ASP.NET Core (Kestrel)** | **scorpiox-server** |
|---|---|---|---|---|---|
| What you install | A Windows Server role | The httpd package + MPM choices | The nginx package | .NET SDK + runtime, `dotnet` CLI | One static binary |
| Routing | web.config, handlers, URL Rewrite module | `.htaccess` + mod_rewrite | `location` blocks in nginx.conf | C# attribute/endpoint routing in code | **The file layout is the routing** |
| Script/runtime execution | CGI/FCGI + external process managers | mod_cgi / mod_fcgid / php-fpm | None natively — proxies to something that does | The app *is* the runtime (C#) | Native exec + `python3`, in-process |
| Auth | Windows auth + modules; JWT via packages | Auth modules per scheme | Auth via config or proxied app | JWT bearer NuGet package + middleware code | **Built in, one config key** |
| Git-push deploy | Web Deploy / MSBuild pipelines | Custom scripts | Custom scripts | `dotnet publish` + CI jobs | **Built in, one flag** |
| Static files | IIS static handler + MIME config | mod_mime + config | `root`/`alias` + `try_files` | `UseStaticFiles()` middleware | **Built in, one directory key** |
| Large/streamed responses | Response-buffering knobs | Proxy buffering directives | `proxy_buffering` / `sendfile` | `Results.Stream` in code | **Per-script streaming list** |
| TLS | IIS bindings + certs | mod_ssl + cert wiring | Server blocks + certs | Kestrel behind IIS/nginx, typically | Any TLS terminator in front |
| Machine-readable responses | App-level concern | App-level concern | App-level concern | App-level concern | **Stated contract on every page** |
| Serving sites behind NAT | Needs tunneling product | Needs tunneling product | Needs tunneling product | Needs tunneling product | **Built in (mesh mode)** |
| Config surface | web.config + app pools + modules | httpd.conf + .htaccess sprawl | nginx.conf + includes | Program.cs + appsettings.json + csproj | One `KEY=VALUE` file, same cascade as the agent |

The honest framing: those stacks are excellent at the things they were built for, and a large organization with dedicated platform teams should keep them. What scorpiox-server changes is the **floor**. The floor for serving a script-backed site with authentication, deployment, static assets, and machine-readable responses drops from *five coordinated products and their wiring* to *one binary, one config file, and one git remote*. And when a site outgrows the floor — when you genuinely need nginx's raw proxy throughput or IIS's Windows integration — the TLS-terminator pattern means the mainstream stack slots in front of scorpiox-server without displacing it.

---

## The operator checklist

Bringing a site up, end to end:

```bash
# 1. Write the site — scripts in a folder
mkdir -p /var/site && cd /var/site
cat > index.py <<'EOF'
#!/usr/bin/env python3
print("Content-Type: text/html")
print()
print("<h1>It works</h1>")
EOF

# 2. Configure
cat > /etc/scorpiox/scorpiox-env.txt <<'EOF'
SERVER_PORT=8080
SERVER_SCRIPT_DIR=/var/site
SERVER_ROUTE_PREFIX=/
SERVER_STATIC_ROOT=/var/www/assets
SERVER_STREAM_SCRIPTS=_fallback,export
SERVER_JWT_SECRET=/etc/scorpiox/jwt.key
SERVER_JWT_PROTECT=/admin
SERVER_JWT_LOGIN_URL=https://login.example.com
SERVER_IP_WHITELIST=203.0.113.0/24
EOF

# 3. Run — or deploy by git and let it follow the branch
scorpiox-server -r https://git.example.com/org/site.git -b main

# 4. Terminate TLS in front, point /api/ping at your monitor
```

Every key above is documented in the shipped `scorpiox-env.txt` and read through the standard cascade — machine-wide, user, project, and profile tiers all work, so a staging server and a production server can be two profiles of the same file. See [Configuration and Profiles](scorpiox-env.md).

---

## Gotchas

- **The default route prefix is not `/`.** The compiled-in default is `/api/platform/websites/` — a legacy integration path. Sites almost always want `SERVER_ROUTE_PREFIX=/`. Git deploy mode defaults to `/` for exactly this reason; bare mode does not. If your routes 404 and the banner shows a long prefix, this is why.
- **`/api/ping`, `/api/otp`, and `/favicon.ico` are reserved.** A script named `api` will not shadow `/api/ping` — the built-in endpoints are matched before script routing. Plan around them.
- **Query parameters are filtered, and never overwrite.** Only names of `[A-Za-z0-9_]` starting with a non-digit and containing at least one lowercase letter are exported, and never over an already-set variable. All-uppercase names (`PATH`, `REQUEST_METHOD`, `GIT_ORG`, `POST_BODY_FILE`) are silently dropped — pass uppercase identifiers in a POST body instead. The payoff: a request can no longer poison the environment your script, the CGI layer, or your own `-e` overrides run in.
- **The idle timeout is not a total timeout.** A script that prints one dot every 299 seconds runs forever. If you need wall-clock limits, enforce them in the script.
- **`SERVER_JWT_PROTECT` and `SERVER_JWT_PUBLIC` match differently.** `PROTECT` is a raw prefix match — `/admin` also protects `/adminX`. `PUBLIC` is whole-segment — `/assets` exempts `/assets/anything` but not `/assetsX`. A trailing slash in either value is trimmed, so `/assets/` behaves the same as `/assets`. If you protect `/admin`, protect against lookalike paths too (use `/admin/` and a route that never starts with the same letters).
- **A valid token on a public route is still validated and exported.** Public means *not required*, not *ignored*. Scripts can rely on `X_USER_ID` on public routes when a token happens to be present — and should check `X_AUTHENTICATED` before trusting it.
- **Git deploy mode serves the branch, not a release artifact.** A bad push goes live within one poll interval. Protect the branch, or use a deploy branch you push to deliberately.
- **The response cap is a buffer cap, not a streaming cap.** A buffered script that would emit 300 MB gets truncated at `SERVER_MAX_RESPONSE_MB`. Raise the cap, or list the script in `SERVER_STREAM_SCRIPTS` to stream its output with no size cap (Unix only).
- **`SERVER_STREAM_SCRIPTS` is Unix-only.** On Windows the setting is ignored with a warning — size the response cap instead, or terminate at a proxy that streams.
- **Static serving is the last resort.** `SERVER_STATIC_ROOT` only runs after every script route (including `_fallback`) has missed, so a `.html` file can never shadow a script of the same name. If a static page 404s while the file clearly exists, check that no `_fallback` script is swallowing the route.
- **`/mcp` has no JWT gate of its own.** MCP tool calls run scripts with your privileges. Protect the port with `SERVER_IP_WHITELIST` or terminate auth at your TLS proxy before exposing it beyond a trusted network.
- **Mesh workers need outbound connectivity only — and that is also their failure mode.** A worker that can no longer reach the hub disconnects; the hub returns `502 node offline` for its routes until it reconnects. Design clients to tolerate transient `502`s on `/node/` routes.
- **The hub is plaintext by design.** Do not "fix" this by putting certificates on the hub — put a TLS terminator (Caddy, nginx, a load balancer) in front, exactly like the product's own mesh does.

---

## Related

- [Native MCP 2.0 and OAuth 2.1](mcp.md) — the client side of MCP, and more on `--schema` self-description.
- [Remote Agent Control and Fleet Management with SCORPIOX BOT](scorpiox-bot.md) — the fleet layer that supervises scorpiox-server as its API tier, and the `?format=json` machine contract in practice.
- [Configuration and Profiles](scorpiox-env.md) — the cascade every `SERVER_*` key is read through.
- [Privacy Architecture and Zero Data Collection](data-privacy.md) — what the server writes to disk, and what never leaves your machine.
