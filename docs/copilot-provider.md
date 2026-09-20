# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

Already paying for GitHub Copilot (Copilot Pro / Pro+ / Business / Enterprise) but you'd rather not juggle API keys? You can use that subscription directly in SCORPIOX CODE. The **Copilot provider** signs you in with a normal GitHub OAuth **device-code login** and runs requests against the same backend your Copilot subscription already pays for — no metering of API tokens.

This page walks through the **device-code login**, how the token is stored and kept fresh, how to switch accounts, and how the Copilot provider differs from the standard [OpenAI API-key provider](openai-provider.md).

Source of truth: `scorpiox-copilot-login.c`, `scorpiox-copilot-refreshtoken.c`, `scorpiox-copilot-fetchtoken.c`, `sx_provider_copilot.c`, and `scorpiox-config.c` at commit `5fd054b`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Pro, Pro+, Business, or Enterprise you already pay for |
| **You don't have (or don't want) an API key** | No pay-per-token billing — requests draw on your subscription |
| **Headless / over SSH** | The default login flow needs no browser on the same machine |

If you are instead calling an OpenAI-compatible API with an API key (`OPENAI_API_KEY` + `OPENAI_BASE_URL`), use `PROVIDER=openai` — see [Using the OpenAI Provider](openai-provider.md).

> **Subscription, not API.** The Copilot provider authenticates with a GitHub OAuth token and uses your Copilot account's usage allowance. It does **not** accept an `OPENAI_API_KEY` and does **not** bill per token. The two are different billing models for the same underlying models.

---

## Step 1 — Sign in with the device-code flow

The default login is the **device-code flow**, deliberately SSH- and headless-friendly: no browser is required on the machine you're running from, and nothing is pasted back into the terminal.

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

### Overwriting existing credentials

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: ~/.copilot/.credentials.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token:

```bash
scorpiox-copilot-login --force
```

### Other login options

```bash
scorpiox-copilot-login --name <account>   # save to ~/.copilot/accounts/<account>.json
scorpiox-copilot-login --vscode          # VS Code app client (ghu_ / .credentials.json only)
scorpiox-copilot-login --help            # show usage
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

Then activate it — persistently or for the session only. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for how profiles resolve across tiers.

---

## Step 2 — Turn on the Copilot provider

Signing in saves the token; you still need to tell SCORPIOX CODE to use it. Set `PROVIDER=copilot`. For a local subscription that's `COPILOT_TOKEN_SOURCE=local`, which reads the file you just created.

```
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
MODEL=claude-sonnet-5
```

Then activate the profile — persistently or for the session only:

```
/profile copilot     # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot         # session-only — gone when the session ends
```

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

On login, SCORPIOX CODE writes the GitHub OAuth token to `~/.copilot/.credentials.json` (mode `0600`, owner-read-only). With `COPILOT_TOKEN_SOURCE=local`, each request reads from that file. No API key is involved, and no key is ever written.

When the stored token is a VS Code token (`ghu_...`), SCORPIOX CODE exchanges it for a short-lived Copilot JWT and caches that in `~/.copilot/.copilot-token-cache.json`. Both files hold live credentials — keep them owner-read-only.

### Which credential file is read

With `COPILOT_TOKEN_SOURCE=local` (and no `COPILOT_CREDENTIALS_FILE` override), the token helper looks in this order and uses the first file that yields a token:

1. `~/.copilot/config.json` — official Copilot CLI token (`gho_...`)
2. `~/.copilot/.credentials.json` — VS Code / device-code login token (`ghu_...`)
3. `~/.config/github-copilot/hosts.json`
4. `~/.config/github-copilot/apps.json`

So the file written by `scorpiox-copilot-login` is found automatically, and an existing Copilot CLI or VS Code credential in one of the other locations is picked up too. On Windows, `~/.config` resolves to `%LOCALAPPDATA%`.

---

## Token persistence and automatic refresh

The GitHub token from the device-code flow is long-lived for VS Code tokens (`ghu_...`), but the per-request Copilot JWT is short-lived by design — and you don't manage that. SCORPIOX CODE handles refresh automatically:

- **Proactive refresh.** When the cached JWT is within 5 minutes of its `expires_at`, the provider re-runs the token exchange and reloads it, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-copilot-refreshtoken            # refresh if the token is expired
scorpiox-copilot-refreshtoken --force    # always refresh
scorpiox-copilot-refreshtoken --verbose  # show old/new tokens
```

Because the GitHub token is stored (not a throwaway refresh token), you generally sign in only once. As long as `~/.copilot/.credentials.json` is intact, refreshes happen behind the scenes. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-copilot-refreshtoken --force   # JWT expired — renew it
scorpiox-copilot-login --force          # token revoked / account changed — re-login
```

> **Official CLI tokens (`gho_...`) skip the exchange.** If your credential (or `COPILOT_GITHUB_TOKEN`) is a `gho_...` token, SCORPIOX CODE uses it directly as the bearer with the official CLI identity. There is no JWT exchange and no expiry to track.

---

## Multi-account: profiles and `scorpiox-config`

Many people have more than one GitHub account (personal + work, for example). The Copilot provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

By default, `scorpiox-copilot-login` writes to `~/.copilot/.credentials.json`. To keep several accounts side by side, give each a name:

```bash
scorpiox-copilot-login --name personal     # → ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work         # → ~/.copilot/accounts/work.json
```

Each `--name <account>` login saves to `~/.copilot/accounts/<account>.json` instead of the default `.credentials.json`.

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
MODEL=claude-sonnet-5
```

### Switch accounts in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```
/profile copilot-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot-personal      # session-only — gone when the session ends
/profile                  # list available profiles and which is active
/profile off              # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `COPILOT_TOKEN_SOURCE`, `COPILOT_MODEL`, and `ACTIVE_PROFILE`, so you can confirm you're pointed at the account you expect before a long run.

---

## Choosing a model

`COPILOT_MODEL` (or the generic `MODEL` when `COPILOT_MODEL` is empty) accepts either a full Copilot model ID or a short alias. The mapping at this commit is:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-5` (default) |
| `opus` | `claude-sonnet-5` |
| `sonnet` | `claude-sonnet-5` |
| `haiku` | `claude-sonnet-5` |
| any `claude-*` / `gpt-*` / `gemini-*` / `kimi-*` ID | passed through as-is |
| anything else | `claude-sonnet-5` (default) |

Set `COPILOT_MODEL` in your profile or switch it at runtime with the `/model` command. You can also list what your account can actually call with `scorpiox-copilot-models`.

---

## Copilot provider vs. the OpenAI API-key provider

It's easy to confuse the two because they both talk to the same families of models. Here's the difference:

| | **Copilot provider** (this page) | **OpenAI provider** ([openai-provider.md](openai-provider.md)) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | GitHub OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your GitHub Copilot subscription allowance | Pay-per-token API usage (or self-hosted, no billing) |
| **Token source** | `COPILOT_TOKEN_SOURCE` (local file / remote / ssh / tcp) | `OPENAI_BASE_URL` + `OPENAI_API_KEY` |
| **Endpoint** | GitHub Copilot backend (`/chat/completions`) | Any OpenAI-compatible `/v1/chat/completions` |
| **Default model** | `claude-sonnet-5` | `default` (server decides) |
| **Best for** | People who already pay for GitHub Copilot | Self-hosted servers, Azure, Together, Groq, raw OpenAI API |

Rule of thumb: **you have a Copilot subscription → `copilot`. You have an API key or a self-hosted endpoint → `openai`.**

---

## Gotchas

- **`COPILOT_TOKEN_SOURCE` defaults to `local`.** Unlike the Codex provider (which defaults to `tcp`), the Copilot provider looks for a local credential file by default — which is exactly what `scorpiox-copilot-login` writes. If you set it to `remote` / `ssh` / `tcp` without configuring that source, SCORPIOX CODE won't find a token.
- **The login command and the provider are separate.** `scorpiox-copilot-login` only writes the token file. You still need `PROVIDER=copilot` active (via a profile or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The device code expires in 15 minutes.** Run the login again if you time out. Never share the code — it's a credential.
- **`.credentials.json` is owner-read-only (`0600`).** Don't loosen the permissions; it holds a live token that stands in for your account.
- **Named accounts live in `~/.copilot/accounts/`, not `.credentials.json`.** `--name work` writes `~/.copilot/accounts/work.json`. A profile only uses it if `COPILOT_CREDENTIALS_FILE` points there.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-copilot-refreshtoken --force` keeps failing (token revoked, account or plan changed), do a full `scorpiox-copilot-login --force` to re-bind.
- **`gho_` vs `ghu_` tokens behave differently.** A `gho_...` token (official CLI) is used directly with no JWT exchange. A `ghu_...` token (VS Code / device-code) is exchanged for a short-lived JWT that SCORPIOX CODE caches and refreshes. Both work; you don't choose which — it's detected from the token.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile can't initialize.
