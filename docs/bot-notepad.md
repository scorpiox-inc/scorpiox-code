# Private Notepad Synced Across Your Devices

You have a thought worth writing down while you are on your phone, and you want it sitting in front of you at your desk tonight. **Notepad** in SCORPIOX BOT is exactly that: one private document per signed-in account, identical on every device you sign in from. Type on your phone, the text is on your desktop. Type at your desk, it is on your phone. No transfer step, no second account, no waiting for a node to be awake.

It is deliberately small: a text field, a status line, and nothing else. The document autosaves while you type, writes are atomic, and the store lives on the SCORPIOX BOT host, so the notepad keeps working even when every one of your agent nodes is offline.

Docs for SCORPIOX CODE @ `ce3b5cd`.

> **The whole idea in one line:** one private notepad per signed-in account; edit it on any device and it is current on all of them, and only you can read it.

---

## What it is (and what it is not)

- **One document per account.** The notepad belongs to your SCORPIO+ account, not to a machine or a browser. Sign in as yourself on a phone and a laptop and you see the same text.
- **No other account can touch it.** The document is stored per account, and no route in SCORPIOX BOT lists or serves another account's notepad. There is no "everyone" notepad and no way for a second account to guess or browse yours.
- **Agents stay out.** The notepad is your private scratch space. Agents and sessions cannot read it through the normal session view, and there is no normal path for an agent to open another account's notepad without an explicit credential you share.
- **It is always available.** The store lives on the SCORPIOX BOT web host, not on a node, so the notepad keeps working even when all your nodes are down.
- **It is opt-out-able.** You can hide it from the chat menu, and the operator who hosts your SCORPIOX BOT can switch the whole feature off.

---

## Opening your notepad

Open your SCORPIOX BOT web, sign in, and go to **`/notepad`**.

The page is exactly what it says it is: a single text field and a footer, and nothing else. The header reads **Notepad** with the subtitle **"Private to your account"**, and the editor's placeholder reads "Your private notepad - shared across every device signed in as you."

- **Just type.** The editor is a monospace text field with spell-check, autocorrect, autocapitalisation, and autocomplete all switched off, so what you type is what you meant. **Tab** inserts two spaces instead of moving focus.
- **Autosave.** There is no save button. The text autosaves 600 ms after you stop typing, and the footer shows the state at each step: **loading** while your text loads, **typing** while you are editing, **saving** while a save is in flight, and **saved** once it lands. A small dot beside the word is green when saved, orange while saving, and red on an error.
- **Character count and version.** The footer shows your document's character count and its version number (**v0** when empty, then **v1**, **v2**, and so on, one per save). The version is what keeps every device in step: it is the number both sides compare before accepting a change.
- **Errors are honest.** If a save cannot complete you see **save failed**, and if you cannot reach the host at all you see **offline**.
- **Live across devices.** The page listens for changes to your notepad: edit on your phone and the copy on your desktop updates on its own, and vice versa. You also get a refresh automatically whenever you return to the tab.

---

## The chat float

While you are chatting with an agent, you can keep your notepad open as a small floating panel next to the conversation, so you can jot thoughts without leaving the chat.

- **Enable it.** In the chat, open the **··· More menu** and switch on **Notepad** ("Your private notes, synced across devices"). It is off by default. Your choice is remembered: close the tab and come back and the float is in the state you left it. You can also hide the menu item entirely under **Settings → Chat More Menu → Notepad**.
- **Same document, same rules.** The float is the same private notepad — same text, same size limit, same autosave, same live sync. Nothing in it routes through a node, so it keeps working even when every node is offline.
- **Move it.** Drag the title bar to move the panel anywhere on the screen; your position is remembered. **Reset** puts it back to its default position and size.
- **Open the full page.** The small up-arrow in the title bar opens the full `/notepad` page in a new tab; the close button hides the float again.
- **State line.** The footer shows the same status dot and word as the full page — typing, saving, saved, or an error — plus your character count and the current version.

---

## Limits

| What | Limit |
|------|-------|
| Document size | 256 KB of text |

If your text is larger than the limit, the save is rejected and the footer shows **save failed**. Trim the document and save again.

---

## Privacy and storage

Your notepad is stored on the **SCORPIOX BOT host**, in a per-account file (default location `/var/scorpiox-bot/notepads`, overridable by the operator). Consequences you should know about:

- **Only your account can read it.** There is no route that lists accounts or serves another account's notepad, so no second account can enumerate or open yours.
- **Agents stay out.** Agents and sessions cannot read your notepad through the normal session view. If a workflow genuinely needs it, the operator can pass an explicit credential (the node password) to a specific call — that is a deliberate act, not a default.
- **Backups are the operator's job.** If your host's disk dies and there is no backup, the text is gone. For anything that matters, keep a copy somewhere you control too.

---

## When the feature is switched off

- **You, in the chat:** the **Notepad** toggle in the chat More menu (and its menu-item switch under Settings → Chat More Menu) only controls the float. The full `/notepad` page is always available to your account unless the operator has switched it off.
- **Your operator, for everyone:** the operator can disable the notepad for the whole SCORPIOX BOT instance. When it is off, the page and the float's data endpoints answer "not found" and the notepad is unavailable for everyone. If you get a "not found" on `/notepad` and you are sure the feature should be on, that is the first thing to ask your operator about.

---

## Quick recipe: phone to desktop

1. On your phone, open your SCORPIOX BOT chat (or go straight to `/notepad`) and type your note. The footer shows **saving** and then **saved** — that is your document stored.
2. On your desktop, open `/notepad` (or the chat float) as the same account. Your text is already there, and it updates live as you keep typing on the phone.

---

*Related: the SCORPIOX BOT dashboard and chat pages, and the companion private file shelf (`/drop`) in the same chat More menu.*
