# Using Grok Build Subscription in SCORPIOX CODE

Have an xAI **Grok Build** subscription (the same sign-in the official Grok CLI uses) but no xAI API key? You can use it directly in SCORPIOX CODE. Instead of metering an `XAI_API_KEY`, the **Grok provider** signs you in with the OAuth **device-auth flow** the official Grok CLI uses and runs requests against the Grok Build endpoint your subscription already pays for.

This page walks through the **OAuth session login**, how the token is stored and kept fresh automatically, how to switch accounts and models with profiles, and how direct Grok Build subscription access differs from the standard xAI API-key usage.

Source of truth: `sx_provider_grok.c`, `scorpiox-grok-login.c`, `scorpiox-grok-fetchtoken.c`, and `scorpiox-grok-refreshtoken.c` at commit `6c70ad6`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Grok Build subscription** | The xAI plan you already pay for and sign into with the Grok CLI |
| **You don't have (or don't want) an `XAI_API_KEY`** | No per-token API billing — requests draw on your subscription |
| **You want Grok models through xAI's own backend** | `grok-4.6` and other `grok-*` model IDs run over one Grok Build sign-in |
| **Headless / over SSH** | The device-auth flow works over SSH: open a link and type a code on any device, no browser needed on the box |

If you are instead calling an xAI-compatible server with an API key (`XAI_API_KEY`), use `PROVIDER=openai` (or the xAI API-key path) instead. The two are different billing models for the same models — see [Grok provider vs. the xAI API-key path](#grok-provider-vs-the-xai-api-key-path) below.

> **Subscription, not API.** The Grok provider authenticates with OAuth and draws on your account's subscription allowance (the weekly / monthly credit windows the Grok Build backend enforces). It does **not** accept an `XAI_API_KEY` for the chat endpoint and does **not** bill you per token against an API account.

---

## Step 1 — Sign in with the device-auth flow

The default login is the **OAuth device-auth flow**. It is deliberately headless- and SSH-friendly: SCORPIOX CODE prints a link and a one-time code, you sign in from a browser on whatever device is already signed in, then the Grok CLI completes the exchange and writes the session credentials to `~/.grok/auth.json`. No redirect server and no pasting a callback URL.

```bash
scorpiox-grok-login
```

`scorpiox-grok-login` first looks for the official **`grok` CLI** on your `PATH` and, when it's present, launches it:

```bash
grok login --device-auth
```

Do exactly what the Grok CLI tells you:

1. **Open the printed link** in a browser — on any device, wherever you're signed into your xAI / Grok Build account.
2. **Sign in** to your account and **enter the one-time code**.
3. Come back to your terminal and wait — the CLI completes the device exchange and saves the session to `~/.grok/auth.json`.

A few things worth knowing about this flow:

- **The code window is about 15 minutes.** If the wait times out, just re-run `scorpiox-grok-login` and a fresh code is issued.
- **Never share the code.** It's a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up — this is the main reason the device-auth flow is the default.

### If the Grok CLI isn't installed

`scorpiox-grok-login` prefers the official binary, but it degrades gracefully. If `grok` isn't on your `PATH`, it tells you how to get the credentials in either of two ways:

```bash
# Option A — install the official CLI and let it run the device flow
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth

# Option B — copy an existing session from a machine you've already signed into
scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

Either way, the provider only needs a valid `~/.grok/auth.json` on disk once you're done. If a credentials file is already present, `scorpiox-grok-login` reports it and won't clobber it.

---

## Step 2 — Turn on the Grok provider

On success, `scorpiox-grok-login` writes a ready-to-use **`grok` profile** for you:

```
Created profile: /home/you/.claude/scorpiox-env/grok.txt
  /profile grok   (or ACTIVE_PROFILE=grok)
```

The profile it writes looks like this:

```
PROVIDER=grok
MODEL=grok-4.6
GROK_MODEL=grok-4.6
GROK_TOKEN_SOURCE=local
```

Activate it in-session:

```
/profile grok        # activate and persist
```

Or set the keys yourself in any cascade tier, a named profile, or as OS environment variables. The minimal set for a personal subscription is:

```
PROVIDER=grok
GROK_TOKEN_SOURCE=local
```

That's it. `MODEL` defaults to `grok-4.6` if left empty.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local` (default — read from `~/.grok/auth.json`), `config` (resolved from this same config), `remote`/`http`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-grok-login`. |
| `GROK_MODEL` | text | *(empty)* | Model override. Takes precedence over the generic `MODEL` key. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts short aliases or a full `grok-*` model ID (see [Choosing a model](#choosing-a-model)). Empty resolves to the built-in default `grok-4.6`. |
| `TOOLS` | bool | `1` | Include the tool-calling block in requests (on by default). |
| `THINKING` | bool | `0` | Set to `1` to enable extended thinking on the Grok endpoint. |
| `GROK_HOME` | text | *(empty)* | Override the directory that holds `auth.json`. Defaults to `~/.grok`. |
| `GROK_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. `GROK_SSH_HOST` and `GROK_SSH_USER` are required in that mode. |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=remote`. |

> **For a personal subscription you only need two keys:** `PROVIDER=grok` and `GROK_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `remote` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Grok login.

### Where the token lives

On login, the Grok CLI writes the OAuth session to `~/.grok/auth.json` (or `$GROK_HOME/auth.json`). It's a small JSON file holding the OIDC access token and its expiry. On every request, `GROK_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written. The file is kept at `0600` permissions.

---

## Token persistence and automatic refresh

The Grok Build access token is short-lived, but the session in `~/.grok/auth.json` can renew it. You don't manage it — with `GROK_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** Before the token gets close to expiring (a 5-minute buffer ahead of its expiry), the provider renews it in place, so in-flight work never hits an expired credential.
- **On 401 retry.** If a request comes back unauthorized, the provider refreshes once and retries before surfacing an error.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-grok-refreshtoken            # refresh if expired (5-minute buffer)
scorpiox-grok-refreshtoken --force    # always refresh
scorpiox-grok-refreshtoken --verbose  # show token details
```

As long as `~/.grok/auth.json` is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
# 1. Refresh the token in place
scorpiox-grok-refreshtoken --force

# 2. If that keeps failing, re-bind with a full login
scorpiox-grok-login
```

### Inspecting usage instead of per-token metering

Because you're on a subscription, there's no per-token metering. Use `scorpiox-grok-usage` to see your credit window and when it resets:

```bash
scorpiox-grok-usage            # pretty: Weekly / Monthly limit NN% used + reset time
scorpiox-grok-usage --json     # raw billing JSON
```

This reports the current billing period (weekly or monthly), the percentage used, the reset time (UTC and local), your prepaid credit balance, and your on-demand spend against its cap.

---

## Profiles and switching

The login writes a single `grok` profile to `~/.claude/scorpiox-env/grok.txt`. You can switch to it (or away from it) live, in-session, with no re-login and no restart:

```
/profile grok        # persistent — writes ACTIVE_PROFILE, survives restarts
/use grok            # session-only — gone when the session ends
/profile              # list available profiles and which is active
/profile off          # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Store more than one login

A Grok session is tied to the `auth.json` it was written from. To keep several accounts side by side, point each profile's `GROK_HOME` (or `GROK_TOKEN_SOURCE` remote/ssh source) at a different `auth.json`, and give each account its own profile file. Create one profile per account, each reading the stored token for that login.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, GROK_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

`MODEL` (or the higher-precedence `GROK_MODEL`) accepts either a short alias or a full Grok model ID. The aliases at this commit resolve to:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `grok-4.6` (built-in default) |
| `opus` | `grok-4.6` |
| `sonnet` | `grok-4.6` |
| `haiku` | `grok-4.5` |
| any `grok-*` ID | passed through as-is |

Any ID that starts with `grok-` is forwarded unchanged, so you can request any model your Grok Build plan exposes. Everything else falls back to the built-in default `grok-4.6`.

> Note: the login-created `grok` profile ships with `MODEL=grok-4.6` out of the box. Change it to the ID you prefer if you want a different default.

You can list every model the xAI endpoint advertises with:

```bash
scorpiox-grok-models              # pretty-printed model list
scorpiox-grok-models --json       # raw JSON to stdout
```

### Image generation (Grok Imagine)

The same Grok Build session also drives xAI's **Grok Imagine** image API. Use `scorpiox-grok-imagegen` to generate or edit images, authenticated with the same session (or an explicit `XAI_API_KEY` / `--token` if you prefer):

```bash
scorpiox-grok-imagegen --prompt "Piha Beach Lion Rock, storm opening to blue"
scorpiox-grok-imagegen --prompt "..." --output /tmp/out.png --aspect 16:9 --resolution 2k --quality medium
scorpiox-grok-imagegen --prompt "add an RTX PRO 6000 on the sand" --image /tmp/piha.png
scorpiox-grok-imagegen --models        # list available Imagine models
```

---

## Grok provider vs. the xAI API-key path

It's easy to confuse the two because they both run Grok models. Here's the difference:

| | **Grok provider** (this page) | **xAI API-key path** |
|---|---|---|
| `PROVIDER` value | `grok` | *(API-key usage via `PROVIDER=openai` / xAI endpoint)* |
| **Authentication** | OAuth device-auth login (`scorpiox-grok-login` → `~/.grok/auth.json`) | `XAI_API_KEY` bearer token |
| **Billing** | Your Grok Build subscription allowance (weekly / monthly credit windows) | Pay-per-token API usage |
| **Token source** | `GROK_TOKEN_SOURCE` (local / config / remote / ssh / tcp) | `XAI_API_KEY` |
| **Endpoint** | `cli-chat-proxy.grok.com/v1/responses` | `api.x.ai` (the API-key path) |
| **Best for** | People who already pay for Grok Build | Pay-as-you-go API access |

Rule of thumb: **you have a Grok Build subscription → `grok`. You have an xAI API key → use the API-key path.** The two hit different endpoints on purpose: the Grok provider talks to the subscription `cli-chat-proxy` backend, not `api.x.ai`.

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-grok-login` writes the token file and offers a profile. You still need `PROVIDER=grok` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file is live.** `~/.grok/auth.json` holds a session that can renew your account — don't commit it, don't share it, and don't loosen its permissions. It's written `0600` by default.
- **Two different endpoints, on purpose.** The Grok provider talks to `cli-chat-proxy.grok.com`, not `api.x.ai`. `api.x.ai` is the API-key path. Don't point the Grok provider at the API-key host expecting subscription billing.
- **`GROK_TOKEN_SOURCE` defaults to `local`.** It reads `~/.grok/auth.json`. If your login lives elsewhere, set `GROK_HOME` or use the `remote` / `ssh` / `tcp` sources — otherwise the provider can't find a token and requests will fail.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Grok account your standing default and `/use` to hop to it for a single session without writing anything.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-grok-refreshtoken --force` keeps failing (expired session, account changed, plan changed), do a full `scorpiox-grok-login` to re-bind.
- **Subscription windows are enforced upstream.** The weekly / monthly credit limits are xAI's, not SCORPIOX CODE's. Check your utilization and reset time with `scorpiox-grok-usage` instead of per-token metering.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile can't initialize.
