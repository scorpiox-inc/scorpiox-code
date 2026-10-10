# Configuration and Profiles

Every setting in SCORPIOX CODE is a `KEY=VALUE` pair, and every pair is read through one ordered list of locations — the **configuration cascade**. Defaults ship in the binary, an install-wide file tunes the whole machine, a user file tunes you, a project file tunes one repository, and an OS environment variable tunes one process. The same key can be set in any of them; the *highest* location that sets it wins.

On top of that sits **profiles**: named bundles of `KEY=VALUE` overrides stored as plain files. A profile is the highest file tier of the cascade, so flipping one on changes only the keys it names and leaves everything else exactly as it was.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** configuration is one ordered cascade where the highest location that sets a key wins, and a profile is a named overlay at the very top of that cascade — switch to it permanently with `/profile`, or for the current session only with `/use`.

---

## The cascade, top to bottom

SCORPIOX CODE resolves every key by walking these tiers in order, later tiers overwriting earlier ones. The **last** tier to set a key wins:

| Order | Tier | Location | Scope |
|-------|------|----------|-------|
| 1 | **default** | built into the binary | always present; the fallback for every key |
| 2 | **global** | `scorpiox-env.txt` next to the installed binaries | every user of this install |
| 3 | **user** | `~/.claude/scorpiox-env.txt` | you, on this machine, everywhere |
| 4 | **project** | `.claude/scorpiox-env.txt` then `.scorpiox/scorpiox-env.txt` in the working directory | one repository |
| 5 | **profile** | `scorpiox-env/<name>.txt` overlay | whatever the active profile names |
| 6 | **OS environment** | a real environment variable of the same name | one process / one launch |

A few rules fall straight out of that table:

- **Resolution is per key, not per file.** If your project file sets `MODEL` and your user file sets `PROVIDER`, you get the project's `MODEL` and the user's `PROVIDER`. Files do not replace each other; individual keys do.
- **Within the project tier, `.scorpiox/` beats `.claude/`.** Both are loaded at the project tier, but `.scorpiox/scorpiox-env.txt` is applied second, so when the same key appears in both, the `.scorpiox/` value wins. Keep one of them per project to avoid confusion.
- **A real OS environment variable outranks every file.** `MODEL=sonnet ./sx` overrides `MODEL` anywhere on disk for that launch, with no file edits. This is the escape hatch for CI, containers, and one-off runs.
- **Defaults are always there.** If no tier sets a key, you get the shipped default. You never have to set a key just to get sane behavior.

### The three files you will actually touch

```ini
# ~/.claude/scorpiox-env.txt        — your personal defaults, every project
MODEL=opus
TOOLS=1

# .scorpiox/scorpiox-env.txt        — this project only
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080

# .claude/scorpiox-env.txt          — git-friendly alternative to .scorpiox/
#                                     (project tier too, lower priority than .scorpiox/)
```

`.claude/` is usually tracked by git, so a project file placed there travels with the repository and configures every teammate who clones it. `.scorpiox/` is the traditional location for local, machine-specific project settings — the same tier, but it tends to be kept out of version control.

---

## Empty versus absent

This is the one subtlety that explains most "why is my setting not taking effect" moments.

- **A key set to an empty value in a file is still a value.** `OPENAI_EXTRA_HEADERS=` in a higher tier stores an empty string and therefore *shadows* a non-empty value in a lower tier. Empty beats absent.
- **An empty OS environment variable is ignored.** The environment tier only counts when the variable is non-empty; an empty one falls through to the files, so exporting `MODEL=` in a shell script does not blank out your config.
- **To truly unset a key, comment it out** rather than assigning it an empty value — unless shadowing a lower tier with "empty" is exactly what you want.

---

## Profiles: named overlays

A profile is a single file of `KEY=VALUE` overrides — same format as `scorpiox-env.txt` — that lives in a `scorpiox-env/` directory next to any of the config files you already have:

```
<exe_dir>/scorpiox-env/<name>.txt        # global profiles, shipped with the install
~/.claude/scorpiox-env/<name>.txt        # your profiles
<project>/.scorpiox/scorpiox-env/<name>.txt   # project-scoped profiles
```

A profile is an **overlay**, not a replacement. When it is active, its keys are applied at the top of the file cascade; every key it does not name keeps the value it already had. This is what makes a profile the right tool for a delta — a different endpoint, a different model, a stricter working directory — without duplicating your whole configuration.

```ini
# ~/.claude/scorpiox-env/<name>.txt
PROVIDER=openai
MODEL=local-llama
OPENAI_BASE_URL=http://localhost:8080
STARTUP_DIR=~/codebases/local-lab
```

**Whole-file shadow, by name.** Profiles are resolved by name from the highest tier that defines them: project, then user, then global. If two tiers both contain `local-llama.txt`, the project one wins *in its entirety* — profiles are chosen whole and never merged across tiers. The keys *inside* the winning profile still overlay the normal cascade.

**Names are constrained.** A profile name is a plain filename: no path separators, no `..`, no spaces, printable characters only, and under 64 characters. Anything else is refused at switch time rather than being allowed to wander the filesystem.

---

## `/use` versus `/profile`

Both commands switch the active profile and both hot-swap the running session to it. The difference is **persistence**:

| | `/use <name>` | `/profile <name>` |
|--|----------------|-------------------|
| Effect | active **for this session only** | active **and saved** for future sessions |
| Writes to disk? | no — sets a runtime override only | yes — writes `ACTIVE_PROFILE` to `~/.claude/scorpiox-env.txt` |
| Survives a restart? | no | yes |
| Scope | the running process | your user config, so every future launch |
| Turn it off | `/use off` | `/profile off` |

Reach for `/use` when you want to try or borrow a setup for one conversation and have it vanish when you quit. Reach for `/profile` when you have decided this is the setup you want next time you launch.

```
/use local-llama        → Profile 'local-llama' (session): local-llama · openai · tools:on
/profile local-llama    → Profile 'local-llama' active: local-llama · openai · tools:on
```

With no argument, both show what is available:

- `/use` lists profiles and marks the active one `(session-only)` — a reminder that nothing was written to disk.
- `/profile` runs the interactive **profile picker** if it is installed next to the binaries, and falls back to a plain list otherwise. The picker shows each profile with its tier badge (project, user, or global) and a preview of the keys it overrides; press Enter to select, Esc to cancel.

Both commands take `off` to deactivate: `/use off` clears the in-session override, and `/profile off` clears the saved one.

### Switching is live, and it reverts on failure

A profile switch is a **hot-swap**: SCORPIOX CODE reloads the configuration, rebuilds the provider with the new settings, and reports the resulting model, provider, and tool state — no restart, no new conversation. If the new provider fails to come up, the switch is **reverted on the spot** and you are told:

```
Profile switch failed (provider error) — reverted.
```

The previous profile is restored and you keep working. A profile that points at a broken endpoint cannot leave you stranded.

Both commands also apply a profile's `STARTUP_DIR` immediately, changing the working directory on the spot when the profile names one — so a profile can relocate a session to the right repository as part of the same switch.

> **Not available mid-turn.** `/profile` and `/use` are deliberately refused while the agent is busy generating — a config swap that changes the provider mid-request would corrupt the turn. Finish or cancel the turn, then switch.

---

## Creating profiles

There are two ways, and both write the same plain file.

**Let a setup wizard write one.** The provider wizards persist exactly this format and tell you the switch line when they finish:

```bash
scorpiox-openai-login --name local-llama   # writes ~/.claude/scorpiox-env/local-llama.txt
scorpiox-openai-login --status             # machine-readable status, incl. the profile path
```

The wizard asks for the endpoint and model, runs a tiny live completion through the final configuration to prove it works, and writes the profile only if it does. Other provider wizards follow the same pattern.

**Or write the file by hand.** Create `~/.claude/scorpiox-env/<name>.txt` with just the keys you want to change, then `/profile <name>`. Nothing else is required.

---

## Inspecting the cascade

When you want to know *why* a key has the value it does, ask the config tool. It reports the effective value and which tier supplied it — the fastest way to find a profile or an environment variable silently outranking a file you edited.

```bash
scorpiox-config --verbose        # cascade diagnostics: every tier path + key → source
scorpiox-config --dump           # all resolved config, flat KEY=VALUE
scorpiox-config --dump --files   # list the cascade file paths and whether each exists
scorpiox-config --dump --show-source   # group keys by the file that supplied them
scorpiox-config --get MODEL      # effective value of one key
scorpiox-config --json MODEL     # value + tier + path, machine-readable
```

You can also edit keys without opening the file:

```bash
scorpiox-config --set MODEL sonnet                    # project tier by default
scorpiox-config --set MODEL sonnet --level user       # write to ~/.claude/scorpiox-env.txt
```

Inside a running session, `/config` opens the interactive editor, which writes to the same cascade and reloads settings on exit.

And for a single launch, the main binary takes a repeatable override flag:

```bash
sx -e MODEL=sonnet -e TOOLS=0     # runtime-only, nothing written to disk
```

---

## How this compares to other harnesses

Configuration layering is not unique to SCORPIOX CODE — the interesting question is what the tiers are and how switching works.

| | Layering | Named presets | Mid-session switch |
|--|----------|---------------|--------------------|
| **SCORPIOX CODE** | defaults → global → user → project → profile → OS env, per key | yes — profiles as plain `KEY=VALUE` files, overlay semantics | yes — `/use` (session) and `/profile` (saved), with revert-on-failure |
| **Claude Code** | user and project settings files plus environment variables | settings profiles are not a first-class named concept | edit and reload; no built-in named switch |
| **OpenAI Codex** | a config file plus environment variables | per-provider config sections | edit and reload |
| **llama.cpp / vLLM / SGLang** | command-line flags and environment variables for the server process | launch scripts, not built-in presets | restart the server |

Two design choices stand out. First, **the profile is a file you own** — the same `KEY=VALUE` syntax as everything else, written by a wizard or by hand, tracked in git or kept local, with no separate schema to learn. Second, **the switch is a first-class command with two persistence levels**, so "try it for this conversation" and "make it my default" are one keystroke apart, and a failed switch cannot leave the session in a broken state.

---

## Configuration reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `ACTIVE_PROFILE` | text | *(empty)* | Name of the active profile overlay. Resolved from any `scorpiox-env/` directory in the cascade (project → user → global). Empty = no profile. Written permanently by `/profile`, overridden per session by `/use`. |
| `STARTUP_DIR` | path | *(empty)* | Working directory to enter on startup, on a config refresh, and on a profile switch. Supports `~`. Works per profile, so each profile can set its own directory. A `--cwd` command-line flag always wins. Empty = stay in the launch directory. |

These are the two keys that drive the profile machinery. Every other key in the reference — provider, model, tools, skills, timeouts — is resolved through the same cascade and can be overridden by a profile the same way. For the full set of keys, see the configuration file shipped with the install.

---

## Gotchas

- **Highest tier wins per key, and a profile is the highest file tier.** A profile can silently re-enable something you disabled in your project file. When behavior surprises you, run `scorpiox-config --verbose` and look at which tier supplies the value — the active profile is the usual culprit.
- **Empty beats absent inside files; empty is ignored in the environment.** `KEY=` in a higher tier shadows a lower tier; `KEY=` exported in your shell does not.
- **`.scorpiox/` beats `.claude/` within the project tier.** If you set the same key in both, the `.scorpiox/` value is the one you get.
- **Profiles are whole-file, chosen by name.** Two files with the same profile name in different tiers do not merge — the highest tier's file is used in full, and only its keys overlay the cascade.
- **`/use` leaves no trace.** If a setting is right in one session and gone after a restart, you switched with `/use` and not `/profile`. Use `/profile` to make it stick.
- **Profile switches are refused mid-turn.** If the command seems to do nothing while the agent is generating, let the turn finish and try again.
- **A failed switch reverts.** If a profile points at an unreachable endpoint, the switch is undone and your previous configuration is restored — you are not left in a half-switched state.

---

## Related

- [Using the OpenAI Provider](openai-provider.md) — points at local endpoints through a profile, with a setup wizard that writes one for you.
- [Session Identity Headers](identity-headers.md) — the profile opt-out pattern for turning request headers on and off.
- [Lazy Skill Loading](lazy-skill-loading.md) — a per-profile switch for large skill libraries.
- [Reasoning Effort Control](reasoning-effort.md) — a live dial plus per-provider keys that all resolve through this cascade.
