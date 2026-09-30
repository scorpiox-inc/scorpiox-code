# Project Instructions in SCORPIOX CODE: CLAUDE.md and AGENTS.md

Every serious coding agent has a small, sharp problem it has to answer before it does anything else: **what should this model know about *this* repo before it writes the first line?** Build commands, test layout, naming conventions, the one file you should never touch, the lint rule that makes everyone wince — none of that lives in the model. It lives in your team's head, your `README`, your tribal knowledge. A project instructions file is the place you pin that knowledge where the agent can read it every session, without you retyping it.

SCORPIOX CODE reads exactly two files for this purpose, in a strict precedence order, and injects the winner into the model's system prompt in a way that keeps the prompt cache hot across an entire session. This page walks through how that works end to end, what to put in the file, and how the design compares to Claude Code, OpenAI's AGENTS.md standard, Cursor's project rules, and Aider's conventions.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** drop a `CLAUDE.md` (or, failing that, `AGENTS.md`) into the root of your repo, and SCORPIOX CODE reads it once per session, wraps it in a stable context block, and folds it into the system prompt. The model sees your instructions every turn, and because the block never changes during the session, the provider's prompt cache keeps hitting — usually well past 99% — instead of rebuilding the context from scratch every turn.

---

## The two files and the precedence rule

SCORPIOX CODE looks in your **current working directory** — the root of the repo you have open — for exactly two filenames, in this order:

| Priority | Filename | Meaning |
|----------|----------|---------|
| 1 (wins) | `CLAUDE.md` | The original convention, introduced by Anthropic's Claude Code. SCORPIOX CODE reads it first. |
| 2 (fallback) | `AGENTS.md` | The cross-vendor convention popularised by the OpenAI / SWE-bench ecosystem. Read only if `CLAUDE.md` is absent. |

The rule is deliberately strict:

- **`CLAUDE.md` always wins.** If both files exist, only `CLAUDE.md` is read. SCORPIOX CODE does not concatenate, merge, or interleave the two.
- **`AGENTS.md` is a fallback, not an alias.** It exists so a repo can ship one instructions file that also works in the OpenAI / SWE-bench / Codex family of tools. If you want SCORPIOX CODE to read it, make sure `CLAUDE.md` is not also present.
- **One file per session.** The file that wins is the file that gets injected. There is no "both, in order" mode.

This is a design decision, not an oversight. Two files telling the model slightly different things is the classic way a team ends up with an agent that obeys the *average* of two conventions instead of one. Picking one file and sticking with it keeps the contract unambiguous.

### Which file should you ship?

A practical rule of thumb:

- **If your team is mixed** (some engineers on Claude Code, some on OpenAI Codex, some on SCORPIOX CODE), ship **`AGENTS.md`** and do not create a `CLAUDE.md`. Every tool that respects the cross-vendor standard will pick it up, and SCORPIOX CODE will pick it up too.
- **If your team is Claude Code + SCORPIOX CODE only**, ship **`CLAUDE.md`**. It is the file those two tools share by name, and it is what Claude Code's `/init` scaffolds for you.
- **Never ship both with different content.** If both exist, SCORPIOX CODE reads only `CLAUDE.md`, and the divergence becomes a trap: the file the OpenAI-side tools read is not the file the Claude-side tools read, and the team silently disagrees.

---

## What the file is for

An instructions file is a *standing directive* — a short, imperative document that tells the model how to behave inside this repo, every turn, without you repeating it. It is not a `README`, a changelog, or a tutorial. It is the set of rules a new engineer would wish someone had told them on day one.

A good instructions file is short, specific, and stable. The model reads it on every turn, so every extra paragraph is paid for in context and in the model's attention budget. A few hundred words, written in the imperative, is usually the right size.

### A minimal, pasteable example

A `CLAUDE.md` at the root of a small TypeScript project might look like this:

```markdown
# Project Instructions

## Build and test
- Build: `pnpm build` (must pass before any commit)
- Tests: `pnpm test` — run the full suite, not a single file, before declaring work done
- Type-check: `pnpm typecheck`

## Conventions
- TypeScript strict mode is on. Do not add `any` or `@ts-ignore`.
- Use named exports only; no default exports.
- All new files need a `// @generated` or `// hand-written` header.
- Do not touch `src/generated/**` — it is produced by `pnpm codegen`.

## Repo layout
- `src/api/` — HTTP handlers, one file per resource
- `src/db/` — query builders, no direct SQL strings
- `tests/` — mirrors `src/`, same relative paths
- `docs/` — architecture notes; update the one matching the area you touch

## Do not
- Do not add new top-level dependencies without calling it out in the PR description.
- Do not rename files in `src/api/`; downstream clients import them by path.
```

An `AGENTS.md` for the same repo would be the same content under a different filename. Pick one name and commit to it.

### What belongs in the file

- **Build, test, lint, format commands** — the exact invocations, not paraphrases. The model will run whatever you write verbatim.
- **Conventions** — naming, file layout, export style, error handling patterns.
- **Boundaries** — "do not touch X", "generated code is here", "this directory is experimental".
- **The one thing that is hard to discover** — a non-obvious port, a credentials file location, an environment variable that must be set.

### What does not belong in the file

- **Long background on the domain.** The model can read the code; the file is for *rules*, not *lore*.
- **Secrets or tokens.** If a rule needs a secret, reference the environment variable or config file; do not paste the value.
- **Ephemeral state.** "We are mid-migration, ignore the old endpoints until Friday" ages badly and will be wrong the day after. Put transient notes in a PR description or a `TODO.md`, not in a file the agent reads every session.

---

## How SCORPIOX CODE injects the file

There are two independent switches that control how your instructions file reaches the model. Both are on by default in the configuration that ships, and both can be turned off.

### 1. The system-prompt block (always on by default)

When a session starts, SCORPIOX CODE reads the winning file from your working directory and appends it to the **system prompt** as a clearly-labelled section:

```
# Project Instructions (CLAUDE.md)

# Project Instructions

## Build and test
- Build: `pnpm build` (must pass before any commit)
...
```

The section heading names the file that won the precedence rule, so if you ever wonder which file is in effect, the model's own context tells you. Because the file is part of the system prompt, the model sees it on **every** turn without you repeating it, and it is treated as part of the model's standing instructions rather than as one user message among many.

If the file is absent — or the feature is switched off — this section is simply not present. There is no placeholder, no empty block, no "no instructions" text. The model proceeds with whatever base prompt is configured.

### 2. The per-turn system-reminder (optional, off by default)

On top of the system-prompt block, SCORPIOX CODE can also wrap the instructions file in a **`<system-reminder>` block** and attach it to the model's view of the conversation. This is the same shape Claude Code uses: a small, stable block that says, in effect, "these are the project instructions, they override default behaviour, follow them exactly."

The reminder block is **built once per session and cached in memory**. Every subsequent turn reuses the exact same bytes. That single decision — *build once, reuse forever* — is what keeps the provider's prompt cache happy (see the next section).

The two switches are independent:

| Behaviour | Config key | Default |
|-----------|------------|---------|
| Include the instructions file in the system prompt | `PROMPT_INCLUDE_CLAUDEMD` | `true` |
| Also attach the cached `<system-reminder>` block per turn | `USER_SYSTEM_REMINDER` | `false` |

Most users run with both at their defaults. The reminder block is the extra insurance policy: if a long conversation starts to drift from your conventions, the reminder reasserts them in the conversation stream itself rather than only in the system prompt. If you want the leanest possible prompt, turn `USER_SYSTEM_REMINDER` off and rely on the system-prompt block alone.

---

## Why "build once, reuse forever" matters for cost and latency

Most agent harnesses treat the conversation as the unit of work: every turn, they reassemble the prompt from scratch — system prompt, tools, instructions, history, current message — and send it. If *any* byte of the prefix changes, the provider's prompt cache misses and the model has to re-read the entire prefix. In a long session with a large instructions file and a long tool catalogue, that means a full re-read on every turn that is supposed to be a cache hit.

SCORPIOX CODE is built around the opposite assumption: **the prefix is fixed for the life of the session, and only the tail (the current user message and its tool results) changes**. Concretely:

- The instructions file is read once and its text is cached in memory.
- The system-prompt block and the `<system-reminder>` block are produced once and reused, byte-for-byte, on every subsequent turn.
- The only thing that grows turn over turn is the message history, and that growth is exactly what the provider's cache is designed to absorb cheaply.

The practical effect is that the prompt-cache hit rate for a typical interactive session sits well past 99% — the prefix is identical on every turn, so every turn is a cache read of the prefix plus a small cache write for the new tail. That is the difference between paying for a full re-read of your context every turn (slow and expensive) and paying for it once and then a small delta on every turn after (fast and cheap).

This is also why the file is read from the **current working directory at session start**, not re-read on every turn. The contract is: the file you had when you started the session is the file that governs the session. If you edit the file mid-session, the change takes effect the next time you start a session. That is the same trade-off you make with any configuration file, and it is what makes the caching story work.

If you do need to change instructions mid-session, the clean move is to end the current session (or compact it — see [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)) and start a new one with the updated file in place.

---

## How this compares to other harnesses

Every serious coding agent has a story for "how does the model learn about *this* repo." The stories differ in file name, precedence, and how the content is injected. Here is the landscape.

### Claude Code (Anthropic)

Claude Code's convention is **`CLAUDE.md`**, introduced in 2024 and now the de-facto reference for "the file a coding agent reads to learn about a project." The file lives at the repo root (and optionally in subdirectories, with the nearest file winning for files in that subtree). Claude Code reads it into the system prompt and also wraps it in a `<system-reminder>` block on each turn, which is the exact shape SCORPIOX CODE's reminder block follows.

Claude Code's `/init` command scaffolds a starter `CLAUDE.md` by analysing the codebase — build commands, test layout, conventions it can infer. That is a good on-ramp, but the file is still a standing document you own: `/init` writes the first draft, you maintain the rest.

SCORPIOX CODE's model is deliberately compatible: a `CLAUDE.md` you wrote for Claude Code is a `CLAUDE.md` you can drop into a repo and use with SCORPIOX CODE, no changes. The precedence rule (`CLAUDE.md` wins, `AGENTS.md` is the fallback) means a repo can serve both tools with one file.

### OpenAI / SWE-bench (`AGENTS.md`)

**`AGENTS.md`** started as a cross-vendor convention in the OpenAI / SWE-bench ecosystem — "a README for agents" — and is now the file the OpenAI Codex family of tools reads by default. The format is plain markdown, and the recommended content is the same kind of standing instructions as a `CLAUDE.md`: build commands, test layout, conventions, boundaries.

The convention's selling point is portability: one file, many tools. SCORPIOX CODE honours that directly — if you have an `AGENTS.md` and no `CLAUDE.md`, SCORPIOX CODE reads it and injects it exactly as it would a `CLAUDE.md`. The heading in the system prompt reflects the filename that won, so the model (and anyone reading the session record) can tell which file is in effect.

The practical implication: **a repo that ships `AGENTS.md` works in SCORPIOX CODE, OpenAI Codex, and any other tool that honours the standard, with one file.** That is the cheapest way to get cross-tool portability, and it is the recommended path for teams that are not single-vendor.

### Cursor (Anysphere)

Cursor's project rules are a different shape. The modern mechanism is **`.cursor/rules/`** — a directory of `.mdc` files, each with YAML frontmatter that can scope the rule to file globs, invocation triggers, or always-on. The older mechanism, a single root-level **`.cursorrules`** file, is still supported and is read as an always-on rule.

The scoping model is the differentiator. A `.cursorrules` file is "always applies to everything in this repo," which is close to SCORPIOX CODE's model. A `.cursor/rules/*.mdc` file can be narrower — "only applies to `src/api/**`" or "only applies when I invoke it with `@rule`" — which is a capability SCORPIOX CODE's single-file model does not aim to match.

If you are coming from Cursor, the translation is: put your always-on project rules (build commands, conventions, boundaries) into `CLAUDE.md` or `AGENTS.md`. If you have Cursor rules that are genuinely scoped to a subtree, either inline them into the relevant subtree's documentation or keep them in Cursor and accept that SCORPIOX CODE will not see them. There is no automatic import, and there is no scoping syntax in the instructions file.

### Aider

Aider's convention is **`CONVENTIONS.md`**, loaded via the `read:` (or `read-only:`) key in **`.aider.conf.yml`**. The file is plain markdown, same content shape as `CLAUDE.md` / `AGENTS.md`: build commands, test layout, naming conventions, the one rule you wish someone had told you on day one.

The difference is the loader. Aider's `CONVENTIONS.md` is *opt-in through the config file* — the file does nothing unless `.aider.conf.yml` says to read it. SCORPIOX CODE's model is *opt-out by convention* — the file is read if it is present in the working directory, no config needed. Both are reasonable; the Aider model is more explicit, the SCORPIOX CODE model is more friction-free.

If you have an existing `CONVENTIONS.md`, the port is a copy: rename it to `CLAUDE.md` (or `AGENTS.md`) and it works. The content format is identical.

### The comparison, in one table

| Harness | File(s) | Precedence | Injection | Scoped rules? |
|---------|---------|-----------|-----------|---------------|
| **SCORPIOX CODE** | `CLAUDE.md`, then `AGENTS.md` | `CLAUDE.md` wins, never merged | System-prompt block + optional cached `<system-reminder>` | No (one file, whole repo) |
| **Claude Code** | `CLAUDE.md` (root + per-subtree) | Nearest file wins in subtree | System prompt + per-turn `<system-reminder>` | Yes, via subtree files |
| **OpenAI / SWE-bench** | `AGENTS.md` | Nearest file wins in subtree | System prompt (tool-specific) | Yes, via subtree files |
| **Cursor** | `.cursorrules` (legacy), `.cursor/rules/*.mdc` (modern) | `.mdc` files with frontmatter scope | Injected per rule, per scope | Yes, via `.mdc` frontmatter |
| **Aider** | `CONVENTIONS.md` (loaded via `.aider.conf.yml`) | Explicit in config file | Loaded into context per config | No (whole repo) |

The pattern: **SCORPIOX CODE sits on the "single file, whole repo" side of the line, with a precedence rule that makes it compatible with both the Claude Code and OpenAI / SWE-bench conventions.** If you are already maintaining one of those files for another tool, SCORPIOX CODE reads it as-is. If you are starting fresh, pick `CLAUDE.md` or `AGENTS.md` based on which tools your team uses, and you are done.

---

## A concrete example: the full session lifecycle

To make the mechanics concrete, here is what happens from "you open a repo" to "the model obeys your conventions," in the default configuration:

1. **You start a session in your repo.** SCORPIOX CODE looks in the working directory for `CLAUDE.md`. It finds it. The `AGENTS.md` question is not asked — `CLAUDE.md` is present, so it wins.
2. **The file is read once** and its content is wrapped in a `# Project Instructions (CLAUDE.md)` section. That section is appended to the system prompt.
3. **The `<system-reminder>` block is built** from the same content, cached in memory. (Only if `USER_SYSTEM_REMINDER` is on; off by default.)
4. **You send your first message.** The prompt the provider sees is: base system prompt + `# Project Instructions (CLAUDE.md)` section + message history (empty) + your message. The provider caches the prefix.
5. **You send your second, third, hundredth message.** The prefix — including the instructions block and the reminder — is byte-identical to the previous turn. The provider serves it from cache and only processes the new tail. Cache hit.
6. **You edit `CLAUDE.md` mid-session.** The change does not take effect this session. The next session reads the new file. This is deliberate: it is what keeps the prefix stable.
7. **You add a new file to the repo.** The model can read it; the instructions file is not affected. If the new file is in a directory the instructions file references, the model follows the rule.

The steady state is: **your instructions are in the prompt, the cache is hot, and the model obeys the same rules on turn 1 and turn 200.** That is the whole design in one paragraph.

---

## Configuration keys

Both switches live in the standard configuration cascade — `scorpiox-env.txt` at any tier, a named profile, or an OS environment variable. See [Configuration and Profiles](scorpiox-env.md) for the full cascade.

| Key | Type | Default | Effect |
|-----|------|---------|--------|
| `PROMPT_INCLUDE_CLAUDEMD` | bool | `true` | Include the winning instructions file in the system prompt. `false` disables the section entirely. |
| `USER_SYSTEM_REMINDER` | bool | `false` | Also attach the cached `<system-reminder>` block to the model's view of the conversation. `true` reasserts the instructions in the conversation stream, not just the system prompt. |

The defaults are the recommended configuration. Turn `PROMPT_INCLUDE_CLAUDEMD` off only if you are running a custom harness that supplies its own instructions channel; turning it on is the whole point of the feature. Turn `USER_SYSTEM_REMINDER` on if you notice long sessions drifting from your conventions and want the extra reassertion; leave it off if you want the leanest prompt.

---

## Gotchas

- **Two files, one wins.** If you ship both `CLAUDE.md` and `AGENTS.md` with different content, SCORPIOX CODE reads `CLAUDE.md` and the OpenAI-side tools read `AGENTS.md`. The team silently disagrees. Pick one name.
- **The file is read at session start, not per turn.** Editing the file mid-session does not change the current session. Start a new session (or compact — see [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)) to pick up changes.
- **The file is read from the working directory, not from the repo root by path.** If you `cd` into a subdirectory and start a session there, SCORPIOX CODE looks for the file *in that subdirectory*. If your instructions live at the repo root and you start a session from a subdirectory, the file is not found. Start sessions from the root, or put a copy of the file where you start.
- **The file is not a `README`.** Long background prose is paid for on every turn in context and in the model's attention. Keep it short, imperative, and stable.
- **Generated files are not instructions.** If a directory is generated, say so in the file ("do not touch `src/generated/**`") rather than trying to make the model infer it. The model obeys what you write, not what is obvious.

---

## The bottom line

A project instructions file is the cheapest, most reliable way to make an agent behave like a member of your team instead of a very fast stranger. SCORPIOX CODE reads `CLAUDE.md` first, `AGENTS.md` second, never merges them, and injects the winner into the system prompt in a shape that keeps the provider's prompt cache hot across an entire session. The file is read once, cached, and reasserted on every turn; the prefix never changes, so the cache never misses on it.

Pick one filename based on which tools your team uses, write a few hundred words of standing instructions, and the model obeys them from the first turn to the last. That is the whole contract.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Deterministic File Editing & Developer Autonomy](file-editing.md)
