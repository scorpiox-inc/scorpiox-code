# Deterministic File Editing & Developer Autonomy in SCORPIOX CODE

Every agent harness has to edit files. The only question is *how*, and that question decides how much control you keep.

Most tools make one of two bad bets. They either hand the model a **fuzzy search-and-replace block** and hope the surrounding context matches byte-for-byte, or they force a **full-file rewrite** and let the model retype the entire file — drifting on whitespace, comments, and formatting it never meant to touch. Both fail the same way: the edit you *intended* is not the edit that *lands*, and you find out only when the diff is wrong.

SCORPIOX CODE takes a different position: **give the agent exact, deterministic line ranges — and never lock you out of doing it your way.** This page explains how the editing model works, what "deterministic" means in practice, and why you are never trapped in a proprietary sandbox.

Source of truth: the built-in `preferred-file-tools` skill and the `scorpiox-readfile`, `scorpiox-editfile`, and `scorpiox-grep` utilities at commit `6c70ad6`.

---

## The problem with "clever" edit formats

The dominant approaches in other harnesses each have a known failure mode:

| Approach | How it works | Where it breaks |
|----------|--------------|-----------------|
| **Fuzzy search-and-replace block** | Model emits "old text" + "new text"; the tool finds the old text and swaps it in | Fails the moment the old text does not match byte-for-byte — a renamed variable, a reformatted line, a trailing space, or the same snippet appearing twice. The match breaks and the edit is rejected. |
| **Unified diff / patch** | Model emits a `@@` hunk with line numbers and context | Breaks the instant the line numbers or context are off by one. The hunk is rejected or misapplied, forcing a re-read and retry. |
| **Full-file overwrite** | Model rewrites the entire file from scratch | Silently drops formatting, blank lines, and unrelated code it did not mean to change. Unscalable past a few hundred lines. |
| **Proprietary "apply" model** | Vendor model regenerates the file from a prompt | Non-deterministic, opaque, and you cannot audit exactly what changed line-by-line. |

The common thread: **the edit boundary is implicit.** The model has to *infer* where its change starts and ends from prose context, and every inference is a place it can be wrong. Line 55 is not a guess.

SCORPIOX CODE makes the boundary **explicit and numeric.** An agent does not say "around the function I was looking at." It says: *replace lines 55 through 57.* That is a coordinate, not a guess.

---

## Full control: deterministic line ranges

The core primitive is a numbered line range. The workflow is three steps, and every step is inspectable before it commits.

### 1. Read with line numbers

`scorpiox-readfile` prints every line prefixed with its exact line number, so the agent (and you) see the real coordinates in the file rather than an abstract context window.

```bash
# Entire file
scorpiox-readfile src/main.c

# Or just the range you care about
scorpiox-readfile src/main.c 50 80
```

Output:

```
50: void handle_request(req) {
51:     if (ptr == NULL) {
52:         return -1;
53:     }
```

### 2. Write the edit as a small, reviewable replacements file

Each block names a `LINE <start>-<end>` range followed by the new content, closed by `END`. This is a plain-text file you can open, read, diff, and review before anything is applied.

```
LINE 55-57
if (ptr == NULL) {
    return -1;
}
END
```

### 3. Apply and verify in one shot

`scorpiox-editfile` applies the change and prints back the changed region with context, in the same call.

```bash
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2
```

`--verify N` prints the changed lines back to you with `N` extra boundary lines of context, using the same numbered format as `scorpiox-readfile`. Apply and inspect become a single step — no round-trip.

### The three operations

All three map cleanly onto the range arithmetic, and all three are unambiguous:

| Operation | Syntax | What it does |
|-----------|--------|--------------|
| **Replace** a range | `LINE 5-7` + new content + `END` | Overwrites lines 5 through 7 with the new content |
| **Delete** a range | `LINE 5-7` + `END` (empty block) | Removes lines 5 through 7 |
| **Insert** before line N | `LINE 3-2` + new content + `END` | Start is greater than end, so it inserts before line 3 |

### Why "deterministic" is doing real work here

- **Line numbers always refer to the original file.** Edits are applied bottom-up — highest line number first — so an earlier range is never shifted by a later insertion. There is no "the line number has now moved, so the next block is wrong" class of bug.
- **Multiple blocks, one file.** One replacements file can carry several `LINE ... END` blocks, and all of them apply in a single call with every coordinate still valid.
- **Fails loudly, not quietly.** A range that starts past the end of the file is an explicit error, not a silent no-op. There is no "did the context match?" probability to reason about.

---

## Encoding and line-ending preservation

The tools are not UTF-8-only. They detect the file encoding on read and write back in the same encoding, so a mixed or legacy codebase does not get silently mangled:

- **UTF-8** (with or without BOM)
- **UTF-16 LE**
- **UTF-16 BE**

Line endings are detected by dominance (LF vs CRLF) and preserved. A Windows checkout with CRLF is not silently converted to LF, and a UTF-16 file does not come back as UTF-8. `scorpiox-readfile` and `scorpiox-grep` handle the same encodings, so the read-edit-verify loop works end-to-end regardless of what the project uses.

This matters in real codebases where mixed line endings or encodings break builds, diffs, and review tools.

---

## The full toolset

SCORPIOX CODE ships four file tools as its preferred defaults. They are called "preferred" for a specific reason: **they are high-precision options, not mandatory constraints.**

| Tool | Purpose | Replaces (but does not forbid) |
|------|---------|-------------------------------|
| `scorpiox-readfile` | Read a file with line numbers, full or ranged | `cat`, `head`, `tail` |
| `scorpiox-grep` | Search files: recursive, regex, case-insensitive, glob filters, context lines | `grep`, `find`, `rg` |
| `scorpiox-editfile` | Line-range edits with encoding preservation and `--verify` | `patch`, `sed`, `awk`, `ed` |
| `CreateFile` | Create a new file with specified content | `echo >`, `cat >`, manual file creation |

A note on `scorpiox-grep`: it is a zero-dependency, cross-platform search that automatically skips `.git`, `.svn`, `.hg`, `node_modules`, `__pycache__`, and binary files, and supports fixed-string, regex, case-insensitive, whole-word, inverted, multi-pattern, context (`-A`/`-B`/`-C`), and per-file match limiting. It reads from stdin when no path is given. It is the search step that feeds accurate line numbers into the edit.

The built-in skill instructs the agent to prefer these tools. But **prefer is not require.**

---

## Developer autonomy: no lock-in, no sandbox

This is the part that separates SCORPIOX CODE from the other harnesses.

The moment a harness says "you *must* use my proprietary edit tool and nothing else," it has quietly reduced your leverage. You can no longer use the editor, the shell command, or the native utility you already trust — even when your tool would do the job better. Every task routes through a format defined by one vendor's abstraction, and that vendor can tighten it later.

SCORPIOX CODE does not do that:

- **Standard tools always work.** `sed`, `awk`, `patch`, `git apply`, your editor of choice — all remain available. Nothing is disabled.
- **Direct shell commands always work.** If the deterministic line-range tool is the right fit, use it. If a one-liner is, use the one-liner. You and the agent decide per task.
- **Native utilities always work.** The preferred tools are cross-platform, dependency-free, and inspectable — and they are the option a user or agent reaches for, not a cage around the rest.

The high-precision tools are *a good option for a specific job*. They are not a lock-in. You keep full control, the agent keeps full control, and neither of you is forced into a sandbox the vendor can ratchet up later.

---

## How this compares to the other harnesses

| Harness | Editing model | What the developer is locked into |
|---------|---------------|-----------------------------------|
| **SCORPIOX CODE** | Deterministic numeric line ranges + `--verify`; standard tools always available | Nothing. Preferred tools are optional, not enforced |
| **Claude Code** | Fuzzy search-and-replace `Edit` tool (old/new text) + full-file `Write` | A proprietary edit format whose "old text" must match byte-for-byte; raw shell is possible, but the model is steered toward the built-in tools |
| **OpenCode** | Built-in edit/patch primitives with fuzzy matching | A vendor-defined edit abstraction over your files |
| **Aider** | Unified diff, whole-file, or search-replace formats (model-configurable) | A diff/whole-file format that must regenerate correctly, or the change is lost |
| **Cursor** | Proprietary "apply" model that regenerates file content | A black-box apply you cannot audit line-by-line |
| **Pi / Hermes** | Harness-specific edit primitives | Each vendor's own editing sandbox |

The pattern across the others: the edit boundary is implicit, the format is proprietary, and the tooling is designed to be the *only* path. SCORPIOX CODE inverts that — explicit numeric boundaries, a plain-text replacements file you can read, a verification step, and no lock-in on any of it.

---

## The workflow, end to end

```bash
# 1. Locate the target
scorpiox-grep -rn "TODO" src/

# 2. Read the exact lines you need, with their real line numbers
scorpiox-readfile src/main.c 50 80

# 3. Write the edit to a uniquely-named replacements file (via CreateFile)
#    /tmp/sx_edit_fix_null_check.txt:
#        LINE 55-57
#        if (ptr == NULL) {
#            return -1;
#        }
#        END

# 4. Apply and verify in one step
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2

# 5. If verify looks wrong: read again, write a new file with a new name, re-apply.
```

A few practical notes:

- **Always read before you edit.** The line numbers in your replacements file must match what `scorpiox-readfile` just showed you. Reading first is what makes the range deterministic.
- **Unique filenames per edit.** Give each replacements file its own descriptive name (e.g. `/tmp/sx_edit_fix_header.txt`, `/tmp/sx_edit_add_logging.txt`). There is no reason to reuse or delete previous ones, and the on-disk file doubles as an audit trail.
- **The replacements file is real, not ephemeral.** You can open it, inspect it, diff it, or commit it.

---

## Why this matters for long-horizon tasks

In a long session, the agent edits files dozens of times. With fuzzy matching, each edit carries a non-trivial failure probability — multiply that across thirty edits and the chance that at least one fails is high, and every failure forces a re-read and retry that burns tokens and adds latency.

With deterministic line ranges, an edit either applies or it does not. The model reads lines 50-65, notes that line 55 is the target, writes `LINE 55-55`, and the edit is unambiguous. `--verify` confirms it in the same shell call — no round-trip, no re-read, no retry loop. Across a long task, that compounds into fewer failed edits, fewer wasted tokens, and a faster finish.

---

## TL;DR

- **Deterministic line ranges, not fuzzy matching.** Edits name explicit `LINE start-end` coordinates, applied bottom-up, so ranges never drift and line numbers always mean what you think they mean.
- **Inspectable and verifiable.** The edit is a plain-text replacements file you can read, and `--verify` prints the changed lines back to you in the same call before you trust the result.
- **Encoding and line endings preserved** — UTF-8, UTF-8 BOM, UTF-16 LE/BE, CRLF and LF, automatically.
- **No lock-in.** The preferred tools are high-precision options. Standard tools, direct shell commands, and native utilities always work — you and the agent choose per task, and the harness never forces a proprietary format.
