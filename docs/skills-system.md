# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

You want the agent to know how to do a specific job — deploy, release, review a PR, run a particular preview or RDP recipe — without that job's instructions sitting in the prompt every single turn, burning tokens on work it is not doing.

SCORPIOX CODE treats skills as **on-demand knowledge**: only a skill's *name and description* are always visible to the agent. The full `SKILL.md` body is loaded into context only in the moment the agent decides to use it. That one decision — a small always-present index instead of a large always-present blob — is what the rest of this page is about, and how it compares with the other agent harnesses you will hear about.

Docs for SCORPIOX CODE @ `41b090b`.

> **The whole idea in one line:** a skill is a folder with a `SKILL.md` in it. The agent always sees the folder's *name* and *one-line description*. It reads the whole document only when it chooses to load it. No config file, no registration step, no manifest — the folder is the registration, and the description is the trigger.

---

## What a skill is

A skill is a directory whose name is the skill name, containing a `SKILL.md` file:

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
| `name` | No* | Informational. SCORPIOX CODE uses the **folder name** as the skill's identity, so matching `name` to the folder keeps the skill portable to other harnesses that do read it. |
| `description` | Yes | One line on *when* to use the skill. This is what the agent sees in the index, and it is the trigger the agent uses to decide whether to load the skill. A skill without a description is skipped. |
| `argument-hint` | No | A hint at what `$ARGUMENTS` will contain when the skill is invoked. |

The body is where the real instructions live. When a skill is loaded, `$ARGUMENTS` in the body is replaced with whatever argument the agent (or you) passed in. A skill with no arguments simply has no `$ARGUMENTS` token and nothing changes. When a skill is loaded, the agent is also told **where on disk it came from**, so it can find and edit the file (the `edit-skill` built-in) without guessing which copy it should touch.

A skill is not limited to a document. A skill can also be an **executable script** (`.sh` on Unix, `.ps1` on Windows). When the agent invokes a script skill, SCORPIOX CODE runs it with the argument passed through, and feeds the exit code and output back to the agent — the same load-and-use contract, but the "content" is the script's result instead of static text.

Custom commands use the same mechanism. A `.md`, `.sh`, or `.ps1` file in `.claude/commands/` becomes a `/command`, and a `SKILL.md` folder is also available as `/skill-name`. "Command" and "skill" are two faces of one system, not two systems.

---

## Where skills live (and who wins)

Skills are discovered from a cascade of locations. Each skill is identified by its **folder name**, and higher-priority locations override lower-priority ones with the same name:

| Location | Scope | Priority |
|----------|-------|----------|
| `./.claude/skills/<name>/SKILL.md` | Project | Highest |
| `~/.claude/skills/<name>/SKILL.md` | User | Middle |
| Built-in skills (shipped inside the product) | Global | Lowest |

A few things worth noticing about this table:

- **Project beats user beats built-in.** Put a skill in your repo and it shadows any same-named skill in your home directory or in the product itself. You can override a built-in by simply naming your own folder the same thing.
- **The product ships built-in skills.** They are embedded in the binary so they work with zero setup — things like `create-skill`, `edit-skill`, `create-command`, `edit-command`, `preferred-sub-agents`, `preferred-preview`, `preferred-rdp`, `preferred-vnc`, `preferred-bot-plugin`, `fast-clone-instructions`, platform-specific file-tool conventions, and more. Because they are the *lowest* priority, you never have to think about them; drop your own and yours win.
- **It reads the open standard folders.** SCORPIOX CODE looks in `.claude/skills/` (and, as a fallback, `.agents/skills/`) rather than inventing a private path. Skills you already maintain for another tool that follows the same `SKILL.md` convention are picked up as-is.
- **You can inspect the built-ins.** The `scorpiox-builtin-skills` CLI lists, searches, shows, and even dumps the built-in skills that are embedded in the binary — `list`, `show <skill>`, `search <query>`, `--dump <folder>` — so the shipped knowledge is auditable, not a black box.

There is also an **additional skills** mechanism: point the agent at extra local skill directories (comma-separated paths in the `ADDITIONAL_SKILLS` setting), and `/skills-update` performs a git pull over any of them that are git repositories. That is how a team keeps a shared, version-controlled skills library in sync across machines without anyone copying folders around.

---

## The on-demand model, precisely

This is the part that matters, so it is worth stating plainly.

At the start of each request, SCORPIOX CODE does **not** paste every skill's body into the prompt. It builds a compact **index** — for each available skill, just its name and its description — and presents that index alongside the skill-loading tool. The full `SKILL.md` body is loaded only when the agent actually invokes the skill.

Consequences, all of which fall out of that single rule:

- **Token cost is flat, not proportional to the library.** Ten skills or two hundred skills: the always-present cost is a list of one-line descriptions, not a stack of documents. A 400-line reference procedure costs essentially nothing until it is needed.
- **The description is the contract.** Because the description is the only thing the agent sees before loading, it is the trigger. A vague description means the agent will not reach for the skill; a specific one ("Use when the user asks to create a consistent release and changelog") does. This is why the frontmatter asks *when* to use it, not just *what* it is.
- **Loading is a deliberate act.** The agent chooses to load. It reads the index, decides the task needs it, pulls the body in. That keeps irrelevant knowledge out of the window so it cannot crowd out the code you are actually working on.
- **The index is stable.** Skills are listed in a deterministic order and the index is cached across turns, so the tools JSON does not churn — and the provider prompt cache is not invalidated by a filesystem's arbitrary directory order.

There are two tools the agent uses for skills, and together they cover "find it" and "load it":

| Tool | What it does |
|------|--------------|
| **InvokeSkill** | Loads a skill by name and returns its content (with `$ARGUMENTS` substituted). For a script skill, it runs the script and returns the exit code and output. If a skill was created mid-session and is not in the index yet, the loader falls back to reading it straight from disk, so a freshly written skill is usable immediately. |
| **SearchSkills** | A grep-powered search over skill names, descriptions, and content. Returns matching names and descriptions. It is the discovery path when the always-visible index is not enough. |

`SearchSkills` is interesting because it is **grep-backed, not model-backed**: it matches literal text inside your skill files, so it finds the thing you actually wrote rather than guessing at what a skill might be about. It is engaged automatically when the visible index is partial (for example, when a list filter is in effect and some skills are hidden), and can also be driven directly with an empty pattern to list everything.

---

## Controlling what the agent can see

The index is not all-or-nothing. You can shape it:

| Control | Effect |
|---------|--------|
| List filters (`SKILL_LIST_INCLUDE` / `SKILL_LIST_EXCLUDE`, or pattern files) | Show only a subset of skills in the always-visible index. When the list is partial, `SearchSkills` is enabled automatically so the hidden ones stay discoverable. |
| Usage-based trimming (`SKILL_LIST_TOP_PERCENT`) | Keep the top N% of skills by recorded usage in the index. A skill that never gets used yields its slot to one that does. |
| Per-skill kill switch (`SKILL_<NAME>=0`, or `/disable_skill` / `/enable_skill`) | A disabled skill stays listed (so the prompt stays cache-stable) but is refused when the agent tries to load it. It is a session-level kill switch, not a deletion. |
| `/skills` | Shows the loaded skills and where they came from. |
| `/skills-update` | Git-pulls the additional skill repositories. |

The important invariant: **filtering only changes what is listed, never what is available.** A skill that is not in the index is still invokable by name and still findable with `SearchSkills`. You are trimming the index to save context, not pruning the capability set.

And there is a **hot reload**: the skill directories are watched, and adding, editing, or removing a `SKILL.md` is picked up without restarting the session. You edit a skill, the next turn sees it.

---

## Required skills: a guarantee, not a hope

Most harnesses leave it to the model to remember to use a skill. SCORPIOX CODE can be stricter. A **required-skills** list (kept per user, per project, and per session in a cascade of small `required_skills.txt` files) names skills that *must* be in play for the work at hand.

When a session begins, the harness checks the required list against what has actually been loaded. If a required skill has not been loaded, it injects a nudge into the conversation asking the agent to load it, and it keeps checking as a safety net across turns, capped at three nudge attempts per loop so it never stalls the work. There is also a **forced mode** that injects the missing required skills' content directly into the context as a single message, skipping the nudge-and-wait round-trip entirely, and a **system-prompt mode** that appends required skills' content verbatim to the system prompt.

In practice this is how you encode "for this repo, always follow the release checklist" or "for this customer's project, always use their deploy procedure" and have it *enforced* rather than merely suggested.

---

## How this compares to other harnesses

All of these harnesses have converged on the same file format — a folder with a `SKILL.md` carrying `name` and `description` — because it is the [open Agent Skills standard](https://agentskills.io). The differences are in *what is in context by default*, *who decides when the body loads*, and *what happens when a skill should have been loaded but was not*. Here is the honest comparison.

| Dimension | SCORPIOX CODE | Claude Code | OpenCode | Codex | Pi | Hermes |
|-----------|---------------|-------------|----------|-------|----|--------|
| Format | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) | `SKILL.md` (Agent Skills) |
| Default in context | Name + description index | Name + description index | Name + description in tool | Name + description, budget-capped | Name + description | Name + description |
| Load trigger | Model invokes `InvokeSkill` | Model or user | Model invokes `skill` tool | Model reads on select | Model reads, or `/skill:name` | Model invokes on demand |
| Forced presence | Required skills + capped nudge + force-inject | Invocation control in frontmatter | Permission `ask` / approval | — | `disable-model-invocation` (opt-out) | — |
| Always-on skills | System-prompt injection | Dynamic context injection | — | — | — | — |
| Index sizing | Glob filters + top-% by usage | Model-managed | Model-managed | Fixed character budget | Model-managed | Hub-managed |
| Discovery when hidden | `SearchSkills` (auto-enabled) | — | — | Warning + read on select | `/skill:name` | Hub browsing |

**Claude Code** is the closest in philosophy and, in practice, where the standard originated. It uses the same progressive disclosure: a skill's body loads only when it is used, and its name-and-description index is what the agent sees. It merged custom commands into skills, uses frontmatter to control who can invoke a skill, runs some skills in a subagent, and supports dynamic context injection. The structural difference: Claude Code leaves the load decision almost entirely to the model's judgment from the description, and has no built-in "this skill is *required* and you have not loaded it yet" loop.

**OpenCode** loads skills on demand through a native `skill` tool: the agent sees the available skills in the tool description and loads full content when it calls the tool. It walks up from the working directory to the git worktree root, collecting `.opencode/skills/`, plus `.claude/skills/` and `.agents/skills/` for cross-tool compatibility. Its distinguishing feature is a **permission layer**: per-skill `allow` / `deny` / `ask` patterns, so a skill can be hidden from the agent entirely or gated behind a user prompt.

**Codex** is the most aggressive about *budgeting* the always-visible list: the initial skills list is capped at 2% of the model's context window (or 8,000 characters when the window is unknown), descriptions are shortened first, and for large libraries some skills are omitted with a warning. It is also the most explicit about *distribution*: skills are packaged as plugins and shared through a universal plugin directory used by both ChatGPT and Codex.

**Pi** is deliberately minimal and the most *explicit*: its skills are prompt-loading files referenced by the agent's instruction file (e.g. `AGENTS.md`) or invoked with `/skill:name`, and its frontmatter can disable model-initiated invocation entirely. It ships without the surrounding control plane — no required-skills tracking, no nudge loop, no usage-based index filtering.

**Hermes** is the only one in this set that *mutates its own skills over time*: it keeps on-demand knowledge documents with progressive disclosure, and a learning loop that creates new skills from experience and improves them during use, alongside a public Skills Hub for install-and-browse.

**Where SCORPIOX CODE differs fundamentally:** it layers a **control plane** over the same on-demand core — a required-skills guarantee with capped nudges and force-injection, usage-ranked index trimming, grep-backed discovery, per-skill kill switches that stay cache-stable, built-in skills embedded in the binary with a CLI to audit them, and a git-backed shared library. All of it is plain text in your repos, tuned by config files rather than by writing a typed plugin module.

---

## When to reach for a skill

Use a skill when you keep pasting the same multi-step procedure, checklist, or set of constraints into a session. If a chunk of instructions has grown into a *procedure* rather than a *fact*, it belongs in a skill, not in the project instruction file. A skill is the right shape for:

- **Repeated workflows** — deploy, release, review, migrate.
- **Domain or platform constraints** — "always use these file-tool conventions on this platform," "always expose RDP/VNC/preview this way."
- **Encapsulated commands** — a named action with a body or script the agent can load on demand.

It is *not* the right shape for a single fact, a one-off note, or anything the agent should see on every turn by default. For always-on project context, use the project instruction file; for deterministic side effects on lifecycle events, use the hook system. A skill is specifically the "large body of procedure that is only sometimes needed" case, and the on-demand model exists precisely so that case stays cheap.

---

## Why this matters

- **You can guarantee competence, not just hope for it.** A line in `required_skills.txt` plus the capped nudge means "the agent should have loaded the release skill before it cuts a release" is something the harness enforces, not something you re-paste into every prompt.
- **The index is a budget you tune, not a cap you hit.** Two hundred skills do not have to crowd out the work. Trim by pattern or usage rank, and everything stays reachable.
- **The whole thing is plain text in git.** Skills, required lists, and filters are files you can read, review, diff, and version-control. There is no compiled plugin, no binary manifest, and no vendor lock-in — a skill folder is the same artifact that Claude Code, OpenCode, Codex, Pi, and Hermes will also accept.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — where `SKILL_LIST_*`, `REQUIRED_SKILLS`, and `ADDITIONAL_SKILLS` live.
- [Sub-Agents and Agent Swarms](sub-agents.md) — a built-in skill in action: `preferred-sub-agents` teaches the agent to spawn, peek at, and orchestrate worker agents.
- [Remote Agent Control and Fleet Management with SCORPIOX BOT](scorpiox-bot.md) — the preview, RDP, and VNC built-in skills publish live artifacts to SCORPIOX BOT.
