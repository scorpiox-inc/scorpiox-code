# Using Grok Build Subscription in SCORPIOX CODE

You have an **xAI Grok Build** subscription — the same sign-in the official Grok CLI uses — and you want SCORPIOX CODE to use that subscription directly. No `XAI_API_KEY`, no pay-per-token API billing.

One `scorpiox-grok-login` sign-in is all it takes. You approve in a browser, the session lands in `~/.grok/auth.json`, SCORPIOX CODE writes a ready-to-use `grok` profile, and every subsequent request reads and refreshes its own token from that stored session.

This page walks through the **OAuth device-auth session login**, where the token is stored, how refresh works, how to switch accounts and models with profiles (`/profile` and `/use`), how to check a login without a TTY (machine mode), and how the Grok Build subscription provider differs from the standard xAI API-key provider.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** sign in with `grok login --device-auth`, and SCORPIOX CODE reads the session from `~/.grok/auth.json`, keeps it fresh automatically, and runs every request against the Grok Build endpoint your subscription already pays for. No API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Grok Build subscription** | The xAI plan you already pay for and sign into with the official Grok CLI |
| **You don't have (or don't want) an `XAI_API_KEY`** | No pay-per-token billing — requests draw on your subscription allowance |
| **Headless or over SSH** | The device-auth flow needs no browser or localhost redirect on the box — open the link on any device, enter the one-time code |
| **Driven by SCORPIOX BOT or CI** | The login state can be checked as a single JSON command (`--status`) with no open TTY — see [Machine mode (no TTY)](#machine-mode-no-tty) |

If you are instead calling the xAI API with an **API key** (`XAI_API_KEY` against `https://api.x.ai`), that is a different billing model — see [Grok provider vs. the xAI API-key path](#grok-provider-vs-the-xai-api-key-path) below.

> **Subscription, not API key.** The Grok provider authenticates with an OAuth session login and draws on your account's subscription allowance (the weekly and monthly credit windows xAI enforces). It does **not** accept an `XAI_API_KEY` for chat, and it does **not** bill per token. The chat endpoint is `cli-chat-proxy.grok.com`, **not** `api.x.ai`.

---

## Step 1 — Sign in with the Grok CLI

The Grok provider reads its OAuth token from the same file the official Grok CLI writes: **`~/.grok/auth.json`**. SCORPIOX CODE ships a login helper that wraps that flow and writes a ready-to-use profile in one shot:

```bash
scorpiox-grok-login
```

It first looks for the official **`grok`** CLI on your `PATH`. When it's present, it runs the device-auth flow for you:

```bash
grok login --device-auth
```

That is the OAuth **device-auth flow** — deliberately headless- and SSH-friendly. The CLI prints a link and a one-time code, you sign in from a browser on whatever device already has your account open, enter the code, and the CLI completes the exchange and saves the session to `~/.grok/auth.json`. There is no redirect server and no pasting a callback URL back.

Do exactly what the CLI tells you:

1. **Open the printed link** in a browser — on any device, wherever you're signed into your xAI / Grok Build account.
2. **Sign in** and **enter the one-time code**.
3. Come back to your terminal and wait — the CLI completes the device exchange and saves the session.

A few things worth knowing about this flow:

- **The code window is short.** If the wait times out, just re-run `scorpiox-grok-login` and a fresh code is issued.
- **Never share the code.** It is a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up — this is the main reason the device-auth flow is the default.

### If the Grok CLI isn't installed

`scorpiox-grok-login` prefers the official binary but degrades gracefully. If `grok` isn't on your `PATH`, it tells you how to get the credentials either way:

```bash
# Option A — install the official CLI and let it run the device-auth flow
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth

# Option B — copy an existing session from a machine you've already signed into
scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

The `--profile-only` flag skips the device-auth launch entirely and just writes the profile — useful when `~/.grok/auth.json` already exists on this machine from a prior sign-in.

### The `grok` profile is written for you

On a successful sign-in, SCORPIOX CODE writes a profile so you can activate the provider without hand-tuning configuration:

```
# ~/.claude/scorpiox-env/grok.txt
PROVIDER=grok
MODEL=grok-4.6
GROK_MODEL=grok-4.6
GROK_TOKEN_SOURCE=local
```

The `grok` CLI login protocol is not something SCORPIOX CODE owns, so the profile ships the defaults above. `GROK_TOKEN_SOURCE=local` means "read `~/.grok/auth.json` (or `$GROK_HOME/auth.json`)" — the standard personal-subscription setup.

---

## Step 2 — Turn on the Grok provider

Activate the provider in-session:

```text
/profile grok                   # persistent — writes ACTIVE_PROFILE, survives restarts
/use grok                       # session-only — gone when the session ends
/profile                        # list profiles and the active one
/profile off                    # deactivate and drop back to the base cascade
```

Or set the environment variable for a non-interactive session:

```bash
ACTIVE_PROFILE=grok scorpiox
# or
PROVIDER=grok GROK_TOKEN_SOURCE=local scorpiox
```

`/profile` and `/use` both trigger a live provider reload, so the new provider takes effect immediately. If the switch cannot complete, SCORPIOX CODE reverts to the previous profile — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local`, `http`/`remote`, `ssh`, `tcp`, or `config`. Use **`local`** for a login you ran with `scorpiox-grok-login`. |
| `GROK_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.grok/auth.json`. Point it at a named account (e.g. `~/.grok-accounts/work/auth.json`) to pin a profile to a specific login. |
| `GROK_HOME` | text | *(empty)* | Alternate home directory for the local credential file. Resolved before the default `~/.grok/` path. |
| `MODEL` / `GROK_MODEL` | text | `grok-4.6` | Which model to run. Accepts short aliases (`opus`, `sonnet`, `haiku`) or full `grok-*` model IDs (see [Choosing a model](#choosing-a-model)). |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=http`. |
| `GROK_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. `GROK_SSH_HOST` and `GROK_SSH_USER` are required in that mode. |
| `GROK_API_BASE` | text | *(empty)* | Alternate `api.x.ai` base for the model-list helper. Falls back to the `XAI_BASE_URL` key if set. |

> **For a personal subscription you only need two keys:** `PROVIDER=grok` and `GROK_TOKEN_SOURCE=local`. The `http`, `ssh`, and `tcp` token sources exist for shared or remote token setups — a token service that hands out tokens to many machines — and are not part of a normal Grok Build login.

### Where the credentials live

On login, the Grok CLI writes the OAuth session to `~/.grok/auth.json`. The file holds one or more OIDC entries keyed by `<issuer>::<client>`:

```json
{
  "https://...::client-id": {
    "key": "<access token>",
    "refresh_token": "<refresh token>",
    "expires_at": "2026-11-26T12:00:00.000000Z",
    "oidc_issuer": "https://...",
    "client_id": "client-id",
    "principal_type": "...",
    "principal_id": "..."
  }
}
```

- **`key`** — the current access token, refreshed automatically by the provider. You never manage it.
- **`refresh_token`** — long-lived, used by `scorpiox-grok-refreshtoken` to renew the access token via the OIDC `token_endpoint`. This is the credential that keeps your session alive between logins.
- **`expires_at`** — ISO-8601 timestamp. The provider treats the token as "expiring soon" five minutes before this instant and refreshes it on the next request.

The file is written with `0600` permissions on refresh. Do not commit it, do not share it, and do not loosen its permissions.

---

## Token persistence and automatic refresh

The provider reads the token from `~/.grok/auth.json` on every request and checks the `expires_at` value. When the stored token is within five minutes of its expiry, or the file is missing/empty, it spawns `scorpiox-grok-refreshtoken --force` to renew the session in place — then continues with the freshly-written access token. If the refresh fails (expired refresh token, account or plan changed), the provider still tries to read the file once more before surfacing the error to you.

You can also refresh manually:

```bash
# 1. Refresh the token in place (no re-login needed)
scorpiox-grok-refreshtoken
scorpiox-grok-refreshtoken --force      # ignore the 5-minute buffer and refresh now

# 2. If that keeps failing, re-bind with a full login
scorpiox-grok-login
```

### Inspect your subscription usage

The Grok Build subscription enforces usage windows (weekly and monthly credit limits). You can see the utilization and reset times:

```bash
scorpiox-grok-usage            # readable usage summary (Weekly/Monthly limit NN% used, resets in ...)
scorpiox-grok-usage --json     # raw billing JSON to stdout
```

Inside SCORPIOX CODE, the in-session `/usage` command runs this same tool under the hood and shows the usage table when a `grok` provider is active.

---

## Profiles and switching

### Store more than one login

If you have multiple Grok Build accounts (personal and work), give each one its own credential file and profile:

```bash
# sign in to the personal account first
GROK_HOME=~/.grok-accounts/personal scorpiox-grok-login

# then the work account
GROK_HOME=~/.grok-accounts/work scorpiox-grok-login --profile-only
```

`GROK_HOME` redirects the credential file path (the default is `~/.grok/`). Write one profile per account that pins the matching credentials file:

```
# ~/.claude/scorpiox-env/grok-personal.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok-accounts/personal/auth.json
MODEL=grok-4.6
```

```
# ~/.claude/scorpiox-env/grok-work.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok-accounts/work/auth.json
MODEL=grok-4.6
```

If `GROK_CREDENTIALS_FILE` is empty, the provider falls back to `~/.grok/auth.json`.

### Switch accounts and models in-session

With those profiles in place, switching accounts or models is just switching profiles — no re-login, no restart:

```text
/profile grok-work          # persistent — writes ACTIVE_PROFILE, survives restarts
/use grok-personal         # session-only — gone when the session ends
/profile                   # list profiles and the active one
/profile off               # deactivate and drop back to the base cascade
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the switch cannot complete, SCORPIOX CODE reverts to the previous profile — you are never left half-switched.

### Inspect what is actually configured

Use `scorpiox-config` to see resolved values and where each one comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, GROK_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `GROK_TOKEN_SOURCE`, and `MODEL`, so you can confirm you are pointed at the account and model you expect before a long run.

---

## Choosing a model

`MODEL` in the profile (or the `GROK_MODEL` key) selects the model. The provider accepts a short alias or a full model ID, and resolves it before the request is sent.

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `grok-4.6` (provider default) |
| `opus` | `grok-4.6` |
| `sonnet` | `grok-4.6` |
| `haiku` | `grok-4.5` |
| any `grok-*` ID | passed through as-is |

You can list every model the endpoint advertises with:

```bash
scorpiox-grok-models              # JSON list of available models
```

### Image generation (Grok Imagine)

The Grok Build session can also drive the Grok Imagine image API. Use `scorpiox-grok-imagegen` to generate or edit images from your subscription (auth via the same `~/.grok/auth.json` session, or an explicit `--token`):

```bash
# Generate
scorpiox-grok-imagegen --prompt "Piha Beach Lion Rock, storm opening to blue" \
    --output /tmp/out.png --aspect 16:9 --resolution 2k --quality medium

# Edit (up to 3 source images)
scorpiox-grok-imagegen --prompt "add an RTX PRO 6000 on the sand" --image /tmp/piha.png

# List models
scorpiox-grok-imagegen --models
```

Defaults: `grok-imagine-image-2.0` model, `16:9` aspect, `2k` resolution, `medium` quality, `https://api.x.ai/v1` endpoint. Precedence is CLI flag, then environment variable, then `scorpiox-env.txt`. Do not pass `--token` in a way that lands in shell history.

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open between "here is the link" and "enter the code" — a background agent, a CI step, or SCORPIOX BOT's provider flow. The Grok login ships an **additive machine mode**: because the device-auth protocol is owned by the official `grok` CLI (not by SCORPIOX CODE), machine mode is **status-only** with a `terminal` flow — the bot reports the state and offers a terminal login rather than driving the exchange itself.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line. |
| `--cancel` | Drop any pending state (a no-op for the terminal flow). |

Example:

```bash
scorpiox-grok-login --status
# {"ok":true,"provider":"grok","name":"Grok (xAI)","flow":"terminal","logged_in":true,
#  "path":"/home/you/.grok/auth.json","identity":"grok CLI installed",
#  "has_refresh":true,"profile":"grok","profile_exists":true,...}
```

- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`.
- **`--status` tells you where you stand.** It reports `logged_in`, whether a refresh token is stored (`has_refresh`), whether the `grok` profile exists, and whether the `grok` CLI is installed. Use it to decide whether a machine needs a login at all.
- **Tokens are never printed.** Only status.

---

## Grok provider vs. the xAI API-key path

It is easy to confuse the two because they both run Grok models. Here is the difference:

| | **Grok provider** (this page) | **xAI API-key path** (`XAI_API_KEY`) |
|---|---|---|
| `PROVIDER` value | `grok` | `openai` (or a custom endpoint pointed at `api.x.ai`) |
| **Authentication** | OAuth device-auth session login (`grok login --device-auth`) | `XAI_API_KEY` bearer token |
| **Billing** | Your Grok Build subscription allowance (weekly + monthly credit windows) | Pay-per-token API usage against your key |
| **Token source** | `GROK_TOKEN_SOURCE` (local file / http / ssh / tcp) | `XAI_API_KEY` environment variable |
| **Chat endpoint** | `cli-chat-proxy.grok.com/v1/responses` | `https://api.x.ai/v1/...` |
| **Token lifecycle** | Short-lived access token, refreshed from the stored refresh token on every request | Static API key, no refresh |
| **Best for** | People who already pay for a Grok Build subscription | Pay-as-you-go API access, custom endpoints, gateways |

Rule of thumb: **you have a Grok Build subscription, use `grok`. You have an API key or a custom endpoint, use the standard xAI API-key path.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-grok-login` writes `~/.grok/auth.json` (via the `grok` CLI) and offers a `grok` profile; it does not switch your session. Activate `grok` with `/profile` (or set `PROVIDER=grok` yourself) before SCORPIOX CODE uses it.
- **The credentials file holds a live refresh token.** `~/.grok/auth.json` can renew your account session — do not commit it, do not share it, and do not loosen its permissions.
- **`GROK_TOKEN_SOURCE=local` still requires a login.** The value means "read the token from `~/.grok/auth.json` (or `$GROK_HOME/auth.json`)". If you have never signed in, that file does not exist and every request fails with a token error. Run `scorpiox-grok-login` first.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Grok Build account your standing default; use `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic; re-login is the fallback.** If `scorpiox-grok-refreshtoken --force` keeps failing (expired refresh token, account or plan changed), do a full `scorpiox-grok-login` to re-bind.
- **`XAI_API_KEY` is a fallback, not a primary.** `scorpiox-grok-fetchtoken` will use an `XAI_API_KEY` environment variable only when no `auth.json` entry is present. Do not rely on it for a subscription you expect to draw on — set up the session login.

---

## Related pages

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Claude Code provider](claude-code-provider.md) — the same OAuth session-login pattern against Anthropic.
- [Codex provider](codex-provider.md) — the same OAuth session-login pattern against OpenAI.
- [Copilot provider](copilot-provider.md) — the same OAuth device-auth pattern against GitHub.
- [Anthropic provider](anthropic-provider.md) — the API-key alternative for Anthropic.
