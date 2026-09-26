# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

A skill is a reusable, self-contained instruction pack. Instead of piling every
procedure you might need into one giant project-instructions file, you write a
small folder for each one — a name, a one-line "when to use me", and the full
instructions. SCORPIOX CODE advertises every skill's **name and description** to
the model up front, and only loads a skill's **full body** the moment it is
actually needed. That is the whole trick: you can install hundreds of skills,
but the model only ever pays the context cost for the few it is currently
using.

This page explains how the skills system works inside SCORPIOX CODE, what makes
its on-demand design different in kind (not just in degree) from the way
Claude Code, OpenCode, OpenAI Codex, Pi, and Hermes do it, and how to manage
skills day to day.

Docs for SCORPIOX CODE @ `b59223a`.

---

## What a skill is

A skill is a directory that contains a `SKILL.md` file:

```
.claude/skills/
└── deploy-staging/
    ├── SKILL.md          <- required
    ├── scripts/          <- optional
    └── references/       <- optional
```

`SKILL.md` starts with a small YAML frontmatter block, followed by the
instructions themselves:

```markdown
---
name: deploy-staging
description: Deploy to staging and run smoke tests. Use when asked to ship to staging.
argument-hint: [branch]
---

1. Build the artifact.
2. Run `deploy $ARGUMENTS`.
3. Run the smoke-test suite and report the result.
```

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | The skill's identifier (kebab-case). Should match the folder name. |
| `description` | Yes | One or two lines on **what it does and when to use it**. This is the only text the model sees until the skill is invoked, so it decides whether the skill is ever picked. |
| `argument-hint` | No | Documents what `$ARGUMENTS` will contain when invoked. |

Skills follow the same open **Agent Skills** layout that the rest of the
industry has converged on: a `SKILL.md` plus optional `scripts/`, `references/`,
and `assets/` folders. That means a skill you write for SCORPIOX CODE generally
works in other Agent Skills–compatible harnesses, and vice versa.

Two small but useful details:

- **`$ARGUMENTS` substitution.** Everything you type after the skill name is
  substituted for `$ARGUMENTS` in the body, so one skill can be parameterized
  (`/deploy-staging main` → the skill sees `main`).
- **Skills can be scripts.** A skill whose file is a shell script (`.sh`, or
  `.ps1` on Windows) is *executed* rather than just read, and its output is fed
  back as the result. That makes a skill a lightweight executable workflow, not
  only a blob of prose.

---

## Where skills live

SCORPIOX CODE looks for skills in several places and merges them into one
catalog. When the same skill name appears in more than one place, the
higher-priority source wins — it is not merged.

| Source | Location | Scope | Priority |
|--------|----------|-------|----------|
| **Project** | `./.claude/skills/<name>/` | This repo | **Highest** |
| **User** | `~/.claude/skills/<name>/` | You, everywhere | Middle |
| **Built-in** | Compiled into the binary | Always available | Lowest |

- **Project overrides user overrides built-in.** A skill of the same name in
  your repo shadows your personal copy, which in turn shadows the built-in one.
  That is how you tune a built-in for one project without touching anything
  global.
- **`.agents/skills/` is a fallback folder name.** If there is no `.claude/`
  directory at a given level, SCORPIOX CODE falls back to `.agents/skills/` at
  that same level (it never merges the two — `.claude/` always wins when both
  exist). This keeps skills portable across the many harnesses that read the
  `.agents/` layout.
- **Extra paths.** Set `ADDITIONAL_SKILLS` to a comma-separated list of
  additional skill directories (for example a shared team repo of skills). They
  are scanned alongside the project and user locations, so you can pull in a
  large, version-controlled skill library without copying it into your repo.

Default to **project-level** skills for anything repo-specific; keep genuinely
personal, everywhere skills in `~/.claude/skills/`.

---

## On-demand: how a skill reaches the model

This is the part that matters. SCORPIOX CODE never dumps every skill body into
the context. It uses **progressive disclosure**:

1. **Advertise (cheap).** At the start of the session, each skill contributes
   only its `name` and `description` to the model's tool context. A hundred
   skills costs only their one-line descriptions — not their bodies.
2. **Invoke (on demand).** When the model decides a skill is relevant — or you
   type the skill's name directly — SCORPIOX CODE loads that skill's full
   `SKILL.md` body and hands it to the model as context for the task.
3. **Use.** The model follows the instructions, resolving any bundled
   `scripts/` and `references/` files relative to the skill's own folder.

The model loads a skill with the **`InvokeSkill` tool** (by name). You can do
the same thing from the keyboard by typing the skill as a slash command, e.g.
`/deploy-staging main`. Both paths behave the same: the body is loaded,
`$ARGUMENTS` is substituted, and (for script skills) the script is executed.

### Discovering skills you did not see listed

When your catalog is large, SCORPIOX CODE may **filter the advertised list** to
keep the context window small (see the next section). A skill that is not in
the advertised list is still fully usable — the **`SearchSkills` tool** finds
it. `SearchSkills` runs a grep-style search across every skill's name,
description, and file contents, so the model can discover the right skill by
keyword even when it was not shown in the upfront list. You can also list or
inspect what is loaded with the `/skills` popup and `/help`.

The net effect: **the advertised list is a hint, not a limit.** A skill is
always discoverable and invokable; only the size of the cheap up-front
advertisement is controlled.

---

## Keeping the advertised list lean

You control exactly which skills appear in the up-front advertisement using the
`SKILL_LIST_*` settings. None of these change what is *usable* — they only
change what is *listed*:

| Setting | What it does |
|---------|--------------|
| `SKILL_LIST_INCLUDE` / `SKILL_LIST_INCLUDE_FILE` | Glob patterns (or a file of them) of skill names to include. Empty means "include all". |
| `SKILL_LIST_EXCLUDE` / `SKILL_LIST_EXCLUDE_FILE` | Glob patterns of skill names to hide from the list. |
| `SKILL_LIST_TOP_PERCENT` | Advertise only the top N% of skills **by historical usage in this project's past sessions**. The list adapts to what you actually reach for. |
| `SKILL_LIST_STYLE` | Set to `names` to advertise only skill names (no descriptions) for the tightest possible footprint. |

Two rules always win, regardless of filters: **built-in skills** and
**required skills** (next section) are always listed. And whenever the filter
hides some skills, SCORPIOX CODE automatically enables `SearchSkills` and appends
a `(+N more skills not listed - use SearchSkills to discover them)` note, so the
model knows the hidden ones exist and how to reach them.

`SKILL_LIST_TOP_PERCENT` is the one to reach for on a big library: pin the
skills you use every session to the front of the context, and let the long tail
stay one `SearchSkills` call away.

---

## Required skills: guaranteeing a session loads what it needs

On-demand is powerful, but it means the model has to *decide* to load a skill.
Sometimes you want a guarantee — "this session must have skill X loaded before
it does anything."

For that, SCORPIOX CODE has **required skills**, tracked per session through a
small cascade of `required_skills.txt` files:

```
~/.claude/required_skills.txt              (you, everywhere)
./.claude/required_skills.txt             (this project)
.scorpiox/required_skills.txt             (this project, alt location)
.scorpiox/sessions/<id>/required_skills.txt  (this session only)
```

At the start of a run, SCORPIOX CODE checks which required skills have not yet
been loaded and **nudges the model to load them** — or, with forced mode on,
**injects their content directly** so it does not even have to spend a round-trip
asking. Skills the model loads during the session are recorded back into the
session's list, so a long session that grows a new required skill stays honest
about what it has and has not loaded.

This is a small feature that other harnesses do not give you: the ability to
declare "these skills are load-bearing for this project/session" and have the
harness enforce it, not just rely on the model noticing.

---

## Built-in skills

SCORPIOX CODE ships a set of **built-in skills compiled into the binary**. They
are always present (no install step), and they are the lowest-priority source —
so any project or user skill with the same name cleanly overrides them.

Typical built-ins cover the harness's own workflows (creating and editing
skills and commands), the offline `scorpiox-code-help` documentation reader,
preferred tooling for file operations and MCP servers, a fast git clone
workflow, and small platform utilities. Where behavior differs by OS, the
appropriate variant (Linux or Windows) is selected at build time.

Because built-ins are overridable, you can take a default, read it to learn the
shape of a skill, then copy and adapt it for your own project.

---

## Managing skills day to day

- **`/skills`** — popup listing every loaded skill, its description, and where
  it came from (project / user / built-in).
- **`/help`** — available commands and their descriptions.
- **`create-skill`** — a built-in skill that scaffolds a new
  `.claude/skills/<name>/SKILL.md` with the frontmatter in place.
- **`edit-skill`** — a built-in skill that opens an existing skill for editing
  (it tells you exactly which copy to edit when the name exists in more than
  one location).

The same `create-skill` / `edit-skill` pair is what you use to grow the
catalog — one skill per job, with a `description` written so the model can tell
from one line whether the skill applies.

---

## How this compares to other harnesses

SCORPIOX CODE is deliberately in the **Agent Skills** camp: a `SKILL.md` per
skill, advertised by name and description, body loaded on demand. That puts it
in the same family as Claude Code, OpenCode, Codex, Pi, and Hermes. The
differences are in *how the list is managed, how discovery works, and how the
harness can guarantee a skill is loaded* — and those are where "on-demand"
stops being a slogan and becomes an architecture.

| Harness | Skill format | Advertised up front | Body loads when | Distinctive mechanism |
|---------|--------------|---------------------|-----------------|----------------------|
| **SCORPIOX CODE** | `SKILL.md` + optional scripts/references (Agent Skills) | Name + description, **filterable per project** | On invoke (`InvokeSkill` or `/name`) | **`SearchSkills`** grep discovery that auto-engages when the list is filtered; **`SKILL_LIST_TOP_PERCENT`** adapts the list to past usage; **required skills** enforced per session; **built-ins compiled into the binary** and overridable |
| **Claude Code** (Anthropic) | `SKILL.md` (custom commands merged into skills) | Name + description for all skills | On invoke; body persists in the conversation as a message and is re-attached on compaction | Richest lifecycle: invocation control, subagent execution, dynamic context injection, per-skill tool pre-approval; cloud-synced skills |
| **OpenCode** | `SKILL.md` | Name + description in a native `skill` tool | On demand via the skill tool | Walks up to the git worktree to discover project skills; reads `.opencode/`, `.claude/`, and `.agents/` skill folders |
| **OpenAI Codex** | `SKILL.md` (+ `openai.yaml`) | Name + description (+ path), **capped at ~2% of context / 8000 chars** | On select | Hard character **budget** on the initial list (shortens descriptions, may omit and warn); `/skills` and `$` mention; record-&-replay skill creator |
| **Pi** (earendil-works) | `SKILL.md` (Agent Skills spec) | Name + description in the system prompt | On demand; `/skill:name` to force | Minimalist harness; `disable-model-invocation` to make a skill command-only; bundled scripts/references resolved relative to the skill |
| **Hermes** (Nous Research) | `SKILL.md` + plugins | Name + description | On demand | Self-improving agent: skill **creation and curation** as a first-class loop, persistent memory, and a large plugin/skill ecosystem around the core |

### The things worth noticing

**Everyone loads the body on demand now.** Progressive disclosure is table
stakes — Claude Code, OpenCode, Codex, Pi, and Hermes all advertise only
name + description and defer the full `SKILL.md` until a skill is chosen.
SCORPIOX CODE shares this. So the honest comparison is not "do you load the
body lazily" — everyone does — but **what you do with the list and the
guarantee**.

**SCORPIOX CODE treats the advertised list as a tunable surface, not a fixed
one.** Codex answers the "too many skills" problem with a hard character budget
(shorten descriptions, drop some, warn). SCORPIOX CODE answers it with
*controls*: include/exclude globs, and `SKILL_LIST_TOP_PERCENT`, which ranks the
list by what you actually used in past sessions. You decide the footprint
explicitly instead of a budget silently trimming it.

**Discovery is a tool, not an accident of the list.** Because SCORPIOX CODE
keeps **`SearchSkills`** available and auto-enables it whenever the list is
filtered, the model can always reach the long tail by keyword — names,
descriptions, and file contents — rather than hoping a skill made the cut. The
list can be tiny; the catalog can be huge.

**The harness can enforce skills, not just suggest them.** The required-skills
cascade plus pre-flight nudge (or direct injection) is the part no other
harness in this table offers: declare the skills a project or session depends
on, and the harness makes sure they are loaded before work begins.

**Built-ins are shipped, not installed.** A curated set of built-in skills is
compiled into the binary — present on every install, and overridable per
project. You get a working scaffold for the harness's own workflows (including
the `create-skill` / `edit-skill` pair) with nothing to set up.

---

## Quick checklist

- [ ] One folder per skill under `.claude/skills/<name>/`, each with a `SKILL.md`.
- [ ] `description` written so the model can decide from one line — this is the only text it sees before invoking.
- [ ] Project skills for repo-specific work; `~/.claude/skills/` for personal, everywhere skills; `ADDITIONAL_SKILLS` for shared libraries.
- [ ] Big catalog? Trim the advertisement with `SKILL_LIST_TOP_PERCENT` and `SKILL_LIST_EXCLUDE` — `SearchSkills` keeps the long tail reachable.
- [ ] Load-bearing skills? List them in `required_skills.txt` so the session is guaranteed to load them.
- [ ] Override a built-in for one project by adding a same-named project skill.
- [ ] Use `/skills` to see what is loaded and where it came from.

---

Docs for SCORPIOX CODE @ `b59223a`.
