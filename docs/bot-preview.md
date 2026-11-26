# Live Artifact Preview in SCORPIOX BOT Chat

You are watching an agent work, and it just finished a PDF, a chart, an HTML page, or a report you actually want to read — without digging through the transcript or the file system. **Preview** in SCORPIOX BOT is a small floating window on the chat page that shows those artifacts to you while the agent keeps working: PDFs, HTML, images, and plain text, all rendered in place. Ask the agent to show a file, and it appears in the window. Rebuild the file, and the window updates.

It is deliberately small: a title bar you can drag and resize, two rows of tabs, a viewer, and a footer of three controls. It renders what the agent has already published — nothing shows up until an artifact is published, and when an artifact is removed the offer stops.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** an agent publishes files into the session's preview store, and SCORPIOX BOT renders them live in a floating window on the chat page — PDF, HTML, image, or text — switching between them with tabs.

---

## What it is (and what it is not)

- **A live view of the session, not a copy.** The window reads the current bytes of whatever the agent pointed it at. When the agent rebuilds a file, the window reflects the new bytes — there is no "download it again" step.
- **Four kinds, rendered safely.** PDFs, HTML, images, and plain text (including diffs and patches). HTML is rendered in a sandboxed frame with scripts disabled, so the content cannot reach back into the chat page.
- **A group, organised into tabs.** One preview is a *group* of artifacts — for example a thesis PDF, its `.tex` source, and a chart. The group is a tab; each artifact inside it is a slot tab. You switch between them the way you switch between browser tabs.
- **Two surfaces, one source.** The floating window on the chat page is the quick view. For the code itself — browsing the tree, editing, and the live **Changes** view of uncommitted work — the chat **Files** item opens the code workspace, whose full page opens in its own tab when you want it. Both read the same session state.
- **Nothing until published.** The window opens empty ("Nothing to preview yet — ask the agent to preview a file") until the agent publishes something. Remove an artifact, and its tab and offer disappear.

---

## Turning it on

Preview is off by default. Open your SCORPIOX BOT chat, then:

- **In the chat, from the ··· More menu:** choose **Preview** ("Live artifact preview float"). The window opens. Your choice is remembered on that device: close the tab and come back and the window is in the state you left it — if you had left it open, it opens again automatically when the chat page loads.
- **To hide the menu item entirely:** under **Settings → Chat More Menu → Preview** ("Live artifact preview float (pdf/html/text)"). Switching it off hides the **Preview** entry from the chat ··· More menu.

You can also turn it off and on again at any time from the ··· More menu, or simply close the window with the **×** in its title bar.

---

## The floating window

When the window is open, it is a small panel you can move around the chat page.

- **Move it.** Drag the title bar to position the panel anywhere on the screen. Your position and size are remembered on that device and restored the next time you open the window.
- **Resize it.** Drag the grip in the lower-right corner to change the size (minimum 280 x 200 px).
- **Maximise it.** The **□** in the title bar toggles the panel to (almost) full-window size; press it again to go back.
- **Reset.** The **Reset** button in the footer restores the default position and size and clears the remembered geometry for this device.
- **Open the raw file in its own tab.** The **↗** in the footer opens the current artifact in a fresh browser tab, so you can do something with it directly (print the PDF, save the image, open the page).
- **Refresh.** The **Refresh** button reloads the current artifact. If the agent registered a build command for it, Refresh re-runs that build on the node with your authority and then shows the new result — the way to say "rebuild and show me the latest."
- **The footer also reports state.** Beside the controls, a small line shows the artifact's type, its size in KB, and the time it was last updated.

The window is scoped to the session you are chatting with, and it follows the node that session runs on, so it keeps working when you switch nodes in the mesh.

---

## The tabs: groups and slots

A preview is a **group** of artifacts. The group name is the first tab row (for example `main`, `thesis`, `dashboard`), and each group tab shows how many artifacts it holds. Inside the open group, the second tab row lists the **slots** — the individual artifacts, each labelled by its title (or its slot number, if it has none). Click a group tab to switch topics; click a slot tab to switch artifacts.

- **Choose a slot.** Click any slot tab to render it. The title bar shows which artifact you are viewing.
- **Mix kinds freely.** A single group can hold a PDF, its HTML version, its source as text, and a chart image, all as sibling tabs.
- **Live updates.** The window listens for changes to the session's preview store. When the agent publishes or updates an artifact, the tabs and the viewer update on their own — you do not poll, and a change shows up as soon as it lands.

---

## How each kind renders

- **PDF.** Rendered in the window by a PDF engine that ships with SCORPIOX BOT (no internet access needed). Pages are fitted to the window width, sharp on high-DPI screens, with a `p N / total` counter in the footer. Use **↗** to open the raw PDF in your browser's own viewer.
- **HTML.** Rendered in a **sandboxed** frame with scripts disabled — static HTML and CSS only. The content cannot read from, write to, or post to the chat page around it.
- **Image.** Shown fitted inside the window at its natural proportions.
- **Text.** Shown as monospaced, selectable text.
- **Diffs and patches.** Recognised automatically and colour-coded: added lines green, removed lines red, hunk and file headers highlighted. Publish a working-tree diff as a `.patch` file and it renders this way in the window; for the live, always-current list of uncommitted changes, use the **Changes** view described below instead.

---

## The full-page viewer: code workspace and Changes

The preview float is for reading artifacts. For the session's code itself, the chat ··· More menu has **Files** ("Browse, edit & diff"): it opens the code workspace — a file tree, a live **Changes** list of uncommitted work, project search, and a full code editor with a per-file Code / Diff toggle. The workspace auto-refreshes its git status, so the Changes list tracks the agent's work as it happens, and **↗** opens the full workspace page in its own tab for maximum room. This is the surface to use when you want to inspect or edit what the agent changed, rather than read a finished document.

---

## How an agent publishes an artifact

You do not publish anything yourself — the agent does, and you just ask. The mechanics are defined by the agent's **`preferred-preview`** skill (a built-in skill in SCORPIOX CODE), and you do not need to know them to use the window. In one sentence: the agent maintains a small index inside the current session's preview store, points each entry at a real file (so a rebuild updates it), and the web layer picks up the change and shows it in your window.

A few things that follow from that:

- **Point, don't copy.** The agent points the preview at the real file. Rebuild that file and the preview follows. It only keeps a local copy for one-off content that has no real file.
- **One build command per artifact (optional).** For a file the agent can rebuild (a LaTeX PDF, a generated chart), it can register a build command that your **Refresh** button re-runs with your authority on the node.
- **Tell you it is ready.** A well-behaved agent tells you what is now in Preview (which group and title) and that you can open it from chat **··· More menu → Preview**.

---

## Safety

- **HTML is sandboxed.** Scripts in a published HTML artifact are disabled, so content the agent shows you cannot act on your chat session.
- **PDFs render offline.** The PDF engine and its fonts are bundled with SCORPIOX BOT, so no CDN or internet round-trip is needed to read a document.
- **Auth is the session's.** The window reads through the same node credentials and session identity as the rest of the chat, so an artifact is visible to you because you are signed in to this session — not because it is public.

---

## Quick recipe: ask and read

1. In your SCORPIOX BOT chat, open the ··· More menu and switch on **Preview**. The window appears (empty at first).
2. Tell the agent what to show: "render the report to PDF and preview it," or "show me that chart."
3. The agent publishes the artifact and tells you it is in Preview. The window fills in and shows it.
4. Switch artifacts with the tabs; **Refresh** to rebuild; **↗** to open the raw file in its own tab; **Reset** to put the window back where it belongs.

---

*Related: the SCORPIOX BOT chat page and its node selection, the sibling VNC float in the same chat More menu, and the companion private file shelf [Drop](bot-drop.md).*
