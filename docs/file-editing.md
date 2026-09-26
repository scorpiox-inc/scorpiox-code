# Deterministic File Editing and Developer Autonomy in SCORPIOX CODE

Every agent harness has to solve the same basic problem: the model wants to change a file, and the harness has to decide *how* that change gets written to disk. Get that mechanism wrong and you get a class of failure that is maddening to debug — the model is confident, the patch looks right, and the file still does not match what the model intended.

Most harnesses solve it by guessing. They hand the model a fuzzy abstraction — "match this block of text somewhere and swap it for that block" — and hope the model's quote of the original file was byte-for-byte identical to what is actually on disk. SCORPIOX CODE solves it by removing the guesswork entirely: **edits are addressed by line number, applied deterministically, and verified in the same call.**

This page explains the mechanism, the tools behind it, and the autonomy guarantee that comes with it.

---

## The problem with fuzzy editing

The dominant editing abstraction in the agent world is the **fuzzy search-and-replace block**. The model quotes the current lines it wants to change, proposes replacement lines, and the harness searches for the quoted text and swaps it. Aider popularized the `<<<<<<< SEARCH` / `>>>>>>> REPLACE` form; most other harnesses — Claude Code, OpenCode, Cursor, and the open projects like Pi and Hermes — run some variant of it.

It sounds reasonable, and it works about 90% of the time. The other 10% is where the pain lives:

- **The match is textual, not positional.** If the model quotes a line with a trailing space it did not notice, a tab where it assumed a space, or one character off, the search fails. Or worse, it succeeds in the *wrong place* when the quoted block appears twice in the file.
- **Failures are delayed and opaque.** The harness tells you "the block did not match" without saying which byte differed. The model re-quotes, you re-run, the block still does not match. A single typo in a quoted line can burn a dozen turns.
- **Encodings are a second failure mode.** A CRLF file edited with an LF assumption, or a UTF-16 source file round-tripped through a UTF-8-only tool, will silently corrupt content. Fuzzy editors typically do not care, because they are matching and writing text, not *the file*.
- **You cannot audit what happened.** The edit is a black box inside the harness. There is no artifact showing what matched, what was replaced, or what the file looked like afterwards.

None of this is the model's fault. It is the consequence of asking a probabilistic system to reproduce exact bytes from memory and then asking a text-matching engine to find them.

---

## The SCORPIOX CODE answer: edits by line number

SCORPIOX CODE's file editing is built around one idea: **the line number is the address**. If you know which lines you want to change, you do not have to quote them. The tooling is arranged so that getting the line numbers is the easy part, and changing the lines is a deterministic operation that cannot misfire.

Four tools form the workflow, and they ship as built-in capabilities of SCORPIOX CODE:

| Tool | Job |
|------|-----|
| `scorpiox-readfile` | Read a file with **line numbers** — the map you will edit against |
| `scorpiox-grep` | Search files across encodings, skipping noise automatically |
| `CreateFile` | Write the *replacements file* — a plain, inspectable artifact |
| `scorpiox-editfile` | Apply the replacements by line range and print back what changed |

The whole loop is: **read, search, write the replacements, apply, verify.** Every step produces something you can open and inspect.

### Read: line numbers are the interface

```bash
scorpiox-readfile src/main.c            # whole file, numbered
scorpiox-readfile src/main.c 50 80      # just lines 50 through 80
```

Output is numbered line by line:

```
50: int main(void) {
51:     init();
52:     run();
53:     return 0;
54: }
```

That numbering is the contract for the next step. You do not need to remember or re-derive the content — you need the addresses.

### Search: `scorpiox-grep`

```bash
scorpiox-grep -rn "TODO" src/                 # recursive, with line numbers
scorpiox-grep -ri --include "*.c" "malloc" .  # case-insensitive, filtered
scorpiox-grep -rnE -C 3 "TODO|FIXME" src/     # regex, 3 lines of context
scorpiox-grep -rl "pattern" .                 # matching filenames only
```

It is a complete grep: fixed-string and regex modes, context lines, include/exclude globs, whole-word, inverted, count, quiet, and stdin. It automatically skips `.git`, `.svn`, `.hg`, `node_modules`, `__pycache__`, and binary files, and it reads UTF-8, UTF-8 BOM, UTF-16 LE, and UTF-16 BE files transparently. Find the line, note the number, edit that number.

### The replacements file: your edit is a document

The heart of the mechanism is the **replacements file**. Instead of embedding an edit inside a tool call, you write it to a plain text file that you — and the agent — can open, diff, review, and keep.

Format:

```
LINE <start>-<end>
<new content>
END
```

Three operations cover everything:

| Operation | How to express it |
|-----------|-------------------|
| **Replace** lines | `LINE 5-7` followed by the new lines, then `END` |
| **Delete** lines | `LINE 5-7` followed immediately by `END` (empty block) |
| **Insert** before line N | `LINE N-(N-1)` — start greater than end — followed by the new lines, then `END`. For example `LINE 3-2` inserts before line 3 |

Multiple blocks go in one file. Replacements are applied **bottom-up** (highest line numbers first), so every `LINE` address refers to the *original* file. You never have to re-number the blocks you wrote earlier in the same file — that is the detail that trips up most line-based editors, and it is built out here.

### Apply and verify in one step

```bash
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2
```

`--verify` applies the edit **and** reads the file back, printing the changed regions with `N` lines of boundary context in the same numbered format as `scorpiox-readfile`. The edit and its proof arrive together. If the printed result is not what you intended, the failure is visible immediately, with line numbers, in the same turn — not discovered twenty turns later by a test that fails for a reason nobody can trace.

A typical edit looks like this end to end:

1. **Read** the region you will touch: `scorpiox-readfile src/main.c 50 80`
2. **Write** a replacements file (via `CreateFile`, with a unique name per edit):

   ```
   LINE 55-57
   if (ptr == NULL) {
       return -1;
   }
   END
   ```

3. **Apply and verify**: `scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2`
4. **Wrong?** Read again, write a *new* replacements file, reapply. Each attempt is a fresh, inspectable artifact.

---

## What "deterministic" buys you in practice

### No context mismatch, by construction

Fuzzy editors fail when the quoted context does not match. Line-addressed edits have no quoted context to mismatch. `LINE 55-57` replaces lines 55 through 57. There is nothing to search, nothing to fuzzy-match, nothing to approximate. The edit either applies at the address you gave or it reports the address as out of range — and that error tells you exactly what to fix.

### Instant, in-the-loop verification

`--verify` closes the loop inside the tool call. The model writes the edit, sees the result with line numbers, and knows in the same turn whether it is right. That single feature eliminates the most expensive failure mode in agent editing: the silent wrong edit that the agent believes succeeded and that only surfaces downstream.

### Encodings and line endings are preserved, not negotiated

The tools detect the file's encoding — UTF-8, UTF-8 with BOM, UTF-16 LE, or UTF-16 BE — and its dominant line ending (LF or CRLF), and write the result back in the same encoding and line ending. A Windows CRLF file stays CRLF. A UTF-16 source file stays UTF-16. The BOM stays where it was. There is no "encoding: utf-8" flag to remember, and no silent re-encoding to discover in review.

### Every edit is an artifact

Because the edit lives in a replacements file, you get a natural review surface. You can open it before applying. You can keep it. You can hand it to a colleague. The history of what an agent changed is the history of plain text files you can read.

---

## Autonomy: the tools are preferred, never enforced

Here is the part that matters most, so it gets its own section.

**SCORPIOX CODE's file tools are high-precision options, not a cage.** The harness *prefers* `scorpiox-readfile`, `scorpiox-grep`, `scorpiox-editfile`, and `CreateFile` for file work — they are the right default because they are the precise ones. But nothing in the harness is forced, locked, or sandboxed.

- Standard shell access is always available. `sed`, `awk`, `patch`, `grep`, `cat`, `head`, `tail`, `perl` — any of them, any time.
- Direct commands work. If you have a script that edits a file, run it.
- Native utilities work. On Windows, PowerShell, `Get-Content`, `Select-String`, and friends are there and functional.
- The agent is a client of the same shell you are. There is no separate "edit-only" API it must route through and no tool it is forbidden from calling.

The philosophy is that the model and the user share one filesystem and one shell, and the harness's job is to make the *precise* path the easy path — not to wall off the *direct* path. If you trust your own `sed` one-liner, it will work. If the agent finds the line-based route clearer for a tricky refactor, it will use that. You are not locked into either.

This is deliberately different from harnesses that make their proprietary editing abstraction the only door. SCORPIOX CODE gives you the best door and leaves the rest open.

---

## How other harnesses approach it

| Harness | Editing model | What you live with |
|---------|---------------|--------------------|
| **Claude Code** | Fuzzy search-and-replace blocks (and whole-file overwrite for larger changes) inside a managed tool loop | Edits are addressed by quoted context; mismatched quotes fail, and you iterate on them. The loop is the harness's; the escape hatch is the shell. |
| **OpenCode** | Same search-and-replace / diff family, pluggable tool layer | Same textual-matching failure mode. The tool layer is configurable, but the edit contract is still "quote me what is there." |
| **Aider** | Multiple named edit formats, the canonical one being `SEARCH`/`REPLACE` blocks (with `wholefile` and diff-style alternatives) | The format is well documented and battle-tested, but it is still fuzzy matching: exact-context quotes, retry loops on mismatch, and format-specific prompt conventions. |
| **Cursor** | IDE-embedded apply model: the editor proposes changes, you accept/reject, and the agent applies through the IDE's edit machinery | Tightly coupled to the IDE's own edit/apply pipeline; the abstraction is the product, and leaving it means leaving the workflow. |
| **Pi** | Minimal open harness; edits flow through the agent's tool calls and the underlying file tools | Autonomy is high, but so is the burden: there is little built-in structure, so precision is on you to assemble. |
| **Hermes** | Agent tool-call editing over the standard shell/file surface | Same trade-off as Pi — full access, and the editing discipline is whatever you build. |

The pattern: the mainstream either **standardizes on fuzzy matching** (Claude Code, OpenCode, Aider) or **couples editing to a proprietary apply pipeline** (Cursor), while the minimal open harnesses (Pi, Hermes) give you freedom but not structure.

SCORPIOX CODE takes the fourth position: **structure without lock-in.** The line-based, verified, encoding-preserving route is built in and preferred. The raw shell is always one step away. You get the precision of the disciplined path and the freedom of the direct path, and you decide, edit by edit, which one to walk.

---

## Gotchas

- **Read before you edit.** The line numbers in your replacements file refer to what `scorpiox-readfile` showed you. If the file has changed since you read it, your addresses are stale — re-read first.
- **Inserts use the start-greater-than-end convention.** `LINE 3-2` means "insert before line 3." It looks like a typo the first time you see it; it is the documented form.
- **Line numbers refer to the original file, always.** Bottom-up application means the blocks in one replacements file do not shift each other's addresses. You can write them in any order.
- **Give each replacements file a unique name and do not reuse them.** Each edit is a distinct artifact; reusing a name blurs the history and invites confusion about which content went with which target.
- **`--verify` is the habit, not the option.** Apply and verify in one call. The few seconds of boundary context are what turn "it probably worked" into "it worked, here is the proof."
- **Out-of-range is an error, not a guess.** If a `LINE` address exceeds the file length, the edit stops and tells you so. That is the system telling you to re-read, not a silent partial apply.
- **Nothing is forced.** If the line-based route is the wrong tool for a particular job — a generated file, a bulk mechanical transform, a one-off script — the shell is open. Use it. The preferred tools are the default, never the requirement.

---

Docs for SCORPIOX CODE @ `b59223a`.
