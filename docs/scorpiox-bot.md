# Remote Agent Control & Fleet Management with SCORPIOX BOT

Running a SCORPIOX CODE agent at your own keyboard is one thing. Running a *fleet* of them — across several machines, several projects, several people — and being able to see what each one is doing, type into any one of them, and kick off new work from a phone or a script, is a different product entirely. **SCORPIO BOT** is that product: a thin, always-on control surface that wraps your running SCORPIOX CODE sessions so you can drive them remotely, watch their output live, and manage every machine they run on from one screen.

It is built from two deliberately small, pure-C programs:

- **`scorpiox-bot-api`** — a headless HTTP API over the agent session state. Every session, prompt, terminal frame, and telemetry reading is exposed as a plain REST or Server-Sent Events (SSE) endpoint. This is the surface for **AI automation** and any client that wants clean JSON.
- **`scorpiox-bot-web`** — a pure-C web dashboard built on top of the API. It gives you the fleet view: every node you have reached, every live session on each node, and a full chat and a real two-way terminal for any one of them.

Neither program modifies SCORPIOX CODE. They read the same session directories the interactive tool already writes, and they route input through the same session machinery, so a session you open in the browser is the same session a local terminal sees — there is no separate copy that can drift from the real thing.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** SCORPIO BOT turns every SCORPIOX CODE session into something you can list, stream, type into, and spawn from any browser or script — and groups many machines into one fleet, from fully self-hosted to heads-behind-NAT via reverse tunnels.

---

## How the pieces fit

```
                         ┌──────────────────────────────────────────────┐
   You (browser) ───────►│  scorpiox-bot-web  :8155                      │
   automation (REST) ───►│  fleet dashboard · chat · terminal · screen   │
                         │        │  proxies ?node=<id>&format=json      │
                         │        ▼                                       │
                         │  scorpiox-bot-api  :8150                       │
                         │  /sessions /inbox /peek /stream /conversation  │
                         └───────────┬────────────────────────────────────┘
                                     │ reads session dirs, routes input
                                     ▼
                         SCORPIOX CODE host (agent sessions on a node)
```

A **node** is one machine running the API (and therefore one SCORPIOX CODE installation). A **fleet** is the set of nodes the web dashboard can see. The web tier is the only thing you point a browser at; it fans out to the nodes using the registry you configure (see [Multi-node clustering](#multi-node-clustering-and-the-fleet)).

Both tiers speak the same dialect. Every route on either tier accepts `?format=json`, so a human in a browser and a model in a loop can use the exact same URL. The web wrappers simply add `?node=<id>` to target a machine and forward the call.

| Half | What it is | Who it is for |
|------|------------|---------------|
| **`scorpiox-bot-api`** | Headless HTTP endpoints over the session files. JSON in, JSON out, SSE for live streams. One small executable per route, served under a standard web contract. | Scripts, CI, AI automation, mobile clients, anyone who wants to drive a session programmatically. |
| **`scorpiox-bot-web`** | A single-page, dark-theme dashboard: fleet overview, per-session chat, live terminal, panes, node management, providers, profiles, settings. | People. Humans who want to see what the fleet is doing and type into it from a browser. |

If you want to *watch and drive*, use the web. If you want to *automate* — feed prompts, harvest output, orchestrate a batch of sessions — use the API. Both hit the same endpoints, so anything you learn about one you already know on the other.

---

## The remote session lifecycle

A SCORPIOX BOT session is just a SCORPIOX CODE session you can reach from outside the machine. The lifecycle is the one you already know — discover it, talk to it, watch it, type into it, end it — and each step maps to a small number of calls.

### 1. Discover what is there

`GET /sessions` is the entry point. It returns every live session on the node, newest first, with enough to act on:

```json
[
  {
    "name": "fix-auth-flow",
    "id": "fix-auth-flow",
    "worktree": "/codebases/myapp",
    "profile": "agent-local",
    "attached": false,
    "harness": "scorpiox",
    "thinking": true,
    "thinking_started": 1759000000000
  }
]
```

The fields you will actually use:

| Field | Meaning |
|-------|---------|
| `name` / `id` | the session id you pass back to every other route |
| `worktree` | the working directory the session runs in |
| `profile` | the SCORPIOX CODE profile it launched under, if any |
| `attached` | whether a real terminal client is currently attached |
| `harness` | what is actually running in it (for example `scorpiox`); absent for a plain terminal |
| `thinking` / `thinking_started` | whether the agent is mid-turn, and since when — the flag you use most in automation |

`GET /sessions_sse` is the live version: a stream that pushes the list whenever it changes, so a dashboard can keep a whole fleet's status current without polling.

### 2. Dispatch a prompt (start a session)

`POST /sessions` starts a new session. The target is **either** a project name (a repository the node knows about) **or** a directory path anywhere on the host (`/tmp/scratch`, `~/work`, `D:\work`). `name` is optional and becomes the session id; `profile` is optional and selects the SCORPIOX CODE profile to launch under.

```bash
curl -X POST "$BOT/sessions" \
  -H "Content-Type: application/json" \
  -d '{"project": "myapp", "name": "fix-auth-flow", "profile": "agent-local"}'
```

A successful launch answers with the new session. If it could not be created — unknown project, directory not found, name already taken — the node answers **`422`** with the tool's own message rather than a bare success, so a caller never silently thinks the work started.

For a plain shell with **no agent** — diagnostics, or running an arbitrary command — use `POST /terminals` with `{"dir": "...", "cmd": "..."}`. These appear in the node's terminal list, not the agent list. `GET /projects` tells you which project names the node accepts before you try.

### 3. Talk to a session that is already running

Dispatching a new session is the heavy path. To send a message to a session that is **already alive** — the normal remote-control case — use the inbox:

```bash
curl -X POST "$BOT/inbox?id=fix-auth-flow&mode=auto" \
  -H "Content-Type: text/plain" \
  --data 'Fix the null check in auth.c and run the tests'
```

The `mode` decides what happens when the agent is busy:

| Mode | Behaviour |
|------|-----------|
| `auto` (default) | The agent picks it up at its next safe point. |
| `queue` | Hold it until the current turn finishes. |
| `interrupt` | Cut into the current turn; the new message takes priority. |

The write is atomic, and the route **waits briefly for the agent to actually consume the message**, so the response tells you whether delivery happened rather than assuming it:

```json
{ "ok": true, "inbox": true, "stem": "1759000420_000000123", "mode": "auto", "session": "fix-auth-flow", "consumed": true }
```

`"consumed": false` means "written, not yet picked up" — not "failed". The write already succeeded either way.

For the shell — not the agent — the low-level `POST /pty_input` and `POST /input_tmux` routes carry raw keystrokes, resize, and interrupt signals. The rule of thumb: **inbox for the agent, pty input for the shell.** If you are driving the model, send a message; if you are at the prompt, type.

### 4. Watch it work

There are three ways to observe a session, and you choose by how much fidelity you need:

| Endpoint | What you get | Best for |
|----------|--------------|----------|
| `GET /conversation?id=…&after=<n>&limit=<n>` | the structured conversation, as messages | reading what the agent did |
| `GET /stream?id=…&rows=<n>` | a live SSE stream of raw ANSI terminal frames | a real, interactive terminal |
| `GET /peek?id=…&format=json` | one snapshot of the terminal as a raster image | a quick "what is on screen" |

**Conversation** is the canonical record. It returns the full history, or a slice when you pass `after` (skip the first N messages) and `limit` (cap the slice). `GET /hashes_sse?id=…&after=<n>` pushes new message markers in real time, so a client can poll cheaply and only fetch the slice it does not already have — this is exactly how the chat view stays live without re-rendering the whole thread.

**Terminal streaming** is how the live terminal works. `GET /stream` first sends an `init` event with the pane size, then a `frame` event each time the screen changes — roughly a dozen frames per second when active, and nothing when idle, so an idle stream costs essentially zero CPU.

```
event: init
data: {"session":"fix-auth-flow","cols":120,"rows":30}

event: frame
data: {"b64":"<base64 ANSI frame>","cols":120,"rows":30}
```

**Peek** renders the current terminal contents to a raster image in-process — no external renderer, millisecond-scale latency. It is one-shot, ideal for a thumbnail or a single "show me the screen" call, and it can return the image as JSON (base64 plus dimensions), as a raw PNG, or as plain text. `GET /peek_sse?id=…` streams the same raster frames and pushes a new image only when the screen actually changes.

### 5. Type back (interactive input)

The terminal screen is two-way. The write side is a POST whose body is a stream of terminal bytes — exactly what a browser terminal would emit — with an `action`:

```
POST /input_tmux?id=fix-auth-flow&action=raw
<body>…keystroke bytes…</body>
```

| Action | Meaning |
|--------|---------|
| `raw` (default) | forward the body as keystrokes |
| `interrupt` / `escape` | send `Esc` |
| `cancel` / `sigint` | send `Ctrl-C` |
| `resize` | pass `?cols=<n>&rows=<m>` to change the pane size |

Printable text, including full UTF-8, is sent literally, so a word like `Enter` in your message is never mistaken for a key. The response reports how much landed. A browser never calls these routes directly — it goes through the web tier's `/terminal`, which stitches `GET /stream` (in) and `POST /input_tmux` (out) into one edge-to-edge screen.

### 6. Answer the agent, not just prompt it

An agent does not only receive prompts; it asks questions and requests permission. SCORPIO BOT exposes those as routes too, so a remote client can answer them:

- **`/askuser`** — the agent's "question for a human" tool. Read the pending question, post an answer.
- **`/permission`** — the agent's "may I run this command" gate. Read the pending request, approve or deny.
- **`/callbacks`** — read or manage the agent's scheduled callbacks remotely, so an autonomous loop can be inspected or stopped without the terminal.
- **`/keepalive`**, **`/ui`** — remote view and control of the cache keep-alive and the agent-driven web UI channel.

Each has a streaming sibling (`/askuser_sse`, `/permission_sse`, and so on) that pushes the moment the agent needs you, so a dashboard can light up and deliver your answer the instant you make it. The net effect: the entire interactive surface of a terminal agent — prompt in, output out, question, permission, scheduled loop — is reachable over HTTP.

### 7. Read the telemetry

Each session publishes small JSON records the API exposes directly. These power the status bar in the web chat view and are handy for automation:

| Endpoint | What it reports |
|----------|-----------------|
| `GET /stats?id=…` | token usage, cache countdown, current branch, working directory |
| `GET /thinking?id=…` | whether the agent is currently thinking, with elapsed seconds |
| `GET /infobox?id=…` | engine telemetry (the floating info box) |
| `GET /slots?id=…` | model slots published by the session (for example a local inference server) |
| `GET /env` | safe diagnostics about the node's environment |
| `GET /cron` | scheduled jobs on the node (read-only) |

Each of the first four has an SSE twin (`/stats_sse`, `/thinking_sse`, `/infobox_sse`, `/slots_sse`) for real-time updates without polling.

### 8. Go beyond chat

The same session-first design reaches into the work itself, all from the dashboard or a script:

- **Resume** — `GET /resume?id=…` lists previous sessions on that node; `POST /resume` hands the chosen one back to the engine exactly as the interactive `/resume` would.
- **Live code changes** — `GET /code_diff?id=…` reports the session's uncommitted work (per-file status and a unified diff), `GET /code?id=…` reads one file, `GET /code_tree?id=…` lists the tree, and `GET /code_search?id=…` searches it. The web's Changes viewer is a thin page over these.
- **Live preview** — `GET /preview?id=…` and `/preview_sse` serve artifacts an agent publishes for the chat preview popup (PDF, HTML, text, image, diff), each with its own slot and optional refresh command.
- **Node configuration** — `GET /providers`, `GET /profiles`, and `GET /config` let a client inspect provider logins, SCORPIOX CODE profiles, and allowlisted config keys. Config writes are deliberately two-step (arm, then confirm) so nothing changes on disk by accident.
- **Quick messages** — `/queue` holds per-project notes that the next agent run in that folder picks up, the "leave a sticky note for the agent" path.

### 9. End it

`DELETE /sessions?id=<session>` kills the session. The session's files remain on disk so you can inspect what happened, the pane goes away, and it disappears from `/sessions`.

---

## Authentication and client integration

Access is enforced before anything is read, and the mode is chosen by how the node is deployed. There are three ways a request can be trusted.

### Self-hosted (node password)

When the node is started with a **node password**, every request must present it in one of three equivalent places:

| Where | How |
|-------|-----|
| Authorization header | `Authorization: Bearer <password>` |
| Custom header | `X-Bot-Password: <password>` |
| Query string | `?pwd=<password>` |

This is the mode to use on your own LAN. With no password set, the API is intentionally open for single-host, private use. The web dashboard keeps node passwords **only in the browser** (per node, in local storage) and attaches them to each proxied call — they are never stored server-side.

### SCORPIO+ (token + permission)

When the node sits behind the house identity service, requests carry a SCORPIO+ token — either a `Bearer` token or the browser `sx_token` cookie set after signing in. The node then checks a claim:

- **`401`** — no token, an invalid token, or a token issued by the wrong service.
- **`403`** — the token is valid, but the account lacks the **`bot`** permission (an **`admin`** account also passes).

Granting `bot` is how an admin lets a specific account — or an automation service account — touch the fleet. Tokens never appear in the API responses.

### Single-user fallback

A box without the identity service has no token, so the web tier would reject everyone. Set a single implicit owner and the dashboard serves that owner even with no identity service present. A signed-in user still wins, so the same box can serve its owner via token and fall back to the single-user identity for everyone else.

### The two client styles

Both styles hit the same endpoints, which is the point.

**AI automation and scripts** talk to the API directly. Plain HTTP, JSON answers, SSE for streams, discoverable route by route:

```bash
# List live sessions on the node
curl -s "https://bot.scorpiox.net/sessions"

# Snapshot a session's screen as an image (PNG) or base64 JSON
curl -s "https://bot.scorpiox.net/peek?id=<session>&format=png" -o screen.png
curl -s "https://bot.scorpiox.net/peek?id=<session>&format=json"

# Dispatch a prompt to a running session, then wait for confirmation
curl -s -X POST "https://bot.scorpiox.net/inbox?id=<session>&mode=auto" \
     -H 'Content-Type: text/plain' --data 'Run the test suite and summarize the failures.'

# Stream the terminal live (SSE)
curl -sN "https://bot.scorpiox.net/stream?id=<session>"
```

**Humans** use the dashboard, which is itself a thin client of the same API:

| View | Route | What it is |
|------|-------|------------|
| Fleet dashboard | `/` | node discovery, live session overview across nodes, spawn and kill |
| Chat | `/chat?id=<session>&node=<id>` | the structured conversation with a live status bar, info box, and terminal modals |
| Terminal | `/terminal?id=<session>&node=<id>` | edge-to-edge interactive terminal (the two-way stream above) |
| Screen | `/screen?id=<session>&node=<id>` | read-only live raster of the terminal — pure observation |
| Panes | `/panes?node=<id>` | a grid of every active session on the node, each a live chat or terminal tile |
| Changes | `/code?id=<session>&node=<id>` | the session's live uncommitted code changes, as a colored diff viewer |
| Nodes | `/servers` | the node registry (add, list, remove) |
| Diagnostics | `/diagnostics` | identity, stored node credentials, tunnel health, fleet ping |

The dashboard is a single-page, dark-theme terminal UI. Selecting a node from the multi-node selector simply prefixes every underlying request with that node's id; the web tier resolves the id to a URL and forwards the call, node password and all.

---

## Multi-node clustering and the fleet

The web tier can see more than the box it runs on. A `?node=` value can resolve from **three places**, in this order:

| Source | What it is | When |
|--------|------------|------|
| **Node files** | plain JSON files on disk, named by a single environment variable | self-hosted; no account, no SQL, no network. Read-only — the operator edits the files. |
| **SCORPIO+ cloud registry** | the account's stored node records | the default. A SCORPIO+ user on their own box sees fleet nodes from the cloud alongside private nodes that never leave the LAN. |
| **Mesh tunnels** | live reverse tunnels, enabled by a hub setting | machines behind NAT (the third source; checked last) |

Every node reports its `source` in the node list, so a mixed fleet — some nodes in the cloud, some that never leave the LAN — shows up with its origin visible. On a self-hosted box the registry can also be kept as plain JSON on disk, so there is no account, no SQL, and no network call at all.

### Reaching machines behind NAT (the mesh)

A home box or a laptop behind a router has no inbound address. Rather than punching holes, the node **dials out** to a hub and holds a WebSocket open; the hub then accepts ordinary HTTP at `/node/<id>/<path>` and forwards it down that tunnel. The node's own identity token is forwarded, so the hub refuses to route to a node owned by a different account.

That is all the `scorpiox-bot --connect <hub>` flag does: it opens the tunnel with **no inbound port open at all**. Omit the hub URL to join the official SCORPIOX mesh hub; a pre-shared key can stand in for a token. Once a node joins, the dashboard's node list picks it up and every route works exactly as if the node were reachable by address. If the tunnel drops, the node reconnects automatically; meanwhile the fleet degrades to the nodes that are still up rather than hanging the page.

---

## Self-hosting

You do not need the cloud. The **`scorpiox-bot`** supervisor starts both daemons under one process with zero cloud dependencies, wires the local environment for you, and never leaves an orphaned daemon behind:

```bash
scorpiox-bot                       # run API (:8150) and Web (:8155)
scorpiox-bot -P secret             # set the node password
scorpiox-bot --api-only            # API only
scorpiox-bot --web-only            # web gateway only
scorpiox-bot --nodes local.json    # explicit node registry
scorpiox-bot init-nodes            # emit a template node file
scorpiox-bot status                # are the daemons answering?
scorpiox-bot login                 # sign in to SCORPIO+ (optional)
scorpiox-bot --connect <hub>       # join a mesh hub over a reverse tunnel
```

The two ports are fixed defaults — **8150** for the API and **8155** for the web — overridable if you need them. A minimal self-hosted web deployment, with the node registry kept as plain JSON on disk, looks like this:

```ini
ExecStart=/usr/local/bin/scorpiox-server -p 8155 \
  -e SERVER_SCRIPT_DIR=/opt/scorpiox-bot-web \
  -e RUNTIMECFG_BACKEND=file \
  -e RUNTIMECFG_DIR=/var/scorpiox-bot/nodes \
  -e BOT_SINGLE_USER=<your-guid>
```

No account, no SQL, no network. On Windows there is one extra requirement: point the environment at where your repositories live, or `/projects` is empty and session lookups fail. There is no auto-detection.

---

## Endpoint reference

### Session control (`scorpiox-bot-api`, :8150)

| Route | Method | Params | Returns |
|-------|--------|--------|---------|
| `/projects` | GET | — | available projects on the node |
| `/sessions` | GET | — | live sessions, newest first |
| `/sessions` | POST | `{project\|dir, name?, profile?}` | the new session, or `422` with the error |
| `/sessions` | DELETE | `?id=<session>` | confirmation |
| `/sessions_sse` | GET | — | live SSE of the session list |
| `/terminals` | POST | `{dir?, cmd?, name?}` | start a plain (non-agent) terminal |

### Conversation and observation

| Route | Method | Params | Returns |
|-------|--------|--------|---------|
| `/conversation` | GET | `?id=&after=&limit=` | full or sliced conversation |
| `/hashes` | GET | `?id=&after=&limit=` | message hash markers |
| `/hashes_sse` | GET | `?id=&after=` | live SSE of new messages |
| `/peek` | GET | `?id=&format=json\|png\|raw&rows=` | one raster snapshot |
| `/peek_sse` | GET | `?id=&rows=` | live SSE raster stream |
| `/stream` | GET | `?id=&rows=` | live SSE ANSI terminal frames |
| `/terminal_tmux` | GET | `?id=` | one-shot terminal read |
| `/screen` | GET | `?id=` | read-only live screen (web tier) |

### Input

| Route | Method | Params | Returns |
|-------|--------|--------|---------|
| `/inbox` | POST | `?id=&mode=auto\|interrupt\|queue` | delivery confirmation |
| `/input_tmux` | POST | `?id=&action=raw\|interrupt\|cancel\|resize` | keystrokes accepted |
| `/pty_input` | POST | `?id=&action=` | raw PTY input |
| `/terminal` | GET | `?id=&font=` | interactive terminal (web tier) |

### Agent interaction

| Route | Method | Params | Returns |
|-------|--------|--------|---------|
| `/askuser` (+ `_sse`) | GET/POST | `?id=` | the agent's pending question and your answer |
| `/permission` (+ `_sse`) | GET/POST | `?id=` | the agent's permission gate and your verdict |
| `/callbacks` (+ `_sse`) | GET/POST/PUT/DELETE | `?id=` | scheduled agent callbacks |
| `/keepalive` (+ `_sse`) | GET/PUT | `?id=` | cache keep-alive state and control |
| `/ui` (+ `_sse`) | GET/POST/DELETE | `?id=` | remote web-UI command channel |

### Telemetry and node info

| Route | Method | Params | Returns |
|-------|--------|--------|---------|
| `/stats` (+ `_sse`) | GET | `?id=` | tokens, cache countdown, branch, cwd |
| `/thinking` (+ `_sse`) | GET | `?id=` | thinking flag and elapsed seconds |
| `/infobox` (+ `_sse`) | GET | `?id=` | engine telemetry |
| `/slots` (+ `_sse`) | GET | `?id=` | published model slots |
| `/env` | GET | — | safe environment diagnostics |
| `/cron` | GET | — | scheduled jobs (read-only) |
| `/providers` | GET/POST | `?provider=` | provider login status and flows |
| `/profiles` | GET/PUT | `?name=` | SCORPIOX CODE profiles |
| `/config` | GET/POST | `{key,value}` | allowlisted node config (approval-gated) |

### Work surfaces

| Route | Method | Params | Returns |
|-------|--------|--------|---------|
| `/resume` | GET/POST | `?id=` | resumable sessions, or a resume request |
| `/code` | GET | `?id=&file=` | one file from the session's tree |
| `/code_diff` | GET | `?id=&file=&stat=` | uncommitted changes, or one file's diff |
| `/code_tree` | GET | `?id=&dir=&depth=` | the session's file tree |
| `/code_search` | GET | `?id=&q=&dir=&limit=` | text search across the tree |
| `/code_save` | POST | `{file,content}` | write a file back |
| `/preview` (+ `_sse`) | GET/POST/DELETE | `?id=&group=&pid=` | live preview groups and slots |
| `/queue` | GET/POST/PUT/DELETE | `?project=&file=` | the per-project quick-message queue |
| `/sglang_metrics` | GET | `?id=&url=&fresh=` | live inference-server metrics |
| `/api/ping` | GET | — | health check |

The web tier (:8155) mirrors the session-control, observation, input, telemetry, and work routes — each accepting `?node=<id>` and `?format=json` — and adds the dashboard views listed earlier. The API's README and the web's README each carry the full route list for their own tier.

---

## Troubleshooting

| Symptom | What it means |
|---------|---------------|
| `401 invalid node password` | A node password is set and your `Bearer`, `X-Bot-Password`, or `?pwd=` did not match it. |
| `401 unauthorized` | No valid SCORPIO+ token reached the route (identity-service mode). |
| `403 forbidden` | Valid token, but the account lacks the `bot` (or `admin`) permission. Ask an admin to grant `bot`. |
| `422` on `POST /sessions` | The session did not start — read the `error` field. Usually an unknown project, a missing directory, or a name already taken. |
| `404` on `/conversation` or `/peek` | The `id` is not a live session on **this** node, or it is on a different node and you forgot `?node=` on the web tier. |
| `"consumed": false` on `/inbox` | The message was written but not picked up yet. It is not a failure; the agent was mid-turn. |
| `/projects` is empty on Windows | The repositories root env variable is not set. There is no auto-detection — set it and relaunch. |
| A node is not in the fleet | Check its `source`: a mesh node only appears while its reverse tunnel is up, and only when the hub setting is enabled on the dashboard. |
| A node was reachable, now it is not | For a mesh node this is normal — the tunnel dropped. It reconnects automatically, and the fleet degrades to the nodes still up rather than hanging the page. |

---

## The bottom line

SCORPIO BOT gives SCORPIOX CODE two remote surfaces over one contract. The **API** is the thing you build against: start a session, submit prompts, stream the terminal, read the conversation, watch telemetry, drive the agent's questions and callbacks, and stop it — all plain REST and SSE. The **web tier** is the thing you look at: a fleet dashboard that spans every node you can reach, with a full chat, a real two-way terminal, and the work surfaces — resume, changes, preview — right beside them. Node files, the SCORPIO+ cloud registry, and reverse-tunnel mesh give you three ways to assemble a fleet, including machines behind NAT with no inbound port. And because both tiers are driven by the same JSON endpoints, a human in a browser and an agent in a loop are doing the identical thing.

---

## Related

- [Configuration and Profiles in SCORPIOX CODE](openai-provider.md) — the model and settings presets a remote session launches under.
- [Session Identity Headers](identity-headers.md) — the stable identifiers the same sessions carry upstream.
- [Privacy Policy and Data Architecture](privacy.md) — the three SCORPIO BOT tiers, from self-hosted zero-collection to optional cloud compute.
- [Using the /keepalive Command](keepalive.md) — the cache keep-alive that SCORPIO BOT exposes and controls remotely.
- [Native MCP 2.0 and OAuth 2.1](mcp.md) — a different tool surface with its own discovery story.
