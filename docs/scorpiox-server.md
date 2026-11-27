# Serving Sites with scorpiox-server

You have a site to serve: a dashboard, a tool, a small internal app, a bot's UI. You could reach for IIS, Apache, nginx, or ASP.NET Core Kestrel — and then spend the rest of the day on runtimes, frameworks, deploy pipelines, virtual-host config, and the reverse-proxy wiring that keeps your API and your static assets from stepping on each other.

**scorpiox-server** is a different shape of answer. It is a single native binary that ships with SCORPIOX CODE. You put Python scripts or compiled executables in a folder, point the binary at it, and every file in that folder is now a route on the web. `GET /hello` runs `hello` or `hello.py`. The routing table is the folder listing. Deployment is a git push. Auth is a JWT the server checks before your script ever starts.

Docs for SCORPIOX CODE @ `41b090b`.

> **The whole idea in one line:** one native binary, zero dependencies, no runtime to install on the target, no web framework, no reverse proxy — the file layout *is* the routing table, git-push *is* the deploy pipeline, and every route you serve must answer in both HTML and machine-readable JSON.

---

## What scorpiox-server is — and is not

scorpiox-server is a lightweight HTTP server written in C, compiled for Linux, macOS, and Windows. It executes the thing behind each route — either a Python script or a native executable — and returns whatever that process writes to stdout as the HTTP response. It is a CGI host in the classic sense, with a few SCORPIOX CODE extensions layered on top:

- **JWT validation in front of the script** — the server checks the token before the script starts, and hands your script the decoded claims as environment variables.
- **Git deploy mode** — point it at a repo URL and it clones, polls, and re-serves on every push.
- **MCP mode** — the same folder of scripts becomes an MCP server, so an AI agent can call each one as a tool.
- **Mesh / inverted transport** — workers on machines with no open ports dial a hub over a single WebSocket and their HTTP routes appear on the hub.

What it is *not*: not a full web framework, not a static-file CDN, not a reverse-proxy in the nginx sense. It executes scripts. Static files are an afterthought: `SERVER_STATIC_ROOT` serves unmatched GET/HEAD routes as files (scripts always beat files).

---

## The contract every route must honour

This is the one rule that matters most to the operator, and it is a requirement of the platform, not an implementation detail. **Every page served on scorpiox-server must come in two shapes**:

| Request | Response |
|---|---|
| `GET /myroute` with a browser `Accept: text/html` | Your HTML page |
| `GET /myroute?format=json` (or `?action=...`, or a `POST`) | The same content as `Content-Type: application/json` |

Your script inspects `QUERY_STRING` (or the relevant parameter) and switches. A browser gets the page; an AI agent, a CI job, or a mobile app gets structured data over plain HTTP. That is the automation surface. If your script does not implement the JSON mode, the route is only half done — you have a page, not a route.

The same dual shape applies to the error surface: an unauthenticated GET/HEAD from a browser is a 302 to your login page; an unauthenticated GET/HEAD from an API client is a `401 {"error":"unauthorized"}`. The server makes that decision from the `Accept` header — the operator does not have to branch on it.

---

## Routing: the file layout is the routing table

The server looks in one or more script directories (see `SERVER_SCRIPT_DIR` below) and resolves each incoming path to a file:

| Request | File served |
|---|---|
| `GET {prefix}{name}` | `{name}` (native executable) or `{name}.py` |
| `POST {prefix}{name}` | same, plus the POST body is available via `POST_BODY_FILE` |
| `GET {prefix}` | `index` or `index.py` (the default page) |
| `*` (no match) | `_fallback` or `_fallback.py` (catch-all) |
| `GET /favicon.ico` | favicon, if `SERVER_FAVICON` is set |
| `GET /api/ping` | health check, returns `ok` |
| `GET /api/otp?a=ACCOUNT&s=SECRET` | TOTP code via the `scorpiox-otp` CLI |
| `POST /mcp` | MCP JSON-RPC 2.0 endpoint, when `--mcp` is on |
| `GET /ws/join`, `GET /mesh/nodes`, `POST /mesh/grant`, `GET /node/<id>/...` | hub endpoints, when `--hub` is on |

Native executables are checked before `.py` files, so a compiled `hello` wins over `hello.py` if both exist. On Windows the native form is `hello.exe`. On Unix the native form must have execute permission. If two directories contain the same script name, the first match wins and a conflict warning is logged at startup.

The route prefix defaults to `/api/platform/websites/`. Set `SERVER_ROUTE_PREFIX=/` for clean website URLs.

### What your script receives

The server invokes your script with a CGI-style environment. The variables it sets include:

- `REQUEST_METHOD`, `CONTENT_TYPE`, `CONTENT_LENGTH`, `HTTP_COOKIE`, `QUERY_STRING`, `HTTP_AUTHORIZATION`, `PATH_INFO`, `REQUEST_URI`
- every request header as `HTTP_<NAME>` (CGI standard)
- `POST_BODY_FILE` when the body is large or chunked (it is spooled to a temp file; your script reads it from that path rather than stdin)
- `X_AUTHENTICATED` (`1` or `0`), and, when a JWT is present, `X_USER_ID`, `X_USER_EMAIL`, `X_JWT_RAW`, plus any custom claim mappings you declared in `SERVER_JWT_CLAIMS`

For `QUERY_STRING` specifically: the default export is the whole string percent-decoded once (legacy). Set `SERVER_QUERY_STRING=raw` to get the query string exactly as sent (CGI / RFC 3875 standard). Individual `?name=x` params are always split-then-decoded in both modes, so a `name=x` query param lands as a proper environment variable either way.

---

## Git deploy mode

This is the feature that replaces the deploy pipeline. Instead of building, uploading, restarting, you give the server a repo URL and it handles the rest:

```
scorpiox-server -r https://host/org/your-site.git
```

The server clones the repo into a secure cache directory, serves the contents, and a background thread polls the remote for new commits. When it sees one, it pulls and re-serves. Default branch is `main`; default poll interval is 10 seconds.

| Flag / Key | What it does | Default |
|---|---|---|
| `-r <url>` | Clone and serve from this repo | — |
| `-b <branch>` | Branch to track | `main` |
| `--poll <seconds>` | Poll interval | `10` |
| `SERVER_GIT_PAT` | Personal access token for private repos | — |
| `SERVER_GIT_CACHE_DIR` | Where the clone lives | `$XDG_CACHE_HOME/scorpiox-server` (Linux/macOS), `%LOCALAPPDATA%\scorpiox-server` (Windows), or `/tmp/scorpiox-server-<uid>` |

Private repos: put a PAT in `SERVER_GIT_PAT`. The server injects it into the auth URL at clone and pull time — it does not appear in the working tree, and the cache directory is created with `0700` on Unix.

Because git deploy *is* the deploy pipeline, you do not also need a CI job that uploads artifacts. Push, and the site updates within one poll interval.

---

## Configuration keys

Everything is set in `scorpiox-env.txt` (the same cascade described in [Configuration and Profiles](./scorpiox-env.md)) or overridden on the command line with `-e KEY=VALUE`.

### Core

| Key | Default | Notes |
|---|---|---|
| `SERVER_PORT` | `8080` | Override with `-p 3000` on the CLI |
| `SERVER_ROUTE_PREFIX` | `/api/platform/websites/` | Set to `/` for clean site URLs |
| `SERVER_SCRIPT_DIR` | — | Comma-separated list; first match wins |
| `SERVER_STATIC_ROOT` | — | Unmatched GET/HEAD routes fall through to this directory as static files (scripts always beat files) |
| `SERVER_MAX_REQUEST_MB` | `200` | Max request body in MB; streamed, not buffered |
| `SERVER_MAX_RESPONSE_MB` | `200` | Max response body in MB |
| `SERVER_SCRIPT_TIMEOUT` | `300` | Idle timeout (seconds of no stdout) — not a wall-clock cap |
| `SERVER_STREAM_SCRIPTS` | — | Comma list of script names whose response is streamed to the client as produced (no size cap, idle timeout only). Unix only. |
| `SERVER_IP_WHITELIST` | empty (allow all) | CIDR list, e.g. `1.2.3.4,10.0.0.0/8` |
| `SERVER_FAVICON` | — | Base64 data or a file path |
| `SERVER_QUERY_STRING` | `decoded` | `raw` = as sent (CGI standard); `decoded` = whole string decoded once (legacy default) |
| `SERVER_SPOOL_BODIES` | `0` | Unix only. Force large/chunked request bodies through a spool file (`POST_BODY_FILE`). Windows always spools. |
| `SERVER_WS_SCRIPTS` | — | Comma list of script names allowed to answer a WebSocket upgrade with `X-Sx-Ws-Target: host:port` (WS-to-TCP relay). The script decides auth and target; the client never names the upstream. |
| `SERVER_PROXY_ROUTES` | — | `/prefix=host:port` pairs; longest prefix wins; the prefix is stripped and sent as `X-Forwarded-Prefix`. Plain HTTP only, no WebSocket, no chunked request bodies. |

### JWT authentication

Set `SERVER_JWT_SECRET` and you get full JWT enforcement in front of your scripts. Empty (default) means the scripts decide auth themselves.

| Key | Default | Notes |
|---|---|---|
| `SERVER_JWT_SECRET` | empty | HMAC-SHA256 secret. If it starts with `/` or `./` it is a file path; otherwise inline. |
| `SERVER_JWT_ISSUER` | empty | Expected `iss` claim. Empty = skip. |
| `SERVER_JWT_AUDIENCE` | empty | Expected `aud` claim. Empty = skip. |
| `SERVER_JWT_COOKIE` | empty | Cookie name for browser clients. The `Authorization: Bearer` header is always checked first. |
| `SERVER_JWT_CLAIMS` | — | Custom `claim=ENV_VAR` pairs (comma-separated). Built-in: `sub->X_USER_ID`, `email->X_USER_EMAIL`. Array claims are joined with commas. |
| `SERVER_JWT_PROTECT` | empty | Route prefixes that require a valid token. Empty = no enforcement. |
| `SERVER_JWT_PUBLIC` | empty | Route prefixes exempt from `SERVER_JWT_PROTECT`. Matches whole path segments (`/assets` covers `/assets` and `/assets/x`, not `/assetsX`). A valid token on an exempt route still exports `X_USER_ID`, etc. |
| `SERVER_JWT_LOGIN_URL` | — | For unauthenticated GET/HEAD: if `Accept` contains `text/html` the client gets a 302 to this URL; otherwise a 401 JSON error. |

A typical protected site:

```
SERVER_JWT_SECRET=/etc/scorpiox/jwt-secret.pem
SERVER_JWT_ISSUER=https://auth.example.com
SERVER_JWT_AUDIENCE=scorpiox-server
SERVER_JWT_COOKIE=sx_session
SERVER_JWT_PROTECT=/
SERVER_JWT_PUBLIC=/assets,/api/health
SERVER_JWT_LOGIN_URL=https://auth.example.com/login
```

Now the whole site requires a valid JWT from `auth.example.com`; static assets and the health check stay public; an unauthenticated browser gets redirected to the login page; an unauthenticated API client gets `401 {"error":"unauthorized"}`.

### MCP mode

MCP (Model Context Protocol) is the protocol an AI agent uses to call tools over HTTP. scorpiox-server can be an MCP server: every script in the served folder becomes a tool, named after the file.

```
scorpiox-server --mcp -p 8888
scorpiox-server -r <repo-url> --mcp
scorpiox-server --mcp --name myserver
```

Each script *can* self-describe by responding to a `--schema` invocation:

```
# Line 1: tool description
# Line 2+: param_name:type:description:required|optional
A tool that does X
input:string:the input to process:required
count:integer:how many items:optional
```

The server runs the script with `--schema` at startup, parses the output, and builds a proper MCP `inputSchema` from it. The protocol is MCP JSON-RPC 2.0 over Streamable HTTP on `POST /mcp`. The handshake name is `MCP_SERVER_NAME` (or `--name` on the CLI). To keep helper scripts out of the tool list, set `MCP_TOOL_EXCLUDE=helper-*,setup,*.bak` (glob patterns).

Connect any MCP client to it — for example the llama.cpp Web UI: *Settings → MCP Servers → Add → `http://host:port/mcp`*.

---

## Mesh / inverted transport

The mesh is for machines that cannot accept inbound connections — a laptop behind a NAT, a box in a data centre behind a firewall, a phone. The worker opens *one* outbound WebSocket to the hub, and from then on the hub routes HTTP requests back down that WebSocket to the worker.

```
# Hub — run once, on a machine that *can* accept connections
scorpiox-server --hub -p 8080

# Worker — run on the machine with no open ports
scorpiox-server --connect wss://hub:8080/ws/join --id mynode
```

The worker's HTTP routes then appear on the hub under `/node/mynode/...`. The hub also serves `GET /mesh/nodes` (the registered node list), `POST /mesh/grant` and `DELETE /mesh/grant` (dispatch consent), and `GET /mesh/grants` (the consent list).

| Flag / Key | Notes |
|---|---|
| `--hub` / `SERVER_MESH_HUB=1` | Run as a hub |
| `--connect <ws-url>` (alias `--join`) / `SERVER_MESH_CONNECT` | Worker: dial this hub |
| `--id <node-id>` / `SERVER_MESH_ID` | Node id (default: hostname) |
| `SERVER_MESH_NAME` | Display name emitted by the hub's `/mesh/nodes` (default: the node id) |
| `--token <sx_token>` / `SERVER_MESH_TOKEN` | Scorpio+ JWT (a token with two dots is a JWT; one with fewer is treated as a legacy pre-shared key with a warning) |
| `--key <psk>` / `SERVER_MESH_KEY` | Pre-shared mesh key, for self-hosted deployments without JWT |
| `--insecure` / `SERVER_MESH_INSECURE=1` | Skip TLS certificate verification on `wss://` |
| `SERVER_MESH_CA_FILE` | CA bundle for `wss://` (default: the system store) |
| `SERVER_MESH_TIMEOUT` | Tunnel request timeout in seconds (default `300`) |

Two auth modes:

- **Scorpio+ multi-tenant** — the hub is started with `SERVER_JWT_SECRET`. Then `/ws/join`, `/mesh/nodes`, and `/node/<id>/` all require a valid JWT. Node ids are bound to the token's `sub`, so tenant A cannot route into tenant B's nodes.
- **Pre-shared key** — `--key` / `SERVER_MESH_KEY`. Only `/ws/join` requires the key (passed as `?token=` on the handshake). Suitable for self-hosted, single-operator deployments.

Worker URL schemes: `ws://` and `http://` are plaintext; `wss://` and `https://` are TLS, verified against the system CA store unless `--insecure`. The hub itself always speaks plaintext — put a TLS terminator (Caddy, nginx, etc.) in front of it.

---

## Why not IIS, Apache, nginx, or Kestrel

| Stack | What it needs | What scorpiox-server does differently |
|---|---|---|
| **IIS** | Windows runtime, IIS role install, app pools, application pools, web.config per site, a deploy pipeline to push files | One native binary. No OS role. No per-site config. Git push is the pipeline. |
| **Apache** | Runtime install, `httpd.conf`, `vhost` files, `.htaccess`, mod_wsgi/mod_cgi for scripts, a deploy step | No `vhost` files. No `mod_*` modules. The folder layout is the routing table. |
| **nginx** | Config syntax, reverse-proxy blocks per API, separate static and dynamic locations, reload cycle, a deploy pipeline | No config reload. No "static location vs dynamic location" split — the server routes both. No reload; git push re-serves in place. |
| **ASP.NET Core (Kestrel)** | .NET runtime, a project, a build step, `Program.cs`, middleware pipeline, a publish step, a host step | No build. No runtime. No `Program.cs`. Your scripts *are* the app. |

The pattern across all four is the same: **a runtime to install, a framework to learn, a config file to write, a deploy pipeline to maintain, and a reverse proxy to wire up when you add an API next to your static assets.** scorpiox-server collapses all of that into one binary where:

- the routing table is the folder listing,
- deployment is a git push,
- auth is a JWT the server validates before your script runs,
- every route is both an HTML page *and* a JSON API (the dual contract above),
- and the same folder of scripts can be exposed as MCP tools to an AI agent.

That is what "zero-dependency native binary where the routing IS the file layout" means in practice: you stop maintaining a stack and start maintaining your site.

---

## A minimal deployment

```
# .scorpiox/scorpiox-env.txt in the repo you are serving
SERVER_ROUTE_PREFIX=/
SERVER_PORT=8080
SERVER_JWT_SECRET=/etc/scorpiox/jwt-secret.pem
SERVER_JWT_PROTECT=/
SERVER_JWT_PUBLIC=/assets,/api/health
SERVER_JWT_LOGIN_URL=https://auth.example.com/login
```

```
scorpiox-server -r https://host/org/your-site.git
```

The server clones `your-site`, serves its contents on `:8080`, polls for new commits every 10 seconds, enforces the JWT on every route except `/assets` and `/api/health`, and redirects unauthenticated browsers to your login page. Every route in the repo that implements `?format=json` is now a JSON API for your agents. No runtime, no reverse proxy, no deploy pipeline.

For the full configuration cascade (profiles, per-project files, environment-variable overrides), see [Configuration and Profiles](./scorpiox-env.md). For driving agent sessions remotely over the same host, see [SCORPIOX BOT](./scorpiox-bot.md).
