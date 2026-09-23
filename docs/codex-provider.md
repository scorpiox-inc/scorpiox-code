# Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE

Have an OpenAI ChatGPT or Codex subscription (Plus, Pro, or a Codex seat) but no OpenAI API key? You can use it directly in SCORPIOX CODE. Instead of metering OpenAI API tokens, the **Codex provider** signs you in with the same OAuth device-code login the official Codex CLI uses and runs requests against the ChatGPT endpoint your subscription already pays for.

This page walks through the **OAuth device-code login**, how the token is stored and kept fresh, how to switch between multiple accounts with profiles, and how the Codex provider differs from the standard OpenAI API-key provider.

Source of truth: `sx_provider_codex.c`, `scorpiox-codex-login.c`, `scorpiox-codex-fetchtoken.c`, and `scorpiox-codex-refreshtoken.c` at commit `6c70ad6`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a ChatGPT / Codex subscription** | Plus or Pro, or a Codex seat you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No per-token API billing — requests draw on your subscription |
| **Headless / over SSH** | The device-code flow works over SSH: open a link and type a code on any device, no browser needed on the box |

If you are instead calling an OpenAI-compatible server with an API key (`OPENAI_API_KEY` and `OPENAI_BASE_URL`), use `PROVIDER=openai`. The two are different billing models for the same models — see [Codex provider vs. the OpenAI API-key provider](#codex-provider-vs-the-openai-api-key-provider) below.

> **Subscription, not API.** The Codex provider authenticates with OAuth and draws on your account's usage allowance (the 5-hour and weekly windows the Codex backend enforces). It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**. It is deliberately headless- and SSH-friendly: SCORPIOX CODE prints a link and a one-time code, you sign in from a browser on whatever device is already signed in, then SCORPIOX CODE polls in the background and saves the tokens when you're authorized. No redirect server and no pasting.

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

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** in a browser — on any device, wherever you're signed into ChatGPT / Codex.
2. **Sign in** to your account and **enter the one-time code**.
3. Come back to your terminal and wait — SCORPIOX CODE polls automatically (every few seconds, for up to 15 minutes) and exchanges the result for an access + refresh token. On success it saves them to `~/.codex/auth.json`.

A few things worth knowing about this flow:

- **The code expires in 15 minutes.** If the wait times out, just re-run `scorpiox-codex-login` and a fresh code is issued.
- **Never share the code.** It's a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up.

### Browser / redirect login (alternate)

If you'd rather use the redirect-and-paste flow (the same one the official Codex CLI uses), pass `--browser`. This listens on `http://localhost:1455/auth/callback` and captures the code automatically when the browser redirects, or lets you paste the full redirect URL back into the terminal.

```bash
scorpiox-codex-login --browser
```

Use the default device-code flow unless you have a reason to prefer the redirect.

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

By default the login writes to `~/.codex/auth.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-codex-login --name work      # writes ~/.codex/accounts/work.json
```

Named accounts live in `~/.codex/accounts/<name>.json`. A profile only uses one of these if `CODEX_CREDENTIALS_FILE` points at it (see [Configuration keys](#configuration-keys)).

---

## Step 2 — Turn on the Codex provider

On success, SCORPIOX CODE offers to write a ready-to-use profile:

```
Create 'codex' config profile?
  Will create: /home/you/.claude/scorpiox-env/codex.txt
  Contents:
    PROVIDER=codex
    CODEX_TOKEN_SOURCE=local
    MODEL=gpt-5.5
Create? [Y/n]
```

Say yes (or create the file yourself), then activate it in-session:

```
/profile codex        # activate and persist
```

Or set the keys yourself in any cascade tier, a named profile, or as OS environment variables. The minimal set for a personal subscription is:

```
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
```

That's it. `MODEL` defaults to `gpt-5.6-terra` if left empty.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `codex` to activate this provider. |
| `CODEX_TOKEN_SOURCE` | choice | `config` | Where to get the OAuth token: `config` (default — resolved from this same config), `local`, `remote`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-codex-login`. |
| `CODEX_CREDENTIALS_FILE` | text | *(empty)* | Override the local credentials path. Defaults to `~/.codex/auth.json`. Point it at a named account (e.g. `~/.codex/accounts/work.json`) to pin a profile to a specific login. |
| `CODEX_REMOTE_URL` | text | *(built-in endpoint)* | Token endpoint, used only when `CODEX_TOKEN_SOURCE=remote`. |
| `CODEX_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. `CODEX_SSH_HOST` and `CODEX_SSH_USER` are required in that mode. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. `TCP_HOST` is required in that mode. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts short aliases or full Codex model IDs (see [Choosing a model](#choosing-a-model)). Empty resolves to the built-in default. |
| `CODEX_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, `high`, `max`, or `off`. Falls back to `REASONING_EFFORT`, then `OPENAI_REASONING_EFFORT`. When `THINKING=1` is set and no explicit effort is given, it defaults to `high`. |
| `THINKING` | bool | `0` | When set to `1`, defaults reasoning effort to `high` if no explicit effort is set. |

> **For a personal subscription you only need two keys:** `PROVIDER=codex` and `CODEX_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `remote` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Codex login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.codex/auth.json`, in the same format the official Codex CLI uses — a `tokens` section holding the access token, refresh token, id token, and account id, plus a `last_refresh` timestamp. On every request, `CODEX_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the device-code login is short-lived by design — but you don't manage that. With `CODEX_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** When a token is within a 5-minute buffer of its expiry, the provider refreshes and reloads it before sending, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries the request. A second 401 after a refresh is reported as a permanent auth failure.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-codex-refreshtoken            # refresh if the token is expired
scorpiox-codex-refreshtoken --force    # always refresh
scorpiox-codex-refreshtoken --verbose  # show old/new tokens
```

The refresh token (stored in the same `~/.codex/auth.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
# 1. Refresh the token in place
scorpiox-codex-refreshtoken --force

# 2. If that keeps failing, re-bind with a full login
scorpiox-codex-login --force
```

### Inspecting usage instead of per-token metering

Because you're on a subscription, there's no per-token metering. Use `scorpiox-codex-usage` to see how much of each allowance window you've consumed and when it resets:

```bash
scorpiox-codex-usage        # human-readable summary (5h session + weekly windows, with reset times)
scorpiox-codex-usage --json # raw JSON to stdout
```

---

## Multi-account and profile switching

### Store more than one login

Log in to several ChatGPT / Codex accounts and give each a name:

```bash
scorpiox-codex-login --name work
scorpiox-codex-login --name personal
```

Each writes its own file under `~/.codex/accounts/`. Create one profile per account, each pointing `CODEX_CREDENTIALS_FILE` at the matching file.

### Switch accounts and models in-session

Switching is just switching profiles — no re-login, no restart:

```
/profile codex        # persistent — writes ACTIVE_PROFILE, survives restarts
/use codex            # session-only — gone when the session ends
/profile              # list available profiles and which is active
/profile off          # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CODEX_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

`MODEL` accepts either a short alias or a full Codex model ID. The aliases at this commit resolve to:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `gpt-5.6-terra` (built-in default) |
| `opus` | `gpt-5.6-terra` |
| `sonnet` | `gpt-5.6-luna` |
| `haiku` | `gpt-5.4-mini` |
| any full `gpt-*` or `codex*` ID | passed through as-is |

Set `MODEL` in your profile or switch it at runtime with the `/model` command.

> Note: the login-created `codex` profile ships with `MODEL=gpt-5.5` out of the box. Change it to the alias you prefer (e.g. `sonnet`, `opus`, or `haiku`) if you want a different default.

You can list every model the endpoint advertises with:

```bash
scorpiox-codex-models              # raw JSON list
```

---

## Codex provider vs. the OpenAI API-key provider

It's easy to confuse the two because they both run OpenAI models. Here's the difference:

| | **Codex provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `codex` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-codex-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your ChatGPT / Codex subscription allowance (5h + weekly windows) | Pay-per-token API usage (or your own server) |
| **Token source** | `CODEX_TOKEN_SOURCE` (local / remote / ssh / tcp) | `OPENAI_API_KEY` (+ optional `OPENAI_BASE_URL`) |
| **Endpoint** | `chatgpt.com/backend-api/codex/responses` | Any OpenAI-compatible `/v1/chat/completions` server |
| **Best for** | People who already pay for ChatGPT / Codex | Pay-as-you-go API access, local or custom endpoints |

Rule of thumb: **you have a ChatGPT / Codex subscription → `codex`. You have an API key or a self-hosted OpenAI-compatible server → `openai`.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-codex-login` writes the token file and offers a profile. You still need `PROVIDER=codex` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file holds a live refresh token.** `~/.codex/auth.json` can renew your account session — don't commit it, don't share it, and don't loosen its permissions. It's written `0600` by default.
- **Named accounts live in `~/.codex/accounts/`, not the default file.** `--name work` writes `~/.codex/accounts/work.json`. A profile only uses it if `CODEX_CREDENTIALS_FILE` points there.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Codex account your standing default and `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-codex-refreshtoken --force` keeps failing (expired refresh token, account changed, plan changed), do a full `scorpiox-codex-login --force` to re-bind.
