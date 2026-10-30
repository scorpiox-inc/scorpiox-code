# Live VNC Desktops in SCORPIOX BOT Chat

An agent is working on a machine that has a real desktop — a VM, a GUI app, a lab box — and you want to watch it, or drive it yourself, without logging into a jump host or opening a separate VNC client. **VNC** in SCORPIOX BOT puts that live desktop in your chat page: a small floating window you can drag and resize, or a full-page viewer in its own tab, both rendered by a bundled noVNC client and both talking to the desktop through SCORPIOX BOT, so the VNC server can sit behind the node and stay bound to loopback.

It is deliberately small: a title bar, a row of target tabs, the live screen, and a footer of four controls. Nothing is on screen until a target is offered — a session must expose a desktop, or a machine must be attached to a node — and when the offer is removed, the target disappears from the list.

Docs for SCORPIOX CODE @ `ce3b5cd`.

> **The whole idea in one line:** a session (or a node) offers VNC desktops by name; SCORPIOX BOT lists them in a floating window on the chat page and streams the live desktop to your browser — mouse, keyboard, and clipboard — with no separate VNC client to install.

---

## What it is (and what it is not)

- **Live, not a picture.** The window holds a real VNC session. You see the desktop as it changes, and — unless you switch on view-only — you can move the mouse, type, and use the clipboard. Reconnect is one click.
- **Two sources, one window.** The tab row has up to two groups:
  - **session** — desktops the agent in *your* session has exposed. Only the owner of that session sees them; a second account chatting elsewhere cannot pick them up.
  - **node** — machines attached to a node itself (added under **Settings → Nodes → Node VNC Machines**), independent of any session. They show for everyone who can use that node.
- **The browser never names a host.** SCORPIOX BOT shows only each target's `vid` and title — the host and port never leave the node. Your browser can only pick a target that was offered to you; it cannot dial an arbitrary host:port.
- **Nothing until offered.** The window opens empty ("No VNC exposed") until an agent exposes a desktop or someone attaches a machine to the node. Remove the offer and the tab disappears.
- **Off by default.** The float is hidden until you switch it on from the chat ··· More menu; your choice is remembered.

---

## Turning it on

In your SCORPIOX BOT chat, open the **··· More menu** and choose **VNC** ("Live desktop the agent exposed"). The window appears — empty at first, with the message "No VNC exposed" and a pointer to the two ways to fill it.

- **Your choice is remembered.** Close the tab and come back and the float is in the state you left it (open or closed).
- **To hide the menu item entirely:** under **Settings → Chat More Menu → VNC**. Switching it off removes **VNC** from the chat ··· More menu.
- **Close it any time** with the **×** in the window's title bar.

---

## The floating window

When the window is open, it is a small panel you can move around the chat page.

- **Move it.** Drag the title bar to position the panel anywhere on the screen. Your position and size are remembered and restored the next time you open it.
- **Resize it.** Drag the grip in the lower-right corner.
- **Maximise it.** The **□** in the title bar toggles the panel to (almost) full-window size; press it again to go back.
- **Reset.** The **Reset** button in the footer restores the default position and size and clears the remembered geometry.

### The tabs: session and node groups

The tab row lists every offered target as a pill labelled by its title (or `#<vid>` if it has no title). When both kinds exist, they are separated by a divider and grouped under small **session** / **node** labels; a single session target shows no group labels at all. Click a tab to switch to that desktop — the window disconnects from the old target and connects to the new one.

The window keeps the list fresh while you watch it: while nothing is connected it re-polls the target list every few seconds, so a desktop the agent exposes a moment later shows up on its own.

### The footer controls

- **View only.** Toggles the window to watch-only: no mouse or keyboard is sent to the desktop. Toggle it off to take control again. (While view-only is on, C-A-D is disabled too.)
- **C-A-D.** Sends **Ctrl+Alt+Del** to the desktop — the classic "I need a fresh logon screen" move. It is a no-op while view-only is on.
- **↻ Reconnect.** Disconnects and reconnects to the current target (or re-fetches the list if nothing is selected).
- **↗ Open in new tab.** Opens the same target in the full-page viewer (below), so you can drive the desktop in a window of its own.
- **State dot.** The footer also shows a status dot and word: orange **connecting…**, green **connected**, red on a failure (with the reason), and a plain "nothing exposed" when the list is empty. If the connection drops on its own you get "connection lost" and can press ↻.
- **Passwords.** The window itself carries no VNC credentials. If a target's VNC server asks for a password, your browser prompts you for it on connect; typing it there is all you do.

---

## The full-page viewer

For driving a desktop at full strength, open the full-page viewer: **↗** in the float's footer, or the **Open** button next to a machine in **Settings → Node VNC Machines**.

It is the same viewer, full-window: a top bar with a back link, the target's title, a target selector (shown only when more than one target is offered), a live status dot, and the same **View only**, **C-A-D**, and **↻ Reconnect** buttons — plus a theme toggle. The desktop fills the rest of the page. The URL encodes the target, so you can share or bookmark the exact desktop you are looking at.

---

## Node VNC machines in Settings

Machines that belong to a node rather than a session are managed under **Settings → Nodes → Node VNC Machines**:

- **Pick a node** in the drop-down at the top of the section; the list below shows that node's attached machines (a **Reload** button re-fetches it).
- **Add a machine:** type a **Host** (resolved on the node — `127.0.0.1` means the node itself), a **Port** (default 5900), and an optional **Title**, then **Add**.
- **Rename:** edit a row's title and click **Save**.
- **Open:** the **Open ↗** button opens that machine in the full-page viewer in a new tab.
- **Remove:** click **Remove**.
- **Every change is two-step.** Add, Save, and Remove all follow the same pattern as the rest of SCORPIOX BOT settings: the first click arms the button — it turns into **Confirm (Ns)** with a short countdown — and you must click it again inside the window to apply. Let it expire or the list change underneath you (HTTP 409) and the change is discarded; arm again and retry.

These machines are stored on the node, not in the session, so they outlive any one conversation and show up as the **node** group in every chat float on that node.

---

## How an agent offers a desktop

You do not offer anything yourself — you ask, and the agent does the rest. The mechanics are defined by the agent's **`preferred-vnc`** skill (a built-in skill in SCORPIOX CODE), and you do not need to know them to use the window. In one sentence: the agent maintains a small `vnc/index.json` inside the current session's directory, pointing at a VNC server that is already listening (bound to loopback on the node), and SCORPIOX BOT reads that file and offers it to you.

A few things that follow from that:

- **Offer, then remove.** Nothing is on the list until the agent writes the file; when the desktop shuts down or you are done, the agent removes the entry, and your float no longer offers it.
- **Only what you asked for.** A well-behaved agent exposes the desktop you named, not every VNC server it can reach, and it tells you the title to look for and that it is available from chat **··· More menu → VNC**.
- **Prefer loopback.** The bot connects from the node, so the VNC server can stay bound to `127.0.0.1`; your browser never needs to reach it directly.
- **Passwords live at the server.** The offer carries no credentials; if the server has a password, noVNC prompts you for it when you connect.

---

## Quick recipe: ask and watch

1. In your SCORPIOX BOT chat, open the ··· More menu and switch on **VNC**. The window appears (empty at first).
2. Tell the agent what to show: "spin up a desktop for me and expose it over VNC," or "let me drive that VM."
3. The agent starts the VNC server, offers it, and tells you the title. The window fills in and connects — the status dot goes green.
4. Drive it: click around, type, **C-A-D** when you are stuck, **View only** when you just want to watch, **↗** to move it into its own tab.
5. When you are done, tell the agent to stop it — the offer disappears from your list.

---

*Related: the SCORPIOX BOT chat page and the chat More menu, which also carries the companion Preview, notepad, and Drop floats; and the **Changes** viewer for live working-tree diffs.*
