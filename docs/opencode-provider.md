# Using OpenCode Zen Subscription in SCORPIOX CODE

You have an **OpenCode Zen** subscription, or you have access to OpenCode's free contributor models, and you want to run them directly in SCORPIOX CODE without setting up a separate API key. The **OpenCode provider** signs you in with the OAuth **device-code flow** (the same one the official OpenCode CLI uses), stores your credentials locally, and sends every request to the OpenCode inference endpoint your account already covers.

One `scorpiox-opencode-login` sign-in is all it takes. You approve in a browser, SCORPIOX CODE polls in the background, stores the token in `~/.config/opencode/auth.json`, writes a ready-to-use `superopencode` profile, and every subsequent request reads its own token from that stored credential.

This page walks through the **device-code login**, where the token is stored, how token resolution works, how to turn the provider on with a profile, how the free-tier tool fingerprint works, how to switch models and accounts (`/profile` and `/use`), how to inspect your account and model list, how to drive a login without a TTY (**machine mode**), and how direct OpenCode access differs from standard API-key usage.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** run `scorpiox-opencode-login`, approve the device in a browser on any device, and SCORPIOX CODE stores the credential where the official OpenCode CLI keeps it, writes a ready-to-use `superopencode` profile, and every request draws on your OpenCode account. No API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have an OpenCode Zen subscription** | Any account the official OpenCode CLI can sign into |
| **You want the free contributor models** | Muse Spark, Nemotron, Ling, MiMo and friends at $0.00 per request |
| **You don't want to manage a vendor API key** | No per-token API billing — requests draw on your OpenCode account |
| **Headless or over SSH** | The device-code flow works over SSH: open a link on any device, enter the one-time code — no browser needed on the box you run from |
| **Driven by SCORPIOX BOT or CI** | The login splits into separate `--start` / `--poll` commands with no open TTY — see [Machine mode (no TTY)](#machine-mode-no-tty) |

If you are instead calling an OpenAI-compatible server with an **API key** (`OPENAI_API_KEY` plus `OPENAI_BASE_URL`), that is a different billing model — use `PROVIDER=openai`. The two speak similar wire formats but are different billing models. See [OpenCode vs. the OpenAI API-key provider](#opencode-vs-the-openai-api-key-provider) below.

> **Subscription, not API key.** The OpenCode provider authenticates with OAuth and draws on your OpenCode account allowance — the free-tier quota for contributor models, or your subscription tier for paid models. It does **not** accept an `OPENAI_API_KEY`, and it does **not** bill per token against a separate API account.

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

You can also run it through the provider dispatcher:

```bash
scorpiox-login opencode          # same thing, via the provider dispatcher
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

1. **Open the printed link** in a browser — on any device, wherever you are already signed in to your OpenCode account.
2. **Enter the one-time code** and approve the device.
3. Come back to your terminal and wait — SCORPIOX CODE polls automatically (every few seconds, for up to 15 minutes), exchanges the result for an access and refresh token, resolves your organization ID, saves everything to `~/.config/opencode/auth.json`, and writes the `superopencode` profile in one shot.

On success you will see:

```
Saved credentials to: /home/you/.config/opencode/auth.json
Created profile: /home/you/.claude/scorpiox-env/superopencode.txt
  To use in scorpiox code: /profile superopencode (or ACTIVE_PROFILE=superopencode)
```

A few things worth knowing about this flow:

- **The code expires in 15 minutes.** If the wait times out, re-run `scorpiox-opencode-login` for a fresh code.
- **Never share the code.** It is a credential. Anyone who has the code (and is watching the terminal) can bind it to their own account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up.
- **The credentials file is mode `0600`.** It holds a live refresh token that can renew your account session. Do not commit it, do not share it, and do not loosen its permissions.
- **Overwriting existing credentials.** If a credential already exists, re-running the login replaces it. Pass `--force` explicitly when you mean to clobber a previous login.

### If you just want to check a login

```bash
scorpiox-opencode-login --status
```

This prints a single JSON line with the login state: whether you are logged in, where the credential file lives, the token expiry, whether a refresh token is stored, and whether a login is mid-flight. Use it to decide whether a machine needs a login at all.

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
/profile superopencode     # persistent — writes ACTIVE_PROFILE, survives restarts
/use superopencode         # session-only — gone when the session ends
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
scorpiox-opencode-fetchtoken -http       # fetch from an HTTP token endpoint
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
| `tcp` | Fetch over a raw TCP token server (`TCP_HOST`, `TCP_PORT`, `TCP_API_KEY` must be set) |
| `http` | Fetch from an HTTP endpoint (`OPENCODE_REMOTE_URL` and `TOKEN_HTTP_API_KEY` must be set; `remote` is accepted as an alias) |

For a personal subscription, `local` is all you need. The `tcp` and `http` sources exist for shared token-server setups — one machine holds the login, a fleet of agents pulls fresh tokens from it — and are not needed for a normal OpenCode login.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Default | Purpose |
|-----|---------|---------|
| `PROVIDER` | *(empty)* | Set to `opencode` (or `zen`) to activate this provider. |
| `MODEL` | *(empty)* | The model to use (alias or full model ID). |
| `OPENCODE_MODEL` | `muse-spark-1.3-contributor-free` | OpenCode-specific model override. Keep in sync with `MODEL`. |
| `OPENCODE_TOKEN_SOURCE` | `local` | How the token is resolved: `local`, `env`, `tcp`, or `http`. |
| `OPENCODE_ORG_ID` | *(empty)* | Your organization ID from the OpenCode console. Written by the login; set manually if needed. |
| `OPENCODE_BASE_URL` | `https://opencode.ai/inference/openai/v1/responses` | Inference endpoint. Override for custom deployments. |
| `OPENCODE_SESSION_ID` | *(auto-generated)* | Session identifier sent with each request. Defaults to a generated `ses_...` id per session; override to pin one. |
| `OPENCODE_TOKEN` | *(empty)* | Use this token directly instead of the stored credential. Intended for token-server setups, not for hand-editing into a profile. |
| `OPENCODE_FREE_TIER_TOOLS` | `0` | Explicit switch for the free-tier tool fingerprint. See [Free-tier tool fingerprinting](#free-tier-tool-fingerprinting). |

> **For a personal subscription you need exactly two keys:** `PROVIDER=opencode` and `OPENCODE_TOKEN_SOURCE=local` (plus the model keys the profile already wrote). Everything else has a working default.

### Inspecting the resolved configuration

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values with their source tier
```

`--verbose` shows the resolved provider, token source, and model, so you can confirm you are pointed where you expect before a long run.

---

## Free-tier tool fingerprinting (`OPENCODE_FREE_TIER_TOOLS`)

OpenCode's free-tier endpoint checks the shape of each request, and the check that matters is the **tool list**: a request must advertise both a `bash` tool and a `read` tool, or the endpoint rejects it with a 403 `FreeTierError` ("can only be used from within OpenCode"). You would see this on a model you know is free, on a request that looks like it came from a client other than the official OpenCode CLI.

SCORPIOX CODE handles this for you automatically:

- **When `TOOLS=1` (the default)**, your full tool set is sent — with `Bash`, `CreateFile`, and `InvokeSkill` mapped onto their OpenCode wire names (`bash`, `write`, `skill`) — and a `read` tool is appended so the pair the gate looks for is always present.
- **When `TOOLS=0`, or every individual tool is switched off**, SCORPIOX CODE sends exactly `bash` and `read` and nothing else, so the request still passes the gate even though no tools will actually run.

The `read` tool is real, not decorative: if the model calls it, SCORPIOX CODE reads the file at the `filePath` parameter and returns the contents. Your own tool settings are untouched by any of this.

`OPENCODE_FREE_TIER_TOOLS` (default `0`) is the explicit switch for the decoy behaviour. Leave it at `0` — the gate is already satisfied. Reach for `1` only when you are deliberately reproducing a minimal request, or debugging why a free-tier request was rejected. Setting it to `1` makes the intent visible in `scorpiox-config --verbose` output and in profiles you share with a team.

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

By default the login writes to `~/.config/opencode/auth.json`. There is one OpenCode account per node — there are no named account files to list. To switch accounts, re-run the login with `--force` or set `OPENCODE_TOKEN` in the environment for the sessions that need a different account.

- **Point a profile at the environment.** A profile that sets `OPENCODE_TOKEN_SOURCE=env` reads `OPENCODE_TOKEN` (and `OPENCODE_ORG_ID`) from the environment, so you can export a different account's values for the sessions that need it.
- **Point a profile at a token server.** `OPENCODE_TOKEN_SOURCE=tcp` or `http` pulls the token from a shared service, which is how fleets run many accounts off one login host.

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

- **Reasoning appears as its own block** when the model produces one, and the status bar reports reasoning tokens alongside output.
- **Cached input tokens are shown** when the endpoint reports them, so you can see how much of your context was served from cache.
- **Images work.** Images you attach (or that the agent reads with `ReadImage`) are sent as native image input, so a free Muse Spark model can look at a screenshot you paste.

---

## Machine mode (no TTY)

When driven by SCORPIOX BOT, a UI, or CI where there is no open terminal, the device-code flow splits into explicit, JSON-output steps:

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line. |
| `--start` | Begin the login: returns the verification URL and one-time code. Keeps the pending state for 15 minutes. |
| `--poll` | One authorization check. Returns `pending` or `done`. |
| `--cancel` | Drop the pending login. |
| `--create-profile` | With `--poll`: also write the `superopencode` profile. |
| `--force` | With `--start`: replace existing credentials. |

A typical machine-mode login:

```bash
# 1. Start — prints the verification URL and code
scorpiox-opencode-login --start
# {"ok":true,"provider":"opencode","flow":"device_code","verify_url":"https://opencode.ai/console/device?...","user_code":"ABCD-EFGH","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open verify_url on any device, sign in, enter the code.

# 3. Poll until it reports done (add --create-profile to also write the profile)
scorpiox-opencode-login --poll --create-profile
#   {"ok":true,"provider":"opencode","state":"pending","interval":5}
#   {"ok":true,"provider":"opencode","state":"done","logged_in":true,"path":"/home/you/.config/opencode/auth.json","profile":"superopencode","profile_created":true}
```

And a status check, which is all SCORPIOX BOT needs to see where a node stands:

```bash
scorpiox-opencode-login --status
# {"ok":true,"provider":"opencode","name":"OpenCode","flow":"device_code","account":null,"logged_in":true,"path":"/home/you/.config/opencode/auth.json","expires_at":1767225599,"expired":false,"has_refresh":true,"pending":false,"profile":"superopencode","profile_exists":true,"named_accounts":false,"protocol":"SXLOGIN-MACHINE-V1"}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the flow state lives in `~/.claude/.login-pending/` (mode `0600`), so the calls can be separate processes on the same node.
- **Fifteen-minute window.** If the code is not approved within 15 minutes the pending state expires and `--poll` fails with `expired` — run `--start` again.
- **`--poll` is the finish step, not `--finish`.** OpenCode is a device-code flow: it is `--start` then `--poll`. A `--finish` call is rejected because OpenCode uses the device-code protocol, not paste-code.
- **`--name` is rejected.** OpenCode keeps a single account per node; there are no named account files to list.
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
| **Token source** | `OPENCODE_TOKEN_SOURCE` (`local` / `env` / `tcp` / `http`) | `OPENAI_API_KEY` + `OPENAI_BASE_URL` |
| **Endpoint** | `opencode.ai/inference/openai/v1/responses` | Any OpenAI-compatible `/v1/chat/completions` server |
| **Best for** | You have an OpenCode account, or want the free models | Pay-as-you-go access, local inference, custom endpoints |

Rule of thumb: **you have an OpenCode account, use `opencode`; you have an API key or a self-hosted OpenAI-compatible server, use `openai`.**

---

## OpenCode vs. the other subscription providers

| | **OpenCode** | **Codex** | **Copilot** | **Claude Code** |
|---|---|---|---|---|
| **Account type** | OpenCode Zen / free contributor | ChatGPT Plus / Pro / Business | GitHub Copilot seat | Anthropic Claude Code subscription |
| **Login** | `scorpiox-opencode-login` | `scorpiox-codex-login` | `scorpiox-copilot-login` | `scorpiox-claudecode-login` |
| **Credential file** | `~/.config/opencode/auth.json` | `~/.codex/auth.json` | `~/.copilot/.credentials.json` | `~/.claude/.credentials.json` |
| **Profile name** | `superopencode` | `codex` | `copilot` | `claude_code` |
| **Free tier** | Yes — contributor models at $0.00 | No | No | No |

All four are OAuth session logins. The difference is what the subscription covers and what the endpoint enforces.

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-opencode-login` writes `~/.config/opencode/auth.json` and the `superopencode` profile; it does not switch your session. Activate the provider with `/profile superopencode` (or set `PROVIDER=opencode` yourself) before SCORPIOX CODE uses it.
- **`OPENCODE_TOKEN_SOURCE=local` still requires a login.** The value means "read the token from `~/.config/opencode/auth.json`". If you have never signed in, that file does not exist and every request fails with a token error. Run `scorpiox-opencode-login` first.
- **The credentials file holds a live refresh token.** `~/.config/opencode/auth.json` can renew your account session — do not commit it, do not share it, and do not loosen its permissions.
- **`/profile` persists; `/use` does not.** Use `/profile` to make an OpenCode account your standing default; use `/use` to hop to it for a single session without writing anything.
- **Re-login is the fallback for expired credentials.** If your token expires and the refresh no longer works (account changed, plan changed, session revoked), re-run `scorpiox-opencode-login --force` to re-bind.
- **The client identity matters.** The endpoint varies its response by `user-agent` and client headers. SCORPIOX CODE sends the same client identity the official OpenCode CLI uses. If you write a custom client or proxy, match the identity or you will see models missing or `UNSUPPORTED_CLIENT` errors.
- **`OPENCODE_ORG_ID` matters for paid models.** Free models are available to all authenticated accounts, but paid models are scoped to your organization. If the login did not resolve an org ID, set `OPENCODE_ORG_ID` in your profile explicitly.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the switch cannot complete.

---

## The bottom line

Run `scorpiox-opencode-login`, approve the device in a browser on any device, and SCORPIOX CODE stores the credential where the official OpenCode CLI keeps it, resolves your organization, and writes a `superopencode` profile that turns the provider on. `/profile superopencode` switches you onto a 1.0M-context free model; `/profile off` switches you back. The token keeps itself fresh until it genuinely expires, the free-tier tool fingerprint is satisfied without you thinking about it, and `scorpiox-opencode-models` and `scorpiox-opencode-usage` tell you what your account can reach. One login, one profile, no API key.

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Using the OpenAI Provider](openai-provider.md) — the API-key alternative for OpenAI-compatible endpoints.
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md) — the same OAuth session-login pattern against ChatGPT.
- [Using GitHub Copilot CLI Subscription in SCORPIOX CODE](copilot-provider.md) — the same OAuth device-auth pattern against GitHub.
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md) — the same OAuth session-login pattern against Anthropic.
- [Using Grok Build Subscription in SCORPIOX CODE](grok-provider.md) — the same OAuth device-auth pattern against xAI.
- [Using Antigravity (Google) Subscription in SCORPIOX CODE](antigravity-provider.md) — the same OAuth session-login pattern against Google.
