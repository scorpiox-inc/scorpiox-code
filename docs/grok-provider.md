# Using Grok Build Subscription in SCORPIOX CODE

You have an **xAI Grok Build** subscription — the plan the official `grok` CLI signs you into — and you don't want to meter per-token against an `XAI_API_KEY`. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Grok provider** signs you in through the official Grok CLI's OAuth **device-auth** flow and sends every request to the `cli-chat-proxy.grok.com` endpoint your subscription already pays for.

This page walks through the **device-auth session login**, where the credential lives and how it refreshes itself, the configuration keys behind `PROVIDER=grok`, what a request looks like on the wire, how to juggle more than one xAI account with profiles (`/profile` and `/use`), how to check your credit window, how to inspect a login without a TTY (**machine mode**), and how direct Grok Build access differs from standard xAI API-key usage in the [OpenAI provider](openai-provider.md).

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** run `scorpiox-grok-login`, open the link on any device, sign in with the official Grok CLI — SCORPIOX CODE reads the OAuth session the CLI stores in `~/.grok/auth.json`, renews it before it expires, and every request rides your existing Grok Build subscription instead of an API bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a Grok Build subscription** | Any xAI plan the official `grok` CLI signs into with `grok login --device-auth` |
| **You don't have (or don't want) an `XAI_API_KEY`** | No pay-per-token billing — requests draw on your subscription's credit window |
| **Headless / over SSH** | The login is a device-auth flow: open the link and sign in on any device. No browser and no localhost redirect needed on the box you run from |
| **Driven by SCORPIO BOT** | The login state can be inspected with a single non-interactive command (`--status`) — see [Machine mode (no TTY)](#machine-mode-no-tty) |
| **Several xAI accounts** | One credential file per account, one profile per file, switch with `/profile` |

If you are instead calling xAI with an **API key** (`XAI_API_KEY`), or pointing at your own OpenAI-compatible server, that is the [OpenAI provider](openai-provider.md) (`PROVIDER=openai` at `api.x.ai`) — a different billing model for the same family of models. See [Grok Build subscription vs. the xAI API-key path](#grok-build-subscription-vs-the-xai-api-key-path) below.

> **Subscription, not API key.** The Grok provider authenticates with an OAuth session and draws on your account's included credits — the same weekly/monthly credit window the official Grok CLI enforces. It does **not** accept an `XAI_API_KEY` for chat and does **not** bill per token. The chat endpoint is `cli-chat-proxy.grok.com`, **not** `api.x.ai`.

---

## Step 1 — Sign in with the device-auth flow

The login is the **official Grok CLI's device-auth flow**. SCORPIOX CODE ships a helper that runs it for you and leaves a ready-to-use profile behind:

```bash
scorpiox-grok-login
```

The helper first looks for the official `grok` binary — on your `PATH`, in `/usr/local/bin`, or under `~/.grok/bin`. When it finds it, it launches the first-party flow itself:

```
Launching: grok login --device-auth
```

Do exactly what the CLI prints:

1. **Open the link** it shows, in a browser — on any device, wherever you are already signed in to xAI.
2. **Sign in** to your Grok Build account and **approve** the device.
3. Come back to the terminal and wait — the CLI completes the device exchange and saves the session credential to `~/.grok/auth.json`.

The flow is deliberately headless-friendly: no redirect server to capture, no long URL to paste back, and no browser required on the machine you run from. Approve from your laptop or phone while the terminal on your build box waits.

A few things worth knowing:

- **The code window is short.** If the device code expires before you approve, just re-run `scorpiox-grok-login` for a fresh one.
- **The code is a credential.** Anyone holding it can bind the device to your account. Never share it or paste it into a chat or ticket.
- **The profile is written either way.** Whether or not the `grok` CLI was found, `scorpiox-grok-login` writes the `grok` profile so the provider is one switch away — see [Step 2](#step-2--turn-on-the-grok-provider).

### If the Grok CLI is not installed

The helper degrades gracefully and prints what to do instead of silently failing:

```bash
# Option A — install the official CLI and let it run the device-auth flow
curl -fsSL https://x.ai/cli/install.sh | bash
grok login --device-auth

# Option B — copy an existing session from a machine you already signed in on
scp ~/.grok/auth.json root@host:~/.grok/auth.json
```

If you already have a session on another machine, copying `~/.grok/auth.json` across is enough — no re-login needed. The helper exits successfully when it finds a credential file at the expected location.

### The `grok` profile is written for you

On the way out, `scorpiox-grok-login` writes a ready-to-use **`grok`** profile:

```
Created profile: /home/you/.claude/scorpiox-env/grok.txt
  /profile grok   (or ACTIVE_PROFILE=grok)
```

The file contains the minimal working set, with the endpoint documented in comments:

```
PROVIDER=grok
MODEL=grok-4.6
GROK_MODEL=grok-4.6
GROK_TOKEN_SOURCE=local
# Token: ~/.grok/auth.json via scorpiox-grok-fetchtoken -local
# Host:  cli-chat-proxy.grok.com (OAuth) — not api.x.ai
```

If the profile already exists, it is left untouched — your edits survive re-logins. You can also create the file by hand with exactly those lines.

---

## Step 2 — Turn on the Grok provider

The provider switches on with `PROVIDER=grok`, and for a personal subscription you set `GROK_TOKEN_SOURCE=local` so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=grok
GROK_TOKEN_SOURCE=local
```

Activate the profile — or put those keys in any cascade tier of `scorpiox-env.txt`, or export them as OS environment variables — and SCORPIOX CODE connects:

```text
/profile grok      # activate this profile and persist it
/use    grok       # activate for this session only (nothing written)
```

You can also select the provider for a single run from the command line:

```bash
sx --provider grok -p 'explain this build failure'
sx --provider grok -m sonnet -p 'review this diff'
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables — the highest tier that sets a key wins. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(empty)* | Set to `grok` to activate this provider. |
| `GROK_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local` (read `~/.grok/auth.json`), `http` / `remote` (an HTTP endpoint), `ssh` (another machine), `tcp` (a token server), or `config` (resolve from the same cascade). Use **`local`** for a login you ran with `scorpiox-grok-login`. |
| `GROK_CREDENTIALS_FILE` | text | *(empty)* | Override the local credential path. Point it at a copy such as `~/.grok/accounts/work.json` to pin a profile to one login. Beats `GROK_HOME` when both are set. |
| `GROK_HOME` | env | *(empty)* | Environment variable — an alternative directory holding `auth.json`, used in preference to `$HOME`. |
| `GROK_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `GROK_TOKEN_SOURCE=http` / `remote`. |
| `GROK_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `GROK_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. Host and user are required in that mode. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | `proxy.scorpiox.net` / `9800` / *(empty)* / *(empty)* | Used only when `GROK_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. `TCP_HOST` accepts a comma-separated list for failover. |
| `GROK_MODEL` | text | *(empty)* | Model override, read before the generic `MODEL` key. |
| `MODEL` | text | *(empty)* | Which model to run: a short alias or a full `grok-*` model ID. Empty resolves to the built-in default `grok-4.6`. See [Choosing a model](#choosing-a-model). |
| `TOOLS` | bool | `1` | Include the tool definitions so the agent can run commands and edit files. |
| `THINKING` | bool | `1` | On this provider the dial is fixed: `1` requests `high` reasoning effort, `0` requests `low`. See [Reasoning Effort Control](reasoning-effort.md). |

> **For a personal subscription you only need two keys:** `PROVIDER=grok` and `GROK_TOKEN_SOURCE=local` (plus optionally a model). The `http`, `ssh`, and `tcp` sources exist for shared or remote token setups and are not part of a normal Grok Build login — but `GROK_TOKEN_SOURCE` defaults to `local` in the shipped configuration, so the credential file is what gets read.

In the config editor (`/config` in a session, or `scorpiox-config` from a shell) the Grok keys live in the **Provider & Authentication** section and appear once `PROVIDER` is `grok`: the token source as a multiple-choice entry, then `GROK_REMOTE_URL` when the source is `http`, the `GROK_SSH_*` set when it is `ssh`, the `TCP_*` set when it is `tcp`, plus `GROK_MODEL` and `GROK_CREDENTIALS_FILE`. `GROK_HOME` is an environment variable, not a cascade key — export it in the shell when you need it.

---

## Where the token lives

On login, the Grok CLI writes the OAuth session to `~/.grok/auth.json` (or `$GROK_HOME/auth.json`, or the path in `GROK_CREDENTIALS_FILE`). The file holds one or more OIDC entries keyed by issuer, each carrying:

```json
{
  "<issuer>::<client>": {
    "key": "<access token>",
    "expires_at": "2026-10-04T08:12:29.000000Z",
    "refresh_token": "...",
    "oidc_issuer": "...",
    "oidc_client_id": "..."
  }
}
```

With `GROK_TOKEN_SOURCE=local` the provider reads that file and always uses the entry with the **latest `expires_at`**, so a file that has accumulated entries from several sign-ins still resolves deterministically. If the file is missing entirely, the reader falls back to `XAI_API_KEY` from the environment — a quiet convenience that also means a stray exported key can mask a missing login. If you expect subscription billing and see API-style behaviour, check for `auth.json` before anything else.

Treat the file as what it is: a live session that can renew your account. Do not commit it, do not copy it into shared dotfile repos, and do not loosen its permissions.

---

## Token persistence and automatic refresh

A Grok Build access token is short-lived; the session in `~/.grok/auth.json` can renew it. You manage neither — with `GROK_TOKEN_SOURCE=local` the provider keeps it fresh on every request:

- **Proactive refresh before expiry.** When the stored token is close to expiring — within a five-minute buffer — the provider renews it in place: the stored refresh token is exchanged through the issuer's OIDC token endpoint (discovered from the issuer's own configuration document) and the new access token is written back to `~/.grok/auth.json`. In-flight work never hits an expired credential, and you generally sign in once per machine.
- **Reactive recovery on 401.** If a request still comes back unauthorized, the provider refreshes once and retries the request before surfacing an error. A second failure after that is reported as a permanent auth error rather than retried forever — that is your cue to sign in again.
- **Remote sources always fetch fresh.** In `http`, `ssh`, or `tcp` modes the token is re-fetched on every request and no refresh is performed locally — the endpoint owning the credential is responsible for keeping it valid.

You can also drive the refresh by hand:

```bash
scorpiox-grok-refreshtoken            # refresh only if inside the 5-minute expiry buffer
scorpiox-grok-refreshtoken --force    # always refresh
scorpiox-grok-refreshtoken --verbose  # show the resolved endpoints and timings
```

On success the tool reports the entry it renewed and the new expiry; the file is rewritten with mode `0600`, so only your account can read it.

For diagnostics you can resolve the token exactly the way the provider does, without making a model request:

```bash
scorpiox-grok-fetchtoken -local       # read ~/.grok/auth.json (or $GROK_HOME)
scorpiox-grok-fetchtoken -config      # resolve GROK_TOKEN_SOURCE from the cascade
scorpiox-grok-fetchtoken -ssh         # read a remote machine's auth.json over SSH
scorpiox-grok-fetchtoken -remote      # fetch from the HTTP endpoint
scorpiox-grok-fetchtoken -tcp         # fetch over the raw TCP socket
```

Every mode prints one JSON line — `{"access_token":"...","expires_at":...}` — so you can confirm the token source resolves to the account you expect before a long run. Treat that output as live credential material: use it for checks, not for logs.

---

## Checking your subscription usage

Because you are on a subscription there is no per-token bill to watch — there is a credit window instead. Check how full it is with:

```bash
scorpiox-grok-usage            # human-readable summary
scorpiox-grok-usage --json     # raw billing JSON
```

The summary reads like this:

```
Weekly limit          42% used  Resets in 2d 5h   2026-10-06 00:00 UTC (12:00 PM NZDT)
Prepaid credits:      $18.40
On-demand:            $0.00 / $0.00
```

You get the current window (weekly or monthly) as a percentage used, the reset time in UTC and your local time, how long until it turns over, and your prepaid balance and on-demand spend. Exactly what you want to know before kicking off a long autonomous run: whether the allowance will hold and when it resets.

Inside a session, `/usage` runs the same tool and shows the result as **Grok Usage** when the Grok provider is active. The status bar itself reports token counts per turn (`in:` / `out:` / `rsn:`), plus cached-input figures when the endpoint reports them and `5h:` / `7d:` utilization when it publishes rate-limit headers — see [Token Usage Observability](usage-observability.md) for the full field list.

You can list the models your account can reach with:

```bash
scorpiox-grok-models               # list models the xAI endpoint advertises (JSON to stdout)
scorpiox-grok-models grok-4.6      # fetch one model's detail by appending its ID
```

---

## What a request looks like

SCORPIOX CODE talks to the Grok Build surface in the OpenAI **Responses** wire format, wearing the identity of the official CLI. Nothing for you to configure; the shape is handled for you.

1. **Endpoint** — `https://cli-chat-proxy.grok.com/v1/responses`, the subscription backend. There is no URL override key: a Grok Build login talks to Grok's own endpoint, never to `api.x.ai`.
2. **Identity** — the OAuth access token as a bearer token, plus the CLI's own client headers: a `grok-shell/<version>` user-agent, matching client-version and client-identifier stamps, and the token-auth markers the real CLI sends. This is what makes the request indistinguishable from the CLI you already use.
3. **Session affinity** — a session UUID held for the provider's lifetime, sent as `x-grok-session-id`, `x-grok-conv-id`, and `x-grok-turn-idx`, and repeated as the `prompt_cache_key` body field. Consecutive turns of a conversation land on a warm prompt cache, so from the second turn on, the prompt prefix is served cached instead of re-read. The names are fixed and not configurable — see [Session Identity Headers](identity-headers.md).
4. **Body** — `stream: true` and `store: false`, your system prompt with project context as the `instructions` field, a `reasoning` block with `high` or `low` effort resolved from `THINKING`, and the tool definitions as function schemas when `TOOLS=1`.
5. **Conversation** — the history as `input` items: user turns as `input_text`, assistant turns as `output_text`, tool calls and results as `function_call` / `function_call_output` pairs, and images (attached or read with `ReadImage`) as `input_image` items with data URLs, so the model can look at a screenshot you paste.
6. **Response** — the reply always arrives as server-sent events, which the provider parses for you. Reasoning summaries are surfaced as their own thinking block, and the status bar reports reasoning tokens alongside output (`rsn:<count>`). With `USAGE_SPEED_ESTIMATE=1` SCORPIOX CODE times the first streamed token and reports `~pp` / `~tg` throughput in the status bar.

Each request gets a five-minute timeout, so long agentic turns are not cut off mid-flight. Transient failures are retried on their own — `429`, `500`, `502`, `503`, `529`, and transient `403` — up to ten attempts with an exponential backoff that doubles from one second, caps at a minute, and adds jitter so many agents do not retry in lockstep. Above the provider retries, the agent loop has its own wider retry policy for long runs; see the `AGENT_RETRY_*` keys in [Configuration and Profiles](scorpiox-env.md).

The reasoning dial is fixed here, and that is worth a word: there is no per-provider reasoning-effort key on this provider. `THINKING=1` (the shipped default) sends `high`; `THINKING=0` sends `low`, because the endpoint rejects the explicit no-reasoning value with a 400. `/reasoning_effort` has no effect while this provider is active — see [Reasoning Effort Control](reasoning-effort.md) for how the chain differs per provider.

---

## Multi-account: profiles and switching

Many people have more than one xAI account (personal plus work, for example). The Grok provider supports that in two layers: **stored logins** and **named config profiles**.

### Save more than one login

A Grok session lives in one `auth.json`, so keep a copy per account and point each profile at its own copy:

```bash
# sign in to account A, then set the file aside
scorpiox-grok-login
cp ~/.grok/auth.json ~/.grok/accounts/personal.json

# sign in to account B
scorpiox-grok-login
cp ~/.grok/auth.json ~/.grok/accounts/work.json
```

The automatic refresh always renews the file at the default location, so refresh **before** copying and re-copy after any login that changes the account. For a long-lived second account, re-copying after each login is the price of the single-file session; keep `GROK_CREDENTIALS_FILE` pointing at the copy so the profile never drifts.

### Bind each login to a profile

Now make one profile per account. Point `GROK_CREDENTIALS_FILE` at the right file so the profile always uses that account:

```
# ~/.claude/scorpiox-env/grok-personal.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/accounts/personal.json
MODEL=grok-4.6
```

```
# ~/.claude/scorpiox-env/grok-work.txt
PROVIDER=grok
GROK_TOKEN_SOURCE=local
GROK_CREDENTIALS_FILE=~/.grok/accounts/work.json
MODEL=grok-4.6
```

Profiles can live at any tier — `~/.claude/scorpiox-env/<name>.txt` for your own machine, or `.scorpiox/scorpiox-env/<name>.txt` committed to a project so the whole team gets the same account and model.

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```text
/profile grok-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use    grok-personal   # session-only — gone when the session ends
/profile                # list profiles (opens the picker where available)
/profile off            # deactivate and drop back to the base cascade
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the switch cannot complete, SCORPIOX CODE reverts to the previous profile — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

### Inspect what is actually configured

Use `scorpiox-config` to see resolved values and where each comes from (which cascade tier won):

```bash
scorpiox-config --get PROVIDER           # one resolved key
scorpiox-config --get GROK_TOKEN_SOURCE  # the token source in effect
scorpiox-config --verbose                # every key plus which tier it came from
```

`--verbose` shows the resolved `PROVIDER`, `MODEL`, and `ACTIVE_PROFILE`, so you can confirm you are pointed at the account and model you expect before a long run.

---

## Choosing a model

`GROK_MODEL` (or the generic `MODEL` when the first is empty) accepts either a short alias or a full Grok model ID. The alias mapping at this commit is:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `grok-4.6` (the provider's built-in default) |
| `opus` | `grok-4.6` |
| `sonnet` | `grok-4.6` |
| `haiku` | `grok-4.5` |
| any `grok-*` ID | passed through as-is |
| anything else | `grok-4.6` (the built-in default) |

The three Claude-style aliases exist so a shared profile can be pointed at different providers without rewriting the model line — on Grok they land on the current frontier and fast models. Any ID that starts with `grok-` is sent unchanged, which is how you run a newer model your plan exposes without editing anything else. Run `scorpiox-grok-models` to see every ID your account can reach.

> Note: the login-created `grok` profile ships with `MODEL=grok-4.6` out of the box. Change it to any ID from `scorpiox-grok-models` if you want a different default.

Set the model in your profile, or switch it at runtime with the in-session `/model` command, which takes the same values as `MODEL`. You can also pin a model for a single run with `sx -m sonnet`.

### Image generation (Grok Imagine)

The same session also drives xAI's **Grok Imagine** image API. Use `scorpiox-grok-imagegen` to generate or edit images, authenticated with the same OAuth session — or with an explicit `XAI_API_KEY` / `--token` if you prefer key billing for that call:

```bash
scorpiox-grok-imagegen --prompt "Piha Beach Lion Rock, storm opening to blue"
scorpiox-grok-imagegen --prompt "..." --output /tmp/out.png --aspect 16:9 --resolution 2k
scorpiox-grok-imagegen --prompt "add an RTX PRO 6000 on the sand" --image /tmp/piha.png
scorpiox-grok-imagegen --models        # list available Imagine models
```

Images are written to `--output` (default `/tmp/grok-imagine.png`) and the path is printed to stdout, so the tool is easy to call from an agent turn. Aspect ratio, resolution, quality, and up to three source images for edits are all flags.

---

## Remote token sources: `http`, `ssh`, `tcp`

`local` is all a personal subscription needs. The other three exist so one credential can serve a whole fleet — a build box, a container, or a machine that should never hold the credential file itself:

| Source | Where the token comes from | Keys that matter |
|--------|----------------------------|------------------|
| `local` | `~/.grok/auth.json` (or `GROK_CREDENTIALS_FILE` / `$GROK_HOME`) | none extra |
| `http` / `remote` | An HTTP endpoint returning the token JSON | `GROK_REMOTE_URL`, optional `TOKEN_HTTP_API_KEY` |
| `ssh` | Another machine's `~/.grok/auth.json`, read over SSH | `GROK_SSH_HOST`, `_USER`, optional `_PORT`, `_PASS` |
| `tcp` | A token server over a raw TCP socket | `TCP_HOST` (comma-separated for failover), `TCP_PORT`, optional `TCP_API_KEY`, optional `TCP_UPSTREAM` |

The token server speaks a small request/response protocol (`SXV1 ... TYPE=grok`) and answers with the same credential JSON the local file holds, so the provider consumes all four sources identically. In `http`, `ssh`, and `tcp` modes the token is re-fetched on every request and no refresh is performed locally — the endpoint owning the credential is responsible for keeping it valid.

> **Picking a source per run.** `sx -e GROK_TOKEN_SOURCE=local` overrides the active profile for one run — handy when a container should pull its token from the fleet's token server while your interactive profile stays `local`. You can test any source from a shell with `scorpiox-grok-fetchtoken -tcp` (or `-ssh`, `-remote`) before blaming the provider.

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open while a login is in flight — a background agent, a CI step, or SCORPIO BOT's provider flow. The login ships an **additive machine mode**: non-interactive flags that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

Grok is the one provider whose sign-in is owned by an external binary: `grok login --device-auth`. Because SCORPIOX CODE does not own that protocol, the Grok login tool's machine mode is **status-only** (flow `"terminal"`): it reports state and points a human at a terminal login. It never drives the device exchange and never writes the profile.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as one JSON line: whether a credential exists and where, whether a refresh token is stored, whether the `grok` profile exists, and whether the Grok CLI is installed. |
| `--cancel` | Drop a pending machine-mode login state (nothing is pending for a terminal flow). |
| `--start` / `--poll` / `--finish` | Recognised, but rejected with `terminal_only`: Grok signs in through the official `grok` CLI, so use a terminal login. |

A typical machine-mode check:

```bash
scorpiox-grok-login --status
# {"ok":true,"provider":"grok","display":"Grok (xAI)","flow":"terminal","path":"/home/you/.grok/auth.json","logged_in":true,"has_refresh":true,"profile":"grok","profile_exists":true,"identity":"grok CLI installed"}
```

If `logged_in` is `false`, the caller surfaces a terminal `scorpiox-grok-login` and lets the user finish the device-auth step. `--status` never prints tokens — only whether a login exists and where. As with every machine-mode tool, everything is on stdout as a single line and the caller reads the last line that starts with `{`; a failure looks like `{"ok":false,"provider":"grok","error":"...","detail":"..."}`.

---

## Grok Build subscription vs. the xAI API-key path

It is easy to confuse the two because they run the same model families. Here is the difference:

| | **Grok provider** (this page) | **xAI API-key path** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `grok` | `openai`, with `OPENAI_BASE_URL` at `api.x.ai` |
| **Authentication** | OAuth device-auth login via the official `grok` CLI (`scorpiox-grok-login`), token refreshed automatically | `XAI_API_KEY` bearer token |
| **Billing** | Your Grok Build subscription allowance (weekly / monthly credit windows) | Pay-per-token xAI API usage on your key |
| **What you need** | A Grok Build seat the `grok` CLI can sign into | An `XAI_API_KEY` and a platform account |
| **Token lifecycle** | Short-lived access token + stored refresh token, renewed for you | Static API key, nothing to refresh |
| **Chat endpoint** | `cli-chat-proxy.grok.com/v1/responses` (fixed) | Any OpenAI-compatible `/v1/chat/completions` you name |
| **Wire format** | Responses API (`input` items, `instructions`, `reasoning`) | Chat Completions, translated for you |
| **Usage visibility** | Credit-window percentage and reset time via the usage tool | Token counts only |

They run the same underlying models, but the **Grok provider bills against your existing subscription** while the **API-key path bills per token against your `XAI_API_KEY`**. The two hit different endpoints on purpose: the subscription path is `cli-chat-proxy.grok.com`, the metered path is `api.x.ai`. Rule of thumb: **you have a Grok Build seat, use `grok`. You have an API key or your own model server, use `openai`.**

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-grok-login` writes the token file and the `grok` profile; SCORPIOX CODE only uses it once `PROVIDER=grok` is active — via `/profile`, `/use`, `ACTIVE_PROFILE`, or the base cascade.
- **Two different endpoints, on purpose.** The Grok provider talks to `cli-chat-proxy.grok.com`, not `api.x.ai`. Do not point the subscription provider at the API-key host and expect subscription billing.
- **A stray `XAI_API_KEY` can mask a missing login.** If `~/.grok/auth.json` cannot be read, the local source falls back to `XAI_API_KEY` from the environment. If you expect subscription behaviour and see something else, check for a leftover exported key before blaming the login.
- **The credential file is live.** `~/.grok/auth.json` holds a session that can renew your account — keep default permissions, keep it out of backups you share, and never commit it.
- **Refreshing is automatic; re-login is the fallback.** If `scorpiox-grok-refreshtoken --force` keeps failing (expired session, revoked device, plan change), run `scorpiox-grok-login` again to re-bind.
- **Second accounts need re-copying.** Automatic refresh renews the file at the default location, not your named copy. Refresh, then re-copy to `~/.grok/accounts/<name>.json`, or export `GROK_CREDENTIALS_FILE` in the shell so the provider and the refresh helper see the same path.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Grok account your standing default and `/use` to hop to it for a single session without writing anything.
- **The reasoning dial is fixed.** `THINKING=1` sends `high`, `THINKING=0` sends `low`, and the endpoint rejects the explicit no-reasoning value — so `/reasoning_effort` has no effect on this provider. See [Reasoning Effort Control](reasoning-effort.md).
- **Subscription windows are enforced upstream.** The weekly / monthly credit limits are xAI's, not SCORPIOX CODE's. Check them with `scorpiox-grok-usage` instead of guessing, before a long autonomous run.
- **Machine mode is status-only.** Because sign-in is the external `grok login --device-auth`, `--start` / `--poll` / `--finish` report `terminal_only`. Check state with `--status` and finish the login in a terminal.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile cannot initialize.

---

## Quick reference

| Goal | What to set |
|------|-------------|
| Personal subscription, first login | `scorpiox-grok-login`, then `/profile grok` |
| Minimal keys by hand | `PROVIDER=grok` + `GROK_TOKEN_SOURCE=local` |
| Re-authenticate | `scorpiox-grok-login` (re-run; an existing profile is left untouched) |
| A second account | copy `auth.json` aside, re-login, then a profile pinning `GROK_CREDENTIALS_FILE` |
| Renew the token by hand | `scorpiox-grok-refreshtoken --force` |
| See the credit window | `scorpiox-grok-usage`, or `/usage` in-session |
| List the endpoint's models | `scorpiox-grok-models` |
| Pick a model | `MODEL=opus` (or `sonnet` / `haiku` / a full `grok-*` ID), or `/model` in-session |
| Generate images | `scorpiox-grok-imagegen --prompt "..."` |
| Fleet token source | `GROK_TOKEN_SOURCE=tcp` (or `http` / `ssh`) with its keys |
| Check a login from a script | `scorpiox-grok-login --status` |
| One-off run override | `sx --provider grok -e GROK_TOKEN_SOURCE=local -p "..."` |

Sign in once, let the refresh loop keep the session alive, and pick the profile that points at the account and model you want.

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Using the OpenAI Provider](openai-provider.md) — the key-based path to OpenAI-compatible endpoints, including `api.x.ai`.
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md) — the other subscription login on the OpenAI-wire side.
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md) — the Anthropic-side subscription login.
- [Using GitHub Copilot CLI Subscription in SCORPIOX CODE](copilot-provider.md) — the GitHub-side subscription login.
- [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md) — the Google-side subscription login.
- [Reasoning Effort Control](reasoning-effort.md) — why this provider has a fixed mapping instead of an effort key.
- [Token Usage Observability](usage-observability.md) — where the token counts and the `5h:` / `7d:` subscription windows show up.
- [Session Identity Headers](identity-headers.md) — how the Grok session UUID drives prompt-cache hits.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim on-disk record of every request, with the bearer token masked.
