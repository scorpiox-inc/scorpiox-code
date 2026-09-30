# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

You have a **GitHub Copilot** subscription (Free, Pro, or Business — any account the official Copilot CLI can sign into) but you don't want to meter per-token against a separate `OPENAI_API_KEY` or juggle a paid API key just to run models you already pay for. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Copilot provider** signs you in with the same OAuth **device-code flow** the official Copilot CLI uses and runs requests against the Copilot endpoint your subscription already pays for.

This page walks through the **device-code login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles (`/profile` and `/use`), how to inspect your models and quota, how to drive a login without a TTY (**machine mode**), and how a Copilot subscription differs from standard OpenAI API-key usage.

Docs for SCORPIOX CODE @ `13253cf`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Free, Pro, or Business — or a Copilot seat you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No pay-per-token billing — requests draw on your Copilot plan |
| **Headless / over SSH** | The login is a device-code flow: open the link and enter the code on any device — no browser and no redirect needed on the box you run from |
| **Driven by SCORPIO BOT** | The login can be started and polled as separate commands (`--start` / `--poll`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the OpenAI API with an API key (`OPENAI_API_KEY`), use `PROVIDER=openai` (or an OpenAI-compatible endpoint). The two are different billing models for overlapping families of models — see [Copilot subscription vs. the OpenAI API-key provider](#copilot-subscription-vs-the-openai-api-key-provider) below.

> **Subscription, not API key.** The Copilot provider authenticates with OAuth and draws on your account's included usage (the same premium-interaction quota and premium-model access the official Copilot CLI enforces). It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**, the same one the official Copilot CLI performs. It is deliberately SSH- and headless-friendly: you open a GitHub link, sign in to your account on whatever device you already use, enter a one-time code, and SCORPIOX CODE picks the result up automatically. There is no localhost redirect to capture and no long URL to paste — nothing long-lived ever leaves the terminal, and no browser is required on the machine you're running from.

```bash
scorpiox-copilot-login
```

You'll see something like this:

```
scorpiox-copilot-login v...

Requesting device code...

┌─────────────────────────────────────────────────────┐
│  Visit: https://github.com/login/device             │
│  Enter code: XXXX-XXXX                              │
└─────────────────────────────────────────────────────┘

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** in your browser — on any device, wherever you're signed into GitHub.
2. **Sign in** to your GitHub account and **enter the one-time code** shown in your terminal.
3. Come back to your terminal. SCORPIOX CODE is already polling in the background; the moment you approve, it receives the access token and saves it.

SCORPIOX CODE then writes the credential to `~/.copilot/.credentials.json` (mode `0600`) and offers to create a ready-to-use profile (see [Step 2](#step-2--turn-on-the-copilot-provider)). A few things worth knowing about this flow:

- **The code is one-time and expires in 15 minutes.** If the prompt times out before you approve, re-run `scorpiox-copilot-login` for a fresh code.
- **Never share the code.** It is a credential. Anyone who has the code while you're waiting can bind the login to their own account.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: /home/you/.copilot/.credentials.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-copilot-login --force
```

### Named accounts

By default the login writes to `~/.copilot/.credentials.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-copilot-login --name personal     # → ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work         # → ~/.copilot/accounts/work.json
```

Names are limited to letters, digits, `-`, `_`, `.` and at most 64 characters.

### Creating the `copilot` profile

At the end of a successful login, SCORPIOX CODE offers to write a ready-made profile for you (the prompt is skipped when stdin is not a TTY):

```
Create 'copilot' config profile?
  Will create: /home/you/.claude/scorpiox-env/copilot.txt
  Contents:
    PROVIDER=copilot
    COPILOT_TOKEN_SOURCE=local
    MODEL=claude-sonnet-5

Create? [Y/n]
```

If you say yes, the profile is written to `~/.claude/scorpiox-env/copilot.txt` and you can activate it with `/profile copilot`. You can also create the file by hand:

```
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
MODEL=claude-sonnet-5
```

---

## Step 2 — Turn on the Copilot provider

The provider switches on with `PROVIDER=copilot`. For a personal subscription you also set `COPILOT_TOKEN_SOURCE=local` so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
```

Activate the profile (or set those keys in any cascade tier, or as OS environment variables) and SCORPIOX CODE connects.

```
/profile copilot      # activate this profile and persist it
/use copilot          # activate for this session only (no file change)
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(empty)* | Set to `copilot` to activate this provider. |
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where to get the token: `local`, `remote`, `ssh`, `tcp`, or `config`. Use **`local`** for a Copilot login you ran with `scorpiox-copilot-login`. |
| `COPILOT_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.copilot/.credentials.json`. Point it at a named account (e.g. `~/.copilot/accounts/work.json`) to pin a profile to a specific login. |
| `COPILOT_GITHUB_TOKEN` | secret | *(empty)* | A Copilot access token to use directly (skips reading the credential file). Convenient for injecting a token from a secret manager. |
| `MODEL` / `COPILOT_MODEL` | text | *(empty)* | Which model to run. `COPILOT_MODEL` takes precedence over `MODEL`. Accepts Copilot model IDs or short aliases (see [Choosing a model](#choosing-a-model)). |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, `high`, or `off`. `REASONING_EFFORT` and `OPENAI_REASONING_EFFORT` are read as fallbacks. |
| `COPILOT_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `COPILOT_TOKEN_SOURCE=remote`. |
| `COPILOT_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |

> **For a personal subscription, you only need two keys:** `PROVIDER=copilot` and `COPILOT_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `ssh` / `remote` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Copilot login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth credential to `~/.copilot/.credentials.json` (mode `0600`) with a `github_token` field (the `gho_...` access token), the `token_type` and `scope`, and a `timestamp`. On every request, `COPILOT_TOKEN_SOURCE=local` reads from that file (or from `COPILOT_CREDENTIALS_FILE` if set). No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The Copilot access token is long-lived by design, but the provider still keeps things tidy for you:

- **No refresh to manage.** The `gho_` access token from the device-code flow does not carry the short-lived JWT expiry that VS Code's path uses, so there is no token rotation to worry about for a normal personal login.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider re-fetches the token and retries once. A second failure after a re-fetch is reported as a permanent auth error.
- **Remote / SSH / TCP sources always fetch fresh.** In `remote`, `ssh`, or `tcp` modes the token is re-fetched on every request (the remote endpoint is the source of truth and may have revoked an old token), so there is no local expiry to worry about.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-copilot-refreshtoken            # refresh if the token is expired
scorpiox-copilot-refreshtoken --force    # always refresh
scorpiox-copilot-refreshtoken --verbose  # also print token details
```

### Inspecting the resolved configuration

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, …) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `COPILOT_TOKEN_SOURCE`, and `MODEL`, so you can confirm you're pointed at the account you expect before a long run.

### Checking your subscription usage

The provider runs against the same quota the official Copilot CLI enforces. Check how full it is with:

```bash
scorpiox-copilot-usage            # human-readable summary
scorpiox-copilot-usage --json     # raw JSON
```

You'll see your plan (`copilot_plan`), the premium-interaction quota (remaining / credits used / percent remaining) and its reset date, so you know when a window resets before you kick off a long run.

### Listing the models you can use

The full set of model IDs your account is entitled to:

```bash
scorpiox-copilot-models            # pretty-printed table of id / name / vendor
scorpiox-copilot-models --json     # raw JSON
```

---

## Switching between accounts

### Save more than one login

```bash
scorpiox-copilot-login --name personal     # → ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work         # → ~/.copilot/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account. Point `COPILOT_CREDENTIALS_FILE` at the right file so the profile always uses that account:

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
MODEL=gpt-5.1
```

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile copilot-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot-personal      # session-only — gone when the session ends
/profile                   # open the profile picker (or list if no picker is installed)
/profile off               # deactivate the profile
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

---

## Choosing a model

`MODEL` (or `COPILOT_MODEL`) accepts a Copilot model ID or a short alias. At this commit the aliases all resolve to the one Claude id that is proven to work with tools on the official CLI path:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-5` (provider default) |
| `opus` | `claude-sonnet-5` |
| `sonnet` | `claude-sonnet-5` |
| `haiku` | `claude-sonnet-5` |
| any full `claude-*`, `gpt-*`, `gemini-*`, or `kimi-*` ID | passed through as-is |

The full list your account is entitled to is always available from `scorpiox-copilot-models`. You can pin a specific version by setting `MODEL` to the full ID in your profile, or switch it at runtime with the `/model` command.

> Note: the login-offered `copilot` profile ships with `MODEL=claude-sonnet-5` out of the box. Change it to any model ID from `scorpiox-copilot-models` depending on the model you want by default.

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open between "here's the code" and "approve on your phone" — a background agent, a CI step, or SCORPIO BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node (add `--name` for a named account). |
| `--start` | Begin the login: returns the `verify_url` and the one-time `user_code`, and keeps the flow state for 15 minutes. |
| `--poll` | One non-blocking check. Returns `pending` until you approve, then the token is saved. |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--poll`: also write the default `copilot` profile when the login completes. |

A typical machine-mode login:

```bash
# 1. Start — prints the verify URL and one-time code
scorpiox-copilot-login --start
# {"ok":true,"provider":"copilot","flow":"device_code","verify_url":"https://github.com/login/device","user_code":"XXXX-XXXX","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open verify_url on any device, sign in to GitHub, enter the code.
# 3. Poll until approved (repeat until the token is saved)
scorpiox-copilot-login --poll
#   {"ok":true,"provider":"copilot","state":"pending","interval":5}
```

A few things worth knowing about machine mode:

- **`--poll` is the finish step, not `--finish`.** Copilot is a device-code flow: it's `--start` then `--poll`. A `--finish` call is rejected because Copilot uses the device-code protocol, not paste-code.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"copilot","error":"...","detail":"..."}`.

---

## Copilot subscription vs. the OpenAI API-key provider

It's easy to confuse the two because they can both run the same families of models. Here's the difference:

| | **Copilot provider** (this page) | **OpenAI API-key provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Draws on your GitHub Copilot plan (included premium usage) | Pay-per-token OpenAI API usage |
| **What you need** | A GitHub Copilot Free / Pro / Business account | An OpenAI API key and a platform account |
| **Token lifecycle** | Long-lived OAuth access token, re-fetched on 401 | Static API key, no refresh |
| **Endpoint** | `api.githubcopilot.com` / `api.business.githubcopilot.com` chat completions | Any OpenAI-compatible `/chat/completions` |

They can reach the same underlying models, but the **Copilot provider bills against your existing subscription** while the **OpenAI API-key provider bills per token against your API key** (and, via `openai`, can point at any OpenAI-compatible server, local or remote). Pick the one that matches how you already pay for models. See [Using the OpenAI Provider](openai-provider.md).

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using the OpenAI Provider](openai-provider.md)
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md)
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md)
