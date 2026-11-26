# Live VNC Desktops in SCORPIOX BOT Chat

An agent is working on a machine that has a real desktop — a VM, a GUI app, a lab box — and you want to watch it, or drive it yourself, without logging into a jump host or opening a separate VNC client. **VNC** in SCORPIOX BOT puts that live desktop in your chat page: a small floating window you can drag and resize, or a full-page viewer in its own tab, both rendered by a bundled noVNC client and both talking to the desktop through SCORPIOX BOT, so the VNC server can sit behind the node and stay bound to loopback.

It is deliberately small: a title bar, a row of target tabs, the live screen, and a footer of controls. Nothing is on screen until a target is offered — a session must expose a desktop, or a machine must be attached to a node — and when the offer is removed, the target disappears from the list.

Docs for SCORPIOX CODE @ `3f5b938`.

> **The whole idea in one line:** a session (or a node) offers VNC desktops by name; SCORPIOX BOT lists them in a floating window on the chat page and streams the live desktop to your browser — mouse, keyboard, and clipboard — with no separate VNC client to install.

---

## What it is (and what it is not)

- **Live, not a picture.** The window holds a real VNC session. You see the desktop as it changes, and — unless you switch on view-only — you can move the mouse, type, and use the clipboard. Reconnect is one click.
- **Two sources, one window.** The tab row has up to two groups:
  - **session** — desktops the agent in *your* session has exposed. Only the owner of that session sees them; a second account chatting elsewhere cannot pick them up.
  - **node** — machines attached to a node itself (added under **Settings → Nodes → Node VNC Machines**), independent of any session. They show for everyone who can use that node.
- **The browser never names a host.** SCORPIOX BOT shows only each target's `vid` and title — the host and port never leave the node. Your browser can only pick a target that was offered to you; it cannot dial an arbitrary host:port.
- **Nothing until offered.** The window opens empty ("No VNC exposed") until an agent exposes a desktop or someone attaches a machine to the node. Remove the offer and the tab disappears.
- **Off by default.** The float is hidden until you switch it on from the chat **··· More** menu; your choice is remembered.
- **Self-contained.** The noVNC client (v1.7.0) is bundled with SCORPIOX BOT and served from the bot-web host — no CDN, no internet access needed by the viewer.

---

## Turning it on

In your SCORPIOX BOT chat, open the **··· More** menu and choose **VNC** ("Live VNC desktop the agent exposed (noVNC)"). The window appears — empty at first, with the message "No VNC exposed" and a pointer to the two ways to fill it.

- **Your choice is remembered.** Close the tab and come back and the float is in the state you left it (open or closed).
- **To hide the menu item entirely:** under **Settings → Chat More Menu → VNC**. Switching it off removes **VNC** from the chat ··· More menu.
- **Close it any time** with the **×** in the window's title bar.

---

## The floating window

When the window is open, it is a small panel you can move around the chat page.

- **Move it.** Drag the title bar to position the panel anywhere on the screen. Your position and size are remembered and restored the next time you open it.
- **Resize it.** Drag the grip in the lower-right corner.
- **Maximise it.** The **□** button in the title bar toggles the panel to (almost) full-window size; press it again to go back.
- **Reset.** The **Reset** button in the footer restores the default position and size and clears the remembered geometry.

### The tabs: session and node groups

The tab row lists every offered target as a pill labelled by its title (or `#<vid>` if it has no title). When both kinds exist, they are separated by a divider and grouped under small **session** / **node** labels; a single session target shows no group labels at all. Click a tab to switch to that desktop — the window disconnects from the old target and connects to the new one.

### Footer controls

The footer bar at the bottom of the window carries the controls you use while connected:

- ****Credentials**..** Opens the credentials dialog to change the username and password you are using. Useful when a server requires different credentials than the one currently set.
- **View only.** Toggles view-only mode. When on, the screen still updates but your mouse and keyboard no longer reach the remote desktop. Handy when you just want to watch without risk of disturbing what the agent is doing. The button highlights while active.
- **C-A-D.** Sends a Ctrl+Alt+Del key sequence to the remote desktop. This is the classic way to get the Windows security screen (or reset a locked session). Only works when view-only is off.
- **↻ (Reconnect).** Disconnects and reconnects to the currently selected target. Useful if the connection drops or the remote desktop restarts.
- **↗ (Open in new tab).** Opens the same desktop in a full-page viewer in a new browser tab. Same controls, bigger screen. The new tab is independent — closing it does not affect the float.
- **Reset.** Restores the window to its default position and size (lower-right corner, standard dimensions) and clears the saved geometry.

### Status indicator

The small dot in the footer (next to the status text) shows the connection state:

- **Green** — connected
- **Orange** — connecting
- **Red** — error or disconnected
- **Grey** — idle (nothing selected)

The text next to the dot gives a short description: "connected", "connecting…", "credentials required", "auth failed", "disconnected", "connection lost", or "nothing exposed".

---

## Credentials

Most VNC servers require a password, and some (macOS Screen Sharing / ARD) require both a username and a password. SCORPIOX BOT does not store these in the target list — the credentials are separate from the offer.

- **First connection.** If the server requires credentials, noVNC pauses the handshake and the credentials dialog appears. Enter the password (and username if needed) and click **Connect**.
- **Change credentials mid-session.** Click **Credentials** in the footer at any time to re-enter credentials without disconnecting.
- **Username is optional** for most servers. The dialog shows "(optional — macOS / ARD)" next to the username field. It becomes required only if the server specifically asks for it.

---

## How an agent offers a desktop

You do not offer anything yourself — you ask, and the agent does the rest. The mechanics are defined by the agent's **`preferred-vnc`** skill (a built-in skill in SCORPIOX CODE), and you do not need to know the internals to use the window. In one sentence: the agent writes a small `vnc/index.json` file inside the current session's directory, pointing at a VNC server that is already listening (bound to loopback on the node), and SCORPIOX BOT reads that file and offers it to you.

A few things that follow from that:

- **Offer, then remove.** Nothing is on the list until the agent writes the file. When the desktop shuts down or you are done, the agent removes the entry, and your float no longer offers it.
- **Only what you asked for.** A well-behaved agent exposes the desktop you named, not every VNC server it can reach, and it tells you the title to look for and that it is available from chat **··· More → VNC**.
- **Prefer loopback.** The bot connects from the node, so the VNC server can stay bound to `127.0.0.1`; your browser never needs to reach it directly.
- **Passwords live at the server.** The offer carries no credentials; if the server has a password, noVNC prompts you for it when you connect.

---

## Node VNC machines (Settings editor)

Sometimes the desktop you want to watch is not tied to a specific session — it is a machine that should be available to anyone who can use that node. You add these under **Settings → Nodes → Node VNC Machines**.

- **Pick a node.** The section has a node selector at the top. Choose the node the machine is attached to.
- **Add a machine.** Enter the host (resolved on the node — `127.0.0.1` means the node itself), the port (default 5900), an optional title, and an optional user hint (for macOS/ARD). Click **Add**.
- **Two-step confirm.** Every change (add, rename, remove) follows a two-step pattern: the first click arms the action and the button changes to **Confirm (Ns)** with a countdown. Click it again within the window to apply. If the countdown expires or the list has changed, the button resets and you can try again.
- **Edit existing.** Each machine row shows its `#vid`, title, and user fields. Change the title or user and click **Save** (same two-step confirm).
- **Remove.** Click **Remove** on a row, then confirm. The machine is removed from the node and disappears from the chat VNC float.
- **Open in a new tab.** Each row has an **Open ↗** button that launches the full-page viewer for that machine directly.

These machines are stored on the node, not in the session, so they outlive any one conversation and show up as the **node** group in every chat VNC float on that node.

---

## Full-page viewer

The **↗** button in the float footer (or the **Open ↗** button in the Settings editor) opens a standalone full-page VNC viewer in a new browser tab. It is the same noVNC client, same WebSocket relay, same credentials flow — just without the float chrome. It has a target selector (visible when multiple targets are available), the same status dot and controls (Credentials, View only, C-A-D), and a back link to the chat page.

Use it when you want a bigger workspace or want to keep the float free for other floats. The new tab is independent — closing it does not affect the float.

---

## Quick recipe: ask and watch

1. In your SCORPIOX BOT chat, open the ··· More menu and switch on **VNC**. The window appears (empty at first).
2. Tell the agent what to show: "spin up a desktop for me and expose it over VNC", or "let me drive that VM."
3. The agent starts the VNC server, offers it, and tells you the title. The window fills in and connects — the status dot goes green.
4. Drive it: click around, type, **C-A-D** when you are stuck, **View only** when you just want to watch, **↗** to move it into its own tab.
5. When you are done, tell the agent to stop it — the offer disappears from your list.

---

*Related: the SCORPIOX BOT chat page and the chat More menu, which also carries the companion Preview, notepad, Drop, and RDP floats. See the SCORPIOX BOT page for the overall remote-control model and node selection, and the SCORPIOX Tmux page for how sessions are created and addressed.*
