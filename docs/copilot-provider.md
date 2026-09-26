# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

Have a GitHub Copilot subscription (Individual, Business, or Enterprise — the same one the official Copilot CLI signs into) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering a vendor API key, the **Copilot provider** signs you in with the same OAuth **device-code login** the official CLI uses and runs requests against the Copilot endpoint your subscription already pays for. No per-token API billing, no per-request metering.

This page walks through the **device-code login**, how the token is stored and kept fresh automatically, how to switch accounts and models with profiles, and how direct Copilot subscription access differs from the standard API-key provider.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | An Individual, Business, or Enterprise seat you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No per-token API billing — requests draw on your Copilot plan |
| **You want Claude / GPT / Gemini models through GitHub's backend** | Claude, GPT, and Gemini model IDs all run over one Copilot sign-in |
| **Headless / over SSH / in a container** | The device-code flow works over SSH: open a link and type a code on any device, no browser needed on the box |

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

Waiting for authorization
```

Do exactly what it says:

1. **Open the printed link** — `https://github.com/login/device` — in a browser, on any device where you're signed into GitHub.
2. **Enter the one-time code** shown in the terminal and approve the request.
3. Come back to your terminal and wait — SCORPIOX CODE polls automatically and, on success, saves the token to `~/.copilot/.credentials.json`.

A few things worth knowing about this flow:

- **The code window is about 15 minutes.** The terminal keeps polling every few seconds until you approve or the window runs out. If it times out, just re-run `scorpiox-copilot-login` and a fresh code is issued.
- **Never share the code.** It's a credential — anyone who has the code and the link can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up. This is the main reason the device-code flow is the default: there's no localhost redirect to capture.

On success the token is written to `~/.copilot/.credentials.json` and SCORPIOX CODE moves straight to the profile prompt in [Step 2](#step-2--turn-on-the-copilot-provider).

### Multiple accounts and overwriting

If a credential already exists, the command refuses to clobber it:

```
Credentials already exist: /home/you/.copilot/.credentials.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token. To sign in to several GitHub accounts, give each a name — each writes its own file under `~/.copilot/accounts/`:

```bash
scorpiox-copilot-login --name personal     # -> ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work         # -> ~/.copilot/accounts/work.json
```

### VS Code app variant

If your credential comes from a VS Code app sign-in rather than the CLI flow, pass `--vscode`. It uses the VS Code OAuth client and writes the same `.credentials.json` shape. Use the default CLI login unless you specifically have a VS Code-bound token.

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

Accepting it writes `~/.claude/scorpiox-env/copilot.txt` with exactly those keys. If you decline (or are on a non-interactive terminal), create it by hand:

```bash
mkdir -p ~/.claude/scorpiox-env
cat > ~/.claude/scorpiox-env/copilot.txt <<'EOF'
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
MODEL=claude-sonnet-5
EOF
```

Then activate it. The minimal set for a personal subscription is just:

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
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local` (read from `~/.copilot/.credentials.json`), `config`, `remote`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-copilot-login`. |
| `COPILOT_CREDENTIALS_FILE` | text | *(empty)* | Override the path of the local credential file. Defaults to `~/.copilot/.credentials.json`. Point it at a named account (e.g. `~/.copilot/accounts/work.json`) to pin a profile to a specific login. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | A `gho_` GitHub token (the official Copilot CLI identity). When set, SCORPIOX CODE skips the JWT exchange and talks to the CLI endpoint directly — useful when you already have a CLI token in the environment. |
| `COPILOT_MODEL` | text | `claude-sonnet-5` | Model override. Takes precedence over the generic `MODEL` key. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts short aliases or a full Copilot model ID (see [Choosing a model](#choosing-a-model)). Empty resolves to the built-in default. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort forwarded to the request: `low`, `medium`, or `high`. Falls back to `REASONING_EFFORT`, then `OPENAI_REASONING_EFFORT`. |
| `COPILOT_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `COPILOT_TOKEN_SOURCE=remote`. |
| `COPILOT_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. |
| `TOOLS` | bool | `1` | Include the tool-calling block in requests (on by default). |
| `THINKING` | bool | `0` | Set to `1` to default the reasoning effort to a higher level when no explicit effort is set. |

> **For a personal subscription you only need two keys:** `PROVIDER=copilot` and `COPILOT_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `remote` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Copilot login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth token to `~/.copilot/.credentials.json` — a small JSON file holding the GitHub token, its type, its scope, and a timestamp. On every request, `COPILOT_TOKEN_SOURCE=local` reads from that file. No API key is involved, and no key is ever written. The file is created with `0600` permissions, so only your user can read it.

---

## Token persistence and automatic refresh

The GitHub token from the device-code login is long-lived, but the short-lived Copilot **JWT** it maps to is not. You don't manage either — with `COPILOT_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** Before the JWT gets close to expiring (a 5-minute buffer ahead of its expiry), the provider re-exchanges the stored GitHub token for a fresh JWT, so in-flight work never hits an expired credential.
- **Caching.** The exchanged JWT is cached locally (`~/.copilot/.copilot-token-cache.json`) and only re-exchanged when it's actually expired — not on every single call.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries with the new token.
- **CLI tokens skip the exchange.** If you're using a `gho_` token (`COPILOT_GITHUB_TOKEN` or the CLI login), there is no JWT step at all; SCORPIOX CODE talks to the endpoint directly.
- **Manual refresh.** You can force a refresh any time:

```bash
scorpiox-copilot-refreshtoken            # refresh if the token is expired
scorpiox-copilot-refreshtoken --force    # always refresh
scorpiox-copilot-refreshtoken --verbose  # show old/new expiry
```

As long as the on-disk credential file is intact, you generally only sign in once. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always one of:

```bash
scorpiox-copilot-refreshtoken --force   # token expired, renew it
scorpiox-copilot-login --force          # account changed, re-login
```

> **Remote, SSH, and TCP sources** fetch the latest token from the configured endpoint on every request, so their "refresh" is just reading the freshest value the server holds. Only `local` relies on the on-disk credential file.

### Inspect your subscription usage

Want to see how much of your allowance is left? The usage helper reads your token and prints the live quota:

```bash
scorpiox-copilot-usage              # human-readable summary
scorpiox-copilot-usage --json       # raw JSON
```

---

## Multi-account and profile switching

Many people have more than one GitHub account (personal + work, for example). The Copilot provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-copilot-login --name personal     # -> ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work         # -> ~/.copilot/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account. Point `COPILOT_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/copilot-personal.txt
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
COPILOT_CREDENTIALS_FILE=~/.copilot/accounts/personal.json
MODEL=sonnet
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
/profile copilot-work       # persistent, writes ACTIVE_PROFILE, survives restarts
/use copilot-personal       # session-only, gone when the session ends
/profile                    # list available profiles and which is active
/profile off                # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print resolved keys (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

`--verbose` shows the resolved `COPILOT_TOKEN_SOURCE`, `MODEL`, and `ACTIVE_PROFILE`, so you can confirm you're pointed at the account you expect before a long run.

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

Set `MODEL` in your profile or switch it at runtime with the `/model` command.

---

## Copilot provider vs. the OpenAI API-key provider

It's easy to confuse the two because they can both run GPT models. Here's the difference:

| | **Copilot provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-copilot-login`) | `OPENAI_API_KEY` bearer token |
| **Billing** | Your GitHub Copilot plan allowance (monthly / premium-interaction quota) | Pay-per-token API usage (or your own server) |
| **Token source** | `COPILOT_TOKEN_SOURCE` (local / config / remote / ssh / tcp) | `OPENAI_API_KEY` (+ optional `OPENAI_BASE_URL`) |
| **Endpoint** | `api.githubcopilot.com` / `api.business.githubcopilot.com` | Any OpenAI-compatible `/chat/completions` server |
| **Best for** | People who already pay for GitHub Copilot | Pay-as-you-go API access, local or custom endpoints |

Rule of thumb: **you have a GitHub Copilot subscription, use `copilot`. You have an OpenAI API key or a self-hosted OpenAI-compatible server, use `openai`.** See [Using the OpenAI Provider](openai-provider.md) for the full API-key configuration. This page is the subscription counterpart to the [Codex provider](codex-provider.md), which does the same idea for OpenAI's ChatGPT/Codex plans.

---

## Gotchas

- **`COPILOT_TOKEN_SOURCE` should be `local` for a personal subscription.** The `remote` / `ssh` / `tcp` sources are for shared or remote token setups. Leaving a non-local source on a normal machine means SCORPIOX CODE will look for a remote token endpoint and won't find one. The login-created profile sets `COPILOT_TOKEN_SOURCE=local` for you.
- **The login command and the provider are separate.** `scorpiox-copilot-login` writes the token file and offers a profile. You still need `PROVIDER=copilot` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **A `gho_` token changes the wire.** Setting `COPILOT_GITHUB_TOKEN` (or using the CLI login, which produces a `gho_` token) switches SCORPIOX CODE to the official CLI identity and the business endpoint, which skips the JWT exchange. That path is what allows Claude models on every turn, including follow-up tool turns.
- **The device-code login is the default.** If you're over SSH or in a container, use the plain `scorpiox-copilot-login` (device code). Use `--vscode` only when you specifically have a VS Code-bound token.
- **Never share the one-time code or the credential file.** `~/.copilot/.credentials.json` (and `~/.copilot/accounts/*.json`) hold live OAuth tokens. Treat them like a password.
- **Re-login after a failed refresh.** If the account changed or the token is no longer valid, `scorpiox-copilot-login --force` re-runs the device-code flow and replaces the stored token.
