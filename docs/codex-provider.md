# Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE

Have a ChatGPT or Codex subscription (Plus, Pro, Team, or Enterprise) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering OpenAI API tokens, the **Codex provider** signs you in with your normal ChatGPT/Codex login and runs requests against the same backend your subscription already pays for.

This page walks through the **OAuth device-code login**, how the token is stored and kept fresh, how to juggle multiple accounts, and how the Codex provider differs from the standard [OpenAI API-key provider](openai-provider.md).

Source of truth: `scorpiox-codex-login.c`, `scorpiox-codex-refreshtoken.c`, `sx_provider_codex.c`, and `scorpiox-config.c` at commit `5fd054b`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a ChatGPT / Codex subscription** | Plus, Pro, Team, or Enterprise plan you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The default login flow needs no browser on the same machine |

If you are instead calling an OpenAI-compatible API with an API key (`OPENAI_API_KEY` + `OPENAI_BASE_URL`), use `PROVIDER=openai` — see [Using the OpenAI Provider](openai-provider.md).

> **Subscription, not API.** The Codex provider authenticates with OAuth and uses your account's usage allowance. It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token. The two are different billing models for the same models.

---

## Step 1 — Sign in with the device-code flow

The default login is the **device-code flow**, which is deliberately SSH- and headless-friendly: no browser is required on the machine you're running from, and nothing is pasted back into the terminal.

```bash
scorpiox-codex-login
```

You'll see something like this:

```
Requesting device code...

Follow these steps to sign in with ChatGPT:

1. Open this link in your browser and sign in to your account
   https://auth.openai.com/codex/device

2. Enter this one-time code (expires in 15 minutes)
   ABCD-EFGH

Device codes are a common phishing target. Never share this code.

Waiting for authorization...
```

Do exactly what it says:

1. **Open the URL** on *any* device — your laptop, your phone, a kiosk, wherever you're signed into ChatGPT.
2. **Enter the one-time code** shown in your terminal.
3. Come back to your terminal. Once you've authorized, SCORPIOX CODE polls, gets the token, and saves it automatically.

A few things worth knowing about this flow:

- **The code expires in 15 minutes.** If you don't finish in time, the command times out — just run `scorpiox-codex-login` again.
- **Never share the code.** It's a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** That's the whole point of device-code login. You approve from wherever you're already signed in.

### Browser (redirect/paste) login

If you prefer a browser on the same machine, you can use the redirect flow instead:

```bash
scorpiox-codex-login --browser
```

This opens an authorize URL and captures the callback on `http://localhost:1455/auth/callback` (or lets you paste the redirect back in if you're on a machine without a local browser). Use the default device-code flow unless you have a specific reason to use this one.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: ~/.codex/auth.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-codex-login --force
```

---

## Step 2 — Turn on the Codex provider

Signing in saves the token; you still need to tell SCORPIOX CODE to use it. Set `PROVIDER=codex` and a token source. For a local subscription that's `CODEX_TOKEN_SOURCE=local`, which reads the file you just created.

The login command offers to create a ready-to-use **`codex`** profile for you at the end:

```
Create 'codex' config profile?
  Will create: ~/.claude/scorpiox-env/codex.txt
  Contents:
    PROVIDER=codex
    CODEX_TOKEN_SOURCE=local
    MODEL=gpt-5.5

Create? [Y/n]
```

Accepting it writes `~/.claude/scorpiox-env/codex.txt` with exactly those keys. If you decline, create it by hand:

```bash
mkdir -p ~/.claude/scorpiox-env
cat > ~/.claude/scorpiox-env/codex.txt <<'EOF'
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
MODEL=gpt-5.5
EOF
```

Then activate it — persistently or for the session only. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how profiles resolve across tiers.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `codex` to activate this provider. |
| `CODEX_TOKEN_SOURCE` | choice | `tcp` | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a ChatGPT/Codex subscription you logged into with `scorpiox-codex-login`. |
| `CODEX_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.codex/auth.json`. Point it at a named account (e.g. `~/.codex/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | text | *(empty)* | Which model to run. Empty defaults to `gpt-5.6-terra`. Accepts full Codex model IDs or short names (see [Choosing a model](#choosing-a-model)). |
| `CODEX_REMOTE_URL` | text | `https://token.scorpiox.net/codex` | Token endpoint, used only when `CODEX_TOKEN_SOURCE=http`. |
| `CODEX_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. |

> **For a personal subscription, you only need three keys:** `PROVIDER=codex`, `CODEX_TOKEN_SOURCE=local`, and (optionally) `MODEL`. The `ssh` / `http` / `tcp` sources exist for shared or remote token setups and are not needed for a normal ChatGPT/Codex login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.codex/auth.json` (mode `0600`, owner-read-only). On every request it reads from that file via `CODEX_TOKEN_SOURCE=local`. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the device-code flow is short-lived by design — but you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** Before a token is close to expiring (a 5-minute buffer ahead of its `expires_at`), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-codex-refreshtoken            # refresh if the token is expired
scorpiox-codex-refreshtoken --force    # always refresh
scorpiox-codex-refreshtoken --verbose  # show old/new tokens
```

The refresh token (stored in the same `auth.json`) is what lets the access token be renewed without signing in again. As long as `~/.codex/auth.json` is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-codex-refreshtoken --force   # token expired — renew it
scorpiox-codex-login --force          # refresh failed / account changed — re-login
```

---

## Multi-account: profiles and `scorpiox-config`

Many people have more than one ChatGPT account (personal + work, for example). The Codex provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

By default, `scorpiox-codex-login` writes to `~/.codex/auth.json`. To keep several accounts side by side, give each a name:

```bash
scorpiox-codex-login --name personal     # → ~/.codex/accounts/personal.json
scorpiox-codex-login --name work         # → ~/.codex/accounts/work.json
```

Each `--name <account>` login saves to `~/.codex/accounts/<account>.json` instead of the default `auth.json`.

### Bind each login to a profile

Now make one profile per account. Point `CODEX_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/codex-personal.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/personal.json
MODEL=gpt-5.5
```

```
# ~/.claude/scorpiox-env/codex-work.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/work.json
MODEL=gpt-5.6-terra
```

### Switch accounts in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile codex-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use codex-personal      # session-only — gone when the session ends
/profile                # list available profiles and which is active
/profile off            # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CODEX_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `CODEX_TOKEN_SOURCE`, `CODEX_REMOTE_URL`, `MODEL`, and `ACTIVE_PROFILE`, so you can confirm you're pointed at the account you expect before a long run.

---

## Choosing a model

`MODEL` accepts either a full Codex model ID or a short alias. The defaults and aliases at this commit are:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `gpt-5.6-terra` (default) |
| `opus` | `gpt-5.6-terra` |
| `sonnet` | `gpt-5.6-luna` |
| `haiku` | `gpt-5.4-mini` |
| `gpt-5.5` | `gpt-5.5` (passed through) |
| any `gpt-*` / `*codex*` ID | passed through as-is |

Set `MODEL` in your profile or switch it at runtime with the `/model` command.

---

## Codex provider vs. the OpenAI API-key provider

It's easy to confuse the two because they both talk to OpenAI models. Here's the difference:

| | **Codex provider** (this page) | **OpenAI provider** ([openai-provider.md](openai-provider.md)) |
|---|---|---|
| `PROVIDER` value | `codex` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-codex-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your ChatGPT / Codex subscription allowance | Pay-per-token API usage |
| **Token source** | `CODEX_TOKEN_SOURCE` (local file / http / ssh / tcp) | `OPENAI_BASE_URL` + `OPENAI_API_KEY` |
| **Endpoint** | Codex backend (`/backend-api/codex/responses`) | Any OpenAI-compatible `/v1/chat/completions` |
| **Best for** | People who already pay for ChatGPT/Codex | Self-hosted servers, Azure, Together, Groq, raw OpenAI API |

Rule of thumb: **you have a subscription → `codex`. You have an API key or a self-hosted endpoint → `openai`.**

---

## Gotchas

- **`CODEX_TOKEN_SOURCE` defaults to `tcp`, not `local`.** For a personal subscription you must set `CODEX_TOKEN_SOURCE=local` (the login-created profile does this for you). Leaving it at the default means SCORPIOX CODE will look for a TCP token source and won't find one.
- **The login command and the provider are separate.** `scorpiox-codex-login` only writes the token file. You still need `PROVIDER=codex` active (via a profile or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The device code expires in 15 minutes.** Run the login again if you time out. Never share the code — it's a credential.
- **`auth.json` is owner-read-only (`0600`).** Don't loosen the permissions; it holds a live refresh token that renews your account.
- **Named accounts live in `~/.codex/accounts/`, not `auth.json`.** `--name work` writes `~/.codex/accounts/work.json`. A profile only uses it if `CODEX_CREDENTIALS_FILE` points there.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-codex-refreshtoken --force` keeps failing (expired refresh token, account changed, plan changed), do a full `scorpiox-codex-login --force` to re-bind.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile can't initialize.
