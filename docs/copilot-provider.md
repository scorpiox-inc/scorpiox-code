# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

Have a GitHub Copilot subscription (Individual, Business, or Enterprise — the same one the official Copilot CLI signs into) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering a vendor API key, the **Copilot provider** signs you in with the same OAuth **device-code login** the official CLI uses and runs requests against the Copilot endpoint your subscription already pays for.

This page walks through the **device-code login**, how the token is stored and kept fresh automatically, how to switch accounts and models with profiles, and how direct Copilot subscription access differs from the standard API-key provider.

Source of truth: `sx_provider_copilot.c`, `scorpiox-copilot-login.c`, `scorpiox-copilot-fetchtoken.c`, `scorpiox-copilot-refreshtoken.c`, and `scorpiox-copilot-models.c` at commit `6c70ad6`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Individual, Business, or Enterprise seat you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No per-token API billing — requests draw on your Copilot plan |
| **You want Claude / GPT / Gemini models through GitHub's backend** | Claude Sonnet, GPT, and Gemini model IDs all run over one Copilot sign-in |
| **Headless / over SSH** | The device-code flow works over SSH: open a link and type a code on any device, no browser needed on the box |

If you are instead calling an OpenAI-compatible server with an API key (`OPENAI_API_KEY` and `OPENAI_BASE_URL`), use `PROVIDER=openai` instead. The two are different billing models for overlapping models — see [Copilot provider vs. the OpenAI API-key provider](#copilot-provider-vs-the-openai-api-key-provider) below.

> **Subscription, not API.** The Copilot provider authenticates with OAuth and draws on your account's plan allowance (the monthly / premium-interaction quota your Copilot seat carries). It does **not** require an `OPENAI_API_KEY` and does **not** bill you per token against an API account.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**. It is deliberately headless- and SSH-friendly: SCORPIOX CODE prints a link and a one-time code, you sign in from a browser on whatever device is already signed in, then SCORPIOX CODE polls in the background and saves the token when you're authorized. No redirect server and no pasting a callback URL.

```bash
scorpiox-copilot-login
```

You'll see something like this:

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

1. **Open the printed link** — `https://github.com/login/device` — in a browser, on any device where you're signed into GitHub.
2. **Enter the one-time code** shown in the terminal and approve the request.
3. Come back to your terminal and wait — SCORPIOX CODE polls automatically and, on success, saves the token to `~/.copilot/.credentials.json`.

A few things worth knowing about this flow:

- **The code window is about 15 minutes.** If the wait times out or the code expires, just re-run `scorpiox-copilot-login` and a fresh code is issued.
- **Never share the code.** It's a credential. Anyone who has the code and is watching the terminal can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up — this is the main reason the device-code flow is the default.

On success you'll see:

```
Authorized!

Login successful
  Saved to /home/you/.copilot/.credentials.json
```

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

By default the login writes to `~/.copilot/.credentials.json`. To keep several GitHub accounts side by side, give each one a name:

```bash
scorpiox-copilot-login --name work      # writes ~/.copilot/accounts/work.json
```

Named accounts live in `~/.copilot/accounts/<name>.json`.

### VS Code app variant

If your credential comes from a VS Code app sign-in rather than the CLI flow, pass `--vscode`. It uses the VS Code OAuth client and writes the same `.credentials.json` shape (a `ghu_` token rather than a `gho_` one). Use the default CLI login unless you specifically have a VS Code-bound token.

```bash
scorpiox-copilot-login --vscode
```

---

## Step 2 — Turn on the Copilot provider

On success, SCORPIOX CODE offers to write a ready-to-use profile:

```
Create 'copilot' config profile?
  Will create: /home/you/.claude/scorpiox-env/copilot.txt
  Contents:
    PROVIDER=copilot
    COPILOT_TOKEN_SOURCE=local
    MODEL=claude-sonnet-5

Create? [Y/n]
```

Say yes (or create the file yourself), then activate it in-session:

```
/profile copilot        # activate and persist
```

Or set the keys yourself in any cascade tier, a named profile, or as OS environment variables. The minimal set for a personal subscription is:

```
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
```

That's it. `MODEL` defaults to `claude-sonnet-5` if left empty.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `copilot` to activate this provider. |
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local` (default — read from `~/.copilot/.credentials.json`), `config` (resolved from this same config), `remote`/`http`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-copilot-login`. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | A `gho_` GitHub token (the official Copilot CLI identity). When set, SCORPIOX CODE skips the JWT exchange and talks to the CLI endpoint directly — useful when you already have a CLI token in the environment. |
| `COPILOT_MODEL` | text | *(empty)* | Model override. Takes precedence over the generic `MODEL` key. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts short aliases or a full Copilot model ID (see [Choosing a model](#choosing-a-model)). Empty resolves to the built-in default. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, or `high`. Falls back to `REASONING_EFFORT`, then `OPENAI_REASONING_EFFORT`. `off` / `none` / `0` disables it. |
| `TOOLS` | bool | `1` | Include the tool-calling block in requests (on by default). |
| `THINKING` | bool | `0` | Set to `1` to default reasoning effort to `high` when no explicit effort is set. |

> **For a personal subscription you only need two keys:** `PROVIDER=copilot` and `COPILOT_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `remote` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Copilot login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth token to `~/.copilot/.credentials.json` — a small JSON file holding the GitHub token (`gho_` for the CLI flow, `ghu_` for the VS Code variant), its type, scope, and a timestamp. On every request, `COPILOT_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written. The file is created with `0600` permissions.

---

## Token persistence and automatic refresh

The GitHub token from the device-code login is long-lived, but the short-lived Copilot **JWT** it maps to is not. You don't manage either — with `COPILOT_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** Before the JWT gets close to expiring (a 5-minute buffer ahead of its expiry), the provider re-exchanges the stored GitHub token for a fresh JWT, so in-flight work never hits an expired credential.
- **Caching.** The exchanged JWT is cached locally (`~/.copilot/.copilot-token-cache.json`) and only re-exchanged when it's actually expired — not on every single call.
- **CLI tokens skip the exchange.** If you're using a `gho_` token (`COPILOT_GITHUB_TOKEN`), there is no JWT step at all; SCORPIOX CODE talks to the endpoint directly.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-copilot-refreshtoken            # refresh if the cached JWT is expired
scorpiox-copilot-refreshtoken --force    # always re-exchange
scorpiox-copilot-refreshtoken --verbose  # show token details
```

As long as `~/.copilot/.credentials.json` is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
# 1. Refresh the JWT in place
scorpiox-copilot-refreshtoken --force

# 2. If that keeps failing, re-bind with a full login
scorpiox-copilot-login --force
```

### Inspecting usage instead of per-token metering

Because you're on a subscription, there's no per-token metering. Use `scorpiox-copilot-usage` to see your plan, your premium-interaction (AIC) quota, and when it resets:

```bash
scorpiox-copilot-usage        # human-readable summary: plan, AIC remaining, reset date
scorpiox-copilot-usage --json # raw JSON to stdout
```

---

## Multi-account and profile switching

### Store more than one login

Log in to several GitHub accounts and give each a name:

```bash
scorpiox-copilot-login --name work
scorpiox-copilot-login --name personal
```

Each writes its own file under `~/.copilot/accounts/`. Create one profile per account, each pointing `COPILOT_TOKEN_SOURCE` at `local` so it reads the right stored token.

### Switch accounts and models in-session

Switching is just switching profiles — no re-login, no restart:

```
/profile copilot        # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot            # session-only — gone when the session ends
/profile              # list available profiles and which is active
/profile off            # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

`MODEL` (or the higher-precedence `COPILOT_MODEL`) accepts either a short alias or a full Copilot model ID. The aliases at this commit resolve to:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-5` (built-in default) |
| `opus` | `claude-sonnet-5` |
| `sonnet` | `claude-sonnet-5` |
| `haiku` | `claude-sonnet-5` |
| any `claude-*` / `gpt-*` / `gemini-*` / `kimi-*` ID | passed through as-is |

Any ID that already starts with a known vendor prefix (`claude-`, `gpt-`, `gemini-`, `kimi-`) is forwarded unchanged, so you can request any model your Copilot plan exposes. Everything else falls back to the built-in default.

> Note: the login-created `copilot` profile ships with `MODEL=claude-sonnet-5` out of the box. Change it to the ID you prefer if you want a different default.

You can list every model the endpoint advertises with:

```bash
scorpiox-copilot-models              # pretty-printed model list
scorpiox-copilot-models --json       # raw JSON to stdout
```

---

## Copilot provider vs. the OpenAI API-key provider

It's easy to confuse the two because they can both run GPT models. Here's the difference:

| | **Copilot provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your GitHub Copilot plan allowance (monthly / AIC quota) | Pay-per-token API usage (or your own server) |
| **Token source** | `COPILOT_TOKEN_SOURCE` (local / config / remote / ssh / tcp) | `OPENAI_API_KEY` (+ optional `OPENAI_BASE_URL`) |
| **Endpoint** | `api.githubcopilot.com` / `api.business.githubcopilot.com` | Any OpenAI-compatible `/v1/chat/completions` server |
| **Best for** | People who already pay for GitHub Copilot | Pay-as-you-go API access, local or custom endpoints |

Rule of thumb: **you have a GitHub Copilot subscription → `copilot`. You have an OpenAI API key or a self-hosted OpenAI-compatible server → `openai`.** See [Using the OpenAI Provider](openai-provider.md) for the full API-key configuration.

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-copilot-login` writes the token file and offers a profile. You still need `PROVIDER=copilot` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The credentials file is live.** `~/.copilot/.credentials.json` holds a token that can renew your account session — don't commit it, don't share it, and don't loosen its permissions. It's written `0600` by default.
- **Named accounts live in `~/.copilot/accounts/`, not the default file.** `--name work` writes `~/.copilot/accounts/work.json`. A profile only uses it if its token source points there.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Copilot account your standing default and `/use` to hop to it for a single session without writing anything.
- **`COPILOT_MODEL` wins over `MODEL`.** If you set both, `COPILOT_MODEL` is the one that takes effect.
- **A `gho_` token changes the wire.** Setting `COPILOT_GITHUB_TOKEN` switches SCORPIOX CODE to the official CLI endpoint and identity, which skips the JWT exchange. That path is what allows Claude models on every turn, including follow-up tool turns.
