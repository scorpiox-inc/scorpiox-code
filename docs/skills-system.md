# Skills in SCORPIOX CODE: On-Demand Architecture vs Other Harnesses

A **skill** is a folder of instructions you write once and the agent loads only when the task calls for it: a checklist, a multi-step procedure, a house convention you keep re-explaining, or a reference you keep pasting into chat. Drop the folder into the right place and it is available. Nothing else to wire up.

The design decision everything follows from is **on-demand by default**. The agent's context carries only each skill's *name and description* as a compact index. The full instructions are pulled in **only when the skill is actually used**, through an explicit tool call you can see in the transcript. A fifty-page reference costs almost nothing until the moment it is needed.

SCORPIOX CODE keeps that on-demand core and adds two layers the other harnesses in this comparison do not have: a **required-skills contract** that can guarantee a load (nudge, auto-inject, or system-prompt), and a **list filter** that lets you size the index without ever hiding a skill from discovery.

Docs for SCORPIOX CODE @ `ad926d7`.

> **The whole idea in one line:** a skill is a `SKILL.md` in a folder that costs you nothing until you use it. The agent sees only its name and description, calls `InvokeSkill` to load the body on demand, and finds skills it was not shown with `SearchSkills` — and if a skill is *required* for the session, SCORPIOX CODE makes sure it gets loaded, not just made available.

---

## What a skill is

A skill is a directory whose name is the skill name, containing a `SKILL.md`. The file has a small YAML frontmatter block on top and a markdown body below it:

```
.claude/skills/
└── git-release/
    └── SKILL.md
```

```markdown
---
name: git-release
description: Draft release notes, propose a version bump, and create a release. Use when preparing a tagged release.
argument-hint: [tag]
---

Draft the release notes from the changes since the last tag.

1. Read the CHANGELOG for the previous version.
2. Summarize what changed.
3. Use `$ARGUMENTS` as the target tag.
4. Show the command to create the release.
```

Three frontmatter fields matter:

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | The skill identifier (kebab-case). Should match the folder name. |
| `description` | Yes | One line on *what it does* and *when to use it*. This is the only part the agent sees before deciding to load the body, so it is the most important line you will write. |
| `argument-hint` | No | A hint for what `$ARGUMENTS` carries when the skill is invoked. |

Three properties of the format are load-bearing:

- **The frontmatter is the routing signal.** The `description` is what the agent matches your request against. Write it to say both *what* the skill does and *when* to reach for it; a vague description is the difference between a skill the agent uses and one it ignores.
- **The body loads only on use.** Until the skill is invoked, only the name and description are in the agent's context. The full instructions enter the conversation at the moment of invocation, not at startup. Because a loaded skill stays in context for the rest of the work, keep the body concise.
- **`$ARGUMENTS` is substituted.** Whatever arguments you (or the agent) pass replace every `$ARGUMENTS` placeholder in the body. That is what turns a static file into a parameterized procedure. A skill with no arguments simply has no `$ARGUMENTS` token and nothing changes.

A skill folder can also bundle anything else the procedure needs: reference docs, templates, scripts. The agent reads or runs those files from disk when the instructions point at them. This is progressive disclosure end to end: metadata always present, instructions on trigger, supporting files only when referenced.

Two shapes coexist under the same name: a **skill** is a folder with `SKILL.md`, and a **command** is a script in `.claude/commands/` (`.sh` / `.ps1` / `.md`). Both surface as `/<name>`. The rest of this page is about skills.

---

## Where skills live, and which one wins

SCORPIOX CODE discovers skills from a fixed set of locations and unions them. When the same skill name exists in more than one place, the higher-priority copy wins and the lower one is shadowed. Nothing is ever concatenated.

| Priority | Location | Scope |
|----------|----------|-------|
| 1 | `./.claude/skills/<name>/SKILL.md` | Project-local (this repo) |
| 2 | `~/.claude/skills/<name>/SKILL.md` | User-global (this machine) |
| 3 | Any directory listed in `ADDITIONAL_SKILLS` | Shared / versioned skill packs |
| 4 | Built-in skills | Compiled into the binary, always present |

A few properties of this layout matter in practice:

- **Project beats user, user beats shared, shared beats built-in.** Put a `git-release` skill in your repo and it overrides a same-named one in your home folder for every session run from that repo. Delete the project copy and the home one reappears.
- **It reads the cross-harness folders.** SCORPIOX CODE looks for `.claude/skills/` and, only where that directory is absent at a given level, falls back to `.agents/skills/` — the folder the emerging Agent Skills standard uses. A skill pack you maintain for another tool is picked up here unchanged, and a pack written here is portable out. The fallback is per level and never merges the two trees; `.claude/` always wins when present at that level.
- **`ADDITIONAL_SKILLS` is how teams share a skill pack.** It is a comma-separated list of local directories. Point it at a git checkout and every session in the workspace sees that shared set. `/skills-update` runs a fast-forward `git pull` on each of those directories so the pack stays current.
- **Built-in skills are the floor.** A handful of skills ship inside the binary itself (the create/edit authors, platform file-tool preferences, and a few convenience skills). They are always available, and they sit at the lowest priority, so the moment you create a same-named skill on disk, yours is the one that loads.

The default guidance: write skills for *this* codebase in `./.claude/skills/`, keep cross-project or personal ones in `~/.claude/skills/`, and use `ADDITIONAL_SKILLS` for anything your team maintains as a shared, versioned set.

---

## How a skill reaches the model, and only then

This is the part that defines the architecture. A skill does not sit in the system prompt as a wall of text. It reaches the model in two stages, and the heavy stage only happens on a tool call.

**Stage 1: the catalog is always in view.** The agent is given, alongside its tools, a compact list of available skills — each one's name and description. That list is part of the `InvokeSkill` tool description. It is small by design, and when it grows you can filter what is shown (see below). The catalogue is cache-aware: it is built in a deterministic order and only rescanned when a skill directory actually changes, so adding a skill does not silently invalidate a large prompt.

**Stage 2: loading is an explicit tool call.** Two tools do the work:

- **`InvokeSkill`** — given a skill name (and optional arguments), it loads the full `SKILL.md`, substitutes `$ARGUMENTS`, and returns the instructions for the agent to follow. The result opens with a location header naming the skill, where it came from, and the path it lives at — so the agent (and you) can see exactly which copy was loaded. Built-in skills are tagged as compiled into the binary with no file, and the header tells you where to create an override. If the skill was created mid-session and is not yet in the catalog, the tool falls back to reading it straight from disk, so a freshly written skill is usable immediately.
- **`SearchSkills`** — a grep-powered search across skill names, descriptions, and file contents. Pass a keyword to find a skill by what it is about; pass an empty pattern to list everything. It is the discovery tool for when you do not already know the skill's name, and it supports the usual grep flags (`-i` case-insensitive by default, `-E` regex, `-w` whole word). It ships off by default and is turned on automatically the moment the catalog is filtered, so hidden skills are always discoverable.

So the flow is: *see the catalog, optionally search, then invoke.* The full instructions occupy context only after `InvokeSkill` returns. This is progressive disclosure with an explicit, inspectable seam: every load is a tool call in the transcript, not an invisible background file read.

`InvokeSkill` and `SearchSkills` are also independently switchable, and both are disabled automatically in the browser build, where there is no shell to read the folders or run the search.

---

## Required skills: making a load a guarantee

Listing a skill and hoping the agent loads it is not enough when a task genuinely cannot be done right without a particular procedure. SCORPIOX CODE has a second layer, and it is why "on-demand" does not mean "optional."

You declare a set of *required* skills in a plain text file — one skill name per line — and the set is the union of a small cascade:

```
~/.claude/required_skills.txt                     (user level)
./.claude/required_skills.txt                     (project level)
./.scorpiox/required_skills.txt                   (project level)
./.scorpiox/sessions/<id>/required_skills.txt     (this session only)
```

The cascade runs from user level down through the project and then to the session, so a session can pin a skill without editing a shared file, and a repo can pin one for everyone. The agent only ever writes to the per-session file; the shared levels are read-only.

SCORPIOX CODE tracks which required skills have actually been loaded during the session — from every `InvokeSkill` call and from anything folded into the system prompt — and closes the gap in one of three modes, selected by configuration:

- **Nudge (the default enforcement).** Before the agent acts, SCORPIOX CODE checks whether any required skill is still missing. If so, it sends the agent a system message naming the missing skills and telling it to load them with `InvokeSkill`. The agent makes the call itself. A soft guarantee: the skill gets loaded, but the model still does the loading.
- **Force-inject.** With `REQUIRED_SKILLS_FORCED` on, every missing required skill is loaded for the agent and its full content is written directly into the conversation as a single system message, one clearly delimited section per skill. There is no round-trip and no chance the model skips it. This is the hard guarantee.
- **System-prompt injection.** With `REQUIRED_SKILLS_IN_SYSTEM_PROMPT` on, the raw content of the required skills is folded into the system prompt at session start. The agent has them from the very first turn and does not need to invoke anything at all. `SYSTEM_PROMPT_SKILLS` adds extra names to this set that are always injected, whether or not they are in a required-skills file.

The master switch is `REQUIRED_SKILLS`. Turn it off and the whole enforcement layer is inert — skills become purely on-demand. Turn it on and you get a catalog you can trust: the procedures your task depends on are in the model's hands, by nudge, by injection, or both.

This property is the clearest divide in the comparison at the end of this page. The other harnesses make skills *available* and leave the loading to the model's judgment. SCORPIOX CODE can make them *mandatory*.

---

## Controlling what the catalog shows

A workspace that has accumulated a few hundred skills will not fit a few hundred name-and-description lines into every prompt without crowding out the work. The `SKILL_LIST_*` settings trim the *list* without hiding the *skills*:

- **`SKILL_LIST_INCLUDE` / `SKILL_LIST_EXCLUDE`** — comma-separated glob patterns to keep or drop skills from the list by name.
- **`SKILL_LIST_INCLUDE_FILE` / `SKILL_LIST_EXCLUDE_FILE`** — the same patterns, one per line, read from a file.
- **`SKILL_LIST_TOP_PERCENT`** — list the top N percent of skills by usage, ranked by how often each appears in past sessions' required-skills files. Built-in skills and required skills are always listed, so an important skill cannot be filtered out by a thin usage history.
- **`SKILL_LIST_STYLE`** — `full` (name plus description, the default) or `names` (names only, for a tighter index).

A dedicated filter log records exactly what was listed and why, so a filtered catalog can always be diagnosed after the fact.

Two invariants keep this safe. First, filtering the *list* never filters the *skills*: a skill removed from the catalog is still loadable by `InvokeSkill` and still findable by `SearchSkills`. Second, the moment the list becomes partial, SCORPIOX CODE enables `SearchSkills` for that run and appends a hint that more skills exist and how to find them. You are choosing what is *visible*, not what is *possible*.

---

## Built-in skills

The binary ships a small set of skills that are always present. They are what make SCORPIOX CODE useful on day zero without you writing a single file: the `create-skill` and `edit-skill` authors, `create-command` and `edit-command` for scripts, platform-specific "preferred tools" skills that steer the agent toward the right read/search/edit commands, a preview skill that publishes files to the chat Preview popup, an offline help skill, a fast-clone context skill, and a couple of convenience skills.

Three rules govern them:

- **They sit at the bottom of the priority stack.** The instant you create a same-named skill on disk, your version loads and the built-in one is shadowed. Overriding a built-in skill is just "write a skill with the same name."
- **Each can be disabled individually.** A built-in skill maps to a `SKILL_<NAME>` configuration key. Set it to `0` and that skill stops appearing in the catalog for your profile, without touching the others.
- **They are marked as such when loaded.** When the agent invokes a built-in skill, the result says it is compiled into the binary and not editable on disk, and points at the `.claude/skills/<name>/SKILL.md` path you would create to override it.

The point of a built-in skill is to make a *procedure* available without a *file*. The point of a disk skill is to make *your* procedure available without a *prompt*. Both load through the same `InvokeSkill` path and participate in the same required-skills enforcement.

---

## Managing skills at the session

A few slash commands give you direct control over the skill set while a session is running, with no config edit or restart:

- **`/skills`** — opens a popup listing every loaded skill, where each one came from (user, project, additional, or built-in), and its description. The fastest way to see what the agent can actually reach right now.
- **`/disable_skill <name>`** — blocks a skill for the rest of the session. The skill stays in the catalog (so the list stays stable and the prompt cache stays warm), but loading it returns an error. A safe way to take a known-bad skill out of play without deleting a file.
- **`/enable_skill <name>`** — reverses a `/disable_skill`.
- **`/skills-update`** — runs a fast-forward `git pull` on each `ADDITIONAL_SKILLS` directory so a shared pack updates in place.

On disk, the everyday operations are covered by the built-in skills themselves: ask the agent to use `create-skill` to add one, `edit-skill` to change one, or their command counterparts, and it creates the folder, writes the frontmatter, and keeps the format valid. A skill's `description` is kept to a short single line (roughly 255 characters) because that is what fits in the catalog.

---

## How SCORPIOX CODE compares to other harnesses

The on-demand skill is now a shared idea. Claude Code, OpenCode, Codex, Pi, and Hermes all converge on the same `SKILL.md` folder and the same progressive-disclosure principle — many of them implementing the open Agent Skills standard. The difference is what each *harness* does around that file, and it is a wide range.

| | **SCORPIOX CODE** | **Claude Code** | **OpenCode** | **Codex** | **Pi** | **Hermes** |
|---|---|---|---|---|---|---|
| **What stays in context** | Catalog: name + description, in the `InvokeSkill` description | Skill name + description index | Name + description in the `skill` tool description | Name + description, plus always-on `AGENTS.md` | Name + description + path in the system prompt | Names, descriptions, categories (Level 0) |
| **How a body loads** | Agent calls `InvokeSkill` (or `SearchSkills`, then invoke) | Model reads the file when the description matches, or you type `/skill-name` | Model calls the native `skill` tool with the exact ID | Model matches by description, or you name it explicitly | Model reads `SKILL.md` when matched; `/skill:name` forces it | Agent calls `skill_view(name)` |
| **Discovery for unlisted skills** | `SearchSkills` — grep over names, descriptions, and contents | None dedicated; description matching | The `skill` tool is both list and loader | None dedicated; description matching with explicit mention as fallback | Model matching; `/skill:name` force command | `skills_list` then `skill_view` |
| **Can a skill be *required*?** | **Yes** — `required_skills.txt` cascade, then nudge, auto-inject, or system-prompt injection | No equivalent; loading is at the model's discretion | No equivalent | No equivalent | No equivalent | No equivalent |
| **Filter the list without hiding the skill** | **Yes** — `SKILL_LIST_*`, hidden skills stay invokable and searchable | No | No | Budgets the list to a slice of the context window | No | No |
| **Cross-harness folders** | `.claude/skills/`, falling back to `.agents/skills/` per level | Native `.claude/skills/` and user folder | `.opencode/` plus Claude- and agent-compatible folders | Its own locations, scanning `.agents/skills` up to the repo root | Native folders plus `.agents/skills/` | `~/.hermes/skills/` plus external dirs |
| **Sharing / distribution** | `ADDITIONAL_SKILLS` git packs; `/skills-update` pulls them | Plugins and skill files | Plugins and per-skill permissions | Plugins distribute skills through a shared directory | Pi packages via npm or git | Skills Hub, plugins, `/learn` |
| **Built-ins** | A few compiled into the binary, individually disable-able, overridable on disk | Bundled skills (`/code-review`, `/debug`, ...) | None shipped | Plugins supply skills | None shipped | A bundled catalog you can opt out of |
| **Runtime / install** | None — plain files, no runtime to install | None for instructions; scripts run in the VM | None for instructions | None for instructions | None for instructions | None for instructions; hub installs are a CLI action |

### What this actually means in practice

- **The load is a visible tool call, not a silent read.** In Claude Code, OpenCode, Codex, Pi, and Hermes, the model reads `SKILL.md` on its own initiative when a task looks like a match. In SCORPIOX CODE the same thing happens, but it goes through `InvokeSkill` (with `SearchSkills` for when the name is unknown), so every load is a step you can see, audit, and reason about in the transcript — and the result tells you which copy, from which folder, was loaded.
- **"Required" is a first-class concept here and absent everywhere else.** This is the architectural divide. The other harnesses make skills *available*; the model decides whether to use them, and a critical procedure can be skipped if the description did not match well enough. SCORPIOX CODE lets you declare that a set of skills *must* be in context for a session and gives three enforcement strengths — nudge, force-inject, system-prompt — so a procedure your work depends on is guaranteed to load, not merely offered.
- **The catalog is a thing you can size.** SCORPIOX CODE gives you explicit include/exclude/top-percent/style controls over the *list* while keeping every skill invokable, and it turns on search automatically the moment anything is hidden. That combination — trim the visible index, never shrink the reachable set — is not something the others offer.
- **It interoperates by reading the same folders.** Because SCORPIOX CODE reads `.claude/skills/` and `.agents/skills/`, a skill pack written for Claude Code or the Agent Skills standard works here without modification, and a pack written here is portable out. You are not locked into a private format.
- **Hermes is a different instrument.** It is not primarily a per-skill loading system at all; it is the self-improving agent that *creates* skills from experience, improves them during use, and carries persistent memory across sessions, with a documented three-level progressive-disclosure pipeline and a `/learn` command that turns a book or a described procedure into a knowledge-base skill. The closest SCORPIOX CODE analog is skills plus scheduled callbacks — but the design goals differ (Hermes curates its own skills; SCORPIOX CODE enforces the ones you declare), so it belongs in a separate conversation.
- **The formats agree; the control plane does not.** Every harness here reads the same `SKILL.md` shape, so the portability argument is settled. The choice is what sits *above* the file: SCORPIOX CODE's required-skills enforcement and filterable catalog are the pieces you get only here.

---

## The bottom line

A skill in SCORPIOX CODE is the simplest correct answer to "give the agent a reusable procedure without bloating its context": a `SKILL.md` in a folder, metadata always in view, full instructions only on an explicit `InvokeSkill`, and a `SearchSkills` tool for when the name is not known. Built-ins make it useful on day zero; the cross-harness folders make your existing skill packs portable; `SKILL_LIST_*` keeps the catalog the size you want without ever hiding a skill from discovery.

On top of that shared, on-demand base, SCORPIOX CODE adds the piece the others do not: a **required-skills** layer that can *guarantee* a load, by nudging the agent, auto-injecting the content, or folding it into the system prompt. On-demand means a skill costs nothing until it is used; required means the skill that matters is the one that *stays*.

---

## See also

- [Project Instructions](project-instructions.md)
- [Event Hooks](hooks-system.md)
- [Scheduled Callbacks](callbacks.md)
- [Conversation Compaction](conversation-compaction.md)
- [Configuration and Profiles](scorpiox-env.md)
