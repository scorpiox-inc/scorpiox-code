# Using Claude Code CLI Subscription in SCORPIOX CODE

You have a **Claude subscription** — Pro, Max, or a Claude Code seat — and you want SCORPIOX CODE to use that subscription directly. No Anthropic API key, no pay-per-token billing.

One `scorpiox-claudecode-login` sign-in is all it takes. You approve in a browser, paste the authorization code back, and SCORPIOX CODE stores a refresh token, writes a ready-to-use profile, and keeps the session fresh on every subsequent request.

This page walks through the **OAuth PKCE session login**, where the token is stored, how refresh works, how to switch accounts and models with profiles (`/profile` and `/use`), how to drive a login without a TTY (machine mode), and how the Claude Code subscription provider differs from the standard Anthropic API-key provider.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** run `scorpiox-claudecode-login`, approve in a browser, paste the code back — SCORPIOX CODE stores a refresh token, writes a ready-to-use profile, and every subsequent request refreshes its own access token from that stored credential. No API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Claude / Claude Code subscription** | Pro, Max, or a Claude Code seat you already pay for |
| **You don't have (or don't want) an Anthropic API key** | No pay-per-token billing — requests draw on your subscription allowance |
| **Headless or over SSH** | Approve in a browser on any device, paste the code back into the SSH session |
| **Driven by SCORPIOX BOT or CI** | The login splits into separate `--start` / `--finish` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the Anthropic API with an **API key** (`PROVIDER=anthropic`), that is a different billing model — see [Claude Code subscription vs. Anthropic API-key usage](#claude-code-subscription-vs-anthropic-api-key-usage) below.

> **Subscription, not API key.** The Claude Code provider authenticates with OAuth and draws on your account's usage allowance. It does **not** accept an `ANTHROPIC_API_KEY`, and it does **not** bill per token against an API key you manage.

---

## Step 1 — Sign in with the OAuth PKCE session flow

The login is the **OAuth authorization-code (PKCE) flow**. It is deliberately SSH- and headless-friendly: you approve from a browser on whatever device is already signed in, then paste the resulting authorization code back into your terminal. No browser is required on the machine you are running from.

```bash
scorpiox-claudecode-login
```

You will see something like this:

```
scorpiox-claudecode-login v...

Generating PKCE challenge...

Open this URL in your browser to authorize:

  https://claude.com/cai/oauth/authorize?...&code_challenge=...&code_challenge_method=S256

After logging in, paste the authorization code below.
Code: >
```

Do exactly what it says:

1. **Open the printed URL** in a browser — on any device, wherever you are signed in to Claude.
2. **Log in** to your Claude account and authorize the request.
3. Come back to your terminal and **paste the authorization code** when prompted. If you paste the full redirect URL, SCORPIOX CODE extracts the `code=` parameter itself and ignores the rest.

SCORPIOX CODE then exchanges the code for tokens (the PKCE verifier proves the request came from the same terminal that started the flow) and saves the access token, refresh token, and expiry. On success you will see:

```
Login successful
  Saved to /home/you/.claude/.credentials.json
  Token expires in 86400s
```

A few things worth knowing about this flow:

- **Paste the code or the whole URL.** The prompt accepts either. If you paste the full redirect address, SCORPIOX CODE extracts the `code=` parameter itself.
- **Never share the authorization code.** It is a credential, it is single-use, and anyone holding it while you are mid-login can bind it to their own request.
- **No browser on the box? No problem.** Approve from your laptop, phone, or a kiosk and paste the code back into the SSH session.
- **The access token is short-lived by design** — you never handle it. See [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh).

### Re-running the login

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: /home/you/.claude/.credentials.json
Use --force to overwrite.
```

Use `--force` when you want to re-authorize deliberately (for example after a password change, a plan change, or a Google-side verification request that invalidated the stored refresh token):

```bash
scorpiox-claudecode-login --force
```

### Named accounts

By default the login writes to `~/.claude/.credentials.json`. To keep several Claude accounts side by side, give each one a name:

```bash
scorpiox-claudecode-login --name personal     # → ~/.claude/accounts/personal.json
scorpiox-claudecode-login --name work         # → ~/.claude/accounts/work.json
```

Each `--name <account>` login saves to `~/.claude/accounts/<account>.json` instead of the default file. You can then pin a profile to a specific account with `CLAUDE_CODE_CREDENTIALS_FILE` (see [Configuration keys](#configuration-keys)).

---

## Step 2 — Turn on the Claude Code provider

The login writes the token file **and** offers to write a ready-to-use profile. When it asks:

```
Create 'claude_code' config profile?
  Will create: /home/you/.claude/scorpiox-env/claude_code.txt
  Contents:
    PROVIDER=claude_code
    CLAUDE_CODE_TOKEN_SOURCE=local
    MODEL=claude-opus-4-6
Create? [Y/n]
```

Say yes (or create the file yourself), then activate it in-session:

```text
/profile claude_code     # persistent — writes ACTIVE_PROFILE, survives restarts
/use claude_code         # session-only — gone when the session ends
```

If you would rather configure it by hand, the minimal set for a personal subscription is two lines in any cascade tier:

```
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
```

`MODEL` defaults to `sonnet` (resolves to `claude-sonnet-4-6`) if left empty. The login-created profile ships `MODEL=claude-opus-4-6` out of the box — change it to `sonnet` or `claude-sonnet-4-6` if you want a faster, cheaper default.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `claude_code` to activate this provider. |
| `CLAUDE_CODE_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a login you ran with `scorpiox-claudecode-login`. |
| `CLAUDE_CODE_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.claude/.credentials.json`. Point it at a named account (e.g. `~/.claude/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts full Claude model IDs or short aliases (see [Choosing a model](#choosing-a-model)). Empty resolves to `sonnet`. |
| `CLAUDE_CODE_API_URL` | text | `api.anthropic.com` | Host the provider calls. The `/v1/messages?beta=true` path is appended automatically. |
| `CLAUDE_CODE_PROXY_URL` | text | *(empty)* | Proxy host to route requests through instead of the direct endpoint. Empty uses the direct `api.anthropic.com` endpoint. |
| `CLAUDE_CODE_PROXY_ENABLED` | bool | *(unset — falls back to `true` if `CLAUDE_CODE_PROXY_URL` is set)* | Enable proxy routing. If the proxy is down and `CLAUDE_CODE_PROXY_STRICT` is `0`, SCORPIOX CODE falls back to the direct endpoint. |
| `CLAUDE_CODE_PROXY_STRICT` | bool | `0` | When `1`, a proxy failure is a hard error. When `0` (default), SCORPIOX CODE retries against the direct endpoint. |
| `CLAUDE_CODE_CLI_VERSION` | text | `2.1.280` | Sent as the `user-agent` header. The endpoint varies its response by client identity — keep this set to a valid Claude CLI version. |
| `CLAUDE_CODE_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CLAUDE_CODE_TOKEN_SOURCE=http`. |
| `CLAUDE_CODE_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. `CLAUDE_CODE_SSH_HOST` and `CLAUDE_CODE_SSH_USER` are required in that mode. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. `TCP_HOST` is required in that mode. |

> **For a personal subscription you only need two keys:** `PROVIDER=claude_code` and `CLAUDE_CODE_TOKEN_SOURCE=local`. The `http`, `ssh`, and `tcp` token sources exist for shared or remote token setups — a token service that hands out tokens to many machines — and are not part of a normal Claude Code login.

### Where the credentials live

On login, SCORPIOX CODE writes the OAuth tokens to `~/.claude/.credentials.json` under a `claudeAiOauth` section:

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat01-...",
    "refreshToken": "...",
    "expiresAt": 1773000000000,
    "scopes": ["org:create_api_key", "user:profile", "user:inference",
               "user:sessions:claude_code", "user:mcp_servers", "user:file_upload"]
  }
}
```

- **`accessToken`** — short-lived, refreshed automatically by the provider. You never manage it.
- **`refreshToken`** — long-lived, used by `scorpiox-claudecode-refreshtoken` to renew the access token. This is the credential that keeps your session alive between logins.
- **`expiresAt`** — Unix timestamp in milliseconds. The provider checks this on every request and refreshes proactively 5 minutes before expiry.

The file holds a live refresh token: it can renew your Claude session without you. Do not commit it, do not share it, and do not loosen its permissions.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you never manage it. With `CLAUDE_CODE_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** The provider checks the token's `expiresAt` before each request. If the token is within 5 minutes of expiring (a buffer to avoid mid-request expiration, since streaming responses can take 60+ seconds), the provider spawns `scorpiox-claudecode-refreshtoken --force`, which performs a refresh-token grant against `https://console.anthropic.com/v1/oauth/token` and writes the updated tokens back to the credentials file. The provider then re-reads the file and uses the fresh access token.
- **Reactive recovery.** If a request comes back with HTTP 401 or 403 (the token expired or was revoked), the provider refreshes once and retries the request with the new token. A second 401 after a refresh is reported as a permanent auth failure.
- **Remote/SSH/TCP sources always fetch fresh.** In `http`, `ssh`, or `tcp` modes the token is fetched from the token service on every request — the service is the source of truth and may have rotated or revoked a token at any time.

You can also resolve the token without making a model request, for diagnostics:

```bash
scorpiox-claudecode-fetchtoken -local      # read from ~/.claude/.credentials.json
scorpiox-claudecode-fetchtoken -config     # read CLAUDE_CODE_TOKEN_SOURCE from scorpiox-env.txt
scorpiox-claudecode-fetchtoken -ssh        # read from a remote machine via SSH
scorpiox-claudecode-fetchtoken -remote     # fetch from the HTTP endpoint
scorpiox-claudecode-fetchtoken -tcp        # fetch over the raw TCP socket
```

The output is a single JSON line:

```
{"access_token":"sk-ant-oat01-...","expires_at":1773000000000}
```

Run it before a long run to confirm the token source resolves to the account you expect. The access token in that output is live — treat the line as a secret and do not paste it anywhere.

### Manual refresh

```bash
scorpiox-claudecode-refreshtoken            # refresh if the token is expired
scorpiox-claudecode-refreshtoken --force    # always refresh
scorpiox-claudecode-refreshtoken --verbose  # show old/new tokens
```

The refresh token (stored in the same credentials file) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always:

```bash
scorpiox-claudecode-refreshtoken --force   # token expired — renew it
scorpiox-claudecode-login --force          # refresh failed / account changed — re-login
```

---

## Multi-account and profile switching

Many people have more than one Claude account (personal + work, for example). The Claude Code provider supports that in two layers: **multiple stored logins** and **named config profiles**.

### Store more than one login

Each `scorpiox-claudecode-login --name <account>` run writes a separate file under `~/.claude/accounts/`. To pin a specific login to a profile, set `CLAUDE_CODE_CREDENTIALS_FILE` in that profile to the path you want:

```
# ~/.claude/scorpiox-env/claude_code_personal.txt
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/personal.json
MODEL=claude-sonnet-4-6
```

```
# ~/.claude/scorpiox-env/claude_code_work.txt
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/work.json
MODEL=claude-opus-4-6
```

If `CLAUDE_CODE_CREDENTIALS_FILE` is empty, the provider falls back to `~/.claude/.credentials.json`.

### Switch accounts and models in-session

With those profiles in place, switching accounts or models is just switching profiles — no re-login, no restart:

```text
/profile claude_code_work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use claude_code_personal     # session-only — gone when the session ends
/profile                       # list profiles and the active one (opens the picker where available)
/profile off                   # deactivate and drop back to the base cascade
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the switch cannot complete, SCORPIOX CODE reverts to the previous profile — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what is actually configured

Use `scorpiox-config` to see resolved values and where each one comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CLAUDE_CODE_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `CLAUDE_CODE_TOKEN_SOURCE`, and `MODEL`, so you can confirm you are pointed at the account and model you expect before a long run.

### Check your subscription usage

The Claude Code subscription enforces usage windows (5-hour session and weekly limits). You can see the utilization and reset times:

```bash
scorpiox-claudecode-usage            # readable usage summary
scorpiox-claudecode-usage --json     # raw JSON to stdout
```

Inside SCORPIOX CODE, the in-session `/usage` command runs this same tool under the hood and shows the usage table when a `claude_code` provider is active.

---

## Choosing a model

`MODEL` in the profile (or the per-provider key) selects the model. The provider accepts a short alias or a full model ID, and resolves it before the request is sent.

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-4-6` (provider default `sonnet`) |
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251001` |
| any `claude-*` ID | passed through as-is |

You can pin a specific version per alias with the override keys (`CLAUDE_MODEL_OPUS_ID_OVERRIDE`, `CLAUDE_MODEL_SONNET_ID_OVERRIDE`, `CLAUDE_MODEL_HAIKU_ID_OVERRIDE`). Set `MODEL` in your profile, or switch it at runtime with the `/model` command, which takes the same values as `MODEL`.

You can list every model the endpoint advertises with:

```bash
scorpiox-claudecode-models              # raw JSON list
scorpiox-claudecode-models <model-id>   # single model detail
```

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open between "here is the URL" and "paste the code" — a background agent, a CI step, or SCORPIOX BOT's provider flow. The login ships an **additive machine mode**: the same PKCE flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line. |
| `--start` | Begin the login: returns the `auth_url`, and keeps the PKCE state for 10 minutes. |
| `--finish <code\|->` | Exchange the pasted code for tokens (`-` reads the code from stdin). |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--finish`: also write the `claude_code` profile. |
| `--name <account>` | Pin the login to a named account file. |

A typical machine-mode login:

```bash
# 1. Start — prints the URL to open
scorpiox-claudecode-login --start
# {"ok":true,"provider":"claudecode","flow":"paste_code","auth_url":"https://claude.com/cai/oauth/authorize?...","expires_in":600,"hint":"Sign in, then paste the code shown on the callback page."}

# 2. Open that URL in any browser, sign in, approve, copy the code.

# 3. Finish — exchange the code
scorpiox-claudecode-login --finish "$CODE"
# {"ok":true,"provider":"claudecode","logged_in":true,"path":"/home/you/.claude/.credentials.json","expires_in":86400,"profile":"claude_code","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--finish` the PKCE verifier and `state` live in `~/.claude/.login-pending/` (mode `0600`), so the two calls can be separate processes on the same node.
- **Ten-minute window.** If `--finish` comes more than 10 minutes after `--start`, the pending state is dropped and the command fails with `expired` — run `--start` again for a fresh challenge.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"claudecode","error":"...","detail":"..."}`.
- **`--finish` does the whole account setup.** It runs the same flow as an interactive login — credential save, permissions restriction, optional profile write — so a completed `--finish` leaves the machine in exactly the state an interactive login would.
- **`--status` tells you where you stand.** It reports `logged_in`, whether a refresh token is stored (`has_refresh`), whether a login is mid-flight (`pending`), the token expiry, and whether the profile exists. Use it to decide whether a machine needs a login at all.

---

## Claude Code subscription vs. Anthropic API-key usage

It is easy to confuse the two because they both run Anthropic models. Here is the difference:

| | **Claude Code provider** (this page) | **Anthropic provider** (`PROVIDER=anthropic`) |
|---|---|---|
| `PROVIDER` value | `claude_code` | `anthropic` |
| **Authentication** | OAuth PKCE session login (`scorpiox-claudecode-login`) | `ANTHROPIC_API_KEY` bearer token |
| **Billing** | Your Claude / Claude Code subscription allowance (5-hour + weekly windows) | Pay-per-token API usage against your key |
| **Token source** | `CLAUDE_CODE_TOKEN_SOURCE` (local file / http / ssh / tcp) | `ANTHROPIC_API_KEY` (+ optional `ANTHROPIC_API_URL`) |
| **Endpoint** | `api.anthropic.com/v1/messages?beta=true` | Any Anthropic endpoint (`ANTHROPIC_API_URL`) |
| **Token lifecycle** | Short-lived access token, refreshed from the stored refresh token on every request | Static API key, no refresh |
| **Best for** | People who already pay for Claude / Claude Code | Pay-as-you-go API access, custom endpoints, gateways |

Rule of thumb: **you have a Claude subscription, use `claude_code`. You have an API key or a custom endpoint, use `anthropic`.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-claudecode-login` writes the token file and offers a profile; it does not switch your session. Activate `claude_code` with `/profile` (or set `PROVIDER=claude_code` yourself) before SCORPIOX CODE uses it.
- **The credentials file holds a live refresh token.** `~/.claude/.credentials.json` can renew your account session — do not commit it, do not share it, and do not loosen its permissions.
- **`CLAUDE_CODE_TOKEN_SOURCE=local` still requires a login.** The value means "read the refresh token from `~/.claude/.credentials.json`". If you have never logged in, that file does not exist and every request fails with a token error. Run the login first.
- **Named accounts live in `~/.claude/accounts/`, not the default file.** `--name work` writes `~/.claude/accounts/work.json`. A profile only uses it if `CLAUDE_CODE_CREDENTIALS_FILE` points there.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Claude Code account your standing default; use `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic; re-login is the fallback.** If `scorpiox-claudecode-refreshtoken --force` keeps failing (expired refresh token, account changed, plan changed), do a full `scorpiox-claudecode-login --force` to re-bind.
- **Subscription windows are enforced upstream.** The 5-hour and weekly limits are Anthropic's, not SCORPIOX CODE's. You can see the utilization and reset times with `scorpiox-claudecode-usage`.
- **The authorization code is single-use and a credential.** Never share it, and never paste it into a ticket or a chat. If the login stalls or the code is rejected, just run `scorpiox-claudecode-login` again for a fresh challenge.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the switch cannot complete.
- **The client identity matters.** The endpoint varies its response by `user-agent` and client headers. SCORPIOX CODE sends the same client identity the Claude CLI uses. If you write a custom client or proxy, match the identity or you will see models missing or `UNSUPPORTED_CLIENT` errors.

---

## Quick reference

| Goal | What to do |
|------|-----------|
| Sign in | `scorpiox-claudecode-login` (or `--force` to re-authorize) |
| Named account | `scorpiox-claudecode-login --name <account>` |
| Check login state | `scorpiox-claudecode-login --status` |
| Headless login | `--start` / `--finish <code\|->` / `--status` / `--cancel` |
| Activate the provider | `PROVIDER=claude_code` + `CLAUDE_CODE_TOKEN_SOURCE=local` |
| Pin a specific account | `CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/<name>.json` |
| Pin a model | `MODEL=claude-opus-4-6` (or `sonnet`, `haiku`, or a full `claude-*` ID) |
| Switch profiles | `/profile <name>` (persistent) or `/use <name>` (session-only) |
| Check subscription usage | `scorpiox-claudecode-usage` (or in-session `/usage` when `claude_code` is active) |
| List available models | `scorpiox-claudecode-models` |
| Inspect token | `scorpiox-claudecode-fetchtoken -config` |

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Direct Anthropic API & Custom Endpoints](anthropic-provider.md) — the key-based path to Claude models, and the `ANTHROPIC_AUTH_PROVIDER` proxy modes.
- [Using Google Antigravity CLI Subscription](antigravity-provider.md) — the Antigravity subscription path to Claude and Gemini models.
- [API Traffic Logging](traffic-logging.md) — the verbatim on-disk record of every request, including the Anthropic wire format.
