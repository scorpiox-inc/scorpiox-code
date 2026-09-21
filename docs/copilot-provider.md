# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

Already paying for GitHub Copilot (Pro, Pro+, Business, or Enterprise) but you'd rather not juggle API keys? You can use that subscription directly in SCORPIOX CODE. The **Copilot provider** signs you in with a normal GitHub OAuth **device-code login** and runs requests against the same backend your Copilot subscription already pays for — no metering of API tokens.

This page walks through the **device-code login flow**, how the token is stored and kept fresh, how to switch accounts and models with profiles, and how the Copilot provider differs from the standard [OpenAI API-key provider](openai-provider.md).

Source of truth: the Copilot OAuth login/refresh tools, the Copilot provider implementation, and the config cascade at commit `24427d8`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Pro, Pro+, Business, or Enterprise you already pay for |
| **You don't have (or don't want) an API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The default login flow needs no browser on the machine you run from |

If you are instead calling an OpenAI-compatible API with an API key (`OPENAI_API_KEY` + `OPENAI_BASE_URL`), use `PROVIDER=openai` — see [Using the OpenAI Provider](openai-provider.md).

> **Subscription, not API.** The Copilot provider authenticates with a GitHub OAuth token and uses your Copilot account's usage allowance. It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token. The two are different billing models for the same underlying models.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**, deliberately SSH- and headless-friendly: no browser is required on the machine you're running from, and nothing is pasted back into the terminal.

```bash
scorpiox-copilot-login
```

You'll see something like this:

```
Requesting device code...

┌─────────────────────────────────────────────────────┐
│  Visit: https://github.com/login/device             │
│  Enter code: ABCD-EFGH                              │
└─────────────────────────────────────────────────────┘

Waiting for authorization...
```

Do exactly what it says:

1. **Open the URL** (`https://github.com/login/device`) on *any* device — your laptop, your phone, a kiosk, wherever you're signed into GitHub.
2. **Enter the one-time code** shown in your terminal.
3. Come back to your terminal. Once you've authorized, SCORPIOX CODE polls, gets the token, and saves it automatically.

A few things worth knowing about this flow:

- **The code expires in 15 minutes.** If you don't finish in time, the command times out — just run `scorpiox-copilot-login` again.
- **Never share the code.** It's a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** That's the whole point of device-code login. You approve from wherever you're already signed into GitHub.
- **Denied or expired?** If GitHub reports the authorization was denied or the code expired, re-run the login — the flow starts over cleanly.

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it. Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-copilot-login --force
```

### Other login options

```bash
scorpiox-copilot-login --name <account>   # save to ~/.copilot/accounts/<account>.json
scorpiox-copilot-login --vscode          # VS Code app client (ghu_ token variant)
scorpiox-copilot-login --help            # show usage
scorpiox-copilot-login --version         # show version
```

### Auto-create a `copilot` profile

Right after a successful login, the tool offers to create a ready-to-use config profile:

```
Create 'copilot' config profile?
  Will create: ~/.claude/scorpiox-env/copilot.txt
  Contents:
    PROVIDER=copilot
    COPILOT_TOKEN_SOURCE=local
    MODEL=claude-sonnet-5

Create? [Y/n]
```

Accepting it writes `~/.claude/scorpiox-env/copilot.txt` with exactly those keys. If you decline, create it by hand:

```bash
mkdir -p ~/.claude/scorpiox-env
cat > ~/.claude/scorpiox-env/copilot.txt <<'EOF'
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
MODEL=claude-sonnet-5
EOF
```

Then activate it — persistently or for the session only (see [Step 2](#step-2---turn-on-the-copilot-provider)).

---

## Step 2 — Turn on the Copilot provider

Signing in saves the token; you still need to tell SCORPIOX CODE to use it. Set `PROVIDER=copilot`. For a local subscription that's `COPILOT_TOKEN_SOURCE=local`, which reads the file you just created.

Then activate the profile — persistently or for the session only:

```
/profile copilot     # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot         # session-only — gone when the session ends
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately without a restart. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how profiles resolve across tiers.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `copilot` to activate this provider. |
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where to get the GitHub OAuth token: `local`, `config`, `remote`, `ssh`, or `tcp`. Use **`local`** for a Copilot subscription you logged into with `scorpiox-copilot-login`. |
| `COPILOT_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.copilot/.credentials.json`. Point it at a named account (e.g. `~/.copilot/accounts/work.json`) to pin a profile to a specific login. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | Optional. An official Copilot CLI token (`gho_...`). When set, it is used directly as the bearer with the official CLI identity — no JWT exchange. |
| `COPILOT_MODEL` | text | `claude-sonnet-5` | Which model to run. Takes precedence over the generic `MODEL`. Accepts full Copilot model IDs or short names (see [Choosing a model](#choosing-a-model)). |
| `MODEL` | text | *(empty)* | Generic model fallback, used only when `COPILOT_MODEL` is empty. |
| `COPILOT_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `COPILOT_TOKEN_SOURCE=remote`. |
| `COPILOT_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort level: `low`, `medium`, `high`, or `off`. Falls back to `REASONING_EFFORT`, then `OPENAI_REASONING_EFFORT`. |

> **For a personal subscription, you only need two keys:** `PROVIDER=copilot` and (optionally) `MODEL`. `COPILOT_TOKEN_SOURCE` already defaults to `local`, so the login-created profile sets it for you. The `remote` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Copilot login.

### Where the token lives

On login, SCORPIOX CODE writes the GitHub OAuth token to `~/.copilot/.credentials.json` (mode `0600`, owner-read-only) and, alongside it, a `~/.copilot/config.json` in the shape the official CLI expects. On every request with `COPILOT_TOKEN_SOURCE=local`, SCORPIOX CODE reads from those files. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The token you get from the device-code flow is a long-lived GitHub token, but the per-request credential is a short-lived **Copilot JWT** exchanged from it — and you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** Before a JWT is close to expiring (a 5-minute buffer ahead of its expiry), SCORPIOX CODE re-exchanges the stored GitHub token for a fresh JWT and reloads it, so in-flight work never hits an expired credential.
- **Cached exchange.** The exchanged JWT is cached in `~/.copilot/.copilot-token-cache.json` and only re-exchanged when it's about to expire — not on every single request.
- **Official CLI tokens skip the exchange.** If you supply a `gho_` token via `COPILOT_GITHUB_TOKEN` (or log in with the official CLI identity), SCORPIOX CODE uses it directly as the bearer against the official CLI backend — no JWT minting at all.
- **Manual refresh / diagnostics.** You can force a refresh or inspect the resolved token without making a model request:

```bash
scorpiox-copilot-refreshtoken            # refresh if the token is expired
scorpiox-copilot-refreshtoken --force    # always refresh
scorpiox-copilot-fetchtoken -local       # resolve token from the local credential file
scorpiox-copilot-fetchtoken -config      # read COPILOT_TOKEN_SOURCE from scorpiox-env.txt
```

Re-login is the fallback: if requests keep coming back unauthorized and a forced refresh doesn't help, re-bind the account with `scorpiox-copilot-login --force`.

---

## Multi-account and profile switching

Many people have more than one GitHub account (personal + work, for example). The Copilot provider supports that in two layers: **multiple stored logins** and **named config profiles**.

### Store more than one login

Each `scorpiox-copilot-login --name <account>` run writes a separate credential file:

```bash
scorpiox-copilot-login --name personal   # → ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work       # → ~/.copilot/accounts/work.json
```

To pin a profile to a specific account, set `COPILOT_CREDENTIALS_FILE` in that profile to the matching file:

```
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/work.json
```

Without it, SCORPIOX CODE uses the default `~/.copilot/.credentials.json`.

### Switch models and accounts in-session

Switching is just switching profiles — no re-login, no restart:

```
/profile copilot     # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot         # session-only — gone when the session ends
/profile            # list available profiles and which is active
/profile off        # deactivate
```

`/profile` and `/use` both reload the provider in place and revert automatically if the new profile can't initialize.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from:

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

The Copilot provider accepts a short alias or a full model ID, resolved to a concrete backend model. `COPILOT_MODEL` takes precedence over the generic `MODEL`; the default is `claude-sonnet-5`.

| You type | Resolves to |
|----------|-------------|
| `opus` / `sonnet` / `haiku` | `claude-sonnet-5` (the only Claude ID verified with tool use) |
| any `claude-*` ID | passed through as-is |
| any `gpt-*` ID | passed through as-is |
| any `gemini-*` ID | passed through as-is |
| any `kimi-*` ID | passed through as-is |
| *(empty / unrecognized)* | `claude-sonnet-5` |

Set the model in your profile, or switch it at runtime with the `/model` command. You can also list what your account can actually reach:

```bash
scorpiox-copilot-models              # pretty-print the available model list
scorpiox-copilot-models --json       # raw JSON to stdout
```

And check your plan and remaining usage:

```bash
scorpiox-copilot-usage               # show usage, quota, and plan
scorpiox-copilot-usage --json        # raw JSON
```

---

## Copilot vs. the OpenAI API-key provider

It's easy to confuse the two because they can both run the same models. Here's the difference:

| | **Copilot** (this page) | **OpenAI-compatible** (`PROVIDER=openai`) |
|---|---|---|
| **Authentication** | GitHub OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your GitHub Copilot subscription allowance | Pay-per-token API usage |
| **Endpoint** | GitHub Copilot backend (subscription) | Any OpenAI-compatible `/v1/chat/completions` |
| **Token source** | `COPILOT_TOKEN_SOURCE` (local file / remote / ssh / tcp) | `OPENAI_BASE_URL` + `OPENAI_API_KEY` |
| **Best for** | People who already pay for GitHub Copilot | Self-hosted servers, Azure, Together, Groq, raw OpenAI API |

Rule of thumb: **you have a Copilot subscription → `copilot`. You have an API key or a self-hosted endpoint → `openai`.** If you use another subscription-backed login, the sibling pages — [Claude Code provider](claude-code-provider.md) and [Codex provider](codex-provider.md) — follow the same device-code pattern for their respective subscriptions.

---

## Gotchas

- **`COPILOT_TOKEN_SOURCE` defaults to `local`.** For a personal subscription you still need to run `scorpiox-copilot-login` first — that's what creates `~/.copilot/.credentials.json`. Without it you'll see a token-load error on the first prompt.
- **The login command and the provider are separate.** `scorpiox-copilot-login` only writes the credentials and (optionally) the profile. You still need a `copilot` profile active (via `/profile` or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The device code expires in 15 minutes.** Run the login again if you time out. Never share the code — it's a credential.
- **`~/.copilot/.credentials.json` is owner-read-only (`0600`).** Don't loosen the permissions; it holds the live token that renews your account.
- **Named accounts live in `~/.copilot/accounts/`, not the default file.** `--name work` writes `~/.copilot/accounts/work.json`. A profile only uses it if `COPILOT_CREDENTIALS_FILE` points there.
- **`gho_` and `ghu_` behave differently.** An official CLI token (`gho_`, or set via `COPILOT_GITHUB_TOKEN`) is used directly with the official CLI identity — no JWT exchange. A VS Code-style token (`ghu_`) is exchanged for a short-lived Copilot JWT on the default path.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-copilot-refreshtoken --force` keeps failing (token revoked, account changed, plan changed), do a full `scorpiox-copilot-login --force` to re-bind.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile can't initialize.
