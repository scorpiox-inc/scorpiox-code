# Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE

You have an **OpenAI Codex / ChatGPT** subscription (a ChatGPT Plus, Pro, or Business plan, or any account the Codex CLI can sign into) but you don't want to meter per-token against an `OPENAI_API_KEY`. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Codex provider** signs you in with the same OAuth device-code flow the official Codex CLI uses and runs requests against the Codex endpoint your subscription already pays for.

This page walks through the **device-code login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles (`/profile` and `/use`), how to inspect and refresh the stored token, how to drive a login without a TTY (**machine mode**), and how the Codex subscription differs from standard OpenAI API-key usage.

Docs for SCORPIOX CODE @ `13253cf`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a ChatGPT / Codex subscription** | ChatGPT Plus, Pro, or Business, or a Codex seat you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The login is a device-code flow: open the link and enter the code on any device — no browser and no redirect needed on the box you run from |
| **Driven by SCORPIO BOT** | The login can be started and polled as separate commands (`--start` / `--poll`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the OpenAI API with an API key (`OPENAI_API_KEY`), use `PROVIDER=openai` (or an OpenAI-compatible endpoint). The two are different billing models for the same family of models — see [Codex subscription vs. the OpenAI API-key provider](#codex-subscription-vs-the-openai-api-key-provider) below.

> **Subscription, not API key.** The Codex provider authenticates with OAuth and draws on your account's usage allowance (the same 5-hour session and weekly windows the Codex CLI enforces). It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**, the same one the official Codex CLI performs. It is deliberately SSH- and headless-friendly: you open a link, sign in to your ChatGPT/Codex account on whatever device you already use, enter a one-time code, and SCORPIOX CODE picks the result up automatically. There is no localhost redirect to capture and no long URL to paste — nothing long-lived ever leaves the terminal, and no browser is required on the machine you're running from.

```bash
scorpiox-codex-login
```

You'll see something like this:

```
scorpiox-codex-login v...

Requesting device code...

Follow these steps to sign in with ChatGPT:

1. Open this link in your browser and sign in to your account
   https://auth.openai.com/codex/device

2. Enter this one-time code (expires in 15 minutes)
   XXXX-XXXX

Device codes are a common phishing target. Never share this code.

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** in your browser — on any device, wherever you're signed into ChatGPT/Codex.
2. **Sign in** to your account and **enter the one-time code** shown in your terminal.
3. Come back to your terminal. SCORPIOX CODE is already polling in the background; the moment you approve, it exchanges the authorization code for tokens and saves them.

SCORPIOX CODE then writes the credentials to `~/.codex/auth.json` (mode `0600`) and offers to create a ready-to-use profile (see [Step 2](#step-2--turn-on-the-codex-provider)). A few things worth knowing about this flow:

- **The code is one-time and expires in 15 minutes.** If the prompt times out before you approve, re-run `scorpiox-codex-login` for a fresh code.
- **Never share the code.** It is a credential. The terminal prints a warning for good reason: anyone who has the code while you're waiting can bind the login to their own account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal you're polling picks it up automatically.

### Browser fallback

If the device-code endpoint is unavailable (some Codex server configurations disable it), the command tells you and offers the **browser redirect/paste** flow:

```bash
scorpiox-codex-login --browser
```

This opens an authorization URL; when the browser is on the same machine the `localhost:1455/auth/callback` redirect is captured automatically, otherwise you paste the resulting redirect URL (or just the code) back into the terminal.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: /home/you/.codex/auth.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-codex-login --force
```

### Named accounts

By default the login writes to `~/.codex/auth.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-codex-login --name personal     # → ~/.codex/accounts/personal.json
scorpiox-codex-login --name work         # → ~/.codex/accounts/work.json
```

Names are limited to letters, digits, `-`, `_`, `.` and at most 64 characters.

### Creating the `codex` profile

At the end of a successful login, SCORPIOX CODE offers to write a ready-made profile for you (the prompt is skipped when stdin is not a TTY):

```
Create 'codex' config profile?
  Will create: /home/you/.claude/scorpiox-env/codex.txt
  Contents:
    PROVIDER=codex
    CODEX_TOKEN_SOURCE=local
    MODEL=gpt-5.5

Create? [Y/n]
```

If you say yes, the profile is written to `~/.claude/scorpiox-env/codex.txt` and you can activate it with `/profile codex`. You can also create the file by hand:

```
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
MODEL=gpt-5.5
```

---

## Step 2 — Turn on the Codex provider

The provider switches on with `PROVIDER=codex`. For a personal subscription you also set `CODEX_TOKEN_SOURCE=local` so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
```

Activate the profile (or set those keys in any cascade tier, or as OS environment variables) and SCORPIOX CODE connects.

```
/profile codex      # activate this profile and persist it
/use codex          # activate for this session only (no file change)
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(empty)* | Set to `codex` to activate this provider. |
| `CODEX_TOKEN_SOURCE` | choice | *(empty)* | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a Codex login you ran with `scorpiox-codex-login`. |
| `CODEX_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.codex/auth.json`. Point it at a named account (e.g. `~/.codex/accounts/work.json`) to pin a profile to a specific login. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts full Codex model IDs or short aliases (see [Choosing a model](#choosing-a-model)). |
| `CODEX_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, `high`, `max`, or `off`. `REASONING_EFFORT` and `OPENAI_REASONING_EFFORT` are read as fallbacks. |
| `CODEX_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CODEX_TOKEN_SOURCE=http`. |
| `CODEX_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |

> **For a personal subscription, you only need two keys:** `PROVIDER=codex` and `CODEX_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `ssh` / `http` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Codex login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth tokens to `~/.codex/auth.json` (mode `0600`) in the same shape the official Codex CLI uses: an `OPENAI_API_KEY` field (always `null`), a `tokens` section with `id_token`, `access_token`, `refresh_token`, and the account ID, plus a `last_refresh` timestamp. On every request, `CODEX_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** When the stored access token is close to expiring (a 5-minute buffer ahead of its expiry), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries. A second failure after a refresh is reported as a permanent auth error.
- **Remote / TCP sources always fetch fresh.** In `http`, `ssh`, or `tcp` modes the token is re-fetched on every request (the remote endpoint is the source of truth and may have revoked an old token), so there is no local expiry to worry about.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-codex-refreshtoken            # refresh if the token is expired
scorpiox-codex-refreshtoken --force    # always refresh
scorpiox-codex-refreshtoken --verbose  # also print the old/new tokens
```

### Inspecting the stored token

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, CODEX_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `CODEX_TOKEN_SOURCE`, and `MODEL`, so you can confirm you're pointed at the account you expect before a long run. To check exactly which file a `local` token source reads, point `CODEX_CREDENTIALS_FILE` at it explicitly and compare.

### Checking your subscription usage

The provider runs against the same usage windows the Codex CLI enforces. Check how full they are with:

```bash
scorpiox-codex-usage            # human-readable summary
scorpiox-codex-usage --json     # raw JSON
```

You'll see your plan type and the per-window utilization and reset times — typically the 5-hour **session** window and the **weekly** window, plus any credits balance — so you know when a window resets before you kick off a long run.

---

## Multi-account: profiles and switching

Many people have more than one OpenAI account (personal + work, for example). The Codex provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-codex-login --name personal     # → ~/.codex/accounts/personal.json
scorpiox-codex-login --name work         # → ~/.codex/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account. Point `CODEX_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/codex-personal.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/personal.json
MODEL=gpt-5.6-terra
```

```
# ~/.claude/scorpiox-env/codex-work.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/work.json
MODEL=gpt-5.4-mini
```

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile codex-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use codex-personal      # session-only — gone when the session ends
/profile                 # open the profile picker (or list if no picker is installed)
/profile off             # deactivate the profile
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

---

## Choosing a model

`MODEL` accepts either a full Codex model ID or a short alias. The aliases at this commit are:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `gpt-5.6-terra` (provider default) |
| `opus` | `gpt-5.6-terra` |
| `sonnet` | `gpt-5.6-luna` |
| `haiku` | `gpt-5.4-mini` |
| any full `gpt-*` ID or `codex-*` ID | passed through as-is |

The full IDs the provider knows about at this commit are `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.5`, `gpt-5.4-mini`, and `codex-auto-review`. You can pin a specific version by setting `MODEL` to the full ID in your profile, or switch it at runtime with the `/model` command.

> Note: the login-offered `codex` profile ships with `MODEL=gpt-5.5` out of the box. Change it to `gpt-5.6-terra` / `gpt-5.6-luna` / `gpt-5.4-mini` (or leave an alias such as `opus` / `sonnet` / `haiku`) depending on the model you want by default.

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open between "here's the code" and "approve on your phone" — a background agent, a CI step, or SCORPIO BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node (add `--name` for a named account). |
| `--start` | Begin the login: returns the `verify_url` and the one-time `user_code`, and keeps the flow state for 15 minutes. |
| `--poll` | One non-blocking check. Returns `pending` until you approve, then the tokens are saved. |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--poll`: also write the default `codex` profile when the login completes. |

A typical machine-mode login:

```bash
# 1. Start — prints the verify URL and one-time code
scorpiox-codex-login --start
# {"ok":true,"provider":"codex","flow":"device_code","verify_url":"https://auth.openai.com/codex/device","user_code":"XXXX-XXXX","interval":5}

# 2. Open verify_url on any device, sign in, enter the code.
# 3. Poll until approved (repeat until "state":"done")
scorpiox-codex-login --poll
#   {"ok":true,"provider":"codex","state":"pending","interval":5}
#   {"ok":true,"provider":"codex","state":"done","logged_in":true,"path":"/home/you/.codex/auth.json","profile":"codex","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the flow state lives in `~/.claude/.login-pending/` (mode `0600`), so the two calls can be separate processes on the same node.
- **Fifteen-minute window.** If you don't approve within 15 minutes the pending state expires and `--poll` fails with `expired` — run `--start` again for a fresh code.
- **`--poll` is the finish step, not `--finish`.** Codex is a device-code flow: it's `--start` then `--poll`. A `--finish` call is rejected because Codex uses the device-code protocol, not paste-code.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"codex","error":"...","detail":"..."}`.

---

## Codex subscription vs. the OpenAI API-key provider

It's easy to confuse the two because they both run OpenAI models. Here's the difference:

| | **Codex provider** (this page) | **OpenAI API-key provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `codex` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-codex-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Draws on your ChatGPT/Codex subscription (usage windows) | Pay-per-token OpenAI API usage |
| **What you need** | A ChatGPT Plus / Pro / Business (or Codex) account | An OpenAI API key and a platform account |
| **Token lifecycle** | Short-lived access token + automatic refresh | Static API key, no refresh |
| **Endpoint** | `chatgpt.com/backend-api/codex/responses` | Any OpenAI-compatible `/v1/chat/completions` |

They run the same underlying OpenAI models, but the **Codex provider bills against your existing subscription** while the **OpenAI API-key provider bills per token against your API key** (and, via `openai`, can point at any OpenAI-compatible server, local or remote). Pick the one that matches how you already pay for OpenAI models. See [Using the OpenAI Provider](openai-provider.md).

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using the OpenAI Provider](openai-provider.md)
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md)
