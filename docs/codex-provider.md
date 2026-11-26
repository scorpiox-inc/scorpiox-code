# Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE

You have an **OpenAI Codex / ChatGPT** subscription — a ChatGPT Plus, Pro, Business, or Enterprise seat that the official Codex CLI signs into — and you don't want to meter per-token against an `OPENAI_API_KEY`. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Codex provider** signs you in with the same OAuth device-code flow the Codex CLI performs and sends every request to the Codex endpoint your subscription already pays for.

This page walks through the **device-code login**, where the token lives and how it refreshes itself, the configuration keys behind `PROVIDER=codex`, what the request looks like on the wire, how to juggle multiple accounts with profiles (`/profile` and `/use`), how to check your usage windows, how to drive a login without a TTY (**machine mode**), and how a Codex subscription differs from standard API-key usage in the [OpenAI provider](openai-provider.md).

Docs for SCORPIOX CODE @ `0cd528b`.

> **The whole idea in one line:** run `scorpiox-codex-login`, open a link on any device, enter the one-time code — SCORPIOX CODE stores an OAuth access + refresh token pair exactly where the Codex CLI keeps its own, refreshes it before it expires, and every request rides your existing ChatGPT/Codex subscription instead of an API bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a ChatGPT / Codex subscription** | ChatGPT Plus, Pro, Business, or Enterprise — any account the Codex CLI signs into |
| **You don't have (or don't want) an OpenAI API key** | No pay-per-token billing — requests draw on your subscription allowance |
| **Headless / over SSH** | The login is a device-code flow: open the link and enter the code on any device. No browser and no localhost redirect needed on the box you run from |
| **Driven by SCORPIO BOT** | The login splits into separate `--start` / `--poll` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |
| **Several OpenAI accounts** | One named credential file per account, one profile per file, switch with `/profile` |

If you are instead calling the OpenAI API with a **key**, or pointing at your own OpenAI-compatible server (llama.cpp, vLLM, Ollama, Azure), that is the [OpenAI provider](openai-provider.md) (`PROVIDER=openai`) — a different billing model for the same family of models. See [Codex subscription vs. the OpenAI API-key provider](#codex-subscription-vs-the-openai-api-key-provider) below.

> **Subscription, not API key.** The Codex provider authenticates with OAuth and draws on your account's usage allowance — the session and weekly windows your ChatGPT plan enforces. It does **not** accept an `OPENAI_API_KEY`, and it does **not** bill per token. In fact no key is ever written: the stored credential file carries an explicit `"OPENAI_API_KEY": null`.

---

## Step 1 — Sign in with the device-code flow

The default login is the **OAuth device-code flow**, the same one the official Codex CLI performs. It is deliberately SSH- and headless-friendly: you open a link, sign in to your ChatGPT account on whatever device you already use, enter a one-time code, and SCORPIOX CODE picks the result up on its own. There is no localhost redirect to capture and no long URL to paste — nothing long-lived leaves the terminal, and no browser is required on the machine you run from.

```bash
scorpiox-codex-login
```

You will see something like this:

```
scorpiox-codex-login v...

Requesting device code...

Follow these steps to sign in with ChatGPT:

1. Open this link in your browser and sign in to your account
   https://auth.openai.com/codex/device

2. Enter this one-time code (expires in 15 minutes)
   XXXX-XXXX

Device codes are a common phishing target. Never share this code.

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** in a browser — on any device, wherever you are signed into ChatGPT.
2. **Sign in** and **enter the one-time code** shown in your terminal.
3. Come back to your terminal. SCORPIOX CODE is already polling; the moment you approve it exchanges the authorization code for tokens and saves them.

The credentials land in `~/.codex/auth.json` (mode `0600`), and the login then offers to write a ready-to-use profile — see [Step 2](#step-2--turn-on-the-codex-provider). A few things worth knowing about this flow:

- **The code is one-time and expires in 15 minutes.** If you do not approve in time, the login stops; run `scorpiox-codex-login` again for a fresh code.
- **Never share the code.** It is a credential: anyone who holds it while you are waiting can bind the login to their own account. The terminal prints that warning for good reason.
- **No browser on the box? No problem.** Approve from your laptop or phone; the terminal you are polling picks it up automatically.
- **Polling is polite.** Checks run every few seconds (the server names the interval; five seconds is typical) and stop on their own at the 15-minute deadline.

### Browser fallback

Some accounts have the device-code endpoint disabled. The login detects that — `device code login is not enabled for this Codex server` — and tells you what to do instead:

```bash
scorpiox-codex-login --browser
```

This is the classic redirect/paste flow: it prints an authorization URL, and when the browser runs on the same machine the `localhost:1455/auth/callback` redirect is captured automatically. Otherwise you paste the resulting redirect URL (or just the code) back into the terminal. Same tokens, same file, same profile offer.

### Overwriting existing credentials

If a credential file already exists, the login refuses to clobber it:

```
Credentials already exist: /home/you/.codex/auth.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token — after an account change, a plan change, or a refresh that can no longer be salvaged.

### Named accounts

By default the login writes to `~/.codex/auth.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-codex-login --name personal     # -> ~/.codex/accounts/personal.json
scorpiox-codex-login --name work         # -> ~/.codex/accounts/work.json
```

Names are limited to letters, digits, `-`, `_`, and `.`, up to 64 characters. A named file is only used by SCORPIOX CODE when a profile points `CODEX_CREDENTIALS_FILE` at it — see [Multi-account](#multi-account-profiles-and-switching).

### Creating the `codex` profile

At the end of a successful login SCORPIOX CODE offers to write a ready-made profile (the prompt is skipped when stdin is not a TTY):

```
Create 'codex' config profile?
  Will create: /home/you/.claude/scorpiox-env/codex.txt
  Contents:
    PROVIDER=codex
    CODEX_TOKEN_SOURCE=local
    MODEL=gpt-5.5

Create? [Y/n]
```

Say yes and the profile is written to `~/.claude/scorpiox-env/codex.txt`, ready to activate with `/profile codex`. You can also create the file by hand with exactly those three lines. In machine mode, add `--create-profile` to `--poll` to write it without prompting.

---

## Step 2 — Turn on the Codex provider

The provider switches on with `PROVIDER=codex`. For a personal subscription you also set `CODEX_TOKEN_SOURCE=local`, so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
```

Activate the profile — or put those keys in any cascade tier of `scorpiox-env.txt`, or export them as OS environment variables — and SCORPIOX CODE connects:

```text
/profile codex      # activate this profile and persist it
/use    codex      # activate for this session only (nothing written)
```

You can also select the provider for a single run from the command line:

```bash
sx --provider codex -p 'explain this build failure'
sx --provider codex -m sonnet -p 'review this diff'
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables — the highest tier that sets a key wins. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `codex` to activate this provider. |
| `CODEX_TOKEN_SOURCE` | choice | `tcp` | Where the OAuth token comes from: `local`, `http`, `ssh`, or `tcp`. Use **`local`** for a login you ran with `scorpiox-codex-login`. |
| `CODEX_CREDENTIALS_FILE` | text | *(empty)* | Override the local credential path (default `~/.codex/auth.json`). Point it at a named account such as `~/.codex/accounts/work.json` to pin a profile to one login. |
| `MODEL` | text | *(unset)* | Which model to run: a short alias or a full model ID. See [Choosing a model](#choosing-a-model). |
| `CODEX_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, `high`, `max`, or `off`. `REASONING_EFFORT` and `OPENAI_REASONING_EFFORT` are read as fallbacks — see [Reasoning Effort Control](reasoning-effort.md). |
| `CODEX_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `CODEX_TOKEN_SOURCE=http`. |
| `CODEX_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `CODEX_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. Host and user are required in that mode. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | `proxy.scorpiox.net` / `9800` / *(empty)* / *(empty)* | Used only when `CODEX_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. `TCP_HOST` accepts a comma-separated list for failover. |
| `TOKEN_HTTP_API_KEY` | text | *(empty)* | Bearer key sent to an HTTP token endpoint, when it requires one. |
| `TOOLS` | bool | `1` | Include the tool definitions so the agent can run commands and edit files. |
| `THINKING` | bool | `1` | With every reasoning-effort key empty, `1` makes the provider request `high` effort. |

> **For a personal subscription you only need two keys:** `PROVIDER=codex` and `CODEX_TOKEN_SOURCE=local` (plus optionally `MODEL`). The `http`, `ssh`, and `tcp` sources exist for shared or remote token setups and are not part of a normal Codex login — but note that `tcp` is the built-in default, so set `local` deliberately.

In the config editor (`/config` in a session, or `scorpiox-config` from a shell) the Codex keys live in the **Provider & Authentication** section and appear once `PROVIDER` is `codex`: `CODEX_TOKEN_SOURCE`, then `CODEX_REMOTE_URL` and `TOKEN_HTTP_API_KEY` when the source is `http`, the `CODEX_SSH_*` set when it is `ssh`, the `TCP_*` set when it is `tcp`, and `CODEX_REASONING_EFFORT` as a multiple-choice entry. `CODEX_CREDENTIALS_FILE` is honored everywhere but is not part of that editor's list — set it in a profile file or the environment.

---

## Where the token lives

On login SCORPIOX CODE writes the OAuth tokens to `~/.codex/auth.json` (mode `0600`) in the same shape the official Codex CLI uses:

```json
{
  "OPENAI_API_KEY": null,
  "tokens": {
    "id_token": "...",
    "access_token": "...",
    "refresh_token": "...",
    "account_id": "..."
  },
  "last_refresh": "2026-10-03T16:41:27Z"
}
```

The `account_id` is the ChatGPT account identifier, read out of the identity token at login time. With `CODEX_TOKEN_SOURCE=local` the provider reads that file on every request through `scorpiox-codex-fetchtoken`. No API key is involved and none is ever written.

The file holds a live refresh token: it can renew your OpenAI session without you. Do not commit it, do not share it, and do not loosen its permissions.

---

## Token persistence and automatic refresh

The access token from the OAuth session is short-lived by design — but you never manage it. SCORPIOX CODE handles renewal on its own:

- **Proactive refresh.** Before each request the provider compares the stored expiry with the clock. Inside a **five-minute buffer** of expiry it runs the refresh step — a renewal against OpenAI's OAuth endpoint using the stored refresh token — writes the new pair back to the credential file (keeping a `.backup` copy alongside it), and re-reads the file. In-flight work never hits an expired credential.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries the request. A second failure after that is reported as a permanent auth error rather than retried forever.
- **Remote sources always fetch fresh.** In `http`, `ssh`, or `tcp` modes the token is re-fetched on every request — the endpoint owning the credential is the source of truth and may have rotated or revoked it — so there is no local expiry to worry about and no refresh is performed locally.

You can force the same renewal from a shell at any time:

```bash
scorpiox-codex-refreshtoken                       # refresh only if expired
scorpiox-codex-refreshtoken --force               # always refresh
scorpiox-codex-refreshtoken --credentials-file ~/.codex/accounts/work.json
```

The tool prints the current and new expiry times, backs up the credential file before touching it, and rewrites only the three OAuth fields plus the `last_refresh` timestamp — everything else in the file is preserved byte for byte. `--verbose` additionally prints the old and new token material to the terminal, which is occasionally useful for diagnostics and exactly as sensitive as it sounds; run it on a machine you trust.

As long as the refresh token in the credential file is intact, you generally sign in **once**. If you ever see an unauthorized or token-expired hint, the fix is almost always one of:

```bash
scorpiox-codex-refreshtoken --force   # token expired — renew it
scorpiox-codex-login --force          # refresh failed / account changed — re-login
```

### Inspecting the resolved token

For diagnostics you can resolve the token exactly the way the provider does, without making a model request:

```bash
scorpiox-codex-fetchtoken -config --verbose
```

It prints one JSON line containing the access token and its expiry, and `--verbose` adds the source it read from. Use it for checks, not for logging — like `--verbose` above, it emits live credential material.

Use `scorpiox-config` to see the resolved configuration and which cascade tier supplied each value:

```bash
scorpiox-config --verbose     # key values (PROVIDER, CODEX_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
scorpiox-config --get MODEL   # one key
```

`--verbose` confirms you are pointed at the account and model you expect before a long run. To check exactly which file a `local` source reads, point `CODEX_CREDENTIALS_FILE` at it explicitly and compare.

---

## Checking your subscription usage

The provider runs against the same usage windows your ChatGPT plan enforces. Check how full they are with:

```bash
scorpiox-codex-usage            # human-readable summary
scorpiox-codex-usage --json     # raw JSON
```

You get your plan type, one row per active window — the session window and the weekly window — each with its utilization percentage and the reset time in UTC and your local timezone, plus any credits balance. That is exactly what you want to know before kicking off a long autonomous run: whether the window will hold and when it turns over.

Inside a session, `/usage` runs the same tool and shows the result as **Codex Usage** when the Codex provider is active. In addition, the per-response `5h:` and `7d:` figures in the status bar come straight from the rate-limit headers the endpoint returns on this provider, so the live utilization is visible while you work — see [Token Usage Observability](usage-observability.md).

---

## Choosing a model

`MODEL` accepts either a short alias or a full model ID. The aliases at this commit resolve to:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `gpt-5.6-luna` (an unset `MODEL` falls back to the built-in alias `sonnet`) |
| `opus` | `gpt-5.6-terra` |
| `sonnet` | `gpt-5.6-luna` |
| `haiku` | `gpt-5.4-mini` |
| any `gpt-*` ID or ID containing `codex` | passed through as-is |
| anything else | `gpt-5.6-terra` (the provider's compiled-in default) |

The full IDs the provider knows about at this commit are `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.5`, `gpt-5.4-mini`, and `codex-auto-review`. Anything starting with `gpt-` or containing `codex` is sent unchanged, which is how you run a newer model your account has access to without editing anything.

> Note: the login-offered `codex` profile ships with `MODEL=gpt-5.5` out of the box. Change it to `gpt-5.6-terra` / `gpt-5.6-luna` / `gpt-5.4-mini` (or leave an alias such as `opus` / `sonnet` / `haiku`) depending on what you want by default.

Set the model in your profile, or switch it at runtime with the in-session `/model` command, which takes the same values as `MODEL`. You can also pin a model for a single run with `sx -m sonnet`, or override any key for that run only with `sx -e CODEX_TOKEN_SOURCE=local`.

### Listing the models you can use

```bash
scorpiox-codex-models
```

The tool resolves your token the same way the provider does and prints the endpoint's model catalog as raw JSON, so the list is exactly what your account can reach.

---

## What a request looks like

SCORPIOX CODE talks to the Codex endpoint in the OpenAI **Responses** wire format, wearing the same identity the Codex CLI wears. Nothing for you to configure; the shape is handled for you.

1. **Endpoint** — `https://chatgpt.com/backend-api/codex/responses`. This is the subscription surface, not `api.openai.com`, and there is no URL override key for it: a Codex login talks to Codex.
2. **Identity** — the access token as an OAuth bearer token, a `codex-cli/0.0.0` user-agent matching the CLI, and a `session_id` header carrying a UUID generated for the session so the backend can correlate the calls of one run.
3. **Body** — `stream: true` and `store: false` (the endpoint requires both), your system prompt with project context as the `instructions` field, a `reasoning` block with the resolved effort, and the tool definitions as strict `function` schemas when `TOOLS=1`.
4. **Conversation** — the history as `input` items: user turns as `input_text`, assistant turns as `output_text`, tool calls and results as `function_call` / `function_call_output` pairs, and images (attached or read with `ReadImage`) as native `input_image` items, so the model can look at a screenshot you paste.
5. **Response** — the reply always arrives as server-sent events, which the provider parses for you. Reasoning summaries are surfaced as their own block, and the status bar reports reasoning tokens alongside output (`rsn:<count>`). Cached input tokens are shown when the endpoint reports them. With `USAGE_SPEED_ESTIMATE=1` SCORPIOX CODE times the first streamed token and reports `~pp` / `~tg` throughput in the status bar.

Each request gets a five-minute timeout, so long agentic turns are not cut off mid-flight. Transient failures are retried on their own — `429`, `500`, `502`, `503`, `529`, and transient `403` — up to ten attempts with an exponential backoff that doubles from one second, caps at a minute, and adds jitter so many agents do not retry in lockstep. Above the provider retries, the agent loop has its own wider retry policy for long runs; see the `AGENT_RETRY_*` keys in [Configuration and Profiles](scorpiox-env.md).

The reasoning dial is worth a word: `CODEX_REASONING_EFFORT` is the provider's own key, `off` is a *sent* value here (it becomes `reasoning: { effort: "none" }` on the wire, the Responses API way of saying no reasoning), and `/reasoning_effort` changes the level live in mid-conversation. See [Reasoning Effort Control](reasoning-effort.md) for the full chain.

---

## Remote token sources: `http`, `ssh`, `tcp`

`local` is all a personal subscription needs. The other three sources exist so that one credential can serve a whole fleet — a build box, a container, or a machine that should never hold the credential file itself:

| Source | Where the token comes from | Keys that matter |
|--------|----------------------------|------------------|
| `local` | `~/.codex/auth.json` (or `CODEX_CREDENTIALS_FILE`) | none extra |
| `http` | An HTTP endpoint returning the token JSON | `CODEX_REMOTE_URL`, optional `TOKEN_HTTP_API_KEY` |
| `ssh` | Another machine's `~/.codex/auth.json`, read over SSH | `CODEX_SSH_HOST`, `_USER`, optional `_PORT`, `_PASS` |
| `tcp` | A token server over a raw TCP socket | `TCP_HOST` (comma-separated for failover), `TCP_PORT`, optional `TCP_API_KEY`, optional `TCP_UPSTREAM` |

The token server speaks a small request/response protocol (`SXV1 ... TYPE=codex`) and answers with the same credential JSON the local file holds, so the provider consumes all four sources identically. In `http`, `ssh`, and `tcp` modes the token is re-fetched on every request and no refresh is performed locally — the endpoint owning the credential is responsible for keeping it valid.

> **Picking a source per run.** `sx -e CODEX_TOKEN_SOURCE=local` overrides the active profile for one run — handy when a container should pull its token from the fleet's token server while your interactive profile stays `local`. You can test any source from a shell with `scorpiox-codex-fetchtoken -tcp` (or `-ssh`, `-remote`) before blaming the provider.

---

## Multi-account: profiles and switching

Many people have more than one OpenAI account (personal plus work, for example). The Codex provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-codex-login --name personal     # -> ~/.codex/accounts/personal.json
scorpiox-codex-login --name work         # -> ~/.codex/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account and point `CODEX_CREDENTIALS_FILE` at the right file, so the profile always uses that account:

```
# ~/.claude/scorpiox-env/codex-personal.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/personal.json
MODEL=gpt-5.6-terra
```

```
# ~/.claude/scorpiox-env/codex-work.txt
PROVIDER=codex
CODEX_TOKEN_SOURCE=local
CODEX_CREDENTIALS_FILE=~/.codex/accounts/work.json
MODEL=gpt-5.4-mini
```

Profiles can live at any tier — `~/.claude/scorpiox-env/<name>.txt` for your own machine, `.scorpiox/scorpiox-env/<name>.txt` committed to a project so the whole team gets the same account and model.

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```text
/profile codex-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use    codex-personal   # session-only — gone when the session ends
/profile                 # list profiles and the active one (opens the picker where available)
/profile off             # deactivate and drop back to the base cascade
```

`/profile` and `/use` both trigger a live provider reload, so the new account and model take effect immediately. If the switch cannot complete, SCORPIOX CODE reverts to the previous profile — you are never left half-switched. The difference is scope: **`/profile <name>` persists** (it writes `ACTIVE_PROFILE`, so it survives restarts), while **`/use <name>` is session-only** (an in-memory switch that disappears when the session ends). See [Configuration and Profiles](scorpiox-env.md) for the full switching semantics.

---

## Machine mode (no TTY)

Sometimes you cannot hold a terminal open between "here is the code" and "approve on your phone" — a background agent, a CI step, or SCORPIO BOT's provider flow. The login ships an **additive machine mode**: the same device-code flow split into separate, non-interactive commands that print a single line of JSON to stdout. Tokens are **never** printed — only status. Running the tool with no machine flag is the original interactive behaviour, unchanged.

| Flag | What it does |
|------|--------------|
| `--status` | Report the login state for this node as a single JSON line (add `--name` for a named account). |
| `--start` | Begin the login: returns the `verify_url` and the one-time `user_code`, and keeps the flow state for 15 minutes. |
| `--poll` | One non-blocking check. Returns `pending` until you approve, then the tokens are saved. |
| `--cancel` | Drop a pending login. |
| `--create-profile` | With `--poll`: also write the `codex` profile, without prompting. |
| `--name <account>` | Operate on a named account instead of the default credential file. |

A typical machine-mode login:

```bash
# 1. Start — prints the verify URL and one-time code
scorpiox-codex-login --start
# {"ok":true,"provider":"codex","flow":"device_code","verify_url":"https://auth.openai.com/codex/device","user_code":"XXXX-XXXX","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open verify_url on any device, sign in, enter the code.

# 3. Poll until approved (repeat until "state":"done")
scorpiox-codex-login --poll
#   {"ok":true,"provider":"codex","state":"pending","interval":5}
#   {"ok":true,"provider":"codex","state":"done","logged_in":true,"path":"/home/you/.codex/auth.json","profile":"codex","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the flow state lives in `~/.claude/.login-pending/` (mode `0600`), so the two calls can be separate processes on the same node.
- **Fifteen-minute window.** If you do not approve within 15 minutes the pending state expires and `--poll` fails with `expired` — run `--start` again for a fresh code.
- **`--poll` is the finish step, not `--finish`.** Codex is a device-code flow: it is `--start` then `--poll`. A `--finish` call is rejected with `wrong_flow`, because Codex does not use the paste-code protocol.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"codex","error":"...","detail":"..."}` — for example `exists` (credentials already there, pass `--force`), `device_code` (the request was rejected, or device login is not enabled for that account), or `denied` (authorization refused).
- **`--status` tells you where you stand.** It reports `logged_in`, whether a refresh token is stored (`has_refresh`), whether a login is mid-flight (`pending`), the stored account identity, and whether the `codex` profile exists. Use it to decide whether a machine needs a login at all.
- **Credential files written this way are mode `0600`.** Only the owning account can read them.

---

## Codex subscription vs. the OpenAI API-key provider

It is easy to confuse the two because they both run OpenAI models. Here is the difference:

| | **Codex provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `codex` | `openai` |
| **Authentication** | OAuth device-code login (`scorpiox-codex-login`), bearer token refreshed automatically | `OPENAI_API_KEY` bearer token |
| **Billing** | Your ChatGPT / Codex subscription allowance | Pay-per-token OpenAI API usage on your key |
| **What you need** | A ChatGPT Plus / Pro / Business seat | An OpenAI API key and a platform account |
| **Token lifecycle** | Short-lived access token + stored refresh token, renewed for you | Static API key, nothing to refresh |
| **Endpoint** | `chatgpt.com/backend-api/codex/responses` (fixed) | Any OpenAI-compatible `/v1/chat/completions` you name |
| **Wire format** | Responses API (`input` items, `instructions`, `reasoning`) | Chat Completions, translated for you |
| **Usage visibility** | Subscription windows: `5h:` / `7d:` in the status bar, plus the usage tool | Token counts only |

Rule of thumb: **you have a ChatGPT / Codex seat, use `codex`. You have an API key or your own model server, use `openai`.**

---

## Gotchas

- **The shipped configuration points at the token server, not your login.** `CODEX_TOKEN_SOURCE` defaults to `tcp` in the built-in defaults and in the shipped `scorpiox-env.txt`. For a personal subscription set it to `local` in your profile — otherwise every request fetches the token from the fleet's token server instead of the credential file you just wrote.
- **The login command and the provider are separate.** `scorpiox-codex-login` writes the token file and offers a profile; SCORPIOX CODE only uses it once `PROVIDER=codex` is active — via `/profile`, `/use`, `ACTIVE_PROFILE`, or the base cascade.
- **Automatic refresh follows the default file.** The in-session refresh runs `scorpiox-codex-refreshtoken --force` against `~/.codex/auth.json`; it does not carry your `CODEX_CREDENTIALS_FILE` override with it. For named accounts, renew them on a schedule with `scorpiox-codex-refreshtoken --force --credentials-file ~/.codex/accounts/<name>.json`.
- **`CODEX_CREDENTIALS_FILE` is honored but not listed in the config editor.** It is read from the cascade like any key, so put it in a profile file or export it — do not look for it in the interactive editor's key list.
- **The device code is a credential.** Never share it, and never paste it into a ticket or a chat. If the login stalls or the code is rejected, just run `scorpiox-codex-login` again for a fresh one.
- **Device login can be disabled for an account.** A `404` from the device endpoint means exactly that. Use the `--browser` fallback the error message points at; the tokens end up in the same file either way.
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Codex account your standing default and `/use` to hop to it for a single session without writing anything.
- **Environment variables beat every file.** A `CODEX_TOKEN_SOURCE` exported in a shell outranks your profile. When a setting "won't stick", check the environment before blaming the files.
- **Refreshing is automatic, but re-login is the fallback.** If `scorpiox-codex-refreshtoken --force` keeps failing — expired refresh token, account changed, plan changed — do a full `scorpiox-codex-login --force` to re-bind.
- **Subscription windows are enforced upstream.** The session and weekly limits are OpenAI's, not SCORPIOX CODE's. Watch them with `scorpiox-codex-usage` instead of guessing.
- **`OPENAI_EXTRA_HEADERS` does not apply here.** That key belongs to the OpenAI provider; the Codex provider sends its own session header. See [Session Identity Headers](identity-headers.md) for how each provider handles identity.
- **Traffic is logged locally.** Every request and response is written verbatim to the session's `traffic/` folder (or a timestamped folder under `.scorpiox/traffics/providers/codex/`), with the bearer token redacted. See [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md).
- **Diagnostics can print live token material.** `scorpiox-codex-fetchtoken -config --verbose` and `scorpiox-codex-refreshtoken --verbose` both emit credential values to the terminal. Fine for a check on your own box; avoid them in shared logs and CI output.

---

## Quick reference

| Goal | What to set |
|------|-------------|
| Personal subscription, first login | `scorpiox-codex-login`, then `PROVIDER=codex` + `CODEX_TOKEN_SOURCE=local` |
| Re-authenticate | `scorpiox-codex-login --force` |
| A second account | `scorpiox-codex-login --name work` + a profile pinning `CODEX_CREDENTIALS_FILE` |
| Renew the token by hand | `scorpiox-codex-refreshtoken --force` (add `--credentials-file` for a named account) |
| See the usage windows | `scorpiox-codex-usage`, or `/usage` in-session |
| List the endpoint's models | `scorpiox-codex-models` |
| Pick a model | `MODEL=opus` (or `sonnet` / `haiku` / a full `gpt-*` ID), or `/model` in-session |
| Set reasoning effort | `CODEX_REASONING_EFFORT=high`, or `/reasoning_effort high` in-session |
| Fleet token source | `CODEX_TOKEN_SOURCE=tcp` (or `http` / `ssh`) with its keys |
| Login from a script | `scorpiox-codex-login --start` then `--poll` |
| One-off run override | `sx --provider codex -e CODEX_TOKEN_SOURCE=local -p "..."` |

Sign in once, let the refresh loop keep the token alive, and pick the profile that points at the account and model you want.

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Using the OpenAI Provider](openai-provider.md) — the key-based path to OpenAI-compatible endpoints, and the opposite URL convention.
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md) — the Anthropic-side subscription login.
- [Using Google Antigravity CLI Subscription in SCORPIOX CODE](antigravity-provider.md) — the Google-side subscription login, including Claude models via Google.
- [Reasoning Effort Control](reasoning-effort.md) — the `CODEX_REASONING_EFFORT` chain and the live `/reasoning_effort` command.
- [Token Usage Observability](usage-observability.md) — where the token counts and the `5h:` / `7d:` subscription windows show up.
- [Session Identity Headers](identity-headers.md) — how each provider identifies a session on the wire.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim on-disk record of every request, with the bearer token masked.
