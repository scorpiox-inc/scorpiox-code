# Remote Agent Control & Fleet Management with SCORPIOX BOT

SCORPIOX CODE runs agents in sessions that live on a machine — your workstation, a build box, a cluster node, or a server somewhere else. **SCORPIOX BOT** is how you reach those sessions from anywhere: dispatch prompts, watch the terminal stream in real time, peek at what is happening, and manage a whole fleet of machines from one screen. It is built from two pure-C components with zero Python in the request path:

- **`scorpiox-bot-api`** — a stateless HTTP + Server-Sent Events (SSE) API that talks directly to the agent session files. This is the surface for **AI automation** and any client that wants clean JSON.
- **`scorpiox-bot-web`** — a single-page C web dashboard for **humans**: fleet overview, live chat, an edge-to-edge terminal, and the node registry. It sits in front of the API and routes every request to the node you pick.

Both are served by `scorpiox-server` (one compiled executable per route, no runtime, no database on the hot path). Nothing about an agent session changes when you add BOT on top — the API and web layer are a thin, read-mostly control surface over the session files SCORPIOX CODE already writes.

Source of truth: the `scorpiox-bot-api` and `scorpiox-bot-web` route handlers, at commit `5fd054b`.

---

## The two surfaces, at a glance

| | Headless API (`scorpiox-bot-api`) | Web dashboard (`scorpiox-bot-web`) |
|---|---|---|
| **Who it is for** | Scripts, AI agents, the iOS app, automation | You, in a browser |
| **Talks to** | Session files directly on one machine | The API, on the node you select (`?node=`) |
| **Shape** | REST JSON + SSE streams | Server-rendered pages + live SSE |
| **Typical port** | `8150` | `8155` (self-hosted example) |
| **Good at** | Dispatching a prompt, streaming output, polling state | Seeing the fleet, chatting, driving a live terminal |

The web dashboard is not a second implementation — it is a proxy. Every web route that targets a node resolves that node and forwards to the API daemon on it, passing your identity along. That means one fleet hub can drive a dozen machines, and the same JSON endpoints work whether the agent is next door or on the other side of the world.

---

## The remote session lifecycle

A session is a running agent (a tmux or `scorpiox-multiplexer` pane) backed by a session directory that SCORPIOX CODE keeps on disk. BOT gives you a full lifecycle over it: **see it, start it, feed it, watch it, and kill it.**

### 1. List what is running

`GET /sessions` returns the active sessions, newest first. Each entry carries the name you use as `?id=`, the stable `session_id`, when it started, whether a client is attached, its working directory, and its profile:

```json
[
  {
    "name": "clang",
    "id": "clang",
    "session_id": "5fd054b_clang_a1b2c3",
    "session_name": "clang a1b2c3",
    "created": 1729000000,
    "updated": 1729000000,
    "attached": false,
    "worktree": "/codebases/clang",
    "profile": "local-llama"
  }
]
```

For live updates instead of polling, `GET /sessions_sse` streams the full list as an SSE `sessions` event whenever it changes, plus periodic keep-alives. `GET /sessions?projects=1` lists the projects available for spawning.

### 2. Start a session

`POST /sessions` spawns a new agent. The target is either a **project name** (a codebase under the codebases root, e.g. `"clang"`) or an explicit **directory path** anywhere on the host (`/tmp/scratch`, `~/work`, `D:\work`). You can optionally pin the session name.

```bash
# spawn in a project
curl -X POST "$BASE/sessions" \
  -H 'Content-Type: application/json' \
  -d '{"project":"clang"}'

# spawn in an arbitrary folder
curl -X POST "$BASE/sessions" \
  -H 'Content-Type: application/json' \
  -d '{"dir":"/tmp/scratch","name":"scratch"}'
```

On success it returns `{"ok":true,"session":"<id>",...}`. Path targets are validated strictly (shell metacharacters are rejected outright), so you cannot inject commands through the target.

### 3. Dispatch a prompt

`POST /inbox` is how you feed a message to a running session. The body is the prompt text (or a small JSON body); the `mode` selects what happens when the agent is busy:

| Mode | Behavior |
|------|----------|
| `auto` (default) | Submit now; if the agent is mid-run it is queued. |
| `queue` | Always queue behind the current turn. |
| `interrupt` | Cancel the current run (sends an escape) and submit this one. |

A prompt beginning with `~` is treated as a queued follow-up, matching the TUI. The response confirms the mode and whether the message was consumed immediately:

```json
{"ok":true,"inbox":true,"stem":"...","mode":"auto","session":"clang","consumed":true}
```

### 4. Stream the terminal

Two streaming endpoints cover the two ways you want to watch:

- **`GET /stream?id=`** — raw ANSI pane frames over SSE, about 12 FPS when the pane changes and essentially zero CPU when idle. The first `init` event carries the pane dimensions; each `frame` event is a base64-encoded ANSI capture. This is what drives a real terminal in the browser (xterm.js).
- **`GET /peek?id=&format=png|json|raw&rows=N`** — a one-shot render of the terminal to a PNG (in-process, no external processes, ~5 ms). Use `format=png` to drop an image anywhere, `format=json` for the image plus dimensions as JSON, or `raw` for the bare bytes. `GET /peek_sse` streams these PNG frames on change.

### 5. Peek at state without a terminal

For "what is it doing right now?" there are lightweight state endpoints:

- **`GET /stats?id=`** — token usage, prompt-cache countdown, branch, and working directory.
- **`GET /thinking?id=`** — whether the agent is actively reasoning, and for how long (`{"active":true,"elapsed_s":42}`).
- **`GET /conversation?id=&after=&limit=`** — the full conversation or an incremental slice: `{"id","total","after","start_index","messages":[...]}`. Pair with `GET /hashes` (FNV-1a per event) and `GET /hashes_sse` for cheap incremental sync — ask "what changed since `after`" instead of re-downloading the whole history.
- **`GET /commands?id=`** — the slash-command list for the session.
- **`GET /infobox?id=`** — the engine info box.

### 6. Drive it like a terminal (or stop it)

When you want full interactive control rather than just prompts, the input endpoints route raw terminal bytes, resizes, and interrupts into the pane. There is a pair per backend:

- `POST /input_tmux?id=&action=raw|interrupt|resize` and its read side `GET /terminal_tmux?id=`
- `POST /input_scorpiox_multiplexer?id=&action=raw|interrupt|resize` and `GET /terminal_scorpiox_multiplexer?id=`

`action=raw` forwards the keystroke body; `action=interrupt` sends an escape; `action=resize` acknowledges a viewport change. There is also `POST /pty_input?id=&action=...` for the legacy PTY path. To stop a session outright, `DELETE /sessions?id=` kills it.

---

## Authentication

Every API route enforces authentication before it touches a session. In the cloud there are two ways in, and the same JWT works for both:

| Client | How it authenticates |
|--------|----------------------|
| **API / iOS / scripts** | `Authorization: Bearer <token>` (the token your client already holds after login). |
| **Browser** | The `sx_token` cookie set after you log in at `auth.scorpiox.net`. |

Access is then gated by a **permission**. A valid token still gets a `403` unless it carries the `bot` or `admin` permission, which admins assign per user. So: `401` means no token, an invalid token, or the wrong issuer; `403` means a valid token that is not allowed into BOT.

### Self-hosted (no `auth.scorpiox.net`)

When you run your own box you do not need the JWT stack at all. Pick one:

- **`BOT_PASSWORD`** — set it and every input/terminal route must present it via the `X-Bot-Password` header, a `Bearer` token, or `?pwd=`. Leave it unset and the endpoint is open, which is fine on a trusted LAN but is the one configuration to get right before you expose anything.
- **`BOT_SINGLE_USER`** — set a single implicit owner GUID and drop the `SERVER_JWT_*` flags entirely. A signed-in user still wins, so the same box can serve its owner by JWT and everyone else by the single-user identity.

A minimal self-hosted web unit looks like this:

```ini
ExecStart=/usr/local/bin/scorpiox-server -p 8155 \
  -e SERVER_SCRIPT_DIR=/opt/scorpiox-bot-web \
  -e SERVER_ROUTE_PREFIX=/ \
  -e RUNTIMECFG_BACKEND=file \
  -e RUNTIMECFG_DIR=/var/scorpiox-bot/nodes \
  -e BOT_SINGLE_USER=00000000-0000-0000-0000-000000000001 \
  -e RELAY_ALLOW_RAW=1
```

> **Zero-trust by design.** Secrets are never stored server-side. The node registry holds only what it needs to find a machine; credentials for talking to a node stay in the client (browser `localStorage`) and are never written to a database.

---

## Multi-node clustering (the fleet)

The web dashboard is the **fleet hub**. It does not know any sessions itself — it resolves a `?node=` against a registry and forwards. The registry is `GET /servers`, and it answers in JSON for any client:

```json
{"ok":true,"servers":[
  {"id":"node-1","name":"build-box","url":"http://192.168.1.50:8150","relay":true,"type":"static","online":true,"source":"cloud"},
  {"id":"mesh-a1b2","name":"laptop","url":"http://...","relay":false,"type":"tunnel","online":true,"source":"mesh"}
]}
```

### Where nodes come from

The registry can be backed by one or both of two stores, chosen at build/config time and never mixed per-entry:

| Backend | Storage |
|---------|---------|
| **`cloud`** (default) | The Scorpio+ `RuntimeConfig` via the KeyValue bridge. Your fleet lives in your account. |
| **`file`** | Plain JSON files under `$RUNTIMECFG_DIR` (e.g. `/var/scorpiox-bot/nodes`). No account, no network, no database. Read-only from the web — the operator owns the files on disk. |
| **`cloud,file`** | **Mix and match.** Reads are unioned and the first backend listed wins an id collision; one backend being down is not fatal, you still get the other's nodes. Writes go to the backend that already owns the id. |

Every node reports its `source`, so a user on their own box sees cloud fleet nodes alongside private nodes that never leave the LAN. A third, live source is merged in when reverse tunnels are enabled: a machine behind NAT dials out to a hub and holds a socket open, and the hub forwards ordinary HTTP to it. Tunnel nodes appear with `type:"tunnel"` and a live `online` flag, while a static registry entry always wins an id collision.

### Routing a request to a node

Every web route that targets a session takes `?node=<id>` (and `&pwd=` when the node is password-protected). Without a node, the web hub shows the fleet and server metadata. The hub resolves the node id, dials its API base, and presents your token to it — so the node you can see is exactly the node you are allowed to drive.

> **`RELAY_ALLOW_RAW`** lets `/relay?node=<url>` accept a raw URL instead of a registry id. That is convenient on your own LAN and is the one flag you never enable in the cloud.

---

## A minimal automation client

Put it together: authenticate, pick a node, dispatch, stream, check state.

```bash
BASE="https://bot.example.net"          # web hub (resolves ?node=) or the API directly
NODE="build-box"
H='Authorization: Bearer <token>'        # or rely on the sx_token cookie in a browser

# what's running on that node?
curl -sH "$H" "$BASE/sessions?node=$NODE"

# give it a task
curl -sX POST -H "$H" -H 'Content-Type: text/plain' \
  --data 'Run the test suite and summarize failures.' \
  "$BASE/inbox?id=clang&node=$NODE&mode=auto"

# watch the terminal live (SSE)
curl -sN -H "$H" "$BASE/stream?id=clang&node=$NODE"

# is it still thinking?
curl -sH "$H" "$BASE/thinking?id=clang&node=$NODE"

# when it's done, take a screenshot of the final pane
curl -sH "$H" -o pane.png "$BASE/peek?id=clang&node=$NODE&format=png"
```

The same calls work against `scorpiox-bot-api` directly (drop `?node=` and point at `:8150`) when you are scripting a single machine.

---

## Gotchas

- **The API is read-mostly by design.** It reflects what SCORPIOX CODE writes to disk. If a session directory does not exist yet (a session that just started and has not written its first event), conversation and stats endpoints will `404` until it does — poll `/sessions` first.

- **`/stream` and `/peek` are different tools.** `/stream` gives you a live ANSI feed for a real terminal; `/peek` gives you a rendered image. Do not render a terminal by hammering `/peek` in a loop when `/stream` will do it.

- **`401` vs `403` is intentional.** `401` = fix your token; `403` = your token is fine but the `bot`/`admin` permission was not granted to your user. An admin has to assign it.

- **A sleeping machine just disappears.** Nodes are not heartbeats; a box that goes offline drops out of `/servers` on its own. Reappear by restarting BOT on it.

- **Self-hosted and open is the same thing until you set `BOT_PASSWORD`.** An unset password means anyone who can reach the port can drive the sessions on it. Set it before the box leaves the LAN.

- **One node per request.** The web hub routes a single `?node=` per call. To fan out across the fleet, iterate the `servers` list — the hub is a router, not a load balancer.
