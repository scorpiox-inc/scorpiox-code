# Remote Agent Control & Fleet Management with SCORPIOX BOT

You have a SCORPIOX CODE agent running on a box three rooms away, on a build server, or on a machine you only get to by SSH. You want to do more than SSH in and hope the terminal is still where you left it: you want to **send it a prompt**, **watch its output live**, **see what it is doing right now**, and **manage a whole fleet of them at once** — from a browser or from a script, on any machine.

SCORPIOX BOT is the remote-control layer for SCORPIOX CODE. It gives every running agent a small, fast, always-on control surface: a **headless HTTP API** for automation and a **web dashboard** for humans, both speaking to the same sessions on the same machines.

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** SCORPIOX BOT turns each SCORPIOX CODE session into something you can talk to, watch, and steer over the network — and it groups many of those machines into one fleet you manage from a single dashboard.

---

## What SCORPIOX BOT is

SCORPIOX BOT is two thin layers over the agent sessions that SCORPIOX CODE already writes to disk. Neither layer changes the agent itself.

| Layer | What it is | Who uses it |
|-------|------------|-------------|
| **Headless API** (`scorpiox-bot-api`) | A set of small HTTP routes: list sessions, send a message, stream the terminal, snapshot a screen, answer a question, manage callbacks. JSON in, JSON out. | Scripts, AI automation, mobile clients, your own tooling. |
| **Web dashboard** (`scorpiox-bot-web`) | A single-page, dark-theme web UI: session grid, live chat view, multi-node selector, panes, per-node profiles, cron, providers. | Humans in a browser. |

Both layers are **pure C** — compiled native executables, no runtime, no Python. Each HTTP route is its own tiny executable served by a generic web server under a standard CGI contract, so a route is trivially small, fast, and independent. Every route also answers `?format=json`, which is what makes the whole thing friendly to AI agents and headless clients.

The split is deliberate. The API is the source of truth and the thing you script against. The web dashboard is a client of the same session data, pointed at one node at a time. If you only ever use the browser, the API is still underneath you; if you only ever script, you never need the browser.

---

## The remote session lifecycle

A SCORPIOX CODE session is a live terminal process with a persistent record on disk. SCORPIOX BOT exposes that session across four operations: **start**, **peek**, **stream**, and **send**.

### 1. List and start sessions

`GET /sessions` returns every live session on the node, newest first: its id, model, provider, working directory, profile, and when it started. That list is what the dashboard's session grid and the API's session picker both render.

To start one, `POST /sessions` takes a target — either a **project name** or a **folder path** — plus an optional session name and an optional **profile** (the model/settings preset for that one session). The same endpoint powers the dashboard's "new session" flow and a `curl` one-liner from a script.

There is also `POST /terminals` for a **plain shell with no agent** — open a working folder and run a command or an interactive shell — when you want a remote terminal rather than an agent.

### 2. Peek at a session (one snapshot)

When you want to *see* a session without attaching to it, you peek. A peek is a single, fast render of the session's current screen.

- **`/peek`** returns a rendered image of the terminal — as **JSON** (the PNG wrapped as base64, plus dimensions) or as a **raw PNG**. The render happens in-process at millisecond latency; there is no screenshot process you are spawning or waiting on. This is the "what is the agent doing *right now*" call: a thumbnail in a card, or a screenshot to attach to a log.
- **`/terminal_tmux`** and **`/terminal_scorpiox_multiplexer`** give you a **text** snapshot of a pane instead — the plain terminal contents, or the same thing as JSON. That is the cheap "just give me the screen as characters" form.
- **`/conversation`** and **`/hashes`** give you the structured conversation as incrementally-syncable events (read everything, or read only what is newer than a point you already have). That is how a client stays caught up without re-downloading history.
- **`/stats`** gives you the per-session diagnostics: token usage, cache countdown, branch, working directory.

A peek is stateless and cheap; it does not disturb the session.

### 3. Stream a session (live output)

A peek is a photo; a stream is a video. **`/stream`** is a Server-Sent Events endpoint that pushes the terminal's ANSI frames to you in real time as they change — capped at roughly a dozen frames per second while the screen is moving. When the screen is not changing, nothing is pushed: a periodic keepalive keeps the connection alive instead.

That quiet-when-idle property is what makes it safe to leave open: a long-running build you are not watching costs you almost nothing. The same streaming pattern exists for the session list (`/sessions_sse`) and for the interactive agent channels described below, so a dashboard can keep a whole fleet's status live with a handful of long-lived connections.

The browser's chat view, the terminal view, and the panes grid are all built on these streams — you are not polling, you are being pushed.

### 4. Send to a session (dispatch a prompt)

The write side is **`/inbox`**. You POST a message to a session, and it lands in that session exactly the way you would type it in the terminal. The `mode` parameter controls how it is delivered:

| `mode` | Behaviour |
|--------|-----------|
| `auto` (default) | Delivered normally; the session picks it up on its next turn. |
| `interrupt` | Cancels whatever the agent is doing right now and delivers your message immediately. This is your "stop and do this instead." |
| `queue` | Queues the message to be picked up after the current work finishes. |

There is a related **`/queue`** for **quick messages** that are not tied to one session you have to be looking at — drop a note against a project or folder and the next agent run there picks it up. That is the "leave a sticky note for the agent" path, distinct from the live `/inbox` dispatch.

### 5. Talk back to the agent (interactive channels)

An agent does not only receive prompts; it asks questions and requests permission. SCORPIOX BOT exposes those as routes too, so a remote client can answer them:

- **`/askuser`** — the agent's "question for a human" tool. Read the pending question, post an answer.
- **`/permission`** — the agent's "may I run this command" gate. Read the pending request, approve or deny.
- **`/callbacks`** — read or manage the agent's scheduled callbacks remotely (see the Scheduled Callbacks page), so an autonomous loop can be inspected or stopped without the terminal.
- **`/thinking`**, **`/infobox`**, **`/commands`** — small read endpoints for the agent's state: is it thinking and for how long, its info box, and the slash commands available in that session.

Each of these has a streaming sibling (for example, the live feed of pending questions) so a dashboard can light up the moment the agent needs you, then deliver your answer the moment you make it.

The net effect: the *entire* interactive surface of a terminal agent — prompt in, output out, question, permission, scheduled loop — is reachable over HTTP. You can drive a SCORPIOX CODE session as if it were a service, not a person at a keyboard.

---

## Two ways to talk to it: API vs dashboard

### The headless API (for automation)

The API is plain HTTP + JSON. Every route returns `application/json` (or an image where a snapshot is requested), and the whole surface is discoverable by trying routes. A minimal loop in any language looks like:

```bash
# List live sessions on the node
curl -s "https://bot.scorpiox.net/sessions"

# Snapshot a session's screen as an image (PNG) or base64 JSON
curl -s "https://bot.scorpiox.net/peek?id=<session>&format=png" -o screen.png
curl -s "https://bot.scorpiox.net/peek?id=<session>&format=json"

# Snapshot a session's screen as plain text
curl -s "https://bot.scorpiox.net/terminal_tmux?id=<session>&format=raw"

# Dispatch a prompt to a session
curl -s -X POST "https://bot.scorpiox.net/inbox?id=<session>&mode=auto" \
     -H 'Content-Type: application/json' \
     -d '{"text": "Run the test suite and summarize the failures."}'

# Stream the terminal live (SSE)
curl -sN "https://bot.scorpiox.net/stream?id=<session>"
```

Because every route is JSON, an AI agent or a CI job can do with SCORPIOX BOT exactly what you do in the browser: check status, send work, read results. There is no private protocol to learn.

### The web dashboard (for humans)

The dashboard is a single page that opens on a **session grid** for the node you have selected. From there:

- **Chat view** — a live, terminal-faithful view of one session with an input box that dispatches through `/inbox`, slash-command help, and the interactive channels (questions, permissions, callbacks) rendered inline as they appear.
- **Panes** — a tmux-style grid of *every* active session on the node at once, each a live pane you can toggle between a chat view and a raw terminal.
- **Multi-node selector** — a dropdown of every node in your fleet (see below). Pick one and the whole dashboard re-points at it: sessions, chat, panes, profiles, cron, providers.
- **Per-node administration** — edit that node's configuration profiles, view and manage its scheduled cron jobs, and log it into model providers, all from the same page.

The dashboard and the API are the same system from two angles. The dashboard's fetch calls *are* the API routes, plus the browser's auth token. Nothing in the browser has a capability the API does not have.

---

## Authentication

SCORPIOX BOT is not open to the internet. Two layers of identity sit in front of the routes.

**1. You (the operator) are identified by JWT.** The API and the dashboard authenticate you with a token issued by the same identity provider that signs the rest of the platform. A browser presents it as a cookie after you sign in; a script or mobile client presents it as an `Authorization: Bearer` header. Every route checks it:

- **401** — no token, an invalid token, or a token from the wrong issuer.
- **403** — a valid token, but your account does not carry the permission to use SCORPIOX BOT.

Access is deliberately **admin-managed**: an account must be granted the bot permission before any route will answer for it. There is no self-serve signup to remote agent control.

**2. The node is protected separately.** Each node can set its own node password. When a node password is set, the terminal/input routes require it (sent as a header or query value); when it is unset, the route is open — which is the self-hosted-on-your-own-LAN case where the box is already behind your firewall. The dashboard keeps each node's password in your browser's local storage and sends it only to that node, so the operator identity (JWT) and the per-node credential are two separate things.

The practical consequence: proving *who you are* is a platform-wide concern, and proving *you may reach this machine* is a per-node concern. A signed-in user talking to a node they own is the normal, fully-authorized path.

---

## Multi-node clustering: the fleet

A single node is useful. A fleet is the point. SCORPIOX BOT treats every machine running a bot API as a **node**, and the dashboard knows a node is just an id plus a URL plus, optionally, a way to reach it.

### Where a node can come from

The dashboard can learn about nodes from up to three sources, and they are merged into one list with a "source" on each entry so you can tell where a node came from:

| Source | What it is | When you use it |
|--------|------------|-----------------|
| **Local files** | Plain JSON node files the operator keeps on the box. Read-only from the web. | Self-hosting a small, fixed set of machines you control by hand. |
| **Cloud registry** | A shared, account-scoped registry so a signed-in user sees the fleet that belongs to them. | A team or a personal cloud where nodes are registered once and appear on every box. |
| **Reverse tunnel (mesh)** | A node behind NAT that dials *out* to a hub and holds the connection open. The hub then forwards ordinary requests down that open socket. | Reachable-anywhere nodes: a laptop at home, a box with no public address, a dev machine on a phone network. |

The first two are **static** (you registered them); a tunnel node is **live** (it appeared because that machine is currently dialed in). A node id that exists in your own registry always wins over a tunnel with the same id, because the deliberate entry beats whatever machine happened to connect.

### How a node is reached

The dashboard resolves the selected node to a URL and points the chat, stream, and API calls at it. Three reachability modes cover the real world:

- **Direct** — the node has an address the dashboard can reach (your LAN, a VPS, a tunnel endpoint).
- **Relay** — for a node that can only be reached through an allowed path on your own network.
- **Auto** — the default: try the right thing and fall back.

For a machine behind NAT you cannot reach at all, the **reverse tunnel** is the answer: the node runs a small hub-facing client that opens the connection for you, and the dashboard reaches it as if it were direct. A tunnel node carries an `online` flag from the hub, so the fleet view honestly shows you which remote machines are actually reachable right now.

### Self-hosting without a cloud account

None of this requires a cloud account. The node registry can be backed by **local files only**, and the dashboard can run with a single implicit owner instead of the platform's identity provider. That combination — file-backed registry, single-user identity, no external network — is the "my own box, my own LAN, no account" deployment. The same dashboard, the same routes, the same fleet view; the only difference is where the node list and the identity come from.

---

## A typical workflow

1. **Open the dashboard** and pick a node from the fleet selector. The session grid populates live.
2. **Start a session** against a project or folder, choosing a profile (model preset) for it.
3. **Watch it work** in the chat view or in the panes grid — output streams in; nothing polls.
4. **Intervene when you need to.** Send a follow-up, or `interrupt` to cut in and redirect, or answer the permission prompt and question that pop up inline.
5. **Leave it running.** The stream goes quiet and costs nothing while idle. Come back later and peek at a snapshot to see where it is, or dispatch the next task.
6. **Scale out.** Register the build server, the home laptop (tunneled), and the lab box; they all appear in one selector, each with its own sessions, profiles, cron, and providers.

The same steps work from a script: `GET /sessions` to find the id, `POST /inbox` to dispatch, `/stream` to follow, `POST /permission` to unblock — the headless path is the browser path without the browser.

---

## Gotchas

- **A node you cannot reach is not a node you can use.** A tunnel node shows offline if its machine is asleep or its uplink is down; the fleet list is honest about reachability, but a dashboard pointed at an unreachable node will not invent sessions.
- **`interrupt` really does interrupt.** Use it to redirect a running agent; a plain `auto` message waits its turn.
- **The API and the dashboard share one identity.** If a signed-in account lacks the bot permission, both the browser and the scripts get a 403 — grant the permission to the account, not to the machine.
- **Node passwords are per-node, not per-user.** The dashboard stores them in your browser's local storage and sends each one only to its own node.
- **The cloud registry path is the default and the stable one.** Switching the registry backend (cloud, files, or a mix) changes where the node list comes from, not how the routes behave — a route you can hit today behaves the same after a backend change.
- **Streaming is push, not poll.** Do not build a poller against a stream; open one connection and let it push. The idle-cost guarantee only holds if you let the endpoint decide when to send.

---

## Related

- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
- [Using the /keepalive Command](keepalive.md)
- [Project Instructions (CLAUDE.md / AGENTS.md)](project-instructions.md)
