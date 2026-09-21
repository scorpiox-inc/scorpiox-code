# Deterministic File Editing & Developer Autonomy in SCORPIOX CODE

Most AI coding tools make one of two bad bets. They either hand the model a **fuzzy search-and-replace block** and hope the surrounding context matches byte-for-byte, or they force a **full-file rewrite** and let the model retype the entire file, drifting on whitespace, comments, and formatting it never meant to touch. Both paths fail the same way: the edit you *intended* is not the edit that *lands*, and you find out only when the diff is wrong.

SCORPIOX CODE takes a different position: **give the agent exact, deterministic line ranges — and never lock you out of doing it your way.** This page explains how the editing model works, what "deterministic" actually means in practice, and why you are never trapped in a proprietary sandbox.

Source of truth: the built-in `preferred-file-tools` skill and the `scorpiox-readfile`, `scorpiox-editfile`, and `scorpiox-grep` utilities at commit `24427d8`.

---

## The problem with "clever" edit formats

The dominant approaches in other harnesses each have a known failure mode:

| Approach | How it works | Where it breaks |
|----------|--------------|-----------------|
| Fuzzy search-and-replace block | Model emits a "context + replacement" snippet; tool finds the context and swaps it | Fails when surrounding context drifts — a renamed variable, a reformatted line, a comment tweak — because the match no longer lines up |
| Unified diff / patch | Model produces a `@@` hunk with line numbers | Breaks the moment the line numbers or context are off by one; the patch is rejected or misapplied |
| Full-file overwrite | Model rewrites the entire file | Silently drops formatting, blank lines, and unrelated code it did not mean to change; unscalable past a few hundred lines |
| Proprietary "apply" model | Vendor model regenerates the file from a prompt | Non-deterministic, opaque, and you cannot audit exactly what changed line-by-line |

The common thread: the *edit boundary is implicit*. The model has to *infer* where its change starts and ends from prose context, and every inference is a place it can be wrong.

SCORPIOX CODE makes the boundary **explicit and numeric**. An agent does not guess "around the function I was looking at." It says: *replace lines 55 through 57.* That is a coordinate, not a guess.

---

## Full control: deterministic line ranges

The core primitive is a numbered line range. The workflow is three steps, and every step is inspectable before it commits:

1. **Read with line numbers.** `scorpiox-readfile` prints every line prefixed with its exact line number, so the agent (and you) see the real coordinates in the file rather than an abstract context window.
   ```bash
   scorpiox-readfile src/main.c 50 80
   # 50: void handle_request(req) {
   # 51:     if (ptr == NULL) {
   # ...
   ```
2. **Write the edit as a small, reviewable replacements file.** Each block names a `LINE <start>-<end>` range and the new content. You can open this file and read exactly what is about to happen.
   ```
   LINE 55-57
   if (ptr == NULL) {
       return -1;
   }
   END
   ```
3. **Apply and verify in one shot.** `scorpiox-editfile` applies the change and prints back the changed region with context.
   ```bash
   scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2
   ```

The three operations map to the range arithmetic, and all three are unambiguous:

| Operation | How | What it does |
|-----------|-----|--------------|
| **Replace** a range | `LINE 5-7` + new content + `END` | Overwrites lines 5 through 7 with the new content |
| **Delete** a range | `LINE 5-7` + `END` (empty block) | Removes lines 5 through 7 |
| **Insert** before line N | `LINE 3-2` + new content + `END` | Starts past the end (start > end), so it inserts before line 3 |

### Why "deterministic" is doing real work here

- **Line numbers always refer to the original file.** Edits are applied bottom-up (highest line number first), so an earlier range is never shifted by a later insertion. There is no "the line number has now moved, so the next block is wrong" class of bug.
- **The result is verifiable before you trust it.** `--verify N` re-reads the file and prints the changed lines with `N` lines of surrounding context. If the output is not what you expected, you read again, write a new replacements file, and re-apply. You never have to *assume* the edit landed correctly.
- **No escaping, no context matching.** Because the target is a numeric range, there is no substring to escape and no surrounding text that has to match. The failures that plague fuzzy edit formats simply have no surface to occur on.

### Encoding and line-ending fidelity

Files in the real world are not always plain UTF-8 with Unix line endings. SCORPIOX CODE detects and preserves the file's existing encoding and line endings automatically:

- **Encodings:** UTF-8, UTF-8 with BOM, UTF-16 LE, UTF-16 BE.
- **Line endings:** CRLF (Windows) and LF (Unix) are detected and preserved, so a Windows checkout is not silently converted.

You are not expected to know or declare any of this. Read it, edit it, and the file comes back in the same encoding with the same line endings it started in.

---

## The tooling you get, and the tooling you are never blocked from

SCORPIOX CODE ships a built-in `preferred-file-tools` skill that recommends a small, high-precision set of utilities:

| Utility | Purpose | Replaces |
|---------|---------|----------|
| `scorpiox-readfile` | Read a file or a line range with exact line numbers | `cat`, `head`, `tail` |
| `scorpiox-grep` | Recursive, encoding-aware search; auto-skips `.git`, `node_modules`, `__pycache__`, and binary files | `grep`, `find` |
| `scorpiox-editfile` | Deterministic line-range edits with `--verify` | `patch`, `sed`, `awk` |
| `CreateFile` | Write the small, uniquely-named replacements file | — |

These are **preferred**, not **required**. That word is the whole philosophy.

The moment a harness says "you *must* use my proprietary edit tool and nothing else," it has quietly reduced your leverage. You can no longer use the editor, the shell command, or the native utility you already trust — even when your tool would do the job better. You are now inside a sandbox defined by one vendor's abstraction, and every task routes through a format you cannot fully control or audit.

SCORPIOX CODE does not do that:

- **Standard tools always work.** `sed`, `awk`, `patch`, `git apply`, your editor of choice — all remain available. Nothing is disabled.
- **Direct shell commands always work.** If the deterministic line-range tool is the right fit, use it. If a one-liner is, use the one-liner. The agent and you decide per task.
- **Native utilities always work.** The preferred tools are cross-platform, dependency-free, and inspectable.

The high-precision tools are an *option that is good at a specific job*. They are not a cage. You keep full control, and the agent keeps full control — without either of you being forced into a lock-in the vendor can tighten later.

---

## How this compares to the other harnesses

| Harness | Editing model | What the developer is locked into |
|---------|---------------|-----------------------------------|
| **SCORPIOX CODE** | Deterministic numeric line ranges + `--verify`; standard tools always available | Nothing. Preferred tools are optional, not enforced |
| **Claude Code** | Fuzzy search-and-replace `Edit` tool with required context | A proprietary edit format whose context must match; raw shell is possible, but the model is steered toward the built-in tool |
| **OpenCode** | Built-in edit/patch primitives with fuzzy matching | A vendor-defined edit abstraction over your files |
| **Aider** | Unified diff or whole-file edit formats (model-configurable) | A diff/whole-file format that must regenerate correctly, or the change is lost |
| **Cursor** | Proprietary apply model that regenerates file content | A black-box "apply" you cannot audit line-by-line |
| **Pi / Hermes** | Harness-specific edit primitives | Each vendor's own editing sandbox |

The pattern across the others: the edit boundary is implicit, the format is proprietary, and the tooling is designed to be the *only* path. SCORPIOX CODE inverts that — explicit numeric boundaries, a plain-text replacements file you can read, a verification step, and no lock-in on any of it.

---

## The workflow, end to end

```bash
# 1. Read the exact lines you need, with their real line numbers
scorpiox-readfile src/main.c 50 80

# 2. Write the edit to a uniquely-named replacements file (via CreateFile)
#    /tmp/sx_edit_fix_null_check.txt:
#        LINE 55-57
#        if (ptr == NULL) {
#            return -1;
#        }
#        END

# 3. Apply and verify in one step
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2

# 4. If verify looks wrong: read again, write a new file with a new name, re-apply.
```

A few practical notes:

- **Multiple blocks in one file.** One replacements file can contain several `LINE ... END` blocks. They are applied bottom-up, so all coordinates stay valid.
- **Always read before you edit.** The line numbers in your replacements file must match what `scorpiox-readfile` showed you. Reading first is what makes the range deterministic.
- **Unique filenames per edit.** Give each replacements file its own descriptive name. There is no reason to reuse or delete previous ones.

---

## TL;DR

- **Deterministic line ranges, not fuzzy matching.** Edits name explicit `LINE start-end` coordinates applied bottom-up, so ranges never drift and line numbers always mean what you think they mean.
- **Inspectable and verifiable.** The edit is a plain-text replacements file you can read, and `--verify` prints the changed lines back to you before you trust the result.
- **Encoding and line endings preserved** — UTF-8, UTF-8 BOM, UTF-16 LE/BE, CRLF and LF, automatically.
- **True autonomy, not lock-in.** The high-precision tools are *preferred*, never *forced*. Standard tools, direct shell commands, and native utilities always work — no proprietary sandbox, no vendor-defined edit format you cannot audit or escape.

You get precise, deterministic control over every edit — and you keep the right to do it any way you want.
