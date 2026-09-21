# Using Claude Code CLI Subscription in SCORPIOX CODE

Have a Claude subscription (Claude Pro, Max, or a Claude Code seat) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering Anthropic API tokens, the **Claude Code provider** signs you in with the same OAuth session login the Claude Code CLI uses and runs requests against the Anthropic endpoint your subscription already pays for.

This page walks through the **OAuth session login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles, and how the Claude Code provider differs from the standard Anthropic API-key provider.

Source of truth: the Claude Code OAuth provider, the `scorpiox-claudecode-*` CLI tools, and the config cascade at commit `24427d8`.

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

Generating PKCE challenge...

Open this URL in your browser to authorize:

  https://claude.com/cai/oauth/authorize?code=true&response_type=code&...&code_challenge=...&code_challenge_method=S256

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

### Creating the `claude_code` profile

At the end of a successful login, SCORPIOX CODE offers to write a ready-made profile for you (the prompt is skipped when stdin is not a TTY):

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

## Step 2 — Turn on the Claude Code provider

The provider switches on with `PROVIDER=claude_code`. Because the token source defaults to a shared/remote mode, you also set `CLAUDE_CODE_TOKEN_SOURCE=local` so it reads the file your login just wrote. That's the whole setup for a personal subscription:

```
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
```

Activate the profile (or set those keys in any cascade tier / as environment variables) and SCORPIOX CODE connects.

```
/profile claude_code      # activate this profile for the session and persist it
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `claude_code` to activate this provider. |
| `CLAUDE_CODE_TOKEN_SOURCE` | choice | `tcp` | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a Claude subscription you logged into with `scorpiox-claudecode-login`. |
| `CLAUDE_CODE_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.claude/.credentials.json`. Point it at a named account (e.g. `~/.claude/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts full Claude model IDs or short aliases (see [Choosing a model](#choosing-a-model)). |
| `CLAUDE_CODE_API_URL` | text | `api.anthropic.com` | Host the provider calls. The `/v1/messages?beta=true` path is appended automatically. |
| `CLAUDE_CODE_PROXY_URL` | text | `anthropic.scorpiox.net` | Proxy host to route requests through instead of the direct endpoint. |
| `CLAUDE_CODE_PROXY_ENABLED` / `CLAUDE_CODE_PROXY_STRICT` | bool / bool | `false` / `false` | Enable proxy routing; when strict is off, SCORPIOX CODE falls back to the direct endpoint if the proxy is down. |
| `CLAUDE_CODE_REMOTE_URL` | text | `https://token.scorpiox.net/claude` | Token endpoint, used only when `CLAUDE_CODE_TOKEN_SOURCE=http`. |
| `CLAUDE_CODE_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` | text | `203.184.53.244` / `9800` / *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. |

> **For a personal subscription, you only need two keys:** `PROVIDER=claude_code` and `CLAUDE_CODE_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `ssh` / `http` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Claude Code login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.claude/.credentials.json` under a `claudeAiOauth` section (`accessToken`, `refreshToken`, `expiresAt`, and the granted scopes). On every request, `CLAUDE_CODE_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** Before a token is close to expiring (a 5-minute buffer ahead of its `expiresAt`), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request comes back with an auth error (HTTP 401/403), the provider refreshes once and retries the request.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-claudecode-refreshtoken            # refresh if the token is expired
scorpiox-claudecode-refreshtoken --force    # always refresh
scorpiox-claudecode-refreshtoken --verbose  # show old/new expiry
```

The refresh token (stored in the same `~/.claude/.credentials.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-claudecode-refreshtoken --force   # token expired — renew it
scorpiox-claudecode-login --force          # refresh failed / account changed — re-login
```

### Checking your subscription usage

The provider runs against the same usage windows the Claude Code CLI enforces. Check how full they are with:

```bash
scorpiox-claudecode-usage            # human-readable summary
scorpiox-claudecode-usage --json     # raw JSON
```

You'll see per-window utilization and reset times — the 5-hour session window, the 7-day weekly window, the per-model weekly windows (Opus, Sonnet), and any extra-usage allowance — so you know when a window resets before you kick off a long run.

---

## Multi-account: profiles and switching

Many people have more than one Claude account (personal + work, for example). The Claude Code provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-claudecode-login --name personal     # → ~/.claude/accounts/personal.json
scorpiox-claudecode-login --name work         # → ~/.claude/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account. Point `CLAUDE_CODE_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/claude-personal.txt
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/personal.json
MODEL=claude-opus-4-6
```

```
# ~/.claude/scorpiox-env/claude-work.txt
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/work.json
MODEL=claude-sonnet-4-6
```

### Switch accounts in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile claude-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use claude-personal      # session-only — gone when the session ends
/profile                 # open the profile picker (or list if no picker is installed)
/profile off             # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CLAUDE_CODE_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `CLAUDE_CODE_TOKEN_SOURCE`, `MODEL`, and `ACTIVE_PROFILE`, so you can confirm you're pointed at the account you expect before a long run.

---

## Choosing a model

`MODEL` accepts either a full Claude model ID or a short alias. The aliases at this commit are:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `sonnet` (provider default) |
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251001` |
| any full `claude-*` ID | passed through as-is |

You can pin a specific version per alias with the override keys (`CLAUDE_MODEL_OPUS_ID_OVERRIDE`, `CLAUDE_MODEL_SONNET_ID_OVERRIDE`, `CLAUDE_MODEL_HAIKU_ID_OVERRIDE`). Set `MODEL` in your profile or switch it at runtime with the `/model` command.

> Note: the login-offered `claude_code` profile ships with `MODEL=claude-opus-4-6` out of the box. Change it to `claude-sonnet-4-6` or leave the alias as `opus`/`sonnet`/`haiku` depending on the model you want by default.

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
| **Best for** | People who already pay for Claude / Claude Code | Pay-as-you-go API access, custom endpoints, Antigravity / Z.AI / custom backends |

Rule of thumb: **you have a subscription → `claude_code`. You have an API key or a custom endpoint → `anthropic`.**

---

## Gotchas

- **`CLAUDE_CODE_TOKEN_SOURCE` defaults to `tcp`, not `local`.** For a personal subscription you must set `CLAUDE_CODE_TOKEN_SOURCE=local` (the login-created profile does this for you). Leaving it at the default means SCORPIOX CODE will look for a TCP token source and won't find one on a normal machine.
- **The login command and the provider are separate.** `scorpiox-claudecode-login` only writes the token file. You still need `PROVIDER=claude_code` active (via a profile or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file holds a live refresh token.** `~/.claude/.credentials.json` can renew your account session — don't commit it, don't share it, and don't loosen its permissions.
- **Named accounts live in `~/.claude/accounts/`, not the default file.** `--name work` writes `~/.claude/accounts/work.json`. A profile only uses it if `CLAUDE_CODE_CREDENTIALS_FILE` points there.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Claude Code account your standing default and `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-claudecode-refreshtoken --force` keeps failing (expired refresh token, account changed, plan changed), do a full `scorpiox-claudecode-login --force` to re-bind.
- **Subscription windows are enforced upstream.** The 5-hour and weekly limits are Anthropic's, not SCORPIOX CODE's — you'll see utilization and reset times with `scorpiox-claudecode-usage` rather than per-token metering.
