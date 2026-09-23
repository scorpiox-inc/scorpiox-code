# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

A **skill** in SCORPIOX CODE is a folder with a `SKILL.md` file that teaches the agent how to do one specific job: a checklist, a multi-step procedure, a convention you keep re-explaining, or a reference you keep pasting into chat. Drop the folder in the right place and it is available. Nothing else to wire up.

The core idea is that a skill is **on-demand by default**. The agent's context only ever carries the skill's *name and description* — a one-line index. The full instructions are pulled in **only when the skill is actually used**. A fifty-page reference doc costs almost nothing until the moment it is needed. That is the design decision everything else on this page follows from, and it is the one the other harnesses make differently.

Source of truth: `sx_slashcmd.c`, `sx_tools.c`, `sx_builtin_skills.c`, `scorpiox-skills.c`, `scorpiox-skills-info.c`, `sx_agent.c`, and `sx_systemprompt.c` at commit `6c70ad6`.

---

## What a skill is

A skill is a directory whose name is the skill name, containing a `SKILL.md`. The file has YAML frontmatter followed by a markdown body:

```
.claude/skills/
└── git-release/
    └── SKILL.md
```

```markdown
---
name: git-release
description: Create consistent releases and changelogs from merged PRs.
argument-hint: [version]
---

## What I do
- Draft release notes from merged PRs
- Propose a version bump
- Provide a copy-pasteable `gh release create` command

## When to use me
Use this when preparing a tagged release.
```

Three frontmatter fields matter:

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | No* | The skill identifier (kebab-case). The **folder name** is what SCORPIOX CODE uses as the skill name. |
| `description` | Yes | One line on *when* to use the skill. This is what the agent sees in the index. A skill with no description is skipped. |
| `argument-hint` | No | A hint for what `$ARGUMENTS` will contain when the skill is invoked. |

\* SCORPIOX CODE takes the name from the folder, so the `name` field is informational here. Matching them keeps the skill portable to other harnesses that do read it.

The body is plain markdown the agent reads and follows when the skill is loaded. If the body references `$ARGUMENTS`, that token is replaced with whatever you passed after the skill name. A skill with no arguments simply has no `$ARGUMENTS` token and nothing changes.

Two shapes coexist under the same name: a **skill** is a folder with `SKILL.md`, and a **command** is a script in `.claude/commands/` (`.sh` / `.ps1` / `.md`). Both surface as `/<name>`. A skill whose body is a script (starts with a `#!` shebang) is *executed* when invoked rather than just read. The rest of this page is about skills.

---

## Where skills live

SCORPIOX CODE scans a small, fixed set of locations and unions them:

| Location | Scope |
|----------|-------|
| `~/.claude/skills/` | User-global — applies everywhere on this machine. |
| `./.claude/skills/` | Project-local — checked in with the repo. |
| `ADDITIONAL_SKILLS` paths | Comma-separated list of extra directories from your config. |
| Built-ins | A handful shipped inside the binary (e.g. `create-skill`, `edit-skill`). |

When the same skill name appears in more than one place, **project-local wins over user-global, which wins over built-in**. That means a repo can override a personal skill without renaming anything, and a user skill can override a built-in. There is no registration step and no manifest to keep in sync — the folders themselves are the registry.

`/skills` lists what is currently loaded and where each skill came from, and `/skills-update` git-pulls any additional skill repositories you have configured.

---

## The on-demand flow

This is the part worth understanding, because it is what "on-demand" actually means at runtime.

1. **Index in context.** When the agent loop starts, SCORPIOX CODE builds a compact list of skill *names and descriptions* and folds it into the description of the built-in **`InvokeSkill`** tool. No bodies. This list is what the agent reasons over to decide whether a skill applies.
2. **Invoke on demand.** When the agent decides a skill fits, it calls **`InvokeSkill`** with the skill name (and optional arguments). SCORPIOX CODE reads the skill, substitutes `$ARGUMENTS`, and returns the body into the conversation for the agent to follow. A skill with a script body is run instead, and its output comes back.
3. **Discover when unsure.** The built-in **`SearchSkills`** tool greps across skill names, descriptions, and file content, so the agent can find a skill by keyword when it does not already know the name. Passing an empty pattern lists every available skill.

There is also a disk fallback: if the agent names a skill that is on disk but was not in the cached index (for example, a skill created mid-session), SCORPIOX CODE reads it straight from the skills directories rather than failing.

---

## Three ways a skill gets into context

On-demand invocation is the default, but two operators keep specific skills standing. Both are driven by plain text files, so they are diffable in git and trivially auditable.

**1. On-demand (default).** The agent loads the skill when it thinks it applies. Nothing is in context until then.

**2. Required skills.** A `required_skills.txt` file names skills that *must* be in play for a task. SCORPIOX CODE tracks, per session, which required skills have actually been loaded. If a required skill has not been loaded by the end of a turn, it nudges the agent to load it — and the nudge is capped (three attempts), so it cannot loop. Two settings change the strength:

- **Nudge (default)** — the agent is reminded to invoke the missing skill itself.
- **Force-inject** — the missing skill's body is written into the conversation directly, skipping the invoke round-trip.

The cascade runs from user level down through the project and then to the session, so a session can pin a skill without editing a shared file, and a repo can pin one for everyone.

**3. System-prompt skills.** A config value lists extra skill names whose raw content is always folded into the system prompt. These union with the required-skills list when system-prompt injection is on. Skills injected this way count as loaded, so they are not nudged. This is for the handful of skills that are load-bearing for the whole project — the conventions you never want the agent to discover late.

The practical split: on-demand for the long tail, required-skills for "must be here for this kind of work," system-prompt for "must be here always."

---

## Keeping the index small

A project that has accumulated two hundred skills will not fit two hundred name-and-description lines into every prompt without crowding out the actual work. SCORPIOX CODE gives you a filter that controls *which skills are listed* in the `InvokeSkill` index:

- **Include / exclude patterns** — keep or drop skills by name pattern, inline or from a file (e.g. list only `*-pipeline-*`).
- **Top-percentage by usage** — list the most-used N% of skills, ranked by how often they show up in past sessions' required-skills files. If there is no usage data yet, it fails open and lists everything.
- **List style** — show name and description (`full`, the default) or names only (`names`) to make the index even leaner.
- **Required skills are always listed** — a filtered list never hides a skill that is required.

The important invariant: **filtering only changes what is listed, never what is available.** A skill that is not in the index is still invokable by name and still findable with `SearchSkills`. When the list is partial, `SearchSkills` is enabled automatically so the hidden skills remain discoverable. You are trimming the index to save context, not pruning the capability set.

---

## How this compares to other harnesses

All of these harnesses converged on the same file format — a folder with a `SKILL.md` carrying `name` and `description` — because it is the [open Agent Skills standard](https://agentskills.io). The differences are in *what is in context by default*, *who decides when the body loads*, and *what happens when a skill should have been loaded but was not*.

### Claude Code

Claude Code is the closest in philosophy and is, in practice, where the standard originated. It uses the same progressive disclosure: a skill's body loads only when it is used, and its name-and-description index is what the agent sees. It merged custom commands into skills, adds frontmatter to control who invokes a skill (you, the model, or both), runs some skills in a subagent, and supports dynamic context injection. Its bundled skills (`/code-review`, `/debug`, `/verify`, and friends) are prompt-based and invoked the same way as custom ones.

The structural difference: Claude Code leaves the load decision almost entirely to the model's judgment from the description, and it has no built-in "this skill is *required* and you have not loaded it yet" loop. SCORPIOX CODE keeps the same on-demand core and layers **required-skills tracking with a capped nudge, force-inject, system-prompt injection, and usage-based top-percentage listing** on top — a control plane for "which skills must be present and which ones deserve a slot in the index" that the operator sets in text files rather than leaving to the model.

### OpenCode

OpenCode loads skills on demand through a native `skill` tool: the agent sees the available skills in the tool description and loads full content when it calls the tool. It walks up from the working directory to the git worktree root, collecting `.opencode/skills/`, plus `.claude/skills/` and `.agents/skills/` for cross-tool compatibility. Its distinguishing feature is a **permission layer**: per-skill `allow` / `deny` / `ask` patterns in config, with wildcards, so a skill can be hidden from the agent entirely or gated behind a user prompt.

The difference from SCORPIOX CODE is the axis of control. OpenCode gates *access* (can this agent load this skill at all, and does it need approval). SCORPIOX CODE gates *presence in context* (is this skill in the index, is it required, is it force-injected) and lets the operator tune the index by usage. OpenCode's permission model is the more security-oriented shape; SCORPIOX CODE's required-skills model is the more "make sure the agent is actually competent for this task" shape.

### Codex

OpenAI's Codex is also progressive disclosure, but it makes the budget explicit: the skills index is capped at a fixed slice of the model's context window, and if many skills are installed it shortens descriptions first and may omit some from the initial list, with a warning. The full `SKILL.md` is still read in full once a skill is selected. Codex discovers skills from `.agents/skills` in every directory from the working directory up to the repo root, plus user, admin, and system locations, and skills are also the unit of distribution inside plugins.

The difference: Codex treats the index as a hard-budgeted surface and is built around a *repo-rooted* discovery model with an admin/machine scope. SCORPIOX CODE does not expose a fixed character budget for the index — it lets you trim by pattern or by usage rank, and it keeps everything invokable no matter what is listed. Codex's budget is a static ceiling; SCORPIOX CODE's listing is a policy you can express as a glob or a usage rule.

### Pi

Pi is a minimal, extensible agent that implements the [Agent Skills specification](https://agentskills.io). Like SCORPIOX CODE, it keeps the body out of context until it is needed: at startup it scans its skill locations and adds each skill's *name, description, and path* to the system prompt, then the model reads the full `SKILL.md` when a task matches. It supports the same Agent Skills frontmatter (`name`, `description`, `disable-model-invocation`, and more), discovers directories containing `SKILL.md` recursively in user and project scopes, and exposes `/skill:name` to force a load when the model might not.

The difference is the surrounding model. Pi is deliberately minimal — it ships without subagents or plan mode and leans on its TypeScript extension API for anything beyond prompts. Its skills are a prompt-loading feature of that minimal core, with no required-skills tracking, no nudge loop, and no usage-based index filtering. SCORPIOX CODE's skills sit on top of a built-in control plane (required skills, nudges, force-inject, system-prompt injection, usage-ranked listing) that is tuned by config files rather than by writing a typed module.

### Hermes

Hermes does not expose a documented, stable skills system in its current release. Reusable behavior is handled through the agent's own tool calls or through external orchestrators that wrap it. There is no `SKILL.md` convention and no on-demand skill loader.

The practical effect: to give Hermes a reusable workflow, you either prompt it to follow the procedure each time or you maintain the procedure in an external system that shells out to Hermes. SCORPIOX CODE's skill is a single checked-in file the agent can discover, load, and re-load on its own.

---

## Side by side

| Dimension | SCORPIOX CODE | Claude Code | OpenCode | Codex | Pi | Hermes |
|-----------|---------------|-------------|----------|-------|----|--------|
| Format | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | none documented |
| Default in context | Name + description index | Name + description index | Name + description in tool | Name + description, budget-capped | Name + description in system prompt | n/a |
| Load trigger | Model invokes `InvokeSkill` | Model or user | Model invokes `skill` tool | Model reads on select | Model reads, or `/skill:name` | n/a |
| Forced presence | Required skills + capped nudge + force-inject | Frontmatter (invocation control) | Permission `ask` / approval | — | `disable-model-invocation` (opt-out) | n/a |
| Always-on skills | System-prompt injection | Dynamic context injection | — | — | — | n/a |
| Index sizing | Glob filter + top-% by usage | Model-managed | Model-managed | Fixed character budget | Model-managed | n/a |
| Discovery when hidden | `SearchSkills` (auto-enabled) | — | — | Warning + read on select | `/skill:name` | n/a |

---

## Why this matters

On-demand loading is now table stakes — every mainstream harness does it. The differences are in the *control plane around loading*, and that is where the behavior you actually feel comes from.

- **You can guarantee competence, not just hope for it.** A `required_skills.txt` line plus the capped nudge means "the agent should have loaded the release skill before it cuts a release" is something the harness enforces, not something you re-paste into every prompt.
- **The index is a budget you tune, not a cap you hit.** Two hundred skills do not have to crowd out the work. Trim by pattern or usage rank, and everything stays reachable.
- **The whole thing is plain text in git.** Skills, required lists, and filters are files you can read, review, diff, and version-control. There is no compiled plugin, no binary manifest, and no vendor lock-in — a skill folder is the same artifact that Claude Code, OpenCode, Codex, and Pi will also accept.

---

## TL;DR

- **On-demand by default.** The context carries only the skill's name and description; the body is loaded the moment a skill is invoked, so a large reference costs almost nothing until it is needed.
- **Two tools, one index.** `InvokeSkill` loads a skill by name; `SearchSkills` greps names, descriptions, and content to find one. A disk fallback picks up skills added mid-session.
- **A control plane other harnesses leave to the model.** Required-skills tracking with a capped nudge, force-inject, system-prompt injection, and usage-ranked index filtering — all set in plain text files, all diffable in git.
- **Portable by construction.** The `SKILL.md` format is the open Agent Skills standard, so a skill written for SCORPIOX CODE is the same file Claude Code, OpenCode, Codex, and Pi will read.

---

*Docs for SCORPIOX CODE @ 6c70ad6*
