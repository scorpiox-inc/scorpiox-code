# Reasoning Effort Control

Reasoning effort is the dial that tells a reasoning model how hard to think before it answers: `low` for quick turns, `high` for hard problems, and everything in between. SCORPIOX CODE exposes it two ways at once — a live slash command you can flip mid-conversation, and a set of per-provider config keys that make the choice permanent — and it shows you exactly which one is active instead of hiding it.

The reason there are two surfaces is that the three providers which read reasoning effort do not agree on their key names, their accepted values, or even what "off" means. The slash command is the universal control; the per-provider keys are how you pin it per provider in the configuration cascade.

Docs for SCORPIOX CODE @ `e30b171`.

> **The whole idea in one line:** `/reasoning_effort <low|medium|high|max|off>` changes reasoning effort for the running session on any provider that supports it, and the `*_REASONING_EFFORT` config keys make the same choice permanent — with each provider reading its own key first, then the generic fallbacks.

---

## The one command: `/reasoning_effort`

Type it with no argument and it tells you where you stand:

```
reasoning_effort: high
Usage: /reasoning_effort <low|medium|high|max|off>
```

Set it with a value:

```
/reasoning_effort low       → reasoning_effort: low
/reasoning_effort max       → reasoning_effort: max
/reasoning_effort off       → reasoning_effort: off (not sent)
```

Three properties worth knowing:

- **It is live mid-turn.** The command is deliberately allowed while the agent is busy generating — you can drop effort from `max` to `medium` while a long tool loop is running and the *next* request picks it up. No restart, no new session.
- **`off` means "not sent", not "zero".** The command clears the setting; the field simply disappears from the request and the provider does whatever its default is. Nothing is sent to say "don't reason".
- **Argument completion.** The input line offers `low`, `medium`, `high`, `max`, `off` as you type, filtered by prefix — you never have to remember the exact spelling.

The command works in the interactive TUI and in machine mode (the same handler backs both entry points), so scripts driving the agent can retune effort the same way a human can.

One value is written no matter which provider you are on: the command updates **`OPENAI_REASONING_EFFORT`** — the shared key that every reasoning-effort-reading provider falls back to. See [The fallback chain](#the-fallback-chain-and-the-gotcha-it-creates) for what that means when provider-specific keys are also set.

---

## What each value does per provider

This is the part that actually matters, because the three providers disagree.

| Provider | Key it reads first | Accepted values | What `off` does | Wire field |
|----------|--------------------|-----------------|-----------------|------------|
| **openai** | `OPENAI_REASONING_EFFORT` | `low`, `medium`, `high`, `max` | Field omitted entirely | `reasoning_effort` in the request body |
| **codex** | `CODEX_REASONING_EFFORT` → `REASONING_EFFORT` → `OPENAI_REASONING_EFFORT` | `low`, `medium`, `high`, `max`, `off` | Sent as `none` (the Responses API way of saying "no reasoning") | `reasoning: { effort: ... }` |
| **copilot** | `COPILOT_REASONING_EFFORT` → `REASONING_EFFORT` → `OPENAI_REASONING_EFFORT` | `low`, `medium`, `high` | Field omitted — you get the provider's default thinking | `reasoning_effort` in the request body |

The details behind that table:

**openai** (`PROVIDER=openai` — any OpenAI-compatible endpoint, cloud or self-hosted). The provider reads `OPENAI_REASONING_EFFORT` and passes it straight through as the `reasoning_effort` field in the Chat Completions body. Whether it has any effect is up to the server on the other end: engines that understand the field act on it, engines that ignore it silently drop it. Only the four levels are offered; there is no `off` value here because an empty key already means "do not send the field", which is the same thing. This is the provider where the slash command and the config key are exactly the same wire effect — the command's write to `OPENAI_REASONING_EFFORT` *is* the provider's first-choice key, so the two never disagree.

**codex** (`PROVIDER=codex` — ChatGPT/Codex subscription). The Codex provider maps your value onto the Responses API's reasoning field, and it is the one provider where `off` is a *sent* value: `off` (or `none`, or `0`) becomes `reasoning: { effort: "none" }` on the wire. `low`, `medium`, `high`, and `max` pass through directly, and any other value is passed through verbatim rather than rejected. If no reasoning key is set at all, the provider falls back to its thinking toggle: with `THINKING=1` (the shipped default) the request carries `high`. So on Codex, out of the box, you are already running at `high` — set a key only to change it.

**copilot** (`PROVIDER=copilot` — GitHub Copilot subscription). The Copilot endpoint accepts `low`, `medium`, `high`, and here `off` (or `none`, or `0`) means **omit the field** — the provider's default thinking behavior applies, which is not the same as Codex's explicit `none`. One model-specific wrinkle: if the selected model is a Gemini model, `max` is mapped down to `high` before sending, because the Copilot surface for Gemini models does not accept `max`. On Claude-family Copilot models `max` passes through as-is.

### How fresh is the value?

Fresh every request, on every provider. The Codex and Copilot request builders re-read the configuration for each outgoing request, and the openai provider passes the setting to its request translator on every call. Change the value mid-session — by slash command, by config editor — and the very next request carries the new effort. There is no apply step and no restart.

---

## The fallback chain, and the gotcha it creates

Codex and Copilot do not read only their own key. Each walks a three-step chain, first-set-wins:

```
CODEX_REASONING_EFFORT    →  REASONING_EFFORT  →  OPENAI_REASONING_EFFORT
COPILOT_REASONING_EFFORT  →  REASONING_EFFORT  →  OPENAI_REASONING_EFFORT
```

This design gives you one generic key (`REASONING_EFFORT`) that sets effort for both subscription providers at once, and one shared key (`OPENAI_REASONING_EFFORT`) that covers all three. But it creates the single most important gotcha on this page:

> **`/reasoning_effort` writes the shared key — which is the *last* fallback.** If you have `CODEX_REASONING_EFFORT=high` in any config file and then run `/reasoning_effort low`, Codex keeps sending `high`, because its own key is checked first and it is still set. The slash command only rules when nothing above the shared key is set.

The rule of thumb:

- **Session tuning** (trying values while you work): `/reasoning_effort` — as long as no provider-specific key is set anywhere in your cascade.
- **A pinned per-provider value**: set `CODEX_REASONING_EFFORT` / `COPILOT_REASONING_EFFORT` in the tier that matches how widely you want it, and accept that the slash command will not override it. To retune that provider in-session, clear the provider-specific key first.
- **One knob for all providers**: leave the provider keys unset and use the shared key (or the slash command) — all three providers follow it.

`REASONING_EFFORT`, the generic middle key, is deliberately file-and-environment only: it has no entry in the interactive config editor, so it never gets written by the editor by accident. Put it in a `scorpiox-env.txt` or a profile when you want it.

---

## Persistence: session-live vs permanent

What `/reasoning_effort` actually does is two writes, both in the running process:

1. It sets the shared key in the in-memory configuration.
2. It sets the matching OS environment variable (`OPENAI_REASONING_EFFORT=low`) in the process.

The second write is why the command is authoritative for the rest of the session: **an OS environment variable outranks every file in the cascade**. If your user-level `scorpiox-env.txt` says `OPENAI_REASONING_EFFORT=high` and you run `/reasoning_effort low`, the session-level `low` wins for as long as this process lives. And that is also the trap: the change lives and dies with the process. Restart the session and the file value comes back.

Making it permanent, in the tier that matches how widely you want it:

| You want it for… | Put the key in |
|-------------------|----------------|
| This project only | `.scorpiox/scorpiox-env.txt` (or `.claude/scorpiox-env.txt`) in the repo |
| Every project on this machine, for you | `~/.claude/scorpiox-env.txt` |
| Everyone using this install | `scorpiox-env.txt` next to the installed binaries |
| A named profile you flip with `/profile` / `/use` | `scorpiox-env/<name>.txt` |

The mechanical way, without opening an editor:

```bash
scorpiox-config --set OPENAI_REASONING_EFFORT high          # project tier (default)
scorpiox-config --set OPENAI_REASONING_EFFORT high --level user
scorpiox-config --set CODEX_REASONING_EFFORT max  --level user
```

The interactive editor (`/config` or `scorpiox-config`) offers the same keys as multiple-choice entries in the **Provider & Authentication** section — `OPENAI_REASONING_EFFORT` and `OPENAI_EXTRA_HEADERS` when `PROVIDER=openai` is active, `CODEX_REASONING_EFFORT` when codex is active, `COPILOT_REASONING_EFFORT` when copilot is active — so you pick from the legal values instead of typing them.

---

## Where you can see the current value

Three places, all live:

- **The info panel** shows a `reasoning: <value>` line — but only when a value is set. If the line is missing, the shared key is empty and every provider is on its default path.
- **The session's `infobox.json`** carries the same value in its `reasoning` field for tools that want to read it programmatically.
- **The captured traffic.** Every request body in the session's traffic capture (`raw/` request logs) contains the exact reasoning field that went out — the ground truth for "what did the provider actually get".

And to see what would resolve after a restart, ask the config tool:

```bash
scorpiox-config --get OPENAI_REASONING_EFFORT     # effective value of the shared key
scorpiox-config --get CODEX_REASONING_EFFORT      # per-provider key
scorpiox-config --verbose                         # every key + which tier it came from
```

`--verbose` is the arbiter for the fallback-chain gotcha: it shows whether a provider-specific key is set anywhere in your cascade, which is exactly what determines whether your slash-command value or your file value wins.

---

## How the rest of the lineup handles it

Reasoning effort is one of two control models in SCORPIOX CODE, and knowing which model a provider uses tells you which knob to reach for:

| Provider | Control model | What you get |
|----------|---------------|--------------|
| **openai / codex / copilot** | Reasoning effort (this page) | Discrete levels, one key per provider, visible in the UI |
| **claude_code / anthropic / scorpiox (router)** | Thinking budget | `THINKING=1/0` on/off plus `THINKING_BUDGET` tokens — a budget, not a level |
| **grok** | Fixed mapping | `THINKING=1` sends `high`, `THINKING=0` sends `low` — no key, no choice (the endpoint rejects `none`) |
| **opencode** | Fixed | `high` whenever thinking is enabled — fully hidden |

The thinking-budget providers are worth a word: there is no reasoning effort key on Anthropic-style providers because their API model is different — you enable thinking and cap it with a token budget rather than picking a level. `THINKING_BUDGET=10000` (the shipped default) is that cap. If you switch a profile from `codex` to `claude_code`, your `CODEX_REASONING_EFFORT` goes dormant and `THINKING`/`THINKING_BUDGET` take over — the two models of control do not translate into each other.

Against the mainstream harnesses: the Codex CLI exposes the same dial as a config-file value (`model_reasoning_effort`) that you edit and reload; Claude Code steers thinking with reserved prompt keywords and a `MAX_THINKING_TOKENS` environment variable rather than a discrete level; the Gemini CLI keeps generation parameters in its settings file. SCORPIOX CODE's approach differs on two axes — the dial is a first-class slash command you can flip mid-conversation, and the same mental model covers three different providers whose underlying fields disagree — which is why the per-provider mapping table above exists at all.

---

## Configuration reference

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `OPENAI_REASONING_EFFORT` | choice | *(empty)* | Shared reasoning effort key. `low`, `medium`, `high`, `max`. Read directly by the openai provider; read as the last fallback by codex and copilot. Empty = field not sent (provider default). This is the key `/reasoning_effort` writes. |
| `CODEX_REASONING_EFFORT` | choice | *(empty)* | Codex provider effort. `low`, `medium`, `high`, `max`, `off`. `off`/`none`/`0` sends `none` on the wire. Overrides the generic and shared keys. Falls back to `THINKING=1` → `high` when all keys are empty. Visible in the config editor when `PROVIDER=codex`. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Copilot provider effort. `low`, `medium`, `high`. `off`/`none`/`0` omits the field. `max` maps to `high` on Gemini-family Copilot models. Overrides the generic and shared keys. Visible in the config editor when `PROVIDER=copilot`. |
| `REASONING_EFFORT` | text | *(empty)* | Generic effort key for codex + copilot (middle of both fallback chains). File/environment only — not offered by the interactive config editor. |
| `THINKING` | bool | `1` | Thinking toggle for the budget-based providers, and the codex fallback: with all reasoning keys empty, `THINKING=1` makes codex send `high`. |

All five keys resolve through the standard cascade — global, user, project, profile, OS environment, highest tier wins — and the provider-specific keys always outrank the generic and shared keys within their provider's chain.

---

## Gotchas

- **`off` means three different things.** openai: omit the field. codex: send explicit `none`. copilot: omit the field and accept provider-default thinking. None of them means "the model spent zero effort" — they mean "SCORPIOX CODE stopped specifying".
- **The slash command writes the shared key, so provider-specific keys silence it.** `CODEX_REASONING_EFFORT` set anywhere in your cascade beats `/reasoning_effort` for the Codex provider. Check `scorpiox-config --verbose` when a slash-command change seems ignored.
- **Live changes do not survive a restart.** The slash command writes an OS environment variable for the process; the next session re-reads the files. Persist with `scorpiox-config --set` (or the editor) when you want it to stick.
- **Empty beats unset in surprising ways.** The chain checks are "is it non-empty", not "is it present". A key set to empty string is treated as unset and the chain moves on — which is exactly what `/reasoning_effort off` relies on.

---

## Related

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Using the OpenAI Provider](openai-provider.md) — the provider that reads the shared key directly, plus `OPENAI_CHAT_TEMPLATE_KWARGS` for engine-side thinking toggles.
- [Using OpenAI Codex & ChatGPT Subscription](codex-provider.md) — the Responses API reasoning field and the subscription billing model.
- [Using GitHub Copilot CLI Subscription](copilot-provider.md) — the Copilot surface, its model catalog, and the Gemini `max` mapping.
- [Session Identity Headers](identity-headers.md) — the other per-request field you can set live and see on the wire.
