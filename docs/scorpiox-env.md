# Configuration and Profiles in SCORPIOX CODE

Every setting SCORPIOX CODE makes — which provider it talks to, which model it defaults to, where it writes logs, whether tools are on — comes from a small, flat set of `KEY=VALUE` pairs. There is no binary blob, no JSON schema you have to validate, no daemon to keep in sync. You edit a text file, and the next session picks it up.

The hard part is never a single file. The hard part is *several* files, in *several* places, each one trying to say the same thing. So SCORPIOX CODE loads its settings through a **configuration cascade**: a fixed list of locations, read in order, where each one that is present is allowed to override the ones below it. On top of that, **profiles** let you bundle a whole set of overrides into a named file and flip between bundles without touching any base config.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** SCORPIOX CODE reads your settings from a stack of known locations in a fixed order, lets the highest one win for any given key, and lets you swap named "profiles" — self-contained override bundles — between them, either for the whole machine and every session, or for just the session in front of you.

---

## The configuration file

The unit of configuration is a single-line `KEY=VALUE` pair in a plain text file named `scorpiox-env.txt`:

```ini
# Provider + model
PROVIDER=claude_code
MODEL=sonnet

# Turn tool use on or off (1 / 0)
TOOLS=1

# Where the agent starts when a session begins
STARTUP_DIR=~/dev

# Logging
LOG_DIR=.scorpiox/logs
```

That is the whole syntax. One `KEY=VALUE` per line, `#` starts a comment, blank lines are ignored. No quoting required for typical values, no sections, no escaping ceremony. If you already have a `scorpiox-env.txt` shipped with your install, that file is your reference for every key that exists — it is annotated and covers the full surface, from provider and model to proxy endpoints and timeouts.

---

## The cascade: where settings are read from, in order

SCORPIOX CODE does not read one file. It walks a stack of five tiers, from the most built-in to the most specific, and for any given key the **highest tier that sets it wins**. Think of it as a set of sheets stacked on a table: the top sheet's value is what you see, and if the top sheet leaves a key blank, the value shows through from the sheet below it.

From lowest to highest priority:

| Tier | Where it lives | What it is for |
|------|----------------|----------------|
| 1 (lowest) | Built into the binary | Sensible out-of-the-box defaults. You never edit this. |
| 2 | `scorpiox-env.txt` next to the installed binaries | **Global install config.** One file that tunes this install for everyone on the machine. |
| 3 | `~/.claude/scorpiox-env.txt` | **User config.** Your personal defaults, shared across all projects on this account. |
| 4 | `.claude/scorpiox-env.txt` or `.scorpiox/scorpiox-env.txt` in your working directory | **Project config.** Committed to the repo so a whole team gets the same behavior in that project. |
| 5 (highest) | `scorpiox-env/<name>.txt` for the active profile | **Profile overlay.** A named bundle of overrides applied last. |

A few rules that fall out of this table:

- **Highest tier wins, key by key.** If your project file sets `MODEL=opus` and your user file sets `MODEL=sonnet`, a session started in that project runs on `opus`. Any key the project file does not mention still falls through to your user value. You are not forced to repeat everything in every layer.
- **Within the project tier, `.scorpiox/` beats `.claude/`.** Both locations are read; if a project ships a `scorpiox-env.txt` in *both* `.claude/` and `.scorpiox/`, the `.scorpiox/` copy is the one that lands on top. `.scorpiox/` is the traditional SCORPIOX CODE home, `.claude/` is there so a project file can coexist with the files other tools already read.
- **The working directory decides which project file applies.** SCORPIOX CODE looks in the directory you started the session from. If you start from a subdirectory, it is that subdirectory's `.claude/` and `.scorpiox/` that are consulted, not the repo root's.

### The one thing above everything: real environment variables

There is a layer even above tier 5. If a key is set as a genuine **OS environment variable** in the shell that launches SCORPIOX CODE, that value wins over every file in the cascade, profile included.

```bash
MODEL=haiku ./scorpiox   # this MODEL beats every scorpiox-env.txt and every profile
```

This is the escape hatch, not the everyday path. It is how CI, containers, and wrapper scripts inject a one-off override without writing a file, and how a temporary `ACTIVE_PROFILE` switch works under the hood (see below). For day-to-day configuration you will almost always want to edit a file so the setting is durable and visible, and reserve environment variables for the moment you need to override a file without changing it.

---

## Profiles: named bundles of overrides

A **profile** is just a `scorpiox-env.txt` with a name. Instead of scattering `PROVIDER=...` and `MODEL=...` across your base files, you collect the things that change together into one file in a `scorpiox-env/` directory:

```ini
# .scorpiox/scorpiox-env/fast.txt
# A fast, cheap, local-flavored profile
PROVIDER=codex
MODEL=haiku
TOOLS=1
```

When that profile is active, its values are applied as the **topmost overlay** on the cascade. It does not replace your base config — it sits on top of it. Anything the profile file does not mention still comes from your project, user, and global files underneath. A profile is a *delta*, not a snapshot.

### Where profiles live, and which one wins

Profiles are looked up in the same per-tier locations, and — exactly like the base files — the **highest tier wins**. For a profile named `fast`, SCORPIOX CODE checks, from top down, and stops at the first match:

| Tier | Profile location |
|------|------------------|
| Project | `.scorpiox/scorpiox-env/<name>.txt` |
| User | `~/.claude/scorpiox-env/<name>.txt` |
| Global | `scorpiox-env/<name>.txt` next to the installed binaries |

So a `fast.txt` you commit into a repo's `.scorpiox/scorpiox-env/` shadows any same-named `fast.txt` in your user or global profile folders, for sessions started in that project. That is how a team ships a project-specific profile while letting people keep their own personal ones elsewhere. Profile names are short, printable, and must not contain spaces, slashes, or `..` — keep them boring and greppable.

### How a profile is selected

The key that names the active profile is `ACTIVE_PROFILE`. Setting it points the cascade at the matching `scorpiox-env/<name>.txt`; leaving it empty means "no profile, run on the base cascade alone." You set it in a file (it persists), or you set it for the session only (it does not) — which is exactly the difference between `/profile` and `/use`.

---

## `/profile` vs `/use`: the one thing people get wrong

Both commands switch the active profile, and both do it **live** — they reload the configuration and the provider in place, no restart. The difference is one word and it is the whole point: **does the switch survive this session?**

| | `/profile` | `/use` |
|---|---|---|
| Persists | Yes — saved | No — session only |
| Writes to disk | Yes, to your user config | No |
| Survives a restart / new session | Yes | No |
| Reverts on its own | No — until you change it back | Yes — when the session ends |
| Typical use | "I'm working on this stack this week." | "Let me try this model for a bit." |

**`/profile <name>` is the saved switch.** It writes `ACTIVE_PROFILE=<name>` into your user-tier config and applies it immediately. Every session from now on starts with that profile active until you change it. It is the durable choice: "this is how I want SCORPIOX CODE to behave."

**`/use <name>` is the temporary switch.** It sets `ACTIVE_PROFILE` for the current session only and writes nothing. The switch is live for everything you do this session, then it is gone. When the session ends, the next one starts from whatever your saved profile is. It is the throwaway choice: "let me poke at this for a few minutes."

Both commands share the same controls:

- **No argument** lists the available profiles and the one that is currently active. `/profile` with no argument will, where a profile picker is available, open it instead.
- **`off`** deactivates the profile and drops back to the plain base cascade. `/profile off` saves that; `/use off` does it for this session only.
- **Unknown name** is rejected with a hint to run the no-argument form and list what exists.

When either command takes effect, SCORPIOX CODE confirms the live result in the chat so you can see exactly what you are now running, for example:

```
Profile 'fast' active: haiku · codex · tools:on
```

That line is the model, the provider, and the tool switch — the three things a profile change is usually there to move. If the reload fails (for instance the new provider cannot start), the switch is **reverted** and you get a clear error rather than a half-applied configuration. A profile may also carry a `STARTUP_DIR`; on a successful switch SCORPIOX CODE moves to that directory, and a bad start directory is skipped without rolling back the profile itself.

### A concrete pattern

```text
# You commit a fast, cheap profile to the project:
#   .scorpiox/scorpiox-env/fast.txt

> /use fast
Profile 'fast' active: haiku · codex · tools:on
> # ... run a few quick, cheap experiments ...
> /use off
No profile active.   (back to your saved default, this session only)

> /profile fast
Profile 'fast' active: haiku · codex · tools:on
> # saved — every future session starts here until you change it
```

The discipline is: **`/use` to try, `/profile` to commit.** If you find yourself re-running the same `/use` in every session, that is SCORPIOX CODE telling you the profile should actually be saved.

---

## Putting it together: a key's full journey

Follow `MODEL` from a session started inside a project, with a `fast` profile active, and `MODEL` also exported in the shell:

1. **Built-in default** supplies a baseline `MODEL`.
2. **Global** install file may override it.
3. **User** `~/.claude/scorpiox-env.txt` may override that.
4. **Project** file may override that.
5. **The active profile** (`scorpiox-env/fast.txt`) overrides that, if it sets `MODEL`.
6. **The OS environment variable** `MODEL` overrides all of the above.

The value that survives is the one from the highest tier that actually set the key. Change any layer, or activate a different profile, and the winning value changes predictably — no other key is touched.

---

## Gotchas

- **Highest wins, per key, not per file.** A profile does not disable the layers beneath it. It only overrides the keys it names. If a profile sets `PROVIDER` but not `MODEL`, you get the profile's provider with the base cascade's model. That is the feature, and it is the surprise.
- **The working directory selects the project tier.** Starting a session from a subfolder reads that subfolder's files. If your project config lives at the repo root and you start from `src/`, the root config is invisible to that session. Start from the root, or place the file where you start.
- **`/use` is invisible to the next session.** It is the most common "why did my setting disappear" — the switch was session-scoped by design. If a change must last, it is `/profile`, or edit a file.
- **Environment variables beat the profile too.** A stray `MODEL` exported in a shell profile will silently outrank a profile you think is controlling it. When a value "won't stick," check the environment before blaming the files.
- **A profile name must resolve somewhere.** `ACTIVE_PROFILE=fast` only means something if a `scorpiox-env/fast.txt` actually exists in one of the profile locations. A dangling name is a no-op with a warning, not an error.
- **Switches revert on failure.** Both `/profile` and `/use` roll back if the new provider cannot start. You will never be left half-switched.

---

## The bottom line

SCORPIOX CODE's configuration is a flat `KEY=VALUE` file read through a five-tier cascade, where the highest tier that sets a key wins, and real environment variables sit above even that. Profiles are the same files with names: a self-contained overlay you point at with `ACTIVE_PROFILE`, applied on top of everything else and resolved the same highest-tier-wins way. And the two switches are deliberately split — **`/use` to try a profile for this session, `/profile` to save it for all of them.**

Keep your durable, shared defaults in the base files, keep your frequently-changed bundles in profiles, and use the temporary switch to experiment without ever touching what is saved.

---

## Related

- [Project Instructions in SCORPIOX CODE: CLAUDE.md and AGENTS.md](project-instructions.md)
- [Long-Horizon Agent Tasks: Conversation Compaction](conversation-compaction.md)
- [Deterministic File Editing & Developer Autonomy](file-editing.md)
