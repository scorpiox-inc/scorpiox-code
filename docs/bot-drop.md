# Private File Drop Synced Across Your Devices

You are on your phone, a file is in your inbox, and your computer is at your desk. Or the reverse: you finished a build artifact at your desk and want to read it on your phone tonight. **Drop** in SCORPIOX BOT is a private file shelf tied to your account — one shelf, shared across every device you are signed in to. Drop a file on your phone, open it on your desktop: no transfer step, no second account, no waiting for a node to be awake.

It is deliberately small: a drop zone, a file list, and a download link per file. Uploads are atomic — a file appears in your list only when it is fully stored, never half-written — and the shelf lives on the SCORPIOX BOT host, so it keeps working even when every one of your agent nodes is offline.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** one private file shelf per signed-in account; drop a file on any device and it is on your shelf on all of them, and only you can see it.

---

## What it is (and what it is not)

- **One shelf per account.** The shelf belongs to your SCORPIO+ account, not to a machine or a browser. Sign in as yourself on a phone and a laptop and you see the same files.
- **No other account can touch it.** Files are stored per account, and no route in SCORPIOX BOT lists or serves another account's files. There is no "everyone" shelf and no way for a second account to guess or browse yours.
- **Agents stay out.** Drop is your personal shelf. Agents and sessions cannot read it through the normal session view — it is not conversation state and not a workspace file.
- **It is always available.** The store lives on the SCORPIOX BOT web host, not on a node, so the shelf keeps working even when all your nodes are down.
- **It is opt-out-able.** You can hide it from the chat menu, and the operator who hosts your SCORPIOX BOT can switch the whole feature off for everyone.

---

## Opening your shelf

Open your SCORPIOX BOT web, sign in, and go to **`/drop`**.

The page is exactly what it says it is: a drop zone, your file list, and nothing else. The header reads **Drop** with the subline "Private to your account", and the dashed drop zone reads "Drop files here" with "or tap to pick — private to your account".

- **Drag and drop.** Drag one or several files onto the dashed zone. It lights up while a file is over it.
- **Pick files.** Click or tap the zone to open your device's file picker; it accepts multiple files at once.
- **Upload progress.** Files upload one after another. While one is in flight you see "Uploading `<name>` …" and a progress bar showing the percentage of the current file. If an upload fails you see an error message, and the rest of the queued files stop — re-drop the ones that were left behind.
- **Live across devices.** The page listens for changes to your shelf: upload a file on your phone and the list on your desktop refreshes on its own, and vice versa.

---

## Your file list

Your files are listed newest first. Each file is a row with its name, its size (formatted as B/KB/MB/GB), and its actions:

- **Open the name** — the file's name is a link. It opens the file in a new tab, and SCORPIOX BOT serves it with the right content type for its extension (images, PDF, text, Markdown, JSON, CSV, video, audio, 3D model formats, and so on), so your browser previews or opens it properly instead of downloading a mystery binary. Unknown extensions are served as a generic download.
- **Delete** — two-step, on purpose. The delete button (an × in each row) arms on the first click — it turns red; click it again within five seconds to actually delete the file. Wait it out and it disarms, so a stray tap cannot delete something you meant to keep.
- **Re-upload replaces.** If you upload a file with the same name, it replaces the existing copy.

---

## The chat float

While you are chatting with an agent, you can keep your shelf open as a small floating panel over the conversation, so you can move files back and forth without leaving the chat.

- **Open it from the chat menu.** In the chat page, open the More menu (**···**) and choose **Drop** — "Your private files, synced across devices". The float opens and the choice is remembered in this browser, so it reopens next time you open the chat. You can also switch the menu item itself on or off under **Settings → Chat More Menu → Drop**.
- **Same shelf, smaller surface.** The float uses the exact same shelf as `/drop`: the same files, the same drop zone (drag files in, or tap it to pick), the same rows with name, size, and the same two-step delete.
- **Move and resize.** Drag the title bar to move the panel anywhere; your position and size are remembered in the browser. **Reset** puts it back to its default position and size.
- **Open the full page.** The arrow (↗) in the title bar opens the full `/drop` page in a new tab.
- **State dot.** The footer shows a small status dot and word — uploading while a file is in flight, done when the queue finishes, and an error state when a load or upload fails — plus a live count of the files on your shelf.

---

## Limits

| What | Limit |
|------|-------|
| Per-file upload size | 200 MB by default (your operator can change it; the hard ceiling is 4 GB) |
| Files per account | 2,000 |
| Filename length | 200 characters |

If a file is larger than the limit, the upload is rejected and you see a message telling you the maximum ("`<name>` is too large (max N MB)").

Filenames are checked for safety: only the base name is kept (any directory part is stripped), and characters that are dangerous in paths — slashes, backslashes, colons, and the like — are replaced or dropped, so no filename can escape your shelf or reach into another account's.

---

## Storage and backups

Your files are stored on the **SCORPIOX BOT host**, in a per-account directory (default location `/var/scorpiox-bot/drop`, overridable by the operator). That has one honest consequence: **backing up the shelf is the operator's responsibility**. If your host's disk dies and there is no backup, the files are gone. For anything that matters, keep a copy somewhere you control too.

---

## When the feature is switched off

- **You, in the chat:** the **Drop** toggle in the chat More menu (and its menu-item switch under Settings → Chat More Menu) only controls the chat float, not the shelf itself.
- **Your operator, for everyone:** the operator can disable Drop for the whole SCORPIOX BOT instance. When it is off, `/drop` and its endpoints answer "not found", the shelf is unavailable for everyone, and no data is served from it. If you get a "not found" on `/drop` and you are sure the feature should be on, that is the first thing to ask your operator about.

---

## Quick recipe: phone to desktop

1. On your phone, open your SCORPIOX BOT chat (or go straight to `/drop`) and drop the file onto the shelf — watch the progress bar until it completes.
2. On your desktop, open `/drop` (or the chat float) as the same account. The file is already there — the list updates live, no refresh needed.
3. Click the file's name; it opens in a new tab with the correct type.

Delete it whenever you are done with a second click on its delete button.

---

*Related: the SCORPIOX BOT dashboard and chat pages, and the companion private notepad float in the same chat More menu.*
