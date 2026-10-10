# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

Every agent harness has the same problem: a model is smart but it does not know *your* procedures. How you cut a release, how your test suite is laid out, the exact steps to rotate a credential, the checklist your team applies before touching the payments service. You could paste that into the chat every time, or you could write it down once as a **skill** — a small, self-contained procedure the agent loads only when the task actually calls for it.

SCORPIOX CODE treats skills as a first-class, on-demand subsystem. Skills live in folders on disk, are discovered and loaded through two dedicated tools, and are governed by a listing policy you control. This page explains how the whole thing works end to end, and where the design diverges from OpenCode, Claude Code, OpenAI Codex, Pi, and Hermes.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** a skill is a folder with a `SKILL.md` in it, and SCORPIOX CODE never asks the model to remember that a skill exists — it either puts the skill in the model's context as a one-line catalog entry or hands the model a real grep search to find it. Reachability is engineered, not hoped for.

---

## What a skill actually is

A skill is a directory containing a file named `SKILL.md`:

```
release-notes/
└── SKILL.md
```

The file opens with a small YAML frontmatter block between `---` markers, then the instructions themselves in Markdown:

```markdown
---
name: release-notes
description: Ship a release with changelog. Use when asked to cut a release.
argument-hint: [version]
---

Draft the release notes from merged PRs, propose a version bump,
and produce a copy-pasteable release command for $ARGUMENTS.
```

Three fields matter:

| Field | Required | What it does |
|-------|----------|--------------|
| `name` | Yes | The identity used by the skill tool and by `/name`. It must match the folder name. |
| `description` | Yes | The one line the agent uses to decide whether this skill is relevant. Keep it short and start with what it does, then a `Use when ...` trigger. |
| `argument-hint` | No | Free-text hint shown in autocomplete describing what `$ARGUMENTS` will contain. |

Two conventions keep a library sane. The folder name *is* the skill name — a folder called `release-notes` becomes the skill `release-notes` and the slash command `/release-notes`. And a skill should do one job; if a procedure has three distinct phases, that is usually three skills.

`$ARGUMENTS` is replaced with whatever you pass at invocation time. Loading a skill as `/release-notes 2.4.0` expands the skill body and substitutes `2.4.0` wherever `$ARGUMENTS` appears, then runs the result as a normal turn.

---

## Where skills live, and which one wins

Skills are discovered from four tiers. When two tiers hold a skill with the same name, the higher tier replaces the lower one entirely — they are never merged.

| Tier | Location | Scope |
|------|----------|-------|
| Project | `.claude/skills/<name>/SKILL.md` | The repo you are working in. Highest precedence. |
| User | `~/.claude/skills/<name>/SKILL.md` | Every project on your machine. |
| Additional | Folders listed in `ADDITIONAL_SKILLS` | Shared or team-managed skill repos. |
| Built-in | Compiled into the binary | Shipped procedures you cannot edit. Lowest precedence. |

The precedence rule means you can override a built-in skill with your own file of the same name: create `.claude/skills/edit-skill/SKILL.md` and your version replaces the compiled-in one. It is also why a project skill named `deploy` silently shadows a user-level `deploy` — the winner is deterministic, and it is always the more specific location.

There is one cross-tool accommodation. If `.claude/skills/` does not exist but `.agents/skills/` does, SCORPIOX CODE uses `.agents/skills/` instead. This is a folder-name fallback, not a merge: the two are never combined, and when both exist `.claude/skills/` always wins. The fallback exists so a repo laid out for the wider Agent Skills ecosystem still works here without a rename.

Once loaded, a skill's body is prefixed with a one-line header naming where it came from, so the agent can tell project copies from user copies and knows which file to edit:

```
[skill: release-notes | Project Skill | path: /repo/.claude/skills/release-notes/SKILL.md]
```

Built-in skills say so instead, and name the path you would create to override them.

---

## The two-tool model: load and discover

SCORPIOX CODE splits skill handling into two tools. This split is the heart of the architecture, and it is what most other harnesses do not have.

**The load tool** takes a skill name and returns its content for the agent to follow. This is the on-demand half: the body of a skill enters context only when it is invoked, not when the session starts. Whether the name came from the visible catalog, a search result, or the agent's own memory, loading works identically.

**The discovery tool** is a real search. It runs a grep across every skill directory — your user folder, the project folder, and each folder in `ADDITIONAL_SKILLS` — matching literal text inside `SKILL.md` files as well as names and descriptions. Built-in skills have no file on disk, so they are matched in memory against their names, descriptions, and loaded content.

| Parameter | Required | What it does |
|-----------|----------|--------------|
| `pattern` | Yes | Text to match against skill file content, names, and descriptions. An empty string lists every skill. |
| `flags` | No | `-i` (case-insensitive, the default), `-E` (regex), `-w` (whole word). Combine them, for example `-i -w` or `-E -i`. |

Because discovery is grep-backed rather than model-backed, a match is a literal string you actually wrote. Write your `SKILL.md` descriptions and bodies with the vocabulary a future task will use — the error text, the tool name, the domain word — and the search will find them. The result is a compact list, one entry per matching skill, each with its name, description, source tier, and file path.

A skill created mid-session is reachable immediately. If a skill is not yet in the session's index, invoking it by name still resolves by reading the folder directly from disk, so you never have to restart to use a skill you just wrote.

---

## The listing is a dial, not a constant

Here is the design decision that separates SCORPIOX CODE from the rest. Most harnesses either always advertise every skill in the tool description (a catalog that grows with your library) or apply a fixed, invisible budget. SCORPIOX CODE makes the listing a policy you set.

The catalog, when present, is a block of `- name: description` lines inside the load tool's description — part of what the model reads every turn. By default every skill is listed. You can narrow that with the listing controls:

| Key | Effect |
|-----|--------|
| `SKILL_LIST_INCLUDE` | Comma-separated glob patterns; only matches are listed. |
| `SKILL_LIST_EXCLUDE` | Glob patterns to drop from the list. |
| `SKILL_LIST_INCLUDE_FILE` / `SKILL_LIST_EXCLUDE_FILE` | Same patterns, one per line in a file, `#` comments allowed. |
| `SKILL_LIST_TOP_PERCENT` | Keep the top N% of skills by recorded usage across sessions. |
| `SKILL_LIST_STYLE` | `full` (name plus description, the default) or `names` (names only, cheaper). |

The filters run as a pipeline: include, then exclude, then top-percent. Built-in skills are always re-listed so the always-available procedures never fall out of the catalog, and skills named in the required-skills cascade are re-listed for the same reason.

The crucial property is that **listing is not loading**. Every skill remains invokable by name and searchable by discovery regardless of how the catalog is filtered. Hiding a skill from the list saves bytes on every request; it does not remove the capability.

Two behaviors follow automatically:

- **Partial listings self-heal.** When some skills are hidden, the discovery tool is enabled in memory for that session, and the tool description carries a hint naming how many skills are hidden and pointing at search. The agent always knows a larger library exists behind the search.
- **Legacy lazy mode is a degenerate case of the same idea.** Setting `TOOL_SEARCHSKILLS=1` with no other filter drops the catalog entirely and leaves only the pointer to search. It is the same dial turned all the way down. See [Lazy Skill Loading](lazy-skill-loading.md) for the full treatment of that mode.

An honesty note on the numbers: the filter log records every listing decision, so you can see exactly which skills were listed and why. Tuning the catalog is observable, not guesswork.

---

## The guarantee layer: required skills

A catalog can be tuned; a procedure that *must* be followed cannot be left to chance. SCORPIOX CODE separates "skills the agent might use" from "skills this session must use" with a contract called the required-skills cascade.

Names are read from up to four files, unioned:

| Level | File |
|-------|------|
| User | `~/.claude/required_skills.txt` |
| Project | `.claude/required_skills.txt` |
| Project | `.scorpiox/required_skills.txt` |
| Session | `.scorpiox/sessions/<id>/required_skills.txt` |

The key word is **union**: unlike the skill folder cascade, where one location replaces another, required-skill files accumulate. A skill required at user level and another at project level are both required. The cascade files are read-only to the agent; only the session file is written, and only through the dedicated tooling that tracks what has actually been loaded.

Three switches govern enforcement:

| Key | Effect |
|-----|--------|
| `REQUIRED_SKILLS` | Master switch for the whole contract. On by default. |
| `REQUIRED_SKILLS_IN_SYSTEM_PROMPT` | Injects the raw `SKILL.md` of required skills into the system prompt. Names injected this way count as loaded. On by default. |
| `REQUIRED_SKILLS_FORCED` | Instead of nudging, force-loads every missing required skill's content into context in one shot at session start. |
| `SYSTEM_PROMPT_SKILLS` | Extra names to inject into the system prompt, always applied, unioned with the files above. |

When enforcement is active, the session tracks which required skills have been loaded — whether through the load tool, a slash invocation, or system-prompt injection. At the end of a turn, any required skill that is still missing triggers a nudge naming it and instructing the agent to load it. The nudge is the safety net; force-inject and system-prompt injection are the fast paths that avoid an extra round-trip.

This is the layer that makes lazy mode safe. If you turn the catalog off and a required skill is not advertised, the nudge still names it explicitly. You get the token savings of a hidden catalog *and* the guarantee that the skills you marked as mandatory are loaded.

---

## Runtime controls

Skills are not just files; they are things you manage while the agent is running.

- **`/skills`** opens a popup listing every skill the session discovered, with descriptions and source tiers. It is the quick way to see what is actually loaded before you ask the agent to use something.
- **`/skills-update`** runs a fast-forward git pull on every `ADDITIONAL_SKILLS` folder that is a git checkout, then reloads the skill index. Point `ADDITIONAL_SKILLS` at a shared team repo once and keep it current from inside the session.
- **`/disable_skill <name>`** blocks a skill for the current session. The skill stays in the listing — deliberately, so the tool description does not change between turns and the prompt cache stays stable — but any attempt to load it is refused with an error telling the agent to proceed without it. **`/enable_skill <name>`** undoes this.
- **`/name [arguments]`** invokes any skill as a slash command, expanding its body and substituting `$ARGUMENTS`. A skill that turns out to need arguments reports this via its `argument-hint`.

Disabled skills are also counted as satisfied by the required-skills nudge, so you never get nudged to load a skill you explicitly turned off.

---

## How this compares to other harnesses

Every tool below has adopted some form of on-demand skills, most of them following the cross-vendor Agent Skills convention — a `SKILL.md` folder with `name` and `description` frontmatter. The convergence on the file format is real. The differences are in discovery, listing policy, and how each tool guarantees a skill gets used.

| Harness | Skill format | How skills are advertised | Discovery beyond the list | Guarantee that a skill is used |
|---------|--------------|---------------------------|---------------------------|--------------------------------|
| **SCORPIOX CODE** | `SKILL.md` folder; `name`, `description`, `argument-hint` | Tunable catalog in the load tool: include/exclude globs, top-N% by usage, names-only mode | A literal grep search across all skill files, names, and descriptions; empty pattern lists all | `required_skills.txt` cascade with end-of-turn nudge, force-inject, and system-prompt injection |
| **Claude Code** | `SKILL.md` folder; Agent Skills standard plus extensions | Name and description in the skill listing; combined description and `when_to_use` truncated at a fixed character cap | Model matches on description; `/skill-name` to force | Model judgement by description; manually invoked skills are user-driven |
| **OpenCode** | `SKILL.md` folder; Agent Skills fields, OpenCode behavior in metadata | `<available_skills>` block in the skill tool description, name plus description | Model matches on description; `opencode/autoinvoke: "false"` keeps a skill out of auto-selection guidance | Model judgement; skills with slash metadata are manually invokable |
| **OpenAI Codex** | `SKILL.md` folder plus optional scripts, references, assets; Agent Skills standard | Name, description, and file path; the initial list is capped at a share of the context window (or a fixed character budget when unknown) | Explicit `$` mention or `/skills`; implicit by description. Large sets shorten descriptions first, then may omit skills with a warning | Model judgement by description; plugin packaging for distribution |
| **Pi** | `SKILL.md` folder; Agent Skills specification | Name, description, and path added to the system prompt at startup | `disable-model-invocation`; forced via `/skill:name` with arguments appended | Model judgement; explicit `/skill:name` overrides |
| **Hermes** | `SKILL.md` folder; agentskills.io spec | Progressive disclosure: name and description first, body on use | Skills Hub registries, `/learn` to author from sources, agent-managed and agent-editable skills, external directories and skill bundles | Model judgement; hub-side security scanning on install |

Read that table as a set of design choices rather than a ranking. The interesting axis is *who is responsible when the model forgets a skill exists*.

**Claude Code, OpenCode, Codex, and Pi** all put the skill's name and description in front of the model and trust the model to reach for it. That is a good design when the library is small and the descriptions are sharp, and all four invest heavily in making descriptions good enough to route on — Codex caps its initial list by context share, Claude Code truncates long descriptions, OpenCode exposes an auto-invoke toggle. But the discovery mechanism is the model's own matching. If the model does not recognize the task as a skill's job, the skill never loads, and there is nothing in the harness that catches the miss.

**Hermes** goes further on the authoring and distribution side — a public Skills Hub, security scanning on install, an agent that can write and edit its own skills, and `/learn` to turn reference material into a skill. The skill surface is managed and shared. But like the others, a skill is reached through description matching; the hub is a distribution channel, not a retrieval index.

**SCORPIOX CODE** makes two choices the others do not. First, discovery is a *tool* — a real grep over the skill files that returns literal matches, so the agent finds a skill by searching for the words you actually wrote rather than hoping its description lands. Second, the listing is a *policy* you set, with a guarantee layer underneath: required skills are tracked, nudged, and can be force-injected, so a must-use procedure does not depend on the model's judgement at all. Other harnesses let you scope or hide a skill; SCORPIOX CODE lets you require one.

There is a cost to being different here, and it is worth naming. The catalog, when you leave it fully on, is bytes on every request — a real cost that grows with the library, which is exactly why the listing controls and lazy mode exist. A harness with a fixed context budget pays a bounded price by construction; SCORPIOX CODE hands you the dial and expects you to set it. The defaults list everything and inject required skills into the system prompt, which is the safe starting point; tuning down from there is where the savings are.

---

## Gotchas worth knowing

- **Folder name is the identity.** A skill whose `name:` field disagrees with its folder name is confusing to everyone including the agent. Keep them equal.
- **Shadowing is silent and total.** A project skill replaces a user or built-in skill of the same name with no warning. If a built-in procedure suddenly behaves differently in one repo, check for an override.
- **The required-skills cascade unions; the skill-folder cascade replaces.** These two cascades behave oppositely on purpose. Adding a required skill at project level does not remove the user-level one — but adding a project skill file *does* hide the user-level skill of the same name.
- **An empty `TOOL_SEARCHSKILLS=` line counts as enabled.** The gate treats any stored value other than `0` as on. Write `TOOL_SEARCHSKILLS=0` to turn lazy mode off, or comment the line out.
- **Hiding a skill does not remove it.** If you exclude a skill from the listing, it is still reachable by name and by search. If you want it genuinely unavailable, use `/disable_skill`, not a filter.
- **The list is read when the tool set is assembled.** A change to the listing keys takes effect on a new session, not mid-session.
- **The catalog is a standing cost.** If your library is large and your sessions are long, a fully-on catalog is bytes on every turn. That is what the filters and lazy mode are for.
- **Write descriptions for the search, not just for the eye.** A description is both the routing hint and a search target. The words you put there are the words a future search will match.

---

## The bottom line

Skills in SCORPIOX CODE come down to three commitments:

1. **Two tools, not one list.** Loading and discovery are separate. The catalog can be tuned to any size, and a real grep search stands behind it so a hidden skill is a search away, never lost.
2. **A tunable catalog.** Listing is a policy — include, exclude, top-N% by usage, names-only — and listing is never loading. Filtering saves tokens without removing capability.
3. **A guarantee underneath.** Required skills are tracked from a four-level cascade, nudged at end of turn, and can be force-injected or written straight into the system prompt. Must-use procedures do not depend on the model noticing them.

Where other harnesses ask the model to remember that a skill exists and trust its judgement to reach for it, SCORPIOX CODE makes reachability a property of the architecture: advertise it or index it, and if it is required, enforce it.

---

## Related

- [Lazy Skill Loading](lazy-skill-loading.md) — the catalog switch turned all the way down: `TOOL_SEARCHSKILLS`, the discovery tool in depth, and when lazy mode pays.
- [Project Instructions in SCORPIOX CODE](project-instructions.md) — the standing counterpart to skills: facts that belong in every turn versus procedures loaded on demand.
- [Configuration and Profiles](scorpiox-env.md) — the cascade that every listing and required-skills key is read through.
- [Native MCP 2.0 and OAuth 2.1](mcp.md) — a different on-demand surface, including the built-in skill that teaches the agent to prefer these MCP commands.
