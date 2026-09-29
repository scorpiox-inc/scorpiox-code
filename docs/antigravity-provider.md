# Using Google Antigravity CLI Subscription in SCORPIOX CODE

You have a **Google Antigravity CLI** subscription (an Antigravity Starter Quota account, or any consumer Google account that the official Antigravity CLI can sign into) but you don't want to run your own Google Cloud project or pay per API token. You can use it directly in SCORPIOX CODE. Instead of metering Google Cloud API tokens, the **Antigravity providers** sign you in with the same OAuth PKCE session login the Antigravity CLI uses and route requests against the Antigravity endpoint your subscription already pays for.

There are two providers behind this one subscription, and you pick whichever model family you want:

| `PROVIDER` value | Model family | Example default |
|------------------|--------------|-----------------|
| `google_gemini` | Gemini models via Antigravity | `gemini-3.7-flash-tiered` |
| `google_claude`  | Claude models via Antigravity | `claude-sonnet-4-6` |

This page walks through the **OAuth PKCE session login**, how the token is stored and kept fresh, how to switch between accounts and models with profiles (`/profile` and `/use`), how to drive a login without a TTY (**machine mode**), and how the Antigravity subscription differs from standard Google Cloud API-key usage.

Docs for SCORPIOX CODE @ `2b0bffd`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Google Antigravity CLI subscription** | An Antigravity Starter Quota or any consumer Google account the Antigravity CLI signs into |
| **You don't run (or want to run) your own GCP project** | Consumer (Gmail) accounts bill against a shared Google-side project — no project ID of your own, no licence, no invoice |
| **You don't have a Google Cloud API key** | No pay-per-token billing — requests draw on your Antigravity usage allowance |
| **Headless / over SSH** | The login flow works over SSH: approve in a browser on any device, paste the code back |
| **Driven by SCORPIO BOT** | The login can be started and finished as separate commands (`--start` / `--finish`) with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling the Google Cloud Gemini API with a `VERTEX_API_KEY` and your own GCP project, use `PROVIDER=gemini_vertex`. The two are different billing models for overlapping models — see [Antigravity subscription vs. Google Cloud API-key usage](#antigravity-subscription-vs-google-cloud-api-key-usage) below.

> **Subscription, not API key.** The Antigravity providers authenticate with OAuth and draw on your Antigravity usage allowance. They do **not** accept a `VERTEX_API_KEY` and do **not** require a Google Cloud project of their own.

---

## Step 1 — Sign in with the OAuth PKCE session flow

The default login is the **OAuth authorization-code (PKCE) flow**, the same one the official Antigravity CLI performs. It is deliberately SSH- and headless-friendly: you approve from a browser on whatever device is already signed in, then paste the resulting authorization code back into your terminal. No browser is required on the machine you're running from.

```bash
scorpiox-antigravity-login
```

You'll see something like this:

```
  SCORPIOX GOOGLE ANTIGRAVITY LOGIN

Please open the following authorization URL in your browser:

  https://accounts.google.com/o/oauth2/auth?access_type=offline&client_id=...&code_challenge=...&code_challenge_method=S256&...

Sign in with your Google Account and approve permissions.
After approval, paste the authorization code (or the full redirect URL) below:

  Authorization Code: >
```

Do exactly what it says:

1. **Open the printed URL** in a browser — on any device, wherever you're signed into Google.
2. **Sign in** to your Google account and approve the permissions.
3. Come back to your terminal and **paste the authorization code** when prompted (the full redirect URL is fine — the state fragment is stripped automatically).

SCORPIOX CODE then exchanges the code for an access + refresh token, fetches your account profile, discovers your usage tier, and saves the credentials. On success you'll see:

```
==================================================
  Login Successful!
==================================================

  Account:    you@gmail.com
  Project:    aicode-consumers
  User Tier:  standard-tier (Standard)

  Configured Profiles:
    - /profile antigravity         (Gemini 3.7 Flash Tiered)
    - /profile antigravity-claude  (Claude Sonnet 4.6 via Google)
```

A few things worth knowing about this flow:

- **The access token is short-lived by design** — but you never manage that (see [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh)).
- **Never share the authorization code.** It is a credential. Anyone who has the code (and is watching the terminal) can bind it to your account.
- **No browser on the box? No problem.** Approve from your laptop, phone, or a kiosk and paste the code back.
- **Consumer accounts have no GCP project.** If you signed in with a plain Gmail account, the "Project" shown is a shared Google-side project ID, not a project you own. That is expected and is what makes inference work on an Antigravity Starter Quota account.

### Overwriting existing credentials

If you're already logged in, the command re-uses the stored refresh token. To force a fresh login (for example after an account change, or to re-authorize a different Google account), pass `--force`:

```bash
scorpiox-antigravity-login --force
```

### Multiple Google accounts

Antigravity does not use named account files the way some other providers do. Every account you log in with is **added to the same account list** automatically, so you can hold several Google accounts side by side and pick which one a profile talks to with `GOOGLE_ACCOUNT` (an email address). See [Multi-account and profile switching](#multi-account-and-profile-switching).

---

## Step 2 — Turn on a provider

The login command writes **two ready-to-use profiles** for you under `~/.claude/scorpiox-env/`:

- **`antigravity`** → `PROVIDER=google_gemini` (Gemini via Antigravity)
- **`antigravity-claude`** → `PROVIDER=google_claude` (Claude via Antigravity)

Each profile sets the provider, the token source, the account, the project ID, and a sensible default model. To start a session with one of them:

```text
/profile antigravity
```

or, for a single session without making it your standing default:

```text
/use antigravity-claude
```

Both commands trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

### Configuration keys

All of these can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Applies to | Notes |
|-----|------------|-------|
| `PROVIDER` | both | `google_gemini` or `google_claude` |
| `GOOGLE_TOKEN_SOURCE` | both | `local` (default for these profiles), `remote`, or `tcp` |
| `GOOGLE_ACCOUNT` | both | The Gmail address to use, when you have more than one |
| `GOOGLE_PROJECT_ID` | both | Project ID. Usually filled in for you at login; left to Google for consumer accounts |
| `GOOGLE_GEMINI_MODEL` | `google_gemini` | See [Choosing a model](#choosing-a-model) |
| `GOOGLE_CLAUDE_MODEL` | `google_claude` | See [Choosing a model](#choosing-a-model) |
| `GOOGLE_REMOTE_URL` | both | Required only when `GOOGLE_TOKEN_SOURCE=remote` |

### Where the credentials live

The login writes three things:

- **`~/.config/google-accounts/antigravity_accounts.json`** — the list of signed-in Google accounts, each with an email and a **refresh token**.
- **`~/.config/google-accounts/selected_account.txt`** — the currently selected account (the most recent login).
- **`~/.claude/scorpiox-env/antigravity.txt`** and **`~/.claude/scorpiox-env/antigravity-claude.txt`** — the two profiles described above.

The accounts file holds a live refresh token. Don't commit it, don't share it, and don't loosen its permissions.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you don't manage that. With `GOOGLE_TOKEN_SOURCE=local`, SCORPIOX CODE handles it on **every request**:

- **Proactive refresh.** When the stored access token is close to expiring, the provider refreshes it against the OAuth token endpoint using the stored refresh token and reloads the new value, so in-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized, the provider refreshes once and retries. A second failure after a refresh is reported as a permanent auth error.
- **Remote / TCP sources always fetch fresh.** In `remote` or `tcp` modes the token is re-fetched on every request (the remote server is the source of truth and may have revoked an old token), so there is no local expiry to worry about.

The refresh token (stored in `~/.config/google-accounts/antigravity_accounts.json`) is what lets the access token be renewed without signing in again. As long as that file is intact, you generally sign in once and it keeps working. If you ever see an "unauthorized" or "token expired" hint, the fix is almost always a re-login:

```bash
scorpiox-antigravity-login --force
```

You can also inspect the resolved token without making a model request, for diagnostics. This helper is shared by both Antigravity providers:

```bash
scorpiox-google-fetchtoken -config    # resolve from the active profile / scorpiox-env
```

The output is a single JSON line: `{"ok":true,"email":"...","access_token":"ya29...","project_id":"..."}`. Use it to confirm the token source resolves to the account you expect before a long run.

---

## Multi-account and profile switching

### Hold more than one Google account

Each `scorpiox-antigravity-login` adds (or refreshes) one entry in the account list. To target a specific account from a profile, set its email in `GOOGLE_ACCOUNT`:

```
PROVIDER=google_gemini
GOOGLE_TOKEN_SOURCE=local
GOOGLE_ACCOUNT=you@gmail.com
GOOGLE_PROJECT_ID=aicode-consumers
GOOGLE_GEMINI_MODEL=gemini-3.7-flash-tiered
MODEL=gemini-3.7-flash-tiered
```

If `GOOGLE_ACCOUNT` is left empty, the provider uses the selected account (`selected_account.txt`).

### Switch accounts and models in-session

```text
/profile antigravity         # Gemini via Antigravity, standing default
/profile antigravity-claude  # Claude via Antigravity, standing default
/use    antigravity           # hop to it for this session only; nothing is written
/profile                      # list profiles and show which is active
/profile off                  # deactivate the profile
```

`/profile` persists the choice in your user file; `/use` is session-only. Both trigger a live provider reload, so the backend and model change takes effect immediately. See [Configuration and Profiles](scorpiox-env.md).

### Inspect what's actually configured

```bash
scorpiox-google-quota            # readable per-model quota table
scorpiox-google-quota --json     # raw quota buckets
scorpiox-google-quota --account you@gmail.com
scorpiox-google-models           # list the models the endpoint advertises
```

The quota tool works for both `google_gemini` and `google_claude` because they share the same Google account / token infrastructure. A `remainingFraction` of `1` is full; `0` is exhausted for that window.

---

## Machine mode (no TTY)

Sometimes you can't hold a terminal open between "here's the URL" and "paste the code" — a background agent, a CI step, or SCORPIO BOT's provider flow. For that, the login tool ships an **additive machine mode**: the same PKCE flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node. |
| `--start` | Begin the login: returns the `auth_url`, and keeps the PKCE state for 15 minutes. |
| `--finish <code\|->` | Exchange the pasted code for tokens (`-` reads the code from stdin). |
| `--cancel` | Drop a pending login. |

A typical machine-mode login:

```bash
# 1. Start — prints the URL to open
scorpiox-antigravity-login --start
#   {"ok":true,"provider":"antigravity","flow":"paste_code","auth_url":"https://accounts.google.com/o/oauth2/auth?...", "expires_in":900, "hint":"Sign in with Google and approve..."}

# 2. Open that URL, sign in, copy the authorization code.

# 3. Finish — exchange the code
scorpiox-antigravity-login --finish "$CODE"
#   {"ok":true,"provider":"antigravity","logged_in":true,"path":"/home/you/.config/google-accounts/antigravity_accounts.json","profile":"antigravity"}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--finish` the PKCE verifier and `state` live in `~/.claude/.login-pending/` (mode `0600`), so the two calls can be separate processes on the same node.
- **Fifteen-minute window.** If `--finish` comes more than 15 minutes after `--start`, the pending state is dropped and the command fails with `expired` — run `--start` again.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"antigravity","error":"...","detail":"..."}`.
- **Paste-code flow, not poll.** Antigravity is a paste-code flow, so there is no `--poll`; it is `--start` then `--finish`.
- **No named accounts.** `--name` is not supported; each Google account is added to the shared account list automatically.

---

## Choosing a model

`MODEL` can be set in a profile, or switched at runtime with the `/model` command. The per-provider model keys control the same selection.

For `google_gemini` (`GOOGLE_GEMINI_MODEL`), the values available at this commit are:

`gemini-3.8-flash-high`, `gemini-3.8-flash-tiered`, `gemini-3.7-flash-tiered`, `gemini-3.6-flash-high`, `gemini-3.5-flash-lite`, `gemini-3.1-pro-high`, `gemini-3-flash`, `gemini-2.5-pro`, `gemini-2.5-flash`

For `google_claude` (`GOOGLE_CLAUDE_MODEL`), the values available at this commit are:

`claude-sonnet-4-6`, `claude-opus-4-6-thinking`

> Note: the login-created `antigravity` profile ships with `gemini-3.7-flash-tiered` and the `antigravity-claude` profile with `claude-sonnet-4-6` out of the box. Change either key in the profile to switch the default.

You can list every model the endpoint advertises with `scorpiox-google-models`.

---

## Antigravity subscription vs. Google Cloud API-key usage

It's easy to confuse the two because they both run Google models. Here's the difference:

| | **Antigravity providers** (this page) | **Google Cloud API-key usage** (`PROVIDER=gemini_vertex`) |
|---|---|---|
| `PROVIDER` value | `google_gemini` / `google_claude` | `gemini_vertex` |
| **Authentication** | OAuth PKCE session login (`scorpiox-antigravity-login`) | `VERTEX_API_KEY` bearer token |
| **Billing** | Your Antigravity usage allowance (Starter Quota / consumer account) | Pay-per-token Google Cloud API usage |
| **Google Cloud project** | None of your own — consumer accounts use a shared Google-side project | You need your own GCP project + API key |
| **Token source** | `GOOGLE_TOKEN_SOURCE` (`local` / `remote` / `tcp`) | `VERTEX_API_KEY` |
| **Best for** | People who already use the Antigravity CLI | Pay-as-you-go API access on your own project |

Rule of thumb: **you use the Antigravity CLI / have an Antigravity allowance → `google_gemini` or `google_claude`. You have your own GCP project and an API key → `gemini_vertex`.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-antigravity-login` writes the token file and the two profiles. You still need `PROVIDER=google_gemini` or `PROVIDER=google_claude` active (via `/profile`, `/use`, or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **The accounts file holds a live refresh token.** `~/.config/google-accounts/antigravity_accounts.json` can renew your account session — don't commit it, don't share it, and don't loosen its permissions.
- **Consumer accounts have no project of their own.** The "Project" shown after a Gmail login is a shared Google-side project ID. That is correct. Don't replace it with a local workspace label — a non-existent project makes Google reject every request.
- **`/profile` persists; `/use` does not.** Use `/profile` to make an Antigravity account your standing default and `/use` to hop to it for a single session without writing anything.
- **Machine mode is additive.** `--status` / `--start` / `--finish` / `--cancel` are only active when you pass one of those flags; an unflagged `scorpiox-antigravity-login` is the interactive flow. `--poll` and `--name` are not supported. Pending machine state expires after 15 minutes.

---

## Related

- [Configuration and Profiles](scorpiox-env.md)
- [Claude Code provider](claude-code-provider.md)
- [OpenAI provider](openai-provider.md)
- [Copilot provider](copilot-provider.md)
