# Lazy Skill Loading

SCORPIOX CODE ships two shapes for the same skill surface. In the default shape the agent sees a **catalog**: every available skill's name and one-line description are baked into the `InvokeSkill` tool description on every request, and the skill body loads only when a skill is actually invoked. In the lazy shape the catalog is dropped — the agent sees a short pointer telling it to *search* for skills instead, and discovery happens through a second tool, **`SearchSkills`**, on demand.

The whole switch is one config key: **`TOOL_SEARCHSKILLS`**.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** set `TOOL_SEARCHSKILLS=1` and the standing skill list disappears from the prompt; the agent finds skills by grep when it needs them, at the cost of one extra tool call the first time it goes looking.

---

## What goes into the prompt in each mode

The difference is entirely inside the `InvokeSkill` tool description — the part of the tool definition the model reads every turn. Nothing else about how skills load, resolve, or run changes.

**Default mode (`TOOL_SEARCHSKILLS=0`)** — the description carries the catalog:

```
Invoke or load a skill by name. Returns the skill content for
you to follow or use as context. Available skills:
- create-command: Checklist for creating a new /slash command...
- create-skill: Checklist for creating a new skill folder...
- release-notes: Ship a release with changelog. Use when asked to cut a release.
...one line per available skill...
Skill folders (<name>/SKILL.md; project overrides user; built-ins have no file): ...
```

One `- name: description` line per skill, in a deterministic order, followed by the skill folders so the agent knows where skills physically live. The cost is flat per skill and grows with your library: a description line is small, but 200 skills is 200 lines that ride along on every single request, cached or not.

**Lazy mode (`TOOL_SEARCHSKILLS=1`)** — the catalog is replaced by a pointer. With no other listing rules in play and no built-in skills to show, the description becomes exactly:

```
Invoke or load a skill by name. Returns the skill content for
you to follow or use as context. Use the SearchSkills tool first
to discover available skills by keyword.
```

That is 167 bytes, and it is the whole description. The names and descriptions of your project, user, and additional skills leave the prompt.

In practice built-in skills — the ones compiled into the binary — stay listed, so the agent keeps a floor of always-visible procedures. When some skills are listed and others are not, the description carries the built-ins, then a hint naming how many are hidden, then the skill-folder paths:

```
...one line per built-in skill...
(+18 more skills not listed - use SearchSkills to discover them)
Skill folders (<name>/SKILL.md; project overrides user; built-ins have no file): /home/you/.claude/skills, /repo/.claude/skills
```

So even in lazy mode the agent knows a larger library exists behind the search, and where it lives, without any of it being enumerated.

Everything stays invokable. `InvokeSkill release-notes` works in lazy mode exactly as it does in catalog mode — the list controls only what is *advertised*, never what is *loadable*.

### When lazy mode saves tokens, and when it costs

| Situation | Lean toward |
|-----------|-------------|
| Large library (dozens of skills), long sessions, prompt-cache-sensitive endpoints | Lazy |
| Small library (a handful of skills you use constantly) | Default |
| Skills the agent must reliably reach for *unprompted* | Default, or lazy plus the required-skills contract |
| Skills invoked only when a task explicitly calls for them | Either; lazy saves the standing bytes |

The trade is a standing cost against a per-task cost. In catalog mode the list is paid on **every** request, including the many turns where no skill is relevant. In lazy mode you pay a **discovery round-trip** once per unfamiliar task: the model calls `SearchSkills`, reads the matches, then calls `InvokeSkill`. That round-trip is small — a grep is cheap and the result is a compact list of names and descriptions — but it is a turn of latency the catalog mode never spends, and it depends on the model actually *thinking* to search.

Lazy mode also has a quiet cache benefit. Tool definitions are part of the prompt the provider caches, so a big catalog raises the baseline size of every cached request. Shrinking the standing tool surface is one of the few ways to make a large skill library *and* a warm cache coexist: adding or editing your own skills no longer rewrites the `InvokeSkill` description, so the cached prefix stays valid across a fast-moving, team-owned library.

---

## Enabling lazy mode

One line, in any cascade tier:

```ini
# .scorpiox/scorpiox-env.txt  (or ~/.claude/scorpiox-env.txt, or a profile)
TOOL_SEARCHSKILLS=1
```

Or for a single run, without touching a file:

```bash
sx -e TOOL_SEARCHSKILLS=1
```

The shipped `scorpiox-env.txt` sets `TOOL_SEARCHSKILLS=0`, the compiled-in default is `0`, and the tool definition itself is marked disabled-by-default — so lazy mode is strictly opt-in. Set the key to `1` in the tier that matches how widely you want it.

One sharp edge: the gate treats *any* stored value other than `0` as enabled, which means an explicitly empty `TOOL_SEARCHSKILLS=` line in a config file counts as **enabled**, not off. Write `TOOL_SEARCHSKILLS=0` to turn it off, or comment the line out.

| You want lazy mode for… | Set `TOOL_SEARCHSKILLS=1` in |
|-------------------------|------------------------------|
| This project only | `.scorpiox/scorpiox-env.txt` (or `.claude/scorpiox-env.txt`) in the repo |
| Every project on this machine, for you | `~/.claude/scorpiox-env.txt` |
| Everyone using this install | `scorpiox-env.txt` next to the installed binaries |
| A named profile you flip with `/profile` / `/use` | `scorpiox-env/<name>.txt` |

As with every key, an OS environment variable outranks all file tiers, and the highest tier that sets the key wins — so a profile can turn lazy mode on for a big-library project while your everyday default stays catalog mode. See [Configuration and Profiles](scorpiox-env.md) for the cascade and the `/profile` vs `/use` semantics.

Verify what actually resolved with:

```bash
scorpiox-config --get TOOL_SEARCHSKILLS     # the effective value
scorpiox-config --json TOOL_SEARCHSKILLS    # plus the tier and file it came from
```

The key is read when the tool list is assembled at the start of a session, so a new session picks up a change immediately.

One honesty note about defaults. The **Tools** section of the interactive config editor shows `TOOL_SEARCHSKILLS` behind a dependency on `TOOLS` — the editor only reveals the row when the `TOOLS` master switch is on, and the `1` stored alongside that rule describes visibility, not a suggested value. It is not a default of `1`. Everything that decides behavior at runtime — the built-in defaults, the shipped `scorpiox-env.txt`, and the tool's own disabled-by-default flag — says `0`. If you deliberately start from a stripped-down config with no entry for this key, trust `scorpiox-config --get TOOL_SEARCHSKILLS`: the runtime answer is off unless something set it to `1`.

---

## SearchSkills: the discovery tool

`SearchSkills` is the tool that replaces the catalog. It takes two parameters:

| Parameter | Required | What it does |
|-----------|----------|--------------|
| `pattern` | Yes | Grep search pattern matched against skill file content, names, and descriptions. Fixed strings by default, regex with `-E`. An empty string lists everything. |
| `flags` | No | Optional flags passed to the grep: `-i` (case insensitive), `-E` (regex), `-w` (whole word). Example: `'-i -w'` or `'-E -i'`. Default: `-i`. |

### What the search actually does

A non-empty pattern shells out to the bundled grep binary in read-file-list mode: `scorpiox-grep -rl <flags> '<pattern>'`, run over the user skills folder, the project skills folder, and any additional skills folders, with a 15-second timeout. The grep reports matching file paths, and the tool turns each path into a skill name — the directory containing the file — deduplicates, and also sweeps the built-in skills in memory (they have no file on disk, so they are matched directly against name, description, and any loaded content).

The result is a compact JSON list, one entry per matching skill:

```json
{"skills":[{"name":"release-notes","description":"Ship a release with changelog...",
            "source":"Project Skill","path":"/repo/.claude/skills/release-notes/SKILL.md"}],
 "matched":3,"total":27}
```

Three fields are worth reading closely. `source` tells you where the skill came from (`Built-in Skill`, `Project Skill`, `User Skill`), which is how you learn that a project skill is shadowing a same-named user skill. `path` is the on-disk location — empty for built-ins, which live in the binary. `total` is the full library size, so the agent always knows how much it is not seeing.

### Listing everything: the empty pattern

Pass an empty `pattern` and the grep is skipped entirely; the tool returns the full library — every skill's name, description, source, and path. This is the "show me the menu" call, and it is the lazy-mode replacement for the catalog that used to sit in the prompt: same information, delivered once, when asked for, instead of on every request.

A keyword search that matches many skills returns at most 128 of them, in the order the grep reports the files; the empty-pattern listing enumerates every skill with no such cap. The JSON payload has a hard ceiling of one megabyte — far above any realistic library — so a single tool result still carries the whole answer.

In practice the flow looks like this:

```
SearchSkills  pattern=""                     -> the whole library (names + descriptions)
SearchSkills  pattern="release" flags="-i"   -> the subset that matters
InvokeSkill   skill_name="release-notes"     -> the body, now in context
```

### The flags, precisely

- **`-i` is the default.** With no flags at all the search runs case-insensitively with a fixed-string pattern — the friendliest behavior, and the right call for keywords.
- **`-E` switches to regex**, so `SearchSkills pattern="release|deploy" flags="-E"` finds skills touching either topic in one call.
- **`-w` requires whole-word matches**, which stops a search for `deploy` from lighting up every skill whose body mentions `deployment`.
- Flags combine (`-i -w`, `-E -i`) and are passed through to the grep. Anything outside the `-i` / `-E` / `-w` whitelist is silently dropped rather than executed, so a malformed flag degrades to a normal case-insensitive search instead of erroring out or running something unexpected. A combined flag like `-iE` is expanded to its parts before the whitelist check, so it survives too.

Searches time out at 15 seconds, which is generous for a recursive grep over skill folders; if a skills directory lives on a slow network mount, keep it small or move it local.

---

## How lazy mode interacts with everything else

**List filters and the auto-enable.** The `SKILL_LIST_*` keys filter the catalog independently of this switch — `SKILL_LIST_INCLUDE` / `SKILL_LIST_EXCLUDE` (plus their `_FILE` variants), `SKILL_LIST_TOP_PERCENT` to keep only the most-used skills, and `SKILL_LIST_STYLE=names` to advertise names without descriptions (a style, not a filter — it does not hide anyone). Filtering and lazy mode are two different answers to the same problem: filters keep a *reduced* catalog always visible; lazy mode removes the catalog and substitutes search.

The two mechanisms meet in one place: **the moment the visible list is partial — some skills exist but are not advertised — `SearchSkills` is enabled.** The auto-enable is in-memory for the running session and is never written back to your config files, so a filter can turn the tool on for a session without permanently flipping the key. Required skills and built-in skills are exempt from filtering and always stay listed, so a filter can never hide the things a session is contractually obliged to load.

**Required skills stay safe.** Lazy mode hides the catalog, but the required-skills contract is not a catalog concern — it is enforced by nudges that name each missing required skill explicitly at the end of a turn. Turn the catalog off and a required skill that was never advertised still gets demanded by name. You keep the token savings of lazy mode and the guarantee of the contract at the same time.

**WASM builds.** `SearchSkills` shells out to a grep process, which a single-threaded browser build cannot do. In WASM the tool is disabled automatically — as is `InvokeSkill`, whose scripts also shell out — so browser sessions run without the skill tools rather than with half-broken ones.

---

## Summary

| | Catalog mode (default) | Lazy mode (`TOOL_SEARCHSKILLS=1`) |
|---|---|---|
| What the prompt carries | Every skill's name + description, every request | A 167-byte pointer to search |
| Discovery | Already visible | One `SearchSkills` call when needed |
| Cost profile | Standing bytes per request, growing with the library | One discovery round-trip per unfamiliar task |
| Prompt cache | Big tool surface rewrites the cached prefix when the library changes | Stable prefix; the library can move under you |
| Best for | Small, hot libraries | Large libraries, cache-sensitive endpoints |

Both modes share the same floor: every skill remains invokable by name, and required skills are guaranteed to load regardless.

See [Skills System](skills-system.md) for the full architecture — folders, cascade, filters, and the required-skills contract that keeps lazy mode safe.
