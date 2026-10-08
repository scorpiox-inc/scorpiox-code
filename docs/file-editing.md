# Deterministic File Editing & Developer Autonomy in SCORPIOX CODE

Every agent has to edit your code. Watch one fail at it and you will see the same three losses on repeat: a "search-and-replace" that cannot find the exact string and bounces the edit back, a patch that drifts one line off and lands in the wrong place, or a whole-file rewrite that silently reflows parts of the file nobody asked to touch. The editing layer is where most agent tools quietly lose reliability — and where they quietly take your autonomy with it, because once editing is proprietary, everything that touches a file has to go through the vendor's format.

SCORPIOX CODE is built on a different position: **editing should be deterministic, inspectable, and reversible — and you should never be forced into a single way of doing it.** This page covers the four preferred file tools, the line-based edit format that removes the entire class of fuzzy-diff failures, and why "preferred" means exactly that: preferred, never mandatory.

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** point at an explicit line range, write the replacement into a plain file anyone can open and read, apply it in one deterministic pass, and get the changed lines printed back in the same command — with the standing freedom to drop to `sed`, `patch`, your editor, or any native tool the moment that is the better move.

---

## The problem: fuzzy editing is a guess

Most harnesses let the agent *describe* an edit and then match that description against the file. Three shapes dominate, and all three are the model guessing against your file:

| Mechanism | How it works | Where it fails |
|-----------|--------------|----------------|
| **Search-and-replace blocks** | The model emits an "old string" it believes is in the file, plus a "new string". The tool locates the old string and swaps it. | The old string must match the file exactly. One whitespace slip, one tab where the file has spaces, one line the model mis-remembered from an earlier read, and the edit is rejected. The model is reconstructing your file from memory instead of addressing it. |
| **Unified diffs / patch formats** | The model emits `---`/`+++` hunks with context lines and a vendor grammar on top. The tool applies hunks positionally. | Off-by-one drift after an earlier hunk, context that no longer matches, or an invented grammar token. A bad hunk is either rejected or, worse, applied somewhere nearby. |
| **Whole-file overwrite** | The model re-types the entire file and the tool replaces it. | Expensive, and every line the model elides or "fixes" is a silent change you have to diff yourself to find. Scales badly past a few hundred lines. |

The common thread is that **the model is regenerating part of your file from memory and hoping it matches.** Precision degrades as files get longer, every failure costs a turn, and the recovery loop (re-read, re-emit, retry) is exactly the loop that burns tokens on no progress. Some harnesses paper over this with fuzzy fallbacks — whitespace normalization, smart-quote folding, similarity thresholds, even a second model that re-applies the edit — which trades a hard failure for a softer one that can silently match the wrong location.

SCORPIOX CODE removes the guess. The agent does not describe a string to find. It **names the line range**. The edit becomes a fact about the file's structure rather than a reconstruction of its contents.

---

## The four preferred file tools

SCORPIOX CODE ships a small, sharp set of file tools. They are the *preferred* path for reading, searching, and editing, each one a plain inspectable command with no hidden state, and each one a standalone binary that works from your own shell exactly as it works from the agent.

| Tool | Job | What makes it deterministic |
|------|-----|-----------------------------|
| **`scorpiox-readfile`** | Read a file or a line range, numbered. | Every line comes back as `N: content`, so the numbers you edit against are the numbers the file actually has. |
| **`scorpiox-grep`** | Search files. | Zero-dependency, cross-platform, auto-skips `.git`, `node_modules`, `__pycache__`, and binary files; reads UTF-8, UTF-8 BOM, and UTF-16 LE/BE. |
| **`scorpiox-editfile`** | Apply line-based edits. | Edits by explicit `LINE start-end` blocks, applied bottom-up so every number refers to the original file. |
| **`CreateFile`** | Write a file from scratch. | Authors each replacement file as a real named file on disk — an artifact you and the agent can both open, diff, and keep. |

The pattern is the same every time: **read the range you care about, write the replacement to a file, apply it, verify it.** Nothing is matched, nothing is fuzzy, nothing lives only in the model's head.

### Reading: numbers you can point at

```bash
# Whole file, numbered
scorpiox-readfile src/main.c

# Just the region you plan to change
scorpiox-readfile src/main.c 50 60
```

Output is one line per source line, prefixed with the real line number:

```
50: void handle_request(req) {
51:     ptr = lookup(req->id);
52:     if (ptr == NULL) {
53:         return -1;
54:     }
55: }
```

Because the numbers are authoritative, the next step is not "hope the string matches" — it is "replace lines 52-54". Reading a range that starts past the end of the file is a hard error that reports the actual line count, so a stale number fails loudly instead of silently editing nothing.

### Searching: the boring, reliable kind

```bash
# Recursive, with line numbers
scorpiox-grep -rn "handle_request" src/

# Case-insensitive, only C files
scorpiox-grep -ri --include "*.c" "TODO" .

# Regex with context lines
scorpiox-grep -rnE -C 3 "TODO|FIXME" src/

# Filenames only, or a count per file
scorpiox-grep -rl "handle_request" .
scorpiox-grep -rc "TODO" src/
```

It does the job you expect from `grep` — fixed-string and basic regex modes, `-i`, `-w`, `-v`, `-o`, `-A`/`-B`/`-C` context, `-e` for multiple patterns, `--include`/`--exclude`/`--exclude-dir` globs, stdin piping, and exit code `0`/`1` for scripted use — without asking which grep variant your platform happened to ship. Hidden directories, VCS folders, dependency trees, and binaries are skipped automatically, and UTF-16 files that other tools would call binary are searched as text.

### Editing: by line range, not by guess

An edit is a small **replacements file** containing one or more blocks. Each block opens with `LINE <start>-<end>` and closes with `END`; the new content sits between them. Three operations cover everything:

| Operation | How | Meaning |
|-----------|-----|---------|
| **Replace** lines | `LINE 5-7` + new content + `END` | Swap lines 5-7 for the new content. |
| **Delete** lines | `LINE 5-7` + `END` (empty block) | Remove lines 5-7. |
| **Insert** before line N | `LINE N-(N-1)` + new content + `END` | `LINE 3-2` inserts before line 3. |

A replacements file that fixes a null check, inserts a log call above it, and deletes a stale comment — all in one pass:

```
LINE 52-54
    if (ptr == NULL) {
        log_error("lookup failed for id=%d", req->id);
        return -1;
    }
END
LINE 58-57
    log_debug("request accepted");
END
LINE 61-61
END
```

Two properties make this deterministic rather than clever:

1. **Bottom-up application.** Blocks are sorted and applied highest line number first, so every line number always refers to the **original** file as read. Inserting ten lines at the top does not shift the range you want to change next. No re-numbering, no drift.
2. **Multiple blocks, one file.** Independent edits batch into a single replacements file and land in one apply. Overlapping ranges are your business, not the tool's — the last block touching a line wins, which keeps the behavior predictable instead of heuristic.

Indentation inside a block is preserved verbatim, because it becomes file content. The `LINE` header must start its line, and `END` must stand alone on its line; everything else is data. That is the entire grammar.

### The workflow, end to end

1. **Read** the target region to get real line numbers.
   `scorpiox-readfile src/main.c 50 60`
2. **Create the replacements file** with the `CreateFile` tool, under a unique descriptive name each time — for example `/tmp/sx_edit_fix_null_check.txt`. Never reuse or delete a previous edit file; each one is your audit trail.
3. **Apply and verify in one command.**
   `scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2`
4. **If the verify output is not what you expected**, read again, write a new replacements file with a new name, and apply again.

Step 3 is the one that removes the failure mode entirely.

### `--verify`: the edit prints itself back

Most tools tell you "applied" and hope. `--verify` **re-reads the file that was just written and prints the changed lines back**, in the same numbered format as `scorpiox-readfile`, one header per block:

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

`--verify` prints the changed region; `--verify N` widens it with `N` boundary lines of context on each side — two is the usual choice, enough to see the edit sitting correctly between its neighbours. Deletions are shown as the surrounding region after the lines are gone.

This is a check on reality, not a confirmation of intent. The tool re-opens the file from disk and prints what is there. If the region looks wrong, the file is wrong — read, rewrite, reapply. "Did that land where I meant?" becomes a line you can look at instead of a diff you have to reconstruct.

### Encoding and line endings: preserved, not rewritten

A file is more than its text. `scorpiox-editfile` detects the target's encoding and dominant line ending, and writes the result back in **the same ones**:

| Dimension | What is preserved |
|-----------|-------------------|
| **Encoding** | UTF-8, UTF-8 with BOM, UTF-16 LE, UTF-16 BE |
| **Line endings** | LF (Unix) and CRLF (Windows), by the file's dominant style |

Edit a UTF-16 file with CRLF endings on a Windows checkout and it stays UTF-16 with CRLF — no silent BOM added or stripped, no mass line-ending conversion that trips a `.gitattributes` policy or dirties every line of a diff. The replacements file itself is plain UTF-8 (a leading BOM is tolerated), so the agent writes it the same way on every platform while the target keeps its own identity.

There is no escaping problem to solve either. The replacement is a file, not a JSON string, not a heredoc, not a PowerShell here-string — so quotes, backslashes, dollars, and braces arrive exactly as written. That single design choice removes the escaping failures that plague shell-side editing on Windows in particular.

---

## Developer autonomy: preferred, never locked-in

The line-based tools are the high-precision path, recommended because they are the least error-prone. But nothing walls you or the agent into them:

- **Standard shell tools always work.** `cat`, `head`, `tail`, `grep`, `sed`, `awk`, `patch`, `perl` — whatever your platform ships keeps working, with no special permission and no wrapper in the way. On Windows installs, the bundled `bin/` directory puts GNU coreutils and `patch` on the shell's `PATH`, so the same one-liners work there too.
- **Native utilities always work.** Your editor, your build system, your version control — the agent can drive the same commands you would, and `Ctrl+X` opens your own `EDITOR`/`VISUAL` (with a bundled minimal vi as the always-present fallback) for the times a human wants the keyboard.
- **You can mix freely.** Read with `scorpiox-readfile`, apply with `sed`, confirm with `git diff`. The tools do not compete; they are layers you reach for when you want that precision.

That distinction is the whole point. A tool that *recommends* a method and a tool that *mandates* a method look identical in a demo and behave oppositely in real work. SCORPIOX CODE does not take your keyboard away to be "safe" — the preferred tools sit on top of an ordinary filesystem, not in front of it.

---

## How other harnesses approach it — and the lock-in they introduce

The alternatives are not bad; they are **narrow**. Each makes one editing mechanism the only door into your files, and each optimizes for a scenario it cannot see: that you might want a different one.

| Harness | Dominant editing mechanism | Where it gets rigid |
|---------|----------------------------|---------------------|
| **Claude Code** | Exact search-and-replace (`old_string`/`new_string`), plus whole-file `Write`. | The old string must match byte-for-byte and appear exactly once; a single whitespace difference is enough to miss, and the harness gates edits behind its own read-before-edit bookkeeping. |
| **OpenCode** | Exact `oldString`/`newString` replacement, plus a patch tool. | The documented behavior is exact match with a multi-match error; fallback matchers with similarity thresholds sit underneath, so a near-miss can land on the wrong-but-similar region. |
| **Aider** | SEARCH/REPLACE fence blocks, unified diffs, or whole-file rewrites, chosen per model. | The search block must match exactly; failed blocks trigger "did you mean" retries and format-conformance loops that end in a re-read. Its own benchmarks favor diffs only because they suppress lazy elision, not because they are more precise. |
| **Cursor** | Editor-driven apply over search-and-replace, with a separate apply path and checkpoints for rollback. | The edit lives inside the IDE's apply flow; precision depends on the model matching the file, and recovery is a checkpoint restore rather than a corrected edit. |
| **Pi** | Minimal four-tool surface with exact `oldText` replacement. | Deliberately tiny and refreshingly honest, but the match-or-fail dynamic is the same: the model's string must find the file's string, and there is no addressing primitive to fall back on. |
| **Hermes** | Fuzzy find-and-replace with nine ordered match strategies, plus whole-file writes and vendor patch dialects. | The tolerance is the feature and the risk: whitespace-insensitive and similarity-thresholded matching means a sloppy old string still applies — somewhere. A telemetry counter exists precisely to track which fuzzy strategy fired and how often edits came back ambiguous or unmatched. |

Read down the columns and the pattern shows: the model is **regenerating part of the file from memory**, and the harness is **the only tool that accepts the result**. That is a sandbox with a polite name. You are not editing your file; you are submitting a guess to a format the harness owns, and the harness decides whether the guess was good enough — sometimes with a fuzzy pass that makes the wrong-but-close answer succeed.

SCORPIOX CODE flips the relationship:

| | **Fuzzy-editing harnesses** | **SCORPIOX CODE** |
|---|---|---|
| **Addressing** | "Find this string." | "Change lines N-M." |
| **Source of truth** | The model's memory of the file. | The file's actual line numbers, read moments before. |
| **Failure mode** | Mismatch → rejected, or near-miss → silently misapplied. | Out-of-range → explicit error naming the line count. In-range → exactly that range. |
| **Verification** | Trust "applied", or a verifier footer after the fact. | `--verify` re-reads the written file, numbered, in the same command. |
| **The edit artifact** | A block the model produced inside one turn. | A plain file on disk you can open, diff, and keep. |
| **Encoding** | Usually normalized to the harness's choice. | BOM, UTF-16 LE/BE, and CRLF are preserved per file. |
| **Your freedom** | Use the harness's one tool. | Use the preferred tool *or* any standard shell / native command. |

The last row is the one the others cannot copy without changing their model: you can opt out of the precision entirely and still be doing exactly what you intended, with tools you already know.

---

## When to use what

| Situation | Reach for |
|-----------|-----------|
| Multi-line change in a file you have just read | `scorpiox-readfile` range, then `scorpiox-editfile` with `--verify 2` |
| Several independent edits in one file | One replacements file, several blocks, one apply |
| Adding a block above or below a known line | Insert form `LINE N-(N-1)` |
| Mechanical one-liner (`s/old/new/g` across a tree) | `sed` or `perl` through the shell |
| A patch you already have from review or a colleague | `git apply` / `patch` |
| Brand-new file | `CreateFile` |
| Large generated output or a full rewrite you authored deliberately | `CreateFile` to a new path, then swap |
| Unknown territory — find the right spot first | `scorpiox-grep -rn` to locate, then read the range |

The preferred tools win when the change is surgical and the file is long enough that retyping it is a risk. The shell wins when the change is mechanical and a one-liner already exists. Neither path is second-class, and mixing them costs nothing.

---

## Gotchas

- **Read before you edit.** The numbers in `LINE start-end` must be the file's real numbers. Read the region first; the `N: content` output is the contract between the two tools.
- **Insert uses `start > end`.** To insert *before* line 3, write `LINE 3-2`. A normal `start-end` (start ≤ end) replaces or deletes; only the inverted form inserts. `LINE 1-0` inserts at the top of an empty or non-empty file alike.
- **Line numbers are 1-based and always refer to the original file.** Blocks apply bottom-up, so earlier blocks never shift later ones. Do not renumber as you go.
- **Out-of-range is a hard error, not a guess.** A start line past the end of the file fails with the actual line count and nothing is written. An end line past the end is clamped to the last line, so a wide range trims rather than failing.
- **Blocks are independent, not chained.** Each block is matched against the original file, so two blocks touching the same lines should be merged into one — the later block in sort order wins, which is rarely what you meant.
- **The replacements file is plain text with a tiny grammar.** `LINE` must start the line and `END` must stand alone; blank lines between blocks are ignored; indentation inside a block is preserved verbatim. A missing trailing `END` simply closes the last block, so a truncated file still applies the edits it did define.
- **Encoding is a read-write round trip, not a conversion.** A UTF-16 target is decoded to UTF-8 internally, edited, and re-encoded — so the replacements file stays plain UTF-8 while the target keeps its BOM and byte order. A file with no BOM stays BOM-free.
- **`--verify` reads the file you just wrote, not your intended output.** That is the point: it is a check on reality. If it looks wrong, the file is wrong — read, rewrite with a fresh file name, reapply.
- **Keep your edit files.** Unique names like `/tmp/sx_edit_add_logging.txt` are a free audit trail of every change the agent made, reviewable with `diff` long after the session ends.

---

## The bottom line

Fuzzy editing is a quiet tax. Every mismatched string, every off-by-one hunk, every whole-file retyping is a turn the model spends re-guessing a file it already has in front of it — and a place where a "close enough" match can silently touch the wrong lines.

SCORPIOX CODE makes editing a fact instead of a guess: point at the line range, write the replacement into a file you can read, apply it deterministically bottom-up, and read the result back with `--verify` in the same command. Encoding and line endings come back exactly as they went in, out-of-range edits fail loudly with real numbers, and every edit leaves a plain file behind that you can inspect. And none of it is mandatory — the moment a standard shell command or a native tool is the right move, you and the agent are free to use it.

That is the whole offer in one sentence: **complete, inspectable control over your files, with no abstraction standing between you and the disk.**

---

## Related

- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md) — how sessions-as-folders keep long edit campaigns auditable on disk.
- [Lazy Skill Loading](lazy-skill-loading.md) — the `SearchSkills` tool, which is the same zero-dependency grep engine pointed at your skill library.
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md) — how a self-running session keeps making verified edits after you stop typing.
- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md) — why every artifact on this page stays on your own disk.
