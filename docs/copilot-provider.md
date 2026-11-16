# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

Have a GitHub Copilot subscription (Individual, Business, or Enterprise — the same one the official Copilot CLI signs into) but no API key? You can use it directly in SCORPIOX CODE. Instead of metering a vendor API key, the **Copilot provider** signs you in with the same OAuth **device-code login** the official CLI uses and runs requests against the Copilot endpoint your subscription already pays for.

This page walks through the **device-code login**, how the token is stored and kept fresh automatically, how to switch accounts and models with profiles, how to drive a login without a TTY (**machine mode**), and how direct Copilot subscription access differs from the standard API-key provider.

Docs for SCORPIOX CODE @ `b978da9`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Individual, Business, or Enterprise seat you already pay for |
| **You don't have (or don't want) an OpenAI API key** | No per-token API billing — requests draw on your Copilot plan |
| **You want Claude / GPT / Gemini models through GitHub's backend** | Claude Sonnet, GPT, and Gemini model IDs all run over one Copilot sign-in |
| **Headless / over SSH** | The device-code flow works over SSH: open a link and type a code on any device, no browser needed on the box |
| **Driven by SCORPIOX BOT** | The login can be started and polled as separate commands (`--start` / `--poll`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

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
3. Come back to your terminal and wait — SCORPIOX CODE polls automatically and, on success, saves the token. You'll see:

```
Authorized!
  Login successful
  Saved to /home/you/.copilot/.credentials.json
  Also wrote /home/you/.copilot/config.json
```

A few things worth knowing about this flow:

- **The code window is about 15 minutes.** If the wait times out or the code expires, just re-run `scorpiox-copilot-login` and a fresh code is issued.
- **Never share the code.** It's a credential. Anyone who has the code and is watching the terminal can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop or phone and the terminal picks it up — this is the main reason the device-code flow is the default.

### Replace existing credentials

If you've already logged in and want to re-bind (new account, rotated token, wrong seat), overwrite with:

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
/profile copilot     # activate and persist
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
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local` (default — read from `~/.copilot/.credentials.json`), `config` (resolved from `scorpiox-env.txt`), `remote`, `ssh`, or `tcp`. Use **`local`** for a subscription you logged into with `scorpiox-copilot-login`. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | A `gho_` GitHub token (the official Copilot CLI identity). When set, SCORPIOX CODE skips the JWT exchange and talks to the CLI endpoint directly — useful when you already have a CLI token in the environment. |
| `COPILOT_MODEL` | text | *(empty)* | Model override. Takes precedence over the generic `MODEL` key. |
| `MODEL` | text | *(empty)* | Which model to run. Accepts short aliases or a full Copilot model ID (see [Choosing a model](#choosing-a-model)). Empty resolves to the built-in default. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, or `high`. Falls back to `REASONING_EFFORT`, then `OPENAI_REASONING_EFFORT`. `off` / `none` / `0` disables it. |
| `TOOLS` | bool | `1` | Include the tool-calling block in requests (on by default). |
| `THINKING` | bool | `0` | Set to `1` to default reasoning effort to `high` when no explicit effort is set. |

> **For a personal subscription you only need two keys:** `PROVIDER=copilot` and `COPILOT_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `remote` / `ssh` / `tcp` sources exist for shared or remote token setups and are not needed for a normal Copilot login.

### Where the token lives

On login, SCORPIOX CODE writes the OAuth token to `~/.copilot/.credentials.json` — a small JSON file holding the GitHub token (`gho_` for the CLI flow, `ghu_` for the VS Code variant), its type, scope, and a timestamp, with `0600` permissions. The CLI login also writes `~/.copilot/config.json` so other Copilot-aware tools can find the same token. On every request, `COPILOT_TOKEN_SOURCE=local` reads from the credentials file. No API key is involved, and no key is ever written.

---

## Token persistence and automatic refresh

The GitHub token from the device-code login is long-lived, but the short-lived Copilot **JWT** it maps to is not. You don't manage either — with `COPILOT_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** Before the JWT gets close to expiring (a 5-minute buffer ahead of its expiry), the provider re-exchanges the stored GitHub token for a fresh JWT, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries.
- **Caching.** The exchanged JWT is cached locally (`~/.copilot/.copilot-token-cache.json`) and only re-exchanged when it's actually expired — not on every single call.
- **CLI tokens skip the exchange.** If you're using a `gho_` token (`COPILOT_GITHUB_TOKEN`), there is no JWT step at all; SCORPIOX CODE talks to the endpoint directly.
- **Remote sources always fetch fresh.** In `remote` / `ssh` / `tcp` modes the token is re-fetched on every request, so there's no local expiry to worry about.
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

You can also inspect the resolved token without making a model request, for diagnostics:

```bash
scorpiox-copilot-fetchtoken -local       # read from ~/.copilot/.credentials.json
scorpiox-copilot-fetchtoken -config      # read source from scorpiox-env.txt
scorpiox-copilot-fetchtoken -ssh         # read from a remote machine via SSH
scorpiox-copilot-fetchtoken -remote      # fetch from the HTTP endpoint
scorpiox-copilot-fetchtoken -tcp         # fetch over the raw TCP socket
```

The output is a single JSON line: `{"access_token":"...","expires_at":...}`. Use it to confirm the token source resolves to the account you expect before a long run.

### Inspecting usage instead of per-token metering

Because you're on a subscription, there's no per-token metering. Use `scorpiox-copilot-usage` to see your plan, your premium-interaction (AIC) quota, and when it resets:

```bash
scorpiox-copilot-usage        # human-readable summary: plan, AIC remaining, reset date
scorpiox-copilot-usage --json # raw JSON to stdout
```

---

## Multi-account and profile switching

Many people have more than one GitHub account (personal + work, for example). The Copilot provider supports that in two layers: **multiple stored logins** and **named config profiles**.

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
/profile copilot     # persistent — writes ACTIVE_PROFILE, survives restarts
/use copilot         # session-only — gone when the session ends
/profile            # list available profiles and which is active
/profile off        # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open while a login is in flight — a background agent, a CI step, or SCORPIOX BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is byte-for-byte the original interactive behaviour.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node (and `--name` if given). |
| `--start` | Begin the login: returns the `verify_url` and `user_code`, and keeps the pending state in `~/.claude/.login-pending/` for 15 minutes. |
| `--poll` | One non-blocking check. Reports `pending` until the user approves, then exchanges and saves the tokens. |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--poll`: also write the `copilot` profile. |
| `--name <account>` | Target a named account file (`~/.copilot/accounts/<account>.json`). |

A typical machine-mode login:

```bash
# 1. Start — prints the link and the one-time code
scorpiox-copilot-login --start
#   {"ok":true,"provider":"copilot","flow":"device_code","verify_url":"https://github.com/login/device","user_code":"ABCD-EFGH","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open that link on any device, sign in, and enter the user code.

# 3. Poll until it reports done (pass --create-profile to also write the profile)
scorpiox-copilot-login --poll
#   {"ok":true,"provider":"copilot","state":"pending","interval":5}
#   {"ok":true,"provider":"copilot","state":"done","logged_in":true,"path":"/home/you/.copilot/.credentials.json","profile":"copilot","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the device-auth state lives in `~/.claude/.login-pending/copilot-<name|default>.json` (mode `0600`), so the calls can be separate processes on the same node.
- **Fifteen-minute window.** If the code hasn't been approved within 15 minutes the pending state is dropped and `--poll` fails with `expired` — run `--start` again.
- **`--poll` is non-blocking.** Each call does one check and returns; keep calling it (honouring the `interval` it reports) until it reports `logged_in`.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"copilot","error":"...","detail":"..."}`.
- **`--status` never prints tokens** — only whether a login exists and where.

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

You can pin a specific version in your profile or switch it at runtime with the `/model` command. You can also set the reasoning effort with `COPILOT_REASONING_EFFORT` (`low`, `medium`, or `high`), or switch it in-session with the `/reasoning_effort` command.

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
- **The device code expires in 15 minutes.** A timed-out login just means running it again for a fresh code; a machine-mode pending state is dropped on the same window.
- **Never share the one-time device code.** It's a credential and a well-known phishing target. Anyone who has the code (and can open the link) can bind it to your account.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Copilot account your standing default and `/use` to hop to it for a single session without writing anything.
- **`COPILOT_MODEL` wins over `MODEL`.** If you set both, `COPILOT_MODEL` is the one that takes effect.
- **A `gho_` token changes the wire.** Setting `COPILOT_GITHUB_TOKEN` switches SCORPIOX CODE to the official CLI endpoint and identity, which skips the JWT exchange. That path is what allows Claude models on every turn, including follow-up tool turns.
- **Machine mode is additive.** `--status` / `--start` / `--poll` / `--cancel` are only active when you pass one of those flags; an unflagged `scorpiox-copilot-login` is the interactive flow.
