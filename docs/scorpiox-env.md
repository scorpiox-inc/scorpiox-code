# Configuration and Profiles in SCORPIOX CODE

SCORPIOX CODE has a lot of knobs: which provider it talks to, which model, how hard it thinks, which tools are on, where it starts up, which credentials it uses. If every one of those had to live in a single place you would rewrite the whole file to change one thing — and two machines, or two accounts, or two projects could never share the same setup. **The configuration cascade** and **profiles** solve that: a small, predictable set of files that layer on top of each other, plus named bundles of overrides you can flip on and off with one command.

This page is the map. It explains exactly which file wins when they disagree, where each one lives, what a profile is, and — the part people trip over most — the difference between **`/use`** (a temporary switch that lasts one session) and **`/profile`** (a saved switch that sticks around).

Docs for SCORPIOX CODE @ `2b0bffd`.

> **The whole idea in one line:** settings are layered from most-generic to most-specific, the most-specific wins, and a *profile* is a named file of overrides you activate at runtime — with `/use` to try it for this session only and `/profile` to make it permanent.

---

## What the cascade is

Instead of one configuration file, SCORPIOX CODE reads several, in a fixed order, and lets later ones override earlier ones for any key they define. That set of files, from the one with the least authority to the one with the most, is the **cascade**.

Think of it as "defaults at the bottom, your machine in the middle, your project on top, and the environment you launched from at the very top." Each layer only needs to set the keys it cares about. Anything it leaves blank is inherited unchanged from the layers below it.

### The tiers, bottom to top

| Tier | Where it lives | Who it's for |
|------|----------------|-------------|
| **Defaults** | Built in | The safe out-of-the-box values. You never edit these. |
| **Global** | `scorpiox-env.txt` next to the installed binary | Machine-wide install defaults. |
| **User** | `~/.claude/scorpiox-env.txt` | Your personal settings on this machine, across all projects. |
| **Project** | `.claude/scorpiox-env.txt` or `.scorpiox/scorpiox-env.txt` in the working directory | This repository. Check it into git so the team shares it. |
| **Profile** | `scorpiox-env/<name>.txt` (see below) | A named bundle of overrides you activate on demand. |
| **OS environment** | Variables in the shell that launched it | The strongest override of all — set a variable and it beats every file. |

Two things about this ordering matter in practice:

- **Only the keys a file defines are changed.** A project file that sets `MODEL` and `PROVIDER` leaves everything else — thinking budget, tools, streaming — inherited from your user file and the defaults. You are not rewriting a config, you are *narrowing* it.
- **The environment beats every file.** If `MODEL` is set as a real OS environment variable, that value wins over `MODEL` in any `scorpiox-env.txt`. This is how you do a one-off launch without touching a single file:

  ```bash
  MODEL=sonnet sx          # one-off: this launch uses sonnet, no file touched
  ```

### Which project file wins?

The project tier has two home directories. If a project ships both, the traditional one wins:

1. `.claude/scorpiox-env.txt` — git-friendly, sits next to other agent config.
2. `.scorpiox/scorpiox-env.txt` — the classic location. **Overrides `.claude/` if both are present.**

You only ever need one. Pick `.claude/` if your repo already uses that layout, otherwise `.scorpiox/`.

---

## What a profile is

A **profile** is a single text file that sets a handful of keys — the keys that differ for a particular provider, account, or project — and loads as the highest *file* tier when it is active. Everything the profile does not set is inherited from the cascade below it, exactly as before.

That last point is what makes profiles cheap to maintain. A "work" profile does not contain your whole configuration; it contains only the things that are different at work. Flip the profile off and you are back on your baseline.

### Where profiles live, and how they shadow

Profiles live in a `scorpiox-env/` folder *next to* the configuration files, at each tier:

- **Global** — `scorpiox-env/` next to the installed binary.
- **User** — `~/.claude/scorpiox-env/`
- **Project** — `.scorpiox/scorpiox-env/` in the working directory.

Each profile is one file named after the profile, for example `~/.claude/scorpiox-env/work.txt`. When you activate a profile by name, SCORPIOX CODE looks for that file and applies it.

If the same profile name exists at more than one tier, the **whole file** shadows the lower tier — it is not merged line by line. Project beats user beats global. So a project can ship a `work.txt` that completely replaces your personal `work.txt` for that repository, without touching your home directory.

A minimal profile, for a second Copilot account, looks like this:

```
# ~/.claude/scorpiox-env/copilot-work.txt
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/work.json
MODEL=claude-sonnet-5
```

Four keys. Nothing else. When active, those four override the cascade; the rest is untouched.

---

## `/use` vs `/profile`: the switch that matters

This is the one distinction that saves people real confusion, so it gets its own section. Both commands activate a profile **immediately, in the running session, with no restart** — they reload the provider and the model on the spot. The only difference is **how long the switch lasts**.

| | **`/profile <name>`** | **`/use <name>`** |
|---|---|---|
| **Persists?** | **Yes** — saved to your user config | **No** — held in memory only |
| **Survives a restart?** | Yes — re-applies next launch | No — gone when the session ends |
| **Writes a file?** | Yes, sets the active profile in `~/.claude/scorpiox-env.txt` | No file is written |
| **Best for** | "I'm working on this account / project for a while" | "Let me quickly try this profile, then go back" |
| **Deactivate** | `/profile off` | `/use off` |
| **No argument** | Opens the profile picker (or lists them) | Lists available profiles |

The practical rule: **use `/use` to experiment, `/profile` to commit.**

```
/use work              # try the work account for this session only
/profile work          # make work the saved, default profile from now on
/profile off           # turn the saved profile off
```

Both switches are **revert-on-fail**: if activating the profile breaks the provider (a bad endpoint, a dead credential), SCORPIOX CODE rolls back to the profile you had before and tells you, instead of leaving you in a broken state.

A note on the mechanism under the hood: a profile is selected by an `ACTIVE_PROFILE` key. `/profile` writes that key into your **user** tier so it survives; `/use` sets it only for the current process, which is why it disappears. You can also point at a profile from outside without any command at all:

```
ACTIVE_PROFILE=work sx     # launch with 'work' already active
```

An OS environment variable is the strongest override, so this beats whatever is saved in a file — handy for scripts and one-off runs.

### The profile picker

Run `/profile` with no argument and SCORPIOX CODE opens an interactive picker (a small terminal UI, not a chat message). It lists every profile it can see across all tiers, shows a badge for the tier each one came from, and previews the keys that profile overrides so you can see exactly what it will change before you press Enter. The first entry is always **off**. If the picker component is not installed for your build, the command falls back to printing the list in the chat instead.

---

## Inspecting the resolved configuration

When several tiers are in play, you sometimes want to see what actually won for a given key. The `scorpiox-config` tool answers that:

```bash
scorpiox-config                 # open the interactive configuration editor
scorpiox-config --verbose       # print key values and which tier each one came from
scorpiox-config --dump          # dump every resolved key
scorpiox-config --get MODEL     # print a single resolved key
```

`--verbose` is the one to reach for: it shows the resolved `PROVIDER`, `MODEL`, and active profile together with their source tier, so you can confirm you are pointed where you think you are before a long run.

---

## One-off overrides without touching a file

For a single launch you can override a key straight from the command line with `-e`. It behaves exactly like setting the key in the cascade but only lives for that process:

```bash
sx -e PROVIDER=copilot -e MODEL=claude-sonnet-5   # this launch only
```

`-e` is repeatable. Combine it with an OS variable and either will beat the files — they are simply the top of the cascade, applied to the running session.

---

## A profile can also change where you start

Profiles are not limited to provider and model. A profile can set the startup working directory, so each profile can drop you into the folder that goes with it. That same directory is re-applied on every profile switch, so `/profile` and `/use` can move you between projects as well as between accounts. The `--cwd` launch flag always wins over the configured directory when you need to override it for a single run.

---

## A worked example

You have a personal account and a work account, both on the Copilot provider, and a side project that runs on a local OpenAI-compatible server. Three profiles, three files:

```
# ~/.claude/scorpiox-env/personal.txt
PROVIDER=copilot
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/personal.json
MODEL=claude-sonnet-5

# ~/.claude/scorpiox-env/work.txt
PROVIDER=copilot
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/work.json
MODEL=claude-sonnet-5
STARTUP_DIR=~/work

# ~/projects/side/.scorpiox/scorpiox-env/local.txt
PROVIDER=openai
OPENAI_BASE_URL=http://localhost:8080/v1
MODEL=local-model
```

Your day:

```
/profile personal          # saved — this is now your default
cd ~/projects/side
/use local                 # try the local server just for this session
# ...build, test...
/use off                   # back to 'personal', nothing on disk changed
```

When you get home the next day and run SCORPIOX CODE, `personal` is still active because it was saved. The `local` profile only ever existed for that one session. That is the whole point: **temporary switches leave no trace, saved ones do.**

---

## The bottom line

SCORPIOX CODE configuration is a stack, not a file. Defaults at the bottom, your machine in the middle, your project on top, and the environment you launched from above all of them — each layer only overrides the keys it names. A **profile** is a named file of overrides that loads as the top *file* tier when you activate it, and it only changes what it explicitly sets.

And the switch you will use every day: **`/use <name>`** to try a profile for this session and leave no trace, **`/profile <name>`** to make it permanent. Both apply instantly, and both roll back automatically if the new profile does not come up.

---

## Related

- [Using GitHub Copilot CLI Subscription in SCORPIOX CODE](copilot-provider.md)
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md)
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md)
- [Using the OpenAI Provider](openai-provider.md)
- [Project Instructions in SCORPIOX CODE: CLAUDE.md and AGENTS.md](project-instructions.md)
- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md)
