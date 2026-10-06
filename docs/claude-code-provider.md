# Using Claude Code CLI Subscription in SCORPIOX CODE

You have a **Claude Code CLI** subscription — a Claude Pro or Max plan, or a Claude Code seat — and you would rather not buy a separate Anthropic API key or meter every turn against per-token billing. You can use it directly in SCORPIOX CODE. Instead of paying per token, the **Claude Code provider** (`PROVIDER=claude_code`) signs you in with the same **OAuth session login** the Claude Code CLI performs and sends every request to the Anthropic endpoint your subscription already pays for.

This page walks through the session login, where the token is stored, how refresh works (and why there is nothing to manage), how to juggle several accounts with profiles (`/profile` and `/use`), how to drive a login without a TTY (**machine mode**), and how a Claude Code subscription differs from standard Anthropic API-key usage.

Docs for SCORPIOX CODE @ `0cd528b`.

> **The whole idea in one line:** run `scorpiox-claudecode-login`, approve in a browser, paste the code back — SCORPIOX CODE stores the OAuth tokens, writes a ready-to-use profile, and every later request renews its own short-lived access token from the stored refresh token. No API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Claude / Claude Code subscription** | Pro, Max, or a Claude Code seat you already pay for |
| **You don't have (or don't want) an Anthropic API key** | Requests draw on your subscription's usage windows, not per-token billing |
| **You share one login across several machines** | The token can be fetched over SSH, HTTP, or a raw TCP socket instead of a local file |
| **Headless / over SSH** | The login is paste-code based: approve in a browser on any device, paste the code back into the terminal |
| **Driven by SCORPIO BOT** | The login splits into separate `--start` / `--finish` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling Anthropic with an **API key**, that is a different provider: `PROVIDER=anthropic`, described in [Direct Anthropic API & Custom Endpoints](anthropic-provider.md). The two run the same models on different billing models — see [Claude Code subscription vs. standard API-key usage](#claude-code-subscription-vs-standard-api-key-usage) below.

> **Subscription, not API key.** The Claude Code provider authenticates with OAuth and draws on your account's usage allowance — the same 5-hour session and weekly windows the Claude Code CLI enforces. It does not accept an `ANTHROPIC_API_KEY` and does not bill per token.

---

## Step 1 — Sign in with the OAuth session flow

The login is the **OAuth authorization-code (PKCE) flow**, deliberately shaped so it works over SSH and on headless machines: you approve in a browser on whatever device is already signed in to Claude, then paste the resulting authorization code back into the terminal. Nothing long-lived is pasted, and no browser is needed on the machine you are running from.

```bash
scorpiox-claudecode-login
```

You will see something like this:

```
scorpiox-claudecode-login v...

Generating PKCE challenge...

Open this URL in your browser to authorize:

  https://claude.com/cai/oauth/authorize?code=true&response_type=code&...&code_challenge=...&code_challenge_method=S256

After logging in, paste the authorization code below.
Code: >
```

Do exactly what it says:

1. **Open the printed URL** in a browser — any device, wherever you are signed in to Claude.
2. **Log in** to your Claude account and approve the request. The requested scopes cover your profile, inference on your subscription, Claude Code sessions, MCP servers, API-key creation, and file upload — the same set the official CLI asks for.
3. Come back to the terminal and **paste the authorization code** shown on the callback page. If you paste the full redirect fragment instead, the trailing `#state` part is stripped automatically.

SCORPIOX CODE then exchanges the code for tokens and saves them:

```
Login successful
  Saved to /home/you/.claude/.credentials.json
  Token expires in 86400s
```

Things worth knowing:

- **The access token is short-lived by design** — and you never manage it. See [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh).
- **The authorization code is a credential.** Anyone who captures it before it is exchanged can bind it to your account. Do not paste it into tickets, chats, or screenshots.
- **No browser on the box? No problem.** Approve from your laptop, phone, or a kiosk and paste the code back into the SSH session.

### Re-running the login

The login refuses to overwrite an existing credential by default:

```
Credentials already exist: /home/you/.claude/.credentials.json
Use --force to overwrite.
```

Pass `--force` when you want to re-authorize deliberately — after an account change, after a plan change, or when a refresh has stopped working:

```bash
scorpiox-claudecode-login --force
```

### More than one login: named accounts

`--name` saves each login as its own file instead of the default one:

```bash
scorpiox-claudecode-login --name personal   # -> ~/.claude/accounts/personal.json
scorpiox-claudecode-login --name work       # -> ~/.claude/accounts/work.json
```

The default credential file and the named files are independent: logging in with `--name` never touches `~/.claude/.credentials.json`. To actually *use* a named account, point a profile at it with `CLAUDE_CODE_CREDENTIALS_FILE` — see [Multi-account: profiles and switching](#multi-account-profiles-and-switching).

### The login offers to create the profile

After a successful interactive login, SCORPIOX CODE offers to write a ready-to-use profile:

```
Create 'claude_code' config profile?
  Will create: /home/you/.claude/scorpiox-env/claude_code.txt
  Contents:
    PROVIDER=claude_code
    CLAUDE_CODE_TOKEN_SOURCE=local
    MODEL=claude-opus-4-6

Create? [Y/n]
```

Say yes and you can activate it immediately with `/profile claude_code`. Two details about this offer:

- **It only appears when the terminal is interactive.** Piping the login (or running it from a script) skips the prompt — use `--create-profile` in machine mode, or write the file by hand.
- **If the profile already exists it is left alone** unless you passed `--force`.

---

## Step 2 — Turn on the Claude Code provider

The provider switches on with `PROVIDER=claude_code`. For a personal subscription you also set `CLAUDE_CODE_TOKEN_SOURCE=local`, so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=claude_code
CLAUDE_CODE_TOKEN_SOURCE=local
```

Activate it and SCORPIOX CODE connects:

```text
/profile claude_code      # activate the login-written profile and persist it
```

The login command and the provider are two separate things: the login writes the token file, the provider reads it. Running the login never switches your session — activate the profile (or set the keys yourself) before SCORPIOX CODE uses it.

You can also pick the provider for a single run without touching any file:

```bash
sx --provider claude_code -p "explain this diff"
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `claude_code` to activate this provider. |
| `CLAUDE_CODE_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local`, `http` (alias `remote`), `ssh`, or `tcp`. Use **`local`** for a login you ran with `scorpiox-claudecode-login`. |
| `CLAUDE_CODE_CREDENTIALS_FILE` | text | *(empty)* | Override the local credential path. Defaults to `~/.claude/.credentials.json`. Point it at a named account (`~/.claude/accounts/work.json`) to pin a profile to one login. |
| `MODEL` | text | `sonnet` | Which model to run. Short aliases or full Claude model IDs — see [Choosing a model](#choosing-a-model). |
| `CLAUDE_CODE_API_URL` | text | `api.anthropic.com` | Host the provider calls. **Domain only** — the `/v1/messages` path is appended automatically. |
| `CLAUDE_CODE_PROXY_URL` | text | `anthropic.scorpiox.net` | Proxy host used instead of the direct endpoint when the proxy is enabled. |
| `CLAUDE_CODE_PROXY_ENABLED` | bool | `0` | Route requests through `CLAUDE_CODE_PROXY_URL` instead of straight to Anthropic. |
| `CLAUDE_CODE_PROXY_STRICT` | bool | `0` | `1` = fail when the proxy is down. `0` = fall back to the direct endpoint. |
| `CLAUDE_CODE_CLI_VERSION` | text | `2.1.280` | CLI version reported in the user agent. Bump it if Anthropic ever requires a newer client — no rebuild needed. The OS environment variable of the same name wins over the file value. |
| `CLAUDE_CODE_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CLAUDE_CODE_TOKEN_SOURCE=http`. |
| `TOKEN_HTTP_API_KEY` | text | *(empty)* | Optional bearer token for the HTTP token endpoint, sent as an `Authorization: Bearer` header. Shared across providers. |
| `CLAUDE_CODE_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=ssh` — read the token from a remote machine over SSH. The port is optional (default `22`). |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | `proxy.scorpiox.net` / `9800` / *(empty)* / *(empty)* | Used only when `CLAUDE_CODE_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. `TCP_HOST` accepts a comma-separated list for failover; `TCP_API_KEY` enables SXV1 auth; `TCP_UPSTREAM` selects a 1-based upstream on the token server. |

> **For a personal subscription you only need two keys:** `PROVIDER=claude_code` and `CLAUDE_CODE_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `ssh`, `http`, and `tcp` sources exist for shared or remote token setups and are not part of a normal Claude Code login.

> **Two URL conventions, one provider family.** `CLAUDE_CODE_API_URL` is a bare domain — the path is appended for you. The Anthropic provider's `ANTHROPIC_API_URL` is the opposite: a *full* endpoint including `/v1/messages`. Mixing the two up is the most common misconfiguration when people switch between them.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.claude/.credentials.json` under a `claudeAiOauth` section — the access token, the refresh token, the expiry timestamp, and the granted scopes:

```json
{
  "claudeAiOauth": {
    "accessToken": "...",
    "refreshToken": "...",
    "expiresAt": 1770699673073,
    "scopes": ["org:create_api_key", "user:profile", "user:inference",
               "user:sessions:claude_code", "user:mcp_servers", "user:file_upload"]
  }
}
```

With `CLAUDE_CODE_TOKEN_SOURCE=local`, every request reads that file (or the file `CLAUDE_CODE_CREDENTIALS_FILE` points at). No API key is involved, and none is ever written. Named accounts live beside it in `~/.claude/accounts/<name>.json`.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived — roughly a day — and the refresh token stored next to it is what renews it without a new sign-in. SCORPIOX CODE handles the whole lifecycle:

- **Proactive refresh.** In `local` mode the provider renews the token when it is within **5 minutes** of expiring, so a long streaming turn never runs into an expired credential mid-flight.
- **Reactive recovery.** If a request still comes back `401` (expired or revoked token) or `403`, the provider refreshes once and retries the same request immediately. A second `401` after a refresh is reported as a permanent authentication failure rather than retried forever.
- **Remote sources always fetch fresh.** In `http`, `ssh`, or `tcp` modes the token is re-fetched on **every** request — the remote endpoint is the source of truth and may have rotated or revoked an old token at any time. There is no local expiry to track.
- **Retries are bounded.** Token renewal itself is attempted twice with a short pause between attempts before the request is abandoned.

You can also force a refresh from the command line:

```bash
scorpiox-claudecode-refreshtoken            # refresh only if the token is expired
scorpiox-claudecode-refreshtoken --force    # always refresh
scorpiox-claudecode-refreshtoken --credentials-file ~/.claude/accounts/work.json
```

The refresh tool checks the stored expiry, renews the tokens with Anthropic's OAuth endpoint, backs up the credential file to `<file>.backup` before writing, and prints the new expiry time. `--verbose` additionally shows the old and new tokens — useful for diagnosing, never for sharing.

If requests keep coming back unauthorized, the fix is almost always one of:

```bash
scorpiox-claudecode-refreshtoken --force   # token expired — renew it
scorpiox-claudecode-login --force          # refresh failed / account changed — re-login
```

### Inspect the token without a model call

The token helper resolves the credential exactly the way the provider does — following the active profile and the whole config cascade — and prints a single JSON line with the access token and its expiry:

```bash
scorpiox-claudecode-fetchtoken -config          # follow CLAUDE_CODE_TOKEN_SOURCE
scorpiox-claudecode-fetchtoken -local           # read the local credential file
scorpiox-claudecode-fetchtoken -config --verbose
```

The same tool drives every source: `-local`, `-ssh`, `-remote` (alias `-http`), and `-tcp`, plus `-config` to follow the configured source. It auto-detects both credential shapes — the local `claudeAiOauth` layout and a flat `{"access_token":..., "expires_in":...}` API response — so the same command works no matter where the token comes from. `--verbose` prints the resolved config keys and which cascade tier each one came from, which is the fastest way to answer "why is it reading *that* file?".

---

## Choosing a model

`MODEL` accepts a short alias or a full Claude model ID. The aliases at this commit resolve to:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `sonnet` — the provider default |
| `opus` | `claude-opus-4-6` |
| `sonnet` | `claude-sonnet-4-6` |
| `haiku` | `claude-haiku-4-5-20251001` |
| any full `claude-*` ID | passed through as-is |

Pin a specific version per alias with the override keys (`CLAUDE_MODEL_OPUS_ID_OVERRIDE`, `CLAUDE_MODEL_SONNET_ID_OVERRIDE`, `CLAUDE_MODEL_HAIKU_ID_OVERRIDE`). Set `MODEL` in your profile, or switch it at runtime with `/model`, which takes the same values.

> The login-written `claude_code` profile ships with `MODEL=claude-opus-4-6`. Change it to `claude-sonnet-4-6` — or set the alias `opus` / `sonnet` / `haiku` — if you want a different default.

To see the model IDs the endpoint currently serves:

```bash
scorpiox-claudecode-models               # list every model (raw JSON)
scorpiox-claudecode-models claude-opus-4-6   # detail for one model
```

---

## What a request looks like

The provider speaks the Anthropic Messages API **as the Claude Code CLI does**, so your subscription recognizes the traffic as a Claude Code session:

- The request goes to `api.anthropic.com/v1/messages` with the Claude Code beta flags, the Anthropic API version header, and a user agent of the form `claude-cli/<version> (external, cli)` plus the CLI's client headers. The version comes from `CLAUDE_CODE_CLI_VERSION`.
- Authentication is an OAuth **bearer token** — the access token from your credential file — not an API key.
- **The system prompt is sent as two text blocks**, each marked for ephemeral caching, and cache breakpoints are placed on the last block of each role. Between turns the stable prefix is served from the provider cache instead of being re-processed, which is what keeps a long agentic session affordable on a subscription.
- **Extended thinking** is included when `THINKING=1`, with `THINKING_BUDGET` tokens (default `10000`, clamped to 1,000–128,000).
- **Tool definitions** are included when `TOOLS=1`.
- The output budget defaults to 32,000 tokens per turn.
- Each request allows up to ten minutes, so long agentic turns are not cut off mid-flight.
- `STREAMING=0` (the default) sends one request and parses one complete response; `STREAMING=1` asks for server-sent events instead and parses the stream. Pair streaming with `USAGE_SPEED_ESTIMATE=1` to get live `~pp` / `~tg` figures in the status bar, timed from the first token the endpoint produces.

### Retries and transient errors

Transient failures are retried inside the request: up to **ten attempts** with an exponential backoff doubling from one second, capped at a minute, plus jitter so many agents do not retry in lockstep. These statuses count as transient:

| Status | Meaning |
|--------|---------|
| `429` | Rate limited |
| `500` | Internal server error |
| `502` | Bad gateway |
| `503` | Service unavailable |
| `529` | Site overloaded |
| `403` | Forbidden — retried once after a token refresh, then treated as transient |

`401` is handled separately: one token refresh and one immediate retry, then a hard authentication failure. Above the provider retries, the agent loop has its own wider retry policy for long runs (`AGENT_RETRY_MAX`, `AGENT_RETRY_INITIAL_DELAY`, `AGENT_RETRY_MAX_DELAY`) — see [Configuration and Profiles](scorpiox-env.md).

### Where the proxy fits in

Set `CLAUDE_CODE_PROXY_ENABLED=1` and requests go through `CLAUDE_CODE_PROXY_URL` instead of straight to Anthropic. The Anthropic host header is always sent, so the proxy can stay transparent. If the proxy is unreachable and `CLAUDE_CODE_PROXY_STRICT=0` (the default), the provider falls back to the direct endpoint for that request and restores the proxy for the next one; with strict mode on, a dead proxy is a hard error instead of a silent detour.

---

## Where every request is recorded

Every request and response is written verbatim to disk: the endpoint URL, all headers (the bearer token redacted), the complete request body, and the response payload with its status and the rate-limit headers Anthropic returned. When session logging is active the capture lands in that session's `traffic/` folder; otherwise it goes to a timestamped folder under `.scorpiox/traffics/providers/claude_code/`. See [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) for the layout and how to replay a captured call.

---

## Checking your subscription usage

The provider runs against the same usage windows the Claude Code CLI enforces. Check how full they are with:

```bash
scorpiox-claudecode-usage            # human-readable summary
scorpiox-claudecode-usage --json     # raw JSON
```

You get one line per window with utilization and the reset time in UTC and your local timezone:

```
Session (5h)          42% used  Resets in 2h13m   2026-10-06 22:00:00 UTC (2026-10-07 11:00:00)
Weekly (7d)           12% used  Resets in 3d04h   2026-10-09 22:00:00 UTC (2026-10-10 11:00:00)
Weekly Opus            8% used  ...
Extra usage:           Off
```

The windows include the 5-hour session window, the weekly window, per-model weekly windows (Opus, Sonnet, OAuth apps), and any extra-usage allowance with its monthly limit — so you know when a window turns over before you kick off a long run. Inside SCORPIOX CODE, the in-session `/usage` command runs this same tool and titles the popup **Claude Usage** while this provider is active.

Live turns also carry the rate-limit headers back in every response, and the status bar turns them into two indicators: `5h:<pct>` and `7d:<pct>`, green below 50%, yellow from 50–80%, red above. They appear only once the provider has reported them, and they are the reason the status bar looks different on a subscription provider than on the key-based Anthropic provider, which has no such headers. See [Token Usage Observability](usage-observability.md).

---

## Multi-account: profiles and switching

Many people hold more than one Claude account — personal and work, for example. The Claude Code provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-claudecode-login --name personal   # -> ~/.claude/accounts/personal.json
scorpiox-claudecode-login --name work       # -> ~/.claude/accounts/work.json
```

### Bind each login to a profile

Make one profile per account and point `CLAUDE_CODE_CREDENTIALS_FILE` at the right file, so the profile always talks to that account:

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

Profiles are looked up in `.scorpiox/scorpiox-env/` (project), `~/.claude/scorpiox-env/` (user), and a `scorpiox-env/` directory next to the installed binaries (global) — the highest tier wins whole-file, so a project-level profile of the same name shadows your personal one for sessions started in that repo.

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```text
/profile claude-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use    claude-personal   # session-only — gone when the session ends
/profile                  # open the profile picker (or list profiles if no picker is installed)
/profile off              # deactivate the profile, drop back to the base cascade
```

Both commands reload the provider live, so the new account and model take effect immediately, and both revert automatically if the new profile cannot initialize — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what is actually configured

Use `scorpiox-config` to see resolved values and where each one comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CLAUDE_CODE_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

`--verbose` shows the resolved `CLAUDE_CODE_TOKEN_SOURCE`, `MODEL`, and `ACTIVE_PROFILE`, so you can confirm you are pointed at the account you expect before a long run.

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open between "here is the URL" and "paste the code" — a background agent, a CI step, or SCORPIO BOT's provider flow. The login ships an **additive machine mode**: the same PKCE flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line (add `--name` for a named account). |
| `--start` | Begin the login: returns the `auth_url`, and keeps the PKCE state for 10 minutes. |
| `--finish <code\|->` | Exchange the pasted code for tokens (`-` reads the code from stdin). |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--finish`: also write the `claude_code` profile. |
| `--force` | Replace existing credentials. |
| `--name <account>` | Operate on a named account file instead of the default one. |

A typical machine-mode login:

```bash
# 1. Start — prints the URL to open
scorpiox-claudecode-login --start
# {"ok":true,"provider":"claudecode","flow":"paste_code","auth_url":"https://claude.com/cai/oauth/authorize?...","expires_in":600,"hint":"Sign in, then paste the code shown on the callback page."}

# 2. Open that URL in any browser, sign in, approve, copy the code.

# 3. Finish — exchange the code
scorpiox-claudecode-login --finish "$CODE" --create-profile
# {"ok":true,"provider":"claudecode","logged_in":true,"path":"/home/you/.claude/.credentials.json","expires_in":86400,"profile":"claude_code","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--finish` the PKCE verifier and state live in `~/.claude/.login-pending/claudecode-<name|default>.json` (mode `0600`), so the two calls can be separate processes on the same node.
- **Ten-minute window.** If `--finish` comes more than ten minutes after `--start`, the pending state is dropped and the command fails with `expired` — run `--start` again for a fresh challenge.
- **One JSON line per call.** Everything is on stdout as a single line; a caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"claudecode","error":"...","detail":"..."}`.
- **`--status` tells you where you stand.** It reports `logged_in`, `expires_at` and whether the stored token has already `expired`, whether a refresh token is stored (`has_refresh`), whether a login is mid-flight (`pending`), and whether the `claude_code` profile exists. Use it to decide whether a machine needs a login at all.
- **The code is validated, never logged.** Whitespace is trimmed, a `#state` fragment is stripped, and a rejected code returns an error suggesting a fresh `--start` — codes are single-use.

---

## Claude Code subscription vs. standard API-key usage

It is easy to confuse the two because they both run Claude models. Here is the difference:

| | **Claude Code provider** (this page) | **Anthropic provider** (`PROVIDER=anthropic`) |
|---|---|---|
| `PROVIDER` value | `claude_code` | `anthropic` |
| **Authentication** | OAuth session login (`scorpiox-claudecode-login`) | `ANTHROPIC_API_KEY` sent as `x-api-key` |
| **Billing** | Your Claude / Claude Code subscription allowance (5-hour + weekly windows) | Pay-per-token API usage on your key |
| **Token lifecycle** | Short-lived access token, renewed automatically from the stored refresh token | Static API key, nothing to refresh |
| **Endpoint** | `api.anthropic.com/v1/messages` with the Claude Code beta flags — or the SCORPIOX proxy | `ANTHROPIC_API_URL` (default `https://api.anthropic.com/v1/messages`), retargetable to any Anthropic-compatible endpoint |
| **Proxy modes** | Transparent proxy to Anthropic (`CLAUDE_CODE_PROXY_*`) | `official`, `custom`, `zai`, `antigravity` — each with its own key and URL |
| **URL key shape** | Domain only (`CLAUDE_CODE_API_URL`) | Full endpoint including the path (`ANTHROPIC_API_URL`) |
| **Usage signal** | `5h:` / `7d:` status-bar indicators, `/usage` popup, subscription windows | Token counts only — no rate-limit headers |
| **Wire identity** | Full Claude Code CLI emulation (user agent, client headers, beta flags) | Plain Messages API, no CLI-emulating headers |
| **Best for** | People who already pay for Claude and want the subscription to cover SCORPIOX CODE too | Pay-as-you-go access, custom endpoints, self-hosted gateways, proxy providers |

Rule of thumb: **you have a subscription, use `claude_code`. You have an API key or a custom endpoint, use `anthropic`.** They reach the same Anthropic Messages API with different credentials, different headers, and different billing — and only the Anthropic provider can be pointed at a non-Anthropic URL.

---

## Quick reference

| Goal | What to do |
|------|-----------|
| Sign in | `scorpiox-claudecode-login` |
| Re-authorize / replace the token | `scorpiox-claudecode-login --force` |
| Save a second account | `scorpiox-claudecode-login --name work` |
| Activate the provider | `/profile claude_code` (or `PROVIDER=claude_code` + `CLAUDE_CODE_TOKEN_SOURCE=local`) |
| Session-only switch | `/use claude-work` |
| Deactivate a profile | `/profile off` |
| Pick a model | `MODEL=opus` (or `sonnet` / `haiku` / a full ID), or `/model <name>` |
| Check subscription windows | `/usage` in-session, or `scorpiox-claudecode-usage` |
| Force a token refresh | `scorpiox-claudecode-refreshtoken --force` |
| Diagnose the token path | `scorpiox-claudecode-fetchtoken -config --verbose` |
| List served models | `scorpiox-claudecode-models` |
| Route through the proxy | `CLAUDE_CODE_PROXY_ENABLED=1` (+ `CLAUDE_CODE_PROXY_STRICT=1` to fail hard) |
| One-off run | `sx --provider claude_code -p "..."` |
| Login without a TTY | `scorpiox-claudecode-login --start` then `--finish <code>` |
| Pin a profile to one account | `CLAUDE_CODE_CREDENTIALS_FILE=~/.claude/accounts/<name>.json` in the profile |

---

## Gotchas

- **`CLAUDE_CODE_TOKEN_SOURCE` has to say `local` for a personal login.** The compiled-in default is `local`, but the sample `scorpiox-env.txt` shipped beside the binaries sets `tcp`, and a deployed copy of it outranks the built-in default. If SCORPIOX CODE cannot find a token, run `scorpiox-config --verbose` and check which tier `CLAUDE_CODE_TOKEN_SOURCE` came from — then set `local` in your profile, which beats every base file.
- **The login command and the provider are separate.** `scorpiox-claudecode-login` writes the credential file; it does not switch your session. Activate `claude_code` with `/profile` (or set `PROVIDER` yourself) before SCORPIOX CODE uses it.
- **No login means no token.** `CLAUDE_CODE_TOKEN_SOURCE=local` reads `~/.claude/.credentials.json`. If you have never logged in, that file does not exist and every request fails with a token error. Run the login first.
- **A profile created by a piped login never happens.** The interactive profile offer is skipped when stdin is not a terminal. In scripts and SCORPIO BOT flows, pass `--create-profile` with `--finish`, or write the profile by hand.
- **Named accounts need an explicit pointer.** `--name work` writes `~/.claude/accounts/work.json`, and nothing uses it until a profile sets `CLAUDE_CODE_CREDENTIALS_FILE` to that path. Refreshing a named account needs the same pointer passed to the refresh tool (`--credentials-file`), otherwise it refreshes the default credential file.
- **`CLAUDE_CODE_API_URL` is a domain, not a URL.** The path is appended automatically; writing a full `https://...` address there produces a malformed endpoint. If you need to name a full endpoint, that is the Anthropic provider's job.
- **Proxy mode is off by default.** Setting `CLAUDE_CODE_PROXY_URL` alone does nothing until `CLAUDE_CODE_PROXY_ENABLED=1`. And with strict mode off, a broken proxy silently degrades to direct Anthropic — if you *meant* to route through the proxy, set `CLAUDE_CODE_PROXY_STRICT=1` so a dead proxy is loud.
- **A project-level profile shadows the login-written one.** Profile lookup goes project (`.scorpiox/scorpiox-env/`) → user (`~/.claude/scorpiox-env/`) → global (next to the binaries), and a higher tier wins whole-file. A profile named `claude_code` inside your repo beats the one in your home directory — useful for pinning a project to a specific account, confusing if you did not mean it.
- **`/profile` persists; `/use` does not.** Use `/profile` to make an account your standing default and `/use` to hop to it for a single session without writing anything.
- **The authorization code is single-use and a credential.** Never share it, never paste it into a ticket or a chat. If the login stalls or the code is rejected, run the login again for a fresh challenge.
- **Refreshing is automatic; re-login is the fallback.** If `scorpiox-claudecode-refreshtoken --force` keeps failing — an expired refresh token, a changed account, a changed plan — do a full `scorpiox-claudecode-login --force` to re-bind.
- **Subscription windows are enforced upstream.** The 5-hour and weekly limits are Anthropic's, not SCORPIOX CODE's. You see utilization and reset times with `scorpiox-claudecode-usage` and the `5h:` / `7d:` status-bar indicators, never per-token metering.
- **The credential file is the crown jewel.** `~/.claude/.credentials.json` (and every file in `~/.claude/accounts/`) holds a live refresh token that can renew your session. The login writes it with restrictive permissions; keep it that way, keep it out of backups you share, and never commit it.

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Direct Anthropic API & Custom Endpoints](anthropic-provider.md) — the key-based path to the same models, and the API-key counterpart compared above.
- [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md) — the Google-side subscription path, which also runs Claude models through `google_claude`.
- [Token Usage Observability](usage-observability.md) — the `5h:` / `7d:` status-bar indicators and where the token counts show up.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim on-disk record of every request, with the bearer token redacted.
- [Scheduled Callbacks and Autonomous Agent Loops](callbacks.md) — pairing a subscription provider with long, self-driving sessions.
