# Using Grok Build Subscription in SCORPIOX CODE

You have an **xAI Grok Build** subscription but you don't want to run on a separate `XAI_API_KEY` metered for per-token API billing. You can use it directly in SCORPIOX CODE. Instead of metering API tokens, the **Grok provider** signs you in with the same **OAuth session login** the official Grok CLI uses and runs requests against the Grok Build endpoint your subscription already pays for.

This page walks through the **OAuth session login**, how the token is stored and kept fresh, how to juggle multiple accounts with profiles (`/profile` and `/use`), and how a Grok Build subscription differs from standard xAI API-key usage.

Docs for SCORPIOX CODE @ `2b0bffd`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Grok Build / xAI subscription** | A Grok Build plan you already pay for |
| **You don't have (or don't want) an xAI API key** | No pay-per-token billing — requests draw on your subscription credits |
| **Headless / over SSH** | The Grok login uses a device-auth flow that works without a local browser |
| **Driven by SCORPIO BOT** | The login state can be checked as a separate command with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the xAI API directly with an API key (`XAI_API_KEY`) against `api.x.ai`, you don't need this provider at all — set the key and SCORPIOX CODE can use it as a bearer token. The two are different billing models for the same models — see [Grok Build subscription vs. the xAI API-key path](#grok-build-subscription-vs-the-xai-api-key-path) below.

> **Subscription, not API key.** The Grok provider authenticates with OAuth and draws on your account's usage allowance (the weekly / monthly credit windows Grok Build enforces). It does **not** bill per token, and the subscription endpoint is `cli-chat-proxy.grok.com` — **not** `api.x.ai` (that is the API-key path).

---

## Step 1 — Sign in with the OAuth session flow

Grok's session login is the **device-auth flow** owned by the official `grok` CLI. SCORPIOX CODE does not re-implement that protocol; it drives it for you.

```bash
scorpiox-grok-login
```

What happens:

1. SCORPIOX CODE checks whether the `grok` CLI is on your `PATH` (or in `~/.grok/bin/grok`, `/usr/local/bin/grok`, `/root/.grok/bin/grok`).
2. If it is, it launches `grok login --device-auth` for you — complete the device-auth prompt in your browser on whatever device is already signed in.
3. If the `grok` CLI is **not** present, it tells you exactly how to get signed in, either by installing the CLI:

```bash
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth
```

   …or by copying an existing credential file from a machine where you've already logged in:

```bash
scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

The login always writes a ready-made `grok` profile for you (skipped if it already exists):

```
Created profile: /home/you/.claude/scorpiox-env/grok.txt
  /profile grok   (or ACTIVE_PROFILE=grok)
```

A few things worth knowing about this flow:

- **The access token is short-lived by design** — but you never manage that (see [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh)).
- **The credential lives at `~/.grok/auth.json`.** That is the same file the official Grok CLI writes, so a login you did once in the Grok CLI is immediately usable by SCORPIOX CODE.
- **No browser on the box? No problem.** Device-auth lets you approve from a laptop or phone and stay signed in on the headless node.

### Where the token lives

SCORPIOX CODE reads the OAuth credential from `~/.grok/auth.json` by default. You can override the location:

| How to override | What it does |
|-----------------|--------------|
| `GROK_CREDENTIALS_FILE` | Point at a specific credential file (useful for pinning a profile to a specific account). |
| `GROK_HOME` | Look under `$GROK_HOME/auth.json` instead of `~/.grok/auth.json`. |

The file holds OIDC entries keyed by issuer and carries an `expires_at` timestamp. SCORPIOX CODE always picks the entry with the latest expiry, so a file containing several logins still resolves to the freshest valid one. If no credential file exists and `XAI_API_KEY` is set in the environment, that key is used as a fallback bearer token — but that is the API-key path, not your subscription.

---

## Step 2 — Turn on the Grok provider

The provider switches on with `PROVIDER=grok`. For a Grok Build subscription you set `GROK_TOKEN_SOURCE=local` so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=grok
GROK_TOKEN_SOURCE=local
MODEL=grok-4.6
```

Activate the profile (or set those keys in any cascade tier, or as OS environment variables) and SCORPIOX CODE connects.

```
/profile grok      # activate this profile and persist it
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `grok` | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a Grok Build login you ran with `scorpiox-grok-login`. |
| `GROK_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.grok/auth.json`. Point it at a named account to pin a profile to a specific login. |
| `MODEL` / `GROK_MODEL` | text | `grok-4.6` | Which model to run. `GROK_MODEL` wins if both are set. Accepts full Grok model IDs or short aliases (see [Choosing a model](#choosing-a-model)). |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=http`. |
| `GROK_SSH_HOST` / `GROK_SSH_PORT` / `GROK_SSH_USER` / `GROK_SSH_PASS` | text | *(empty)* | SSH host details, used only when `GROK_TOKEN_SOURCE=ssh`. |

The endpoint the provider calls is fixed to `https://cli-chat-proxy.grok.com/v1/responses` — the Grok Build OAuth surface, not `api.x.ai`.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** When the stored access token is close to expiring (a 5-minute buffer ahead of its expiry), the provider runs the refresh step and reloads the token, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries. A second failure after a refresh is reported as a permanent auth error.
- **Remote / TCP sources always fetch fresh.** In `http`, `ssh`, or `tcp` modes the token is re-fetched on every request (the remote endpoint is the source of truth and may have revoked an old token), so there is no local expiry to worry about.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-grok-refreshtoken            # refresh if the token is expired (5 min buffer)
scorpiox-grok-refreshtoken --force    # always refresh
scorpiox-grok-refreshtoken --verbose  # also print the old/new tokens
```

### Inspecting the resolved configuration

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, GROK_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `GROK_TOKEN_SOURCE`, and `MODEL`, so you can confirm you're pointed at the account you expect before a long run. To check exactly which file a `local` token source reads, point `GROK_CREDENTIALS_FILE` at it explicitly and compare.

### Checking your subscription usage

The provider runs against the same credit windows Grok Build enforces. Check how full they are with:

```bash
scorpiox-grok-usage            # human-readable summary
scorpiox-grok-usage --json     # raw billing JSON
```

You'll see the **weekly** and **monthly** limits with their percentage used and reset times, plus any **prepaid credits** and **on-demand** balance — so you know when a window resets before you kick off a long run.

To see which models the endpoint currently advertises:

```bash
scorpiox-grok-models            # list available models
```

---

## Multi-account: profiles and switching

Many people have more than one Grok / xAI account (personal + work, for example). The Grok provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

Each login lives in a credential file. Keep several side by side by pointing `GROK_CREDENTIALS_FILE` at a different file per account (copy each from a device signed in to that account):

```bash
scp ~/.grok/auth.json root@host:~/.grok/auth-personal.json
scp ~/.grok/auth.json root@host:~/.grok/auth-work.json
```

### Bind each login to a profile

Now make one profile per account. Point `GROK_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/grok-personal.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/auth-personal.json
GROK_MODEL=grok-4.6
```

```
# ~/.claude/scorpiox-env/grok-work.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/auth-work.json
GROK_MODEL=grok-4.5
```

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile grok-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use grok-personal      # session-only — gone when the session ends
/profile                # open the profile picker (or list if no picker is installed)
/profile off            # deactivate the profile
```

---

## Choosing a model

`MODEL` (or `GROK_MODEL`, which takes precedence) accepts either a full Grok model ID or a short alias. The aliases at this commit are:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `grok-4.6` (provider default) |
| `opus` | `grok-4.6` |
| `sonnet` | `grok-4.6` |
| `haiku` | `grok-4.5` |
| any full `grok-*` ID | passed through as-is |

You can pin a specific version per alias by setting `MODEL` to the full ID in your profile, or switch it at runtime with the `/model` command. The other models the endpoint advertises can also be set directly in `MODEL` — list them with `scorpiox-grok-models`.

> Note: the login-offered `grok` profile ships with `MODEL=grok-4.6` out of the box. Change it to `grok-4.5` (or leave the alias as `opus` / `sonnet` / `haiku`) depending on the model you want by default.

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open during login — a background agent, a CI step, or SCORPIO BOT's provider flow. The login tool ships an **additive machine mode** for reporting state without a TTY.

Because Grok's sign-in is the official `grok login --device-auth` flow (a protocol SCORPIOX CODE does not own), machine mode is **status-only**: it reports whether this node is signed in and offers the terminal login, rather than splitting the device-auth flow into `--start` / `--poll` steps.

| Flag | What it does |
|------|--------------|
| `--status` | Report the Grok login state for this node as JSON (signed-in, whether a refresh token is present, whether the profile exists). |
| `--cancel` | No-op acknowledgement for a pending machine flow. |

A typical status check:

```bash
scorpiox-grok-login --status
#   {"ok":true,"provider":"grok","flow":"terminal","display":"Grok (xAI)","path":"~/.grok/auth.json","profile":"grok","logged_in":true,"has_refresh":true,"profile_exists":true,"identity":"grok CLI installed"}
```

Things worth knowing:

- **It never prints tokens.** Only status fields, so the output is safe to log.
- **The actual sign-in still happens in a terminal** via `scorpiox-grok-login` (or `grok login --device-auth`). Machine mode tells you when to do that, not how.
- **Running the tool with no machine flag** is the original interactive behaviour, unchanged.

---

## Grok Build subscription vs. the xAI API-key path

It's easy to confuse the two because they both run Grok models. Here's the difference:

| | **Grok provider** (this page) | **xAI API-key usage** |
|---|---|---|
| `PROVIDER` value | `grok` | *(none)* — you just supply the key |
| **Authentication** | OAuth session login (`scorpiox-grok-login`, device-auth) | `XAI_API_KEY` bearer token |
| **Billing** | Draws on your Grok Build subscription (credit windows) | Pay-per-token xAI API usage |
| **What you need** | A Grok Build / xAI subscription | An xAI API key and a platform account |
| **Token lifecycle** | Short-lived access token + automatic refresh | Static API key, no refresh |
| **Endpoint** | `cli-chat-proxy.grok.com` (Grok Build OAuth surface) | `api.x.ai` (standard API surface) |

They run the same underlying Grok models, but the **Grok provider bills against your existing Grok Build subscription** while the **API-key path bills per token against your `XAI_API_KEY`**. Pick the one that matches how you already pay for Grok.

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md)
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md)
- [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md)
