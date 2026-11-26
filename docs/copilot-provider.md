# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

You have a **GitHub Copilot subscription** — a Copilot Pro, Pro+, or a Copilot Business seat on your GitHub account — and you would rather not buy a separate API key or meter every turn against per-token billing. You can use it directly in SCORPIOX CODE. The **Copilot provider** (`PROVIDER=copilot`) signs you in with the same **OAuth device-code login** the official GitHub Copilot CLI performs and sends every request to the Copilot endpoint your subscription already pays for.

This page walks through the device-code login, where the token is stored, how refresh works (and why there is nothing to manage), how to juggle several accounts with profiles (`/profile` and `/use`), how to drive a login without a TTY (**machine mode**), and how a Copilot subscription differs from standard API-key usage.

Docs for SCORPIOX CODE @ `3f5b938`.

> **The whole idea in one line:** run `scorpiox-copilot-login`, open the printed GitHub link, enter the one-time device code from any browser, and SCORPIOX CODE polls in the background and saves the `gho_` token. From then on every request draws on your Copilot allowance — no API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Pro, Pro+, or a Copilot Business seat you already pay for |
| **You don't have (or don't want) a Copilot API key** | Requests draw on your subscription's usage windows, not per-token billing |
| **Headless / over SSH** | The device-code flow is SSH-friendly: open a link and type a code on any device — no browser needed on the box |
| **Driven by SCORPIOX BOT** | The login splits into separate `--start` / `--poll` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |
| **Claude models through Copilot** | `claude-sonnet-5` (and other vendor models) run on the official-CLI identity |

If you are instead calling an OpenAI-compatible server with an **API key** (`OPENAI_API_KEY` and `OPENAI_BASE_URL`), that is a different provider: `PROVIDER=openai`, described in [Using the OpenAI Provider in SCORPIOX CODE](openai-provider.md). The two run the same models on different billing models — see [Copilot subscription vs. standard API-key usage](#copilot-subscription-vs-standard-api-key-usage) below.

> **Subscription, not API key.** The Copilot provider authenticates with OAuth and draws on your account's Copilot allowance (the 5-hour and weekly windows GitHub enforces). It does **not** accept an OpenAI API key and does **not** bill per token.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**. It is deliberately headless- and SSH-friendly: SCORPIOX CODE prints a link and a one-time code, you sign in from a browser on whatever device is already signed in to GitHub, then SCORPIOX CODE polls in the background and saves the token when you're authorized. No redirect server and no pasting.

```bash
scorpiox-copilot-login
```

You will see something like this:

```
scorpiox-copilot-login v...
Requesting device code...

┌─────────────────────────────────────────────────────┐
│  Visit: https://github.com/login/device              │
│  Enter code: XXXX-XXXX                               │
└─────────────────────────────────────────────────────┘

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** in a browser — on any device, wherever you are signed into GitHub.
2. **Sign in** to your account and **enter the one-time code**.
3. Come back to your terminal and wait — SCORPIOX CODE polls automatically (every few seconds, for up to 15 minutes) and, once you approve, saves the token to `~/.copilot/.credentials.json`.

A few things worth knowing about this flow:

- **The code expires in about 15 minutes.** If the wait times out, just re-run `scorpiox-copilot-login` and a fresh code is issued.
- **Never share the code.** It is a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up.

The login authenticates as the **official Copilot CLI**, which is what lets the subscription path run Claude models (including `claude-sonnet-5`) on every turn, including tool-result follow-ups. It saves two files:

- `~/.copilot/.credentials.json` — the `gho_` GitHub token plus its type, scope, and an issue timestamp (mode `0600`).
- `~/.copilot/config.json` — the same token in the shape the official Copilot CLI reads, so the two can interoperate.

### Re-running the login

The login refuses to overwrite an existing credential by default. Pass `--force` when you want to re-authorize deliberately — after an account change, a plan change, or when refresh has stopped working:

```bash
scorpiox-copilot-login --force
```

### VS Code identity (alternate)

By default the login uses the CLI client identity (`gho_`). If you specifically want the **VS Code** identity — a `ghu_` token, no `config.json` — pass `--vscode`:

```bash
scorpiox-copilot-login --vscode
```

The two identities differ in how the provider talks to the backend: the official-CLI `gho_` path skips the JWT exchange and uses the business host with the `copilot-developer-cli` headers; the VS Code `ghu_` path exchanges the token for a short-lived Copilot JWT first. Use the default (official CLI) unless you have a reason to prefer the VS Code path.

### More than one login: named accounts

`--name` saves each login as its own file instead of the default one:

```bash
scorpiox-copilot-login --name personal   # -> ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work       # -> ~/.copilot/accounts/work.json
```

Named accounts live in `~/.copilot/accounts/<name>.json`. A profile only uses one of these if `COPILOT_CREDENTIALS_FILE` points at it (see [Configuration keys](#configuration-keys)).

### The login offers to create the profile

After a successful interactive login, SCORPIOX CODE offers to write a ready-to-use profile:

```
Create 'copilot' config profile?
  Will create: /home/you/.claude/scorpiox-env/copilot.txt
  Contents:
    PROVIDER=copilot
    COPILOT_TOKEN_SOURCE=local
    MODEL=claude-sonnet-5

Create? [Y/n]
```

Say yes and you can activate it immediately with `/profile copilot`. Two details about this offer:

- **It only appears when the terminal is interactive.** Piping the login (or running it from a script) skips the prompt — use `--create-profile` in machine mode, or write the file by hand.
- **If the profile already exists it is left alone** unless you passed `--force`.

---

## Step 2 — Turn on the Copilot provider

The provider switches on with `PROVIDER=copilot`. For a personal subscription you also set `COPILOT_TOKEN_SOURCE=local`, so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
```

Activate it and SCORPIOX CODE connects:

```
/profile copilot      # activate the login-written profile and persist it
```

The login command and the provider are two separate things: the login writes the token file, the provider reads it. Running the login never switches your session — activate the profile (or set the keys yourself) before SCORPIOX CODE uses it.

You can also pick the provider for a single run without touching any file:

```bash
sx --provider copilot -p "explain this diff"
```

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `copilot` to activate this provider. |
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local`, `remote`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-copilot-login`. |
| `COPILOT_CREDENTIALS_FILE` | text | *(empty)* | Override the local credentials path. Defaults to `~/.copilot/.credentials.json` (falling back to `~/.copilot/config.json`). Point it at a named account (e.g. `~/.copilot/accounts/work.json`) to pin a profile to one login. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | An explicit `gho_` Copilot CLI token. When set, SCORPIOX CODE skips the JWT exchange entirely and uses the official-CLI identity on the business host. |
| `COPILOT_MODEL` | text | *(empty)* | Which model to run. Accepts short aliases or full model IDs (see [Choosing a model](#choosing-a-model)). Falls back to `MODEL`. |
| `COPILOT_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `COPILOT_TOKEN_SOURCE=remote`. |
| `COPILOT_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. `COPILOT_SSH_HOST` and `COPILOT_SSH_USER` are required in that mode. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, or `high` (also `off`/`none` to disable). Falls back to `REASONING_EFFORT`, then `OPENAI_REASONING_EFFORT`. |
| `THINKING` | bool | `0` | Enables thinking when set to `1`. |
| `COPILOT_SYSTEM_PROMPT` | text | *(empty)* | Optional custom system prompt for Copilot requests. |

> **For a personal subscription you only need two keys:** `PROVIDER=copilot` and `COPILOT_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `remote`, `ssh`, and `tcp` sources exist for shared or remote token setups and are not needed for a normal Copilot login.

#### Where the token lives

On login, SCORPIOX CODE writes the token to `~/.copilot/.credentials.json` in the same shape the official Copilot CLI uses — a `github_token` (`gho_...`), its `token_type`, the granted `scope`, and a `timestamp`:

```json
{
  "github_token": "gho_...",
  "token_type": "bearer",
  "scope": "read:user",
  "timestamp": "2026-11-24T02:00:00Z"
}
```

With `COPILOT_TOKEN_SOURCE=local`, every request reads that file (or the file `COPILOT_CREDENTIALS_FILE` points at). No API key is involved, and none is ever written. Named accounts live beside it in `~/.copilot/accounts/<name>.json`.

---

## Token persistence and automatic refresh

The `gho_` token is a long-lived GitHub token — unlike the Claude Code provider, it does not carry a short access token you must renew. SCORPIOX CODE handles the lifecycle on every request:

- **Official-CLI `gho_` path.** When the token is a `gho_` GitHub token, the provider uses it directly on the business host and does **not** mint a Copilot JWT. There is nothing to refresh — the token stays valid for the life of the grant.
- **VS Code `ghu_` path.** When the token is a `ghu_` token, SCORPIOX CODE exchanges it for a short-lived Copilot JWT and caches the result. The cached JWT is reused until it is within a 60-second margin of expiring, then re-exchanged.
- **Reactive recovery.** If a request still comes back `401` (expired or revoked token), the provider refreshes once and retries the same request immediately. A second `401` after a refresh is reported as a permanent authentication failure rather than retried forever.
- **Remote sources always fetch fresh.** In `remote`, `ssh`, or `tcp` modes the token is re-fetched on every request — the remote endpoint is the source of truth and may have rotated or revoked an old token at any time.

You can also force a refresh from the command line:

```bash
scorpiox-copilot-refreshtoken          # refresh if the cached JWT is expired
scorpiox-copilot-refreshtoken --force  # always re-exchange
scorpiox-copilot-refreshtoken --verbose
```

If requests keep coming back unauthorized, the fix is almost always to re-login:

```bash
scorpiox-copilot-login --force
```

### Check your usage

```bash
scorpiox-copilot-usage              # pretty summary
scorpiox-copilot-usage --json       # raw JSON
```

The summary reads the same quota endpoint the official Copilot CLI's `/usage` uses — your plan, your premium-interaction allowance, percent used, and the quota reset date. Inside SCORPIOX CODE, the in-session `/usage` command runs this same tool and titles the popup **Copilot Usage** while this provider is active.

---

## Multi-account: profiles and switching

Many people hold more than one GitHub account — personal and work, for example. The Copilot provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-copilot-login --name personal   # -> ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work       # -> ~/.copilot/accounts/work.json
```

### Bind each login to a profile

Make one profile per account and point `COPILOT_CREDENTIALS_FILE` at the right file, so the profile always talks to that account:

```
# ~/.claude/scorpiox-env/copilot-personal.txt
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/personal.json
MODEL=claude-sonnet-5
```

```
# ~/.claude/scorpiox-env/copilot-work.txt
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/work.json
MODEL=claude-sonnet-5
```

Profiles are looked up in the project tier, then your user tier, then the global tier — the highest tier wins whole-file, so a project-level profile of the same name shadows your personal one for sessions started in that repo.

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile copilot-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use    copilot-personal   # session-only — gone when the session ends
/profile                   # list available profiles and which is active
/profile off               # deactivate
```

Both commands reload the provider live, so the new account and model take effect immediately, and both revert automatically if the new profile cannot initialize — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

`MODEL` (or `COPILOT_MODEL`) accepts either a short alias or a full model ID. The aliases at this commit resolve as follows:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-5` (built-in default) |
| `opus` | `claude-sonnet-5` |
| `sonnet` | `claude-sonnet-5` |
| `haiku` | `claude-sonnet-5` |
| any full `claude-*`, `gpt-*`, `gemini-*`, or `kimi-*` ID | passed through as-is |

The short Claude aliases all map to `claude-sonnet-5`, the Claude model verified to work reliably with tool calls on the Copilot backend. Any explicit vendor model ID is sent unchanged.

You can list every model the endpoint advertises with:

```bash
scorpiox-copilot-models            # pretty model table
scorpiox-copilot-models --json     # raw JSON list
```

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open between "here is the link" and "enter the code" — a background agent, a CI step, or SCORPIOX BOT's provider flow. The login ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line (add `--name` for a named account). |
| `--start` | Begin the login: returns the `verify_url` and `user_code`, and keeps the device state for 15 minutes. |
| `--poll` | One non-blocking poll. Returns `pending` until the user authorizes, then `done`. |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--poll`: also write the `copilot` profile. |
| `--force` | Replace existing credentials. |
| `--name <account>` | Operate on a named account file instead of the default one. |

A typical machine-mode login:

```bash
# 1. Start — prints the URL and the one-time code to enter
scorpiox-copilot-login --start
# {"ok":true,"provider":"copilot","flow":"device_code","verify_url":"https://github.com/login/device","user_code":"XXXX-XXXX","interval":5,"expires_in":900,...}

# 2. Open that URL in any browser, sign in, and enter the code.

# 3. Poll until it reports done
scorpiox-copilot-login --poll --create-profile
# {"ok":true,"provider":"copilot","state":"done","logged_in":true,"path":"/home/you/.copilot/.credentials.json","profile":"copilot","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the device code lives in `~/.claude/.login-pending/copilot-<name|default>.json` (mode `0600`), so the calls can be separate processes on the same node.
- **Fifteen-minute window.** If `--poll` comes more than 15 minutes after `--start`, the pending state is dropped and the command fails with `expired` — run `--start` again for a fresh code.
- **One JSON line per call.** Everything is on stdout as a single line; a caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"copilot","error":"...","detail":"..."}`.
- **`--status` tells you where you stand.** It reports `logged_in`, `path`, whether a login is mid-flight (`pending`), whether the `copilot` profile exists, and whether named accounts are supported. Use it to decide whether a machine needs a login at all.

---

## Copilot subscription vs. standard API-key usage

It is easy to confuse the two because they both run the same models. Here is the difference:

| | **Copilot provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your Copilot subscription allowance (5-hour + weekly windows) | Pay-per-token API usage (or your own server) |
| **Token source** | `COPILOT_TOKEN_SOURCE` (local / remote / ssh / tcp) | `OPENAI_API_KEY` (+ optional `OPENAI_BASE_URL`) |
| **Endpoint** | `api.githubcopilot.com/chat/completions` (or the business host on the official-CLI path) | Any OpenAI-compatible `/v1/chat/completions` server |
| **Best for** | People who already pay for GitHub Copilot | Pay-as-you-go API access, local or custom endpoints |

Rule of thumb: **you have a GitHub Copilot subscription → `copilot`. You have an API key or a self-hosted OpenAI-compatible server → `openai`.**

---

## Gotchas

- **`COPILOT_TOKEN_SOURCE` has to say `local` for a personal login.** The compiled-in default is `local`, but a deployed `scorpiox-env.txt` beside the binaries can set a different source and outrank the built-in default. If SCORPIOX CODE cannot find a token, run `scorpiox-config --verbose` and check which tier `COPILOT_TOKEN_SOURCE` came from — then set `local` in your profile, which beats every base file.
- **The login command and the provider are separate.** `scorpiox-copilot-login` writes the credential file; it does not switch your session. Activate `copilot` with `/profile` (or set `PROVIDER` yourself) before SCORPIOX CODE uses it.
- **No login means no token.** `COPILOT_TOKEN_SOURCE=local` reads `~/.copilot/.credentials.json`. If you have never logged in, that file does not exist and every request fails with a token error. Run the login first.
- **A profile created by a piped login never happens.** The interactive profile offer is skipped when stdin is not a terminal. In scripts and SCORPIOX BOT flows, pass `--create-profile` with `--poll`, or write the profile by hand.
- **Named accounts need an explicit pointer.** `--name work` writes `~/.copilot/accounts/work.json`, and nothing uses it until a profile sets `COPILOT_CREDENTIALS_FILE` to that path.
- **`/profile` persists; `/use` does not.** Use `/profile` to make an account your standing default and `/use` to hop to it for a single session without writing anything.
- **A project-level profile shadows the login-written one.** Profile lookup goes project → user → global, and a higher tier wins whole-file. A profile named `copilot` inside your repo beats the one in your home directory — useful for pinning a project to a specific account, confusing if you did not mean it.
- **The device code is single-use and a credential.** Never share it, never paste it into a ticket or a chat. If the login stalls or the code expires, run the login again for a fresh code.
- **Re-login is the fallback.** If refresh keeps failing — an expired token, a changed account, a changed plan — do a full `scorpiox-copilot-login --force` to re-bind.
- **Subscription windows are enforced upstream.** The 5-hour and weekly limits are GitHub's, not SCORPIOX CODE's. You see utilization and reset times with `scorpiox-copilot-usage`, never per-token metering.
- **The credential file is the crown jewel.** `~/.copilot/.credentials.json` (and every file in `~/.copilot/accounts/`) holds a live GitHub token that can act as your account. The login writes it with restrictive permissions (mode `0600`); keep it that way, keep it out of backups you share, and never commit it.

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Using the OpenAI Provider in SCORPIOX CODE](openai-provider.md) — the key-based path to the same models, and the API-key counterpart compared above.
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md) — the closest sibling: another OAuth subscription provider with the same device-code shape.
