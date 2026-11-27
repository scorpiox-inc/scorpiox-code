# Using Your Google Antigravity CLI Subscription in SCORPIOX CODE

You have a Google AI subscription — AI Pro, or even a plain consumer "Antigravity Starter Quota" account — and you want SCORPIOX CODE to use it, not a separately billed API key. Or you already use the official Antigravity CLI and you want your agent traffic to go through the same subscription, on the same account, with the same model list.

This page is the verified path for exactly that: one interactive login, then SCORPIOX CODE talks to the Antigravity subscription endpoint directly, with tokens refreshed automatically in the background.

Docs for SCORPIOX CODE @ `41b090b`.

> **The whole idea in one line:** run one login command, sign in with your Google account, and from then on `PROVIDER=google_gemini` or `PROVIDER=google_claude` works against your subscription — no GCP project, no API key, no per-request billing setup.

---

## What "Antigravity" means here

"Antigravity" is Google's consumer-facing AI subscription, the one behind the Antigravity CLI (`agy`). It is not a GCP product: a plain Gmail account with a subscription — or even the free starter quota — has **no GCP project of its own**. Google bills the traffic against a shared consumer-side project, and the login flow resolves which one that is for your account automatically.

That is the important contrast, and it is why this path exists at all:

| | Antigravity subscription (`google_gemini` / `google_claude`) | Standard Vertex / API-key path (`gemini_vertex` / `gcloud_vertex`) |
|---|---|---|
| Auth | OAuth 2.0 + PKCE sign-in with your Google account | `VERTEX_API_KEY`, or a GCloud application-default-credentials file |
| Billing | Your AI subscription quota (5-hour and weekly windows) | Your GCP billing account, per-token |
| Project setup | None — resolved for you at login | You must have a GCP project and, for Vertex, a location |
| Model access | The models your subscription tier unlocks, fetched at login | Whatever your project/region serves |
| Token lifetime | Access token refreshed automatically from a stored refresh token | Key does not expire (or ADC refresh handled separately) |

Both are first-class providers in SCORPIOX CODE and share the same session model, tool calling, and profiles. You pick the one that matches how you are already paying.

---

## Quick start

```bash
# 1. One-time login (opens a URL for you to authorize in a browser)
scorpiox-antigravity-login

# 2. Pick a mode. Login already wrote ready-made profiles:
sx
```

Inside the session:

```
/profile antigravity          # your subscription, Gemini models
/profile antigravity-claude   # your subscription, Claude models
/profile                     # back to your base config
```

That is the entire setup. What the login did for you, in order:

1. Generated a PKCE challenge and printed an authorization URL.
2. You opened it, signed in with your Google account, approved the scopes, and pasted back the authorization code (or the whole redirect address — both work).
3. SCORPIOX CODE exchanged the code for tokens, confirmed who you are, and discovered the companion project for your account — onboarding your tier if the account had not been used through this path yet.
4. Saved your refresh token to `~/.config/google-accounts/antigravity_accounts.json` and recorded your account as the selected one in `selected_account.txt` next to it.
5. Wrote ready-made profiles (see [Profiles](#profiles-one-command-per-mode)) that point at your subscription with the newest models your account actually has.

If you have more than one Google account, run the login once per account. Accounts accumulate in the same JSON file, and a profile can pin a specific one with `GOOGLE_ACCOUNT=<email>`.

---

## The keys that matter

Everything lives in your configuration cascade (see [Configuration and Profiles](scorpiox-env.md) for where that file lives and what wins). The keys these providers read:

| Key | Required | What it does |
|---|---|---|
| `PROVIDER=google_gemini` or `PROVIDER=google_claude` | Yes | Selects the Antigravity subscription provider. Gemini models or Claude models, both on the same subscription. |
| `GOOGLE_TOKEN_SOURCE` | defaults to `local` | `local` reads `~/.config/google-accounts/antigravity_accounts.json` (what login writes). `remote` fetches from `GOOGLE_REMOTE_URL`. `tcp` fetches over a raw socket. |
| `GOOGLE_ACCOUNT` | No | Which Google account to use when several are stored. Empty means the first stored account. |
| `GOOGLE_PROJECT_ID` | No | Companion project. Login fills this into the profiles it writes; a plain Gmail account has no project of its own and falls back to Google's shared consumer project. |
| `GOOGLE_GEMINI_MODEL` | in practice, yes | The model id, e.g. `gemini-3.8-flash-high`. Short aliases (`opus`, `sonnet`, `haiku`, `pro`) resolve to a sensible default model for you. |
| `GOOGLE_CLAUDE_MODEL` | in practice, yes | The model id, e.g. `claude-sonnet-4-6`. |
| `GOOGLE_ACCOUNTS_FILE` | No | Override the credentials file location (default `~/.config/google-accounts/antigravity_accounts.json`). |
| `GOOGLE_REMOTE_URL` | only for `remote` | Your token service endpoint, e.g. `https://token.scorpiox.net/google`. |
| `GOOGLE_GEMINI_MAX_RETRIES` | No | Retries for transient errors (429, 500, 502, 503, 529). Default `5`; set `0` to disable. |

A profile the login writes looks like this, so you can see the shape:

```
PROVIDER=google_gemini
GOOGLE_TOKEN_SOURCE=local
GOOGLE_ACCOUNT=you@gmail.com
GOOGLE_PROJECT_ID=aicode-consumers
GOOGLE_GEMINI_MODEL=gemini-4-neon-high
MODEL=gemini-4-neon-high
```

Only the keys it names override your base configuration; everything else keeps falling through as usual.

---

## How the token refresh works

Access tokens are short-lived. You should never see yourself exchange them — and you do not:

- At session start (and whenever a token expires mid-session), the provider reads the stored refresh token for your account from `antigravity_accounts.json` and performs the OAuth `refresh_token` grant against Google's token endpoint.
- A fresh access token is used for the rest of the session. The refresh token itself is not consumed — it stays valid, so a machine that sits idle for weeks still signs back in transparently on next use.
- The credentials file is plain JSON: a list of `{email, refresh_token}` entries. Back it up like any secret, and `chmod 600` it if several users share a machine.

`GOOGLE_TOKEN_SOURCE=remote` exists for fleets: run one token service that does the refresh on behalf of many clients, and point `GOOGLE_REMOTE_URL` at it. The provider then expects that service to answer with `{"ok": true, "access_token": "...", "project_id": "..."}`.

---

## Profiles: one command per mode

Login does not just save credentials — it writes profiles into your `scorpiox-env/` directory, one per model mode your account unlocks:

| Profile | Provider | Model |
|---|---|---|
| `antigravity` | `google_gemini` | the newest Gemini model your account has |
| `antigravity-claude` | `google_claude` | the newest Claude model your account has |
| `antigravity-opus` | `google_claude` | Opus, only if your tier includes it |
| `antigravity-argon` | `google_gemini` | Gemini 4 Argon, only if your tier includes it |

The login step checks your account's actual model list rather than hard-coding names — a Pro account gets its Pro models, a Free account gets its. Re-run the login (with `--force` if your model list changed) to refresh the profiles.

Switching inside a session — no restart, no file editing:

```
/profile antigravity         # load the profile and switch to it
/use antigravity-claude      # session-only switch, writes no file
/profile                    # back to your base config
```

That is the same mechanism as in [Configuration and Profiles](scorpiox-env.md): a profile overrides only the keys it names.

---

## Models you can use

`/model` in-session switches the model for the session. What resolves where:

- **Explicit ids pass through.** `gemini-3.8-flash-high`, `gemini-4-neon-high`, `claude-sonnet-4-6` — use whatever your account lists.
- **Short aliases resolve for you.** On `google_gemini`: `opus`, `sonnet`, `haiku`, `pro`, `flash` all resolve to the default flash model; `gemini-4-neon`, `gemini-4-argon`, `gemini-4-hydrogen` resolve to their tiered variants. On `google_claude`: `opus`/`pro` to Opus, `sonnet`/`flash`/`haiku` to Sonnet.

Two helpers keep you honest about what your subscription actually serves:

```bash
scorpiox-google-models            # list every model your account can use
scorpiox-google-models --claude   # ...only the Claude models
scorpiox-google-quota             # quota per window (5-hour, weekly)
```

And inside the session, `/usage` opens the same quota view as a popup.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `no refresh_token for account ... in antigravity_accounts.json` | Your stored credentials do not include the account the profile names. Run `scorpiox-antigravity-login` for that account, or clear `GOOGLE_ACCOUNT` in the profile. |
| `no refresh_token` even though the file exists | The file has an entry for your email but the `refresh_token` field is empty — the original login completed before Google issued one. Re-run `scorpiox-antigravity-login --force`. |
| `401` / "invalid license" style rejections from the subscription endpoint | Usually a stale access token or a wrong companion project. The provider auto-refreshes, but if it persists: re-run the login (`--force`), which re-resolves the companion project and re-onboards your tier. |
| Login says "login took too long" / expired | The PKCE state expires after a while. Start the login again. |
| `UNSUPPORTED_CLIENT` or a model that 404s | The endpoint keys off the client identity it expects. SCORPIOX CODE sends the correct one; if you are routing through a custom token service, make sure it does not strip or rewrite the client headers. |
| Models your tier should have are missing | Run `scorpiox-google-models` to see the real list, and re-run the login so the profiles pick up the right defaults. |
| Wrong account selected with several stored | Set `GOOGLE_ACCOUNT=<email>` in the profile, or re-run the login for that account (it becomes the selected one). |

---

## What you get

One browser sign-in, then your AI subscription is just another provider: the same session, the same tool calling, the same profiles, the same `/usage` — billed against the quota you already pay for, refreshed in the background, and switchable to your API-key provider with a single command.

*Docs for SCORPIOX CODE @ `41b090b`.*
