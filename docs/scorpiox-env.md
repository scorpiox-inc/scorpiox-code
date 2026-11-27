# Configuration and Profiles

You have three projects and one of them talks to a router, one to your own Anthropic key, and one to a self-hosted OpenAI-compatible endpoint. You do not want to edit a config file, restart, and flip back and forth every time you switch projects — and you do not want the right keys leaking into the wrong project.

**Profiles** in SCORPIOX CODE are exactly that: named sets of configuration values you can load and unload on demand. A profile is a small file that overrides only the settings it names, and sits on top of your normal configuration. You keep one file per "mode" — `router`, `anthropic`, `local-dev` — and switch between them with a single command. No restart, no copy-paste, no editing the same line back and forth.

Docs for SCORPIOX CODE @ `41b090b`.

> **The whole idea in one line:** one file per mode, and a single command to load it. A profile only overrides the keys it names; everything else keeps falling through to your normal configuration.

---

## The configuration cascade (and what wins)

SCORPIOX CODE reads its settings from several places, in this order. When the same key is set in more than one place, **the higher one wins** — and a profile, when active, is the highest file layer of all.

| Priority | Layer | Where it lives | Who it is for |
|---|---|---|---|
| Highest | OS environment variable | Your shell environment | Escape hatch for CI, scripts, one-off overrides |
| | **Active profile** | `scorpiox-env/<name>.txt` | Named modes you switch into on demand |
| | Project | `.scorpiox/scorpiox-env.txt` in your working directory | Per-repo settings, safe to commit |
| | User | `~/.claude/scorpiox-env.txt` | Your personal defaults across all projects |
| | Global (install) | `scorpiox-env.txt` next to the SCORPIOX CODE binaries | Fleet / install-time defaults |
| Lowest | Built-in defaults | Compiled into the binary | What you get with zero configuration |

A few properties follow directly from this:

- **Lower layers fill in what higher layers do not set.** A project file only has to name the keys that differ from your defaults — you do not re-declare everything. Same for a profile.
- **Project settings override personal settings.** Commit a project config and every teammate on the machine picks it up; it is scoped to that directory, so it never leaks into your other repos.
- **`.scorpiox/` beats `.claude/` at the project level.** If your working directory has both `.scorpiox/scorpiox-env.txt` and `.claude/scorpiox-env.txt`, the `.scorpiox/` one wins.
- **An environment variable always wins.** For scripting and CI you can set `PROVIDER=...` in your shell and it takes effect immediately, above every file.

Settings are `KEY=VALUE` lines; a `#` starts a comment and blank lines are ignored. The install ships with a fully commented template at `scorpiox-env.txt` next to the binaries — read it once and you will know the keys.

---

## What a profile is

A profile is a **separate file** that overlays the cascade. It lives in a `scorpiox-env/` directory (note the plural — it is a folder, not the `scorpiox-env.txt` file) alongside your configuration layers, and it is named `<name>.txt`:

- **User-level:** `~/.claude/scorpiox-env/<name>.txt`
- **Project-level:** `.scorpiox/scorpiox-env/<name>.txt` in your working directory (or `.claude/scorpiox-env/<name>.txt` if you prefer the git-friendly directory)
- **Global (install):** `scorpiox-env/<name>.txt` next to the binaries

When a profile is active, its keys are loaded **last**, so they shadow every layer below them — and any key the profile does not name simply keeps its cascade value. That is what makes a profile safe: it is an overlay, not a replacement. A profile that sets `PROVIDER` and `OPENAI_API_KEY` leaves all your other settings untouched.

A profile can also change **where you start** by setting `STARTUP_DIR` — SCORPIOX CODE moves its working directory there when the profile loads, which is a clean way to give a mode its own repo. A bad path is skipped, not fatal.

---

## `/profile` — a saved switch

`/profile` makes the switch **stick**. It is the one to use when a mode should be your default for the next session, the next machine, the next time you open a terminal.

```
/profile <name>        # activate profile <name> and remember it
/profile off           # turn the profile off
/profile              # (no name) list what is active and what is available
```

What it does, concretely:

- **Writes the choice to your user config.** It records the active profile in `~/.claude/scorpiox-env.txt` (the `ACTIVE_PROFILE` key). Because that is a file on disk, the choice survives a restart and is present the next time you start SCORPIOX CODE.
- **Switches immediately, and rolls back on failure.** The provider and model are re-created right away so you see the change take effect. If the new provider cannot be created (bad key, unreachable endpoint), SCORPIOX CODE restores the previous profile and tells you, rather than leaving you half-switched.
- **Tells you the result.** You get a one-line summary of the now-active model, provider, and tool state.

The **list form** (`/profile` with no argument) is your inspector: it shows which profile is active and every profile it can find, each tagged with the layer it came from (`project`, `user`, or `global`) so you know exactly which file you are activating.

**Use `/profile` when the choice should outlive this session.** "I am working on the billing repo this week, which uses the self-hosted endpoint" — that is a saved switch.

---

## `/use` — a temporary session switch

`/use` makes the switch for **this session only**. It is the one to use when you want to try a mode, do a quick test, or A/B two providers without committing to either.

```
/use <name>           # load profile <name> for this session only
/use off              # drop it back to the base configuration
/use                  # (no name) show the session-only active profile
```

The difference from `/profile` is the one that matters:

- **No file is written.** `/use` changes configuration in memory only. Close the session and it is gone — your `~/.claude/scorpiox-env.txt` is untouched, and your next session starts exactly as it would have.
- **Same safety guarantees.** It still validates that the profile exists, switches the provider live, and rolls back if the new provider fails to start.

**Use `/use` when you do not want to commit.** "Let me check whether this prompt works against the router before I switch the whole project over" — that is a temporary switch. You can flip modes back and forth in a session with `/use` and only run `/profile` once you decide which one to keep.

> **The rule of thumb:** `/use` is a trial; `/profile` is a decision. If you would not want it still on in a week, use `/use`.

---

## Picking a profile

- **Type it directly** with `/profile <name>` or `/use <name>` when you know the name.
- **Browse interactively.** On a terminal, if the `scorpiox-profile` picker is installed next to your binaries, a bare `/profile` opens a small screen that lists every profile with its layer, shows which keys each one overrides, and lets you pick with your keyboard. It is the friendliest way to discover what you have.
- **Test before you switch.** `scorpiox-profile-test <name>` runs one minimal turn through the real provider stack for a given profile, with no session files and no writes to your config. It is the honest way to confirm a profile's credentials and endpoint actually work before you make it active. Add `--isolated` to require the key to come from the profile itself rather than your base config.

---

## A practical layout

```
my-repo/
  .scorpiox/
    scorpiox-env.txt          # project base config (committed)
    scorpiox-env/
      router.txt              # PROVIDER=scorpiox ...
      selfhosted.txt          # PROVIDER=openai  OPENAI_API_KEY=...
      local-dev.txt           # CLAUDE_CODE_TOKEN_SOURCE=local ...

~/.claude/
  scorpiox-env.txt            # your personal defaults (ACTIVE_PROFILE lives here)
  scorpiox-env/
    prod.txt                  # shared modes you keep at user level
```

Switch inside the repo with `/use selfhosted`, confirm the credentials with `scorpiox-profile-test selfhosted`, and once it is what you want make it stick with `/profile selfhosted`. When you are done and the choice was only for this session, `/use off` returns you to the committed project defaults.

---

## What to watch for

- **A missing profile is an error, not a silent no-op.** `/use nothere` and `/profile nothere` both tell you the profile was not found and point you back to the list.
- **Profile names are plain and local.** No slashes, no `..`, no spaces or colons, and only printable characters. Keep them short — they are filenames.
- **A profile overlays, it does not replace.** A key the profile does not set keeps its cascade value, so you can be selective about exactly what a mode changes.
- **The provider switch can roll back.** If the newly-selected provider fails to start, SCORPIOX CODE restores the previous profile and tells you — you are never left half-switched.
- **`/use` is gone at the next session.** If a switch needs to survive a restart or a different machine, that is `/profile`.

---

*Related: the configuration cascade and the commented `scorpiox-env.txt` template shipped with your install, and `scorpiox-profile-test` for validating a profile's credentials before you switch into it.*
