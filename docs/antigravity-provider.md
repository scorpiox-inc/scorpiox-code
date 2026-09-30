# Using Google Antigravity CLI Subscription in SCORPIOX CODE

You have a **Google Antigravity CLI** subscription (an Antigravity Starter Quota account, or any consumer Google account the official Antigravity CLI signs into) and you don't want to open a Google Cloud project, wire up billing, or meter per-token against an API key. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Antigravity providers** sign you in with the same **OAuth 2.0 PKCE** session login the Antigravity CLI performs and route every request through the Cloud Code Assist backend your subscription already pays for.

One sign-in unlocks two providers, so you can pick the model family per session or per profile:

| `PROVIDER` value | Model family | Default model |
|------------------|--------------|---------------|
| `google_gemini` | Gemini models via Antigravity | `gemini-3.8-flash-high` |
| `google_claude` | Claude models via Google's Cloud Code Assist surface | `claude-sonnet-4-6` |

This page walks through the **session login**, where the token is stored, how refresh works (and why there is nothing to manage), how to switch accounts and models with profiles (`/profile` and `/use`), how to drive a login without a TTY (**machine mode**), and how the Antigravity subscription differs from standard Google Cloud API-key usage.

Docs for SCORPIOX CODE @ `13253cf`.

> **The whole idea in one line:** run `scorpiox-antigravity-login`, approve in a browser, paste the code back — SCORPIOX CODE stores a refresh token, writes two ready-to-use profiles, and every subsequent request refreshes its own access token from that stored credential. No Google Cloud project, no API key, no per-token bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have an Antigravity CLI subscription** | An Antigravity Starter Quota account, or any consumer Google account the official Antigravity CLI signs into |
| **You don't run (or want to run) your own GCP project** | Consumer (Gmail) accounts bill against a shared Google-side project — no project of your own, no licence to buy, no invoice |
| **You want Claude models through Google** | `google_claude` runs Claude Sonnet and Claude Opus through the same sign-in as Gemini |
| **Headless / over SSH** | The login works over SSH: approve in a browser on any device, paste the code back into the terminal |
| **Driven by SCORPIO BOT** | The login splits into separate `--start` / `--finish` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |

If you are instead calling Google with a **Google Cloud API key** and your own GCP project (`PROVIDER=gemini_vertex`), that is a different billing model for overlapping models — see [Antigravity subscription vs. Google Cloud API-key usage](#antigravity-subscription-vs-google-cloud-api-key-usage) below.

> **Subscription, not API key.** The Antigravity providers authenticate with OAuth and draw on your Antigravity usage allowance. They do **not** accept a Google Cloud API key, and they do **not** bill per token against a project you own.

---

## The two providers behind one login

The Antigravity sign-in produces one OAuth refresh token, and both providers consume it. You do not log in twice, and you do not keep two credentials in sync.

| | `google_gemini` | `google_claude` |
|---|---|---|
| Runs | Gemini models through Cloud Code Assist | Claude models through Cloud Code Assist |
| Model key | `GOOGLE_GEMINI_MODEL` | `GOOGLE_CLAUDE_MODEL` |
| Login-created profile | `antigravity` | `antigravity-claude` |
| Default model | `gemini-3.8-flash-high` | `claude-sonnet-4-6` |

Both profiles set `GOOGLE_TOKEN_SOURCE=local` and pin the same `GOOGLE_ACCOUNT`, so switching between them is a profile switch, not a re-login.

---

## Step 1 — Sign in with the OAuth PKCE session flow

The login is the **OAuth authorization-code flow with PKCE** (Proof Key for Code Exchange) — the same flow the official Antigravity CLI performs, using the Antigravity client identity. SCORPIOX CODE generates the PKCE verifier and challenge for you, prints one authorization URL, and waits for you to paste the resulting authorization code back. It is deliberately SSH- and headless-friendly: no browser is required on the machine you run from.

```bash
scorpiox-antigravity-login
```

You will see something like this:

```
  SCORPIOX GOOGLE ANTIGRAVITY LOGIN

Please open the following authorization URL in your browser:

  https://accounts.google.com/o/oauth2/auth?access_type=offline&client_id=...&code_challenge=...&code_challenge_method=S256&state=...

Sign in with your Google Account and approve permissions.
After approval, paste the authorization code (or the full redirect URL) below:

  Authorization Code: >
```

Do exactly what it says:

1. **Open the printed URL** in a browser — on any device, wherever you are signed into Google.
2. **Sign in** to your Google account and **approve** the requested permissions.
3. The browser lands on a Google callback page. **Copy the whole address** (or just the `code=` parameter) and **paste it back** into the terminal.

SCORPIOX CODE then exchanges the code for tokens (the PKCE verifier proves the request came from the same terminal that started the flow), fetches your account profile, discovers your usage tier, onboards the account onto Cloud Code Assist, and saves everything. On success you will see:

```
==================================================
  Login Successful!
==================================================

  Account:    you@gmail.com
  Project:    aicode-consumers
  User Tier:  standard-tier (Standard)

  Configured Profiles:
    • /profile antigravity         (Gemini 3.7 Flash Tiered)
    • /profile antigravity-claude  (Claude Sonnet 4.6 via Google)
```

A few things worth knowing about this flow:

- **Paste the code or the whole URL.** The prompt accepts either. If you paste the full redirect address, SCORPIOX CODE extracts the `code=` parameter itself and ignores the rest.
- **Never share the authorization code.** It is a credential, it is single-use, and anyone holding it while you are mid-login can bind it to their own request.
- **No browser on the box? No problem.** Approve from your laptop, phone, or a kiosk and paste the code back into the SSH session.
- **The access token is short-lived by design** — you never handle it. See [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh).

### Re-running the login

The login is idempotent: running it again refreshes the entry for that Google account in the account list and rewrites both profiles to point at it. Use `--force` when you want to re-authorize deliberately (for example after an account change, or after Google asks you to verify again):

```bash
scorpiox-antigravity-login --force
```

### Multiple Google accounts

Antigravity does not use named credential files the way some other providers do. Every account you log in with is **added to the same account list**, so you can hold several Google accounts side by side and pick which one a profile talks to with `GOOGLE_ACCOUNT`. See [Multi-account and profile switching](#multi-account-and-profile-switching).

### Account verification gate

A Google account that has not passed Google's own verification check comes back flagged `VALIDATION_REQUIRED`. SCORPIOX CODE stops the login rather than writing a half-broken profile, and prints the exact link to open:

```
  Account verification required
  -----------------------------
  Verify your account to continue.

  Open this URL in a browser signed in as you@gmail.com:

  https://accounts.google.com/...

  Then re-run: scorpiox-antigravity-login --force
```

This is a Google-side account check, not a SCORPIOX CODE error, and it cannot be worked around from the client. Open the link in a browser signed in as that account, complete the verification, then re-run the login with `--force`.

---

## Step 2 — Turn on an Antigravity provider

The login writes **two ready-to-use profiles** under `~/.claude/scorpiox-env/`:

```
# ~/.claude/scorpiox-env/antigravity.txt
PROVIDER=google_gemini
GOOGLE_TOKEN_SOURCE=local
GOOGLE_ACCOUNT=you@gmail.com
GOOGLE_PROJECT_ID=aicode-consumers
GOOGLE_GEMINI_MODEL=gemini-3.7-flash-tiered
MODEL=gemini-3.7-flash-tiered
```

```
# ~/.claude/scorpiox-env/antigravity-claude.txt
PROVIDER=google_claude
GOOGLE_TOKEN_SOURCE=local
GOOGLE_ACCOUNT=you@gmail.com
GOOGLE_PROJECT_ID=aicode-consumers
GOOGLE_CLAUDE_MODEL=claude-sonnet-4-6
MODEL=claude-sonnet-4-6
```

Activate one and SCORPIOX CODE connects:

```text
/profile antigravity          # Gemini via Antigravity — persist it
/profile antigravity-claude   # Claude via Google — persist it
/use    antigravity           # this session only, nothing written
```

If you would rather configure it by hand, the minimal set for a personal subscription is two lines in any cascade tier:

```
PROVIDER=google_gemini
GOOGLE_TOKEN_SOURCE=local
```

`GOOGLE_ACCOUNT` and `GOOGLE_PROJECT_ID` are conveniences, not requirements — see [Configuration keys](#configuration-keys) for what each one does when left empty.

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `google_gemini` or `google_claude` to activate an Antigravity provider. |
| `GOOGLE_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local`, `remote`, or `tcp`. Use **`local`** for a login you ran with `scorpiox-antigravity-login`. |
| `GOOGLE_ACCOUNT` | text | *(empty)* | Which stored Google account to use, by email. Leave empty to use the first account in the file — but prefer pinning the email, because new logins are appended after it. |
| `GOOGLE_PROJECT_ID` | text | *(empty)* | Companion GCP project ID. Leave empty for a consumer (Gmail) account: the shared Google-side project is used automatically. |
| `GOOGLE_ACCOUNTS_FILE` | text | *(empty)* | Override the path of the local account file. Defaults to `~/.config/google-accounts/antigravity_accounts.json`. Point it at a copy to pin a profile to a specific snapshot of the account list. |
| `GOOGLE_GEMINI_MODEL` | text | `gemini-3.8-flash-high` | Model for `google_gemini`. Accepts a full `gemini-*` ID (passed through as-is) or a short alias. |
| `GOOGLE_CLAUDE_MODEL` | text | `claude-sonnet-4-6` | Model for `google_claude`. Accepts a full `claude-*` ID or a short alias. |
| `GOOGLE_GEMINI_MAX_RETRIES` | text | `5` | Retry attempts for transient errors (`429`, `500`, `502`, `503`, `529`) on the Gemini side. `0` disables retrying. |
| `GOOGLE_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GOOGLE_TOKEN_SOURCE=remote`. |

> **For a personal subscription you need two keys:** `PROVIDER=google_gemini` (or `google_claude`) and `GOOGLE_TOKEN_SOURCE=local`. The `remote` and `tcp` token sources exist for shared token services — a server that hands out tokens to many machines — and are not part of a normal Antigravity login.

### Where the credentials live

The login writes four things:

- **`~/.config/google-accounts/antigravity_accounts.json`** — the account list. Each entry holds an `email` and a **refresh token**. This is the file both providers read on every request.
- **`~/.config/google-accounts/selected_account.txt`** — the email of the most recent login.
- **`~/.claude/scorpiox-env/antigravity.txt`** — the Gemini profile, rewritten on every login.
- **`~/.claude/scorpiox-env/antigravity-claude.txt`** — the Claude profile, rewritten on every login.

The accounts file holds a live refresh token: it can renew your Google session without you. Do not commit it, do not share it, and do not loosen its permissions.

### Why the project ID says `aicode-consumers`

Consumer (Gmail) accounts have no Google Cloud project of their own, so Google bills Antigravity usage against a **shared Google-side project**. That shared project ID is what SCORPIOX CODE writes into the profile and what it sends on every request.

Do not replace it with a project name you invented — in particular not with the local workspace label the official CLI keeps on disk, which is a local cache value and is never a real project. Sending a project that does not exist makes Google reject every request with a misleading "no valid license" error. If `GOOGLE_PROJECT_ID` is empty, SCORPIOX CODE falls back to the shared consumer project on its own, which is exactly what you want on a plain Google account.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you never manage it. With `GOOGLE_TOKEN_SOURCE=local`, the provider refreshes the token **on every request**:

- **Refresh before every call.** The provider reads the stored refresh token from `antigravity_accounts.json`, performs a refresh-token grant against Google's OAuth endpoint, and uses the fresh access token for that request. Nothing is cached long enough to expire mid-session, so in-flight work never hits a stale credential.
- **Project resolution on the same path.** If `GOOGLE_PROJECT_ID` is not set, the token step also resolves the companion project from Google and uses the shared consumer project as the final fallback.
- **Remote / TCP sources always fetch fresh.** In `remote` or `tcp` modes the token is fetched from the token service on every request — the service is the source of truth and may have rotated or revoked a token.
- **Re-login is the fallback.** As long as `antigravity_accounts.json` is intact, you generally sign in once and it keeps working. If a session keeps coming back unauthorized — a revoked grant, a password change, a verification gate — re-bind the account:

```bash
scorpiox-antigravity-login --force
```

You can also resolve the token without making a model request, for diagnostics. The helper is shared by both Antigravity providers and reads `GOOGLE_TOKEN_SOURCE` from the active profile or config cascade:

```bash
scorpiox-google-fetchtoken -config    # resolve the way the provider will
```

The output is a single JSON line:

```
{"ok":true,"email":"you@gmail.com","access_token":"ya29...","project_id":"aicode-consumers"}
```

Run it before a long run to confirm the token source resolves to the account and project you expect. (The access token in that output is live — treat the line as a secret and do not paste it anywhere.)

---

## Multi-account and profile switching

### Hold more than one Google account

Each `scorpiox-antigravity-login` adds one entry to the account list (or refreshes the entry that already exists for that email). The login also rewrites both profiles so they pin the account you just authorized.

To point a profile at a specific stored account, set `GOOGLE_ACCOUNT` to that email:

```
PROVIDER=google_gemini
GOOGLE_TOKEN_SOURCE=local
GOOGLE_ACCOUNT=work@gmail.com
GOOGLE_GEMINI_MODEL=gemini-3.8-flash-high
MODEL=gemini-3.8-flash-high
```

Leave `GOOGLE_ACCOUNT` empty and the provider uses the **first** entry in the account file. Because new logins are appended after the existing entries, that is the oldest account, not the newest — which is why the login-written profiles always pin the email explicitly, and why a hand-written profile should do the same.

### One profile per account, per family

The two login-written profiles cover the common case. For more accounts, make one profile per account and family and pin each one:

```
# ~/.claude/scorpiox-env/ag-personal-gemini.txt
PROVIDER=google_gemini
GOOGLE_TOKEN_SOURCE=local
GOOGLE_ACCOUNT=you@gmail.com
GOOGLE_GEMINI_MODEL=gemini-3.8-flash-high
MODEL=gemini-3.8-flash-high
```

```
# ~/.claude/scorpiox-env/ag-work-claude.txt
PROVIDER=google_claude
GOOGLE_TOKEN_SOURCE=local
GOOGLE_ACCOUNT=work@gmail.com
GOOGLE_CLAUDE_MODEL=claude-opus-4-6-thinking
MODEL=claude-opus-4-6-thinking
```

### Switch accounts and models in-session

With those profiles in place, switching accounts or model families is just switching profiles — no re-login, no restart:

```text
/profile ag-work-claude     # persistent — writes ACTIVE_PROFILE, survives restarts
/use    ag-personal-gemini  # session-only — gone when the session ends
/profile                    # list profiles and show which is active
/profile off                # deactivate and drop back to the base cascade
```

`/profile` and `/use` both trigger a live provider reload, so the new account, backend, and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what is actually configured

Use `scorpiox-config` to see resolved values and where each one comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, GOOGLE_TOKEN_SOURCE, GOOGLE_ACCOUNT, MODEL, ACTIVE_PROFILE, …) with their source tier
```

`--verbose` shows the resolved `PROVIDER`, `GOOGLE_TOKEN_SOURCE`, and `MODEL`, so you can confirm you are pointed at the account and family you expect before a long run.

### Check your Antigravity quota

Both providers run against the same per-model quota Google enforces on the account, so one tool covers both:

```bash
scorpiox-google-quota            # readable per-model quota table
scorpiox-google-quota --json     # raw quota buckets
scorpiox-google-quota --account you@gmail.com
```

You will see a row per model with the remaining fraction (`100.00%` is full, `0.00%` is exhausted for that window), the token type, and the reset time — so you know when the window turns over before you kick off a long run. Inside SCORPIOX CODE, the in-session `/usage` command runs this same tool under the hood and shows it as **Google Quota** when an Antigravity provider is active.

---

## Choosing a model

`MODEL` in the profile (or the per-provider key) selects the model. Both providers accept a short alias or a full model ID, and resolve it before the request is sent.

### `PROVIDER=google_gemini` (`GOOGLE_GEMINI_MODEL`)

| Value | Resolves to |
|-------|-------------|
| *(empty)* | `gemini-3.8-flash-high` (provider default) |
| `opus` / `pro` / `sonnet` / `flash` / `haiku` | `gemini-3.8-flash-high` — every Claude-style alias lands on the same Gemini model |
| any `gemini-*` ID | passed through as-is |

Pass a full `gemini-*` ID whenever you want a specific model — for example `gemini-3.7-flash-tiered`, the default the login writes into the profile. Values available at this commit include `gemini-3.8-flash-high`, `gemini-3.8-flash-tiered`, `gemini-3.7-flash-tiered`, `gemini-3.6-flash-high`, `gemini-3.5-flash-lite`, `gemini-3.1-pro-high`, `gemini-3-flash`, `gemini-2.5-pro`, and `gemini-2.5-flash`.

### `PROVIDER=google_claude` (`GOOGLE_CLAUDE_MODEL`)

| Value | Resolves to |
|-------|-------------|
| *(empty)* | `claude-sonnet-4-6` (provider default) |
| `opus` / `pro` | `claude-opus-4-6-thinking` |
| `sonnet` / `flash` / `haiku` | `claude-sonnet-4-6` |
| any `claude-*` ID | passed through as-is |

The login-written `antigravity-claude` profile ships `claude-sonnet-4-6`. Switch it to `claude-opus-4-6-thinking` for heavier work.

Set the model in your profile, or switch it at runtime with the `/model` command, which takes the same values as `MODEL`.

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open between "here is the URL" and "paste the code" — a background agent, a CI step, or SCORPIO BOT's provider flow. The login ships an **additive machine mode**: the same PKCE flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line. |
| `--start` | Begin the login: returns the `auth_url`, and keeps the PKCE state for 15 minutes. |
| `--finish <code\|->` | Exchange the pasted code for tokens (`-` reads the code from stdin). |
| `--cancel` | Drop a pending login. |
| `--poll` | Recognised, but rejected with `wrong_flow`: Antigravity is a paste-code flow, not a polling flow. |
| `--name <account>` | Not supported — each Google account is added to the shared account list automatically. |

A typical machine-mode login:

```bash
# 1. Start — prints the URL to open
scorpiox-antigravity-login --start
# {"ok":true,"provider":"antigravity","flow":"paste_code","auth_url":"https://accounts.google.com/o/oauth2/auth?...","expires_in":900,"hint":"Sign in with Google and approve..."}

# 2. Open that URL in any browser, sign in, approve, copy the code.

# 3. Finish — exchange the code
scorpiox-antigravity-login --finish "$CODE"
# {"ok":true,"provider":"antigravity","state":"done","logged_in":true,"path":"/home/you/.config/google-accounts/antigravity_accounts.json","identity":"you@gmail.com","profile":"antigravity","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--finish` the PKCE verifier and `state` live in `~/.claude/.login-pending/` (mode `0600`), so the two calls can be separate processes on the same node.
- **Fifteen-minute window.** If `--finish` comes more than 15 minutes after `--start`, the pending state is dropped and the command fails with `expired` — run `--start` again for a fresh challenge.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"antigravity","error":"...","detail":"..."}`.
- **`--finish` does the whole account setup.** It runs the same flow as an interactive login — profile fetch, tier discovery, onboarding, profile writes — so a completed `--finish` leaves the machine in exactly the state an interactive login would.
- **`--status` tells you where you stand.** It reports `logged_in`, whether a refresh token is stored (`has_refresh`), whether a login is mid-flight (`pending`), and whether the `antigravity` profile exists. Use it to decide whether a machine needs a login at all.

---

## Antigravity subscription vs. Google Cloud API-key usage

It is easy to confuse the two because they both run Google models. Here is the difference:

| | **Antigravity providers** (this page) | **Google Cloud API key** (`PROVIDER=gemini_vertex`) |
|---|---|---|
| `PROVIDER` value | `google_gemini` / `google_claude` | `gemini_vertex` |
| **Authentication** | OAuth PKCE session login (`scorpiox-antigravity-login`) | Google Cloud API key (`VERTEX_API_KEY`) |
| **Billing** | Your Antigravity usage allowance (Starter Quota / consumer account) | Pay-per-token against your own GCP billing account |
| **Google Cloud project** | None of your own — consumer accounts use a shared Google-side project | You need your own GCP project and billing account |
| **Token lifecycle** | Short-lived access token, refreshed from the stored refresh token on every request | Static API key, no refresh |
| **Model families** | Gemini and Claude-via-Google | Gemini via the Vertex AI publisher endpoint |
| **Best for** | People who already use the Antigravity CLI and want their subscription to pay for it | GCP workloads that need dedicated quotas and per-token accounting |

Rule of thumb: **you have an Antigravity / Google subscription, use `google_gemini` or `google_claude`. You have a Google Cloud project and an API key, use `gemini_vertex`.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-antigravity-login` writes the account file and the two profiles; it does not switch your session. Activate `antigravity` or `antigravity-claude` with `/profile` (or set `PROVIDER` yourself) before SCORPIOX CODE uses it.
- **`GOOGLE_TOKEN_SOURCE=local` still requires a login.** The value means "read the refresh token from `~/.config/google-accounts/antigravity_accounts.json`". If you have never logged in, that file does not exist and every request fails with a token error. Run the login first.
- **Leave `GOOGLE_PROJECT_ID` empty for a plain Google account.** A consumer (Gmail) account has no project of its own; SCORPIOX CODE falls back to the shared consumer project automatically. Writing a made-up project name — especially the official CLI's local workspace label — is what produces the misleading "no valid license" error on the first prompt.
- **Pin `GOOGLE_ACCOUNT` in hand-written profiles.** An empty `GOOGLE_ACCOUNT` selects the *first* entry in the account file, and new logins are appended after it. If you add a second account and it seems ignored, the profile is still talking to the first one.
- **The authorization code is single-use and a credential.** Never share it, and never paste it into a ticket or a chat. If the login stalls or the code is rejected, just run `scorpiox-antigravity-login` again for a fresh challenge.
- **Verification failures are Google's, not yours.** A `VALIDATION_REQUIRED` gate means Google wants that account verified in a browser. Open the printed link, verify, re-run with `--force`.
- **Refreshing is automatic; re-login is the fallback.** If requests keep coming back unauthorized, re-bind the refresh token with `scorpiox-antigravity-login --force`.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile cannot initialize.
- **The accounts file is the crown jewel.** `~/.config/google-accounts/antigravity_accounts.json` can renew your Google session. Keep default permissions, keep it out of backups you share, and never commit it.

---

## See also

- [Configuration and Profiles](scorpiox-env.md)
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md)
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md)
- [Using GitHub Copilot CLI Subscription in SCORPIOX CODE](copilot-provider.md)
- [Using Grok Build Subscription in SCORPIOX CODE](grok-provider.md)
- [Direct Anthropic API & Custom Endpoints](anthropic-provider.md)
