# Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE

Have a ChatGPT or OpenAI Codex subscription but no OpenAI API key? You can use it directly in SCORPIOX CODE. Instead of metering API tokens, the **Codex provider** signs you in with the same OAuth device-code flow the Codex CLI uses and runs requests against the endpoint your subscription already pays for.

This page walks through the **device-code login**, how the token is stored and kept fresh, how to switch between multiple accounts with profiles, how to drive a login without a TTY (**machine mode**), and how the Codex subscription provider differs from the standard OpenAI API-key provider.

Docs for SCORPIOX CODE @ `b978da9`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a ChatGPT / Codex subscription** | A Codex plan tied to your OpenAI account that you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The device-code login works over SSH: open the link and enter the one-time code on any device, no browser or localhost redirect needed |
| **Driven by SCORPIOX BOT** | The login can be started and polled as separate commands (`--start` / `--poll`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the OpenAI API with an API key (`OPENAI_API_KEY`), use `PROVIDER=openai`. The two are different billing models for overlapping models — see [Codex provider vs. the OpenAI provider](#codex-provider-vs-the-openai-provider) below.

> **Subscription, not API.** The Codex provider authenticates with OAuth and draws on your account's usage allowance. It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token.

---

## Step 1 — Sign in with the OAuth device-code flow

The default login is the **OAuth device-code flow**. It is deliberately SSH- and headless-friendly: SCORPIOX CODE prints a link and a one-time code, you approve the sign-in from a browser on *any* device, and the terminal picks the result up automatically. No browser on the machine you're running from, no localhost redirect, no URL pasting.

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
   ABCD-1234

Device codes are a common phishing target. Never share this code.

Waiting for authorization....
```

Do exactly what it says:

1. **Open the printed link** in a browser — on any device, wherever you're signed into your OpenAI account.
2. **Sign in** and **enter the one-time code** shown in your terminal.
3. Come back to your terminal — it polls on its own and, the moment you approve, exchanges the code for an access + refresh token and saves them. On success you'll see:

```
Authorized. Exchanging code for tokens...
✓ Login successful
  Saved to /home/you/.codex/auth.json
```

A few things worth knowing about this flow:

- **The one-time code is a credential.** The login warns you itself: device codes are a common phishing target. Never share it, and never enter it on a page that wasn't the printed link.
- **It expires after 15 minutes.** If you don't approve in time, the login times out — just run it again for a fresh code.
- **No browser on the box? No problem.** Approve from your laptop, phone, or another machine and the terminal you started from saves the tokens.

### Browser fallback

If device-code login isn't available on your account, or you simply prefer a browser redirect, there's a fallback:

```bash
scorpiox-codex-login --browser
```

This opens an authorization URL instead. When the browser is on the same machine, the `localhost` callback is captured automatically; otherwise paste the resulting redirect URL (or just the code) when prompted. The `#` state fragment is stripped automatically if you paste the full redirect URL.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it. Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-codex-login --force
```

### Named accounts

By default the login writes to `~/.codex/auth.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-codex-login --name work
```

Each `--name <account>` login saves to `~/.codex/accounts/<account>.json` instead of the default file.

---

## Step 2 — Activate the Codex provider

Signing in stores the token; it does not by itself make SCORPIOX CODE use it. Right after a successful login, SCORPIOX CODE offers to create a ready-made **`codex` config profile**:

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
/profile codex     # activate and persist
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
| `CODEX_TOKEN_SOURCE` | choice | *(empty)* | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-codex-login`. Empty means the token is resolved from `scorpiox-env.txt` (the built-in default). |
| `CODEX_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CODEX_TOKEN_SOURCE=http`. |
| `CODEX_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. `CODEX_SSH_HOST` and `CODEX_SSH_USER` are required in that mode. |
| `CODEX_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort level: `low`, `medium`, `high`, `max`, or `off`. Empty means it isn't sent. |
| `MODEL` | text | *(empty)* | Which model to run. Empty resolves to the provider default `gpt-5.6-terra`. |

> **For a personal subscription you only need two keys:** `PROVIDER=codex` and `CODEX_TOKEN_SOURCE=local` (plus optionally `MODEL` and `CODEX_REASONING_EFFORT`). The `http` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Codex login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.codex/auth.json` (an `id_token`, `access_token`, and `refresh_token`, plus a `last_refresh` timestamp), with `0600` permissions. On every request, `CODEX_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. With `CODEX_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** When a token is close to its expiry (a 5-minute buffer), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries the request.
- **Remote sources always fetch fresh.** In `http` / `ssh` / `tcp` modes the token is re-fetched on every request (the remote server is the source of truth and may have revoked an old token), so there is no local expiry to worry about.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-codex-refreshtoken            # refresh if the token is expired
scorpiox-codex-refreshtoken --force    # always refresh
scorpiox-codex-refreshtoken --verbose  # show old/new tokens
```

The refresh token (stored in the same `~/.codex/auth.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-codex-refreshtoken --force    # token expired — renew it
scorpiox-codex-login --force           # refresh failed / account changed — re-login
```

You can also inspect the resolved token without making a model request, for diagnostics:

```bash
scorpiox-codex-fetchtoken -local       # read from ~/.codex/auth.json
scorpiox-codex-fetchtoken -config      # read CODEX_TOKEN_SOURCE from scorpiox-env.txt
scorpiox-codex-fetchtoken -ssh         # read from a remote machine via SSH
scorpiox-codex-fetchtoken -remote      # fetch from the HTTP endpoint
scorpiox-codex-fetchtoken -tcp         # fetch over the raw TCP socket
```

The output is a single JSON line: `{"access_token":"...","expires_at":...}`. Use it to confirm the token source resolves to the account you expect before a long run.

---

## Multi-account and profile switching

Many people have more than one OpenAI account (personal + work, for example). The Codex provider supports that in two layers: **multiple stored logins** and **named config profiles**.

### Store more than one login

Each `scorpiox-codex-login --name <account>` run writes a separate file under `~/.codex/accounts/`. To pin a specific login to a profile, point the profile at that account file; otherwise the provider falls back to the default `~/.codex/auth.json`.

### Switch accounts and models in-session

Switching is just switching profiles — no re-login, no restart:

```
/profile codex       # persistent — writes ACTIVE_PROFILE, survives restarts
/use codex           # session-only — gone when the session ends
/profile            # list available profiles and which is active
/profile off        # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CODEX_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open while a login is in flight — a background agent, a CI step, or SCORPIOX BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is byte-for-byte the original interactive behaviour.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node (and `--name` if given). |
| `--start` | Begin the login: returns the `verify_url` and `user_code`, and keeps the pending state in `~/.claude/.login-pending/` for 15 minutes. |
| `--poll` | One non-blocking check. Reports `pending` until the user approves, then exchanges and saves the tokens. |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--poll`: also write the `codex` profile. |
| `--name <account>` | Target a named account file (`~/.codex/accounts/<account>.json`). |

A typical machine-mode login:

```bash
# 1. Start — prints the link and the one-time code
scorpiox-codex-login --start
#   {"ok":true,"provider":"codex","flow":"device_code","verify_url":"https://auth.openai.com/codex/device","user_code":"ABCD-1234","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open that link on any device, sign in, and enter the user code.

# 3. Poll until it reports done (pass --create-profile to also write the profile)
scorpiox-codex-login --poll
#   {"ok":true,"provider":"codex","state":"pending","interval":5}          # not approved yet
#   {"ok":true,"provider":"codex","logged_in":true,"path":"/home/you/.codex/auth.json","profile":"codex","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the device-auth state lives in `~/.claude/.login-pending/codex-<name|default>.json` (mode `0600`), so the calls can be separate processes on the same node.
- **Fifteen-minute window.** If the code hasn't been approved within 15 minutes the pending state is dropped and `--poll` fails with `expired` — run `--start` again.
- **`--poll` is non-blocking.** Each call does one check and returns; keep calling it (honouring the `interval` it reports) until it reports `logged_in`.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"codex","error":"...","detail":"..."}`.
- **`--status` never prints tokens** — only whether a login exists and where.

---

## Choosing a model

`MODEL` accepts a full Codex model ID. The built-in set at this commit is:

| `MODEL` value | Notes |
|---------------|-------|
| *(empty)* | `gpt-5.6-terra` (provider default) |
| `gpt-5.6-terra` | Default model |
| `gpt-5.6-luna` | |
| `gpt-5.5` | |
| `gpt-5.4-mini` | |
| `codex-auto-review` | |

You can pin a specific version in your profile or switch it at runtime with the `/model` command. You can also set the reasoning effort with `CODEX_REASONING_EFFORT` (`low`, `medium`, `high`, `max`, or `off`), or switch it in-session with the `/reasoning_effort` command.

You can list every model the endpoint advertises with:

```bash
scorpiox-codex-models
```

---

## Codex provider vs. the OpenAI provider

It's easy to confuse the two because they both run OpenAI models. Here's the difference:

| | **Codex provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `codex` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-codex-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your ChatGPT / Codex subscription allowance | Pay-per-token API usage |
| **Token source** | `CODEX_TOKEN_SOURCE` (local file / http / ssh / tcp) | `OPENAI_API_KEY` |
| **Endpoint** | The Codex backend your subscription pays for (built in) | Any OpenAI-compatible `/v1/chat/completions` server (`OPENAI_BASE_URL`) — local or remote |
| **Best for** | People who already pay for ChatGPT / Codex | Pay-as-you-go API access, self-hosted and OpenAI-compatible servers |

Rule of thumb: **you have a subscription → `codex`. You have an API key or a self-hosted OpenAI-compatible server → `openai`.** See [Using the OpenAI Provider](openai-provider.md) for the full `openai` configuration and launch examples.

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-codex-login` writes the token file and offers a profile. You still need `PROVIDER=codex` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file holds a live refresh token.** `~/.codex/auth.json` can renew your account session — don't commit it, don't share it, and don't loosen its permissions.
- **Named accounts live in `~/.codex/accounts/`, not the default file.** `--name work` writes `~/.codex/accounts/work.json`. A profile only uses it if it points there.
- **The device code expires in 15 minutes.** A timed-out login just means running it again for a fresh code; a machine-mode pending state is dropped on the same window.
- **Never share the one-time device code.** It's a credential and a well-known phishing target. Anyone who has the code (and can open the link) can bind it to your account.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Codex account your standing default and `/use` to hop to it for a single session without writing anything.
- **Machine mode is additive.** `--status` / `--start` / `--poll` / `--cancel` are only active when you pass one of those flags; an unflagged `scorpiox-codex-login` is the interactive flow.
