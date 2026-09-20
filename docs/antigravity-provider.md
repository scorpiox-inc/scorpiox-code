# Using Google Antigravity CLI Subscription in SCORPIOX CODE

Have a Google Antigravity CLI subscription (the free "Antigravity Starter Quota" that ships with a Google account) but no Google Cloud project or API key? You can use it directly in SCORPIOX CODE. Instead of metering Google Cloud API tokens, the **Antigravity providers** sign you in with your normal Google account and run requests against the same Cloud Code Assist backend your subscription already draws on.

This page walks through the **OAuth 2.0 PKCE login**, how the token is stored and kept fresh, how to switch between the two Antigravity models (Gemini and Claude-via-Google), and how Antigravity subscription access differs from the standard Google Cloud API-key provider.

Source of truth: `scorpiox-antigravity-login.c`, `scorpiox-google-fetchtoken.c`, `sx_provider_google_gemini.c`, `sx_provider_google_claude.c`, and `sx_provider_gemini_vertex.c` at commit `5fd054b`.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Google Antigravity CLI subscription** | The free Starter Quota that comes with a Google account |
| **You don't have (or don't want) a Google Cloud API key** | No GCP project, no billing account, no pay-per-token |
| **You want Claude models through Google's backend** | Claude Sonnet via the same Antigravity sign-in |

If you are instead calling Google's Vertex AI / AI platform with a Google Cloud API key (`VERTEX_API_KEY`), use `PROVIDER=gemini_vertex` — see [Antigravity vs. the Vertex API-key provider](#antigravity-vs-the-vertex-api-key-provider).

> **Subscription, not API.** The Antigravity providers authenticate with OAuth and use your account's usage tier. They do **not** accept a `VERTEX_API_KEY` and do **not** bill per token against your GCP project. The two are different billing models for overlapping models.

---

## The two Antigravity providers

A single Antigravity sign-in unlocks **two** `PROVIDER` values, because the same OAuth token feeds both backends. SCORPIOX CODE writes one profile for each when you log in:

| `PROVIDER` | What it runs | Login-created profile | Default model |
|------------|--------------|-----------------------|---------------|
| `google_gemini` | Gemini models via Cloud Code Assist | `/profile antigravity` | `gemini-3.7-flash-tiered` |
| `google_claude` | Claude models via Google's backend | `/profile antigravity-claude` | `claude-sonnet-4-6` |

Both profiles share `GOOGLE_TOKEN_SOURCE=local` and the same stored Google account — so one login covers both.

---

## Step 1 — Sign in with the OAuth PKCE flow

Log in with the dedicated command:

```bash
scorpiox-antigravity-login
```

This is a **PKCE** (Proof Key for Code Exchange) OAuth 2.0 authorization-code flow. SCORPIOX CODE generates the cryptographic challenge for you and prints a single authorization URL. Do exactly what it asks:

1. **Open the printed URL** in a browser on any device.
2. **Sign in with your Google account** and approve the requested permissions.
3. Google redirects to a callback. **Copy the authorization code** from the redirect (or the full redirect URL) and **paste it back into the terminal** when prompted.

Once you paste the code, SCORPIOX CODE exchanges it for an access + refresh token, fetches your account email, discovers your usage tier and companion project, and saves everything automatically. On success you'll see:

```
  Account:    you@gmail.com
  Project:    aicode-consumers
  User Tier:  standard-tier (...)

  Configured Profiles:
    /profile antigravity          (Gemini 3.7 Flash Tiered)
    /profile antigravity-claude   (Claude Sonnet 4.6 via Google)
```

A few things worth knowing about this flow:

- **Paste the code or the full URL.** The prompt accepts either the bare authorization code or the whole redirect URL — SCORPIOX CODE extracts the `code=` parameter from a URL for you.
- **Never share the authorization code.** It's a credential and is single-use. Anyone with the code can bind it to your account.
- **Re-run with `--force` to re-login.** If a credential already exists, the command refuses to clobber it. Pass `--force` to sign in again and replace the stored refresh token.

### Account verification gate

A Google account that hasn't passed Google's own verification check is flagged `VALIDATION_REQUIRED`. SCORPIOX CODE will stop the login and print the exact verification URL to open:

```
  Account verification required
  -----------------------------
  ...
  Open this URL in a browser signed in as you@gmail.com:
  https://accounts.google.com/...
  Then re-run: scorpiox-antigravity-login --force
```

This is a Google-side check, not a SCORPIOX CODE error. Open the link, complete verification in the browser, then re-run the login with `--force`.

---

## Step 2 — Turn on an Antigravity provider

Signing in saves the token **and** writes two ready-to-use profiles. Activate one in-session:

```
/profile antigravity          # Gemini (PROVIDER=google_gemini)
/profile antigravity-claude   # Claude via Google (PROVIDER=google_claude)
```

Or set the keys yourself in any cascade tier, a named profile, or as OS environment variables. The minimal set for a personal subscription is:

```
PROVIDER=google_gemini
GOOGLE_TOKEN_SOURCE=local
```

That's it. `GOOGLE_ACCOUNT`, `GOOGLE_PROJECT_ID`, and the model are already filled in by the login-created profiles.

### Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | `claude_code` | Set to `google_gemini` or `google_claude` to activate an Antigravity provider. |
| `GOOGLE_TOKEN_SOURCE` | choice | `local` | Where to get the OAuth token: `local`, `remote`, or `tcp`. Use **`local`** for an account you logged into with `scorpiox-antigravity-login`. |
| `GOOGLE_ACCOUNT` | text | *(empty)* | Which Google account to use. If empty, the most recently logged-in account is used. |
| `GOOGLE_PROJECT_ID` | text | *(empty)* | Companion GCP project. Empty falls back to the shared consumer project — leave it empty for a plain Google account. |
| `GOOGLE_ACCOUNTS_FILE` | text | *(empty)* | Override the credentials file path. Defaults to `~/.config/google-accounts/antigravity_accounts.json`. |
| `GOOGLE_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GOOGLE_TOKEN_SOURCE=remote`. |
| `GOOGLE_GEMINI_MODEL` | text | *(empty)* | Gemini model for `PROVIDER=google_gemini`. Empty resolves to the built-in default. Accepts short names (`opus`, `sonnet`, `haiku`) or a full `gemini-*` ID. |
| `GOOGLE_CLAUDE_MODEL` | text | *(empty)* | Claude model for `PROVIDER=google_claude`. Empty resolves to `claude-sonnet-4-6`. Accepts short names (`opus`, `sonnet`, `haiku`) or a full `claude-*` ID. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` | text | *(empty)* | Used only when `GOOGLE_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. |

> **For a personal subscription you only need two keys:** `PROVIDER=google_gemini` (or `google_claude`) and `GOOGLE_TOKEN_SOURCE=local` — the login-created profiles already set both. The `remote` and `tcp` sources exist for shared or remote token setups and are not needed for a normal Antigravity login.

### Where the credentials live

On login, SCORPIOX CODE writes:

- **`~/.config/google-accounts/antigravity_accounts.json`** — an array of logged-in accounts, each with its `email` and `refresh_token`. This is the file the providers read on every request.
- **`~/.config/google-accounts/selected_account.txt`** — the account most recently logged in (used when `GOOGLE_ACCOUNT` is empty).
- **`~/.claude/scorpiox-env/antigravity.txt`** and **`~/.claude/scorpiox-env/antigravity-claude.txt`** — the two ready-to-use profiles.

The file holds a live refresh token that renews your account. Keep the default permissions; don't loosen them.

---

## Token persistence and automatic refresh

The OAuth access token is short-lived by design — but you don't manage that. With `GOOGLE_TOKEN_SOURCE=local`, the provider handles it on **every request**:

- **Automatic refresh per request.** Before each call the provider reads the stored `refresh_token` from `antigravity_accounts.json` and performs a `refresh_token` grant against Google's token endpoint, so a fresh access token is always used. You never copy-paste a token by hand.
- **Re-login is the fallback.** As long as `antigravity_accounts.json` is intact you generally sign in once. If you ever see an "unauthorized" or "token expired" hint and it keeps recurring, re-bind the account:

```bash
scorpiox-antigravity-login --force
```

You can also inspect the resolved token without making a model request, for diagnostics:

```bash
scorpiox-google-fetchtoken -config    # read GOOGLE_TOKEN_SOURCE from scorpiox-env.txt
scorpiox-google-fetchtoken -remote    # fetch from the HTTP endpoint
scorpiox-google-fetchtoken -tcp       # fetch over the raw TCP socket
```

The output is a JSON line: `{"ok":true,"email":"...","access_token":"...","project_id":"..."}`. Use it to confirm the token source resolves to the account you expect before a long run.

---

## Multi-account and profile switching

Many people have more than one Google account (personal + work, for example). The Antigravity providers support that in two layers: **multiple stored accounts** and **named config profiles**.

### Store more than one login

Each `scorpiox-antigravity-login` run appends to the account array in `antigravity_accounts.json`. To pin a specific account to a profile, set `GOOGLE_ACCOUNT` in that profile to the email you want:

```
GOOGLE_ACCOUNT=work@gmail.com
```

If `GOOGLE_ACCOUNT` is empty, the most recently logged-in account (the one in `selected_account.txt`) is used.

### Switch models and accounts in-session

Switching is just switching profiles — no re-login, no restart:

```
/profile antigravity        # Gemini (persistent — writes ACTIVE_PROFILE, survives restarts)
/profile antigravity-claude # Claude via Google
/use antigravity            # session-only — gone when the session ends
/profile                    # list available profiles and which is active
/profile off                # deactivate
```

`/profile` and `/use` both trigger a live provider reload, so the new backend and model take effect immediately. If the new profile fails to initialize, SCORPIOX CODE reverts to the previous one. See [Configuration Cascade and Environment Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what's actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config            # open the interactive config editor
scorpiox-config --verbose  # print key values (PROVIDER, GOOGLE_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
```

---

## Choosing a model

Both Antigravity providers accept a short alias or a full model ID, resolved to a concrete backend model:

`PROVIDER=google_gemini` (`GOOGLE_GEMINI_MODEL`):

| Value | Resolves to |
|-------|-------------|
| `opus` / `pro` | `gemini-3.1-pro-preview` |
| `sonnet` / `flash` | `gemini-3-flash-preview` |
| `haiku` | `gemini-3.1-flash-lite-preview` |
| any `gemini-*` ID | passed through as-is |
| *(empty)* | `gemini-3.1-pro-preview` |

`PROVIDER=google_claude` (`GOOGLE_CLAUDE_MODEL`):

| Value | Resolves to |
|-------|-------------|
| `opus` / `pro` | `claude-opus-4-6-thinking` |
| `sonnet` / `flash` | `claude-sonnet-4-6` |
| `haiku` | `claude-sonnet-4-6` |
| any `claude-*` ID | passed through as-is |
| *(empty)* | `claude-sonnet-4-6` |

Set the model in your profile, or switch it at runtime with the `/model` command. The login-created profiles ship `gemini-3.7-flash-tiered` and `claude-sonnet-4-6` respectively.

---

## Antigravity vs. the Vertex API-key provider

It's easy to confuse the two because they both reach Google models. Here's the difference:

| | **Antigravity** (this page) | **Vertex AI** (`PROVIDER=gemini_vertex`) |
|---|---|---|
| **Authentication** | OAuth PKCE login (`scorpiox-antigravity-login`) | Google Cloud API key (`VERTEX_API_KEY`) |
| **Billing** | Your Antigravity subscription tier | Pay-per-use against your GCP billing account |
| **Endpoint** | Cloud Code Assist (subscription backend) | `aiplatform.googleapis.com` |
| **Models** | Gemini *and* Claude-via-Google | Gemini via Vertex AI |
| **Best for** | People who already have a Google/Antigravity subscription | GCP workloads already on a Google Cloud billing project |

Rule of thumb: **you have an Antigravity/Google subscription → `google_gemini` / `google_claude`. You have a Google Cloud API key and a GCP project → `gemini_vertex`.**

---

## Gotchas

- **`GOOGLE_TOKEN_SOURCE` is `local` by default here, but you still need a login.** The value means "read from `~/.config/google-accounts/antigravity_accounts.json`." If you haven't run `scorpiox-antigravity-login`, that file doesn't exist and you'll get `Failed to fetch Google access token`.
- **The login command and the provider are separate.** `scorpiox-antigravity-login` writes the credentials and the two profiles. You still need an Antigravity profile active (via `/profile` or `ACTIVE_PROFILE`) for SCORPIOX CODE to use it.
- **Leave `GOOGLE_PROJECT_ID` empty for a plain Google account.** A consumer (Gmail) account has no project of its own; SCORPIOX CODE falls back to the shared consumer project automatically. Writing a made-up project name is what causes the misleading "no valid license" error on the first prompt.
- **One login, two providers.** The same stored token feeds both `google_gemini` and `google_claude`. You don't log in twice.
- **The authorization code is single-use and a credential.** Never share it. If the login times out or the code is rejected, just run `scorpiox-antigravity-login` again.
- **Verification failures are Google's, not yours.** A `VALIDATION_REQUIRED` gate means Google wants you to open a link and verify the account. Open it, then re-run with `--force`.
- **Refreshing is automatic, but re-login is the fallback.** If requests keep coming back unauthorized, do a full `scorpiox-antigravity-login --force` to re-bind the refresh token.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile can't initialize.
