# Deterministic File Editing & Developer Autonomy in SCORPIOX CODE

Every agent harness has to edit your code. That sounds trivial until you watch it fail: a "search-and-replace" that can't find the exact string, a patch that misaligns one line off and silently corrupts a file, or a diff the model has to get byte-perfect or the whole edit is rejected. The editing layer is where most agent tools quietly lose the plot — and where they quietly take your autonomy with it.

SCORPIOX CODE is built on a different principle: **editing should be deterministic, inspectable, and reversible — and you should never be forced into a single way of doing it.** This page explains the four preferred file tools, why the line-based design removes the whole class of fuzzy-diff failures, and why "preferred" means exactly that: preferred, never locked-in.

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** edit by explicit line range, write the replacement to a plain file you can read, apply it deterministically, and get the changed lines back in the same command — with the full freedom to drop down to any standard tool whenever you want.

---

## The problem: fuzzy editing is a guess

Most harnesses let the agent describe an edit and then *match* it against the file. There are three common shapes, and all three are a guessing game the model plays against your file:

| Mechanism | How it works | How it fails |
|-----------|--------------|--------------|
| **Fuzzy / exact search-and-replace** | The model emits an "old string" it expects to find verbatim, plus a "new string". The tool locates the old string and swaps it in. | The old string must match **exactly** — one stray whitespace, one tab-vs-spaces, one line the model mis-remembered, and the edit is rejected. The model has to reconstruct the file from memory instead of pointing at it. |
| **Unified diffs** | The model emits a `---`/`+++` diff with context lines. The tool applies it positionally. | Off-by-one line drift, missing context, or a hunk that no longer matches after an earlier hunk shifts things. A bad diff either fails or, worse, lands in the wrong place. |
| **Whole-file overwrite** | The model re-emits the entire file and the tool replaces it. | Expensive, and any line the model forgets or rewrites is now a silent change you have to diff yourself to notice. The model is effectively re-typing your file. |

The common thread: **the model is re-generating part of your file from memory and hoping it matches.** Precision drops as files get longer, and every failure costs a turn.

SCORPIOX CODE removes the guess. You don't describe a string to find — you **name the line range**. The edit is a fact about the file's structure, not a reconstruction of its contents.

---

## The four preferred file tools

SCORPIOX CODE ships a small, sharp set of file tools. They are the *preferred* path for reading, searching, and editing — and each one is a plain, inspectable command with no hidden state.

| Tool | Job | What makes it deterministic |
|------|-----|-----------------------------|
| **`scorpiox-readfile`** | Read a file or a line range, numbered. | Every line comes back as `N: content`, so the line numbers you edit against are the line numbers the file actually has. |
| **`scorpiox-grep`** | Search files. | Zero-dependency, cross-platform, auto-skips `.git`, `node_modules`, `__pycache__`, and binary files; handles every common encoding. |
| **`scorpiox-editfile`** | Apply line-based edits. | Edits by explicit `LINE start-end` range, applied bottom-up so numbers always refer to the original file. |
| **`CreateFile`** | Write a file from scratch. | Used to author each replacement file as a real, named file on disk — something you and the agent can both open and read. |

The pattern is the same every time: **read the range you care about, write the replacement to a file, apply it, verify it.** Nothing is matched, nothing is fuzzy, nothing is hidden in the model's head.

### Reading: numbers you can point at

```bash
# Whole file, numbered
scorpiox-readfile src/main.c

# Just the region you're about to edit
scorpiox-readfile src/main.c 50 80
```

Output:

```
50: void handle_request(req) {
51:     ptr = lookup(req->id);
52:     if (!ptr) {
53:         return -1;
54:     }
```

Because the numbers are authoritative, the next step is not "hope the string matches" — it is "replace lines 52–54."

### Searching: the boring, reliable kind

```bash
# Recursive, with line numbers
scorpiox-grep -rn "handle_request" src/

# Case-insensitive, only C files
scorpiox-grep -ri --include "*.c" "TODO" .

# Regex with 3 context lines
scorpiox-grep -rnE -C 3 "TODO|FIXME" src/
```

It does the job you expect from `grep` — including regex, context, and file filters — without you having to remember which variant your platform shipped.

### Editing: by line range, not by guess

An edit is a small **replacements file** with one or more blocks. Each block opens with `LINE <start>-<end>` and closes with `END`; the new content sits in between. Three operations cover everything:

| Operation | How | Meaning |
|-----------|-----|---------|
| **Replace** lines | `LINE 5-7` + new content + `END` | Swap lines 5–7 for the new content. |
| **Delete** lines | `LINE 5-7` + `END` (empty block) | Remove lines 5–7. |
| **Insert** before line N | `LINE N-(N-1)` + new content + `END` | `LINE 3-2` inserts before line 3. |

A full edit to `src/main.c`, replacing the null-check block with a proper one:

```
LINE 52-54
    if (ptr == NULL) {
        log_error("lookup failed for id=%d", req->id);
        return -1;
    }
END
```

Two properties make this deterministic rather than clever:

1. **Bottom-up application.** Blocks are applied highest line number first, so every line number always refers to the **original** file. Inserting ten lines at the top does not shift the line you want to change next. No re-computation, no drift.
2. **Multiple blocks, one file.** You can batch independent edits in a single replacements file and apply them all in one shot.

### The workflow, end to end

1. **Read** the target region to get real line numbers.
   `scorpiox-readfile src/main.c 50 60`
2. **Create the replacements file** with the `CreateFile` tool, under a unique, descriptive name each time.
   Path: `/tmp/sx_edit_fix_null_check.txt`
3. **Apply and verify in one command.**
   `scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2`
4. **If the verify output is not what you expected**, read again, write a new replacements file (new name), and apply again.

Step 3 is the one that removes the failure mode entirely.

### `--verify`: the edit prints itself back

Most editing tools tell you "applied" and hope. SCORPIOX CODE's `--verify` **re-reads the file it just wrote and prints back the changed lines** in the same numbered format as `scorpiox-readfile`, with optional context.

```bash
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2
```

```
--- verify (lines 50-56) ---
50: void handle_request(req) {
51:     ptr = lookup(req->id);
52:     if (ptr == NULL) {
53:         log_error("lookup failed for id=%d", req->id);
54:         return -1;
55:     }
56: }
```

`--verify` prints the changed region; `--verify N` adds `N` boundary lines of context on either side. You are not trusting the tool's claim — you are **reading the actual result on disk**, in the same format you used to plan the edit. The fuzzy-diff failure mode ("did that land where I meant?") is replaced by a line you can look at.

### Encoding and line endings: preserved, not rewritten

A file is more than its text. `scorpiox-editfile` detects the original encoding and line endings and writes the result back in the **same ones** — no silent BOM added or stripped, no Unix-to-Windows line-ending conversion that breaks a `.gitattributes` policy or a build.

| Dimension | What is preserved |
|-----------|-------------------|
| **Encoding** | UTF-8, UTF-8 with BOM, UTF-16 LE, UTF-16 BE |
| **Line endings** | LF (Unix) and CRLF (Windows), per the file's dominant style |

Edit a UTF-16 file on Windows with CRLF line endings and it stays UTF-16 with CRLF. The tool changes the lines you told it to change and nothing else.

---

## Developer autonomy: preferred, never locked-in

This is the part that matters most, and it is easy to miss in a feature list: **nothing in SCORPIOX CODE forces you to use the preferred tools.**

`scorpiox-readfile`, `scorpiox-grep`, and `scorpiox-editfile` are high-precision **options**, recommended because they are the least error-prone path. But the agent — and you — are never walled into them:

- **Standard shell tools always work.** `cat`, `head`, `tail`, `grep`, `sed`, `awk`, `patch`, `perl`, whatever your platform ships — they keep working, with no special permission and no wrapper.
- **Native utilities always work.** Your editor, your build system, your version control — the agent can drive the same commands you would.
- **You can mix freely.** Read with `scorpiox-readfile`, apply with `sed`, verify with `git diff`. The tools do not compete; they are layers you reach for when you want that precision.

That distinction is the whole point. A tool that *recommends* a method and a tool that *mandates* a method look identical in a demo and behave oppositely in real work. When the preferred tool is the right one, reach for it. When you have a one-liner in `sed` or a patch you already have, use it. SCORPIOX CODE does not take your keyboard away to be "safe."

---

## How other harnesses approach it — and the lock-in they introduce

The alternatives are not bad; they are **narrow**. Each one makes a single editing mechanism the only door in and out of your files, and each one optimizes for a scenario it cannot see: that you might want a different one.

| Harness | Dominant editing mechanism | Where it gets rigid |
|---------|----------------------------|---------------------|
| **Claude Code** | Search-and-replace blocks (`old_string` / `new_string`), plus multi-edit and whole-file variants. | The old string must match exactly; the model reconstructs it from context and the edit is rejected on the first mismatch. The file is a thing you describe, not a thing you address. |
| **OpenCode** | Patch / file-write based edits. | Edits are wrapped in the harness's patch format; the file is reached through that format, not directly. |
| **Aider** | Unified diffs and whole-file / search-replace edit formats. | The model must produce a correct diff; line drift or a hunk that no longer matches fails the edit or falls back to a full rewrite. |
| **Cursor** | Editor-driven apply, search-replace blocks, whole-file overwrite. | The edit lives inside the IDE's apply flow; precision depends on the model matching the file it is editing. |
| **Pi** | Search-and-replace blocks. | Same match-or-fail dynamic: the model's string must find the file's string. |
| **Hermes** | Unified diffs / search-replace. | Diff alignment is positional; a bad hunk lands in the wrong place or is rejected. |

Read the column and a pattern shows up: the model is **regenerating part of the file from memory** and the harness is **the only tool that accepts the result.** That is a sandbox with a polite name. You are not editing your file; you are submitting a guess to a format the harness owns, and the harness decides whether the guess was good enough.

SCORPIOX CODE flips the relationship:

| | **Fuzzy-editing harnesses** | **SCORPIOX CODE** |
|---|---|---|
| **Addressing** | "Find this string." | "Change lines N–M." |
| **Source of truth** | The model's memory of the file. | The file's actual line numbers, read moments before. |
| **Failure mode** | Mismatch → rejected, or misaligned → silent corruption. | Out-of-range → explicit error. In-range → exactly that range. |
| **Verification** | Trust "applied." | `--verify` reads the written file back, numbered, with context. |
| **The edit artifact** | A block the model produced in a turn. | A plain file on disk you can open, diff, and keep. |
| **Your freedom** | Use the harness's one tool. | Use the preferred tool *or* any standard shell / native command. |

The last row is the one the others cannot copy without changing their model: you can opt out of the precision entirely and still be doing exactly what you intended, with the tools you already know.

---

## When to use what

- **Default to the preferred tools** for anything you will read back or verify. `scorpiox-readfile` → `CreateFile` → `scorpiox-editfile --verify` is the path where "did it land where I meant?" has a one-glance answer.
- **Use `--verify N` with context** when the edit is near code you didn't write in that turn — the boundary lines let you confirm you didn't nudge the wrong function.
- **Batch with multiple blocks** when several independent changes touch one file; one apply, one verify.
- **Drop to a standard tool** for the obvious one-liner. `sed`, `awk`, `patch`, `perl`, `git apply` — if you would do it by hand, the agent can too. There is no penalty for choosing the direct route.
- **Write the replacement file with a unique name each time** (`/tmp/sx_edit_fix_header.txt`, `/tmp/sx_edit_add_logging.txt`). Never reuse or delete prior edit files — they are a free audit trail of exactly what changed.

---

## Gotchas

- **Read before you edit.** The line numbers you pass to `LINE start-end` must be the file's real numbers. Read the region first; the `N: content` output is the contract.
- **Insert uses `start > end`.** To insert *before* line 3, write `LINE 3-2`. A normal `start-end` (start ≤ end) replaces or deletes; only `start > end` inserts.
- **Line numbers are 1-based and refer to the original file.** Because blocks apply bottom-up, earlier blocks do not shift later ones. You do not re-number as you go.
- **Out-of-range is a hard error, not a guess.** Pointing at a line past the end of the file fails loudly with the file length. Nothing is applied to the wrong line.
- **The replacement file is plain text.** `LINE` must start the line and `END` must be a line by itself. Indentation inside a block is preserved verbatim — it becomes the file's content.
- **`--verify` reads the file you just wrote, not your intended output.** That is the point — it is a check on reality, not a confirmation of intent. If it is wrong, the file is wrong; read, rewrite, reapply.

---

## The bottom line

Fuzzy editing is a quiet tax. Every mismatched string, every off-by-one diff, every whole-file retyping is a turn the model spends re-guessing a file it already has in front of it — and a place where a bad guess can silently touch the wrong line.

SCORPIOX CODE makes editing a fact instead of a guess: point at the line range, write the replacement to a file you can read, apply it deterministically bottom-up, and read the result back with `--verify`. The encoding and line endings come back exactly as they went in. And none of it is mandatory — the moment a standard shell command or a native tool is the right move, you and the agent are free to use it.

That is the whole offer in one sentence: **complete, inspectable control over your files, with no abstraction standing between you and the disk.**

---

## Related

- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md)
- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md)
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
