# Using Grok Build Subscription in SCORPIOX CODE

Have an xAI **Grok Build** subscription but no xAI API key? Use it directly in SCORPIOX CODE. The **Grok provider** signs you in with the same OAuth session login the official Grok CLI uses and runs requests against the subscription endpoint your plan already pays for — no per-token API billing.

This page walks through the **OAuth session login**, how the token is stored and kept fresh automatically, how to switch accounts and models with profiles, and how direct Grok Build subscription access differs from the standard xAI API-key path.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Grok Build subscription** | The xAI plan you already pay for and sign into with the Grok CLI |
| **You don't have (or don't want) an xAI API key** | No per-token billing — requests draw on your subscription |
| **You want Grok models through xAI's own backend** | `grok-4.6` and other `grok-*` model IDs run over one Grok Build sign-in |
| **Headless / over SSH / in a container** | The device-auth flow works over SSH: approve in a browser on any device, no localhost redirect needed on the box |

If you are instead calling the xAI API with an API key (`XAI_API_KEY`), use `PROVIDER=openai` with the xAI API base URL. The two are different billing models for the same family of models — see [Grok provider vs. the xAI API-key path](#grok-provider-vs-the-xai-api-key-path) below.

> **Subscription, not API.** The Grok provider authenticates with OAuth session login and draws on your account's subscription allowance. It does **not** accept an `XAI_API_KEY` for the chat endpoint and does **not** bill per token. The endpoint is `cli-chat-proxy.grok.com`, not `api.x.ai`.

---

## Step 1 — Sign in with the Grok CLI

The Grok provider reads its OAuth token from the same file the official Grok CLI writes: **`~/.grok/auth.json`**.

If the Grok CLI is already installed and you have logged in, you can skip straight to [Step 2](#step-2--turn-on-the-grok-provider).

### Install the Grok CLI (if needed)

```bash
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth
```

This opens the OAuth **device-auth** flow:

1. The CLI prints a link and a one-time code.
2. Open the link in a browser, sign in to your xAI account, and enter the code.
3. The CLI polls and saves the session to `~/.grok/auth.json` on success.

The device-auth flow is deliberately headless-friendly: you approve from whatever device already has your account open, so no browser is required on the machine you're running SCORPIOX CODE from.

### Or use the login helper

SCORPIOX CODE ships a login helper that wraps the Grok CLI and offers to create a ready-to-use profile in one shot:

```bash
scorpiox-grok-login
```

It first looks for the official **`grok`** CLI on your `PATH`. When it's present it runs `grok login --device-auth` for you and, on the way out, writes a ready-to-use **`grok`** profile:

```
Created profile: ~/.claude/scorpiox-env/grok.txt
  /profile grok   (or ACTIVE_PROFILE=grok)
```

If the Grok CLI is not on your `PATH`, it tells you how to get the credentials either way:

```
No grok CLI on PATH. Either:
  curl -fsSL https://x.ai/cli/install.sh | bash
  grok login --device-auth
or copy an existing auth file:
  scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

A few things worth knowing about this flow:

- **The code window is short.** If the wait times out, just re-run `scorpiox-grok-login` and a fresh code is issued.
- **Never share the code or the auth file.** `~/.grok/auth.json` holds live OAuth tokens — treat it like a password.
- **Already have a login on another box?** Copy `~/.grok/auth.json` across and no re-login is needed — just make sure the file is present and move to Step 2.

---

## Step 2 — Turn on the Grok provider

Signing in saves the token; you still need to tell SCORPIOX CODE to use it. Set `PROVIDER=grok` and a token source. For a personal subscription that is `GROK_TOKEN_SOURCE=local`, which reads `~/.grok/auth.json`.

The login command creates the profile for you. If you declined (or are on a non-interactive terminal), create it by hand:

```bash
mkdir -p ~/.claude/scorpiox-env
cat > ~/.claude/scorpiox-env/grok.txt <<'EOF'
PROVIDER=grok
MODEL=grok-4.6
GROK_MODEL=grok-4.6
GROK_TOKEN_SOURCE=local
EOF
```

Then activate it, persistently or for the session only (see [Switch accounts in-session](#switch-accounts-in-session)). See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how profiles resolve across tiers.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | — | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local`, `http`, `tcp`, or `ssh`. Use **`local`** for a subscription you logged into with the Grok CLI. |
| `MODEL` / `GROK_MODEL` | text | `grok-4.6` | Which model to run. Accepts Grok model IDs or short aliases (see [Choosing a model](#choosing-a-model)). `GROK_MODEL`, when set, takes precedence over `MODEL`. |
| `GROK_CREDENTIALS_FILE` | path | *(empty)* | Override the path of the local auth file. Defaults to `~/.grok/auth.json`. Point it at a separate file to pin a profile to a specific account. |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=http`. |
| `GROK_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. Shared by all providers that use a TCP token source. |

> **For a personal subscription you only need three keys:** `PROVIDER=grok`, `GROK_TOKEN_SOURCE=local`, and (optionally) `MODEL`. The rest only matter for shared or remote token setups.

### Where the token lives

On login, the Grok CLI writes the OAuth tokens to **`~/.grok/auth.json`**. On every request, SCORPIOX CODE reads from that file via `GROK_TOKEN_SOURCE=local`. No API key is involved, and no key is ever written. The `GROK_HOME` environment variable (not a `scorpiox-env` key) can redirect the auth file to `$GROK_HOME/auth.json` — handy for side-by-side accounts.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design, but you do not manage that. SCORPIOX CODE keeps it fresh in two ways:

- **Proactive refresh.** Before a token gets close to expiring (a five-minute buffer ahead of its expiry), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries the request with the new token.

The refresh reads `~/.grok/auth.json`, calls the xAI OIDC refresh endpoint, and writes the updated token back in place. You normally never touch this, but you can force a refresh any time:

```bash
scorpiox-grok-refreshtoken            # refresh only if expired
scorpiox-grok-refreshtoken --force    # always refresh
```

> **Re-login is the fallback.** If the refresh token has rotated, been revoked, or the account/plan changed so refresh keeps failing, run `grok login --device-auth` (or `scorpiox-grok-login`) to re-bind the session.

### Inspect your subscription usage

Because usage is drawn from your subscription rather than metered per token, check the allowance the way the Grok CLI does:

```bash
scorpiox-grok-usage            # weekly / monthly limit, % used, reset times
scorpiox-grok-usage --json     # raw billing JSON
```

---

## Multiple accounts and profiles

Many people have more than one xAI account (personal + work, for example). The Grok provider supports that with **config profiles**.

### Create one profile per account

If you keep separate auth files (for example under different `GROK_HOME` directories), point each profile at the right one with `GROK_CREDENTIALS_FILE`:

```
# ~/.claude/scorpiox-env/grok-personal.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/auth.json
MODEL=grok-4.6
```

```
# ~/.claude/scorpiox-env/grok-work.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok-work/auth.json
GROK_MODEL=grok-4.6
MODEL=grok-4.6
```

### Switch accounts in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile grok-work       # persistent, writes ACTIVE_PROFILE, survives restarts
/use grok-personal       # session-only, gone when the session ends
/profile                 # open the profile picker (or list if no picker is installed)
/profile off             # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print resolved keys (PROVIDER, GROK_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

`MODEL` (or `GROK_MODEL`) accepts either a full Grok model ID or a short alias. The aliases at this commit are:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `grok-4.6` (provider default) |
| `opus` | `grok-4.6` |
| `sonnet` | `grok-4.6` |
| `haiku` | `grok-4.5` |
| any `grok-*` ID | passed through as-is |

You can pin a specific version by setting `MODEL` to the full ID directly. Set `MODEL` in your profile or switch it at runtime with the `/model` command. List the models your account can access with:

```bash
scorpiox-grok-models
```

> **Image generation is separate.** `scorpiox-grok-imagegen` can generate and edit images through the Grok Imagine API on the same sign-in (or an `XAI_API_KEY`). It is a standalone tool, not part of the chat request path.

---

## Grok provider vs. the xAI API-key path

It is easy to confuse the two because they both run Grok models. Here is the difference:

| | **Grok provider** (this page) | **xAI API-key provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `grok` | `openai` |
| **Authentication** | OAuth session login (Grok CLI device-auth flow) | `XAI_API_KEY` bearer token |
| **Billing** | Your Grok Build subscription allowance | Pay-per-token API usage |
| **Endpoint** | `cli-chat-proxy.grok.com` (subscription backend) | `api.x.ai/v1` (API-key backend) |
| **Token source** | `GROK_TOKEN_SOURCE` (local file / http / tcp / ssh) | `XAI_API_KEY` |
| **Best for** | People who already pay for Grok Build | Pay-as-you-go API access |

Rule of thumb: **you have a Grok Build subscription, use `PROVIDER=grok`. You have an xAI API key, use `PROVIDER=openai` pointed at the xAI API base URL.**

---

## Gotchas

- **The Grok CLI must have been run at least once** on the machine (or `~/.grok/auth.json` must have been copied over) before the provider can work. There is no built-in login in the Grok provider itself — it delegates to the official CLI.
- **`GROK_TOKEN_SOURCE` should be `local` for a personal subscription.** The `http` / `tcp` / `ssh` sources are for shared or remote token setups. Leaving a non-local source on a normal machine means SCORPIOX CODE will look for a remote token endpoint and will not find one.
- **Never share the credential file.** `~/.grok/auth.json` holds live OAuth tokens, including a refresh token that can renew your session. Don't commit it, don't share it, and don't loosen its permissions.
- **The endpoint is not `api.x.ai`.** The Grok provider talks to `cli-chat-proxy.grok.com`. If you see `api.x.ai` in your configuration, you are on the API-key path, not the subscription path.
- **Re-login after a failed refresh.** If the refresh token has rotated or the account changed, run `grok login --device-auth` again to re-establish the session.
- **Model aliases differ from other providers.** `opus` / `sonnet` / `haiku` resolve to Grok model IDs here, not Claude or OpenAI IDs. The default model at this commit is `grok-4.6`.
