# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

You want the agent to know how to do a specific job — deploy, release, review a PR, call a particular API — without that job's instructions sitting in the system prompt every single turn, burning tokens on work it is not doing.

SCORPIOX CODE treats skills as **on-demand knowledge**: only a skill's *name and description* are always visible to the agent. The full `SKILL.md` body is loaded into context only in the moment the agent decides to use it. That one decision — a small always-present index instead of a large always-present blob — is what the rest of this page is about, and how it compares with the other agent harnesses you will hear about.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** a skill is a folder with a `SKILL.md` in it. The agent always sees the folder's *name* and *one-line description*. It reads the whole document only when it chooses to load it. No config file, no registration step, no manifest — the folder is the registration, and the description is the trigger.

---

## What a skill is

A skill is a directory containing a `SKILL.md` file. The file has a short YAML frontmatter and a markdown body:

```markdown
---
name: deploy
description: Ship the current branch to staging. Use when the user says "deploy".
argument-hint: [target]
---

One-line summary of what this skill does.

## Rules

1. **Rule one** — the most important constraint.
2. **Rule two** — the second.

## Steps

Exact commands, templates, and examples.
```

| Frontmatter field | Required | Purpose |
|-------------------|----------|---------|
| `name` | Yes | The skill's identifier (kebab-case). It becomes the command name. |
| `description` | Yes | One line on *when* to use it. This is what the agent reads to decide. Keep it specific. |
| `argument-hint` | No | A hint at what `$ARGUMENTS` will contain when the skill is invoked. |

The body is where the real instructions live. When a skill is loaded, `$ARGUMENTS` in the body is replaced with whatever argument the agent (or you) passed in.

A skill is not limited to a document. A skill can also be an **executable script** (`.sh`, `.ps1`, …). When the agent invokes a script skill, SCORPIOX CODE runs it and feeds the exit code and output back to the agent — the same load-and-use contract, but the "content" is the script's result instead of static text.

---

## Where skills live (and who wins)

Skills are discovered from a cascade of directories. Each skill is identified by its **folder name**, and higher-priority locations override lower-priority ones with the same name:

| Location | Scope | Priority |
|----------|-------|----------|
| `./.claude/skills/<name>/SKILL.md` | Project | Highest |
| `~/.claude/skills/<name>/SKILL.md` | User | Middle |
| Built-in skills (shipped inside the product) | Global | Lowest |

A few things worth noticing about this table:

- **Project beats user beats built-in.** Put a skill in your repo and it shadows any same-named skill in your home directory or in the product itself. You can override a built-in by simply naming your own folder the same thing.
- **The product ships built-in skills.** They are embedded in the binary so they work with zero setup — things like `create-skill`, `edit-skill`, `create-command`, `edit-command`, and platform-specific helpers (preferred file-tool conventions, a fast-clone procedure, etc.). Because they are the *lowest* priority, you never have to think about them; drop your own and yours win.
- **It reads the open standard folders.** SCORPIOX CODE looks in `.claude/skills/` (and, as a fallback, `.agents/skills/`) rather than inventing a private path. Skills you already maintain for another tool that follows the same `SKILL.md` convention are picked up as-is.

There is also an **additional skills** mechanism: you can point the agent at extra local skill directories, and `/skills-update` performs a git pull over any of them that are git repositories. That is how a team keeps a shared, version-controlled skills library in sync across machines without anyone copying folders around.

Custom commands live in the same model. A markdown command file creates a `/command` exactly like a skill, so "command" and "skill" are two faces of the same mechanism rather than two systems.

---

## The on-demand model, precisely

This is the part that matters, so it is worth stating plainly.

At the start of the agent loop, SCORPIOX CODE does **not** paste every skill's body into the prompt. It builds a compact **index** — for each available skill, just its name and its description — and presents that index alongside the skill-loading tool. The full `SKILL.md` body is loaded only when the agent actually invokes the skill.

Consequences, all of which fall out of that single rule:

- **Token cost is flat, not proportional to the library.** Ten skills or two hundred skills: the always-present cost is a list of one-line descriptions, not a stack of documents. A 400-line reference procedure costs essentially nothing until it is needed.
- **The description is the contract.** Because the description is the only thing the agent sees before loading, it is the trigger. A vague description means the agent will not reach for the skill; a specific one ("Use when the user asks to create a consistent release and changelog") does. This is why the frontmatter asks *when* to use it, not just *what* it is.
- **Loading is a deliberate act.** The agent chooses to load. It can read the index, decide the task needs it, and pull the body in. That keeps irrelevant knowledge out of the window so it cannot crowd out the code you are actually working on.

There are two tools the agent uses for skills, and together they cover "find it" and "load it":

| Tool | What it does |
|------|--------------|
| **InvokeSkill** | Loads a skill by name and returns its content (with `$ARGUMENTS` substituted). For a script skill, it runs the script and returns the result. |
| **SearchSkills** | A grep-powered search over skill names, descriptions, and content. Returns matching names and descriptions. It is the discovery path when the always-visible index is not enough. |

`SearchSkills` is interesting because it is **grep-backed, not model-backed**: it matches literal text inside your skill files, so it finds the thing you actually wrote rather than guessing at what a skill might be about. It is engaged automatically when the visible index is partial (for example, when a list filter is in effect and some skills are hidden), and it can also be driven directly with an empty pattern to list everything.

---

## Controlling what the agent can see

The index is not all-or-nothing. You can shape it:

| Control | Effect |
|---------|--------|
| List filters | Show only a subset of skills in the always-visible index. When the list is partial, `SearchSkills` is enabled automatically so the hidden ones stay discoverable. |
| Per-skill disable (`/disable_skill`, `/enable_skill`) | A disabled skill stays listed (so the agent and the prompt stay cache-stable) but is refused when the agent tries to load it. It is a session-level kill switch, not a deletion. |
| `/skills` | Shows the loaded skills and where they came from. |
| `/skills-update` | Git-pulls the additional skill repositories. |

And there is a **hot reload**: the skill directories are watched, and adding, editing, or removing a `SKILL.md` is picked up without restarting the session. You edit a skill, the next turn sees it.

---

## Required skills: a guarantee, not a hope

Most harnesses leave it to the model to remember to use a skill. SCORPIOX CODE can be stricter. A **required-skills** list (kept per user, per project, and per session in a cascade of small text files) names skills that *must* be in play for the work at hand.

When a session begins, the harness checks the required list against what has actually been loaded. If a required skill has not been loaded, it injects a nudge into the conversation asking the agent to load it, and it keeps checking as a safety net across turns. There is also a forced mode that injects the missing required skills' content directly into the context as a single message, skipping the nudge-and-wait round-trip entirely, and a system-prompt mode that appends required skills' content verbatim to the system prompt.

In practice this is how you encode "for this repo, always follow the release checklist" or "for this customer's project, always use their deploy procedure" and have it *enforced* rather than merely suggested.

---

## How this compares to other harnesses

The short version: the industry has largely converged on the same core idea — **progressive disclosure** — and SCORPIOX CODE is squarely in that camp, not outside it. The differences are in the details: where skills live, how they are discovered, and what extra guarantees the harness adds on top. Here is the honest comparison.

| Harness | On-demand? | Where skills live | Discovery | Notable extras |
|---------|-----------|-------------------|-----------|----------------|
| **SCORPIOX CODE** | Yes — name + description only in prompt; body loads on invoke | `.claude/skills` (project/user), built-ins in the binary, additional git-backed dirs | Always-visible index + grep-powered `SearchSkills` (auto-enabled when the list is partial) | Built-in skill layer, required-skills nudge/force-inject guarantee, document-or-script skills, hot reload, per-skill runtime disable |
| **OpenCode** | Yes — native `skill` tool lists skills; body loads when the agent calls it | `.opencode/skills`, plus `.claude/skills` and `.agents/skills` (project and global) | Skills listed in the `skill` tool's description; the agent loads by calling the tool | Pattern-based permissions (`allow` / `deny` / `ask`) over which skills an agent may use |
| **Claude Code** | Yes — "a skill's body loads only when it's used" | `.claude/skills`, `.claude/commands` (merged into skills) | Description is always available; the agent loads when relevant, or you run `/skill-name` | Follows the Agent Skills open standard; bundled skills; subagent execution and dynamic context injection as extensions |
| **OpenAI Codex** | Yes — "progressive disclosure" | A skill is a directory with a `SKILL.md` plus optional scripts/references | Name + description shown first; full `SKILL.md` read on selection. The initial list is budget-capped (roughly 2% of the context window, or 8,000 characters) so large libraries do not crowd the prompt | Skills packaged as plugins and distributed through a shared plugin directory (ChatGPT and Codex) |
| **Pi** | Partially — explicit and file-driven | `.pi/skills/*.md` (plus `.pi/prompts`) | Skills are referenced by explicit links from the agent's instruction file (e.g. `AGENTS.md`) rather than an always-on index | An extension system; skills are loaded when the instruction file points at them |
| **Hermes** | Yes — "on-demand knowledge documents" with progressive disclosure | `~/.hermes/skills/` (primary), plus external skill directories | Name + description shown; full document loaded on demand | A **learning loop**: the agent can create skills from experience and improve them during use; a public Skills Hub for install-and-browse |

A few observations from that table, stated plainly:

- **You are not choosing a fundamentally new paradigm by picking SCORPIOX CODE.** OpenCode, Claude Code, and Codex all do progressive disclosure, and most of them follow the same open `SKILL.md` convention. The "load the body only when needed" idea is now the standard, not a differentiator.
- **What is genuinely different is the surrounding machinery.** SCORPIOX CODE ships a built-in skill layer you can override by name, a grep-based `SearchSkills` that searches what you actually wrote, a required-skills mechanism that *enforces* that certain skills are loaded (with a force-inject option), a git-backed additional-skills path, document-or-script skills, and hot reload. None of those are in the core open standard; they are what the harness adds on top.
- **Where the others lean differently:** Codex is the most aggressive about *budgeting* the always-visible list (a hard percentage of the context window). OpenCode is the most explicit about *permissioning* which skills an agent may touch. Hermes is the only one in this set that *mutates its own skills over time* — it creates them from experience and improves them in use. Pi is the most *explicit*: its skills are linked from the agent's instruction file rather than surfaced through a standing index.

If your priority is a large, shared, team-maintained skills library that the agent can reliably find and is guaranteed to honor for a given project, the combination of the built-in + project + additional cascade, `SearchSkills`, and required-skills enforcement is the part of SCORPIOX CODE you will feel. If your priority is cross-tool portability, the fact that it reads the standard `.claude/skills` / `.agents/skills` folders means the same skills move across harnesses with no rewriting.

---

## When to reach for a skill

Use a skill when you keep pasting the same multi-step procedure, checklist, or set of constraints into a session. If a chunk of instructions has grown into a *procedure* rather than a *fact*, it belongs in a skill, not in the project instruction file. A skill is the right shape for:

- **Repeated workflows** — deploy, release, review, migrate.
- **Domain or platform constraints** — "always use these file-tool conventions on this platform," "always call this API this way."
- **Encapsulated commands** — a named action with a body the agent can load on demand.

It is *not* the right shape for a single fact, a one-off note, or anything the agent should see on every turn by default. For always-on project context, use the project instruction file; for deterministic side effects on lifecycle events, use the hook system; for recurring timed work, use scheduled callbacks. A skill is specifically the "large body of procedure that is only sometimes needed" case, and the on-demand model exists precisely so that case stays cheap.

---

## Related

- [Project Instructions](project-instructions.md) — always-on project context, and when a fact belongs there instead of in a skill.
- [Event Hooks in SCORPIOX CODE](hooks-system.md) — folder-based deterministic side effects on lifecycle events.
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md) — recurring and timed work.
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md) — how the context window is managed as a session grows.
- [Data Privacy and Zero Data Collection](data-privacy.md) — where your skills and sessions are (and are not) sent.
