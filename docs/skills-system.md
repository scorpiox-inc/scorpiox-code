# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

A **skill** in SCORPIOX CODE is a folder of instructions the agent loads only when it needs them. You write a skill once, and it stays out of the way until the agent decides it is relevant — or until your project tells the agent it is *required*. That is the whole idea: **expertise on demand, not expertise in the prompt.**

This page explains how the skills system works, how to use it, and — because the design choices are the point — how it differs fundamentally from the skills and context models in OpenCode, Claude Code, OpenAI Codex, Pi, and Hermes.

Source of truth: the skills subsystem at commit `5fd054b`.

---

## The one-sentence version

Most agent harnesses either shove context into the prompt and leave it there (context-file harnesses) or let the agent pull in a file when it wants to (progressive-disclosure harnesses). SCORPIOX CODE does the second thing **and adds a third layer the others do not have: a session-scoped *required skills* contract that actively nudges the agent to load the skills a project depends on.**

---

## What a skill is

A skill is a directory containing a `SKILL.md` file. The file starts with YAML frontmatter and a body:

```markdown
---
name: release-notes
description: Draft release notes from merged PRs. Use when preparing a tagged release.
---

## What this does
- Collects merged PRs since the last tag
- Groups changes by area
- Produces a copy-pasteable changelog section

## How
1. ...
```

Only two frontmatter fields drive discovery: **`name`** and **`description`**. The description is the single most important line you will ever write in a skill, because it is the *only* part the agent sees before it decides to load the whole thing. If the description does not say clearly *what it does* and *when to use it*, the agent will not reach for the skill.

The body is the full instruction set. It is **not** in the model's context until the skill is invoked. That is the core of the architecture.

---

## Where skills live

SCORPIOX CODE scans skills in a fixed priority order. A higher-priority skill with the same name overrides a lower-priority one:

| Location | Scope | Priority |
|----------|-------|----------|
| `./.claude/skills/<name>/SKILL.md` | Project | Highest — wins over user and built-in |
| `~/.claude/skills/<name>/SKILL.md` | User | Overrides built-in |
| Additional skill paths | Extra | Set via the `ADDITIONAL_SKILLS` config (comma-separated paths) |
| Built-in skills | System | Shipped with SCORPIOX CODE; lowest priority |

A few consequences fall out of this ordering:

- **Project skills are the default.** Put skills that only matter to this repo in `./.claude/skills/`. Use `~/.claude/skills/` for things you want everywhere.
- **You can override a built-in skill** by creating a project or user skill with the same name. This is how you customize built-in behavior without editing anything shipped.
- **Built-in skills are individually switchable.** Each built-in skill maps to a `SKILL_<NAME>` config key (for example `SKILL_CREATE_COMMAND`). Set it to `0` to turn that one skill off. You do not have to keep skills you never use.

You can inspect exactly what is loaded at any time with the **`/skills`** command, which opens a popup listing every discovered skill, its source (user / project / additional / built-in), and its description.

---

## How the agent sees skills

This is where the design gets concrete, and where SCORPIOX CODE is deliberately different from a "dump everything in the prompt" approach.

At any point in a session, the agent's context contains only a **list**: each skill's name and one-line description. That is it. The body of every skill is *not* present. The list is assembled into the `InvokeSkill` tool's description so the agent always knows what is available, but the list is compact — names and short descriptions, not full instructions.

Two tools do the work:

| Tool | What it does | Enabled |
|------|--------------|---------|
| **`InvokeSkill`** | Loads a skill by name and returns its full content for the agent to follow. | Yes, by default |
| **`SearchSkills`** | Grep-style search across skill names, descriptions, and content. Returns matches so you can *discover* a skill before invoking it. | Auto |

`SearchSkills` is the discovery path. If you have a large skill library, the agent can search it by keyword (it supports case-insensitive matching, regex, and whole-word flags) rather than being handed every description up front.

### The list is filtered, not truncated

You control which skills are *listed* in the agent's context using the `SKILL_LIST_*` config. The important guarantee: **filtering changes only what is listed, never what is usable.** Every skill remains invokable and searchable no matter what you filter out of the list. When a filter hides skills, SCORPIOX CODE automatically turns on `SearchSkills` so the hidden ones stay discoverable, and it appends a hint to the tool description like `(+3 more skills not listed — use SearchSkills to discover them)`.

This is a deliberate split between *what the agent sees by default* and *what the agent can reach.* You can keep the context tiny for a focused session while leaving the full library one search away.

### Why the ordering of the list matters (a quiet win)

The list is rendered in a **deterministic, sorted order**, and the loader skips a rescan when the source directories have not changed. The reason is subtle but important: the skill list is part of the model's tool definition, and a shuffled list would rewrite that definition and bust the provider's prompt cache on every reload. Keeping the order byte-stable and the rescan guarded means adding, removing, or editing skills does not silently tank your cache-hit ratio. You do not think about this, but it is why a big skill library does not quietly make your tokens more expensive.

---

## The required-skills contract (the part the others lack)

This is the layer that makes SCORPIOX CODE's model fundamentally different, not just a re-skin of progressive disclosure.

Every harness that uses on-demand skills has the same weakness: **the agent can forget to load a skill it is supposed to use.** The skill is on disk, the description is in the prompt, but the model never decides to invoke it, and you get generic behavior instead of the specialized workflow you wrote down.

SCORPIOX CODE treats "these skills must be loaded in this session" as a first-class, *enforced* concept.

You declare required skills in a cascade of `required_skills.txt` files, one skill name per line, read and unioned in this order:

| File | Scope |
|------|-------|
| `~/.claude/required_skills.txt` | User |
| `.claude/required_skills.txt` | Project |
| `.scorpiox/required_skills.txt` | Project |
| `.scorpiox/sessions/<id>/required_skills.txt` | This session only (agent-managed) |

The cascade is read-only for the agent; it only ever writes to the per-session file. That means a project can pin a set of skills that *every* session in that repo is expected to have loaded, and a session can add its own on top.

When a session starts, SCORPIOX CODE knows which required skills are still *not* loaded. If the agent gets to work without loading them, the harness injects a nudge — a system-level reminder naming exactly which required skills are missing and telling the agent to load them with `InvokeSkill`. So the "forgot to load it" failure mode is corrected automatically, in-loop, without you having to type "did you load the release-notes skill?"

Think of it as a contract with an enforcement hook:

- **You** state the expectation once, in a project file.
- **The harness** tracks loaded-vs-required per session.
- **The agent** gets a nudge to close the gap if it drifts.

No other harness in this comparison ships a mechanism like this. They can *list* a skill; they cannot *require* it and notice when it is missing.

---

## A skill, end to end

A concrete picture of a single skill's lifecycle:

1. **Author.** You create `./.claude/skills/release-notes/SKILL.md` with a good name, a sharp description, and a full body.
2. **Discover.** On session start the loader scans the skill directories and picks it up. Its name and description enter the agent's context via the `InvokeSkill` tool description — nothing more.
3. **Require (optional).** If this repo's releases always follow the same process, you add `release-notes` to `.claude/required_skills.txt`.
4. **Load.** The agent invokes `InvokeSkill` with `release-notes` — either because the task matched, or because the harness nudged it as a missing required skill. Now the full body is in context.
5. **Follow.** The agent works from the loaded instructions for the rest of the task.

At no point does the body cost context tokens until step 4. That is the entire value proposition, and it is what lets you run a large skill library without paying a permanent context tax.

---

## How this compares to the other harnesses

The table below is the heart of the page. "On-demand" means the skill body is out of context until loaded. "Required/enforced" means the harness can mark a skill as mandatory and notice if it is not loaded. "Discovery" means a way to find a skill you did not list up front.

| Capability | SCORPIOX CODE | Claude Code | Codex CLI | OpenCode | Pi / Hermes |
|-----------|---------------|-------------|-----------|----------|-------------|
| Skill format | `SKILL.md` + frontmatter | `SKILL.md` + frontmatter | `SKILL.md` + frontmatter | `SKILL.md` + frontmatter | Context files (`AGENTS.md`-style), no dedicated skill loader |
| On-demand loading (body out of context) | Yes | Yes | Yes | Yes | No — context files are always in the prompt |
| What stays in context by default | Name + description | Name + description | Name + description (and always-on `AGENTS.md`) | Name + description | The whole file |
| Required / enforced skills | Yes — `required_skills.txt` cascade + in-loop nudge | No | No | No | No |
| Discovery tool for unlisted skills | Yes — `SearchSkills` (grep, regex, auto-enabled) | Model-driven, no dedicated tool | Model-driven (`$` mention or auto-select) | Native skill tool, model-driven | N/A |
| Filter what is listed without hiding what is usable | Yes — `SKILL_LIST_*`, hidden skills stay searchable | No | No | No | N/A |
| Cache-aware tool/list ordering | Yes — deterministic order + guarded rescan | Not documented | Not documented | Not documented | N/A |
| Built-in skills, individually disableable | Yes — `SKILL_<NAME>` keys | Pre-built skills (document tasks) | Plugins distribute skills | No | No |

### Claude Code — the closest relative

Claude Code's Agent Skills are the nearest thing to SCORPIOX CODE's model, and the similarities are real: filesystem-based, `SKILL.md` with YAML frontmatter, name and description pre-loaded, full content loaded on demand, progressive disclosure. If you already know Claude Code skills, SCORPIOX CODE skills will feel instantly familiar.

The differences are exactly the three layers Claude Code does not ship:

- **Required skills.** Claude Code can list a skill; it cannot mark one as required and nudge the model to load it when it is missing. SCORPIOX CODE's `required_skills.txt` cascade plus the in-loop nudge is the single biggest behavioral difference.
- **A real search tool.** SCORPIOX CODE's `SearchSkills` is a grep-backed discovery tool that is auto-enabled when your list is partial. Claude Code relies on the model noticing a skill in its list.
- **Listing filters.** `SKILL_LIST_*` lets you shrink what is listed while keeping everything reachable. Claude Code has no equivalent knob.

SCORPIOX CODE also reads the same on-disk layout Claude Code uses (`.claude/skills/`), so skills you already have for Claude Code typically work here without edits.

### OpenAI Codex CLI — skills plus an always-on file

Codex uses the same `SKILL.md` format and supports on-demand skills, but it layers in a second, always-on mechanism: **`AGENTS.md`**, which is read into context for every session (globally and per repo). So Codex is a hybrid — some context is always on, some is on-demand. It also distributes skills as **plugins** and lets you invoke a skill explicitly (the `$` mention) or let Codex auto-select it from the description.

Where SCORPIOX CODE diverges: there is no always-on `AGENTS.md`-style tax in the skills model — the base context stays small by default — and there is no required/enforced layer. Codex can ship and surface skills; it cannot *require* one and detect its absence.

### OpenCode — a clean, permissive on-demand model

OpenCode is also genuinely on-demand: one folder per skill, `SKILL.md` inside, a native skill tool the agent calls to load content when needed. Its discovery paths are generous — it looks in its own directory *and* reads Claude-compatible and agent-compatible locations — which makes porting easy.

What it does not have that SCORPIOX CODE does: the required-skills enforcement, a dedicated grep search tool, and the listing filters. OpenCode's model is a faithful progressive-disclosure implementation; SCORPIOX CODE's is that model plus an enforcement and curation layer on top.

### Pi and Hermes — context-file harnesses

Pi and Hermes are lightweight harnesses built around the context-file pattern: a single always-in-context file (an `AGENTS.md`-style document) that carries your instructions, conventions, and preferences. There is no separate on-demand skill layer, no skill loader tool, and no notion of a required skill that gets enforced.

That is a legitimate design — for a small personal setup, one well-written context file is enough and simpler to reason about. The trade-off is that everything in that file costs context tokens in every session, whether you need it today or not. SCORPIOX CODE's skills model is what you reach for when that single file stops scaling: move each concern into its own skill, keep only the descriptions in context, and let the agent (or the required-skills contract) pull in the body only when it matters.

---

## The fundamental difference, stated plainly

On-demand skill loading is now common. Claude Code, Codex, and OpenCode all ship it. What separates SCORPIOX CODE is not that skills load on demand — it is what wraps that mechanism:

1. **A required-skills contract that is enforced in the loop.** You can declare that a session must have certain skills loaded, and the harness will tell the agent to load them if they are missing. This turns a skill from "available if the model remembers" into "guaranteed to be in play."
2. **A curation layer.** You can shrink the listed set for context economy without hiding anything, because every skill stays invokable and searchable, and a dedicated search tool covers the gap.
3. **Cache-aware plumbing.** The list order and reload behavior are designed so a big or changing skill library does not silently destroy your prompt-cache hit rate.

So the honest summary is this: SCORPIOX CODE's skills system is progressive disclosure with a contract, a search tool, and cache-awareness bolted on. The contract is the part no one else in this comparison has, and it is the reason a project can depend on a specialized workflow actually being loaded, rather than hoping for it.

---

## Quick start

Create your first project skill:

```bash
mkdir -p .claude/skills/release-notes
cat > .claude/skills/release-notes/SKILL.md << 'EOF'
---
name: release-notes
description: Draft release notes from merged PRs. Use when preparing a tagged release.
---

## What this does
- Collects merged PRs since the last tag
- Groups changes by area
- Produces a copy-pasteable changelog section
EOF
```

Run `/skills` to confirm it shows up as a project skill. If this repo should always use it, add the name to a project-level `.claude/required_skills.txt`:

```
release-notes
```

Now every session in this repo is expected to have `release-notes` loaded, and the agent will be nudged to load it if it starts working without it.

Rules of thumb:

- **Default to project skills** (`./.claude/skills/`). Use `~/.claude/skills/` only for skills you genuinely want everywhere.
- **Write the description for the agent, not for a human.** It must say what the skill does and when to use it — that line decides whether the skill ever gets loaded.
- **One skill, one job.** Small, focused skills load faster to trust than a single mega-skill.
- **Turn off built-ins you never use** via their `SKILL_<NAME>` key instead of living with descriptions you do not need.
- **Filter the list, never the library.** Use `SKILL_LIST_*` to keep context small; rely on `SearchSkills` to keep everything reachable.
