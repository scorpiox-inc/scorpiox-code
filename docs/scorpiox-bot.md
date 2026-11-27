# Remote Agent Control and Fleet Management with SCORPIOX BOT

An agent session is running on a box that is not the one you are sitting at. Maybe it is a Linux box in another room, a Mac Mini in the rack, a Windows machine across the LAN, or a laptop behind a router you cannot port-forward. **SCORPIOX BOT** is the control plane for exactly that situation: it lets you watch a running SCORPIOX CODE agent, send it messages, answer its questions, and manage every session on every machine in your fleet — from a browser or from plain HTTP calls, on any device, on any machine.

It is built from two parts: a stateless HTTP API that runs on each agent machine (scorpiox-bot-api) and a pure C web frontend (scorpiox-bot-web) that is the fleet dashboard and the live chat and terminal views. Both are compiled native C — no Python, no transpiled frontend bundle, no node_modules — and both run on the same `scorpiox-server` CGI host you already use for SCORPIOX CODE.

Docs for SCORPIOX CODE @ `41b090b`.

> **The whole idea in one line:** one web address for your whole fleet — every node, every session, every live terminal, every pending question — and every route is also a plain JSON API, so an AI agent or a CI script can drive it exactly the way a browser does.

---

## The architecture: a stateless node API, a stateless web hub

- **scorpiox-bot-api — the node side.** One small HTTP route per capability, all stateless, all served by `scorpiox-server` with a CGI contract. It knows exactly one thing: the agent sessions on **its** machine. It reads the live session state on disk, captures terminal panes, renders them, dispatches your messages, and answers the agent's prompts. It holds no fleet state and no browser logic.
- **scorpiox-bot-web — the fleet side.** The dashboard, the chat page, the fullscreen terminal, and the settings and diagnostics pages. It holds no session data either. It is a router: every route accepts a `?node=` parameter, resolves which machine that node is, and proxies the request to that machine's bot-api. The web layer adds the multi-node selector, the node registry, and the rendering — the node layer does the work.

That split is what makes clustering cheap: adding a machine to your fleet is a registry entry, not a new deployment. And because the web layer is a pure proxy, every route in the web UI is simultaneously a route in the API — the browser and your automation script are talking to the same thing.

---

## The fleet: nodes, clustering, and how `?node=` resolves

A **node** is one machine running SCORPIOX BOT (that is, running scorpiox-bot-api behind `scorpiox-server`, with your agent sessions living on it). `GET /servers` on the web host returns your fleet as JSON:

```json
{ "userId": "...", "servers": [
  { "id": "office", "name": "Office Box", "url": "http://192.168.1.40:8150", "relay": false, "type": "static" },
  { "id": "lab-vm", "name": "Lab VM", "url": "https://bot.example.com/node/lab-vm", "relay": false, "type": "tunnel" }
] }
```

Every other route in the web UI accepts `?node=<id>` (or a raw URL on your own LAN, when your operator has allowed it), and that is how one browser reaches every machine in the fleet. A node can be registered in one of three ways:

- **Static nodes** — a machine reachable at a URL on your network or a public endpoint. The node record is a name, a URL, and a node password. On self-hosted setups these live in plain JSON files (no account, no database, no network dependency); on the cloud they live in your SCORPIO+ RuntimeConfig.
- **Cloud registry nodes** — the same records, synced through your SCORPIO+ account, so a fleet you manage from the cloud appears on every device you sign in with, including your self-hosted box.
- **Tunnel nodes** — for machines behind NAT or a firewall that cannot be reached by IP. The node dials **out** to a `scorpiox-server` hub and holds a WebSocket open; the hub then accepts ordinary HTTP at `/node/<id>/<path>` and forwards it down that socket. To the rest of SCORPIOX BOT a tunnel node is indistinguishable from a direct one — you point `?node=` at its id and the web layer takes care of the detour.

Each node reports its `source` in `/servers?format=json`, so on your own box you can see your fleet nodes from the cloud alongside private nodes that never leave the LAN. The dashboard shows the union, and a dead backend is not fatal: the other one keeps answering.

---

## Authentication (two independent layers)

- **Identity, at the web host.** SCORPIOX BOT uses the same JWT as the rest of SCORPIOX: in a browser that is the `sx_token` cookie you get after signing in; in a client (an iOS app, a script, an AI agent) it is an `Authorization: Bearer <token>` header. The token must carry the `bot` permission (or `admin`), which is managed per account — you either have access to SCORPIOX BOT or you get a 401.
- **Node password, at each node.** Every node can additionally require a per-node password (its `BOT_PASSWORD`); routes that stream or accept keystrokes — the terminal, the input, and the native tmux routes — check it, accepting it as an `Authorization: Bearer` value, an `X-Bot-Password` header, or a `?pwd=` query parameter. The web UI stores each node's password in your browser's local storage, keyed per node, and never sends it to the cloud — passwords and secrets stay client-side. If the node has no password set, those routes are open, which is what you want on your own LAN.

For **self-hosted** deployments without a JWT provider there is an escape hatch: set `BOT_SINGLE_USER` to one owner id and drop the JWT flags entirely — every request is attributed to that owner, and a signed-in user still wins when one is present.

---

## Client integration: the whole UI is a JSON API

Every route in both the web UI and the node API accepts `?format=json` and returns machine-readable JSON. That is the intended automation surface: an AI agent, a CI job, or a mobile app can dispatch a prompt, stream the terminal, answer a permission gate, and read the result over plain HTTP — no browser required. There is also an MCP connector (the `connector-mcp` program in the API repo) if you want to expose a node's routes as MCP tools.

The route map on a node, in the order you will use it:

| Route | What it does |
|-------|--------------|
| `GET /sessions` | Live session list on this node, newest first, joined with each session's metadata (id, title, first message, model, provider, profile, working directory, attached, thinking) |
| `POST /sessions` | Spawn a new agent session: `{"project": "<repo>" or <folder path>}` plus optional `name`, `profile`, `assistant` |
| `DELETE /sessions?id=<session>` | Kill a session (agent and terminal with it) |
| `GET /sessions?projects=1` | List the project names you can spawn against |
| `GET /sessions_sse` | SSE stream of the live list; the full array is re-sent whenever anything changes |
| `GET /conversation?id=<session>[&after=N][&limit=N]` | The conversation, or a slice of it, in the standard `{"id","total","after","messages":[...]}` shape |
| `GET /history[?project=X][&id=<session>]` | Past sessions from disk, zero idle agent RAM; list mode or full transcript read mode |
| `POST /inbox?id=<session>` | Dispatch a message to a running agent (see below) |
| `GET /stream?id=<session>` | SSE live terminal stream — base64 ANSI frames for xterm.js |
| `GET /peek?id=<session>[&format=json|png|raw][&rows=N]` | A rendered snapshot of the terminal, in-process, in milliseconds |
| `GET /peek_sse?id=<session>` | SSE live raster stream — PNG data URLs as the screen changes |
| `POST /pty_input?id=<session>&action=raw|interrupt|resize` | Send keystrokes, break the agent mid-turn, or resize the terminal |
| `POST /input_tmux?id=<session>&action=raw|interrupt|resize` | Same, for a native tmux session |
| `GET /permission?id=<session>` / `POST /permission` | Read and answer the agent's permission gate |
| `GET /askuser?id=<session>` / `POST /askuser` | Read and answer the agent's questions |
| `GET /callbacks?id=<session>` / `POST` `PUT` `DELETE` | Inspect and manage scheduled callbacks |
| `GET /keepalive?id=<session>` / `PUT` | Read or set the context cache keep-alive window |
| `GET /resume?id=<session>` / `POST` | Resume a previous session, exactly the way `/resume` works in the TUI |
| `GET /stats?id=<session>` / `/stats_sse` | Token usage, cache countdown, branch, working directory |
| `GET /hashes?id=<session>` / `/hashes_sse` | Token and cost history, live |
| `GET /thinking_sse?id=<session>` | Live "the agent is thinking" signal |
| `GET /infobox?id=<session>` / `/infobox_sse` | Engine telemetry for the chat statusbar |
| `GET /code?id=<session>&file=<path>` | Read a file in the session's workspace |
| `GET /code_tree` / `/code_search` / `/code_diff` | Browse, search, and diff the workspace |
| `POST /code_save` | Write a file back, approval-gated |
| `GET /image?id=<session>&f=images/<name>` | Serve images the agent saved, so they render on any device |
| `POST /upload?id=<session>&name=<orig>` | Upload a file straight into the session's workspace (up to 25 MB) |
| `GET /commands?id=<session>` | The session's slash commands, with their options |
| `GET /models` | Live model discovery on this node |
| `GET /providers` | Provider login status; `POST` starts a login flow |
| `GET /envcfg[?reveal=1]` / `POST` / `PUT` | Read and edit the node's user configuration; secrets masked unless revealed |
| `GET /profiles` / `POST /profile_test` | Profiles on this node; a live inference test |
| `GET /node_filters` / `POST` / `PUT` / `DELETE` | Per-node rules for which sessions the dashboard shows |
| `GET /vnc` / `/rdp` / `/node_vnc` / `/node_rdp` | Live VNC and RDP targets the agent exposed |
| `GET /queue` / `POST` / `PUT` / `DELETE` | The Quick Message queue per project or folder |
| `GET /queue?project=X&file=F` | Read one queued message |

Everything on the node is backed by the session state on disk, and every write is atomic (temporary file, then rename), so a reader never sees a half-written file. And because the state is on disk, an idle agent costs you nothing: no resident process, no held RAM.

---

## The fleet dashboard

Open your SCORPIOX BOT web and sign in. The hub (`/`) is the whole fleet in one page:

- **Your nodes**, with their names, online state, and source (direct, cloud, or tunnel). Add or edit a node in the node modal — name, access URL, and (for nodes that use one) the node password and an active profile.
- **Live session cards**, per node. Each card shows the session's name and title, its first message, its model, provider, profile, and working directory, and whether anyone is attached. While the agent is thinking, the card pulses — you can see from the dashboard, on your phone, that a box across the LAN is working.
- **New** — a split button: spawn an **Agent Session** (choose the node, the repo or folder, optionally a name and a profile, or assistant mode for a general-purpose chat that does not bind to a repo), or spawn a plain **Terminal** (any shell or command), or drop a **Quick Message** into a project's queue.
- **Kill** — each card's menu offers **Kill session**, with a confirmation.
- **Diagnostics** — `/diagnostics` shows your identity, lets you manage the node passwords your browser stores, purge stale offline tunnels, and ping every node to check reachability.

---

## The session lifecycle, end to end

**1. Spawn it.** `POST /sessions` on the target node (or the **New** button in the dashboard). Point it at a project name or a folder path; optionally give the session a name and a profile. The response tells you the session id, and a failure comes back as a 422 with the first line of the error, so automation knows exactly why.

**2. Watch it.** Three streaming choices, pick the one your client needs:

- **`/stream`** — a Server-Sent Events stream of base64-encoded ANSI frames, the way xterm.js wants them. It is event-driven: about twelve frames per second while the screen changes, near-zero CPU when idle. Events are `init` (the session, dimensions, cursor state, scrollback depth), `frame` (a pane update), `scrollback` (a full history resync) and `scrollback_append` (incremental lines that scrolled off).
- **`/peek`** — a one-shot rendered snapshot. The terminal is rasterised to PNG in-process, in roughly five milliseconds — no external renderer, no Python. `?format=png` (or `raw`) returns the image; `?format=json` wraps it as base64 with the grid dimensions.
- **`/peek_sse`** — the same rendering as a live SSE stream: a PNG data URL every time the screen changes. This is what the dedicated full-page **Screen** view uses — zero reloads, no canvas, just the stream.

**3. Talk to it.** The chat page (`/chat?id=<session>`) renders the conversation server-side — the first paint has no network wait — and then keeps it live over the token stream and the history stream, with the statusbar telemetry and the infobox. Your message goes to `POST /inbox`, which takes `{"text": "...", "mode": "auto"}` (or a raw text body) and delivers it to the session's inbox — the same path the terminal uses. Three modes:

- `auto` — the message lands in the agent's inbox; the endpoint waits up to two seconds and returns `"consumed":true/false` once the agent has picked it up, so your client knows it arrived rather than merely that the server accepted it.
- `interrupt` — break the agent mid-turn and hand it your message at once.
- `queue` — park the message for the next turn. (Text that starts with `~` is queued automatically, matching the terminal's own convention.)

There is also the **Quick Message** queue: a small store of messages per project or folder that any session in that project will pick up. It is the "leave a note for the next agent" tool, and it is visible, editable, and deletable on the queue page.

**4. Answer it.** When the agent needs you, it does not block the world — it writes a pending request to the session's state, and you answer it whenever you are looking at a device:

- **Permission gate** — the agent's Bash permission request sits in `permission.json`; `GET /permission` shows it, `POST /permission` sends your verdict back. The `permission_sse` stream wakes your chat the moment a gate opens, so a prompt on a box across the LAN appears on your phone without polling.
- **Questions** — the agent's AskUserQuestion fires `askuser.json`; `POST /askuser` answers it, and the chat renders it as an inline card.
- **Callbacks** — `GET /callbacks` lists the agent's scheduled callbacks; `POST`, `PUT`, and `DELETE` add, update, and cancel them; `callbacks_sse` streams the changes.
- **Keep-alive** — `GET /keepalive` shows the context cache keep-alive window; `PUT` sets it (on, off, or a duration).
- **Resume** — `POST /resume` with a previous session's id hands `/resume` to the engine through the same inbox path, so a fresh session inherits the thread, the required skills, and the plan-mode prompt — remotely, exactly like the TUI.

**5. Read its workspace.** `/code`, `/code_tree`, `/code_search`, and `/code_diff` let any client read files, browse the tree, search, and see the live diff of the session's changes. The chat's **Changes** view and the code viewer page read the same state. `/code_save` writes a file back, approval-gated. Images the agent saved render through `/image` — including images inside a container that your browser could not reach directly.

**6. Close it.** `DELETE /sessions?id=<session>` kills the agent and its terminal. Or, while you are watching, `POST /pty_input?id=<session>&action=interrupt` sends the break first, so you can then send a final message.

---

## The fullscreen terminal

`/terminal?id=<session>` is an edge-to-edge xterm.js terminal with no shell clutter: live SSE frames in, keystrokes out, window resize synchronised back to the node. Two modes, and the choice is persisted:

- **Operator** — the keyboard is live. Keystrokes are routed to the node as raw terminal bytes (or to `input_tmux` for native tmux sessions), so you are typing in the agent's actual terminal — you can drop to a shell, run commands, and come back.
- **Read-only** — a pure spectator. The view keeps streaming, but the keyboard is inert, so a touch device or a shared screen cannot type into someone else's session by accident.

On touch devices there is a built-in virtual keyboard and accessory bar. The chat page also offers a peek terminal modal, for a quick glance at the raw terminal without leaving the conversation.

---

## The assistant page

`/assistant` is a full chat page for general-purpose conversations: a dedicated non-git workspace, so it never loads a repo's instructions or creates a worktree. It streams tokens as they are generated, renders rich Markdown (code highlighting, math, diagrams, tables), shows the agent's thinking and tool steps in a collapsible panel, and has a composer with paste and file upload plus an artifacts pane for the files the agent publishes. Use it for research and drafting on any node in your fleet — the `?node=` parameter works here exactly as everywhere else.

---

## Self-hosting

SCORPIOX BOT runs with zero cloud dependency. The moving parts are two `scorpiox-server` units (one for the node API on each machine, one for the web host), a node registry, and — optionally — a hub for tunnel nodes:

| Setting | What it does |
|---------|--------------|
| `RUNTIMECFG_BACKEND` | `file` (default): the node registry is plain JSON files on disk — no account, no database. `cloud`: your SCORPIO+ RuntimeConfig. `cloud,file`: both — reads are unioned, and one backend being down is not fatal |
| `RUNTIMECFG_DIR` | Where the file registry lives (default `/var/scorpiox-bot/nodes`) |
| `BOT_SINGLE_USER` | Names one implicit owner; use it when you have no JWT provider |
| `BOT_LOGIN_URL`, `BOT_ACCOUNT_URL`, `BOT_LOGOUT_URL` | Redirects for browsers without a cloud identity; unset means the feature is absent |
| `RELAY_ALLOW_RAW` | Lets `?node=` take a raw URL instead of a registry id — for your own LAN |
| `BOT_MESH_HUB` | Points the web host at the hub that carries tunnel nodes |

Example unit for a self-hosted web host:

```ini
ExecStart=/usr/local/bin/scorpiox-server -p 8155 \
  -e SERVER_SCRIPT_DIR=/opt/scorpiox-bot-web \
  -e SERVER_ROUTE_PREFIX=/ \
  -e RUNTIMECFG_BACKEND=file \
  -e RUNTIMECFG_DIR=/var/scorpiox-bot/nodes \
  -e BOT_SINGLE_USER=00000000-0000-0000-0000-000000000001 \
  -e RELAY_ALLOW_RAW=1
```

Build both repos with `make` (both also ship a Windows build), deploy the route directories to your host, and point a reverse proxy at it. The node API on each machine is the same pattern: a `scorpiox-server` unit with the API route directory and, per node, its `BOT_PASSWORD` if you want one.

---

## What it is, in one paragraph

SCORPIOX BOT is a pure-C, stateless control plane over your agent sessions: a per-node HTTP API that knows its own machine, and a pure-C web layer that turns any number of those machines into one dashboard, one chat, and one JSON API. Sessions are files, writes are atomic, streams are event-driven and idle-cheap, and every surface the browser uses is the same route a script uses. One address, every machine, every session, every keystroke — and nothing in the middle that has to stay awake.
