# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

You have a **GitHub Copilot** subscription (a Copilot Individual, Pro, Business, or Enterprise seat — any account the official Copilot CLI can sign into) but you don't want to meter per-token against an API key. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Copilot provider** signs you in with the same OAuth device-code flow the official Copilot CLI uses and runs requests against the Copilot endpoint your subscription already pays for.

This page walks through the **device-code login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles (`/profile` and `/use`), how to inspect the stored token and your usage, how to drive a login without a TTY (**machine mode**), and how the Copilot subscription differs from standard API-key usage.

Docs for SCORPIOX CODE @ `2b0bffd`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Copilot Individual, Pro, Business, or Enterprise, or a Copilot seat you already pay for |
| **You don't have (or don't want) an OpenAI / Anthropic API key** | No pay-per-token billing — requests draw on your Copilot allowance |
| **Headless / over SSH** | The login is a device-code flow: open the link and enter the code on any device — no browser and no redirect needed on the box you run from |
| **Driven by SCORPIO BOT** | The login can be started and polled as separate commands (`--start` / `--poll`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling an OpenAI-compatible endpoint with an API key (`OPENAI_API_KEY`), use `PROVIDER=openai`. The two are different billing models — see [Copilot subscription vs. the OpenAI API-key provider](#copilot-subscription-vs-the-openai-api-key-provider) below.

> **Subscription, not API key.** The Copilot provider authenticates with GitHub OAuth and draws on your account's usage allowance (the same premium-interaction windows the Copilot CLI enforces). It does **not** accept an API key and does **not** bill per token.

---

## Step 1 — Sign in with the device-code flow

The default login is the **GitHub OAuth device-code flow**, the same one the official Copilot CLI performs. It is deliberately SSH- and headless-friendly: you open a link, sign in to your GitHub account on whatever device you already use, enter a one-time code, and SCORPIOX CODE picks the result up automatically. There is no localhost redirect to capture and no long URL to paste — nothing long-lived ever leaves the terminal, and no browser is required on the machine you're running from.

```bash
scorpiox-copilot-login
```

You'll see something like this:

```
scorpiox-copilot-login v...

Requesting device code...

┌─────────────────────────────────────────────────────┐
│  Visit: https://github.com/login/device              │
│  Enter code: XXXXXX-XXXXX                            │
└─────────────────────────────────────────────────────┘

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** (`https://github.com/login/device`) in your browser — on any device, wherever you're signed in to GitHub.
2. **Enter the one-time code** shown in your terminal and approve the authorization.
3. Come back to your terminal. SCORPIOX CODE is already polling in the background; the moment you approve, it exchanges the device code for a token and saves it.

The code is valid for **15 minutes** (GitHub's window), after which the flow expires — just run the login again for a fresh code.

SCORPIOX CODE then writes the token to `~/.copilot/.credentials.json` (mode `0600`) and offers to create a ready-to-use profile (see [Step 2](#step-2--turn-on-the-copilot-provider)).

### Overwriting existing credentials

A second login is refused if credentials already exist, so you never silently clobber an account. Force it when you mean to:

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
| `COPILOT_TOKEN_SOURCE` | choice | *(empty)* | Where to get the token: `local`, `remote`, `ssh`, `tcp`, or `config`. Use **`local`** for a login you ran with `scorpiox-copilot-login`. |
| `COPILOT_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.copilot/.credentials.json`. Point it at a named account (e.g. `~/.copilot/accounts/work.json`) to pin a profile to a specific login. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | An explicit official-CLI token (`gho_…`). When set, SCORPIOX CODE uses the official CLI identity directly and skips the token exchange. |
| `COPILOT_MODEL` | text | *(empty)* | Which model to run. Takes precedence over the generic `MODEL`. Accepts full Copilot model IDs or short aliases (see [Choosing a model](#choosing-a-model)). |
| `MODEL` | text | *(empty)* | Generic model key, used when `COPILOT_MODEL` is empty. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, `high`, or `off`. `REASONING_EFFORT` and `OPENAI_REASONING_EFFORT` are read as fallbacks. |
| `THINKING` | bool | *(empty)* | Enable extended thinking when the model supports it. |
| `TOOLS` | bool | *(empty)* | Include the tool-calling block in requests. |

> **For a personal subscription, you only need two keys:** `PROVIDER=copilot` and `COPILOT_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `remote` / `ssh` / `tcp` / `config` sources exist for shared or remote token setups and are not needed for a normal Copilot login.

### Where the token lives

On login, SCORPIOX CODE writes the GitHub token to `~/.copilot/.credentials.json` (mode `0600`): a `github_token` field (a `gho_…` access token), the `token_type`, the OAuth `scope`, and a `timestamp` of when it was issued. On every request, `COPILOT_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

You don't manage token lifetimes. SCORPIOX CODE keeps the credential fresh automatically:

- **Proactive refresh.** Ahead of each request, the provider checks whether the stored token needs renewing (with a 5-minute buffer ahead of expiry) and refreshes it before sending, so in-flight work never hits an expired credential.
- **Cached exchange.** For token types that require a short-lived exchange token, that token is cached under `~/.copilot/` and only re-minted when it is close to expiring — so repeated requests in a session don't churn the network.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries. A second failure after a refresh is reported as a permanent auth error.
- **Remote / SSH / TCP / config sources always fetch fresh.** In those modes the token is re-fetched on each request (the external endpoint is the source of truth and may have revoked an old token), so there is no local expiry to worry about.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-copilot-refreshtoken            # refresh if the token is expired
scorpiox-copilot-refreshtoken --force    # always refresh
scorpiox-copilot-refreshtoken --verbose  # also print token details
```

### Inspecting the stored token

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `COPILOT_TOKEN_SOURCE`, and `MODEL`, so you can confirm you're pointed at the account you expect before a long run. To check exactly which file a `local` token source reads, point `COPILOT_CREDENTIALS_FILE` at it explicitly and compare.

### Checking your subscription usage

The provider runs against the same usage allowance the Copilot CLI enforces. Check how full it is with:

```bash
scorpiox-copilot-usage            # human-readable summary
scorpiox-copilot-usage --json     # raw JSON
```

You'll see your Copilot plan and the premium-interaction snapshot — entitlement, remaining, credits used, percent remaining, and the quota reset date — so you know when a window resets before you kick off a long run.

You can also list the models your account can see:

```bash
scorpiox-copilot-models           # pretty-print model list
scorpiox-copilot-models --json    # raw JSON to stdout
```

---

## Multi-account: profiles and switching

### Save more than one login

Log in several accounts with `--name` (see [Named accounts](#named-accounts)). Each writes its own file:

```bash
scorpiox-copilot-login --name personal    # → ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work        # → ~/.copilot/accounts/work.json
```

### Bind each login to a profile

Give each account its own profile, pinning `COPILOT_CREDENTIALS_FILE` to the right file:

```
# ~/.claude/scorpiox-env/copilot-personal.txt
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/personal.json
MODEL=claude-sonnet-5

# ~/.claude/scorpiox-env/copilot-work.txt
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/work.json
MODEL=claude-sonnet-5
```

### Switch accounts and models in-session

```
/profile copilot-personal    # persist this account as the default
/use copilot-work            # hop for this session only
/model <id>                  # switch the model at runtime
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

---

## Choosing a model

`MODEL` (or `COPILOT_MODEL`) accepts either a full Copilot model ID or a short alias. The alias mapping at this commit is:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-5` (provider default) |
| `opus` | `claude-sonnet-5` |
| `sonnet` | `claude-sonnet-5` |
| `haiku` | `claude-sonnet-5` |
| any `claude-*` / `gpt-*` / `gemini-*` / `kimi-*` ID | passed through as-is |

You can pin a specific version per alias by setting `MODEL` to the full ID in your profile, or switch it at runtime with the `/model` command. Run `scorpiox-copilot-models` to see every ID your account can see and set one directly in `MODEL`.

> Note: the login-offered `copilot` profile ships with `MODEL=claude-sonnet-5` out of the box. Change it in the profile if you want a different model by default.

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open between "here's the code" and "approve on your phone" — a background agent, a CI step, or SCORPIO BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node (add `--name` for a named account). |
| `--start` | Begin the login: returns the `verify_url` and the one-time `user_code`, and keeps the flow state for 15 minutes. |
| `--poll` | One non-blocking check. Returns `pending` until you approve, then the tokens are saved. |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--poll`: also write the default `copilot` profile when the login completes. |

A typical machine-mode login:

```bash
# 1. Start — prints the link and the one-time code
scorpiox-copilot-login --start
#   {"ok":true,"provider":"copilot","flow":"device_code","verify_url":"https://github.com/login/device","user_code":"XXXXXX-XXXXX","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open that link on any device, sign in to GitHub, enter the code, approve.

# 3. Poll until approved (repeat until "state":"done")
scorpiox-copilot-login --poll
#   {"ok":true,"provider":"copilot","state":"pending","interval":5}
#   {"ok":true,"provider":"copilot","state":"done","logged_in":true,"path":"/home/you/.copilot/.credentials.json","profile":"copilot","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the flow state lives in `~/.claude/.login-pending/` (mode `0600`), so the two calls can be separate processes on the same node.
- **Fifteen-minute window.** If you don't approve within 15 minutes the pending state expires and `--poll` fails with `expired` — run `--start` again for a fresh code.
- **`--poll` is the finish step, not `--finish`.** Copilot is a device-code flow: it's `--start` then `--poll`. A `--finish` call is rejected because Copilot uses the device-code protocol, not paste-code.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"copilot","error":"...","detail":"..."}`.

---

## Copilot subscription vs. the OpenAI API-key provider

It's easy to confuse the two because they can run the same model families. Here's the difference:

| | **Copilot provider** (this page) | **OpenAI API-key provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | GitHub OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Draws on your GitHub Copilot subscription (premium-interaction windows) | Pay-per-token API usage against the key's account |
| **Endpoint** | GitHub Copilot's own endpoint | Any OpenAI-compatible `/v1/chat/completions` server (local or remote) |
| **Local / air-gapped** | No — requires a live GitHub Copilot subscription | Yes — can point at llama.cpp, vLLM, SGLang, or any self-hosted endpoint |

They can run the same underlying models, but the **Copilot provider bills against your existing subscription** while the **OpenAI API-key provider bills per token against your API key** (and, via `openai`, can point at any OpenAI-compatible server, local or remote). Pick the one that matches how you already pay for models. See [Using the OpenAI Provider](openai-provider.md).

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-copilot-login` writes the token file and the profile. You still need `PROVIDER=copilot` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file holds a live GitHub token.** `~/.copilot/.credentials.json` can renew your account session — don't commit it, don't share it, and don't loosen its permissions.
- **`COPILOT_TOKEN_SOURCE=local` reads a fixed default path.** To use a named account at runtime, set `COPILOT_CREDENTIALS_FILE` to that account's file — the profile you're given points at the default path, not at named accounts.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Copilot account your standing default and `/use` to hop to it for a single session without writing anything.
- **Machine mode is additive.** `--status` / `--start` / `--poll` / `--cancel` are only active when you pass one of those flags; an unflagged `scorpiox-copilot-login` is the interactive flow. `--finish` is not supported (Copilot is a device-code flow). Pending machine state expires after 15 minutes.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Using the OpenAI Provider](openai-provider.md)
- [Codex provider](codex-provider.md)
