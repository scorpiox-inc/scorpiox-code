# Using Claude Code CLI Subscription in SCORPIOX CODE

Have a Claude subscription (Claude Pro, Max, or a Claude Code seat) but no Anthropic API key? You can use it directly in SCORPIOX CODE. Instead of metering Anthropic API tokens, the **Claude Code provider** signs you in with the same OAuth session login the Claude Code CLI uses and runs requests against the Anthropic endpoint your subscription already pays for.

This page walks through the **OAuth session login**, how the token is stored and kept fresh, how to switch between multiple accounts with profiles, and how the Claude Code provider differs from the standard Anthropic API-key provider.

Source of truth: `sx_provider_claude_code.c`, `scorpiox-claudecode-login.c`, `scorpiox-claudecode-fetchtoken.c`, and `scorpiox-claudecode-refreshtoken.c` at commit `6c70ad6`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Claude / Claude Code subscription** | Pro, Max, or a Claude Code seat you already pay for |
| **You don't have (or don't want) an Anthropic API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The login flow works over SSH: approve in a browser on any device, paste the code back |

If you are instead calling the Anthropic API with an API key (`ANTHROPIC_API_KEY`), use `PROVIDER=anthropic`. The two are different billing models for the same models — see [Claude Code provider vs. the Anthropic API-key provider](#claude-code-provider-vs-the-anthropic-api-key-provider) below.

> **Subscription, not API.** The Claude Code provider authenticates with OAuth and draws on your account's usage allowance (the same 5-hour session and weekly windows the Claude Code CLI enforces). It does **not** accept an `ANTHROPIC_API_KEY` and does **not** bill per token.

---

## Step 1 — Sign in with the OAuth session flow

The default login is the **OAuth authorization-code (PKCE) flow**. It is deliberately SSH- and headless-friendly: you approve from a browser on whatever device is already signed in, then paste the resulting authorization code back into your terminal. No browser is required on the machine you're running from.

```bash
scorpiox-claudecode-login
```

You'll see something like this:

```
scorpiox-claudecode-login v...

Generating PKCE challenge...

Open this URL in your browser to authorize:

  https://claude.com/cai/oauth/authorize?...&code_challenge=...&code_challenge_method=S256

After logging in, paste the authorization code below.
Code: >
```

Do exactly what it says:

1. **Open the printed URL** in a browser — on any device, wherever you're signed into Claude.
2. **Log in** to your Claude account and authorize the request.
3. Come back to your terminal and **paste the authorization code** when prompted (the `#` state fragment is stripped automatically if you paste the full redirect URL).

SCORPIOX CODE then exchanges the code for an access + refresh token and saves them. On success you'll see:

```
Login successful
  Saved to /home/you/.claude/.credentials.json
  Token expires in 86400s
```

A few things worth knowing about this flow:

- **The access token is short-lived by design** — but you never manage that (see [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh)).
- **Never share the authorization code.** It's a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop, phone, or a kiosk and paste the code back.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: /home/you/.claude/.credentials.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-claudecode-login --force
```

### Named accounts

By default the login writes to `~/.claude/.credentials.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-claudecode-login --name personal     # → ~/.claude/accounts/personal.json
scorpiox-claudecode-login --name work         # → ~/.claude/accounts/work.json
```

Each `--name <account>` login saves to `~/.claude/accounts/<account>.json` instead of the default file.

---

## Step 2 — Turn on the Claude Code provider

Signing in saves the token **and** offers to write a ready-to-use profile. When it asks:

```
Create 'claude_code' config profile?
  Will create: /home/you/.claude/scorpiox-env/claude_code.txt
  Contents:
    PROVIDER=claude_code
    CLAUDE_CODE_TOKEN_SOURCE=local
    MODEL=claude-opus-4-6
Create? [Y/n]
```

say yes (or create the file yourself), then activate it in-session:

```
/profile claude_code     # activate and persist
```

Or set the keys yourself in any cascade tier, a named profile, or as OS environment variables. The minimal set for a personal subscription is:

```
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
```

That's it. `MODEL` defaults to `sonnet` if left empty.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `claude_code` to activate this provider. |
| `CLAUDE_CODE_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a Claude subscription you logged into with `scorpiox-claudecode-login`. |
| `CLAUDE_CODE_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.claude/.credentials.json`. Point it at a named account (e.g. `~/.claude/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts full Claude model IDs or short aliases (see [Choosing a model](#choosing-a-model)). Empty resolves to `sonnet`. |
| `CLAUDE_CODE_API_URL` | text | `api.anthropic.com` | Host the provider calls. The `/v1/messages?beta=true` path is appended automatically. |
| `CLAUDE_CODE_PROXY_URL` | text | *(empty)* | Proxy host to route requests through instead of the direct endpoint. Empty uses the direct `api.anthropic.com` endpoint. |
| `CLAUDE_CODE_PROXY_ENABLED` / `CLAUDE_CODE_PROXY_STRICT` | bool / bool | `true` / `false` | Enable proxy routing; when strict is off, SCORPIOX CODE falls back to the direct endpoint if the proxy is down. |
| `CLAUDE_CODE_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CLAUDE_CODE_TOKEN_SOURCE=http`. |
| `CLAUDE_CODE_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. `CLAUDE_CODE_SSH_HOST` and `CLAUDE_CODE_SSH_USER` are required in that mode. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. `TCP_HOST` is required in that mode. |

> **For a personal subscription you only need two keys:** `PROVIDER=claude_code` and `CLAUDE_CODE_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `http` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Claude Code login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.claude/.credentials.json` under a `claudeAiOauth` section (`accessToken`, `refreshToken`, `expiresAt`, and the granted scopes). On every request, `CLAUDE_CODE_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. With `CLAUDE_CODE_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** Before a token is close to expiring (a 5-minute buffer ahead of its `expiresAt`), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401) or forbidden (HTTP 403), the provider refreshes once and retries the request. A second 401 after a refresh is reported as a permanent auth failure.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-claudecode-refreshtoken            # refresh if the token is expired
scorpiox-claudecode-refreshtoken --force    # always refresh
scorpiox-claudecode-refreshtoken --verbose  # show old/new tokens
```

The refresh token (stored in the same `~/.claude/.credentials.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-claudecode-refreshtoken --force   # token expired — renew it
scorpiox-claudecode-login --force          # refresh failed / account changed — re-login
```

You can also inspect the resolved token without making a model request, for diagnostics:

```bash
scorpiox-claudecode-fetchtoken -local      # read from ~/.claude/.credentials.json
scorpiox-claudecode-fetchtoken -config     # read CLAUDE_CODE_TOKEN_SOURCE from scorpiox-env.txt
scorpiox-claudecode-fetchtoken -ssh        # read from a remote machine via SSH
scorpiox-claudecode-fetchtoken -remote     # fetch from the HTTP endpoint
scorpiox-claudecode-fetchtoken -tcp        # fetch over the raw TCP socket
```

The output is a single JSON line: `{"access_token":"...","expires_at":...}`. Use it to confirm the token source resolves to the account you expect before a long run.

---

## Multi-account and profile switching

Many people have more than one Claude account (personal + work, for example). The Claude Code provider supports that in two layers: **multiple stored logins** and **named config profiles**.

### Store more than one login

Each `scorpiox-claudecode-login --name <account>` run writes a separate file under `~/.claude/accounts/`. To pin a specific login to a profile, set `CLAUDE_CODE_CREDENTIALS_FILE` in that profile to the path you want:

```
CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/work.json
```

If `CLAUDE_CODE_CREDENTIALS_FILE` is empty, the provider falls back to `~/.claude/.credentials.json`.

### Switch accounts and models in-session

Switching is just switching profiles — no re-login, no restart:

```
/profile claude_code     # persistent — writes ACTIVE_PROFILE, survives restarts
/use claude_code         # session-only — gone when the session ends
/profile                 # list available profiles and which is active
/profile off             # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CLAUDE_CODE_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

`MODEL` accepts either a short alias or a full Claude model ID. The aliases at this commit resolve to:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-4-6` (provider default `sonnet`) |
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251001` |
| any full `claude-*` ID | passed through as-is |

You can pin a specific version per alias with the override keys (`CLAUDE_MODEL_OPUS_ID_OVERRIDE`, `CLAUDE_MODEL_SONNET_ID_OVERRIDE`, `CLAUDE_MODEL_HAIKU_ID_OVERRIDE`). Set `MODEL` in your profile or switch it at runtime with the `/model` command.

> Note: the login-created `claude_code` profile ships with `MODEL=claude-opus-4-6` out of the box. Change it to `claude-sonnet-4-6` (or the `sonnet` alias) if you want a faster, cheaper default.

You can list every model the endpoint advertises with:

```bash
scorpiox-claudecode-models              # raw JSON list
scorpiox-claudecode-models <model-id>   # single model detail
```

---

## Claude Code provider vs. the Anthropic API-key provider

It's easy to confuse the two because they both run Anthropic models. Here's the difference:

| | **Claude Code provider** (this page) | **Anthropic provider** (`PROVIDER=anthropic`) |
|---|---|---|
| `PROVIDER` value | `claude_code` | `anthropic` |
| **Authentication** | OAuth session login (`scorpiox-claudecode-login`) | `ANTHROPIC_API_KEY` bearer token |
| **Billing** | Your Claude / Claude Code subscription allowance (5h + weekly windows) | Pay-per-token API usage |
| **Token source** | `CLAUDE_CODE_TOKEN_SOURCE` (local file / http / ssh / tcp) | `ANTHROPIC_API_KEY` (+ optional `ANTHROPIC_API_URL`) |
| **Endpoint** | `api.anthropic.com/v1/messages?beta=true` | Any Anthropic endpoint (`ANTHROPIC_API_URL`) |
| **Best for** | People who already pay for Claude / Claude Code | Pay-as-you-go API access, custom endpoints |

Rule of thumb: **you have a subscription → `claude_code`. You have an API key or a custom endpoint → `anthropic`.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-claudecode-login` writes the token file and offers a profile. You still need `PROVIDER=claude_code` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file holds a live refresh token.** `~/.claude/.credentials.json` can renew your account session — don't commit it, don't share it, and don't loosen its permissions.
- **Named accounts live in `~/.claude/accounts/`, not the default file.** `--name work` writes `~/.claude/accounts/work.json`. A profile only uses it if `CLAUDE_CODE_CREDENTIALS_FILE` points there.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Claude Code account your standing default and `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-claudecode-refreshtoken --force` keeps failing (expired refresh token, account changed, plan changed), do a full `scorpiox-claudecode-login --force` to re-bind.
- **Subscription windows are enforced upstream.** The 5-hour and weekly limits are Anthropic's, not SCORPIOX CODE's. You can see the utilization and reset times with `scorpiox-claudecode-usage` instead of per-token metering.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile can't initialize.
