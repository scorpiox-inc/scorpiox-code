# Deterministic File Editing & Developer Autonomy in SCORPIOX CODE

Most AI coding tools make one of two bad bets about how the model should touch your files.

The first bet is **fuzzy search-and-replace**: the model emits a block of "old text" surrounded by context, and the harness searches the file for that block and swaps it in. The second is **full-file rewrite**: the model retypes the entire file from scratch. Both paths share the same failure — the edit you *intended* is not the edit that *lands*, and you find out only when the diff is wrong.

SCORPIOX CODE takes a different position: **give the agent exact, deterministic line ranges — and never lock you out of doing it your own way.** This page explains how the editing model works, what "deterministic" means in practice, and why you are never trapped in a proprietary editing sandbox.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** an agent does not guess "somewhere around the function I was looking at." It says *replace lines 55 through 57* — a coordinate, not a guess — and you can inspect the exact edit before it commits.

---

## The problem with "clever" edit formats

The dominant approaches in other harnesses each have a known failure mode:

| Approach | Where it breaks |
|----------|-----------------|
| **Fuzzy search-and-replace block** | The model's "old text" must match the file byte-for-byte. A renamed variable, a reformatted line, or a duplicate snippet anywhere in the file breaks the match. The harness reports "cannot find matching context" and the agent loops. |
| **Unified diff / patch** | The model must reproduce the surrounding context lines and exact line numbers for every hunk. Off by one line and the patch is rejected or misapplied. |
| **Full-file overwrite** | The model retypes the entire file. Whitespace drift, dropped blank lines, and unrelated formatting changes creep in silently — and the approach stops scaling past a few hundred lines. |
| **Proprietary apply model** | A vendor model regenerates the file from a prompt. Non-deterministic, opaque, and you cannot audit exactly what changed line by line. |

The common thread: **the edit boundary is implicit.** The model has to *infer* where its change starts and ends from prose context, and every inference is a place it can be wrong.

SCORPIOX CODE makes the boundary **explicit and numeric.**

---

## Full control: deterministic line ranges

The core primitive is a numbered line range. The workflow is three steps, and every step is inspectable before it commits:

### Step 1 — Read with line numbers

`scorpiox-readfile` prints every line prefixed with its exact line number, so the agent (and you) see the real coordinates in the file rather than an abstract context window:

```bash
scorpiox-readfile src/main.c 50 80
```

```
50: void handle_request(req) {
51:     if (ptr == NULL) {
52:         log_warning("null pointer");
53:         return -1;
54:     }
```

### Step 2 — Write the edit as a small, reviewable file

The edit is a plain-text **replacements file**. Each block names a `LINE <start>-<end>` range and the new content. You can open this file and read exactly what is about to happen:

```
LINE 51-53
if (ptr == NULL) {
    return -1;
}
END
```

### Step 3 — Apply and verify in one shot

`scorpiox-editfile` applies the change and, with `--verify`, prints back the changed region with surrounding context:

```bash
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2
```

The three operations map cleanly to the range arithmetic:

| Operation | Syntax | What it does |
|-----------|--------|--------------|
| **Replace** a range | `LINE 5-7` + new content + `END` | Overwrites lines 5 through 7 with the new content |
| **Delete** a range | `LINE 5-7` + `END` (empty block) | Removes lines 5 through 7 |
| **Insert** before line N | `LINE N-(N-1)` + new content + `END` | `LINE 3-2` inserts before line 3 (start > end signals insert) |

### Why "deterministic" is doing real work

- **Line numbers always refer to the original file.** Edits are applied **bottom-up** (highest line number first), so an earlier range is never shifted by a later insertion. There is no "the line number has now moved, so the next block is wrong" class of bug.
- **The result is verifiable before you trust it.** `--verify N` re-reads the file and prints the changed lines with `N` lines of surrounding context. If the output is not what you expected, you read again, write a new replacements file, and re-apply. You never have to *assume* the edit landed correctly.
- **No escaping, no context matching.** Because the target is a numeric range, there is no substring to escape and no surrounding text that has to match. The failures that plague fuzzy edit formats have no surface to occur on.
- **Multiple edits in one pass.** One replacements file can contain several `LINE ... END` blocks. They are all applied in a single invocation, bottom-up, so every coordinate stays valid.

### Encoding and line-ending fidelity

Files in the real world are not always plain UTF-8 with Unix line endings. SCORPIOX CODE detects and preserves the file's existing encoding and line endings automatically:

| Property | Supported values |
|----------|-----------------|
| **Encodings** | UTF-8, UTF-8 with BOM, UTF-16 LE, UTF-16 BE |
| **Line endings** | LF (Unix) and CRLF (Windows) |

You do not declare any of this. Read the file, edit it, and it comes back in the same encoding with the same line endings it started in. A Windows checkout is not silently converted; a UTF-16 file is not mangled into Latin-1.

---

## The tools you get, and the tools you are never blocked from

SCORPIOX CODE ships a built-in **preferred-file-tools** skill that recommends a small, high-precision set of cross-platform utilities:

| Tool | Purpose | Replaces |
|------|---------|----------|
| `scorpiox-readfile` | Read a file or a line range with exact line numbers | `cat`, `head`, `tail`, `Get-Content` |
| `scorpiox-grep` | Recursive, encoding-aware search; auto-skips `.git`, `node_modules`, `__pycache__`, and binary files | `grep`, `find`, `Select-String` |
| `scorpiox-editfile` | Deterministic line-range edits with `--verify` | `patch`, `sed`, `awk` |
| `CreateFile` | Write the small, uniquely-named replacements file | — |

These are **preferred**, not **required.** That word is the whole philosophy.

The moment a harness says "you *must* use my proprietary edit tool and nothing else," it has quietly reduced your leverage. You can no longer use the editor, the shell command, or the native utility you already trust — even when your own tool would do the job better. You are now inside a sandbox defined by one vendor's abstraction, and every task routes through a format you cannot fully control or audit.

SCORPIOX CODE does not do that:

- **Standard tools always work.** `sed`, `awk`, `patch`, `git apply`, your editor of choice — all remain available. Nothing is disabled.
- **Direct shell commands always work.** If the deterministic line-range tool is the right fit, use it. If a one-liner is the right fit, use the one-liner. The agent and you decide per task.
- **Native utilities always work.** The preferred tools are cross-platform and dependency-free, but they are an option, not a cage.

The high-precision tools are *good at a specific job*. They are not a lock-in mechanism the vendor can tighten later.

---

## How this compares to the other harnesses

| Dimension | SCORPIOX CODE | Claude Code / Cursor / OpenCode | Aider | Pi / Hermes |
|-----------|---------------|---------------------------------|-------|-------------|
| **Edit boundary** | Explicit numeric line range (`LINE 55-57`) | Implicit — inferred from surrounding context text | Explicit line numbers, but inside a diff hunk | Varies; often implicit context matching |
| **Failure mode** | Line number out of range — explicit and recoverable | "Cannot find matching context" — opaque, agent loops | Hunk does not apply — fuzz-dependent, may silently misapply | Context mismatch or full-file drift |
| **Inspectable before commit** | Yes — the replacements file is plain text you can open and review | No — the context block is parsed by the harness, not by you | Partial — the diff is readable but the hunk anchors are fragile | Depends on harness |
| **Multiple edits in one pass** | Yes — multiple `LINE` blocks in one file, applied bottom-up | Usually one edit per tool call | Yes, but each hunk must anchor independently | Varies |
| **Encoding safety** | UTF-8, UTF-8 BOM, UTF-16 LE/BE; LF and CRLF preserved automatically | Most assume UTF-8 / LF; BOM and UTF-16 files can break | Assumes UTF-8 by default | Varies |
| **Can you bypass the tool?** | Yes — shell, `sed`, `patch`, your editor all work | No — the harness controls how edits reach the filesystem | No — Aider's patch pipeline is the only path | Varies |
| **Lock-in risk** | None — the tools are preferred, never forced | High — you are trapped in the harness's editing model | High — you are trapped in Aider's diff pipeline | Medium to high |

The distinction is not that SCORPIOX CODE's tools are the only ones that can work. It is that **you are never blocked** — and the precision tool is something you can *opt into* because it is genuinely better for a common task, not something the vendor *forces* because it controls the pipeline.

---

## A concrete example

Suppose you are working in a C file and need to add a null-pointer guard.

```bash
# 1. Read the exact lines you need, with their real line numbers
scorpiox-readfile src/main.c 50 80

# 2. Write the edit to a uniquely-named replacements file
#    /tmp/sx_edit_fix_null_check.txt:
#        LINE 55-57
#        if (ptr == NULL) {
#            return -1;
#        }
#        END

# 3. Apply and verify in one step
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2
```

`--verify 2` prints the changed region with two lines of context above and below, in the same numbered format as `scorpiox-readfile`. If the output is not what you expected, you read again, write a new replacements file with a different name, and re-apply. There is no hidden state, no session to reset, and no proprietary format to learn.

### Practical notes

- **Always read before you edit.** The line numbers in your replacements file must match what `scorpiox-readfile` showed you. Reading first is what makes the range deterministic.
- **Unique filenames per edit.** Give each replacements file its own descriptive name (e.g. `sx_edit_fix_null_check.txt`, `sx_edit_add_logging.txt`). There is no reason to reuse or delete previous ones.
- **Bottom-up means order does not matter.** Whether you write the top-of-file block first or the bottom-of-file block first in the replacements file, the result is the same. The tool sorts and applies from the highest line number down.
- **Out-of-range is an error, not a silent corruption.** If you specify `LINE 200-250` on a 100-line file, the tool reports the error and does not write anything.

---

## TL;DR

- **Deterministic line ranges, not fuzzy matching.** Edits name explicit `LINE start-end` coordinates applied bottom-up, so ranges never drift and line numbers always mean what you think they mean.
- **Inspectable and verifiable.** The edit is a plain-text replacements file you can read, and `--verify` prints the changed lines back to you before you trust the result.
- **Encoding and line endings preserved automatically.** UTF-8, UTF-8 BOM, UTF-16 LE/BE, CRLF and LF — no configuration, no surprises.
- **True autonomy, not lock-in.** The high-precision tools are *preferred*, never *forced*. Standard tools, direct shell commands, and native utilities always work — no proprietary sandbox, no vendor-defined edit format you cannot audit or escape.

You get precise, deterministic control over every edit — and you keep the right to do it any way you want.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Data Privacy and Zero Data Collection](data-privacy.md)
