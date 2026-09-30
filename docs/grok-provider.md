# Using Grok Build Subscription in SCORPIOX CODE

You have an **xAI Grok Build** subscription (the plan behind the official `grok` CLI) but you don't want to meter per-token against an `XAI_API_KEY` or run a separate paid API key just to use the models you already pay for. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Grok provider** signs you in through the official Grok CLI's OAuth **device-auth** flow and runs requests against the `cli-chat-proxy.grok.com` endpoint your subscription already pays for.

This page walks through the **session login**, how the token is stored and kept fresh by automatic refresh, how to juggle accounts with profiles (`/profile` and `/use`), how to inspect your models and credit usage, how to drive the login state without holding a TTY (**machine mode**), and how a Grok Build subscription differs from standard xAI API-key usage.

Docs for SCORPIOX CODE @ `13253cf`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have an xAI Grok Build subscription** | The plan behind the official `grok` CLI — anything `grok login --device-auth` can sign in to |
| **You don't have (or don't want) an `XAI_API_KEY`** | No pay-per-token billing — requests draw on your Grok Build plan's credits |
| **Headless / over SSH** | The login is a device-auth flow: open the link on any device and sign in — no browser or redirect needed on the box you run from |
| **Driven by SCORPIO BOT** | The login state can be inspected as a single non-interactive command (`--status`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the xAI API with an API key (`XAI_API_KEY`), use `PROVIDER=openai` pointed at `api.x.ai` (or an OpenAI-compatible endpoint). The two are different billing models for overlapping model families — see [Grok Build subscription vs. the xAI API-key path](#grok-build-subscription-vs-the-xai-api-key-path) below.

> **Subscription, not API key.** The Grok provider authenticates with an OAuth session and draws on your account's included credits (the same weekly/monthly credit limit the official Grok CLI enforces). It does **not** accept an `XAI_API_KEY` for chat and does **not** bill per token.

---

## Step 1 — Sign in with the device-auth flow

The Grok login hands the sign-in to the **official Grok CLI's `grok login --device-auth`** flow — the same OAuth device-auth the first-party CLI uses. It is deliberately SSH- and headless-friendly: you open a link, sign in to your xAI account on whatever device you already use, and the result lands in the credential file on the machine you're running from. There is no long URL to paste and no localhost redirect to capture, and nothing long-lived leaves the terminal.

```bash
scorpiox-grok-login
```

SCORPIOX CODE first checks for the official `grok` binary on your `PATH`. If it's present, it launches `grok login --device-auth` and lets the official flow do the sign-in:

1. **Open the printed link** in your browser — on any device, wherever you're signed in to xAI.
2. **Sign in** to your xAI / Grok Build account and **approve** the device.
3. Come back to your terminal. The official flow polls and, the moment you approve, writes the session credential to `~/.grok/auth.json`.

If the official `grok` CLI is **not** installed, `scorpiox-grok-login` tells you how to get it (or copy an existing credential file to a headless box):

```
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth
```

or, if you already signed in somewhere else:

```bash
scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

Whether the official CLI is installed or not, `scorpiox-grok-login` **always** writes a ready-to-use `grok` profile so the provider is one switch away (see [Step 2](#step-2--turn-on-the-grok-provider)). A few things worth knowing about this flow:

- **The credential is a session, not a key.** The login stores an OAuth session (an access token plus a refresh token and its expiry) in `~/.grok/auth.json`. No API key is involved and no key is ever written.
- **The file is the source of truth.** Everything the Grok provider needs is read from `~/.grok/auth.json` (or `$GROK_HOME/auth.json`, or the path in `GROK_CREDENTIALS_FILE`). Point the provider there and it works.

---

## Step 2 — Turn on the Grok provider

The provider switches on with `PROVIDER=grok`. For a personal Grok Build subscription you set `GROK_TOKEN_SOURCE=local` so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=grok
GROK_TOKEN_SOURCE=local
```

Activate the profile (or set those keys in any cascade tier, or as OS environment variables) and SCORPIOX CODE connects.

```
/profile grok      # activate this profile and persist it
/use grok          # activate for this session only (no file change)
```

The `grok` profile `scorpiox-grok-login` writes looks like this:

```
PROVIDER=grok
MODEL=grok-4.6
GROK_MODEL=grok-4.6
GROK_TOKEN_SOURCE=local
# Token: ~/.grok/auth.json via scorpiox-grok-fetchtoken -local
# Host:  cli-chat-proxy.grok.com (OAuth) — not api.x.ai
```

You can also create it by hand under `~/.claude/scorpiox-env/grok.txt` if you prefer.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(empty)* | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where to get the token: `local`, `remote`, `ssh`, `tcp`, or `config`. Use **`local`** for a Grok Build login you ran with `scorpiox-grok-login`. `config` makes the provider read this same key from the environment at runtime. |
| `GROK_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.grok/auth.json` (or `$GROK_HOME/auth.json`). Point it at a named account to pin a profile to a specific login. |
| `MODEL` / `GROK_MODEL` | text | *(empty)* | Which model to run. `GROK_MODEL` takes precedence over `MODEL`. Defaults to `grok-4.6` when neither is set. |
| `GROK_HOME` | env | *(empty)* | Environment variable — an alternative base directory for `auth.json`, used in preference to `$HOME`. |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=remote`. |
| `GROK_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |

> **For a personal subscription, you only need two keys:** `PROVIDER=grok` and `GROK_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `ssh` / `remote` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Grok Build login.

### Where the token lives

On login, the credential is written to `~/.grok/auth.json` (mode `0600`). It holds one or more OIDC entries, each with an access token (`key`), its `expires_at` timestamp, and a `refresh_token`. On every request, `GROK_TOKEN_SOURCE=local` reads from that file (or from `GROK_CREDENTIALS_FILE` / `$GROK_HOME` if set), always picking the entry with the latest expiry. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

Unlike a static API key, a Grok Build session token is **short-lived** — but you don't have to manage that. The provider keeps it fresh for you:

- **Proactive refresh before expiry.** When the stored token is close to expiring (within a five-minute buffer), the provider runs `scorpiox-grok-refreshtoken`, which exchanges the stored refresh token for a new access token via the OIDC refresh grant and writes it back to `~/.grok/auth.json`. You never have to think about it.
- **Reactive recovery on 401.** If a request still comes back unauthorized (HTTP 401), the provider refreshes and retries once. A second failure after a refresh is reported as a permanent auth error — that's your cue to re-run `scorpiox-grok-login`.
- **Remote / SSH / TCP sources always fetch fresh.** In `remote`, `ssh`, or `tcp` modes the token is re-fetched on every request (the remote endpoint is the source of truth and may have revoked an old token), so there is no local expiry to worry about.
- **Manual refresh.** You can refresh any time you like:

```bash
scorpiox-grok-refreshtoken            # refresh if the token is expired (5-min buffer)
scorpiox-grok-refreshtoken --force    # always refresh
scorpiox-grok-refreshtoken --verbose  # also print token details
```

### Inspecting the resolved configuration

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, GROK_TOKEN_SOURCE, MODEL, …) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `GROK_TOKEN_SOURCE`, and `MODEL`, so you can confirm you're pointed at the account and model you expect before a long run.

### Checking your subscription usage

The provider runs against the same credit quota the official Grok CLI enforces. Check how full it is with:

```bash
scorpiox-grok-usage            # human-readable summary
scorpiox-grok-usage --json     # raw billing JSON
```

You'll see your current credit limit (weekly or monthly) as a percentage used, the reset time (UTC and local) and how long until it resets, plus your prepaid balance and on-demand spend. Useful for knowing when a window resets before you kick off a long run.

### Listing the models you can use

The full set of model IDs your account can reach:

```bash
scorpiox-grok-models            # pretty-printed model list
scorpiox-grok-models --json     # raw JSON
```

---

## Switching between accounts

### Save more than one login

A Grok session lives in a single `auth.json`, so to use more than one xAI account you point each profile at its own copy of the file:

```bash
# sign in to account A (writes ~/.grok/auth.json), then set it aside
cp ~/.grok/auth.json ~/.grok/accounts/personal.json
# sign in to account B
scorpiox-grok-login
cp ~/.grok/auth.json ~/.grok/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account. Point `GROK_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/grok-personal.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/accounts/personal.json
MODEL=grok-4.6
```

```
# ~/.claude/scorpiox-env/grok-work.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/accounts/work.json
MODEL=grok-4.6
```

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile grok-work       # persistent — writes ACTIVE_PROFILE, survives restarts
/use grok-personal       # session-only — gone when the session ends
/profile                 # open the profile picker (or list if no picker is installed)
/profile off             # deactivate the profile
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

---

## Choosing a model

`MODEL` (or `GROK_MODEL`) takes a Grok model ID; `GROK_MODEL` takes precedence over `MODEL`. When neither is set, the provider defaults to `grok-4.6`.

| `MODEL` value | Behaviour |
|---------------|-----------|
| *(empty)* | `grok-4.6` (provider default) |
| any `grok-*` ID | passed through as-is |

The full list your account can reach is always available from `scorpiox-grok-models`. You can pin a specific version by setting `MODEL` in your profile, or switch it at runtime with the `/model` command.

> Note: the login-offered `grok` profile ships with `MODEL=grok-4.6` out of the box. Change it to any model ID from `scorpiox-grok-models` depending on the model you want by default.

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open while a device-auth sign-in is in progress — a background agent, a CI step, or SCORPIO BOT's provider flow. The Grok login ships an **additive machine mode**: the same non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

Because Grok Build signs in through the **official Grok CLI** (`grok login --device-auth`) — a protocol SCORPIOX CODE does not own — machine mode is **status-only** (flow `"terminal"`): it reports the state and points you at a terminal login. It does not start or poll the device-auth flow itself, and it never writes the profile; the interactive path below still does.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line. |
| `--cancel` | Cancel a pending machine-mode login state. |
| `--start` / `--poll` / `--finish` | Recognised, but rejected with `terminal_only`: Grok signs in through the official `grok` CLI, so use Terminal login. |

A typical machine-mode check:

```bash
scorpiox-grok-login --status
# {"ok":true,"provider":"grok","display":"Grok (xAI)","flow":"terminal","path":"~/.grok/auth.json","logged_in":true,"has_refresh":true,"profile_exists":true,...}
```

If `logged_in` is `false`, drive the user to a terminal `scorpiox-grok-login`. A few things worth knowing:

- **`--status` is the whole machine surface.** Grok's device-auth is owned by the official CLI, so there is no `--start`/`--poll` to poll against — the bot shows the state and offers a terminal login instead.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"grok","error":"...","detail":"..."}`.

---

## Grok Build subscription vs. the xAI API-key path

It's easy to confuse the two because they can both reach Grok models. Here's the difference:

| | **Grok provider** (this page) | **xAI API-key path** (`PROVIDER=openai` at `api.x.ai`) |
|---|---|---|
| `PROVIDER` value | `grok` | `openai` (pointed at `api.x.ai`) |
| **Authentication** | OAuth session login (`scorpiox-grok-login` via the official `grok` CLI) | `XAI_API_KEY` bearer token |
| **Billing** | Draws on your Grok Build plan's credits (the same credit limit the official Grok CLI enforces) | Pay-per-token xAI API usage |
| **What you need** | An xAI Grok Build subscription the `grok` CLI can sign in to | An `XAI_API_KEY` and a platform account |
| **Token lifecycle** | Short-lived OAuth session, auto-refreshed before expiry and on 401 | Static API key, no refresh |
| **Chat endpoint** | `cli-chat-proxy.grok.com/v1/responses` (OpenAI Responses wire) | `api.x.ai` chat completions |

They can reach the same underlying models, but the **Grok provider bills against your existing Grok Build subscription** while the **xAI API-key path bills per token against your `XAI_API_KEY`** (and, via `openai`, can point at any OpenAI-compatible server, local or remote). Note the chat provider deliberately targets `cli-chat-proxy.grok.com` — **not** `api.x.ai`, which is the API-key path. Pick the one that matches how you already pay for models. See [Using the OpenAI Provider](openai-provider.md).

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using the OpenAI Provider](openai-provider.md)
- [Using GitHub Copilot CLI Subscription in SCORPIOX CODE](copilot-provider.md)
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md)
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md)
