# Deterministic File Editing & Developer Autonomy in SCORPIOX CODE

Every agent harness has to solve the same problem: an LLM decided that lines 55 to 57 of `src/main.c` need changing — how does that decision actually land in the file, reliably? Almost every harness answers with a **fuzzy abstraction**: a search/replace block that must guess which lines you meant, a unified diff that breaks when the file changed by one line, or a whole-file rewrite that silently normalizes your encoding and line endings. When the fuzzy match fails, the agent burns a turn regenerating the block and hoping.

SCORPIOX CODE takes the opposite approach: **deterministic, line-addressed editing.** You read a file with line numbers, you write a small replacements file that says exactly which line ranges to replace, delete, or insert, and you apply it. There is no guessing, no context-matching, no "did I quote the old text exactly?" — the line numbers are the contract, and the file on disk is the source of truth.

And the part that matters most: **nothing is forced.** These tools are the high-precision options the agent reaches for first. Your `sed`, your `awk`, your `perl -pi`, your Python scripts, your editor, your `git apply` — they all keep working. No sandbox, no locked tool list, no "the only way to touch this file is through our API."

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** read a file to get real line numbers, describe the edit as `LINE start-end` blocks in a plain-text replacements file, apply it with one command, and see the exact changed lines printed back instantly — with your encoding, BOM, and line endings preserved byte for byte.

---

## Why fuzzy editing is a trap

Most harnesses edit files through one of three fragile mechanisms:

| Mechanism | Where you see it | Why it is fragile |
|-----------|-----------------|------------------|
| **Fuzzy search/replace blocks** | Claude Code (`Edit`), Hermes, OpenCode | The old text must match the file exactly, or the harness does a fuzzy context match that can land on the *wrong* lines or fail entirely. The agent then regenerates the block and retries — burning tokens and turns on a round-trip that a line number would have avoided. |
| **Unified diffs** | Aider (`udiff` format), Cursor, OpenCode patches | A diff encodes context lines. If the file changed at all — one line above your target — the diff no longer applies. The agent must regenerate, and in a long session where the file keeps changing, that is a constant tax. |
| **Whole-file overwrite** | Aider (`wholefile`), Claude Code (`Write`) | The model re-emits the entire file. Any line it "forgets" or re-words is lost, and the write pipeline can normalize your BOM, your UTF-16, your CRLF endings — changes the model never made. |

The deeper problem is not the specific format. It is that the **contract between the model and the file is implicit**: the model has to *describe* the change in prose the harness then re-interprets. SCORPIOX CODE makes the contract explicit: line numbers.

---

## The four tools, in order

The built-in **preferred-file-tools** skill tells the agent to work in this loop. Each tool is a small, predictable command:

| Tool | Job | Replaces |
|------|-----|----------|
| `scorpiox-readfile` | Read a file (or a line range) with every line numbered | `cat`, `head`, `tail`, `Get-Content` |
| `scorpiox-grep` | Search files with line numbers; skips binaries, `.git`, `node_modules`, `__pycache__` | `grep`, `find`, `Select-String` |
| `CreateFile` | Write a new file — used to stage each replacements file with a unique name | heredocs, `echo >`, `Set-Content`, here-strings |
| `scorpiox-editfile` | Apply line-addressed edits to the target file and print back the changed lines | `patch`, `sed`, `awk`, string manipulation |

### 1. `scorpiox-readfile` — get the ground truth

```bash
scorpiox-readfile src/main.c            # whole file, numbered
scorpiox-readfile src/main.c 50 80      # just lines 50-80
```

Output is exactly what you see:

```
50:  if (ptr == NULL) {
51:      log("missing ptr");
52:  }
53:  do_work(ptr);
```

Handles **UTF-8, UTF-8 BOM, UTF-16 LE, UTF-16 BE** transparently — a UTF-16 file prints as readable numbered lines, not as garbage between every character. Those line numbers are the coordinates you will use for the edit, so there is no second source of truth to drift from.

### 2. `scorpiox-grep` — find the right lines fast

```bash
scorpiox-grep -rn "TODO" src/                 # recursive, line numbers
scorpiox-grep -ri --include "*.c" "retry" .   # case-insensitive, filtered
scorpiox-grep -rnE -C 3 "error|warn" src/     # regex with context lines
```

Same output shape as `scorpiox-readfile`: `path:lineno: content`, so the line number you find is the line number you edit.

### 3. `CreateFile` — stage the replacements file

The edit itself is described in a plain-text file. The agent writes it with the built-in `CreateFile` tool, giving each one a unique, descriptive name:

```
/tmp/sx_edit_fix_null_check.txt
```

```
LINE 50-52
if (ptr == NULL) {
    return -1;
}
END
```

Plain text, no escaping rules, no quotes to fight. Binary characters, tabs, long lines, multi-line content — all fine, because nothing is being quoted through a shell or a template engine. `CreateFile` refuses to overwrite an existing file (it parks the content in a temp file instead), so a stale replacements file can never silently clobber a new one.

### 4. `scorpiox-editfile` — apply and verify in one step

```bash
scorpiox-editfile src/main.c /tmp/sx_edit_fix_null_check.txt --verify 2
```

That single command does two things: applies the edit, then **re-reads the file from disk** and prints back the changed region plus 2 lines of context on each side (`--verify 2`), in the same numbered format as `scorpiox-readfile`. You are not being told the edit was applied — you are seeing the actual bytes that are now in the file.

---

## The `LINE start-end` format

Three operations, all expressed with line ranges against the **original** file:

| Operation | Syntax | Effect |
|-----------|--------|--------|
| **Replace** | `LINE 50-52` + new content + `END` | Lines 50-52 are replaced with the new content |
| **Delete** | `LINE 50-52` + `END` (no content) | Lines 50-52 are removed |
| **Insert** | `LINE 53-52` + new content + `END` (start > end) | New content is inserted **before** line 53 |

Two properties make this deterministic:

- **Bottom-up application.** Blocks are applied from the highest line number down, so every line number in the replacements file refers to the original file, even when you describe ten edits in one file. There is no "but that edit shifted the line numbers" problem — a whole batch of edits is safe in one command.
- **No context matching.** There is nothing to match. `LINE 50-52` means lines 50 through 52, full stop. If the line numbers are wrong, you will see it in the `--verify` output on the same command, not in a diff a minute later.

A single replacements file can carry many blocks, so a multi-region change (fix a header, add a log line, delete a dead block) is one `scorpiox-editfile` call.

---

## Encoding and line endings: preserved, not normalized

Files in the real world are not always UTF-8 with LF endings. `scorpiox-editfile` detects the target file's encoding and line endings and writes the result back in the same form:

| Preserved | What it means for you |
|-----------|----------------------|
| **UTF-8** | Default; unchanged |
| **UTF-8 with BOM** | BOM stays at the front of the file |
| **UTF-16 LE / UTF-16 BE** | A UTF-16 file stays UTF-16 after the edit |
| **CRLF vs. LF** | A Windows line-ending file keeps CRLF; a Unix file keeps LF |

Whole-file overwrite tools cannot promise any of that — the model re-emits text and the pipeline decides what it becomes. Here the guarantee is mechanical: the new content is written in the file's existing representation.

---

## The complete workflow

The recommended loop, and the one the built-in skill trains the agent to follow:

1. **Find**: `scorpiox-grep -rn "pattern" path/` — get the line numbers.
2. **Read**: `scorpiox-readfile path/file.c 50 80` — confirm the exact lines.
3. **Stage**: `CreateFile` writes the replacements file (unique name, e.g. `/tmp/sx_edit_fix_header.txt`).
4. **Apply and verify**: `scorpiox-editfile path/file.c /tmp/sx_edit_fix_header.txt --verify 2` — edit and see the result in one step.
5. **If the verify output looks wrong**, go back to step 2 with the new line numbers and stage a fresh replacements file. You never have to guess why a fuzzy match failed, because there is no fuzzy match to fail.

---

## Developer autonomy: preferred, never required

"Preferred file tools" is the name for a reason. The skill steers the agent toward the tools above because they are the most reliable options — numbered reads, encoding-safe edits, verifiable results. But they are **options, not a lockdown**:

- **Standard tools always work.** `sed`, `awk`, `perl`, `patch`, `git apply`, `python`, `node`, PowerShell — all of them can still read and write your files. Nothing in SCORPIOX CODE intercepts or restricts them.
- **Direct shell access is untouched.** Pipes, here-docs, `dd`, your build system, your formatters, your linters — the ordinary shell of your machine behaves exactly as it does without an agent in the loop.
- **Your editor is still your editor.** SCORPIOX CODE edits files on disk like any other program. Open the file in your IDE, review the change, keep it or roll it back with your normal tools.
- **You choose the workflow.** If your project has a strict `patch`-based review process, keep using it. If your CI expects a specific diff format, produce it. The agent has a precise tool *and* the full toolbox of the machine.

Contrast that with harnesses that route every file access through a curated tool list, where the supported way to edit is the only way. In SCORPIOX CODE the precision tools exist to make the common case bulletproof — and every uncommon case falls back to a tool you already know, on your terms.

---

## How the alternatives compare

Mechanisms differ across harnesses; the table summarizes the editing contract each one gives your agent and your workflow.

| Harness | Editing mechanism | What it constrains |
|---------|-------------------|--------------------|
| **Claude Code** | `Edit` (fuzzy search/replace with context) and `Write` (whole file) | The old text must be quoted to match; on mismatch the tool retries or fails. No line-addressed editing. |
| **OpenCode** | Edit/patch tools built on fuzzy matching | Same class of fragility as above: description-based matching, not line-based. |
| **Aider** | Configurable edit formats: `wholefile`, `udiff` (unified diff), `search/replace` blocks | You pick one format and live with its failure mode; unified diffs break on any unrelated change above the edit site; whole-file rewrites can normalize encoding and endings. |
| **Cursor** | Model generates a unified diff applied by the IDE | Editing is coupled to the Cursor editor; the diff must still re-match context lines to apply. |
| **Pi** | Minimal edit tooling | Simpler still, but still text-matching based rather than line-addressed. |
| **Hermes** | Fuzzy search/replace blocks | Context matching; wrong-line and no-match failures are handled by regeneration. |
| **SCORPIOX CODE** | `scorpiox-editfile` with `LINE start-end` ranges + `--verify` readback | Nothing: line ranges are exact; verification is built in; encoding/endings preserved; and every standard tool remains available alongside. |

The honest framing: fuzzy formats can work — for the easy files, on the easy days. The point of SCORPIOX CODE is that the hard cases (UTF-16 files, CRLF files, multi-region edits, files that changed a minute ago) stop being a gamble. And when a format is not right for you at all, you are never trapped: the shell is always open.

---

## Tips

- **Always read before you edit.** The line numbers in your replacements file must come from a `scorpiox-readfile` of the current file. If the file changed, re-read.
- **Give each replacements file a unique name** (`/tmp/sx_edit_fix_header.txt`, `/tmp/sx_edit_add_logging.txt`) and never reuse old ones. `CreateFile` protects you from overwriting, and a fresh name makes your edit history inspectable.
- **Always pass `--verify`** — `scorpiox-editfile target reps.txt --verify 2` gives you the edit and its proof in one command. The default `--verify 0` prints the changed lines exactly; the boundary count adds context.
- **Batch related edits into one replacements file.** Because application is bottom-up, many blocks in one file are as safe as one.
- **Trust the disk.** `--verify` output is generated by re-reading the written file, so it shows what is actually on disk, not what the tool believed it wrote.

---

*Docs for SCORPIOX CODE @ `655c38f` — deterministic line-based editing, full tool freedom, zero fuzzy matches.*
