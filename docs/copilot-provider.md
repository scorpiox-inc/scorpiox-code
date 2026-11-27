# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

Have a GitHub Copilot subscription (Individual, Business, or Enterprise — the same one the official Copilot CLI signs into) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering a vendor API key, the **Copilot provider** signs you in with the same OAuth **device-code login** the official CLI uses and runs requests against the Copilot endpoint your subscription already pays for.

This page walks through the **device-code login**, how the token is stored and kept fresh automatically, how to switch accounts and models with profiles (`/profile` and `/use`), how to inspect tokens with `scorpiox-config`, and how direct Copilot subscription access differs from the standard OpenAI API-key provider.

Docs for SCORPIOX CODE @ `655c38f`.

> **The whole idea in one line:** run `scorpiox-copilot-login`, open the printed link and type the one-time code on any device — SCORPIOX CODE stores the token, writes a ready-to-use profile, and every subsequent request refreshes its own short-lived JWT from that stored credential. No API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Individual, Business, or Enterprise seat you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No per-token API billing — requests draw on your Copilot plan allowance |
| **You want Claude / GPT / Gemini models through GitHub's backend** | Claude Sonnet, GPT, and Gemini model IDs all run over one Copilot sign-in |
| **Headless or over SSH** | Open a link and type a code on any device; no browser is needed on the box you are running from |
| **Driven by SCORPIOX BOT or CI** | The login splits into separate `--start` / `--poll` / `--cancel` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling an OpenAI-compatible server with an API key (`OPENAI_API_KEY` and `OPENAI_BASE_URL`), use `PROVIDER=openai` — the [OpenAI provider](openai-provider.md). The two are different billing models for overlapping models. See [Copilot provider vs. the OpenAI API-key provider](#copilot-provider-vs-the-openai-api-key-provider) below.

> **Subscription, not API key.** The Copilot provider authenticates with OAuth and draws on your account's plan allowance (the monthly / premium-interaction quota your Copilot seat carries). It does **not** accept an `OPENAI_API_KEY`, and it does **not** bill per token.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**. It is deliberately headless- and SSH-friendly: SCORPIOX CODE prints a link and a one-time code, you sign in from a browser on whatever device is already signed in, then SCORPIOX CODE polls in the background and saves the token when you are authorized. No redirect server, no pasting.

You can run the dedicated login binary directly, or through the `scorpiox-login` dispatcher:

```bash
scorpiox-copilot-login          # device-code login (default)
scorpiox-login copilot          # same thing, via the provider dispatcher
```

You will see something like this:

```
scorpiox-copilot-login v...

Requesting device code...

┌─────────────────────────────────────────────────────┐
│  Visit: https://github.com/login/device             │
│  Enter code: ABCD-EFGH                              │
└─────────────────────────────────────────────────────┘

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** — `https://github.com/login/device` — in a browser, on any device where you are signed in to GitHub.
2. **Enter the one-time code** and approve the request.
3. Come back to your terminal and wait — SCORPIOX CODE polls automatically (every few seconds, for up to 15 minutes) and, on success, saves the token to `~/.copilot/.credentials.json`.

A few things worth knowing about this flow:

- **The code expires in about 15 minutes.** If the wait times out, just re-run `scorpiox-copilot-login` and a fresh code is issued.
- **Never share the code.** It is a credential. Anyone who has the code (and is watching your terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up — this is the main reason the device-code flow is the default.

On success, SCORPIOX CODE offers to create a config profile:

```
Create 'copilot' config profile?
  Will create: ~/.claude/scorpiox-env/copilot.txt
    PROVIDER=copilot
    COPILOT_TOKEN_SOURCE=local
    MODEL=claude-sonnet-5
```

Accept and the profile is written, ready to use with `/profile copilot`.

### Machine mode (no TTY)

For SCORPIOX BOT or CI pipelines that cannot hold an open TTY, the login splits into discrete JSON-mode commands:

```bash
scorpiox-copilot-login --status [--name N]
scorpiox-copilot-login --start  [--name N] [--force] [--create-profile]
scorpiox-copilot-login --poll   [--name N] [--create-profile]
scorpiox-copilot-login --cancel [--name N]
```

Each emits a single JSON object on stdout. `--start` prints the verification URL and code; `--poll` checks once (not a blocking loop) and reports the current state. Poll in a loop until the result is `authorized`, then optionally pass `--create-profile` to write the config profile.

---

## Token persistence and automatic refresh

The token from the device-code login is a GitHub OAuth token (a `ghu_` or `gho_` string). SCORPIOX CODE handles the rest on **every request**:

- **JWT exchange.** For VS Code-style `ghu_` tokens, SCORPIOX CODE exchanges the token for a short-lived Copilot JWT via the GitHub API before each request. The JWT is cached in `~/.copilot/.copilot-token-cache.json` and only re-exchanged when it is about to expire.
- **Proactive refresh.** When the cached JWT is within a 5-minute buffer of its expiry, the provider refreshes it before sending, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries the request. A second 401 after a refresh is reported as a permanent auth failure.
- **Official CLI path.** When the token is a `gho_` token (the official Copilot CLI), SCORPIOX CODE skips the JWT exchange entirely and talks directly to `api.business.githubcopilot.com` with the `copilot-developer-cli` integration headers.

You can force a refresh manually:

```bash
scorpiox-copilot-refreshtoken            # refresh if the token is expired
scorpiox-copilot-refreshtoken --force    # always refresh
scorpiox-copilot-refreshtoken --verbose  # show old/new tokens
```

As long as the credentials file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
# 1. Refresh the token in place
scorpiox-copilot-refreshtoken --force

# 2. If that keeps failing, re-bind with a full login
scorpiox-copilot-login --force
```

### Token sources

The provider reads the token through `scorpiox-copilot-fetchtoken`, which auto-detects the credential format from a list of well-known locations:

```
~/.copilot/.credentials.json        # from scorpiox-copilot-login (default)
~/.copilot/config.json              # official Copilot CLI
~/.config/github-copilot/hosts.json # VS Code
~/.config/github-copilot/apps.json  # VS Code
```

You can also point the provider at a remote or SSH token source for shared setups:

| `COPILOT_TOKEN_SOURCE` | Behaviour |
|------------------------|-----------|
| `local` (default) | Read from the credential files above |
| `config` | Resolve via `scorpiox-env.txt` (respects `ACTIVE_PROFILE`) |
| `remote` | Fetch from an HTTP endpoint (`COPILOT_REMOTE_URL`) |
| `ssh` | Fetch from a remote machine over SSH (`COPILOT_SSH_*`) |
| `tcp` | Fetch over a raw TCP socket (`TCP_HOST` / `TCP_PORT`) |

### Inspecting usage instead of per-token metering

Because you are on a subscription, there is no per-token metering. Use `scorpiox-copilot-usage` to see your plan, quota, and how much of the premium-interaction allowance you have consumed:

```bash
scorpiox-copilot-usage         # human-readable summary (plan, premium interactions, reset date)
scorpiox-copilot-usage --json # raw JSON to stdout
```

---

## Multi-account and profile switching

### Store more than one login

Log in to several GitHub Copilot accounts and give each a name:

```bash
scorpiox-copilot-login --name work
scorpiox-copilot-login --name personal
```

Each writes its own file under `~/.copilot/accounts/`. Create one profile per account, each pointing `COPILOT_CREDENTIALS_FILE` at the matching file.

### Switch accounts and models in-session

Switching is just switching profiles — no re-login, no restart:

```
/profile copilot        # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot            # session-only — gone when the session ends
/profile               # list available profiles and which is active
/profile off           # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what is actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

Set `COPILOT_MODEL` (or `MODEL`) in your profile. The default is `claude-sonnet-5`. The endpoint accepts any model ID your Copilot plan is entitled to — Claude, GPT, and Gemini families are all available through the same sign-in.

List every model your subscription advertises with:

```bash
scorpiox-copilot-models             # pretty-printed model list
scorpiox-copilot-models --json     # raw JSON to stdout
```

Switch models at runtime with the `/model` command, or set `COPILOT_MODEL` in the profile.

> **Reasoning effort.** For models that support it, set `COPILOT_REASONING_EFFORT` to `low`, `medium`, or `high` to control how much compute the model spends per response. Empty (the default) lets the backend choose.

---

## The keys

Everything is a `KEY=VALUE` pair in the [configuration cascade](scorpiox-env.md). For a personal Copilot subscription you only need two lines:

```
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
```

That is the whole thing. Add `COPILOT_CREDENTIALS_FILE` when you use a named account, and `COPILOT_GITHUB_TOKEN` when you are using an official CLI `gho_` token.

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `copilot` to activate this provider. |
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local` (default), `config`, `remote`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-copilot-login`. |
| `COPILOT_MODEL` | text | `claude-sonnet-5` | Which model to run. Accepts any model ID your Copilot plan allows. |
| `COPILOT_CREDENTIALS_FILE` | text | *(empty)* | Override the local credentials path. Defaults to `~/.copilot/.credentials.json`. Point it at a named account (e.g. `~/.copilot/accounts/work.json`) to pin a profile to a specific login. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | Official Copilot CLI token (`gho_...`). When set, skips the JWT exchange and uses `api.business.githubcopilot.com` directly. |
| `COPILOT_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `COPILOT_TOKEN_SOURCE=remote`. |
| `COPILOT_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=ssh`. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=tcp`. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, or `high`. Empty lets the backend choose. |
| `COPILOT_SYSTEM_PROMPT` | text | *(empty)* | Override the system prompt for Copilot requests. |

---

## Copilot provider vs. the OpenAI API-key provider

It is easy to confuse the two because they both run Claude, GPT, and Gemini models. Here is the difference:

| | **Copilot provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your GitHub Copilot subscription allowance (premium-interaction quota) | Pay-per-token API usage (or your own server) |
| **Token source** | `COPILOT_TOKEN_SOURCE` (local / remote / ssh / tcp) | `OPENAI_API_KEY` (+ optional `OPENAI_BASE_URL`) |
| **Endpoint** | `api.githubcopilot.com` or `api.business.githubcopilot.com` | Any OpenAI-compatible `/v1/chat/completions` server |
| **Best for** | People who already pay for GitHub Copilot | Pay-as-you-go API access, local or custom endpoints |

Rule of thumb: **you have a GitHub Copilot subscription → `copilot`. You have an API key or a self-hosted OpenAI-compatible server → `openai`.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-copilot-login` writes the token file and offers a profile. You still need `PROVIDER=copilot` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file holds a live token.** `~/.copilot/.credentials.json` can be used to authenticate as your GitHub account — do not commit it, do not share it, and do not loosen its permissions. It is written `0600` by default.
- **Named accounts live in `~/.copilot/accounts/`, not the default file.** `--name work` writes `~/.copilot/accounts/work.json`. A profile only uses it if `COPILOT_CREDENTIALS_FILE` points there.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Copilot account your standing default and `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-copilot-refreshtoken --force` keeps failing (expired token, account changed, plan changed), do a full `scorpiox-copilot-login --force` to re-bind.
- **Usage is quota-based, not per-token.** There is no token counter to read; use `scorpiox-copilot-usage` to see how much of your premium-interaction allowance you have consumed and when it resets.
- **`gho_` vs `ghu_` tokens behave differently.** A `gho_` token (official Copilot CLI) talks directly to the business endpoint with no JWT exchange. A `ghu_` token (VS Code) goes through a JWT exchange on every refresh. If you pass a `gho_` token via `COPILOT_GITHUB_TOKEN`, the provider detects it and uses the CLI path automatically.
