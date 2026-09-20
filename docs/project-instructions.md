# Project Instructions in SCORPIOX CODE: CLAUDE.md and AGENTS.md

Every agent session starts with a blank mind. The only way to make an agent behave the way *your* project needs — the build command, the test convention, "always run the formatter," "do not touch generated files" — is to tell it, in a file, before it does anything. That file is your **project instructions**.

SCORPIOX CODE reads one project instructions file from your working directory and folds it into what the model sees. This page explains exactly which file wins, how it is injected, why the design keeps your prompt cache warm, and how it compares to Claude Code, the AGENTS.md standard, Cursor, and Aider.

Source of truth: the system-prompt assembly in `sx_systemprompt.c` at commit `5fd054b`.

---

## The one-sentence version

SCORPIOX CODE looks in your current working directory for a `CLAUDE.md`; if that file does not exist it falls back to an `AGENTS.md`. It never merges the two, and `CLAUDE.md` always wins. Whatever it finds is wrapped in a single, stable `<system-reminder>` block that tells the model the instructions override default behavior, and it is cached in memory for the life of the session so your prompt cache does not bust on every turn.

---

## Which file wins

SCORPIOX CODE checks the working directory you launch from, in this order:

| Order | File | When it is used |
|-------|------|-----------------|
| 1 | `CLAUDE.md` | Always preferred. If it exists, this is the file. |
| 2 | `AGENTS.md` | Only when `CLAUDE.md` is absent. |

Three consequences fall out of this rule:

- **`CLAUDE.md` is the primary contract.** If you want your instructions to be authoritative in SCORPIOX CODE, put them in `CLAUDE.md`.
- **`AGENTS.md` is a fallback, not a second source.** It exists so a repository already committed to the cross-tool `AGENTS.md` standard works out of the box. The moment you add a `CLAUDE.md`, the `AGENTS.md` is ignored.
- **The two are never merged.** SCORPIOX CODE does not concatenate `CLAUDE.md` and `AGENTS.md`. There is exactly one project instructions file in the prompt. This is deliberate: merging would change what is injected when you add or remove either file, and a stable, single source is what keeps the prompt cache alive.

A few practical notes:

- **Only the working directory is checked.** SCORPIOX CODE does not walk up the directory tree looking for instructions in parent folders. Launch from the directory that contains the file you want loaded.
- **One file, one location.** There is no user-wide `CLAUDE.md` and no enterprise layer in this mechanism. Project instructions are per-repo, in the directory you start from.

---

## How the file gets into the model

When a project instructions file is found, SCORPIOX CODE reads its contents and wraps them in a `<system-reminder>` block. The block is intentionally structured so the model understands the *weight* of what it is reading:

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.

Contents of /path/to/your/repo/CLAUDE.md (project instructions, checked into the codebase):

[your file, verbatim]

      IMPORTANT: this context may or may not be relevant to your tasks.
You should not respond to this context unless it is highly relevant to your task.
</system-reminder>
```

Three details in that block matter:

- **`# claudeMd`** is a labeled context section, the same shape Claude Code uses. It tells the model this is project-instruction context, not a user turn.
- **"These instructions OVERRIDE any default behavior"** is the operative line. It is what makes the file a contract rather than a suggestion. Keep your instructions specific and imperative; that is what makes the model actually obey them.
- **The file is included verbatim, with its real path.** The model can see the full path of the file, which helps it reason about "what this repo's rules are" versus "what the user just asked."

### When the reminder appears

The reminder is generated once per session. It can be surfaced two ways depending on configuration:

- **In the system prompt** — the instructions are part of the standing context the model works with.
- **Prepended to user messages** — when the `user_system_reminder` option is enabled, the same block is prepended to user messages, matching how Claude Code delivers project instructions so behavior stays consistent across providers.

Either way, the *content* is identical across turns. What changes is only where that stable content is placed.

---

## Why "cache stability" is the whole design

Prompt caching is how LLM providers let you pay for a long context once and reuse it on every later turn. The cache only works when the beginning of your prompt is **byte-for-byte identical** from one call to the next. If the harness reshuffles the prompt, adds a timestamp to the top, or re-sorts a context list every turn, the cache prefix changes, the provider cannot reuse the cached prefix, and every turn is billed as if it were the first.

SCORPIOX CODE treats project instructions as one of the *stable* parts of the prompt, deliberately:

- **Generated once, then cached in memory.** The `<system-reminder>` block is built the first time it is needed and held for the rest of the session. Subsequent turns reuse the exact same bytes.
- **Never re-sorted or re-merged.** Because only one file is ever loaded (and it is never merged with a second file), there is no list to re-sort and no union to recompute. The block is the same whether it is turn 3 or turn 300.
- **Invalidate only when you change directories.** The cache is cleared when the working directory changes (for example after a `/cd`), so a new directory's instructions are picked up — and a new directory's prompt prefix is built exactly once, then cached again.

The practical payoff: a session that runs for hundreds of turns keeps a **99%+ prompt cache hit rate** on the project-instructions prefix, instead of the near-100% cache-miss you get from a harness that restructures its prompt every turn. You do not manage this. It is the default, and it is why long sessions do not quietly become expensive.

---

## Turning it off (and how it is configured)

Project instructions are on by default. You can disable the whole mechanism through configuration: the `include_claudemd` option (config key `PROMPT_INCLUDE_CLAUDEMD`) controls whether the working directory's instructions file is loaded at all. When it is off, SCORPIOX CODE emits a minimal, empty `<system-reminder>` placeholder instead, so the prompt shape stays stable even with instructions disabled.

A file that is empty, or larger than the internal limit, is also skipped — SCORPIOX CODE does not truncate a file silently, it simply does not inject one it cannot fit. If your instructions file is unexpectedly absent from the model's context, check that it is non-empty and within the size limit.

---

## A minimal CLAUDE.md you can paste

```markdown
# Project: example-service

## Build
- `make build` compiles. `make test` runs the suite. There is no npm.

## Conventions
- Go: gofmt everything before you finish.
- Never edit files in `gen/` — they are generated.
- Tests live next to the code, `_test.go`.

## Boundaries
- Do not change the public API in `pkg/api` without flagging it.
- CI plan is in `.github/workflows/ci.yml`.
```

Keep it to things that must be true in *every* session. If an instruction is a multi-step procedure or only applies to one area of the codebase, it belongs in a skill or a scoped rule, not here. (See the skills page for the on-demand mechanism.)

---

## How SCORPIOX CODE compares to other harnesses

The industry has converged on two ideas: a **project instructions file** (CLAUDE.md, AGENTS.md, .cursorrules, CONVENTIONS.md) and a **stable place to put it**. Where the harnesses differ is in *how many files*, *how they are combined*, and *how hard they work to keep the prompt cache alive*. Here is the honest side-by-side.

### Claude Code (CLAUDE.md)

Claude Code's native instructions file is `CLAUDE.md`. It is more layered than SCORPIOX CODE:

- **Multiple files, merged in load order.** Claude Code supports an enterprise-level file, a project-level file (in the repo, plus files in subdirectories walked up from where you launch), and a user-level file in `~/.claude/CLAUDE.md`. These are combined, so a session can carry several CLAUDE.md files at once.
- **Tree walking.** Claude Code walks up the directory tree, so a `CLAUDE.md` in a parent directory is loaded in addition to one in the working directory.
- **AGENTS.md is also supported.** Claude Code can read a repository's `AGENTS.md`, on its own or alongside `CLAUDE.md`.

SCORPIOX CODE is intentionally simpler here: **one directory, one file, no tree walking, no merging.** The trade-off is worth understanding. Claude Code's layered model is powerful for large organizations that want enterprise-wide policy plus per-repo rules. SCORPIOX CODE's flat model is easier to reason about and, more importantly, keeps the injected block byte-stable, which is what protects the cache. If you come from Claude Code and keep a `CLAUDE.md` in your repo root, it just works in SCORPIOX CODE.

### AGENTS.md (the cross-tool standard)

`AGENTS.md` is an open, vendor-neutral file proposed as a single predictable place for the context coding agents need — build steps, test commands, conventions — so it does not have to clutter a `README.md`. It is deliberately generic and has been adopted across a wide ecosystem (OpenAI Codex, Google's Jules, Aider, Cursor, VS Code, Devin, Windsurf, GitHub Copilot's coding agent, and many others).

SCORPIOX CODE supports `AGENTS.md` as its **fallback**, so a repository that standardizes on `AGENTS.md` — or a monorepo where a human contributor already maintains one — works with SCORPIOX CODE with zero changes. The rule is the same as everywhere: if you also drop a `CLAUDE.md` in, the `CLAUDE.md` wins and the `AGENTS.md` is not read.

The clean strategy: **if you want to be portable across many agents, keep one `AGENTS.md`; if you want to target SCORPIOX CODE (or Claude Code) specifically, a `CLAUDE.md` is fine and takes priority.**

### Cursor (.cursorrules / .cursor/rules)

Cursor uses a different shape entirely. Historically it read a single `.cursorrules` file in the project root. It has since moved to a folder-based model: rules live in `.cursor/rules/` as Markdown files (`.mdc`), each with frontmatter that can include **glob patterns** to scope a rule to matching files, and an `alwaysApply` flag for rules that should be present regardless of the open file.

The key difference from SCORPIOX CODE: **Cursor scopes rules per-file via globs, SCORPIOX CODE loads one whole file per session.** Cursor's model is better when you genuinely need different rules for different file types; SCORPIOX CODE's model is better when you want one stable, cache-friendly contract. If you have a `.cursorrules` file you like, its contents are a fine starting point for a `CLAUDE.md`.

### Aider (.aider.conf.yml / CONVENTIONS.md)

Aider's conventions file is `CONVENTIONS.md` in the project root, which Aider loads automatically when present. Aider also reads its settings from `.aider.conf.yml` (and `~/.aider.conf.yml` for user-wide settings) and can be pointed at other files with `--read`. Aider is part of the AGENTS.md-adjacent ecosystem and can load `AGENTS.md` / `CLAUDE.md` as read files.

Compared to SCORPIOX CODE, Aider's convention file is a single root file much like ours, but Aider does not advertise a cache-stability guarantee around it. SCORPIOX CODE's explicit one-time generation and in-memory caching is the difference: the same convention file, but with a hard guarantee that it costs you the same cache hit rate on turn 300 as on turn 1.

### The comparison at a glance

| | File(s) | Multiple files merged? | Tree walking? | Cache-stability guarantee |
|---|---------|------------------------|---------------|---------------------------|
| **SCORPIOX CODE** | `CLAUDE.md`, else `AGENTS.md` | No — one file, never merged | No — working directory only | Yes — generated once, cached in memory, 99%+ cache hits |
| **Claude Code** | `CLAUDE.md` (enterprise / project / user), `AGENTS.md` | Yes — layered and combined | Yes — walks up the tree | Implicit — layered context is stable while files are unchanged |
| **AGENTS.md standard** | `AGENTS.md` | Varies by adopting tool | Varies | Depends on the adopting harness |
| **Cursor** | `.cursorrules`, `.cursor/rules/*.mdc` | Yes — many rules, glob-scoped | No | Not emphasized |
| **Aider** | `CONVENTIONS.md`, `AGENTS.md`/`CLAUDE.md` via `--read` | Read files can be multiple | No | Not emphasized |

The through-line: SCORPIOX CODE does not try to be the most featureful context loader. It tries to be the most **predictable** one, because predictability is what keeps the prompt cache warm, and the prompt cache is what keeps long sessions affordable.

---

## Choosing your file, in one paragraph

Put your project instructions in a `CLAUDE.md` in your repo root if you want to be explicit and have it win unambiguously in SCORPIOX CODE. If your team already maintains an `AGENTS.md` for portability across many agents, keep it — SCORPIOX CODE reads it automatically when no `CLAUDE.md` is present, and a `CLAUDE.md` will simply take over if you add one later. Keep the file to facts that must hold in every session, write them imperative and specific, and you are done: SCORPIOX CODE loads it, wraps it in a stable override block, and keeps it out of the way of your prompt cache for the entire session.
