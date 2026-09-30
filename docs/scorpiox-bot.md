# Remote Agent Control and Fleet Management with SCORPIOX BOT

Running an agent at your desk is one thing. Running a *fleet* of agents — across several machines, several projects, several people — and being able to see what each one is doing, type into any one of them, and kick off new work from anywhere, is a different product entirely. SCORPIOX BOT is that product: a thin, always-on layer that wraps your running SCORPIOX CODE sessions so you can drive them remotely, watch their output live, and manage the whole set of machines they run on.

It is built from two deliberately small pieces. **`scorpiox-bot-api`** is a headless HTTP API — a set of tiny endpoints that read and write the session state SCORPIOX CODE already keeps on disk, with no changes to the agent itself. **`scorpiox-bot-web`** is a pure C web dashboard on top of that API: a single page for your whole fleet, a live chat view for each session, and a full edge-to-edge terminal. Because both halves are plain HTTP and plain C, the API is something you can script and point an AI at, and the web is something you open in a browser. Same data, two front doors.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** SCORPIOX BOT turns every SCORPIOX CODE session into something you can list, stream, type into, and spawn from any browser or script — on the machine it runs on, or across a cluster of machines, with one dashboard and one set of REST calls.

---

## What it is (and what it is not)

The single most important thing about SCORPIOX BOT is how little it changes. It does **not** fork SCORPIOX CODE, add an agent-in-the-middle, or proxy your model calls. Each running agent session already writes its own state — the conversation, the live terminal pane, the per-session event stream — to your filesystem under `.scorpiox/sessions/`. SCORPIOX BOT reads and writes *that* state through ordinary HTTP.

That means a few honest properties you should rely on:

- **The agent is unchanged.** Everything SCORPIOX CODE does locally still works the same way. SCORPIOX BOT is an additional surface over the same sessions, not a replacement.
- **The session files are the source of truth.** The API and the dashboard are both just views over the same on-disk state. There is no separate "remote copy" that can drift from the real thing.
- **It is stateless on the wire.** Each endpoint does one thing and answers. You can call them from a browser, from `curl`, or from an automation pipeline, and they behave identically.
- **It is opt-in per node.** A machine only appears in your fleet if you (or an admin) registered it. Nothing about a node reaches the dashboard until it is added.

Two front doors, one dataset:

| Half | What it is | Who it is for |
|------|------------|---------------|
| **`scorpiox-bot-api`** | Headless HTTP endpoints over the session files. JSON in, JSON out, SSE for live streams. | Scripts, CI, AI automation, mobile clients, anyone who wants to drive a session programmatically. |
| **`scorpiox-bot-web`** | A pure C single-page dashboard: fleet overview, per-session chat, live terminal, node management. | People. Humans who want to see what the fleet is doing and type into it from a browser. |

If you want to *watch and drive*, use the web. If you want to *automate* — feed prompts, harvest output, orchestrate a batch of sessions — use the API. Both hit the same endpoints, so anything you learn about one you already know on the other.

---

## The two pieces

### `scorpiox-bot-api` — the headless core

This is the part that actually touches your sessions. It is a collection of small, self-contained routes, one per concern, each served the same way. There is no persistent server process holding state; each request is handled on its own, which keeps it trivially cheap to run and easy to reason about.

The routes fall into a few groups, and you only ever need to know the handful that matter to you:

| Group | Endpoints | What they do |
|-------|-----------|--------------|
| **Sessions** | `GET /sessions` | List every live session on the node, newest first: name, project or working folder, profile, whether it is attached, and a live "thinking" flag when the agent is mid-turn. |
| | `POST /sessions` | Spawn a new session for a project name or a folder path, optionally with a custom name and profile. |
| | `DELETE /sessions?id=<session>` | Kill a session. |
| **Conversation** | `GET /conversation?id=<session>` | Read the full conversation, or a slice after a given index with an optional limit. Backed by the per-session event files, so it is exactly what the agent has. |
| | `GET /hashes_sse?id=<session>` | A live server-sent stream that pushes each new message as it lands. This is how the chat view stays real-time without polling. |
| **Input** | `POST /inbox?id=<session>` | Send a message into a running session. Accepts a `mode` of `auto`, `interrupt`, or `queue` so you can choose whether to nudge, cut in, or wait your turn. |
| | `POST /pty_input?id=<session>` | Low-level terminal input: raw keystrokes, viewport resize, or a signal interrupt. This is what makes the live terminal actually interactive. |
| **Output / streaming** | `GET /stream?id=<session>` | A live server-sent stream of the terminal pane — raw ANSI frames, pushed only when the screen actually changes, so an idle session costs essentially nothing. |
| | `GET /peek?id=<session>` | A one-shot capture of the current terminal screen, as JSON or as a rendered image. "What is it looking at right now?" answered in a single call. |
| | `GET /peek_sse?id=<session>` | The continuous version of peek: a stream of rendered screen frames. |
| **Fleet / node** | `GET /env` | Safe diagnostics: what is running, what the node sees. Read-only. |
| | `GET /providers`, `GET /profiles`, `GET /projects` | What is available on *this* node: which providers are logged in, which profiles exist, which projects are launchable. |
| **Misc** | `GET /api/ping` | Liveness check. |

Two properties of the API are worth calling out because they are what make automation sane:

- **Every route is JSON.** There is no proprietary wire format. You can `curl` any of these from a laptop, a cron job, or an agent, and parse the answer with the same code you would for any other REST service.
- **Streaming is server-sent events, not websockets.** The live endpoints (`/stream`, `/hashes_sse`, `/peek_sse`, `/sessions_sse`) are plain SSE. That means they work through the same HTTP gate as everything else, they are easy to consume, and they cost almost nothing while idle — a frame is only pushed when the screen changes, and a keep-alive otherwise.

### `scorpiox-bot-web` — the dashboard

The web half is a single-page application written entirely in C, with no build step, no JavaScript bundle, and no Python. It sits in front of the API and gives you three views:

- **The fleet hub** (`/`) — every node you can see, every live session on each, the ability to spawn a session or a plain terminal, and a node selector to jump between machines.
- **Chat** (`/chat?id=<session>`) — the conversation with the agent, rendered as cards, with a live status bar, the infobox telemetry, and the same real-time streaming the API exposes. This is the "read the agent's output and answer it" view.
- **Terminal** (`/terminal?id=<session>`) — an edge-to-edge, full-screen xterm terminal bound to the live session: keystrokes go in, frames stream out, and the pane resizes with your window. This is the "be there" view.

There is also a **screen** view that is purely an observer — a live picture of the terminal with no input — and a **diagnostics** page for node health, credentials, and tunnels.

The dashboard is the friendliest way to learn what the API can do, because every control you click is a direct call to one of the endpoints above. When something on the dashboard does what you want, the matching HTTP call is right there in the request.

---

## The remote session lifecycle

A SCORPIOX BOT session is just a SCORPIOX CODE session you can reach from outside the machine. The lifecycle is the same one you already know — start it, talk to it, watch it, end it — and each step maps to a small number of calls.

### 1. Discover what is there

`GET /sessions` is the entry point. It returns every live session on the node, sorted newest first, with enough to act on:

```json
[
  {
    "name": "fix-auth-flow",
    "id": "fix-auth-flow",
    "worktree": "/codebases/myapp",
    "profile": "agent-local",
    "attached": false,
    "thinking": true,
    "thinking_started": 1759000000000
  }
]
```

The `thinking` flag is the one you will use most in automation: it tells you the agent is mid-turn, so you know whether to interrupt, queue, or wait. `attached` tells you whether a real terminal client is on it. `worktree` and `profile` tell you what you launched and where.

### 2. Spawn one

`POST /sessions` starts a new session. The target is either a **project name** (a directory under your codebases, e.g. `myapp`) or a **folder path** anywhere on the host (`/tmp/scratch`, `~/work`, `D:\work`). Both arrive the same way; a path is just a project name that happens to start with `/`, `~`, `\`, or a drive letter.

```bash
curl -X POST "$BOT/sessions" \
  -H "Content-Type: application/json" \
  -d '{"project": "myapp", "name": "fix-auth-flow", "profile": "agent-local"}'
```

The answer is the new session's name. If the node cannot create it — unknown project, name already taken — you get a clear `422` with the reason rather than a silent success. Once it is up, it shows in `/sessions` like anything else.

You can also spawn a **plain terminal** with no agent at all, for the "I just want a shell on that box" case. That is a separate call from the web's "new terminal" action and does not create an agent session.

### 3. Watch it work

This is where the two streaming modes earn their keep, and it is worth being precise about because they answer different questions.

**"What is the agent's conversation?"** Use the conversation stream. `GET /conversation?id=<session>` gives you the whole thing at once; `GET /hashes_sse?id=<session>` gives you each new message as it lands, live. This is the text — the model's replies, the tool calls, the results — and it is the feed the chat view renders.

**"What is the terminal showing?"** Use the screen stream. `GET /stream?id=<session>` pushes raw ANSI frames of the live pane, only when it changes. `GET /peek?id=<session>` is the one-shot version — a single capture of the current screen, as JSON or as an image, for the "just show me the picture" case. `GET /peek_sse` is the continuous version.

In practice the chat view uses the conversation stream and the terminal view uses the screen stream, and you can watch both at once. An idle session emits nothing but a keep-alive, so leaving a view open is effectively free.

### 4. Talk to it

`POST /inbox?id=<session>` is how you send a message into a running session. The body is the text; the `mode` decides what happens when the agent is busy:

| Mode | Behaviour |
|------|-----------|
| `auto` | The default. The agent decides when to pick it up. |
| `interrupt` | Cut into the current turn. The new message takes priority. |
| `queue` | Wait your turn. Delivered when the agent reaches a natural break. |

There is also a low-level `POST /pty_input` for raw keystrokes, resize, and interrupt — which is what the live terminal uses rather than the inbox, because typing into a shell is not the same as sending the agent a prompt.

The rule of thumb: **inbox for the agent, pty_input for the shell.** If you are driving the model, send a message. If you are at the prompt, type.

### 5. End it

`DELETE /sessions?id=<session>` kills the session. That is the whole thing. The session's files remain on disk (so you can inspect what happened), the pane goes away, and it disappears from `/sessions`.

---

## Multi-node clustering and the fleet

The single-machine case is the easy one. The reason for SCORPIOX BOT is the case where you have several machines — your desktop, a build box, a server, a box at home behind a router — and you want them to look like one fleet.

### Nodes

A **node** is a machine running the API, registered so the dashboard knows where to reach it. The dashboard keeps a node selector: pick a node, and every view — sessions, chat, terminal — resolves against that node's API. The same session name on two different nodes is two different sessions, and the selector is what keeps them apart.

Nodes come from a registry, and the registry can be backed however you like:

- **A local file registry** for a self-hosted setup — plain JSON on the node, no account, no network dependency. This is the default and it is the one you want if you are running your own fleet on your own LAN.
- **A cloud registry** for a setup that wants nodes to follow you across devices, with each node reporting where it lives and how to reach it.
- **Live tunnels** for the node that cannot be reached by IP at all — a machine behind NAT that dials *out* to a hub and holds a connection open, so the dashboard can reach it even though nothing can reach it directly.

The three sources are layered, and an entry that is deliberately registered always beats one that merely appeared. Every node the dashboard shows carries a `source` and an online/offline state, so you can tell at a glance whether a node is a registered one you added or a tunnel that just dialed in — and whether it is actually up right now.

### Reaching a node

For each node, the dashboard resolves a target URL and proxies your calls to it. The same endpoint works the same way no matter which node you are on: the dashboard takes the request, finds the node's address from the registry (or from the live tunnel if that is the source), and forwards it, presenting your identity along the way. That is why a chat view on a tunnelled node behaves exactly like one on a LAN node — the node resolution is a detail the dashboard absorbs.

You can also point the dashboard at a node by raw URL on your own LAN, which is handy for trying a node before you register it. That mode is intentionally something you enable yourself and is not something you would do for a public fleet.

### What clustering buys you

- **One selector, many machines.** Spawn, watch, and kill sessions across the whole set without SSHing anywhere.
- **Per-node context.** Profiles, providers, and projects are properties of a node, and the dashboard reads them from the node you are looking at, so "what can this box run" is always accurate for the box you mean.
- **Honest liveness.** A node that is down or a tunnel that has dropped shows as offline instead of silently failing, and the dashboard degrades to what it can actually reach rather than hanging.

---

## Authentication and access

SCORPIOX BOT is a remote-control surface, so it is gated by default. Every API route checks authentication before it does anything, and the answer is one of:

| Outcome | Meaning |
|---------|---------|
| `401` | No valid identity. The token is missing, invalid, or from the wrong issuer. |
| `403` | Valid identity, but it does not have the permission to use the bot. |
| `404` | Authenticated, but that session does not exist on this node. |

There are two ways to authenticate, and which one you use depends on whether you are on a single trusted box or in a shared setup.

**A node password** for a single-user, self-hosted box. If the node is set up with a local password, a client presents it as a `Bearer` token or a header, and that is all there is to it. No account, no external service. This is the mode to reach for when the API is only ever talking to you.

**A signed-in identity** for a shared or cloud setup. A normal signed-in user presents their token, the route verifies it, and then checks that the identity carries the permission to use the bot. Access is admin-assigned — being able to sign in is not the same as being allowed to drive the fleet, and the permission check is what separates the two. The dashboard uses the same identity: a signed-in browser is authenticated once, and every node it touches presents that identity on the user's behalf.

Two practical consequences of this:

- **You are always driving as a real identity.** Even on the single-node password path, the call is attributable; there is no anonymous lane into a fleet.
- **The dashboard never smuggles a credential it should not have.** When it proxies your request to a node, it presents *your* identity, not a service credential of its own, so the node still sees who is actually in the browser.

---

## Choosing the front door

The decision is simple and you rarely need both at once:

- **Use the web dashboard** to operate the fleet by hand: see what is running, open a chat, jump into a terminal, spawn work on another box. It is the fastest way to understand the system, and it is what you will use day to day.
- **Use the API** to automate: a script that dispatches a prompt to every box in the fleet and waits for the result; a CI step that spins up a session, streams its output, and collects it; an AI that orchestrates other SCORPIOX CODE sessions by calling the same endpoints. Because every route is plain JSON and streaming is plain SSE, the API is consumable from anywhere HTTP goes.

The two share the same data and the same session lifecycle, so the boundary is not about capability — anything the dashboard does, the API does too. It is about who is on the other end: a person, or a program.

---

## Related

- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Project Instructions in SCORPIOX CODE](project-instructions.md)
