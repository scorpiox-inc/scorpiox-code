# Lazy Skill Loading

SCORPIOX CODE ships two shapes for the same skill surface. In the default shape the agent sees a **catalog**: every available skill's name and one-line description is baked into the `InvokeSkill` tool description on every request, and the body loads only when a skill is invoked. In the lazy shape the catalog is dropped: the agent sees a short pointer telling it to *search* for skills instead, and discovery happens through a second tool — **`SearchSkills`** — on demand.

The whole switch is one config key: **`TOOL_SEARCHSKILLS`**.

Docs for SCORPIOX CODE @ `77c49df`.

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

**Lazy mode (`TOOL_SEARCHSKILLS=1`)** — the catalog is replaced by a pointer:

```
Invoke or load a skill by name. Returns the skill content for
you to follow or use as context. Use the SearchSkills tool first
to discover available skills by keyword.
```

That is the entire description when no other listing rules are in play. The names and descriptions of your project, user, and additional skills leave the prompt. Built-in skills — the ones compiled into the binary — stay listed, so the agent keeps a floor of always-visible procedures (`create-skill`, `edit-skill`, `preferred-*`, and friends). The description then carries the familiar `(+N more skills not listed - use SearchSkills to discover them)` note and the skill-folder paths, so the agent knows a larger library exists behind the search even though nothing about it is enumerated.

Everything still invokable. `InvokeSkill deploy` works in lazy mode exactly as it does in catalog mode — the list controls only what is *advertised*, never what is *loadable*.

### When lazy mode saves tokens, and when it costs

| Situation | Lean toward |
|-----------|-------------|
| Large library (dozens of skills), long sessions, prompt-cache-sensitive endpoints | Lazy |
| Small library (a handful of skills you use constantly) | Default |
| Skills the agent must reliably reach for *unprompted* | Default, or lazy plus the required-skills contract |
| Skills invoked only when a task explicitly calls for them | Either; lazy saves the standing bytes |

The trade is a standing cost against a per-task cost. In catalog mode the list is paid on **every** request, including the many turns where no skill is relevant. In lazy mode you pay a **discovery round-trip** once per unfamiliar task: the model calls `SearchSkills`, reads the matches, then calls `InvokeSkill`. That round-trip is small — a grep is cheap and the result is a compact list of names and descriptions — but it is a turn of latency the catalog mode never spends, and it depends on the model actually *thinking* to search.

For a concrete scale check: this repository's own `.claude/skills/` folder holds 23 skills. Their catalog entries sum to about 5.5 KB of description text on every request; the nine always-listed built-ins plus the lazy pointer and folder hint come to roughly half that, and the pointer-only form is 167 bytes. A library two or three times larger tips the arithmetic decisively toward lazy. A library of five well-named skills gains little and loses the always-on visibility that makes the agent reach for them without being asked.

Lazy mode also has a quiet cache benefit. Tool definitions are part of the prompt the provider caches, so a big catalog raises the baseline size of every cached request. Shrinking the standing tool surface is one of the few ways to make a large skill library *and* a warm cache coexist.

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

The shipped `scorpiox-env.txt` sets `TOOL_SEARCHSKILLS=0`, and that is also the compiled-in default — the tool registry and the built-in defaults both say off. So lazy mode is strictly opt-in: set the key to `1` in the tier that matches how widely you want it. The gate reads *any* stored value other than `0` as enabled, which has one sharp edge — an explicitly empty `TOOL_SEARCHSKILLS=` line in a config file counts as enabled, not off. Use `TOOL_SEARCHSKILLS=0` to turn it off.

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

There is no restart ritual: the key is read when the tool list is assembled at the start of a session, so a new session picks up a change immediately.

One honesty note about defaults. The interactive config editor's entry for this key declares a default of `1`, which disagrees with everything that actually decides the behavior at runtime — the tool registry, the compiled-in defaults, and the shipped `scorpiox-env.txt`, all of which say `0`. You will not normally notice, because the shipped file always sets the key explicitly. But if you ever start from a stripped-down config, trust `scorpiox-config --get` over any documented default: the runtime answer is off unless something set it to `1`.

---

## SearchSkills: the discovery tool

`SearchSkills` is what makes lazy mode work. It is a real grep — the product's own zero-dependency `scorpiox-grep` binary, invoked with `-rl` (recursive, filenames only) — run across the skill directories: your user skills folder, the project skills folder, and every additional skills directory you have configured. Built-in skills have no file on disk, so they are matched in memory against their names and descriptions instead.

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

Pass an empty `pattern` and the grep is skipped entirely; the tool returns the full library — every skill's name, description, source, and path, up to a cap of 128 matches. This is the "show me the menu" call, and it is the lazy-mode replacement for the catalog that used to sit in the prompt: same information, delivered once, when asked for, instead of on every request.

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
- Flags combine (`-i -w`, `-E -i`) and are passed through to the grep. Anything outside the `-i` / `-E` / `-w` whitelist is silently dropped rather than executed, so a malformed flag degrades to a normal case-insensitive search instead of erroring out or running something unexpected.

Searches time out at 15 seconds, which is generous for a recursive grep over skill folders; if a skills directory lives on a slow network mount, keep it small or move it local.

---

## How lazy mode interacts with everything else

**List filters and the auto-enable.** The `SKILL_LIST_*` keys filter the catalog independently of this switch — `SKILL_LIST_INCLUDE` / `SKILL_LIST_EXCLUDE` (plus their `_FILE` variants), `SKILL_LIST_TOP_PERCENT` to keep only the most-used skills, and `SKILL_LIST_STYLE=names` to advertise names without descriptions (a style, not a filter — it does not hide anyone). Filtering and lazy mode are two different answers to the same problem: filters keep a *reduced* catalog always visible; lazy mode removes the catalog and substitutes search.

The two mechanisms meet in one place: **the moment the visible list is partial — whether because a filter hid skills or because lazy mode is on — `SearchSkills` is enabled.** The auto-enable is in-memory for the running session and is never written back to your config files, so a filter can turn the tool on for a session without permanently flipping the key. Required skills and built-in skills are exempt from filtering and always stay listed, so a filter can never hide the things a session is contractually obliged to load.

**Required skills still work.** The required-skills cascade (`required_skills.txt` at user, project, and session level) is unchanged in lazy mode: a required skill that has not been loaded still triggers the end-of-turn nudge naming it, and force-inject still puts its body straight into context. If you go lazy, this is the mechanism that keeps must-use skills guaranteed — the catalog no longer advertises them, but the nudge still names them and tells the agent to load them. See [Skills in SCORPIOX CODE](skills-system.md) for the full contract.

**Prompt-cache stability.** In catalog mode, adding or renaming a skill rewrites the `InvokeSkill` description and can invalidate the cached prompt prefix; the loader already minimizes that by rescanning only when a skill directory's mtime changes. Lazy mode removes most of that churn: the description shrinks to a static pointer plus a built-in index, so adding or editing your own skills leaves the tool JSON untouched. A fast-moving, team-owned skills library is therefore a natural fit for lazy mode.

**Where the feature does not reach.** The browser/WASM build cannot run `SearchSkills` — the grep is a subprocess and the sandbox has no processes — so lazy mode is a native-build feature. On WASM the catalog stays. And `/disable_skill` behaves the same in both modes: the skill stays advertised (so the list stays stable), and loading it returns an error for the rest of the session.

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

- **Lazy mode is opt-in and stays opt-in.** `TOOL_SEARCHSKILLS=0` is both the shipped value and the compiled-in default. If you expected the catalog to vanish and it did not, check `scorpiox-config --get TOOL_SEARCHSKILLS` and remember an OS environment variable outranks every file.
- **Empty means two different things depending on where you set it.** An empty OS environment variable is ignored (the cascade falls through to the files), but an empty value *in a file* is a stored value — and any stored value other than `0` turns the tool on. Comment the line out or write `0` explicitly; do not rely on `TOOL_SEARCHSKILLS=`.
- **Built-ins never go lazy.** The compiled-in skills stay listed in lazy mode by design — they are the small, universal procedures every session benefits from seeing. What leaves the prompt is your project, user, and additional skills.
- **An empty pattern is a listing, not a search.** `SearchSkills` with `pattern: ""` returns every skill and skips grep entirely. When a search returns the whole library, the usual cause is a pattern that arrived empty rather than a grep that matched everything.
- **The description is still the trigger.** In lazy mode the agent finds skills by matching text against descriptions and bodies. A vague description is invisible in both modes, but in lazy mode there is no catalog to compensate for it — write `Use when ...` triggers the way you would for a search engine.
- **Discovery costs a turn only when the agent thinks to search.** Lazy mode shifts responsibility for finding skills from the harness to the model. Required skills are the safety net for the ones that must never be skipped.
- **Filtering and lazy mode are alternatives, not ingredients.** If `SKILL_LIST_*` filters are active, they shape the catalog and `SearchSkills` auto-enables for the hidden remainder; setting `TOOL_SEARCHSKILLS=1` on top collapses the catalog to built-ins. Both end at the same place — search as the path to everything not listed — so pick one and let it do the work.
- **No `SearchSkills` in the browser build.** The grep runs as a subprocess; WASM cannot spawn one, so the tool is force-disabled there and the catalog is the only discovery surface.

---

## The bottom line

Catalog mode buys always-on visibility with a standing per-skill cost. Lazy mode buys a small standing prompt and a cache-friendlier tool surface, and pays for it with an occasional discovery call — a grep over your own files, with an empty pattern when you want the full menu. The switch is `TOOL_SEARCHSKILLS`, the discovery tool is `SearchSkills` (`-i` / `-E` / `-w`, empty pattern to list all), and the guarantee layer — required skills, built-ins, and the nudge — survives the change intact.

---

## Related

- [Skills in SCORPIOX CODE](skills-system.md) — the on-demand architecture this page tunes: where skills live, how they load, and the required-skills contract.
- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Deterministic File Editing & Developer Autonomy](file-editing.md) — `scorpiox-grep` itself, the engine behind the search.
- [Native MCP 2.0 and OAuth 2.1](mcp.md) — a different tool surface with its own discovery story.
