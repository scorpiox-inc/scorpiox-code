# Using Grok Build Subscription in SCORPIOX CODE

Have an xAI Grok Build subscription (SuperGrok / Grok for Work) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering xAI API tokens, the **Grok provider** signs you in with your normal Grok account and runs requests against the same backend your subscription already pays for.

This page walks through the **OAuth session login**, how the token is stored and kept fresh, how to switch models and accounts with profiles, and how the Grok provider differs from the standard xAI API-key usage.

Source of truth: `scorpiox-grok-login.c`, `scorpiox-grok-fetchtoken.c`, `scorpiox-grok-refreshtoken.c`, `scorpiox-grok-usage.c`, and `sx_provider_grok.c` at commit `5fd054b`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Grok Build / SuperGrok subscription** | A plan you already pay for, with weekly or monthly credit limits |
| **You don't have (or don't want) an xAI API key** | No pay-per-token billing — requests draw on your subscription credits |
| **Headless / over SSH** | The Grok CLI device-auth flow needs no browser on the same machine |

If you are instead calling xAI with an API key (`XAI_API_KEY` against `api.x.ai`), you don't need this provider at all — set the key and SCORPIOX CODE falls back to it. See [Grok provider vs. xAI API-key usage](#grok-provider-vs-xai-api-key-usage) at the end of this page.

> **Subscription, not API.** The Grok provider authenticates with an OAuth session and uses your account's credit allowance. It does **not** require an `XAI_API_KEY` and does **not** bill per token. The two are different billing models for the same models.

---

## Step 1 — Sign in with the Grok CLI

The Grok provider reuses the credentials that the official xAI **Grok CLI** stores. There are two ways to get a valid session file:

### Option A — install the Grok CLI (recommended)

SCORPIOX CODE ships a login helper that runs the official device-auth flow for you when the `grok` binary is available:

```bash
scorpiox-grok-login
```

If the `grok` CLI is on your `PATH`, this launches `grok login --device-auth`. You'll be shown a URL and a one-time code; open the URL on *any* device you're signed into Grok with, enter the code, and the token is saved automatically to `~/.grok/auth.json`. This is SSH- and headless-friendly — no browser is needed on the machine you're running from.

If the `grok` binary is not installed, the helper prints the install command instead:

```bash
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth
```

### Option B — copy an existing session file

If you already signed in with the Grok CLI on another machine (or on this one), you can just bring the session file over:

```bash
scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

SCORPIOX CODE reads that file directly — it does not create it. Either way, once `~/.grok/auth.json` exists, the provider is ready to use it.

> The `grok` CLI is the source of the session file. SCORPIOX CODE never writes your Grok credentials itself; it reads what the CLI saved.

---

## Step 2 — Turn on the Grok provider

Signing in gives you a token; you still need to tell SCORPIOX CODE to use it. Running `scorpiox-grok-login` already writes a ready-to-use **profile** for you:

```
~/.claude/scorpiox-env/grok.txt
```

with exactly these keys:

```
PROVIDER=grok
MODEL=grok-4.6
GROK_MODEL=grok-4.6
GROK_TOKEN_SOURCE=local
```

You can edit that file by hand, or recreate it:

```bash
mkdir -p ~/.claude/scorpiox-env
cat > ~/.claude/scorpiox-env/grok.txt <<'EOF'
PROVIDER=grok
GROK_TOKEN_SOURCE=local
MODEL=grok-4.6
EOF
```

Then activate it — persistently or for the session only. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how profiles resolve across tiers.

> **For a personal subscription you only need three keys:** `PROVIDER=grok`, `GROK_TOKEN_SOURCE=local`, and (optionally) `MODEL`. The `ssh` / `http` / `tcp` token sources exist for shared or remote token setups and are not needed for a normal Grok login.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local`, `http` (alias `remote`), `ssh`, `tcp`, or `config`. Use **`local`** for a Grok account you logged into with the Grok CLI. |
| `GROK_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local session file. Defaults to `~/.grok/auth.json` (or `$GROK_HOME/auth.json`). Point it at a different file to pin a profile to a specific login. |
| `MODEL` / `GROK_MODEL` | text | `grok-4.6` | Which model to run. Accepts full `grok-*` model IDs or short names (see [Choosing a model](#choosing-a-model)). |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=http`. |
| `GROK_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=ssh` — fetch the session file from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. |

### Where the token lives

The Grok CLI writes the OAuth tokens to `~/.grok/auth.json` (owner-read-only). On every request, the provider reads from that file when `GROK_TOKEN_SOURCE=local`. No API key is involved, and no key is ever written. You can relocate the session file with `GROK_HOME` or `GROK_CREDENTIALS_FILE` if you keep several logins side by side.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** Before a token is close to expiring (a 5-minute buffer ahead of its `expires_at`), the provider refreshes the session and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized, the provider refreshes once and retries.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-grok-refreshtoken            # refresh if the token is expired
scorpiox-grok-refreshtoken --force    # always refresh
scorpiox-grok-refreshtoken --verbose  # show extra diagnostics
```

The refresh token (stored in the same `~/.grok/auth.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-grok-refreshtoken --force    # token expired — renew it
grok login --device-auth              # refresh failed / account changed — re-login
```

---

## Checking your subscription usage

Because a Grok Build subscription is a credit allowance rather than an open-ended API, you can check how much of it you've used. The usage tool hits the same billing backend your subscription reports to:

```bash
scorpiox-grok-usage          # pretty: "Weekly limit NN% used  Resets in ..."
scorpiox-grok-usage --json   # raw billing JSON
```

It reports the current weekly or monthly limit, the percentage used, and when the period resets. Use it before a long batch run if you want to know where your allowance stands.

To list the models your account can see, use:

```bash
scorpiox-grok-models
```

---

## Profiles, model selection, and switching

### Choosing a model

`MODEL` accepts either a full `grok-*` model ID or a short alias. The defaults and aliases at this commit are:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `grok-4.6` (default) |
| `opus` | `grok-4.6` |
| `sonnet` | `grok-4.6` |
| `haiku` | `grok-4.5` |
| any `grok-*` ID | passed through as-is |

Set `MODEL` in your profile or switch it at runtime with the `/model` command.

### Multiple accounts with profiles

If you have more than one Grok account (personal + work, for example), give each its own session file and its own profile. Point `GROK_CREDENTIALS_FILE` at the right file so the profile always uses that account:

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
MODEL=grok-4.5
```

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile grok-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use grok-personal      # session-only — gone when the session ends
/profile               # list available profiles and which is active
/profile off           # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, GROK_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

---

## Grok provider vs. xAI API-key usage

It's easy to confuse the two because they both reach xAI models. Here's the difference:

| | **Grok provider** (this page) | **xAI API key** |
|---|---|---|
| **Authentication** | OAuth session from the Grok CLI (`~/.grok/auth.json`) | `XAI_API_KEY` bearer token |
| **Billing** | Your Grok Build / SuperGrok subscription credits | Pay-per-token API usage |
| **Endpoint** | `https://cli-chat-proxy.grok.com/v1/responses` | `api.x.ai` |
| **Token source** | `GROK_TOKEN_SOURCE` (local file / http / ssh / tcp) | A single environment variable |
| **Best for** | People who already pay for a Grok subscription | Self-hosted servers, scripts, raw xAI API |

Rule of thumb: **you have a Grok subscription → `PROVIDER=grok`. You have an xAI API key → set `XAI_API_KEY`.** If there's no session file present, the provider transparently falls back to `XAI_API_KEY` as a last resort — so an API key still works even in `local` mode.

---

## Gotchas

- **The session file belongs to the Grok CLI, not SCORPIOX CODE.** `scorpiox-grok-login` either runs `grok login --device-auth` for you or tells you to copy `~/.grok/auth.json`. SCORPIOX CODE never writes the file itself. If it's missing, run the Grok CLI login or copy it over.
- **`GROK_TOKEN_SOURCE` defaults to `local`.** That's the right value for a personal subscription. The `ssh` / `http` / `tcp` sources exist for shared or remote token setups — leave them off unless you actually have a remote token master.
- **The login helper and the provider are separate.** `scorpiox-grok-login` only ensures a session file exists and writes the `grok` profile. You still need `PROVIDER=grok` active (via a profile or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **`auth.json` is owner-read-only.** Don't loosen the permissions; it holds a live refresh token that renews your account.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-grok-refreshtoken --force` keeps failing (expired refresh token, account changed, plan changed), do a full `grok login --device-auth` to re-bind.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile can't initialize.
