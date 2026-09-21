# Project Instructions in SCORPIOX CODE: CLAUDE.md and AGENTS.md

Give SCORPIOX CODE a single file in your repo and it will follow your build commands, coding style, and non-negotiable rules for every session — without you re-typing them. That file is a **project instructions file**: a plain markdown document checked into source control that the agent reads at startup and treats as instructions that **override its default behavior**.

SCORPIOX CODE reads `CLAUDE.md` and `AGENTS.md`. Both are plain markdown. Pick one, write it once, and it ships with the repo.

This page covers how SCORPIOX CODE finds that file, exactly how it lands in the model's context, why the design keeps prompt-cache hits above 99%, and how it lines up with what the rest of the industry does.

Source of truth: the system-prompt builder and the config cascade at commit `24427d8`.

---

## The two files, and who wins

SCORPIOX CODE looks for exactly two file names in your **current working directory**:

| File | What it is | Status |
|------|------------|--------|
| `CLAUDE.md` | The native project-instructions file | **Checked first.** If it exists, it is used. |
| `AGENTS.md` | The cross-tool standard (Codex, Jules, Aider, Cursor, Copilot, and 40+ more) | **Fallback only.** Used when `CLAUDE.md` is absent. |

Three rules, no exceptions:

1. **`CLAUDE.md` always wins.** If both files exist, SCORPIOX CODE reads `CLAUDE.md` and ignores `AGENTS.md`. It does not merge them.
2. **Only the current directory is checked.** SCORPIOX CODE does not walk up the folder tree looking for ancestors' files, and it does not auto-load a file from a subdirectory as you work. It reads the one file in the directory you launched from.
3. **One file, verbatim.** Whatever is in the winning file is included in full. There is no truncation, no summarization, and no second file mixed in.

```text
repo/
├── CLAUDE.md      <- loaded (this one wins)
└── AGENTS.md      <- ignored while CLAUDE.md exists
```

> **Which should you write?** If you use only SCORPIOX CODE, `CLAUDE.md` is the native name. If your repo is already shared with Codex, Jules, Aider, Cursor, or Copilot and you do not want to maintain two files, write `AGENTS.md` and drop `CLAUDE.md` — SCORPIOX CODE will pick it up automatically. If you want SCORPIOX CODE to use a specific file, name it `CLAUDE.md`; it takes precedence.

---

## How the file reaches the model

SCORPIOX CODE has two places it can carry your instructions, controlled by a single setting. Both are **static**: the text is generated once per session and then held constant for the rest of the conversation.

### Default: the system prompt

With the default configuration, the winning file is wrapped in a header and placed in the model's **system prompt** — the highest-priority slot in the context window, ahead of the conversation.

What the model actually sees looks like this:

```
# Project Instructions (CLAUDE.md)

Your build/test commands, style rules, and conventions here...
```

Because it sits in the system prompt, it is the first thing in scope when the model plans an edit, and it stays there for the entire session.

### Alternative: the system-reminder block

You can switch the delivery into a `<system-reminder>` block that is attached to the user-message side of the conversation. In that mode the content is framed explicitly so the model knows the instructions override its defaults:

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.

Contents of /your/repo/CLAUDE.md (project instructions, checked into the codebase):

[your file, verbatim]

      IMPORTANT: this context may or may not be relevant to your tasks.
      You should not respond to this context unless it is highly relevant to your task.
</system-reminder>
```

The `OVERRIDE any default behavior` line is the load-bearing part: it tells the model your file outranks what it would have done otherwise.

### What happens when there is no file

If neither file exists (or the file is disabled, see below), SCORPIOX CODE sends a minimal placeholder instead of nothing. This is deliberate — it keeps the prompt structure byte-for-byte identical across sessions whether or not you have a file, so the provider's prompt cache still matches. An empty file or a file over the size limit is skipped the same way, with a warning in the logs.

---

## Why the design keeps prompt-cache hits above 99%

Prompt caching is what keeps a long session fast and cheap. The provider caches the **prefix** of your context. As long as that prefix is identical from one turn to the next, the turn is a cache *read* (a fraction of the cost and latency of a full re-encode). The moment the prefix changes, the cache misses and the whole conversation is re-encoded — the exact thing you are trying to avoid.

Most harnesses rebuild context dynamically: they re-sort tools, re-rank rules by relevance, or splice in a fresh timestamp and directory listing on every turn. That changes the prefix every turn, and the cache keeps busting. SCORPIOX CODE does the opposite, and the difference is the whole feature:

- **Generated once, then frozen.** The instructions (and the surrounding context block) are built a single time per session and then cached in memory. Every subsequent turn reuses the exact same bytes. The cache is only invalidated when the working directory actually changes, so a `cd` is the one thing that legitimately resets it.
- **Static framing, not dynamic re-ranking.** Rules are not scored per turn and reordered. The file goes in as-is, in the order you wrote it. A constant prefix is a cache hit.
- **The placeholder trick.** Even with no file, or with instructions disabled, the slot still exists with a fixed placeholder. "No file" and "file present" both produce a stable prefix shape, so toggling a file in and out does not silently wreck your cache.

The practical effect: across a normal interactive session, the instructions contribute **zero** cache churn. You should see cache reads on nearly every turn — in steady state, well over 99%. If your cache hit rate is collapsing mid-session, the cause is almost never the instructions file; it is a prefix that changes elsewhere (a growing, unstable context block). Keep the file, keep the prefix.

---

## Configuration

Two settings control project instructions. Both live in your SCORPIOX CODE config (the same `scorpiox-env.txt` style file the rest of the product uses).

| Key | Default | What it does |
|-----|---------|--------------|
| `PROMPT_INCLUDE_CLAUDEMD` | `1` | Master switch. `0` turns project instructions off entirely — no file is read, and the minimal placeholder is used instead. `1` (default) loads the file. |
| `USER_SYSTEM_REMINDER` | `0` | Delivery mode. `0` puts the file in the system prompt. `1` attaches it as a `<system-reminder>` block on the user side, framed as an override. |

```ini
# scorpiox-env.txt (relevant lines)

# 1 = load project instructions, 0 = off
PROMPT_INCLUDE_CLAUDEMD=1

# 0 = system prompt (default), 1 = system-reminder block
USER_SYSTEM_REMINDER=0
```

You rarely need to touch either. Leave `PROMPT_INCLUDE_CLAUDEMD=1` so the file is loaded, and choose the delivery mode based on how strongly you want the override framing. The system-reminder mode is the more forceful of the two because of the explicit `OVERRIDE any default behavior` line.

> **One file, one directory.** Because SCORPIOX CODE only reads the file in the directory you launch from, a monorepo package that needs its own conventions should carry its own `CLAUDE.md` at that package root — and you should launch SCORPIOX CODE from inside that package.

---

## Writing a file that actually gets followed

The file is **context, not enforcement** — the model reads it and weighs it, so how you write it decides how reliably it is followed. Specific, short, and well-structured beats long and vague.

### What to put in it

- Build, run, and test commands the model cannot guess.
- Non-negotiable conventions (formatter, indentation, naming, error-handling shape).
- Where things live ("API handlers live in `src/api/handlers/`").
- Gotchas ("integration tests need Docker running first").

### What to keep out

- Anything the model already knows from the code.
- Contradictions. If two rules fight, the model may pick one arbitrarily.
- Reminders of obvious steps. State the goal and let the model exercise judgment.

A file that is easy to scan is a file that gets followed:

```markdown
# Project: acme-api

## Commands
- Install deps: `pnpm install`
- Dev server: `pnpm dev`
- Run all tests: `pnpm test`
- Lint: `pnpm lint`

## Conventions
- TypeScript strict mode; single quotes, no semicolons.
- Handlers live in `src/api/handlers/`.
- Every endpoint change gets a test. Run `pnpm test` before committing.
- Integration tests require Docker running: `docker compose up -d`.

## Do not
- Do not commit `.env` or generated `*.lock` changes.
```

A few hundred lines is plenty. If a file is getting large, split the concern or tighten it — a bloated file costs context tokens on every turn and dilutes the rules you do want followed.

---

## How this compares to other harnesses

SCORPIOX CODE's model is deliberately simple: one file, in the working directory, loaded once, frozen for the session. Here is how that stacks against the rest of the field.

| Harness | File / location | Loading behavior | Merging / precedence |
|---------|----------------|------------------|----------------------|
| **SCORPIOX CODE** | `CLAUDE.md`, else `AGENTS.md`, in cwd | Loaded once per session; frozen in memory | `CLAUDE.md` wins; **never merged** |
| **Claude Code** (Anthropic) | `CLAUDE.md` / `.claude/CLAUDE.md`, plus `~/.claude/CLAUDE.md` and org-managed policy | Layered: managed → user → project, walked up the directory tree; subdirectory files load on demand | Multiple files combine; `@import` expansion; `.claude/rules/` for path-scoped rules |
| **AGENTS.md standard** | `AGENTS.md` at repo root | Adopted by 40+ agents (Codex, Jules, Aider, Cursor, Copilot, Devin, and more) | Single open file; "a README for agents" |
| **Cursor** | `.cursor/rules/*.mdc` (front-matter scoped) or `AGENTS.md` | Per-rule: always-apply, intelligent (model picks), glob-scoped, or manual `@mention` | Many rules can coexist; `AGENTS.md` is the simple single-file alternative |
| **Aider** | `CONVENTIONS.md` via `--read` / `.aider.conf.yml` | Loaded as a read-only, cache-friendly chat file | Additive; you choose which files to read |

### The two things worth noticing

**Claude Code is the broadest.** It layers several instruction files (managed policy, user, project), walks the directory hierarchy, supports `@file` imports and path-scoped `.claude/rules/`, and has an auto-memory system that writes its own notes. That is more machinery than most teams need, and it is exactly the kind of dynamic, multi-file, per-turn composition that makes prompt-cache stability harder to reason about. SCORPIOX CODE intentionally does not do all of this: one file, in one place, frozen once.

**SCORPIOX CODE reads the same standard everyone else is converging on.** `AGENTS.md` is the open, cross-tool format, and SCORPIOX CODE treats it as a first-class fallback. That means a repo you maintain for Codex or Aider today gets its instructions honored by SCORPIOX CODE with zero extra files. The one behavioral difference: SCORPIOX CODE does not *merge* `CLAUDE.md` and `AGENTS.md`. It picks one — `CLAUDE.md` if present, otherwise `AGENTS.md` — so there is no ambiguity about which rule wins and no duplicate context.

The shared property across all of these is the design goal itself: a checked-in, predictable file the agent reads before it acts. SCORPIOX CODE's contribution is doing it with the minimum moving parts, and freezing it in memory so the provider's prompt cache stays warm for the whole session.

---

## Migrating from another tool

- **From Claude Code:** copy your `./CLAUDE.md` (or `.claude/CLAUDE.md`) to the repo root as `CLAUDE.md`. SCORPIOX CODE reads the file in the working directory, so drop any reliance on `@import`, directory-tree walking, and `.claude/rules/` — if you need path-scoped guidance, put it in the file for the package you run from.
- **From a repo that already uses `AGENTS.md`:** nothing to do. Delete `CLAUDE.md` if it exists (otherwise `CLAUDE.md` wins and your `AGENTS.md` is ignored), and SCORPIOX CODE reads `AGENTS.md` directly.
- **From Aider:** whatever you were pointing at with `--read CONVENTIONS.md`, rename it `AGENTS.md` (or `CLAUDE.md`) at the package root. It is now loaded automatically.

---

## Quick checklist

- [ ] One instructions file at the package root you run SCORPIOX CODE from: `CLAUDE.md` (preferred) or `AGENTS.md`.
- [ ] Build/test commands, hard conventions, and the gotchas the model cannot guess are all in it.
- [ ] No contradictions; a few hundred lines max; headers and bullets, not walls of text.
- [ ] `PROMPT_INCLUDE_CLAUDEMD=1` (default) so the file is actually loaded.
- [ ] Pick a delivery mode: system prompt (default) or `USER_SYSTEM_REMINDER=1` for the explicit override framing.
- [ ] Long-running session and cache hits staying above 99% — if they are not, the problem is a changing prefix elsewhere, not this file.
