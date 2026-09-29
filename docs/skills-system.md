# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

You want the agent to know how to do a specific thing well — release a build, run the project's test matrix, follow a code-review checklist, format a changelog — without pasting a wall of instructions into every prompt or bloating your project instructions file. That is what a **skill** is for.

A skill in SCORPIOX CODE is a small folder on disk that holds one job's worth of instructions. The agent does not read all of them up front. It is told *that they exist and what each one is for*, and it pulls one into context only when the task needs it. That "tell it the menu, load on request" split is the whole design, and it is the part that separates SCORPIOX CODE from most other agent harnesses.

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** a skill is a folder with a `SKILL.md` in it. SCORPIOX CODE advertises the skills it can see to the model and loads a skill's body only when it is asked — so your context stays lean, your instructions stay portable, and adding a capability is a copy-paste, not a config edit.

---

## What a skill is

A skill is a directory containing a single `SKILL.md` file:

```
.claude/skills/
└── release-single-platform/
    └── SKILL.md
```

`SKILL.md` has a short YAML frontmatter block followed by a markdown body. The frontmatter is the *ad* — what the model sees in the listing. The body is the *manual* — what the model reads only after it decides the skill is relevant.

```markdown
---
name: release-single-platform
description: Build and ship a single-platform release. Use when the user asks to release one target.
argument-hint: [target]
---

## Rules
1. Never tag before the test matrix is green.
2. Always record the commit hash in the release notes.

## Steps
1. Run the build for `$ARGUMENTS`.
2. ...
```

Three frontmatter fields matter:

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | The skill's identifier (kebab-case). Matches the folder name. |
| `description` | Yes | One line on *when* to use it. This is what the model reads to decide whether to load the body.
| `argument-hint` | No | A hint at what `$ARGUMENTS` will contain when the skill is invoked. |

The body is plain markdown. You write exact commands, numbered rules, templates, examples — anything that makes the task reliable. `$ARGUMENTS` in the body is replaced with whatever arguments were passed at invocation, so one skill can serve many inputs.

A skill is deliberately small. One skill, one job. If you are writing two unrelated things, that is two skills.

---

## Where skills live, and who wins

SCORPIOX CODE looks for skills in a few places, from most specific to least. When two skills share a name, the higher-priority location wins — the lower one is shadowed, not merged.

| Location | Scope | Priority |
|----------|-------|----------|
| `./.claude/skills/` | Project-local | Highest |
| `~/.claude/skills/` | User-global | Below project |
| Paths in the `ADDITIONAL_SKILLS` setting | Shared / extra repos | Below user |
| Built-in skills (shipped in the binary) | Always available | Lowest |

A few points worth knowing:

- **Project beats user, user beats the rest.** Drop a `release-single-platform/` skill in the project and it silently overrides a same-named one in your home directory. That is how a repo pins its own workflow without touching your personal setup.
- **The folder name is the skill's identity.** The `name` field describes it; the directory holds it. To rename a skill, make a new one and delete the old — do not move the directory under a running assumption of an old name.
- **Built-ins have no file on disk.** A few skills (such as the ones that teach the agent how to create and edit skills) ship inside the binary so they work even in an empty directory. You can override them the same way: write your own `SKILL.md` with the same name in a project or user skills folder and yours takes precedence.
- **`ADDITIONAL_SKILLS` points at more folders.** Set it to a comma-separated list of local paths and SCORPIOX CODE scans those too. This is how a team shares a skills repository across machines — the path is the registration step, and `/skills-update` pulls those repos from git so the shared set stays current.
- **Default to project-local.** Put a skill in `./.claude/skills/` unless you genuinely want it to follow you across every project. Project-local skills are versioned with the repo, reviewed in pull requests, and disappear with the repo.

There is also a fallback folder, `.agents/skills/`, used only when a level has no `.claude/skills/` directory at all. It exists for cross-tool portability; it is never merged with `.claude/skills/`, and `.claude` always wins.

---

## The on-demand model: how it actually loads

This is where SCORPIOX CODE differs from a naive "dump everything into the system prompt" approach. The agent never sees every skill body at once. It sees a *listing* — names and descriptions — and it loads a body on demand.

The mechanism is two tools the agent can call:

- **Search** — a grep-powered search over the skills the agent can see. It matches against skill names, descriptions, and file content, with regex and case-insensitive support. An empty pattern lists everything. This is discovery: "what skills do I have that touch releases?"
- **Invoke** — load a skill by name. It returns the skill's full body (with `$ARGUMENTS` substituted) for the agent to follow as instructions or context.

So a run looks like: the model reads the listing, notices a relevant skill, optionally searches to confirm the exact name, invokes it, and only then does that skill's body enter the conversation. A skill you do not use costs nothing in context. A twenty-skill project does not make the prompt twenty skills longer — it makes a short index a little longer.

The listing itself is configurable, because a large team of skills can be as noisy as a giant system prompt:

- You can include or exclude skills from the listing by name or glob pattern, and rank the listing so the most-used skills surface first.
- Skills can be filtered in and out per environment without deleting them.
- A skill that is *not listed* is still invocable and still searchable — listing controls visibility, not availability.

This separation is the design intent: the listing is a prompt-caching and context-budget knob, while the on-disk skills remain the source of truth.

One detail that earns its keep: the listing is built in a stable, deterministic order. The same set of skills always produces the same listing text, so the prompt does not churn on every reload. That keeps provider prompt caches effective — a reshuffled listing would invalidate the cache and burn tokens on every turn, so the order is part of the contract, not an accident.

---

## Required skills: making a skill non-optional

On-demand means the agent decides when to load. Sometimes you do not want a decision — you want a skill to *be* present for a class of work. That is what the required-skills mechanism is for.

You can declare a set of required skills in a small text file, and SCORPIOX CODE cascades them across four levels:

| Level | File |
|-------|------|
| User | `~/.claude/required_skills.txt` |
| Project (Claude) | `.claude/required_skills.txt` |
| Project (Scorpiox) | `.scorpiox/required_skills.txt` |
| Session | `.scorpiox/sessions/<id>/required_skills.txt` |

Each line is a skill name (comments and blanks are ignored). The cascade is the union of all four, so a project can require skills on top of what a user has required, and a session can add to both. The agent tracks, per session, which required skills it has already loaded.

There are two ways to enforce it, and they differ in how pushy they are:

- **Nudge.** Before the agent acts, SCORPIOX CODE checks whether any required skill is still missing. If so, it nudges the agent to load it. The agent loads the skill itself through the normal on-demand path.
- **Force-inject.** With the force-inject option on, missing required skills are loaded for the agent directly and placed into context in one go, skipping the nudge-and-invoke round-trip entirely. This is the stronger guarantee: the skill is present, whether or not the agent would have remembered.

A skill that has been disabled for the session is treated as satisfied, so the enforcement loop never asks for something you explicitly turned off.

The practical pattern: a repo that requires its release skill to always be in play writes one line in `.scorpiox/required_skills.txt` and turns on force-inject, and every session in that repo starts with the release procedure already loaded.

---

## Managing skills from the prompt

You control skills without leaving the terminal:

- `/skills` — show the skills currently loaded, where each one came from, and what it is for. This is the "what do I actually have right now" view.
- `/skills-update` — pull the git-backed skill repositories named in `ADDITIONAL_SKILLS` so a shared skills set stays up to date.
- `/disable_skill <name>` — block a skill for the current session. It stays listed, but invoking it returns an error. Useful when a skill is present but not appropriate for the run.
- `/enable_skill <name>` — re-enable a skill you disabled.

Disabling is per-session and reversible. It does not touch the files, so the same skill behaves normally in the next session.

---

## How this differs from other harnesses

The short version: SCORPIOX CODE treats a skill as a *first-class, on-demand, filesystem-registered instruction unit* with an explicit discovery tool, an explicit load tool, a per-session enforcement loop, and a listing tuned for prompt-cache stability. Other harnesses reach for the same goal — "give the agent reusable, task-specific knowledge" — but they tend to do it through a different shape, and the shape has consequences.

**Claude Code** is the closest relative, and it is worth being precise about the difference because the formats look similar. Claude Code's skills also live in folders with a `SKILL.md` and also rely on progressive disclosure: the model sees a name and description and reads the body when it judges the skill relevant. SCORPIOX CODE keeps that folder-and-frontmatter model but makes the skill *explicit machinery* rather than an emergent behavior. The skills are advertised through a dedicated listing, discovered with a dedicated grep-based search tool, and loaded through a dedicated invoke tool, so the agent is never guessing whether a skill exists or how to reach it. On top of that, SCORPIOX CODE adds the pieces Claude Code leaves to convention: a cascading required-skills loop with a nudge or a force-inject, per-session disable, a configurable and ranked listing, and a deterministic order chosen to keep prompt caches alive. Same family, more of the heavy lifting done for you.

**OpenCode** reaches extensibility through a different axis entirely: it is built around *plugins* (code you register) and *agent* definitions (markdown persona/config files) rather than a searchable skill registry. You can give it reusable knowledge, but the unit is a plugin or an agent, not a small task-scoped `SKILL.md` the model pulls in on demand. There is no grep-across-skills discovery step and no per-session required-skill enforcement. The result is more powerful for code-level extensions and less granular for "just tell the agent how to do this one job."

**Codex** leans on a monolithic `AGENTS.md` plus custom prompt files. Everything the agent should know tends to land in one big instructions file that is read as a block. That is simple, but it does not scale the way on-demand skills do: the file grows with every capability you add, and the whole thing is in context whether or not the current task needs it. SCORPIOX CODE's skills are the opposite trade — many small files, an index in context, bodies loaded only when used.

**Pi** is intentionally minimal. Its extension surface is code-defined tools you write and register, not a markdown skill layer the model reads. You get fine-grained control over *tools*, but there is no convention for dropping a task instruction set into a folder and having the agent discover and load it without you writing tool code. Skills in SCORPIOX CODE are a layer above tools: plain instructions, no code, discovered by search, loaded on demand.

**Hermes** similarly models its extensibility as plugins and configuration rather than an on-demand, searchable instruction registry. You wire capabilities in and they are available; you do not hand the agent a menu it can query and pull from at runtime.

The pattern across all of them: most harnesses either (a) stuff knowledge into one always-present file, (b) require code-level plugins, or (c) rely on the model's general file-reading to find instructions. SCORPIOX CODE takes a fourth shape — a registry of small files, advertised by a tunable listing, found by a dedicated search tool, loaded by a dedicated invoke tool, and optionally enforced per session. That is what "on-demand" means here: the cost of a capability is paid only when the capability is used, and the registry itself is cheap enough to keep dozens of skills available at once.

---

## A skill, end to end

1. Make a folder: `.claude/skills/release-single-platform/`.
2. Write `SKILL.md` with `name`, `description`, and the steps. Use `$ARGUMENTS` where an input goes.
3. That is the whole registration. It is now in the listing, searchable, and invocable.
4. If the repo should always have it, add `release-single-platform` to `.scorpiox/required_skills.txt` and turn on force-inject if you want it loaded rather than nudged.
5. To share it, put the folder in a git repo, point `ADDITIONAL_SKILLS` at the local clone, and `/skills-update` to refresh it.

No manifest, no schema, no build step. The folder is the registration; the listing is the index; the load is on demand.

---

## The bottom line

A skill in SCORPIOX CODE is a folder with a `SKILL.md` in it: a short frontmatter that advertises the skill and a markdown body that the model reads only when it is asked. The model sees a tunable, deterministic listing of what is available, searches with a grep-based tool, and loads a body with an invoke tool — so context stays lean and every capability costs nothing until it is used. A cascading required-skills loop makes a skill non-optional when you need it to be, and per-session disable, git-backed shared repos, and a stable listing order round it out. Add a skill by dropping a file in a folder; remove one by deleting it; keep the whole thing portable, diffable, and versioned like any other file in the repo.

---

## Related

- [Event Hooks in SCORPIOX CODE: Folder-Based Automation Architecture](hooks-system.md)
- [Long-Horizon Agent Tasks: Conversation Compaction and Filesystem Session Architecture](conversation-compaction.md)
- [Deterministic File Editing & Developer Autonomy in SCORPIOX CODE](file-editing.md)
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md)
