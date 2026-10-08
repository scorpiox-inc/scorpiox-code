# Using GitHub Copilot CLI Subscription in SCORPIOX CODE

You have a **GitHub Copilot** subscription — a Copilot Individual, Pro, Business, or Enterprise seat, any account the official Copilot CLI can sign into — and you don't want to meter per-token against an API key. You can use it directly in SCORPIOX CODE. Instead of billing API tokens, the **Copilot provider** signs you in with the same OAuth device-code flow the official Copilot CLI performs and sends every request to the Copilot chat endpoint your subscription already pays for.

This page walks through the **device-code login**, where the token lives and how it stays fresh, the configuration keys behind `PROVIDER=copilot`, what the request looks like on the wire, how to juggle multiple accounts with profiles (`/profile` and `/use`), how to check your usage allowance, how to drive a login without a TTY (**machine mode**), and how a Copilot subscription differs from standard API-key usage in the [OpenAI provider](openai-provider.md).

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** run `scorpiox-copilot-login`, open a link on any device, enter the one-time code — SCORPIOX CODE stores the GitHub token exactly where the Copilot tooling expects it, keeps the short-lived chat credential renewed on its own, and every request rides your existing Copilot subscription instead of an API bill.

---

## When to use this provider

| Scenario | Example |
|----------|---------|
| **You have a GitHub Copilot subscription** | Copilot Individual, Pro, Business, or Enterprise — any seat the Copilot CLI signs into |
| **You don't have (or don't want) an OpenAI / Anthropic API key** | No pay-per-token billing — requests draw on your Copilot allowance |
| **Headless / over SSH** | The login is a device-code flow: open the link and enter the code on any device. No browser and no localhost redirect needed on the box you run from |
| **Driven by SCORPIO BOT** | The login splits into separate `--start` / `--poll` commands with no open TTY — see [Machine mode](#machine-mode-no-tty) |
| **Several GitHub accounts** | One named credential file per account, one profile per file, switch with `/profile` |

If you are instead calling an OpenAI-compatible endpoint with an **API key**, or pointing at your own server (llama.cpp, vLLM, SGLang, Ollama, Azure), that is the [OpenAI provider](openai-provider.md) (`PROVIDER=openai`) — a different billing model that can run some of the same model families. See [Copilot subscription vs. the OpenAI API-key provider](#copilot-subscription-vs-the-openai-api-key-provider) below.

> **Subscription, not API key.** The Copilot provider authenticates with GitHub OAuth and draws on your account's usage allowance — the same premium-interaction windows the Copilot CLI enforces. It does **not** accept an API key and does **not** bill per token. No API key is ever written to disk by this provider.

---

## Step 1 — Sign in with the device-code flow

The default login is the **GitHub OAuth device-code flow**, the same one the official Copilot CLI performs. It is deliberately SSH- and headless-friendly: you open a link, sign in to your GitHub account on whatever device you already use, enter a one-time code, and SCORPIOX CODE picks the result up on its own. There is no localhost redirect to capture and no long URL to paste — nothing long-lived leaves the terminal, and no browser is required on the machine you run from.

```bash
scorpiox-copilot-login
```

You will see something like this:

```
scorpiox-copilot-login v...

Requesting device code...

┌─────────────────────────────────────────────────────┐
│  Visit: https://github.com/login/device             │
│  Enter code: XXXX-XXXX                              │
└─────────────────────────────────────────────────────┘

Waiting for authorization...
```

Do exactly what it says:

1. **Open the printed link** (`https://github.com/login/device`) in a browser — on any device, wherever you are signed into GitHub.
2. **Sign in** and **enter the one-time code** shown in your terminal, then approve the authorization.
3. Come back to your terminal. SCORPIOX CODE is already polling; the moment you approve it exchanges the device code for an access token and saves it.

The credentials land in `~/.copilot/.credentials.json` (mode `0600`), and the login then offers to write a ready-to-use profile — see [Step 2](#step-2--turn-on-the-copilot-provider). A few things worth knowing about this flow:

- **The code is one-time and expires in 15 minutes.** If you do not approve in time, the login stops; run `scorpiox-copilot-login` again for a fresh code.
- **Never share the code.** It is a credential: anyone who holds it while you are waiting can bind the login to their own account.
- **No browser on the box? No problem.** Approve from your laptop or phone; the terminal you are polling picks it up automatically.
- **Polling is polite.** Checks run every few seconds (GitHub names the interval; five seconds is typical, and it backs off if GitHub asks it to slow down) and stop on their own at the 15-minute deadline.

### Overwriting existing credentials

If a credential file already exists, the login refuses to clobber it:

```
Credentials already exist: /home/you/.copilot/.credentials.json
Use --force to overwrite.
```

Pass `--force` to log in again and replace the stored token — after an account change, a plan change, or when you simply want a different account in the default slot.

### Named accounts

By default the login writes to `~/.copilot/.credentials.json`. To keep several accounts side by side, give each one a name:

```bash
scorpiox-copilot-login --name personal     # -> ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work         # -> ~/.copilot/accounts/work.json
```

Names are limited to letters, digits, `-`, `_`, and `.`, up to 64 characters. A named file is only used by SCORPIOX CODE when a profile points `COPILOT_CREDENTIALS_FILE` at it — see [Multi-account](#multi-account-profiles-and-switching).

### Signing in with the VS Code identity

Two OAuth identities can front the Copilot surface, and SCORPIOX CODE supports both:

- **The Copilot CLI identity (default).** What `scorpiox-copilot-login` does out of the box. The stored token starts with `gho_`, requests ride the Copilot CLI's own endpoint and integration headers, and Claude-family models are available on every turn of a conversation, including after tool calls.
- **The VS Code identity (`--vscode`).** `scorpiox-copilot-login --vscode` signs in with the VS Code Copilot client instead. The stored token starts with `ghu_`, only `.credentials.json` is written, and requests ride the VS Code chat surface. This is the path to use when your account is only provisioned for the editor integration — but note that on this surface some Claude models are refused on follow-up turns of a tool-using conversation.

Run `scorpiox-copilot-login --help` for the full flag list. Both identities end up in the same credential file shape, so everything else on this page works the same either way.

### Creating the `copilot` profile

At the end of a successful login SCORPIOX CODE offers to write a ready-made profile (the prompt is skipped when stdin is not a TTY):

```
Create 'copilot' config profile?
  Will create: /home/you/.claude/scorpiox-env/copilot.txt
  Contents:
    PROVIDER=copilot
    COPILOT_TOKEN_SOURCE=local
    MODEL=claude-sonnet-5

Create? [Y/n]
```

Say yes and the profile is written to `~/.claude/scorpiox-env/copilot.txt`, ready to activate with `/profile copilot`. If it already exists the login says so and leaves it alone (pass `--force` to re-offer). In machine mode, add `--create-profile` to `--poll` to write it without prompting. You can also create the file by hand with exactly those three lines.

---

## Step 2 — Turn on the Copilot provider

The provider switches on with `PROVIDER=copilot`. For a personal subscription you also set `COPILOT_TOKEN_SOURCE=local`, so the provider reads the credential file your login just wrote. That is the whole setup:

```
PROVIDER=copilot
COPILOT_TOKEN_SOURCE=local
```

`local` is also the built-in default for this provider, so unlike some others you do not have to fight a fleet default here — but writing it out makes the profile self-documenting. Activate the profile — or put those keys in any cascade tier of `scorpiox-env.txt`, or export them as OS environment variables — and SCORPIOX CODE connects:

```text
/profile copilot      # activate this profile and persist it
/use    copilot      # activate for this session only (nothing written)
```

You can also select the provider for a single run from the command line:

```bash
sx --provider copilot -p 'explain this build failure'
sx --provider copilot -m sonnet -p 'review this diff'
```

---

## Configuration keys

All keys can live in any cascade tier of `scorpiox-env.txt`, in a named profile, or as OS environment variables — the highest tier that sets a key wins. See [Configuration and Profiles](scorpiox-env.md) for the full cascade and precedence rules.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `PROVIDER` | choice | *(unset)* | Set to `copilot` to activate this provider. |
| `COPILOT_TOKEN_SOURCE` | choice | `local` | Where the OAuth token comes from: `local`, `remote`, `ssh`, `tcp`, or `config`. Use **`local`** for a login you ran with `scorpiox-copilot-login`. |
| `COPILOT_CREDENTIALS_FILE` | text | *(empty)* | Override the local credential path (default `~/.copilot/.credentials.json`). Point it at a named account such as `~/.copilot/accounts/work.json` to pin a profile to one login. |
| `COPILOT_GITHUB_TOKEN` | text | *(empty)* | An explicit official-CLI token (`gho_…`). When set, SCORPIOX CODE uses that identity directly and skips the credential file entirely. |
| `COPILOT_MODEL` | text | `claude-sonnet-5` | Which model to run. Takes precedence over the generic `MODEL`. Accepts full Copilot model IDs or short aliases — see [Choosing a model](#choosing-a-model). |
| `MODEL` | text | `sonnet` | Generic model key, used when `COPILOT_MODEL` is empty. |
| `COPILOT_REASONING_EFFORT` | choice | *(empty)* | Reasoning effort: `low`, `medium`, or `high`. `REASONING_EFFORT` and `OPENAI_REASONING_EFFORT` are read as fallbacks — see [Reasoning Effort Control](reasoning-effort.md). |
| `COPILOT_REMOTE_URL` | text | *(empty)* | Token endpoint, used only when `COPILOT_TOKEN_SOURCE=remote`. |
| `COPILOT_SSH_HOST` / `_PORT` / `_USER` / `_PASS` | text | *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=ssh` — fetch the token from a remote machine over SSH. Host and user are required in that mode. |
| `TCP_HOST` / `TCP_PORT` / `TCP_API_KEY` / `TCP_UPSTREAM` | text | `proxy.scorpiox.net` / `9800` / *(empty)* / *(empty)* | Used only when `COPILOT_TOKEN_SOURCE=tcp` — fetch the token over a raw TCP socket. `TCP_HOST` accepts a comma-separated list for failover. |
| `TOKEN_HTTP_API_KEY` | text | *(empty)* | Bearer key sent to an HTTP token endpoint, when it requires one. |
| `TOOLS` | bool | `1` | Include the tool definitions so the agent can run commands and edit files. |
| `THINKING` | bool | `1` | Accepted for consistency with the other providers, but the Copilot surface is driven by the reasoning-effort chain above — see the note under [What a request looks like](#what-a-request-looks-like). |

> **For a personal subscription you only need two keys:** `PROVIDER=copilot` and `COPILOT_TOKEN_SOURCE=local` (plus optionally a model). The `remote`, `ssh`, and `tcp` sources exist for shared or remote token setups and are not part of a normal Copilot login.

In the config editor (`/config` in a session, or `scorpiox-config` from a shell) the `COPILOT_*` keys live in the **Provider & Authentication** section and appear once `PROVIDER` is `copilot`: the token source as a multiple-choice entry, then `COPILOT_REMOTE_URL` and `TOKEN_HTTP_API_KEY` when the source is `remote`, the `COPILOT_SSH_*` set when it is `ssh`, the `TCP_*` set when it is `tcp`, `COPILOT_CREDENTIALS_FILE`, `COPILOT_GITHUB_TOKEN`, and `COPILOT_REASONING_EFFORT` as a choice. The provider also accepts `COPILOT_TOKEN_SOURCE=config` — "dispatch from the cascade" — which behaves like the built-in default and is handy in generated profiles, even though the editor does not list it.

---

## Where the token lives

On login, SCORPIOX CODE writes the GitHub token to `~/.copilot/.credentials.json` (mode `0600`):

```json
{
  "github_token": "gho_...",
  "token_type": "bearer",
  "scope": "read:user",
  "timestamp": "2026-10-03T18:41:27Z"
}
```

With the default CLI identity the login also writes `~/.copilot/config.json` (mode `0600`) holding the same token under a `copilotTokens` key — the file the official Copilot CLI keeps its own credential in, which is what makes SCORPIOX CODE and the CLI interchangeable on the same machine. With `--vscode` only `.credentials.json` is written.

With `COPILOT_TOKEN_SOURCE=local` the provider reads that credential on every request. The local reader also understands the other common layouts, so a machine that already has Copilot tooling signed in works without a new login:

1. `COPILOT_CREDENTIALS_FILE`, when you set it
2. `~/.copilot/config.json` — the official CLI's `copilotTokens`
3. `~/.copilot/.credentials.json` — what `scorpiox-copilot-login` writes
4. `~/.config/github-copilot/hosts.json` — the editor extensions' store (`oauth_token`)
5. `~/.config/github-copilot/apps.json` — the same store, alternate name

On Windows the last two resolve under `%LOCALAPPDATA%` instead of `~/.config`. Named account files under `~/.copilot/accounts/` are **not** in that search order — they are only read when a profile points `COPILOT_CREDENTIALS_FILE` at them.

Treat the credential file as what it is: a live GitHub token that can renew your account session. Do not commit it, do not copy it into shared dotfile repos, and do not loosen its permissions.

---

## Token persistence and automatic refresh

You do not manage token lifetimes. SCORPIOX CODE keeps the credential fresh on its own:

- **Proactive refresh.** Before each request the provider asks its token helper for a credential whenever it has none, or when the one it holds is inside a **five-minute buffer** of expiring. In-flight work never hits an expired credential.
- **The CLI token needs no renewal.** The default `gho_` token from `scorpiox-copilot-login` is handed straight back by the helper: no exchange, no expiry clock, no refresh traffic. You generally sign in **once** per machine.
- **The VS Code token is exchanged for a short-lived chat token.** A `ghu_` token (from `--vscode`, or found in `hosts.json` / `apps.json`) is traded at GitHub's Copilot token endpoint for a short-lived Copilot token, which is cached under `~/.copilot/` and only re-minted when it is close to expiring. Repeated requests in a session do not churn the network.
- **Reactive recovery.** If a request still comes back unauthorized (HTTP 401), the provider refreshes once and retries the request. A second failure after that is reported as a permanent auth error rather than retried forever.
- **Remote sources always fetch fresh.** In `remote`, `ssh`, or `tcp` modes the token is re-fetched on every request — the endpoint owning the credential is the source of truth and may have rotated or revoked it — so there is no local expiry to worry about and no refresh is performed locally.

You can force the same renewal from a shell at any time:

```bash
scorpiox-copilot-refreshtoken                       # refresh only if needed
scorpiox-copilot-refreshtoken --force               # always refresh
scorpiox-copilot-refreshtoken --verbose             # also print token details
scorpiox-copilot-refreshtoken --credentials-file ~/.copilot/accounts/work.json
```

`--force` re-runs the exchange (or re-reads the CLI token) even when the cached copy still looks valid; `--verbose` prints the source file and the new expiry. The tool also honors `COPILOT_CREDENTIALS_FILE` when it is exported as an OS environment variable — see the [Gotchas](#gotchas) for why that matters with named accounts.

If the helper keeps failing — the account was deauthorized, the plan changed, or the stored token was revoked — the fix is a full re-login:

```bash
scorpiox-copilot-login --force
```

### Inspecting the resolved token

For diagnostics you can resolve the token exactly the way the provider does, without making a model request:

```bash
scorpiox-copilot-fetchtoken -local --verbose
```

It prints one JSON line containing the resolved token, and `--verbose` adds which file each attempt read. Use it for checks, not for logging — it emits live credential material.

Use `scorpiox-config` to see the resolved configuration and which cascade tier supplied each value:

```bash
scorpiox-config --verbose     # key values (PROVIDER, COPILOT_TOKEN_SOURCE, MODEL, ACTIVE_PROFILE, ...) with their source tier
scorpiox-config --get MODEL   # one key
```

`--verbose` confirms you are pointed at the account and model you expect before a long run. To check exactly which file a `local` source reads, point `COPILOT_CREDENTIALS_FILE` at it explicitly and compare.

---

## Checking your subscription usage

The provider runs against the same usage allowance the Copilot CLI enforces. Check how full it is with:

```bash
scorpiox-copilot-usage            # human-readable summary
scorpiox-copilot-usage --json     # raw JSON
```

You get something like this:

```
GitHub Copilot Usage
====================

  User:           octocat
  Plan:           copilot_pro
  Account type:   Individual

  Chat:           enabled
  CLI:            enabled

  Plan AIC:       128 / 300  (43% used)
  Remaining:      172 AIC
  Resets:         2026-11-01
```

That is the premium-interaction snapshot — entitlement, how much you have used, what remains, and the quota reset date — plus your plan, SKU, account type, which Copilot features your seat enables, and the organizations and enterprises the account belongs to. Exactly what you want to know before kicking off a long autonomous run: whether the allowance will hold and when it turns over. Accounts without a metered allowance report as unlimited.

Inside a session, `/usage` runs the same tool and shows the result as **Copilot Usage** when the Copilot provider is active. Unlike the Anthropic and Codex providers, this surface does not publish per-response rate-limit headers, so the status bar shows token counts rather than `5h:` / `7d:` percentages — the allowance picture lives in this tool. See [Token Usage Observability](usage-observability.md) for what the status bar does show.

You can also list the models your account can see:

```bash
scorpiox-copilot-models           # pretty-print model list
scorpiox-copilot-models --json    # raw JSON to stdout
```

And when you want ground truth rather than a catalog — which models on your plan actually accept a chat completion, with and without streaming, with and without tools — run:

```bash
scorpiox-copilot-probe                    # catalog + ping every model
scorpiox-copilot-probe --model claude-sonnet-5   # ping one id
scorpiox-copilot-probe --tools --json     # include the stream+tools column
```

The probe pings each model the way the provider does and prints a table of what answered. It is the fastest way to settle "does my seat allow this model with tools?" before you commit it to a profile. It reads the same credential sources the provider does — `COPILOT_GITHUB_TOKEN`, `~/.copilot/config.json`, then `~/.copilot/.credentials.json` — which is exactly what the default login writes.

---

## Choosing a model

`COPILOT_MODEL` (or the generic `MODEL` when the first is empty) accepts either a full Copilot model ID or a short alias. The alias mapping at this commit is:

| `MODEL` value | Resolves to |
|---------------|-------------|
| *(empty)* | `claude-sonnet-5` (the provider's compiled-in default) |
| `opus` | `claude-sonnet-5` |
| `sonnet` | `claude-sonnet-5` |
| `haiku` | `claude-sonnet-5` |
| any `claude-*`, `gpt-*`, `gemini-*`, or `kimi-*` ID | passed through as-is |
| anything else | `claude-sonnet-5` (the compiled-in default) |

All three Claude aliases collapse onto `claude-sonnet-5` on purpose: that is the Claude ID the Copilot surface has proven reliable for agentic, tool-calling conversations end to end. Anything that looks like a real model ID from one of the four supported families is sent unchanged — which is how you run a newer model your account has access to without editing anything. Run `scorpiox-copilot-models` to see every ID your seat can reach, and `scorpiox-copilot-probe --tools` to see which of them actually take tool calls.

> Note: the login-offered `copilot` profile ships with `MODEL=claude-sonnet-5` out of the box. Change it in the profile if you want a different model by default.

Set the model in your profile, or switch it at runtime with the in-session `/model` command, which takes the same values as `MODEL`. You can also pin a model for a single run with `sx -m sonnet`, or override any key for that run only with `sx -e COPILOT_TOKEN_SOURCE=local`.

---

## What a request looks like

SCORPIOX CODE talks to the Copilot chat endpoint in the OpenAI **Chat Completions** wire format, wearing the same identity the official tooling wears. Nothing for you to configure; the shape is handled for you.

1. **Endpoint** — the Copilot chat-completions surface for the identity in play: the CLI endpoint for a `gho_` token, the editor endpoint for a `ghu_`/JWT token. There is no URL override key: a Copilot login talks to Copilot.
2. **Identity** — the token as a bearer token, plus the integration headers that make the request indistinguishable from the real thing: a Copilot integration id, an editor/version string, the conversation-agent intent, an API version stamp, and a matching user-agent. This is what lets a Copilot seat run an agentic session at all.
3. **Body** — `stream: true` with usage requested in the final chunk, your system prompt with project context as the `system` message, the resolved `reasoning_effort` when one is set, and the tool definitions as strict `function` schemas when `TOOLS=1`.
4. **Conversation** — the history as chat messages: user and assistant turns as plain content, tool calls as an assistant message with a `tool_calls` array, tool results as `role: tool` replies keyed by call id, and images (attached or read with `ReadImage`) as data-URL `image_url` parts so the model can look at a screenshot you paste. Image data travels inside the tool message itself so a tool call and its picture stay adjacent — some models on this surface reject the pair when it is split.
5. **Response** — the reply arrives as server-sent events, which the provider parses for you. Reasoning text is surfaced as its own thinking block, and the status bar reports reasoning tokens alongside output (`rsn:<count>`) plus cached input tokens when the endpoint reports them. With `USAGE_SPEED_ESTIMATE=1` SCORPIOX CODE times the first streamed token and reports `~pp` / `~tg` throughput in the status bar.

On the reasoning dial: `low`, `medium`, and `high` are sent as `reasoning_effort`; `off` (also `none` or `0`) means the field is **omitted** and the provider's default thinking applies — which is not the same as Codex's explicit `none`. If the selected model is a Gemini model, `max` is mapped down to `high` before sending, because the Copilot surface for Gemini models does not accept `max`; on Claude-family models `max` passes through as-is. The setting is re-read for every request, so `/reasoning_effort` takes effect on the very next turn. See [Reasoning Effort Control](reasoning-effort.md) for the full chain.

Each request gets a five-minute timeout, so long agentic turns are not cut off mid-flight. Transient failures are retried on their own — `429`, `500`, `502`, `503`, `529`, and transient `403` — up to ten attempts with an exponential backoff that doubles from one second, caps at a minute, and adds jitter so many agents do not retry in lockstep. A `401` is handled separately: refresh the credential once, then retry (see [Token persistence and automatic refresh](#token-persistence-and-automatic-refresh)). Above the provider retries, the agent loop has its own wider retry policy for long runs; see the `AGENT_RETRY_*` keys in [Configuration and Profiles](scorpiox-env.md).

Every request and response is recorded verbatim under your session's traffic folder with the bearer token masked — see [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md).

---

## Remote token sources: `remote`, `ssh`, `tcp`

`local` is all a personal subscription needs. The other three sources exist so that one credential can serve a whole fleet — a build box, a container, or a machine that should never hold the credential file itself:

| Source | Where the token comes from | Keys that matter |
|--------|----------------------------|------------------|
| `local` | The credential files listed in [Where the token lives](#where-the-token-lives) | none extra |
| `remote` | An HTTP endpoint returning the token JSON | `COPILOT_REMOTE_URL`, optional `TOKEN_HTTP_API_KEY` |
| `ssh` | Another machine's credential file, read over SSH | `COPILOT_SSH_HOST`, `_USER`, optional `_PORT`, `_PASS` |
| `tcp` | A token server over a raw TCP socket | `TCP_HOST` (comma-separated for failover), `TCP_PORT`, optional `TCP_API_KEY`, optional `TCP_UPSTREAM` |

The token server speaks a small request/response protocol (`SXV1 ... TYPE=copilot`) and answers with the same credential JSON the local file holds, so the provider consumes all four sources identically. In `remote`, `ssh`, and `tcp` modes the token is re-fetched on every request and no refresh is performed locally — the endpoint owning the credential is responsible for keeping it valid.

> **Picking a source per run.** `sx -e COPILOT_TOKEN_SOURCE=local` overrides the active profile for one run — handy when a container should pull its token from the fleet's token server while your interactive profile stays `local`. You can test any source from a shell with `scorpiox-copilot-fetchtoken -tcp` (or `-ssh`, `-remote`) before blaming the provider.

---

## Multi-account: profiles and switching

Many people have more than one GitHub account (personal plus work, for example). The Copilot provider supports that in two layers: **named credential files** and **named config profiles**.

### Save more than one login

```bash
scorpiox-copilot-login --name personal     # -> ~/.copilot/accounts/personal.json
scorpiox-copilot-login --name work         # -> ~/.copilot/accounts/work.json
```

### Bind each login to a profile

Now make one profile per account and point `COPILOT_CREDENTIALS_FILE` at the right file, so the profile always uses that account:

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

Profiles can live at any tier — `~/.claude/scorpiox-env/<name>.txt` for your own machine, `.scorpiox/scorpiox-env/<name>.txt` committed to a project so the whole team gets the same account and model.

### Switch accounts and models in-session

With those profiles in place, switching accounts is just switching profiles — no re-login, no restart:

```text
/profile copilot-work      # persistent — writes ACTIVE_PROFILE, survives restarts
/use    copilot-personal   # session-only — gone when the session ends
/profile                   # list profiles and the active one (opens the picker where available)
/profile off               # deactivate and drop back to the base cascade
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
| `--create-profile` | With `--poll`: also write the `copilot` profile, without prompting. |
| `--name <account>` | Operate on a named account instead of the default credential file. |

A typical machine-mode login:

```bash
# 1. Start — prints the verify URL and one-time code
scorpiox-copilot-login --start
# {"ok":true,"provider":"copilot","flow":"device_code","verify_url":"https://github.com/login/device","user_code":"XXXX-XXXX","interval":5,"expires_in":900,"hint":"Open the link, enter the code, approve. This page checks automatically."}

# 2. Open verify_url on any device, sign in, enter the code.

# 3. Poll until approved (repeat until "state":"done")
scorpiox-copilot-login --poll
#   {"ok":true,"provider":"copilot","state":"pending","interval":5}
#   {"ok":true,"provider":"copilot","state":"done","logged_in":true,"path":"/home/you/.copilot/.credentials.json","profile":"copilot","profile_created":true}
```

Things worth knowing:

- **State survives between commands.** Between `--start` and `--poll` the flow state lives in `~/.claude/.login-pending/` (mode `0600`), so the two calls can be separate processes on the same node.
- **Fifteen-minute window.** If you do not approve within 15 minutes the pending state expires and `--poll` fails with `expired` — run `--start` again for a fresh code.
- **`--poll` is the finish step, not `--finish`.** Copilot is a device-code flow: it is `--start` then `--poll`. A `--finish` call is rejected with `wrong_flow`, because Copilot does not use the paste-code protocol.
- **One JSON line per call.** Everything is on stdout as a single line; the caller reads the last line that starts with `{`. A failure looks like `{"ok":false,"provider":"copilot","error":"...","detail":"..."}` — for example `exists` (credentials already there, pass `--force`), `device_code` (the request was rejected), `expired` (the 15 minutes ran out), or `denied` (authorization refused).
- **`--status` tells you where you stand.** It reports `logged_in`, whether a login is mid-flight (`pending`), the credential path, and whether the `copilot` profile exists. Use it to decide whether a machine needs a login at all.
- **Credential files written this way are mode `0600`.** Only the owning account can read them.

---

## Copilot subscription vs. the OpenAI API-key provider

It is easy to confuse the two because they can run some of the same model families. Here is the difference:

| | **Copilot provider** (this page) | **OpenAI provider** (`PROVIDER=openai`) |
|---|---|---|
| `PROVIDER` value | `copilot` | `openai` |
| **Authentication** | GitHub OAuth device-code login (`scorpiox-copilot-login`), credential kept fresh for you | `OPENAI_API_KEY` bearer token |
| **Billing** | Your GitHub Copilot subscription allowance (premium-interaction windows) | Pay-per-token API usage on your key |
| **What you need** | A Copilot Individual / Pro / Business / Enterprise seat | An API key, or a server that needs none |
| **Token lifecycle** | OAuth token stored on login; the short-lived chat credential is renewed automatically | Static API key, nothing to refresh |
| **Endpoint** | The Copilot chat surface (fixed for the identity in play) | Any OpenAI-compatible `/v1/chat/completions` you name |
| **Local / air-gapped** | No — requires a live Copilot subscription | Yes — llama.cpp, vLLM, SGLang, or any self-hosted endpoint |
| **Session identity headers** | None — the Copilot integration headers *are* the identity | `X-Sx-Session-Id` / `X-Sx-Thread-Id` via `OPENAI_EXTRA_HEADERS` |
| **Usage visibility** | Allowance snapshot from the usage tool | Token counts only |

Rule of thumb: **you have a Copilot seat, use `copilot`. You have an API key or your own model server, use `openai`.** They can run the same underlying models, but the Copilot provider bills against your existing subscription while the OpenAI provider bills per token against your key — and only the OpenAI provider can be pointed at a URL you choose. See [Using the OpenAI Provider](openai-provider.md).

The same split exists on the Anthropic side: the [Claude Code provider](claude-code-provider.md) is the subscription path to Claude models, the [Anthropic provider](anthropic-provider.md) is the key-based one. On the Codex side, the [Codex provider](codex-provider.md) is the subscription path to OpenAI's coding models.

---

## Gotchas

- **The login command and the provider are separate.** `scorpiox-copilot-login` writes the credential file and offers a profile. SCORPIOX CODE only uses it once `PROVIDER=copilot` is active — via `/profile`, `/use`, `ACTIVE_PROFILE`, or the base cascade.
- **The credential file holds a live GitHub token.** `~/.copilot/.credentials.json` can renew your account session — do not commit it, do not share it, and do not loosen its permissions.
- **Named accounts are invisible until a profile points at them.** `--name work` writes `~/.copilot/accounts/work.json`, which is not in the default search order. A profile only uses it if `COPILOT_CREDENTIALS_FILE` points there. Leaving the override empty always means the default files.
- **In-session refresh follows the default file, not your named one.** The automatic refresh runs the token helper against the default credential location; a `COPILOT_CREDENTIALS_FILE` set only inside a profile file is not visible to it. For named accounts, renew them explicitly — `scorpiox-copilot-refreshtoken --force --credentials-file ~/.copilot/accounts/work.json` — or export `COPILOT_CREDENTIALS_FILE` in the shell so both the session and its helpers see the same path. (The CLI token the default login writes does not expire locally, so this only matters for `--vscode` identities and external token files.)
- **`/profile` persists; `/use` does not.** Use `/profile` to make a Copilot account your standing default and `/use` to hop to it for a single session without writing anything.
- **Environment variables beat every file.** A `COPILOT_TOKEN_SOURCE` exported in a shell outranks your profile. When a setting "won't stick", check the environment before blaming the files.
- **The two identities ride different endpoints.** The default CLI login (`gho_`) uses the Copilot CLI's own endpoint and integration headers; `--vscode` (`ghu_`) uses the editor surface. Claude-family models behave best on the CLI identity — if a follow-up turn after a tool call is suddenly refused on a Claude model, check which identity your credential file holds.
- **The device code is a credential.** Never share it, and never paste it into a ticket or a chat. If the login stalls or the code is rejected, just run `scorpiox-copilot-login` again for a fresh one — the code only lives for 15 minutes.
- **`THINKING` does not drive this provider.** The key is accepted so shared profiles stay portable, but on the Copilot surface the thinking dial is the reasoning-effort chain (`COPILOT_REASONING_EFFORT` first). Set that instead; see [Reasoning Effort Control](reasoning-effort.md).
- **`off` means "omit", not "none".** On Codex, `off` sends an explicit no-reasoning value; on Copilot it omits the field so the provider's default thinking applies. Different providers, different meanings — do not copy a Codex profile value over without checking.
- **Subscription allowances are enforced upstream.** The premium-interaction windows are GitHub's, not SCORPIOX CODE's. Watch them with `scorpiox-copilot-usage` instead of guessing, and remember the status bar shows token counts, not allowance percentages, on this provider.
- **Diagnostics can print live token material.** `scorpiox-copilot-fetchtoken -local --verbose` and `scorpiox-copilot-refreshtoken --verbose` both emit credential values to the terminal. Fine for a check on your own box; avoid them in shared logs and CI output.
- **Machine mode is additive.** `--status` / `--start` / `--poll` / `--cancel` are only active when you pass one of those flags; an unflagged `scorpiox-copilot-login` is the interactive flow. `--finish` is not supported (Copilot is a device-code flow), and pending machine state expires after 15 minutes.
- **Profile switches are live and safe.** `/profile` and `/use` swap the provider in place and revert automatically if the new profile cannot initialize.

---

## Quick reference

| Goal | What to set |
|------|-------------|
| Personal subscription, first login | `scorpiox-copilot-login`, then `PROVIDER=copilot` + `COPILOT_TOKEN_SOURCE=local` |
| Re-authenticate | `scorpiox-copilot-login --force` |
| A second account | `scorpiox-copilot-login --name work` + a profile pinning `COPILOT_CREDENTIALS_FILE` |
| Sign in with the VS Code identity | `scorpiox-copilot-login --vscode` |
| Use an existing official CLI token | `COPILOT_GITHUB_TOKEN=gho_...` |
| Renew a credential by hand | `scorpiox-copilot-refreshtoken --force` (add `--credentials-file` for a named account) |
| See your allowance | `scorpiox-copilot-usage`, or `/usage` in-session |
| List the endpoint's models | `scorpiox-copilot-models` |
| Check which models take tools | `scorpiox-copilot-probe --tools` |
| Pick a model | `MODEL=sonnet` (or `opus` / `haiku` / a full `claude-*`, `gpt-*`, `gemini-*`, `kimi-*` ID), or `/model` in-session |
| Set reasoning effort | `COPILOT_REASONING_EFFORT=high`, or `/reasoning_effort high` in-session |
| Fleet token source | `COPILOT_TOKEN_SOURCE=tcp` (or `remote` / `ssh`) with its keys |
| Login from a script | `scorpiox-copilot-login --start` then `--poll` |
| One-off run override | `sx --provider copilot -e COPILOT_TOKEN_SOURCE=local -p "..."` |

Sign in once, let the refresh loop keep the credential alive, and pick the profile that points at the account and model you want.

---

## See also

- [Configuration and Profiles](scorpiox-env.md) — the cascade every key above is read through, and `/profile` vs `/use`.
- [Using the OpenAI Provider](openai-provider.md) — the key-based path to OpenAI-compatible endpoints, and the opposite billing model.
- [Using OpenAI Codex & ChatGPT Subscription in SCORPIOX CODE](codex-provider.md) — the other subscription login on the OpenAI side.
- [Using Claude Code CLI Subscription in SCORPIOX CODE](claude-code-provider.md) — the Anthropic-side subscription login.
- [Direct Anthropic API & Custom Endpoints](anthropic-provider.md) — the key-based path to Claude models.
- [Reasoning Effort Control](reasoning-effort.md) — the `COPILOT_REASONING_EFFORT` chain, the Gemini `max` mapping, and the live `/reasoning_effort` command.
- [Token Usage Observability](usage-observability.md) — what the status bar shows for this provider and how it differs from allowance windows.
- [Session Identity Headers](identity-headers.md) — how the providers that send SCORPIOX session headers differ from this one.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim on-disk record of every request, with the bearer token masked.
