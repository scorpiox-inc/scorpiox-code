# Using Claude Code CLI Subscription in SCORPIOX CODE

You have a **Claude Code CLI** subscription (a Claude Pro or Max plan, or a Claude Code seat) but you don't want to run on a separate Anthropic API key or meter per-token API billing. You can use it directly in SCORPIOX CODE. Instead of metering Anthropic API tokens, the **Claude Code provider** signs you in with the same OAuth session login the Claude Code CLI uses and runs requests against the Anthropic endpoint your subscription already pays for.

This page walks through the **OAuth session login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles (`/profile` and `/use`), how to drive a login without a TTY (**machine mode**), and how the Claude Code subscription differs from standard Anthropic API-key usage.

Docs for SCORPIOX CODE @ `13253cf`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Claude / Claude Code subscription** | Pro, Max, or a Claude Code seat you already pay for |
| **You don't have (or don't want) an Anthropic API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The login flow works over SSH: approve in a browser on any device, paste the code back |
| **Driven by SCORPIO BOT** | The login can be started and finished as separate commands (`--start` / `--finish`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the Anthropic API with an API key (`ANTHROPIC_API_KEY`), use `PROVIDER=anthropic`. The two are different billing models for the same models — see [Claude Code provider vs. the Anthropic API-key provider](#claude-code-provider-vs-the-anthropic-api-key-provider) below.

> **Subscription, not API key.** The Claude Code provider authenticates with OAuth and draws on your account's usage allowance (the same 5-hour session and weekly windows the Claude Code CLI enforces). It does **not** accept an `ANTHROPIC_API_KEY` and does **not** bill per token.

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
2. **Log in** to your Claude account and approve the request.
3. Come back to your terminal and **paste the authorization code** when prompted (the `#state` fragment is stripped automatically if you paste the full redirect fragment).

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

### Creating the `claude_code` profile

At the end of a successful login, SCORPIOX CODE offers to write a ready-made profile for you (the prompt is skipped when stdin is not a TTY):

```
Create 'claude_code' config profile?
  Will create: /home/you/.claude/scorpiox-env/claude_code.txt
  Contents:
    PROVIDER=claude_code
    CLAUDE_CODE_TOKEN_SOURCE=local
    MODEL=claude-opus-4-6

Create? [Y/n]
```

If you say yes, the profile is written to `~/.claude/scorpiox-env/claude_code.txt` and you can activate it with `/profile claude_code`. You can also create the file by hand:

```
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
MODEL=claude-opus-4-6
```

---

## Step 2 — Turn on the Claude Code provider

The provider switches on with `PROVIDER=claude_code`. For a personal subscription you also set `CLAUDE_CODE_TOKEN_SOURCE=local` so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
```

Activate the profile (or set those keys in any cascade tier, or as OS environment variables) and SCORPIOX CODE connects.

```
/profile claude_code      # activate this profile and persist it
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `claude_code` to activate this provider. |
| `CLAUDE_CODE_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a Claude Code login you ran with `scorpiox-claudecode-login`. |
| `CLAUDE_CODE_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.claude/.credentials.json`. Point it at a named account (e.g. `~/.claude/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts full Claude model IDs or short aliases (see [Choosing a model](#choosing-a-model)). |
| `CLAUDE_CODE_API_URL` | text | `api.anthropic.com` | Host the provider calls. The `/v1/messages?beta=true` path is appended automatically. |
| `CLAUDE_CODE_PROXY_URL` | text | `anthropic.scorpiox.net` | Proxy host to route requests through instead of the direct endpoint. |
| `CLAUDE_CODE_PROXY_ENABLED` / `CLAUDE_CODE_PROXY_STRICT` | bool / bool | `false` / `false` | Enable proxy routing; when strict is off, SCORPIOX CODE falls back to the direct endpoint if the proxy is down. |
| `CLAUDE_CODE_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CLAUDE_CODE_TOKEN_SOURCE=http`. |
| `CLAUDE_CODE_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` | text | `proxy.scorpiox.net` / `9800` / *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. |

> **For a personal subscription, you only need two keys:** `PROVIDER=claude_code` and `CLAUDE_CODE_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `ssh` / `http` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Claude Code login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.claude/.credentials.json` under a `claudeAiOauth` section (`accessToken`, `refreshToken`, `expiresAt`, and the granted scopes). On every request, `CLAUDE_CODE_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** When the stored access token is close to expiring (a 5-minute buffer ahead of its `expiresAt`), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized, the provider refreshes once and retries. A second failure after a refresh is reported as a permanent auth error.
- **Remote / TCP sources always fetch fresh.** In `http`, `ssh`, or `tcp` modes the token is re-fetched on every request (the remote endpoint is the source of truth and may have revoked an old token), so there is no local expiry to worry about.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-claudecode-refreshtoken            # refresh if the token is expired
scorpiox-claudecode-refreshtoken --force    # always refresh
scorpiox-claudecode-refreshtoken --verbose  # show the resolved values
```

The refresh token (stored in the same `~/.claude/.credentials.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-claudecode-refreshtoken --force   # token expired — renew it
scorpiox-claudecode-login --force          # refresh failed / account changed — re-login
```

You can also inspect the resolved token without making a model request, for diagnostics. The helper prints a single JSON line with the access token and expiry (tokens are resolved from the active profile / config cascade):

```bash
scorpiox-claudecode-fetchtoken -config
```

### Checking your subscription usage

The provider runs against the same usage windows the Claude Code CLI enforces. Check how full they are with:

```bash
scorpiox-claudecode-usage            # human-readable summary
scorpiox-claudecode-usage --json     # raw JSON
```

You'll see per-window utilization and reset times — the 5-hour session window, the weekly window, per-model windows (Opus, Sonnet), and any extra-usage allowance — so you know when a window resets before you kick off a long run.

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

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile claude-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use claude-personal      # session-only — gone when the session ends
/profile                 # open the profile picker (or list if no picker is installed)
/profile off             # deactivate the profile
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

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

> Note: the login-offered `claude_code` profile ships with `MODEL=claude-opus-4-6` out of the box. Change it to `claude-sonnet-4-6` (or leave the alias as `opus` / `sonnet` / `haiku`) depending on the model you want by default.

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open between "here's the URL" and "paste the code" — a background agent, a CI step, or SCORPIO BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same PKCE flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node (add `--name` for a named account). |
| `--start` | Begin the login: returns the `auth_url`, and keeps the PKCE state for 10 minutes. |
| `--finish <code\|->` | Exchange the pasted code for tokens (`-` reads the code from stdin). |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--finish`: also write the `claude_code` profile. |

A typical machine-mode login:

```bash
# 1. Start — prints the URL to open
scorpiox-claudecode-login --start
# {"ok":true,"provider":"claudecode","flow":"paste_code","auth_url":"...","expires_in":600,...}

# 2. Open the auth_url in any browser, sign in, then finish
scorpiox-claudecode-login --finish <code> --create-profile
```

`--start` and `--finish` are separate invocations; the PKCE verifier and state are kept in `~/.claude/.login-pending/` (mode `0600`, valid for 10 minutes) so the two halves can run in different processes on the same node.

---

## Claude Code provider vs. the Anthropic API-key provider

It's easy to confuse the two because they both run Anthropic models. Here's the difference:

| | **Claude Code provider** (this page) | **Anthropic provider** (`PROVIDER=anthropic`) |
|---|---|---|
| `PROVIDER` value | `claude_code` | `anthropic` |
| **Authentication** | OAuth session login (`scorpiox-claudecode-login`) | `ANTHROPIC_API_KEY` bearer token |
| **Billing** | Draws on your Claude Code subscription (usage windows) | Pay-per-token Anthropic API billing |
| **What you need** | A Claude Pro / Max / Claude Code seat | An Anthropic API key and console account |
| **Token lifecycle** | Short-lived access token + automatic refresh | Static API key, no refresh |
| **Endpoint** | `api.anthropic.com` (Claude Code OAuth surface) | `api.anthropic.com` (standard API surface) |

They run the same underlying Claude models, but the **Claude Code provider bills against your existing subscription** while the **Anthropic provider bills per token against your API key**. Pick the one that matches how you already pay for Claude.

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md)
