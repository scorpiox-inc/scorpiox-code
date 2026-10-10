# Using OpenCode Zen Subscription in SCORPIOX CODE

You have an **OpenCode Zen** subscription, or you have access to OpenCode's free contributor models, and you want to run them directly in SCORPIOX CODE without setting up a separate API key. The **OpenCode provider** signs you in with the same OAuth **device-code login** the official OpenCode CLI uses, stores your credentials locally, and sends every request to the OpenCode inference endpoint your account already covers.

This page walks through the **device-code login**, how the token is stored and resolved, how to turn the provider on with a profile, how the free-tier tool gate works, how to switch models and accounts (`/profile` and `/use`), how to inspect your account and model list, how to drive a login without a TTY (**machine mode**), and how direct OpenCode access differs from standard API-key usage.

Docs for SCORPIOX CODE @ `e30b171`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have an OpenCode Zen subscription** | Any account the official OpenCode CLI can sign into |
| **You want the free contributor models** | Muse Spark, Nemotron, Ling, MiMo and friends at $0.00 per request |
| **You don't want to manage a vendor API key** | No per-token API billing — requests draw on your OpenCode account |
| **Headless / over SSH** | The device-code flow works over SSH: open a link, enter a code on any device — no browser needed on the box you run from |
| **Driven by SCORPIO BOT** | The login splits into separate `--start` / `--poll` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling an OpenAI-compatible server with an API key (`OPENAI_API_KEY` plus `OPENAI_BASE_URL`), use `PROVIDER=openai`. The two are different billing models for overlapping wire formats — see [OpenCode vs. the OpenAI API-key provider](#opencode-vs-the-openai-api-key-provider).

> **Subscription, not API key.** The OpenCode provider authenticates with OAuth and draws on your OpenCode account allowance — the free-tier quota for contributor models, or your subscription tier for paid models. It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token against a separate API account.

---

## The free models you get

These models are available at no cost through OpenCode Zen contributor access:

| Model ID | Context | Reasoning |
|----------|---------|-----------|
| `muse-spark-1.3-contributor-free` *(default)* | 1.0M tokens | High |
| `muse-spark-1.2-contributor-free` | 1.0M tokens | High |
| `nemotron-3.5-lightning-free` | 200K tokens | On |
| `nemotron-3-ultra-free` | 200K tokens | — |
| `ling-3.0-flash-fin-free` | 200K tokens | — |
| `mimo-v2.5-free` | 200K tokens | — |
| `big-pickle` | 200K tokens | — |

All of them are unlimited at $0.00 per 1M tokens. Paid models in your OpenCode account are reachable too — list everything your account exposes with `scorpiox-opencode-models` (see [Listing the models you can use](#listing-the-models-you-can-use)).

The 1.0M-context Muse Spark models are the reason this provider is worth a profile even if you also run bigger-name subscriptions: a million tokens of context on a free model is a lot of room for a large codebase.

---

## Step 1 — Sign in with the device-code flow

```bash
scorpiox-opencode-login
```

The login starts a device authorization against the OpenCode console and prints a link and a one-time code:

```
====================================================

 OpenCode Zen Device Authorization

====================================================

Open this link to approve login:
  https://opencode.ai/console/device?user_code=XXXX-XXXX&client_id=opencode-cli

Code: XXXX-XXXX

Waiting for approval in browser...
```

Do exactly what it says:

1. **Open the printed link** in a browser — on any device, wherever you are already signed in to OpenCode.
2. **Enter the one-time code** and approve the device.
3. Come back to your terminal. SCORPIOX CODE has been polling in the background (every five seconds, for up to 15 minutes); the moment you approve, it exchanges the code for tokens and saves them.

On success you will see:

```
Logged in successfully!
Saved credentials to: /home/you/.config/opencode/auth.json
Created profile: /home/you/.claude/scorpiox-env/superopencode.txt
  To use in scorpiox code: /profile superopencode (or ACTIVE_PROFILE=superopencode)
```

Three things happened in that moment:

- **The credentials were written to `~/.config/opencode/auth.json`** (mode `0600`) — the same file and shape the official OpenCode CLI uses: `access_token`, `refresh_token`, `token_type`, `expires_in`, `expires_at`, and your `org_id`.
- **Your organization ID was resolved** from the console and stored alongside the token, so requests are attributed to the right org without you typing anything.
- **A ready-to-use profile was written** to `~/.claude/scorpiox-env/superopencode.txt`, which is the switch that turns the provider on.

A few things worth knowing about this flow:

- **The code is one-time and expires in 15 minutes.** If you miss the window, re-run `scorpiox-opencode-login` for a fresh code.
- **Never share the code.** While you are waiting, anyone who sees it can bind the login to their own account.
- **The interactive login always writes the profile**, overwriting a previous one. Re-running the login is the supported way to refresh everything at once.

---

## Step 2 — Turn on the OpenCode provider

The profile the login wrote is the whole setup:

```
# OpenCode Zen / Free Models Profile
PROVIDER=opencode
MODEL=muse-spark-1.3-contributor-free
OPENCODE_MODEL=muse-spark-1.3-contributor-free
OPENCODE_TOKEN_SOURCE=local
OPENCODE_ORG_ID=<your-org-id>
```

Activate it:

```
/profile superopencode     # activate this profile and persist it
/use superopencode         # activate for this session only (no file change)
```

`PROVIDER=opencode` is the switch. `zen` is accepted as an alias for the same provider, so `PROVIDER=zen` behaves identically.

You can also create the profile by hand at `~/.claude/scorpiox-env/superopencode.txt`, or put the same keys in any cascade tier of `scorpiox-env.txt`, or export them as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

> **Set both `MODEL` and `OPENCODE_MODEL` to the same value.** `MODEL` is the generic model key every provider reads; `OPENCODE_MODEL` is the OpenCode-specific override the provider picks up. The login-written profile pins both so there is no ambiguity — if you change one, change the other.

---

## How the token is resolved

On every request SCORPIOX CODE resolves the OpenCode token in this order, and the first match wins:

| Priority | Source |
|----------|--------|
| 1 | `OPENCODE_TOKEN` environment variable (or `OPENCODE_CONSOLE_TOKEN`) |
| 2 | `OPENCODE_TOKEN` in the config cascade |
| 3 | `scorpiox-opencode-fetchtoken -config`, which follows `OPENCODE_TOKEN_SOURCE` |
| 4 | A direct read of `~/.config/opencode/auth.json` |

Two consequences worth knowing:

- **A live `OPENCODE_TOKEN` in the environment always wins**, whatever `OPENCODE_TOKEN_SOURCE` says. If your token "won't change", check the environment before blaming the profile.
- **The resolved token is cached until shortly before it expires.** The stored `expires_at` is checked on each request; inside the last minute the token is re-resolved from its source, so in-flight work never runs on a stale credential.

You can inspect the resolution without making a model request:

```bash
scorpiox-opencode-fetchtoken -local      # read ~/.config/opencode/auth.json
scorpiox-opencode-fetchtoken -config     # follow OPENCODE_TOKEN_SOURCE
scorpiox-opencode-fetchtoken -tcp        # fetch over a raw TCP token server
scorpiox-opencode-fetchtoken -remote     # fetch from an HTTP token endpoint
scorpiox-opencode-fetchtoken --verbose   # add per-step diagnostics on stderr
```

The output is a single JSON line:

```
{"access_token":"...","org_id":"...","expires_at":1767225599}
```

Run it before a long session to confirm the token resolves to the account and org you expect. The line contains a live token — treat it as a secret and never paste it into a ticket or chat.

### Token source configuration

`OPENCODE_TOKEN_SOURCE` controls how `scorpiox-opencode-fetchtoken -config` resolves the token:

| Value | Behaviour |
|-------|-----------|
| `local` *(default)* | Read `~/.config/opencode/auth.json` (a live `OPENCODE_TOKEN` in the environment still wins) |
| `env` | Read `OPENCODE_TOKEN` / `OPENCODE_CONSOLE_TOKEN` from the environment |
| `tcp` | Fetch over a raw TCP token server (`TCP_HOST`, `TCP_PORT`, `TCP_API_KEY`) |
| `remote` | Fetch from an HTTP endpoint (`OPENCODE_REMOTE_URL`) |

For a personal subscription, `local` is all you need. The `tcp` and `remote` sources exist for shared token-server setups — one machine holds the login, a fleet of agents pulls fresh tokens from it — and are not needed for a normal OpenCode login.

> **Shared token sources require their own configuration.** Unlike a personal `local` login, `tcp` and `remote` do not assume anything about your infrastructure. A `tcp` profile must set `TCP_HOST`, `TCP_PORT`, and `TCP_API_KEY`; a `remote` profile must set `OPENCODE_REMOTE_URL` and `TOKEN_HTTP_API_KEY`. If one is missing, `scorpiox-opencode-fetchtoken` stops and names the exact key to set rather than silently trying a default host — so a misconfigured fleet source fails loudly, and no one's endpoint assumptions are baked into the app.

### Token lifetime, refresh, and re-login

The access token issued by the device flow is long-lived — around thirty days by default — and SCORPIOX CODE keeps using it until it is within a minute of expiring. There is no separate refresh command to run on a schedule.

When the token does run out, the fix is one command:

```bash
scorpiox-opencode-login
```

That re-issues a fresh token, rewrites `auth.json`, and refreshes the profile. If you see an authorization error mid-session, this is always the right first move. If a request fails before login has ever happened, the error message tells you directly: `No OpenCode OAuth token found. Run scorpiox-opencode-login or set OPENCODE_TOKEN`.

---

## Configuration reference

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables.

| Key | Default | Description |
|-----|---------|-------------|
| `PROVIDER` | *(empty)* | Set to `opencode` (or `zen`) to activate this provider. |
| `OPENCODE_TOKEN_SOURCE` | `local` | How the token is resolved: `local`, `env`, `tcp`, or `remote`. |
| `OPENCODE_MODEL` | `muse-spark-1.3-contributor-free` | The model the provider runs. Keep in sync with `MODEL`. |
| `MODEL` | *(empty)* | The generic model key. Accepts aliases and full model IDs (see below). |
| `OPENCODE_ORG_ID` | *(auto-detected)* | Your OpenCode organization ID, captured at login. Sent with every request. |
| `OPENCODE_BASE_URL` | `https://opencode.ai/inference/openai/v1/responses` | Inference endpoint. Override only if you are pointed at a different OpenCode-compatible deployment. |
| `OPENCODE_SESSION_ID` | *(auto-generated)* | Session identifier sent with each request. Defaults to a generated `ses_...` id per session; override to pin one. |
| `OPENCODE_TOKEN` | *(empty)* | Use this token directly instead of the stored credential. Intended for token-server setups, not for hand-editing into a profile. |
| `OPENCODE_FREE_TIER_TOOLS` | `0` | Explicit switch for the free-tier tool fingerprint. See the next section. |

> **For a personal subscription you need exactly two keys:** `PROVIDER=opencode` and `OPENCODE_TOKEN_SOURCE=local` (plus the model keys the profile already wrote). Everything else has a working default.

### Inspecting the resolved configuration

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, OPENCODE_TOKEN_SOURCE, OPENCODE_MODEL, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

`--verbose` shows the resolved provider, token source, and model, so you can confirm you are pointed where you expect before a long run.

---

## Free-tier tool fingerprinting (`OPENCODE_FREE_TIER_TOOLS`)

OpenCode's free-tier endpoint checks the shape of each request, and the check that matters is the **tool list**: a request must advertise both a `bash` tool and a `read` tool, or the endpoint rejects it as if it had not come from OpenCode at all. You would see a 403 with a tool-related error on a model you know is free.

SCORPIOX CODE handles this for you, and has done since the gate appeared:

- **When `TOOLS=1` (the default)**, your full tool set is sent — with `Bash`, `CreateFile`, and `InvokeSkill` mapped onto their OpenCode wire names (`bash`, `write`, `skill`) — and a `read` tool is appended so the pair the gate looks for is always present.
- **When `TOOLS=0`, or every individual tool is switched off**, SCORPIOX CODE sends exactly `bash` and `read` and nothing else, so the request still passes the gate even though no tools will actually run.

The `read` tool is real, not decorative: if the model calls it, SCORPIOX CODE reads the file at `filePath` and returns the contents, and your own tool settings are untouched by any of this.

`OPENCODE_FREE_TIER_TOOLS` (default `0`) is the explicit switch for that decoy behaviour. Leave it at `0` — the gate is already satisfied — and reach for `1` only when you are deliberately reproducing a minimal request, or debugging why a free-tier request was rejected. Setting it to `1` makes the intent visible in `scorpiox-config --verbose` output and in profiles you share with a team.

> **Why this exists.** The endpoint distinguishes free-tier clients by what their requests look like, and the tool list is the fingerprint. Matching the shape the official CLI sends is what keeps free models free to use from SCORPIOX CODE — no workarounds, no patched endpoint, nothing to maintain when the official client updates.

---

## Profile switching

The login creates a `superopencode` profile, so switching to OpenCode — and back to whatever you were running — is a profile switch, not a reconfiguration:

```
/profile superopencode     # persistent — writes ACTIVE_PROFILE, survives restarts
/use superopencode         # session-only — gone when the session ends
/profile                   # list profiles and show which is active
/profile off               # deactivate and drop back to the base cascade
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

### More than one account

OpenCode keeps one credential set per machine — the login does not support named accounts, by design, matching the official CLI. To run a second account on the same node, give it its own token source instead of its own file:

- **Point a profile at the environment.** A profile that sets `OPENCODE_TOKEN_SOURCE=env` reads `OPENCODE_TOKEN` (and `OPENCODE_ORG_ID`) from the environment, so you can export a different account's values for the sessions that need it.
- **Point a profile at a token server.** `OPENCODE_TOKEN_SOURCE=tcp` or `remote` pulls the token from a shared service, which is how fleets run many accounts off one login host.

Keep tokens in the environment or on a token server, not pasted into profile files you sync or commit.

---

## Choosing a model

The model aliases at this commit:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `muse-spark-1.3-contributor-free` (provider default) |
| `opus` | `muse-spark-1.3-contributor-free` |
| `sonnet` | `muse-spark-1.3-contributor-free` |
| `haiku` | `muse-spark-1.2-contributor-free` |
| any full model ID (e.g. `nemotron-3.5-lightning-free`) | passed through as-is |

The aliases are there so the same profile works across providers — `opus`, `sonnet`, and `haiku` mean "the biggest, the balanced, and the fast model of whatever provider is active". On OpenCode they land on the two Muse Spark contributor models. Set `MODEL` (and `OPENCODE_MODEL`) in your profile, or switch at runtime:

```
/model nemotron-3.5-lightning-free
```

> Note: the login-written `superopencode` profile ships with `MODEL=muse-spark-1.3-contributor-free` out of the box — the newest Muse Spark with the 1.0M context window. Switch to `muse-spark-1.2-contributor-free` or one of the 200K free models if you want a change of behaviour, and use `/model` to try one without editing anything.

---

## Listing the models you can use

```bash
scorpiox-opencode-models            # pretty-printed table
scorpiox-opencode-models --json     # raw JSON to stdout
```

The table shows model ID, display name, cost (`Free` when the input price is zero), context window, and whether the model reasons:

```
MODEL ID                           NAME                   COST      CONTEXT     REASONING
--------------------------------------------------------------------------------------
muse-spark-1.3-contributor-free    Muse Spark 1.3         Free      1.0M        yes
...
```

The command resolves your token the same way the provider does (environment, then `fetchtoken -config`, then `auth.json`), and sends your organization ID with the request, so the list is exactly what your account can reach — free models always, paid models according to your plan.

---

## Checking your account

```bash
scorpiox-opencode-usage         # human-readable summary
scorpiox-opencode-usage --json  # raw JSON to stdout
```

You get the signed-in account email, the organization name and ID, the authentication status, and the current free-model roster with context windows and reasoning flags. There is no per-token meter to watch — free models are unlimited at $0.00 — so this is the "which account am I actually on, and what can it reach" check rather than a quota readout.

Inside SCORPIOX CODE, the in-session `/usage` command runs this same tool under the hood and shows it as **OpenCode Zen** when the provider is active.

---

## What a request looks like

SCORPIOX CODE talks to OpenCode in the OpenAI **Responses** wire format — streamed end to end, with the reasoning summary surfaced separately from the answer text. Nothing for you to configure; the shape is handled for you. A few visible consequences:

- **Reasoning appears as its own block** when the model produces one, and the status bar reports reasoning tokens alongside output (`rsn:<count>`).
- **Cached input tokens are shown** when the endpoint reports them, so you can see how much of your context was served from cache.
- **Images work.** Images you attach (or that the agent reads with `ReadImage`) are sent as native image input, so a free Muse Spark model can look at a screenshot you paste.
- **Each request carries your session identity.** The session ID is sent with every call and also serves as the prompt-cache key, so consecutive turns on the same session reuse what they can.
- **Statusbar speed figures are opt-in.** With `USAGE_SPEED_ESTIMATE=1`, SCORPIOX CODE times the first streamed token and reports `~pp` / `~tg` throughput; the default (`0`) leaves the status bar as it is.

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open between "here is the link" and "approved" — a background agent, a CI step, or SCORPIO BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node. |
| `--start` | Begin the login: returns the `verify_url` and one-time `user_code`, and keeps the flow state for 15 minutes. |
| `--poll` | One non-blocking check. Reports `pending` until you approve, then saves the token. |
| `--cancel` | Drop a pending login. |
| `--force` | With `--start`: replace existing credentials instead of failing. |
| `--create-profile` | With `--poll`: also write the `superopencode` profile. |

A typical machine-mode login:

```bash
# 1. Start — prints the link and the one-time code
scorpiox-opencode-login --start
# {"ok":true,"provider":"opencode","flow":"device_code","verify_url":"https://opencode.ai/console/device?...","user_code":"ABCD-EFGH","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open verify_url on any device, sign in, enter the code.

# 3. Poll until it reports done (add --create-profile to also write the profile)
scorpiox-opencode-login --poll
#   {"ok":true,"provider":"opencode","state":"pending","interval":5}
#   {"ok":true,"provider":"opencode","state":"done","logged_in":true,"path":"/home/you/.config/opencode/auth.json","profile":"superopencode","profile_created":true}
```

And a status check, which is all SCORPIO BOT needs to see where a node stands:

```bash
scorpiox-opencode-login --status
# {"ok":true,"provider":"opencode","name":"OpenCode","flow":"device_code","account":null,"logged_in":true,"path":"/home/you/.config/opencode/auth.json","expires_at":1767225599,"expired":false,"has_refresh":true,"pending":false,"profile":"superopencode","profile_exists":true,"named_accounts":false,"protocol":"SXLOGIN-MACHINE-V1"}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the flow state lives in `~/.claude/.login-pending/` (mode `0600`), so the calls can be separate processes on the same node.
- **Fifteen-minute window.** If the code is not approved within 15 minutes the pending state expires and `--poll` fails with `expired` — run `--start` again.
- **`--poll` is the finish step, not `--finish`.** OpenCode is a device-code flow: it is `--start` then `--poll`. A `--finish` call is rejected because OpenCode uses the device-code protocol, not paste-code.
- **`--name` is rejected.** OpenCode keeps a single account per node; there are no named account files to list.
- **`--start` refuses to clobber an existing login.** Pass `--force` when you mean to replace the stored credentials.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"opencode","error":"...","detail":"..."}`.
- **`--status` never prints tokens** — only whether a login exists, where, and whether it has expired.

---

## OpenCode vs. the OpenAI API-key provider

It is easy to confuse the two, because OpenCode speaks the OpenAI Responses wire format. Here is the difference:

| | **OpenCode provider** (this page) | **OpenAI API-key provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `opencode` (or `zen`) | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-opencode-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your OpenCode account — free contributor models at $0.00, or your Zen subscription | Pay-per-token API usage, or your own server |
| **Token source** | `OPENCODE_TOKEN_SOURCE` (`local` / `env` / `tcp` / `remote`) | `OPENAI_API_KEY` + `OPENAI_BASE_URL` |
| **Endpoint** | `opencode.ai/inference/openai/v1/responses` | Any OpenAI-compatible `/v1/chat/completions` server |
| **Best for** | You have an OpenCode account, or want the free models | Pay-as-you-go access, local inference, custom endpoints |

Rule of thumb: **you have an OpenCode account, use `opencode`; you have an API key or a self-hosted OpenAI-compatible server, use `openai`.** See [Using the OpenAI Provider](openai-provider.md).

---

## OpenCode vs. the other subscription providers

| | **OpenCode** | **Codex** | **Copilot** | **Claude Code** |
|---|---|---|---|---|
| **Account type** | OpenCode Zen / free contributor | ChatGPT Plus / Pro / Business | GitHub Copilot seat | Anthropic Claude Code subscription |
| **Login** | `scorpiox-opencode-login` | `scorpiox-codex-login` | `scorpiox-copilot-login` | `scorpiox-claudecode-login` |
| **Credential file** | `~/.config/opencode/auth.json` | `~/.codex/auth.json` | `~/.copilot/.credentials.json` | `~/.claude/.credentials.json` |
| **Profile created** | `superopencode` | `codex` | `copilot` | `claude_code` |
| **Default model** | `muse-spark-1.3-contributor-free` | `gpt-5.5` | `claude-sonnet-5` | `claude-opus-4-6` |
| **Free tier** | Yes — the whole roster above is $0.00 | No — draws on your ChatGPT plan | No — draws on your Copilot plan | No — draws on your Claude plan |

All of them use the same device-code login pattern, the same credential-file convention, and the same `/profile` / `/use` switching mechanics. What differs is the account you draw from and the models it exposes — and OpenCode is the one where the default model costs nothing.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `No OpenCode OAuth token found. Run scorpiox-opencode-login or set OPENCODE_TOKEN` | You have not logged in, or `~/.config/opencode/auth.json` is missing. Run `scorpiox-opencode-login`. |
| `error: no token found (run scorpiox-opencode-login)` from `scorpiox-opencode-models` or `-usage` | Same cause — the helper tools read the same credential. Log in first. |
| A 403 with a tool-related error on a free model | The free-tier tool gate did not see `bash` and `read`. Confirm `TOOLS` and the per-tool switches are at their defaults, and that `OPENCODE_FREE_TIER_TOOLS` is `0` unless you deliberately set it. |
| Wrong organization on requests | Set `OPENCODE_ORG_ID` in your profile to the org you want, or re-run `scorpiox-opencode-login` and approve under the right org. |
| Token expired / repeated authorization errors | Re-run `scorpiox-opencode-login` for a fresh token. The device flow reissues everything, including the profile. |
| `OPENCODE_TOKEN_SOURCE=tcp` or `remote` failing | Check `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` or `OPENCODE_REMOTE_URL` / `TOKEN_HTTP_API_KEY`, then test with `scorpiox-opencode-fetchtoken -tcp` or `-remote` to isolate the token server. A missing key reports its own name before any request is sent. |
| Model not found | Run `scorpiox-opencode-models` to see what your account actually exposes. Free models are always there; paid models depend on your plan. |
| Your `OPENCODE_TOKEN` "will not go away" | The environment beats the config cascade. Check for an exported `OPENCODE_TOKEN` before assuming the profile is wrong. |
| Machine-mode login stuck at `pending` | The code expired or was never approved. Run `--cancel`, then `--start` again for a fresh code. |

---

## The bottom line

Run `scorpiox-opencode-login`, approve the device in a browser, and SCORPIOX CODE stores the credential where the official OpenCode CLI keeps it, resolves your organization, and writes a `superopencode` profile that turns the provider on. `/profile superopencode` switches you onto a 1.0M-context free model; `/profile off` switches you back. The token keeps itself fresh until it genuinely expires, the free-tier tool fingerprint is satisfied without you thinking about it, and `scorpiox-opencode-models` and `scorpiox-opencode-usage` tell you what your account can reach. One login, one profile, no API key.

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using the OpenAI Provider](openai-provider.md)
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md)
- [Using GitHub Copilot CLI Subscription in SCORPIOX CODE](copilot-provider.md)
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md)
- [Using Grok Build Subscription in SCORPIOX CODE](grok-provider.md)
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md)
