# Project Instructions in SCORPIOX CODE: CLAUDE.md and AGENTS.md

Every coding agent has to answer one question before it does anything useful: **what should this model know about *this* repo before it writes the first line?** Build commands, test layout, naming conventions, the one directory you must never touch, the lint rule that makes everyone wince. None of that lives in the model. It lives in your team's head, your `README`, your tribal knowledge. A project instructions file is where you pin that knowledge so the agent reads it at the start of every session, without you retyping it.

SCORPIOX CODE reads exactly two files for this purpose, in a strict precedence order, and folds the winner into the system prompt in a way that keeps the provider's prompt cache hot for the whole session. This page walks through how that works end to end, what to put in the file, and how the design compares to Claude Code, OpenAI's `AGENTS.md` standard, Cursor's project rules, and Aider's conventions.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** drop a `CLAUDE.md` (or, failing that, an `AGENTS.md`) into the directory you start SCORPIOX CODE in, and the agent reads it once per session, wraps it in a stable context block, and carries it into the system prompt. The model sees your instructions every turn, and because the block never changes during the session, the provider's prompt cache keeps hitting — typically well past 99% — instead of rebuilding the context from scratch every turn.

---

## The two files and the precedence rule

SCORPIOX CODE looks in your **current working directory** — the root of the repo you have open — for exactly two filenames, in this order:

| Priority | Filename | Meaning |
|----------|----------|---------|
| 1 (wins) | `CLAUDE.md` | The convention introduced by Anthropic's Claude Code. SCORPIOX CODE reads it first. |
| 2 (fallback) | `AGENTS.md` | The cross-vendor convention established by the OpenAI / SWE-bench ecosystem. Read only when `CLAUDE.md` is absent. |

The rule is deliberately strict:

- **`CLAUDE.md` always wins.** If both files exist, only `CLAUDE.md` is read. SCORPIOX CODE does not concatenate, merge, or interleave the two.
- **`AGENTS.md` is a fallback, not an alias.** It exists so a repo can ship a single instructions file that also works in the OpenAI Codex family and other tools that honour the cross-vendor standard. For SCORPIOX CODE to read it, `CLAUDE.md` must not be present.
- **One file per session.** The file that wins is the file that gets injected. There is no "read both, in order" mode.

This is a design decision, not an oversight. Two files telling the model slightly different things is the classic way a team ends up with an agent that obeys the *average* of two conventions instead of one. Picking one file and sticking with it keeps the contract unambiguous.

### Which file should you ship?

A practical rule of thumb:

- **If your team is mixed** — some engineers on Claude Code, some on OpenAI Codex, some on SCORPIOX CODE — ship **`AGENTS.md`** and do not create a `CLAUDE.md`. Every tool that respects the cross-vendor standard picks it up, and SCORPIOX CODE picks it up too.
- **If your team is Claude Code plus SCORPIOX CODE only**, ship **`CLAUDE.md`**. It is the file those two tools share by name, and it is what Claude Code's `/init` scaffolds for you.
- **Never ship both with different content.** If both exist, SCORPIOX CODE reads only `CLAUDE.md` while the OpenAI-side tools read `AGENTS.md`, and the divergence becomes a trap: the file one half of the team's tooling reads is not the file the other half reads, and the team silently disagrees.

---

## What the file is for

A project instructions file is a *standing directive* — a short, imperative document that tells the model how to behave inside this repo, every turn, without you repeating it. It is not a `README`, a changelog, or a tutorial. It is the set of rules a new engineer would wish someone had told them on day one.

Put in it:

- **Build and test commands.** The exact commands that must pass before a change is committed.
- **Project layout.** Where the important things live, and where they do not.
- **Conventions.** Style, naming, import rules, commit message shape — anything a linter does not enforce.
- **Boundaries.** Generated directories to leave alone, files that are off-limits, migrations that need a human.
- **Workflow rules.** Branching, review expectations, "always run X before Y."

Leave out of it:

- Long background prose. Every line is paid for on every turn, in cost and in the model's attention.
- Anything that changes often. The file is read once per session; churn means restarts.
- Secrets, credentials, or machine-specific paths.

Keep it short, imperative, and stable. A good instructions file is one screen to two screens long. If it grows past that, you are probably documenting the codebase rather than directing the agent.

---

## How the file reaches the model

There are two injection points, and they are controlled by two independent switches. Most users run the defaults.

### 1. The system prompt section (on by default)

When the file is found, its text is wrapped in a simple, labelled section and appended to the system prompt:

```
# Project Instructions (CLAUDE.md)

<your file contents, verbatim>
```

The heading names the file that won the precedence rule, so the model's own context tells you which file is in effect. Because the section is part of the system prompt, the model sees it on **every** turn without you repeating it, and it is treated as standing instructions rather than as one user message among many.

If the file is absent — or the feature is switched off — the section is simply not present. There is no placeholder and no "no instructions" text; the model proceeds with whatever base prompt is configured.

### 2. The per-turn system-reminder block (off by default)

On top of the system-prompt section, SCORPIOX CODE can attach the same content to the model's view of the conversation as a **`<system-reminder>`** block — the exact shape Claude Code uses. It reads, in effect:

```
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them
exactly as written.

Contents of /path/to/your/repo/CLAUDE.md (project instructions, checked into the codebase):

<your file contents, verbatim>

      IMPORTANT: this context may or may not be relevant to your tasks. You should not
      respond to this context unless it is highly relevant to your task.
</system-reminder>
```

The wording is doing real work. The block tells the model that these instructions **override default behaviour** and must be followed exactly, and it frames the content as project instructions *checked into the codebase* — so the model treats it as authoritative rather than as casual context. That is the same contract Claude Code establishes, which is why a `CLAUDE.md` written for one tool behaves the same in the other.

The reminder block is **built once per session and cached in memory**. Every subsequent turn reuses the exact same bytes. That single decision — *build once, reuse forever* — is what keeps the provider's prompt cache happy (see the next section). It is attached through the same message path for every supported provider, so the behaviour is consistent no matter which backend you run against.

The two switches are independent:

| Behaviour | Config key | Default |
|-----------|------------|---------|
| Include the winning instructions file in the system prompt | `PROMPT_INCLUDE_CLAUDEMD` | `true` |
| Also attach the cached `<system-reminder>` block to the conversation | `USER_SYSTEM_REMINDER` | `false` |

The reminder block is an extra insurance policy: if a long conversation starts to drift from your conventions, the reminder reasserts them in the conversation stream itself rather than only in the system prompt. If you want the leanest possible prompt, leave `USER_SYSTEM_REMINDER` off and rely on the system-prompt section alone.

---

## Why "build once, reuse forever" matters for cost and latency

Most agent harnesses treat the conversation as the unit of work: every turn, they reassemble the prompt from scratch — system prompt, tools, instructions, history, current message — and send it. If *any* byte of the prefix changes, the provider's prompt cache misses and the model has to re-read the entire prefix. In a long session with a large instructions file and a long tool catalogue, that means a full re-read on every turn that was supposed to be a cache hit.

SCORPIOX CODE is built around the opposite assumption: **the prefix is fixed for the life of the session, and only the tail — the current message and its tool results — changes.** Concretely:

- The instructions file is read once and the assembled blocks are held in memory.
- The system-prompt section and the `<system-reminder>` block are produced once and reused, byte-for-byte, on every subsequent turn.
- Even when there is no instructions file at all, a minimal, fixed block is still produced, so the shape of the prefix does not shift mid-session.
- The only thing that grows turn over turn is the message history — and that growth is exactly what the provider's cache is designed to absorb cheaply.

The practical effect is that the prompt-cache hit rate for a typical interactive session sits well past 99%: the prefix is identical on every turn, so each turn is a cache read of the prefix plus a small cache write for the new tail. That is the difference between paying for a full re-read of your context every turn (slow and expensive) and paying for it once and then a small delta on every turn after (fast and cheap).

This is also why the file is read from the **current working directory at session start**, not re-read on every turn. The contract is simple: the file you had when you started the session governs the session. If you edit the file mid-session, the change takes effect the next time you start a session. That is the same trade-off you make with any configuration file, and it is what makes the caching story work.

### The static-block design versus dynamic prompt rebuilding

A useful lens for comparing harnesses is *what changes between turns*:

| Design | What changes per turn | Cache behaviour |
|--------|----------------------|-----------------|
| **Static block, cached** (SCORPIOX CODE) | Only the new tail (current message + tool results) | Prefix re-used byte-for-byte; hit rate typically above 99% |
| **Dynamic prompt restructuring** | Instructions, tool list, or system prompt can be rebuilt per turn | Any changed byte invalidates the prefix; the model re-reads everything |

When a harness rebuilds or reorders the instruction content per turn — inserting timestamps, re-serialising the tool catalogue, re-deriving instructions from a live file read — it pays a cache miss on every turn where that content shifts. SCORPIOX CODE avoids that class of miss by construction: the instructions block is frozen at session start and never rewritten. The one deliberate exception is invalidating the cache when the session's context genuinely changes (for example, a new session is started), at which point the whole block is rebuilt once and then frozen again.

---

## A session in motion

To make the mechanics concrete, here is what happens from "you open a repo" to "the model obeys your conventions," in the default configuration:

1. **You start a session in your repo.** SCORPIOX CODE looks in the working directory for `CLAUDE.md`. It finds it. The `AGENTS.md` question is not asked — `CLAUDE.md` is present, so it wins.
2. **The file is read once** and wrapped in a `# Project Instructions (CLAUDE.md)` section appended to the system prompt.
3. **The `<system-reminder>` block is built** from the same content and cached in memory. (Only when `USER_SYSTEM_REMINDER` is on; off by default.)
4. **You send your first message.** The prompt the provider sees is: base system prompt + instructions section + message history (empty) + your message. The provider caches the prefix.
5. **You send your second, third, hundredth message.** The prefix — including the instructions section and the reminder — is byte-identical to the previous turn. The provider serves it from cache and only processes the new tail. Cache hit.
6. **You edit `CLAUDE.md` mid-session.** The change does not take effect this session. The next session reads the new file. This is deliberate: it is what keeps the prefix stable.
7. **You add a new file to the repo.** The model can read it; the instructions file is unaffected. If the new file is in a directory the instructions reference, the model follows the rule.

The steady state is: **your instructions are in the prompt, the cache is hot, and the model obeys the same rules on turn 1 and turn 200.** That is the whole design in one paragraph.

---

## Configuration keys

Both switches live in the standard configuration cascade — `scorpiox-env.txt` at any tier, a named profile, or an OS environment variable. See [Configuration and Profiles](scorpiox-env.md) for the full cascade.

| Key | Type | Default | Effect |
|-----|------|---------|--------|
| `PROMPT_INCLUDE_CLAUDEMD` | bool | `true` | Include the winning instructions file in the system prompt. `false` disables it entirely. |
| `USER_SYSTEM_REMINDER` | bool | `false` | Also attach the cached `<system-reminder>` block to the conversation view. `true` reasserts the instructions in the conversation stream, not just the system prompt. |

The defaults are the recommended configuration. Turn `PROMPT_INCLUDE_CLAUDEMD` off only if you are running a custom setup that supplies its own instructions channel; turning it on is the whole point of the feature. Turn `USER_SYSTEM_REMINDER` on if you notice long sessions drifting from your conventions and want the extra reassertion; leave it off if you want the leanest prompt.

Because the file is read through the same configuration machinery as everything else, you can stage different instructions per project or per profile with no code changes — for example, a profile that points sessions at a stricter working directory, or an environment variable that toggles the feature in CI.

---

## How this compares to other harnesses

Every serious coding agent has a story for "how does the model learn about *this* repo." The stories differ in file name, precedence, and how the content is injected. Here is the landscape.

### Claude Code (Anthropic)

Claude Code's convention is **`CLAUDE.md`**, introduced in 2024 and now the de-facto reference for "the file a coding agent reads to learn about a project." The file can live at the repo root, in a `.claude/` directory, or in the directory hierarchy above the working directory, with more specific files taking precedence for the subtree they cover. Claude Code also reads `AGENTS.md` in place of `CLAUDE.md` when no `CLAUDE.md` exists, and wraps the content in a `<system-reminder>` block on each turn — the exact shape SCORPIOX CODE's reminder block follows.

Claude Code's `/init` command scaffolds a starter `CLAUDE.md` by analysing the codebase — build commands, test layout, conventions it can infer. That is a good on-ramp, but the file is still a standing document you own: `/init` writes the first draft, you maintain the rest.

SCORPIOX CODE's model is deliberately compatible: a `CLAUDE.md` you wrote for Claude Code is a `CLAUDE.md` you can drop into a repo and use with SCORPIOX CODE, unchanged.

### OpenAI / SWE-bench (`AGENTS.md`)

**`AGENTS.md`** began as a cross-vendor convention in the OpenAI / SWE-bench ecosystem — "a README for agents" — and is now the file the OpenAI Codex family of tools reads by default, with broad adoption and stewards drawn from multiple vendors. The format is plain markdown, and the recommended content is the same kind of standing instruction as a `CLAUDE.md`: build commands, test layout, conventions, boundaries.

The convention's selling point is portability: one file, many tools. SCORPIOX CODE honours that directly — if you have an `AGENTS.md` and no `CLAUDE.md`, SCORPIOX CODE reads it and injects it exactly as it would a `CLAUDE.md`, and the heading in the system prompt reflects the filename that won. A repo that ships `AGENTS.md` therefore works in SCORPIOX CODE, OpenAI Codex, and every other tool that honours the standard, with one file.

### Cursor (Anysphere)

Cursor's project rules are a different shape. The modern mechanism is **`.cursor/rules/`** — a directory of `.mdc` files, each with YAML frontmatter that can scope the rule to file globs, invocation triggers, or always-on. The older mechanism, a single root-level **`.cursorrules`** file, is still supported and read as an always-on rule.

The scoping model is the differentiator. A `.cursorrules` file is "always applies to everything in this repo," which is close to SCORPIOX CODE's model. A `.cursor/rules/*.mdc` file can be narrower — "only applies to `src/api/**`" or "only applies when I invoke it" — a capability SCORPIOX CODE's single-file model does not aim to match. If you are coming from Cursor, put your always-on project rules into `CLAUDE.md` or `AGENTS.md`; if you have rules genuinely scoped to a subtree, either inline them into that subtree's documentation or keep them in Cursor and accept that SCORPIOX CODE will not see them. There is no automatic import.

### Aider

Aider's convention is **`CONVENTIONS.md`**, loaded through the `read` / `read-only` keys in **`.aider.conf.yml`** (or with `--read CONVENTIONS.md` on the command line). The content shape is the same as `CLAUDE.md` and `AGENTS.md`: build commands, test layout, naming conventions, the one rule you wish someone had told you on day one. Because it is loaded as a read-only file, Aider can cache it across turns if prompt caching is enabled — conceptually the same "pin it once, reuse it" idea SCORPIOX CODE applies to the whole instructions block.

### At a glance

| Harness | Primary file | Fallback / variants | Injection | Cache stability |
|---------|-------------|---------------------|-----------|-----------------|
| **SCORPIOX CODE** | `CLAUDE.md` | `AGENTS.md` (only if no `CLAUDE.md`) | System-prompt section + optional cached `<system-reminder>` | Static, built once per session |
| Claude Code | `CLAUDE.md` | `AGENTS.md` when no `CLAUDE.md`; managed, user, and local scopes | System prompt + `<system-reminder>` each turn | Stable per session, with hierarchical merge |
| OpenAI / SWE-bench | `AGENTS.md` | — | Read as project instructions | Per-tool |
| Cursor | `.cursor/rules/*.mdc` | `.cursorrules` (legacy) | Prepended to model context, glob- or trigger-scoped | Per-rule, can be scoped |
| Aider | `CONVENTIONS.md` | any file via `read:` | Loaded as a read-only context file | Cacheable when enabled |

Read that table the way an operator would. Every harness here can carry standing instructions; the difference is *how much can change between turns*. SCORPIOX CODE's contribution is the strict static block: pick one file, wrap it once, freeze it, and let the cache do the rest.

---

## Gotchas worth knowing

- **Two files, one wins.** If you ship both `CLAUDE.md` and `AGENTS.md` with different content, SCORPIOX CODE reads `CLAUDE.md` and the OpenAI-side tools read `AGENTS.md`. The team silently disagrees. Pick one name.
- **The file is read at session start, not per turn.** Editing the file mid-session does not change the current session. Start a new session — or compact, see [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md) — to pick up the change.
- **It is read from the working directory, not from the repo root by path.** If you `cd` into a subdirectory and start a session there, SCORPIOX CODE looks for the file *in that subdirectory*. If your instructions live at the repo root and you start from a subdirectory, the file is not found. Start sessions from the root, or place the file where you start.
- **There is a size ceiling.** Very large instructions files are skipped rather than silently truncated into nonsense; keep the file well inside the limit so it never falls over the edge.
- **The file is not a `README`.** Long background prose is paid for on every turn, in context and in the model's attention. Keep it short, imperative, and stable.
- **Generated files are not self-documenting.** If a directory is generated, say so in the file ("do not touch `src/generated/**`") rather than expecting the model to infer it. The model obeys what you write, not what is obvious.
- **Turning the feature off removes the section entirely.** With `PROMPT_INCLUDE_CLAUDEMD=0`, neither the system-prompt section nor the reminder content is present; a minimal, fixed block remains so the prefix shape stays stable.

---

## The bottom line

Project instructions in SCORPIOX CODE come down to three commitments:

1. **One file, one truth.** `CLAUDE.md` wins; `AGENTS.md` is the cross-tool fallback. Never merged, so there is never an ambiguity about what the model was told.
2. **A clear contract.** The instructions are injected as project context that explicitly overrides default behaviour, in the same `<system-reminder>` shape Claude Code uses — so a file authored for one tool works in the other.
3. **A hot cache.** The block is built once per session and reused byte-for-byte, so the provider's prompt cache stays well above 99% and long sessions stay fast and cheap.

Write a short, imperative file, drop it in the directory you start from, and SCORPIOX CODE will carry it into every turn of the session — without you repeating a word.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — the full cascade where `PROMPT_INCLUDE_CLAUDEMD` and `USER_SYSTEM_REMINDER` live.
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md) — how long sessions are compressed, and what survives.
- [Skills in SCORPIOX CODE](skills-system.md) — the on-demand counterpart: procedures loaded only when a task calls for them.
- [Native MCP 2.0 and OAuth 2.1](mcp.md) — a different standing surface: external tools instead of project instructions.
- [Data Privacy and Zero Data Collection](privacy.md) — what stays local, and why your instructions file never leaves your machine except to the model you chose.
