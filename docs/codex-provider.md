# Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE

Have an OpenAI ChatGPT or Codex subscription (ChatGPT Plus/Pro or a Codex plan) but no API key? Use it directly in SCORPIOX CODE. The **Codex provider** signs you in with the same OAuth device-code login the Codex CLI uses and runs requests against the Codex endpoint your subscription already pays for. No metering of API tokens, no per-request billing.

This page walks through the **device-code login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles, and how the Codex provider differs from the standard OpenAI API-key provider.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a ChatGPT / Codex subscription** | A ChatGPT or Codex plan you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No pay-per-token billing; requests draw on your subscription |
| **Headless / over SSH / in a container** | The device-code flow works over SSH: approve in a browser on any device, no localhost redirect needed |

If you are instead calling the OpenAI API with an API key (`OPENAI_API_KEY`), use `PROVIDER=openai` and the standard OpenAI provider. The two are different billing models for the same family of models. See [Using the OpenAI Provider](openai-provider.md) for the API-key path.

> **Subscription, not API.** The Codex provider authenticates with OAuth device-code login and uses your account's usage allowance. It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**. It is deliberately SSH- and headless-friendly: SCORPIOX CODE prints a link and a one-time code, you approve from a browser on whatever device is already signed into ChatGPT, and the terminal polls until you've authorized. No localhost redirect and no URL pasting required.

```bash
scorpiox-codex-login
```

You'll see something like this:

```
scorpiox-codex-login v...

Requesting device code...

Follow these steps to sign in with ChatGPT:

1. Open this link in your browser and sign in to your account
   https://auth.openai.com/codex/device

2. Enter this one-time code (expires in 15 minutes)
   ABCD-EFGH

Device codes are a common phishing target. Never share this code.

Waiting for authorization......
```

Do exactly what it says:

1. **Open the printed link** in a browser, on any device, wherever you're signed into ChatGPT.
2. **Enter the one-time code** shown in your terminal and authorize the request.
3. Come back to your terminal — it is **polling every few seconds** and will finish on its own once you approve.

On success you'll see the tokens saved:

```
  Login successful
  Saved to /home/you/.codex/auth.json
```

A few things worth knowing about this flow:

- **The code expires in 15 minutes.** The terminal keeps polling until you approve or the 15-minute window runs out.
- **Never share the one-time code.** It's a credential; anyone who has the code and the link can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop, phone, or a kiosk. The device-code flow is the one that works cleanly over SSH and in containers because there is no localhost redirect to capture.

### Browser fallback

If your Codex server has device-code login disabled (the tool reports it), or you simply prefer the redirect flow, use the browser mode:

```bash
scorpiox-codex-login --browser
```

This opens an authorization URL, listens for the callback on `http://localhost:1455/auth/callback`, or lets you paste the redirect back in — useful when you're sitting at a machine with a real browser.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: /home/you/.codex/auth.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-codex-login --force
```

### Named accounts

Each `--name <account>` login saves to `~/.codex/accounts/<account>.json` instead of the default file:

```bash
scorpiox-codex-login --name personal     # -> ~/.codex/accounts/personal.json
scorpiox-codex-login --name work         # -> ~/.codex/accounts/work.json
```

---

## Step 2 — Turn on the Codex provider

Signing in saves the token; you still need to tell SCORPIOX CODE to use it. Set `PROVIDER=codex` and a token source. For a personal subscription that's `CODEX_TOKEN_SOURCE=local`, which reads the file you just created.

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

Accepting it writes `~/.claude/scorpiox-env/codex.txt` with exactly those keys. If you decline (or are on a non-interactive terminal), create it by hand:

```bash
mkdir -p ~/.claude/scorpiox-env
cat > ~/.claude/scorpiox-env/codex.txt <<'EOF'
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
MODEL=gpt-5.5
EOF
```

Then activate it, persistently or for the session only. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how profiles resolve across tiers.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `codex` | Set to `codex` to activate this provider. |
| `CODEX_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-codex-login`. |
| `CODEX_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.codex/auth.json`. Point it at a named account (e.g. `~/.codex/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | text | `gpt-5.6-terra` | Which model to run. Accepts Codex model IDs or short aliases (see [Choosing a model](#choosing-a-model)). |
| `CODEX_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort forwarded to the request: `low`, `medium`, `high`, `max`, or `off`. Empty means it is not sent. |
| `CODEX_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CODEX_TOKEN_SOURCE=http`. |
| `CODEX_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. |

> **For a personal subscription, you only need three keys:** `PROVIDER=codex`, `CODEX_TOKEN_SOURCE=local`, and (optionally) `MODEL`. The `http` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Codex login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.codex/auth.json`. On every request it reads from that file via `CODEX_TOKEN_SOURCE=local`. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design, but you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** Before a token gets close to expiring, the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401) or rejected, the provider refreshes once and retries the request with the new token.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-codex-refreshtoken            # refresh if the token is expired
scorpiox-codex-refreshtoken --force    # always refresh
scorpiox-codex-refreshtoken --verbose  # show old/new expiry
```

The refresh token (stored in the same `~/.codex/auth.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-codex-refreshtoken --force   # token expired, renew it
scorpiox-codex-login --force          # refresh failed / account changed, re-login
```

> **Remote, SSH, and TCP sources** fetch the latest token from the configured endpoint on every request, so their "refresh" is just reading the freshest value the server holds. Only `local` relies on the on-disk refresh token.

### Inspect your subscription usage

Want to see how much of your allowance is left? The usage helper reads your OAuth token and prints the live windows:

```bash
scorpiox-codex-usage            # human-readable summary
scorpiox-codex-usage --json     # raw JSON
```

---

## Multi-account: profiles and switching

Many people have more than one OpenAI account (personal + work, for example). The Codex provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-codex-login --name personal     # -> ~/.codex/accounts/personal.json
scorpiox-codex-login --name work         # -> ~/.codex/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account. Point `CODEX_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/codex-personal.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/personal.json
MODEL=sonnet
```

```
# ~/.claude/scorpiox-env/codex-work.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/work.json
MODEL=opus
```

### Switch accounts in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile codex-work       # persistent, writes ACTIVE_PROFILE, survives restarts
/use codex-personal       # session-only, gone when the session ends
/profile                 # open the profile picker (or list if no picker is installed)
/profile off             # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print resolved keys (PROVIDER, CODEX_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

`--verbose` shows the resolved `CODEX_TOKEN_SOURCE`, `MODEL`, and `ACTIVE_PROFILE`, so you can confirm you're pointed at the account you expect before a long run.

---

## Choosing a model

`MODEL` accepts either a full Codex model ID or a short alias. The aliases at this commit are:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `gpt-5.6-terra` (provider default) |
| `opus` | `gpt-5.6-terra` |
| `sonnet` | `gpt-5.6-luna` |
| `haiku` | `gpt-5.4-mini` |
| any `gpt-*` or `codex*` ID | passed through as-is |

You can pin a specific version per alias by setting `MODEL` to the full ID directly. Set `MODEL` in your profile or switch it at runtime with the `/model` command.

> Note: the login-offered `codex` profile ships with `MODEL=gpt-5.5` out of the box. Change it to a full ID or leave an alias (`opus` / `sonnet` / `haiku`) depending on the model you want by default.

---

## Codex provider vs. the OpenAI API-key provider

It's easy to confuse the two because they both run OpenAI models. Here's the difference:

| | **Codex provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `codex` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-codex-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your ChatGPT / Codex subscription allowance | Pay-per-token API usage |
| **Token source** | `CODEX_TOKEN_SOURCE` (local file / http / ssh / tcp) | `OPENAI_API_KEY` (+ optional `OPENAI_BASE_URL`) |
| **Endpoint** | The Codex backend your subscription is bound to | Any OpenAI-compatible endpoint (`OPENAI_BASE_URL`) |
| **Best for** | People who already pay for ChatGPT / Codex | Pay-as-you-go API access, custom and self-hosted OpenAI-compatible backends |

Rule of thumb: **you have a ChatGPT / Codex subscription, use `codex`. You have an API key or a self-hosted OpenAI-compatible server, use `openai`.** See [Using the OpenAI Provider](openai-provider.md) for the full API-key reference.

---

## Gotchas

- **`CODEX_TOKEN_SOURCE` should be `local` for a personal subscription.** The `http` / `ssh` / `tcp` sources are for shared or remote token setups. Leaving a non-local source on a normal machine means SCORPIOX CODE will look for a remote token endpoint and won't find one. The login-created profile sets `CODEX_TOKEN_SOURCE=local` for you.
- **The device-code login is the default.** If you're over SSH or in a container, use the plain `scorpiox-codex-login` (device code). Use `--browser` only when you have a real browser and want the redirect/paste flow, or when device-code login is disabled on your server.
- **Never share the one-time code or the credential file.** `~/.codex/auth.json` (and `~/.codex/accounts/*.json`) hold live OAuth tokens. Treat them like a password.
- **Model aliases differ from the Claude Code provider.** `opus` / `sonnet` / `haiku` resolve to Codex model IDs here, not Claude IDs. The default model at this commit is `gpt-5.6-terra`.
- **Re-login after a failed refresh.** If the refresh token has rotated or the account changed, `scorpiox-codex-login --force` re-runs the device-code flow and replaces the stored token.
