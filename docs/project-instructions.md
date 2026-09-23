# Project Instructions in SCORPIOX CODE: CLAUDE.md and AGENTS.md

You already have a file at the root of your repo that tells the agent how to build, test, and behave. The only question is which one SCORPIOX CODE reads, and whether it survives a long session without corrupting the prompt cache.

Most harnesses answer that question badly. They read one of several competing files, they merge them unpredictably, or they re-emit the instructions on every turn — which busts the model's prompt cache and quietly inflates your token bill.

SCORPIOX CODE makes the decision **deterministic, single-file, and cache-stable by design.** This page explains exactly which file loads, where it goes in the prompt, why the prompt cache stays warm across a hundred turns, and how the architecture compares to the other harnesses you have probably tried.

Source of truth: the system-prompt module at commit `6c70ad6`.

> **The whole idea in one line:** SCORPIOX CODE reads **one** instruction file from your working directory — `CLAUDE.md`, or `AGENTS.md` if there is no `CLAUDE.md` — wraps it in a single static `<system-reminder>` block, caches that block in memory for the lifetime of the session, and therefore never rewrites the bytes that the model has already cached.

---

## Which file loads, and why only one

The resolution rule is short, and the shortness is the point:

| Working directory contains | SCORPIOX CODE loads |
|-----------------------------|---------------------|
| `CLAUDE.md` and nothing else | `CLAUDE.md` |
| `AGENTS.md` and nothing else | `AGENTS.md` |
| **Both** `CLAUDE.md` and `AGENTS.md` | **`CLAUDE.md` only** |
| Neither file | Nothing (the instructions section is omitted) |

Three properties fall out of that table, and they are the whole reason this design exists:

1. **`CLAUDE.md` always wins.** If both files are present, `AGENTS.md` is ignored. There is no "both" row and no "which one is newer" heuristic.
2. **Never merged.** The files are not concatenated, diffed, or reconciled. You get exactly one of them, verbatim, byte for byte. A merge is where two sources of truth start disagreeing; refusing to merge keeps there being one.
3. **`AGENTS.md` is the cross-tool fallback.** If you standardise a repo on the open `AGENTS.md` format and have not written a `CLAUDE.md`, SCORPIOX CODE still works with zero changes. Add a `CLAUDE.md` later and it takes over automatically.

The file is read from the **current working directory** — the directory SCORPIOX CODE was launched in, not a hard-coded project root. Run it from a subdirectory and it looks in that subdirectory. That is deliberate: a monorepo with per-package `AGENTS.md` files behaves naturally when you `cd` into a package.

---

## Where the instructions live in the prompt

Once the file is selected, SCORPIOX CODE has two ways to hand it to the model, and they map to two different configuration choices.

### Default: a static block in the system prompt

With the default configuration the selected file is embedded **once**, in the `system` field of the request, under a header that names the file:

```
# Project Instructions (CLAUDE.md)

<your CLAUDE.md content, verbatim>
```

The system field is sent **in two blocks**, each individually cache-marked: a short identity line, then the block that carries your instructions. Splitting them is what lets the provider cache the stable prefix without re-sending the instructions on every turn.

### Optional: a per-turn `<system-reminder>`

There is a second, opt-in mode for providers that treat the system field as a fixed prefix. When enabled, the selected file is instead wrapped in a `<system-reminder>` envelope and **prepended to each user message**:

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to these
instructions. IMPORTANT: These instructions OVERRIDE any default behavior
and you MUST follow them exactly as written.

Contents of /path/to/CLAUDE.md (project instructions, checked into the codebase):

<your file, verbatim>

      IMPORTANT: this context may or may not be relevant to your tasks.
      You should not respond to this context unless it is highly relevant
      to your task.
</system-reminder>
```

Two things stand out in that envelope, and both are load-bearing:

- **`# claudeMd`** — the marker tells the model the content is project memory that **overrides default behaviour**. It is the same convention the upstream Claude Code harness uses, so a `CLAUDE.md` written for any tool reads the same here.
- **A static fallback string.** When the feature is disabled, or there is no instruction file, or the file is too large, the envelope degrades to a single short, fixed line — `<system-reminder>Context for cache consistency</system-reminder>` — rather than an empty or varying block. The bytes in that position never change shape, so the cache key never shifts.

---

## Cache stability: why the prompt cache survives a long session

This is the architectural bet, and it is the difference between a harness that gets cheap as a session grows and one that does not.

LLM providers cache a **prefix** of each request. If the first N tokens of turn 5 are identical to the first N tokens of turn 4, those tokens are served from cache — cheap and fast. The moment a prefix byte changes, the cache is invalidated from that byte to the end of the context, and the next turn re-prices everything after it.

Most harnesses bust their own cache for a mundane reason: they **re-derive the prompt each turn.** A timestamp, a working-directory string, or a freshly-joined instructions block changes between turn 4 and turn 5, the prefix diverges, and the provider re-scores the whole conversation. Over a long session that is the entire cost of "the agent is slow and expensive."

SCORPIOX CODE is built the other way around. The instruction block is **generated once, at the first turn, and pinned in memory** for the lifetime of the session:

| What changes between turns | What SCORPIOX CODE does | Effect on the cache |
|----------------------------|-------------------------|---------------------|
| User types a new message | Appends the user message; the instruction block is byte-identical | Prefix cache hit |
| A tool call returns output | Appends the output; the instruction block is untouched | Prefix cache hit |
| A few minutes pass | The cached block is not regenerated from a wall clock | Prefix cache hit |
| The instruction file is edited on disk | No re-read; the in-memory block from turn 1 is reused until the session ends | Prefix cache hit |
| A provider that wants a user-side reminder | A `<system-reminder>` of **exactly the same bytes** is prepended | Prefix cache hit |

Because the block that carries your instructions is immutable within a session, the provider's cache key for that prefix is stable, and the prefix cache hit rate stays at the high 90s across long sessions. The design does not chase a cache by luck — it **cannot** miss, because nothing in the prefix is allowed to move.

This is the "static system-reminder" property. The alternative — rebuilding the reminder on each turn so it reflects "the current state" — is what pushes competitors out of the cache. The trade-off is deliberate: instructions are **session-stable by design**, which is also exactly what a project instruction file should mean. If you change your instructions mid-session, you change them for the next session, not the middle of this one.

---

## Sizing: what happens to a large file

The loader enforces a hard size ceiling on the instruction file, with separate headroom for the two injection modes. If the file exceeds the budget, SCORPIOX CODE does **not** truncate it mid-sentence and inject the head. It **skips the file entirely** and, in reminder mode, substitutes the short fixed placeholder described above.

The consequence is worth knowing: a bloated `CLAUDE.md` does not become a half-loaded `CLAUDE.md`. It becomes a *no-op*, and the model behaves as if the file were absent. Keep the file small — well under the ceiling — and the full content is what the model sees. This also means a runaway or auto-generated instruction file can never silently rewrite the first N tokens of your context and nuke the cache with a 512 KB blob.

---

## Toggling it off

Both the file load and the reminder injection are configuration-gated, not hard-coded:

| What | Off by default | On |
|------|----------------|----|
| Load the instruction file into the prompt | Set the include-CLAUDE.md option to `0` | Default (`1`) |
| Prepend the per-turn `<system-reminder>` | Default (`0`) | Set the user-reminder option to `1` |

These are independent switches. You can turn the reminder off while the file still loads into the system prompt (the default), turn the reminder on to reinforce the context on every user turn, or disable the file entirely for a clean baseline. None of the three states changes the prefix-cache behaviour, because each state is itself a fixed, cacheable shape.

---

## How SCORPIOX CODE compares to the other harnesses

The table below is a side-by-side of the four instruction conventions you are most likely to have in your repo today. The rows that matter for you are *which file* and *how it is injected*, because those two rows are what determine prompt-cache behaviour and what happens when you have more than one file.

| Harness | Primary file | Fallback / related files | Merge behaviour | Injection point | Cache behaviour |
|---------|--------------|--------------------------|-----------------|-----------------|-----------------|
| **SCORPIOX CODE** | `CLAUDE.md` | `AGENTS.md` (only if no `CLAUDE.md`) | Never merged; single file wins | Static system block (default) **or** per-turn `<system-reminder>` (opt-in) | Block generated once, pinned in memory; prefix cache hit across turns |
| **Anthropic Claude Code** | `CLAUDE.md` (project) | Managed policy, `~/.claude/CLAUDE.md` (user), `CLAUDE.local.md` (local), `.claude/rules/*.md`, plus `AGENTS.md` as a fallback when no `CLAUDE.md` exists | Multiple scopes load in order, broadest-to-most-specific; `@path` imports expand at launch; `CLAUDE.md` wins over `AGENTS.md` when both exist in the working directory | System context, loaded at the start of each session | Multiple sources are concatenated at session start; stability depends on the user keeping those sources static |
| **OpenAI / SWE-bench ecosystem** | `AGENTS.md` | None — the file *is* the standard | n/a (single file by definition) | Provided as project context; consumed by Codex, Jules, Aider, Goose, Zed, Copilot, Devin, Cursor, and others | Stable when the file is static; the format is deliberately vendor-neutral so it works across tools |
| **Cursor** | `.cursorrules` (legacy) / `.cursor/rules/*.mdc` | Project rules, user rules, enterprise rules; `.mdc` files with frontmatter and glob `paths` scoping | Rules are merged from multiple scopes; path-scoped rules trigger on file reads | Injected into the model's context; rules without `paths` apply unconditionally | Multiple rules are joined; path-scoped rules add to context on demand, which changes the prompt as you navigate |
| **Aider** | `CONVENTIONS.md` (or any `.md`) | `--read` files, `.aider.conf.yml` | Files are read into the chat and marked read-only | Added to the conversation; `.aider.conf.yml` can pin a file to load every session | Aider documents its own prompt-caching mode; pinned `--read` files are marked read-only so they can be cached |

### What the comparison actually means for you

- **Single-file determinism is the differentiator.** Claude Code loads up to four scopes plus rules plus imports; Cursor merges project, user, and enterprise rules with path-triggered additions. Both are powerful, but both mean the effective instruction set is a *composition*, and compositions drift. SCORPIOX CODE reads exactly one file. You can reason about what the model saw because you can point at the bytes.
- **`AGENTS.md` interop is shared, but the fallback direction differs.** Claude Code reads `AGENTS.md` *only when no `CLAUDE.md` exists* in the working directory or above it — the same precedence SCORPIOX CODE uses. Aider has no fallback; the conventions file is whatever you point `--read` at. A repo standardised on `AGENTS.md` works with all four; a repo standardised on `CLAUDE.md` works with Claude Code and SCORPIOX CODE out of the box.
- **The cache story is the one the others do not engineer for.** Claude Code and Cursor both *can* cache, but the prefix is a concatenation of scopes that changes as you add files, change directories, or cross a rule's glob boundary. Aider exposes a caching mode but the convention file is a chat participant, not a pinned system prefix. SCORPIOX CODE makes the prefix *provably* stable by refusing to regenerate it — which is why long sessions stay in the cache without the user having to know about prompt caching at all.

### Choosing your file

- **You only ever run SCORPIOX CODE.** Write `CLAUDE.md`. It loads first and wins.
- **You run multiple agent tools on the same repo.** Write `AGENTS.md`. It is the cross-tool standard, SCORPIOX CODE reads it when no `CLAUDE.md` is present, and the same file feeds Codex, Aider, Zed, Copilot, and the rest.
- **You want both, intentionally.** Write `CLAUDE.md` for SCORPIOX CODE-specific behaviour and `AGENTS.md` for the rest of the ecosystem. Know that SCORPIOX CODE will use only `CLAUDE.md` — `AGENTS.md` is invisible to it in that configuration.
- **Your file is getting large.** Split it. The size ceiling is real, and a file over the budget is skipped, not truncated. Keep each instruction file small enough to be the whole thing the model sees.

---

## Gotchas

- **`CLAUDE.md` wins, always.** If both files exist, `AGENTS.md` is not read, not merged, and not even counted. There is no precedence flag and no "read both" mode in this design.
- **The file is read once per session.** Editing `CLAUDE.md` on disk mid-session does not change what the model sees until the next session. That is the cache-stability guarantee, not a bug.
- **Over the size ceiling, the file is skipped, not truncated.** A 512 KB instruction file behaves exactly like no file at all. Keep it small; there is no partial-load mode.
- **The lookup is from the launch directory, not the repo root.** Run SCORPIOX CODE from a subdirectory and it looks there. Per-package `AGENTS.md` files in a monorepo only load when you launch from that package.
- **The reminder and the system block are separate switches.** Disabling the per-turn `<system-reminder>` does not remove the file from the system prompt, and vice versa. Configure the two independently.
- **No merging means no cross-file references.** A `CLAUDE.md` that says "see AGENTS.md for the build steps" does not pull in `AGENTS.md` — SCORPIOX CODE does not follow file references in the instruction content. Inline what the model needs.

---

## TL;DR

- **One file, deterministic.** `CLAUDE.md` if present, else `AGENTS.md`, never both, never merged.
- **Injected statically.** A pinned block in the system prompt by default, or a per-turn `<system-reminder>` when you opt in — same bytes either way.
- **Cache-stable by construction.** The block is generated once per session and pinned in memory; nothing in the prefix is allowed to move, so the provider's prefix cache stays warm across the whole session.
- **Size-guarded.** An oversized file is skipped, not truncated, so a runaway file cannot silently rewrite the first N tokens of your context.
- **Interoperable.** `AGENTS.md` is the cross-tool fallback, so a repo standardised on the open format works without changes; `CLAUDE.md` takes precedence when both exist.

---

*Docs for SCORPIOX CODE @ 6c70ad6*
