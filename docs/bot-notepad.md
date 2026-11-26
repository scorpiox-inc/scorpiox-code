# Private Notepad Synced Across Your Devices

You are on your phone, a thought or a snippet is in your head, and you want it waiting on your computer. Or the reverse: you scribbled a note at your desk and want it in your phone tonight. **Notepad** in SCORPIOX BOT is a private document tied to your account — one page of plain text, shared across every device you are signed in to. Write on your phone, keep typing on your desktop: same document, no transfer step, no second account, no waiting for a node to be awake.

It is deliberately small: one textarea, a status line, and a live link to your other devices. Saves are atomic — your text is stored only once a write completes, never half-written — and the document lives on the SCORPIOX BOT host, so it keeps working even when every one of your agent nodes is offline.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** one private notepad per signed-in account; edit it on any device and the change reaches all of them, and only you can see it.

---

## What it is (and what it is not)

- **One notepad per account.** The document belongs to your SCORPIO+ account, not to a machine or a browser. Sign in as yourself on a phone and a laptop and you are looking at the same text.
- **No other account can touch it.** The notepad is stored per account, and no route in SCORPIOX BOT serves another account's document or lets anyone list the accounts that have one. There is no "everyone" notepad and no way for a second account to read or even enumerate yours.
- **Agents stay out.** The notepad is yours. Agents and sessions cannot read it through the normal session view — it is not conversation state and not a workspace file.
- **It is always available.** The store lives on the SCORPIOX BOT web host, not on a node, so the notepad keeps working even when all your nodes are down.
- **It is opt-out-able.** You can hide the chat menu item and stop the float from auto-opening, and the operator who hosts your SCORPIOX BOT can switch the whole feature off for everyone.

---

## Opening your notepad

Open your SCORPIOX BOT web, sign in, and go to **`/notepad`**.

The page is exactly what it says it is: a single textarea and a status line, nothing else. The header reads **Notepad** with the subline "Private to your account", and the empty editor reads "Your private notepad - shared across every device signed in as you."

- **Just type.** The editor is a plain monospaced textarea. There is nothing to configure: it starts with your existing text (or empty, if you have never saved), and whatever you type becomes the document.
- **Autosave, about half a second after you stop.** While you type, the status line shows **typing**. Half a second after your last keystroke, the document is written automatically — you never press save. While the write is in flight you see **saving**, and the moment it lands you see **saved**. If the host is unreachable you see **offline**; keep typing and the next autosave retries.
- **The status line.** At the bottom you always see three things: a coloured status dot with its state (loading, typing, saving, saved, offline, save failed), your current character count, and the document's version number (**v0** before your first save, incrementing by one with every save). The version is how SCORPIOX BOT tells your devices whether the document changed.
- **Tab inserts two spaces.** The editor is a code-style textarea, so Tab indents instead of moving focus.
- **It refreshes when you come back.** If you switch away from the page (another tab, another app, backgrounding the phone) and return, and you are not in the middle of your own edit, the page quietly re-reads the current document so you are never looking at stale text.

---

## How changes reach your other devices

Your devices stay in sync with a small, honest loop. Each open device listens for a change signal from the host. The moment the document's version goes up — say you saved on your phone — the host tells your other open devices "the notepad changed", and they re-read the new text and update their editors on their own, with no manual refresh.

Two rules keep this sane:

- **Your own edit always wins locally.** While a device is typing, saving, or mid-autosave, it does not yank your text out from under your cursor. It applies the incoming change the moment your local edit settles.
- **Devices only fetch what changed.** A device that already has the current version is answered with a short "unchanged" reply and does not re-download your text. A large notepad costs nothing to keep open on several devices at once.

---

## The chat float

Besides the full page, the notepad is also available as a small floating window inside your SCORPIOX BOT chat, so you can keep notes open next to a conversation.

- **Open it from the More menu.** In chat, click the **More** button (three dots) and choose **Notepad** — the menu entry reads "Your private notes, synced across devices". The float opens as a small window in the lower-left of the chat page. It is off by default: it opens when you choose it, and stays closed until you open it again.
- **The same document.** The float edits exactly the same per-account document as `/notepad`, with the same autosave, the same status dot and state text, the same character count and version, and the same live sync to your other devices. Write in the float on your phone and the full page on your desktop updates.
- **Drag it anywhere.** Grab the titlebar and move the window wherever you like; you can also resize it. Its position and size are remembered on that device, so it opens back where you left it next time.
- **Reset position and size.** A **Reset** button in the float's footer snaps it back to its default place and size (lower-left, default dimensions).
- **Open the full page.** The ↗ button in the float's titlebar opens `/notepad` in a new tab, for when you want the roomier full-page editor.
- **Close it.** The × button closes the float. Your document is unaffected — closing the window never changes the text.

You can also make the float open automatically whenever your chat page loads, as a per-device preference (see the opt-outs below).

---

## Limits

| What | Limit |
|------|-------|
| Notepad size | 256 KB of text |
| Documents per account | 1 |

If your text grows past the limit, the save is rejected and you see a **save failed** state in the status line. Shorten the document (or copy the overflow elsewhere) and keep typing; the next autosave succeeds.

---

## Privacy

- **Only your account can see it.** The document is stored per account, and every read and write in SCORPIOX BOT is scoped to the account you are signed in as. A second account — even one on the same host — cannot read your notepad or enumerate the list of accounts that have one.
- **No agent can read it.** Agents and sessions work in your sessions, not in your personal notepad. It does not appear in any session view and is not part of any agent's context unless you deliberately type it in.

---

## Storage and backups

Your notepad is stored on the **SCORPIOX BOT host** (default location `/var/scorpiox-bot/notepads`, overridable by the operator), in a per-account file that is written atomically — a reader never sees a half-saved document. That has one honest consequence: **backing up your notepad is the operator's responsibility**. If your host's disk dies and there is no backup, the document is gone. For anything that matters, keep a copy somewhere you control too.

---

## Opting out, and when the feature is switched off

- **You, in the chat:**
  - **Hide the menu item.** Under **Settings → Chat More Menu**, switch off the **Notepad** entry ("Your private notepad, synced across devices"). The item disappears from the More menu, so you can no longer open the float from there.
  - **Auto-open the float (your call).** A per-device setting, **`bot_show_notepad`**, decides whether the chat float opens automatically when your chat page loads. Off by default; turn it on and the float is there waiting whenever you open chat on that device.
  - **Close or hide the open float.** The × button closes it, and **Reset** in its footer returns it to its default position and size.
- **Your operator, for everyone:** the operator can disable Notepad for the whole SCORPIOX BOT instance. When it is off, `/notepad` and its endpoints answer "not found", the notepad is unavailable for everyone, and no document is served. If you get a "not found" on `/notepad` and you are sure the feature should be on, that is the first thing to ask your operator about.

---

## Quick recipe: phone to desktop

1. On your phone, open your SCORPIOX BOT chat and choose **Notepad** from the More menu (or go straight to `/notepad`), then type your note. Watch the status dot settle on **saved**.
2. On your desktop, open chat (or `/notepad`) as the same account. Your text is already there — the editor updates live, no refresh needed. Keep typing; your phone follows along the same way.

---

*Related: the SCORPIOX BOT chat page, and the companion private file shelf [Drop](bot-drop.md) in the same chat More menu.*
