# Lazy Skill Loading

SCORPIOX CODE ships two shapes for the same skill surface. In the default shape the agent sees a **catalog**: every available skill's name and one-line description is baked into the `InvokeSkill` tool description on every request, and the body loads only when a skill is invoked. In the lazy shape the catalog is dropped — the agent sees a short pointer telling it to *search* for skills instead, and discovery happens through a second tool, **`SearchSkills`**, on demand.

The whole switch is one config key: **`TOOL_SEARCHSKILLS`**.

Docs for SCORPIOX CODE @ `e30b171`.

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

**Lazy mode (`TOOL_SEARCHSKILLS=1`)** — the catalog is replaced by a pointer. When there are no other listing rules in play and no built-in skills to show, the description becomes exactly:

```
Invoke or load a skill by name. Returns the skill content for
you to follow or use as context. Use the SearchSkills tool first
to discover available skills by keyword.
```

That is 167 bytes, and it is the whole description. The names and descriptions of your project, user, and additional skills leave the prompt.

In practice built-in skills — the ones compiled into the binary — stay listed, so the agent keeps a floor of always-visible procedures (`create-skill`, `edit-skill`, `preferred-*`, and friends). When some skills are listed and others are not, the description carries the built-ins, then a hint naming how many are hidden, then the skill-folder paths:

```
...one line per built-in skill...
(+18 more skills not listed - use SearchSkills to discover them)
Skill folders (<name>/SKILL.md; project overrides user; built-ins have no file): /home/you/.claude/skills, /repo/.claude/skills
```

So even in lazy mode the agent knows a larger library exists behind the search, and where it lives, without any of it being enumerated.

Everything stays invokable. `InvokeSkill deploy` works in lazy mode exactly as it does in catalog mode — the list controls only what is *advertised*, never what is *loadable*.

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

`SearchSkills` is what makes lazy mode work. It is a real grep — the product's own zero-dependency `scorpiox-grep` binary, invoked with `-rl` (recursive, filenames only) — run across the skill directories: your user skills folder, the project skills folder, and every additional skills directory you have configured. Built-in skills have no file on disk, so they are matched in memory against their names, descriptions, and loaded content instead.

| Parameter | Required | What it does |
|-----------|----------|--------------|
| `pattern` | Yes | Text to match against skill file contents, names, and descriptions. **An empty string lists every skill.** |
| `flags` | No | `-i` (case-insensitive, the default), `-E` (regex), `-w` (whole word). Examples: `-i -w`, `-E -i`. |

It is grep-backed, not model-backed: matches are literal text inside your skill files, so it finds the thing you actually wrote rather than guessing at what a skill might be about. That makes it a reliable index into your own prose — write your `SKILL.md` descriptions and bodies with the vocabulary a future task will use, and the search finds them.

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

**Filtering and lazy mode are alternatives, not ingredients.** If `SKILL_LIST_*` filters are active they shape the catalog, and `SearchSkills` auto-enables for the hidden remainder. Setting `TOOL_SEARCHSKILLS=1` on top collapses the catalog to the built-ins. Both roads end at the same place — search as the path to everything not listed — so pick one and let it do the work.

**Required skills still work.** The required-skills cascade (`required_skills.txt` at user, project, and session level) is unchanged in lazy mode: a required skill that has not been loaded still triggers the end-of-turn nudge naming it, and force-inject still puts its body straight into context. If you go lazy, this is the mechanism that keeps must-use skills guaranteed — the catalog no longer advertises them, but the nudge still names them and tells the agent to load them. See [Skills in SCORPIOX CODE](skills-system.md) for the full contract.

**Prompt-cache stability.** In catalog mode, adding or renaming a skill rewrites the `InvokeSkill` description and can invalidate the cached prompt prefix; the loader already minimizes that by rescanning only when a skill directory's mtime changes. Lazy mode removes most of that churn: the description shrinks to a static pointer plus a built-in index, so adding or editing your own skills leaves the tool JSON untouched.

**Where the feature does not reach.** The browser/WASM build cannot run `SearchSkills` — the grep is a subprocess and the sandbox has no processes — so the tool is force-disabled there and lazy mode has nothing to fall back on. The catalog is the only discovery surface in the browser. And `/disable_skill` behaves the same in both modes: the skill stays advertised (so the list stays stable), and loading it returns an error for the rest of the session.

---

## Configuration reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `TOOL_SEARCHSKILLS` | bool | `0` | Enable lazy skill loading and the `SearchSkills` discovery tool. `1` removes the skill catalog from the `InvokeSkill` description and points the agent at search instead; built-in skills stay listed. Any value other than `0` enables. |
| `SKILL_LIST_INCLUDE` | text | empty | Comma-separated glob patterns of skills to list. Empty = list all. Independently of this switch; when it hides skills, `SearchSkills` auto-enables. |
| `SKILL_LIST_EXCLUDE` | text | empty | Comma-separated glob patterns of skills to hide from the list. |
| `SKILL_LIST_INCLUDE_FILE` / `SKILL_LIST_EXCLUDE_FILE` | text | empty | The same patterns, one per line, read from a file. |
| `SKILL_LIST_TOP_PERCENT` | text | `0` | Keep only the top N% of skills by usage across past sessions. `0` = off; fails open (lists all) when there is no usage data. |
| `SKILL_LIST_STYLE` | choice | `full` | `full` = name plus description; `names` = names only. |

`TOOL_SEARCHSKILLS` lives in the **Tools** section of the config editor. All of the keys above can be set in any cascade tier, in a profile overlay, or as OS environment variables — the highest tier that sets each one wins.

---

## Gotchas

- **Lazy mode is opt-in and stays opt-in.** `TOOL_SEARCHSKILLS=0` is the shipped value, the compiled-in default, and the tool's own disabled-by-default flag. If you expected the catalog to vanish and it did not, check `scorpiox-config --get TOOL_SEARCHSKILLS` and remember an OS environment variable outranks every file.
- **Empty means two different things depending on where you set it.** An empty OS environment variable is ignored (the cascade falls through to the files), but an empty value *in a file* is a stored value — and any stored value other than `0` turns the tool on. Comment the line out or write `0` explicitly; do not rely on `TOOL_SEARCHSKILLS=`.
- **The editor's `1` is not a default.** The Tools-section row is gated on `TOOLS`; that number is the visibility rule, not a recommended value. Runtime defaults are `0` everywhere that matters.
- **Built-ins never go lazy.** The compiled-in skills stay listed in lazy mode by design — they are the small, universal procedures every session benefits from seeing. What leaves the prompt is your project, user, and additional skills.
- **An empty pattern is a listing, not a search.** `SearchSkills` with `pattern: ""` returns every skill and skips grep entirely. When a search returns the whole library, the usual cause is a pattern that arrived empty rather than a grep that matched everything.
- **The description is still the trigger.** In lazy mode the agent finds skills by matching text against descriptions and bodies. A vague description is invisible in both modes, but in lazy mode there is no catalog to compensate for it — write `Use when ...` triggers the way you would for a search engine.
- **Discovery costs a turn only when the agent thinks to search.** Lazy mode shifts responsibility for finding skills from the harness to the model. Required skills are the safety net for the ones that must never be skipped.
- **No `SearchSkills` in the browser build.** The grep runs as a subprocess; WASM cannot spawn one, so the tool is force-disabled there and the catalog is the only discovery surface.

---

## The bottom line

Lazy skill loading is a prompt-size decision, not a capability one. The catalog is a cost you pay on every request for the chance that the agent reaches for a skill unprompted; lazy mode trades that standing cost for a grep-backed search the agent runs when a task actually calls for it. If your library is large, your sessions are long, or your cache is precious, flip `TOOL_SEARCHSKILLS=1` and let `SearchSkills` carry the index. If your handful of skills is in constant use, leave the catalog on — visibility you can rely on is worth the bytes.

---

## Related

- [Skills in SCORPIOX CODE](skills-system.md) — the on-demand architecture this page tunes: where skills live, how they load, and the required-skills contract.
- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Deterministic File Editing & Developer Autonomy](file-editing.md) — `scorpiox-grep` itself, the engine behind the search.
- [Native MCP 2.0 and OAuth 2.1](mcp.md) — a different tool surface with its own discovery story.
