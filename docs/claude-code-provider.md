# Using Claude Code CLI Subscription in SCORPIOX CODE

Have a Claude subscription (Claude Pro, Max, or a Claude Code seat) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering Anthropic API tokens, the **Claude Code provider** signs you in with the same OAuth session login the Claude Code CLI uses and runs requests against the Anthropic endpoint your subscription already pays for.

This page walks through the **OAuth session login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles, and how the Claude Code provider differs from the standard Anthropic API-key provider.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Claude / Claude Code subscription** | Pro, Max, or a Claude Code seat you already pay for |
| **You don't have (or don't want) an Anthropic API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The login flow works over SSH: you approve in a browser on any device and paste the code back |

If you are instead calling the Anthropic API with an API key (`ANTHROPIC_API_KEY`), use `PROVIDER=anthropic`. The two are different billing models for the same models.

> **Subscription, not API.** The Claude Code provider authenticates with OAuth and uses your account's usage allowance (the same 5-hour session and weekly windows the Claude Code CLI enforces). It does **not** accept an `ANTHROPIC_API_KEY` and does **not** bill per token.

---

## Step 1 — Sign in with the OAuth session flow

The default login is the **OAuth authorization-code (PKCE) flow**. It is deliberately SSH- and headless-friendly: you approve from a browser on whatever device is already signed in, then paste the resulting authorization code back into your terminal. Nothing long-lived is pasted, and no browser is required on the machine you're running from.

```bash
scorpiox-claudecode-login
```

You'll see something like this:

```
scorpiox-claudecode-login v...

Open this URL in your browser to authorize:

  https://claude.com/cai/oauth/authorize?code=true&response_type=code&...

After logging in, paste the authorization code below.
Code: >
```

Do exactly what it says:

1. **Open the printed URL** in a browser — on any device, wherever you're signed into Claude.
2. **Log in** to your Claude account and authorize the request.
3. Come back to your terminal and **paste the authorization code** when prompted (the `#` state fragment is stripped automatically if you paste the full redirect fragment).

SCORPIOX CODE then exchanges the code for tokens and saves them. On success you'll see:

```
Login successful
  Saved to /home/you/.claude/.credentials.json
  Token expires in 86400s
```

A few things worth knowing about this flow:

- **The access token is short-lived by design** — but you never manage that (see [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh)).
- **Never share the authorization code.** It is a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop, phone, or a kiosk and paste the code back.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: /home/you/.claude/.credentials.json
```

Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-claudecode-login --force
```

### Named accounts

By default the login writes to `~/.claude/.credentials.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-claudecode-login --name personal     # -> ~/.claude/accounts/personal.json
scorpiox-claudecode-login --name work         # -> ~/.claude/accounts/work.json
```

Each `--name <account>` login saves to `~/.claude/accounts/<account>.json` instead of the default file.

---

## Step 2 — Turn on the Claude Code provider

Signing in saves the token; you still need to tell SCORPIOX CODE to use it. Set `PROVIDER=claude_code` and a token source. For a personal subscription that's `CLAUDE_CODE_TOKEN_SOURCE=local`, which reads the file you just created.

The login command offers to create a ready-to-use **`claude_code`** profile for you at the end:

```
Create 'claude_code' config profile?
  Will create: ~/.claude/scorpiox-env/claude_code.txt
  Contents:
    PROVIDER=claude_code
    CLAUDE_CODE_TOKEN_SOURCE=local
    MODEL=claude-opus-4-6

Create? [Y/n]
```

Accepting it writes `~/.claude/scorpiox-env/claude_code.txt` with exactly those keys. If you decline, create it by hand:

```bash
mkdir -p ~/.claude/scorpiox-env
cat > ~/.claude/scorpiox-env/claude_code.txt <<'EOF'
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
MODEL=claude-opus-4-6
EOF
```

Then activate it — persistently or for the session only. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how profiles resolve across tiers.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | — | Set to `claude_code` to activate this provider. |
| `CLAUDE_CODE_TOKEN_SOURCE` | choice | `tcp` | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a Claude subscription you logged into with `scorpiox-claudecode-login`. |
| `CLAUDE_CODE_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.claude/.credentials.json`. Point it at a named account (e.g. `~/.claude/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | choice | — | Which model to run. Accepts the short aliases `opus` / `sonnet` / `haiku` or a full Claude model ID (see [Choosing a model](#choosing-a-model)). |
| `CLAUDE_CODE_API_URL` | text | `api.anthropic.com` | Host the provider calls. The `/v1/messages?beta=true` path is appended automatically. |
| `CLAUDE_CODE_PROXY_URL` / `CLAUDE_CODE_PROXY_ENABLED` / `CLAUDE_CODE_PROXY_STRICT` | text / bool / bool | *(empty)* / `0` / `0` | Route requests through a proxy instead of the direct endpoint. When the proxy is down, non-strict mode falls back to the direct endpoint. |
| `CLAUDE_CODE_CLI_VERSION` | text | `2.1.280` | CLI version sent in the User-Agent. Bump it if the endpoint starts requiring a newer CLI — no rebuild needed. |
| `CLAUDE_CODE_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CLAUDE_CODE_TOKEN_SOURCE=http`. |
| `CLAUDE_CODE_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. Shared by all providers that use a TCP token source. |

### Where the token lives

For `CLAUDE_CODE_TOKEN_SOURCE=local`, the provider reads OAuth tokens from `~/.claude/.credentials.json` (or the path in `CLAUDE_CODE_CREDENTIALS_FILE`). That is the file `scorpiox-claudecode-login` creates. The other token sources (`http`, `ssh`, `tcp`) are for sharing a login across machines and are covered by their respective keys above.

---

## Token persistence and automatic refresh

The access token SCORPIOX CODE uses is short-lived, so it cannot be the thing you log in with. The credentials file stores both an **access token** and a **refresh token**. On each request, SCORPIOX CODE checks the access token's expiry and, if it is close (refreshed a few minutes before it actually lapses), it runs the refresh step first and then re-reads the updated token from the credentials file.

You normally never touch this. The refresh binary is `scorpiox-claudecode-refreshtoken`, and it can also be run on its own:

```bash
scorpiox-claudecode-refreshtoken          # refresh only if expired
scorpiox-claudecode-refreshtoken --force  # always refresh
```

Before rewriting the credentials file, the refresher makes a `.backup` copy, so a bad refresh does not destroy a working token.

> **Re-login is the fallback.** If refresh keeps failing (an expired or revoked refresh token, an account change, a plan change), do a full `scorpiox-claudecode-login --force` to re-bind your session.

### Inspect your subscription usage

Because usage is drawn from your subscription rather than metered per token, check the allowance the way the Claude Code CLI does:

```bash
scorpiox-claudecode-usage
```

This reports the 5-hour and weekly window utilization and reset times, so you can see how much of your allowance remains.

---

## Multi-account: profiles and switching

### Save more than one login

Log in to each account under a name (see [Named accounts](#named-accounts)), then give each one its own profile that pins the matching credentials file.

### Bind each login to a profile

```bash
# ~/.claude/scorpiox-env/claude-personal.txt
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/personal.json
MODEL=claude-sonnet-4-6

# ~/.claude/scorpiox-env/claude-work.txt
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/work.json
MODEL=claude-sonnet-4-6
```

### Switch accounts in-session

- **`/profile <name>`** — makes a profile your standing default (persisted) and hot-swaps the provider.
- **`/use <name>`** — hops to a profile for this session only (no file is written).
- **`/profile`** or **`/use`** with no argument — lists the available profiles and marks the active one.
- **`/profile off`** / **`/use off`** — clear the active profile.

```
/profile            # list profiles
/profile claude-work
/use claude-personal
```

Both `/profile` and `/use` trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

You can always print a key's effective value and see which tier it came from:

```bash
scorpiox-config --get PROVIDER
scorpiox-config --get CLAUDE_CODE_TOKEN_SOURCE
scorpiox-config --get CLAUDE_CODE_CREDENTIALS_FILE
```

---

## Choosing a model

`MODEL` accepts either a short alias or a full Claude model ID. The aliases at this commit resolve to:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `sonnet` (provider default) |
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251001` |
| any full `claude-*` ID | passed through as-is |

You can pin a specific version per alias with the override keys (`CLAUDE_MODEL_OPUS_ID_OVERRIDE`, `CLAUDE_MODEL_SONNET_ID_OVERRIDE`, `CLAUDE_MODEL_HAIKU_ID_OVERRIDE`). Set `MODEL` in your profile or switch it at runtime with the `/model` command.

> Note: the login-offered `claude_code` profile ships with `MODEL=claude-opus-4-6` out of the box. Change it to `claude-sonnet-4-6` or leave the alias as `opus` / `sonnet` / `haiku` depending on the model you want by default.

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

See [Anthropic Provider](anthropic-provider.md) for the API-key provider's full key set.

---

## Gotchas

- **`CLAUDE_CODE_TOKEN_SOURCE` defaults to `tcp`, not `local`.** For a personal subscription you must set `CLAUDE_CODE_TOKEN_SOURCE=local` (the login-created profile does this for you). Leaving it at the default means SCORPIOX CODE will look for a TCP token source and won't find one on a normal machine.
- **The login command and the provider are separate.** `scorpiox-claudecode-login` only writes the token file. You still need `PROVIDER=claude_code` active (via a profile or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file holds a live refresh token.** `~/.claude/.credentials.json` can renew your account session — don't commit it, don't share it, and don't loosen its permissions.
- **Named accounts live in `~/.claude/accounts/`, not the default file.** `--name work` writes `~/.claude/accounts/work.json`. A profile only uses it if `CLAUDE_CODE_CREDENTIALS_FILE` points there.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Claude Code account your standing default and `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-claudecode-refreshtoken --force` keeps failing (expired refresh token, account changed, plan changed), do a full `scorpiox-claudecode-login --force` to re-bind.
- **Subscription windows are enforced upstream.** The 5-hour and weekly limits are Anthropic's, not SCORPIOX CODE's — you'll see utilization and reset times with `scorpiox-claudecode-usage` rather than per-token metering.
