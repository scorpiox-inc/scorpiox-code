# Using Grok Build Subscription in SCORPIOX CODE

You have a **Grok Build subscription** (a Grok CLI or Grok Build plan from xAI) and you would rather not buy a separate xAI API key or meter every turn against per-token billing. You can use it directly in SCORPIOX CODE. Instead of paying per token, the **Grok provider** (`PROVIDER=grok`) signs you in with the same **OAuth session login** the Grok CLI performs and sends every request to the `cli-chat-proxy.grok.com` endpoint your subscription already pays for.

This page walks through the session login, where the token is stored, how refresh works (and why there is nothing to manage), how to juggle several accounts with profiles (`/profile` and `/use`), and how a Grok Build subscription differs from standard xAI API-key usage.

Docs for SCORPIOX CODE @ `d100592`.

> **The whole idea in one line:** run `scorpiox-grok-login`, approve the device-auth code in your browser — SCORPIOX CODE stores the OAuth tokens in `~/.grok/auth.json`, writes a ready-to-use profile, and every later request renews its own short-lived access token from the stored refresh token. No API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Grok Build / Grok CLI subscription** | A Grok Build plan or a Grok CLI seat you already pay for |
| **You don't have (or don't want) an xAI API key** | Requests draw on your subscription's credit windows, not per-token billing |
| **You share one login across several machines** | The token can be fetched over SSH, HTTP, or a raw TCP socket instead of a local file |
| **Headless / over SSH** | The login is device-auth based: approve in a browser on any device, the code lands in the terminal |
| **Driven by SCORPIOX BOT** | The login supports `--status` for non-interactive checks — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling xAI with an **API key** (`XAI_API_KEY` against `api.x.ai`), that is a different provider: `PROVIDER=openai` (or a custom OpenAI-compatible endpoint), described in [OpenAI-Compatible Providers](openai-provider.md). The two run the same models on different billing models — see [Grok Build subscription vs. standard API-key usage](#grok-build-subscription-vs-standard-api-key-usage) below.

> **Subscription, not API key.** The Grok provider authenticates with OAuth and draws on your account's credit allowance — the same weekly and monthly windows the Grok CLI enforces. It does not accept an `XAI_API_KEY` and does not bill per token.

---

## Step 1 — Sign in with the OAuth device-auth flow

The login is the **OAuth device-auth flow** the Grok CLI uses: it prints a URL and a short code, you approve in a browser on any device, and the token appears in `~/.grok/auth.json`. No browser is needed on the machine you are running from.

```bash
scorpiox-grok-login
```

What happens:

1. **SCORPIOX CODE checks for existing credentials.** If `~/.grok/auth.json` already exists, it reports that and skips re-login.
2. **A profile is written.** SCORPIOX CODE creates `~/.claude/scorpiox-env/grok.txt` with the provider settings you need (see [Step 2](#step-2--turn-on-the-grok-provider) below).
3. **The Grok CLI is launched.** If the `grok` binary is on your `PATH`, SCORPIOX CODE runs `grok login --device-auth` for you. You see a URL and a code:

```
Launching: /usr/local/bin/grok login --device-auth

Open this URL in your browser to complete login:
  https://x.ai/cli/auth?device_code=XXXXXXXX

Enter the code on your device:
  ABCD-1234
```

4. **You approve in your browser.** Open the URL, log in to your xAI account, enter the code. The Grok CLI then writes `~/.grok/auth.json` with your access token, refresh token, and expiry.

If the `grok` CLI is not installed, SCORPIOX CODE prints the install command:

```bash
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth
```

You can also copy an existing `auth.json` from another machine:

```bash
scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

> **Token file location.** `~/.grok/auth.json` (or `$GROK_HOME/auth.json` if you set `GROK_HOME`). This is the same file the Grok CLI writes — SCORPIOX CODE reads it, never overwrites it.

---

## Step 2 — Turn on the Grok provider

The provider switches on with `PROVIDER=grok`. For a personal subscription you also set `GROK_TOKEN_SOURCE=local`, so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=grok
GROK_TOKEN_SOURCE=local
```

The login command already wrote a ready-to-use profile for you. Activate it and SCORPIOX CODE connects:

```text
/profile grok      # activate the login-written profile and persist it
```

You can also pick the provider for a single run without touching any file:

```bash
sx --provider grok -p "explain this diff"
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local`, `http` (alias `remote`), `ssh`, or `tcp`. Use **`local`** for a login you ran with `scorpiox-grok-login`. |
| `GROK_CREDENTIALS_FILE` | text | *(empty)* | Override the local credential path. Defaults to `~/.grok/auth.json`. Point it at a named account to pin a profile to one login. |
| `GROK_MODEL` | text | `grok-4.6` | Which model to run. Short aliases: `opus` or `sonnet` map to `grok-4.6`, `haiku` maps to `grok-4.5`. Full `grok-*` model IDs pass through unchanged. |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=http`. |
| `TOKEN_HTTP_API_KEY` | text | *(empty)* | Optional bearer token for the HTTP token endpoint, sent as an `Authorization: Bearer` header. Shared across providers. |
| `GROK_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=ssh` — read the token from a remote machine over SSH. The port is optional (default `22`). |

### Token sources at a glance

| `GROK_TOKEN_SOURCE` | Reads token from | Typical use |
|---------------------|------------------|-------------|
| `local` (default) | `~/.grok/auth.json` (or `GROK_CREDENTIALS_FILE`) | You logged in on this machine |
| `http` / `remote` | `GROK_REMOTE_URL` (JSON `{"access_token":"...","expires_at":...}`) | Token service on another host |
| `ssh` | `cat $HOME/.grok/auth.json` on `GROK_SSH_HOST` | Token lives on a build server |
| `tcp` | Raw TCP socket (`TCP_HOST:TCP_PORT`) | Minimal, dependency-free transport |

---

## How token refresh works

You do nothing.

Every request, SCORPIOX CODE checks the token expiry in `~/.grok/auth.json`. If the access token is within **5 minutes** of expiring, SCORPIOX CODE automatically runs the refresh: it takes the stored refresh token, exchanges it for a fresh access token, and writes the updated file back. If the refresh token itself has expired, you re-login (`scorpiox-grok-login`) — that is the only time you touch the flow again.

The refresh uses the same OIDC endpoint the Grok CLI uses, so the credentials stay valid in both tools simultaneously.

> **Why 5 minutes?** It is a safety buffer so a slow network or a briefly-stopped process does not race the expiry. You will never see a "token expired" error in normal operation.

---

## Checking your usage

Grok Build subscriptions use credit windows (weekly and monthly), not per-token billing. Check where you stand:

```bash
scorpiox-grok-usage           # pretty: Weekly/Monthly limit NN% used
scorpiox-grok-usage --json    # raw billing JSON
```

The command hits the same billing endpoint the Grok CLI uses (`cli-chat-proxy.grok.com/v1/billing?format=credits`) with your OAuth token, so you see the exact same numbers the CLI shows. Inside SCORPIOX CODE, the in-session `/usage` command runs this same tool and titles the popup **Grok Usage** while this provider is active.

---

## Multi-account: profiles and switching

If you hold more than one xAI account, or want to pin a project to a specific Grok login, use **named config profiles**.

### Save more than one login

Each login writes to `~/.grok/auth.json` by default. To keep separate credentials, use `GROK_CREDENTIALS_FILE` to point each profile at a different file:

```
# ~/.claude/scorpiox-env/grok-personal.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/accounts/personal.json
GROK_MODEL=grok-4.6
```

```
# ~/.claude/scorpiox-env/grok-work.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/accounts/work.json
GROK_MODEL=grok-4.5
```

To create the separate credential files, log in on a machine where that account is active (or copy the `auth.json` over), then copy it to the desired path.

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```text
/profile grok-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use    grok-personal   # session-only — gone when the session ends
/profile              # open the profile picker (or list profiles)
/profile off          # deactivate the profile, drop back to the base cascade
```

Both commands reload the provider live, so the new account and model take effect immediately, and both revert automatically if the new profile cannot initialize. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

---

## Machine mode (no TTY)

When driven by SCORPIOX BOT or a script, the login is status-only because the Grok CLI owns the device-auth protocol:

```bash
scorpiox-grok-login --status    # JSON: { "provider":"grok", "logged_in":true, ... }
```

The output tells the bot whether `~/.grok/auth.json` exists and whether the `grok` CLI is installed. If it is not installed, the bot reports `"terminal_only"` and the user is directed to run the install + login in a terminal.

---

## Grok Build subscription vs. standard API-key usage

| | Grok Build subscription (`PROVIDER=grok`) | xAI API key (`XAI_API_KEY`) |
|---|---|---|
| **Auth** | OAuth device-auth; token in `~/.grok/auth.json` | Bearer token in env or config |
| **Endpoint** | `cli-chat-proxy.grok.com/v1/responses` | `api.x.ai/v1/chat/completions` |
| **Billing** | Credit windows (weekly / monthly), drawn from your subscription | Per-token, metered against your xAI account balance |
| **Token lifetime** | Short-lived access token, auto-refreshed from stored refresh token | Static key, valid until revoked |
| **Refresh** | Automatic, transparent, nothing to manage | None (key does not expire) |
| **Models** | Grok 4.6, Grok 4.5 (and any future Grok model xAI adds to the CLI surface) | Same model family, via the standard OpenAI-compatible API |
| **Quota visibility** | `scorpiox-grok-usage` shows your remaining credits | No built-in quota check; see your xAI dashboard |
| **Multi-machine** | Token can be fetched over SSH / HTTP / TCP | Key is just a string; copy it anywhere |

Use the **Grok Build subscription** when you already pay for a Grok plan and want to avoid per-token billing. Use the **xAI API key** when you need the standard OpenAI-compatible endpoint, a programmatic key in CI, or a model not yet exposed through the CLI surface.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `No grok CLI on PATH` | Install it: `curl -fsSL https://x.ai/cli/install.sh \| bash` |
| Token expired and refresh failed | Re-run `scorpiox-grok-login` (or `grok login --device-auth`) |
| `~/.grok/auth.json` missing on a remote machine | `scp ~/.grok/auth.json user@host:~/.grok/` or set `GROK_TOKEN_SOURCE=ssh` |
| Wrong model / model not found | Check `GROK_MODEL` in your profile; short aliases `opus`, `sonnet`, `haiku` are mapped to the correct model IDs |
| 401 / auth error | The access token may have expired without a successful refresh — run `scorpiox-grok-refreshtoken --force`, then re-login if that also fails |
| Billing / usage shows 0% | Confirm your subscription is active in the xAI dashboard; the usage endpoint reflects server-side credits, not local state |

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — cascade tiers, `ACTIVE_PROFILE`, `/profile` and `/use`
- [Token Usage Observability](usage-observability.md) — status bar indicators, `/usage` command
- [Claude Code CLI Subscription](claude-code-provider.md) — the analogous Anthropic subscription provider
- [Google Antigravity CLI Subscription](antigravity-provider.md) — the analogous Google subscription provider
