# Token Usage Observability

Every API response SCORPIOX CODE receives carries a token-usage report: how many tokens went in, how many came out, how much of your prompt was served from the provider's cache, how close you are to your rate limits, and how fast tokens are flowing. SCORPIOX CODE makes that report visible in three places — the live status bar in the terminal interface, a machine-readable `stats.json` file in the session folder, and a status event pushed to host applications that embed the agent. On top of those local surfaces, an opt-in tracker can post usage counts to a usage API.

All three local surfaces are computed from the same data, so they always agree — and none of them sends anything anywhere.

Docs for SCORPIOX CODE @ `77c49df`.

> **The whole idea in one line:** the status bar shows the last three turns live, `.scorpiox/sessions/<session>/stats.json` mirrors the same telemetry as a JSON file refreshed about once a second for your own tooling, and embedded hosts receive the same numbers as a status event every two seconds — while anything that leaves your machine is off unless you turn it on.

---

## The surfaces at a glance

| Surface | Where it lives | Refresh | Audience |
|---------|----------------|---------|----------|
| Status bar | Bottom of the terminal UI | Every frame | You, while you work |
| `stats.json` | `.scorpiox/sessions/<session>/stats.json` | ~1 Hz + on every response | Scripts, dashboards, `watch` |
| Embedding status event | Delivered to the host app (callback or event queue) | Every 2 s | Host applications (WPF, bots, SDK consumers) |
| Usage tracking (opt-in) | HTTPS POST to `USAGE_API_URL` | After every API response | A usage collection endpoint you choose |

The first three are local-only. The fourth is the one thing that can cross the network, and it ships disabled.

---

## The live status bar

The bottom of the terminal interface stacks the last **three** responses, newest at the bottom of the block and drawn at full brightness; the two older rows are dimmed (70% and 50%) so your eye lands on the current turn. A typical row reads:

```
rsn:412 T:1847 in:212 R:98341 W:1204 out:1635 5h:12% 7d:3% ~pp:2143t/s ~tg:87t/s
```

### Per-turn fields

| Field | Meaning | Shown when |
|-------|---------|------------|
| `rsn:` | Reasoning tokens the model spent thinking | Provider reports them (> 0) |
| `T:` | Total for this API call — input + output tokens | Always |
| `in:` | Input tokens processed fresh this turn | Always |
| `R:` | Cache-read tokens (prompt served from cache — green, the good stuff) | > 0 |
| `W:` | Cache-write tokens (prompt prefix newly written to cache — orange) | > 0 |
| `out:` | Output tokens generated | Always |
| `5h:` | 5-hour rate-limit utilization, percent | Provider reports it |
| `7d:` | 7-day rate-limit utilization, percent | Provider reports it |
| `pp:` | Prompt-processing speed, tokens/second | Server reports timings, or estimate enabled |
| `tg:` | Token-generation speed, tokens/second | Server reports timings, or estimate enabled |

Two refinements on the speed fields: a tilde (`~pp:`, `~tg:`) means the number is a **client-side estimate**, not a server-reported timing; and `~e2e:` instead of `~tg:` means only an end-to-end rate was measurable (no first-token timestamp), so generation speed is approximated as output over total wall time. Server-reported timings always win over estimates. Estimates are enabled with `USAGE_SPEED_ESTIMATE=1` (off by default), and on OpenAI-compatible endpoints they additionally need `OPENAI_STREAM=1` so the client can see the first token arrive.

The rate-limit fields are color-coded by utilization: green below 50%, yellow from 50–80%, red above 80%.

Around the usage rows, the status bar carries the session chrome:

| Element | Meaning |
|---------|---------|
| `folder icon` + path | Current working directory (left, clickable — opens the file explorer). Home is abbreviated to `~` |
| `branch icon` + name | Current git branch (hidden outside a repo) |
| `CTX:190K` | The configured context-window threshold — the point where SCORPIOX CODE compacts the conversation (default 190,000 tokens; shown as `CTX:190K`) |
| `T 4:32` | Cache countdown — time left before the provider's prompt cache expires (see below) |
| `KA` indicator | Cache keep-alive state: `KA:<n>` green when pings are keeping the cache warm, `KA*` while a ping is in flight, `KA||` when paused after misses, `KA-` idle, hidden when keep-alive is off |
| `[CB:n 2:14]` | Scheduled callbacks: how many are active and time until the next one fires |
| `No token data yet` | Placeholder until the first response arrives |

### The cache timer

The `T m:ss` countdown tracks how long your prompt cache stays hot. The total window follows the keep-alive trigger: with keep-alive disabled it is the provider's default 5 minutes (green while more than 60% remains, yellow under 3 minutes, red under 1 minute, and `T --:--` once expired). When keep-alive is enabled, the window matches `CACHE_KEEPALIVE_TRIGGER` (default 270 s), so the timer doubles as a preview of when the next keep-alive ping will fire. Every real response — and every successful keep-alive ping — resets the clock.

---

## `stats.json` — the machine-readable contract

Everything the status bar knows is also written to a JSON file in the session folder, refreshed at most once per second while the interface runs and immediately after each response lands. This is the surface for your own tooling: point `watch`, a Grafana JSON datasource, a CI script, or a sidebar widget at it and you get the same numbers the UI shows.

```
.scorpiox/sessions/<session-id>/stats.json
```

The session id looks like `2026_10_03_prickly_liskov`. The file is written **atomically** — the content lands in a temporary file that replaces the old one in one step — so a reader never observes a half-written file, even at 1 Hz.

### Structure

```json
{
  "updated_ms": 1759420692000,
  "model": "sonnet",
  "cwd": "~/projects/demo",
  "branch": "main",
  "context_max": 190000,
  "history": [
    {"T":1847,"in":212,"R":98341,"W":1204,"rsn":412,"out":1635,
     "rl_5h":0.1200,"rl_7d":0.0340,"pp":2143.10,"tg":87.40,"speed_estimated":1},
    {"T":980,"in":180,"R":97120,"W":0,"rsn":0,"out":800,
     "rl_5h":0.1100,"rl_7d":0.0330,"pp":0.00,"tg":0.00,"speed_estimated":0}
  ],
  "cache_timer": {
    "last_response_ms": 1759420678000,
    "elapsed_s": 14,
    "remaining_s": 286,
    "fresh_s": 180,
    "total_s": 300,
    "color": "green"
  },
  "keepalive": { "state": "DISABLED", "ping_count": 0 },
  "callbacks": { "active_count": 0, "next_fire_ms": 0, "remaining_s": 0 },
  "bash": { "running": false, "command": "" },
  "bgtasks": { "running": 0, "total": 0 },
  "retry": {
    "active": false, "http_code": 0, "attempt": 0, "max_attempts": 10,
    "next_at_ms": 0, "remaining_s": 0, "reason": ""
  },
  "usage": {
    "input_tokens": 212,
    "output_tokens": 1635,
    "cache_read_tokens": 98341,
    "cache_creation_tokens": 1204,
    "reasoning_tokens": 412,
    "total_tokens": 1847,
    "rl_5h": 0.1200,
    "rl_7d": 0.0340,
    "prompt_tps": 2143.10,
    "gen_tps": 87.40,
    "speed_estimated": 1
  }
}
```

### Field reference

**Top level**

| Key | Type | Meaning |
|-----|------|---------|
| `updated_ms` | int | Epoch milliseconds when this snapshot was written — check it to detect a stale file |
| `model`, `cwd`, `branch` | string | Current model name, working directory, git branch (empty when outside a repo) |
| `context_max` | int | Context-window threshold in tokens (the `CTX:` indicator's raw value) |
| `history` | array | Up to **3** usage entries, **newest first** — same fields as the status bar rows (see below) |
| `usage` | object | The latest response's usage, exploded into full names (this is the convenient one for scripts) |

**`history` entries** (and the equivalent `usage` block, which uses its long-form key names)

| Key | Meaning |
|-----|---------|
| `T` | Total for the call: input + output tokens |
| `in` / `out` | Input and output tokens |
| `R` / `W` | Cache-read and cache-write tokens |
| `rsn` | Reasoning tokens |
| `rl_5h` / `rl_7d` | Rate-limit utilization as a fraction (0.12 = 12%); `0` when the provider doesn't report it |
| `pp` / `tg` (history), `prompt_tps` / `gen_tps` (usage block) | Prompt-processing and generation speed, tokens/second; `0` when unknown |
| `speed_estimated` | `0` = server-reported timings, `1` = client-side `pp`/`tg` estimate, `2` = end-to-end estimate only |

Note the two speed-key spellings: compact `pp`/`tg` inside `history`, long-form `prompt_tps`/`gen_tps` inside `usage`.

**`cache_timer`**

| Key | Meaning |
|-----|---------|
| `last_response_ms` | Epoch ms of the newest response (what the countdown counts from) |
| `elapsed_s` / `remaining_s` | Seconds since it, seconds left (capped at the window and at 99:59) |
| `fresh_s` / `total_s` | Green threshold and full window in seconds — follows `CACHE_KEEPALIVE_TRIGGER` when keep-alive is on (default 300/180 when off) |
| `color` | `green`, `yellow`, `red`, or `expired` — the same color the UI timer uses |

**Status blocks**

| Key | Meaning |
|-----|---------|
| `keepalive` | `state`: `DISABLED`, `IDLE`, `ACTIVE`, `PINGING`, or `PAUSED`; `ping_count`: pings sent for the current session |
| `callbacks` | `active_count`, `next_fire_ms` (epoch, `0` = none), `remaining_s` until the next fire |
| `bash` | Minimized background shell: `running` + the command it is running |
| `bgtasks` | Background task counters: `running` and `total` |
| `retry` | Transient-error retry state: `active`, upstream `http_code`, `attempt`/`max_attempts`, `next_at_ms`, `remaining_s`, and a short `reason` |

### Reading it

```bash
# Live view, once per second
watch -n 1 cat .scorpiox/sessions/$(ls -t .scorpiox/sessions | head -1)/stats.json

# Pull the latest cache-read count with jq
jq '.usage.cache_read_tokens' .scorpiox/sessions/*/stats.json
```

If you work inside a **git worktree**, the file is dual-written: once under the worktree's `.scorpiox/sessions/<id>/` and once under the main repository's, so a dashboard watching the main repo still sees worktree sessions.

---

## The embedding / SDK path

Host applications that embed the agent engine (a WPF shell, a bot process, any consumer of the `sx.dll` P/Invoke interface) receive the same telemetry as a **status event** — a JSON string pushed to the host every **2 seconds**, either through a registered event callback (instant delivery) or by polling the event queue.

```json
{
  "model": "sonnet", "provider": "anthropic", "session": "2026_10_03_prickly_liskov",
  "tools": true, "thinking": true, "busy": false,
  "mem_mb": 412.3, "cpu_pct": 7.1, "forks": 0, "fds": 34,
  "cwd": "C:\\projects\\demo", "branch": "main",
  "in": 212, "out": 1635, "cache_r": 98341, "cache_w": 1204, "rsn": 412,
  "total": 101392, "ctx": 190000,
  "cb_active": 0, "cb_next_sec": -1, "cache_sec": 286,
  "rl_5h": 0.1200, "rl_7d": 0.0340,
  "pp": 2143.10, "tg": 87.40, "speed_estimated": 1
}
```

| Field | Meaning |
|-------|---------|
| `model`, `provider`, `session` | Active model, provider name, session id |
| `tools`, `thinking`, `busy` | Tool use enabled, thinking enabled, agent currently processing |
| `mem_mb`, `cpu_pct`, `forks`, `fds` | Host process resource snapshot |
| `cwd`, `branch` | Working directory and git branch |
| `in`, `out`, `cache_r`, `cache_w`, `rsn` | The latest response's usage breakdown |
| `total` | **Input + output + both cache fields** — see the gotcha below before comparing this to `T` |
| `ctx` | Context-window threshold (the `CTX:` value) |
| `cb_active`, `cb_next_sec` | Active callback count; seconds until the next fire (`-1` = none) |
| `cache_sec` | Seconds of cache lifetime remaining on a fixed 5-minute window (`-1` = no response yet) |
| `rl_5h`, `rl_7d` | Rate-limit utilization fractions |
| `pp`, `tg`, `speed_estimated` | Speeds and their provenance, same semantics as `stats.json` |

Sessions run headless or with SDK consumers can additionally emit **per-turn usage files**: with `--emit-session` (implied by `--headless`), each response writes `messages/msg_NNNN_usage.json` inside the session folder, containing `turn`, `in`, `out`, `cache_read`, `cache_create`, plus the full rate-limit set (`rl_5h`, `rl_7d`, `rl_overage`, `rl_status`, `rl_claim`). Unlike `stats.json`, these files are an append-only per-turn record — every turn, forever, not just the last three.

---

## Opt-in usage tracking

By default, **nothing described so far leaves your machine**. The one outbound surface is the usage tracker, and it ships disabled:

```ini
# scorpiox-env.txt
USAGE_TRACKING=0                      # default: off
USAGE_API_URL=https://code.scorpiox.net/usage-send
```

With `USAGE_TRACKING=1`, after every API response SCORPIOX CODE fires a one-shot `scorpiox-usage send` command (fire-and-forget, never blocks your turn) that POSTs a small JSON record to the configured endpoint. The record contains:

```json
{
  "metadata": {
    "session_id": "2026_10_03_prickly_liskov",
    "provider": "anthropic",
    "model": "sonnet",
    "hostname": "workstation-1",
    "username": "alice",
    "os": "linux", "arch": "x86_64", "os_version": "6.8.0",
    "project": "demo", "branch": "main",
    "scorpiox_version": "1.2.3",
    "ratelimit_5h_utilization": 0.1200,
    "ratelimit_7d_utilization": 0.0340,
    "ratelimit_overage_utilization": 0.0000
  },
  "usage": {
    "input_tokens": 212,
    "output_tokens": 1635,
    "cache_creation_input_tokens": 1204,
    "cache_read_input_tokens": 98341
  }
}
```

Notes on the payload:

- Machine metadata (`hostname`, `username`, `os`, `arch`, `os_version`) is collected automatically; `project` defaults to the basename of your working directory and `branch` to the current git branch. Optional fields (`model`, `project`, `branch`, rate-limit strings) are omitted when empty.
- `service_tier` is included when the provider reports one.
- Point `USAGE_API_URL` at your own collector to keep the data entirely in-house. An empty value falls back to the built-in default endpoint — set a real URL rather than blanking it.
- Turn tracking back off at any time with `USAGE_TRACKING=0` in any configuration-cascade tier; the setting is read per invocation, so it takes effect immediately. See [Configuration and Profiles](scorpiox-env.md) for the tier order.

This is the same zero-collection stance as everything else in SCORPIOX CODE: see [Privacy Architecture and Zero Data Collection](data-privacy.md) for what stays on disk and what never leaves your machine, and [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) for the verbatim on-disk capture of every request.

---

## Gotchas

- **`T` means different things on different surfaces.** In the status bar and `stats.json`, `T` is input + output for that call (cache tokens are visible separately as `R`/`W`). In the embedding status event, `total` is input + output + cache-read + cache-write. Do not compare `T` to `total` directly — reconstruct whichever definition you need from the individual fields, which are identical everywhere.
- **`stats.json` is a live snapshot, not an audit log.** `history` holds only the last three responses. For a permanent per-turn record, use the `msg_NNNN_usage.json` SDK files (headless/SDK sessions) or traffic logging.
- **Absent is not zero.** `5h`/`7d` and the speed fields are hidden or `0` when the provider doesn't report them — a missing rate-limit field means "unknown", not "unlimited".
- **Estimated speeds wear a tilde.** `~pp`/`~tg` are client-side estimates that include network latency and are lower bounds for prompt processing; `~e2e` is even coarser. Server timings (llama.cpp and compatible endpoints) always take precedence.
- **The cache timer window follows keep-alive.** With keep-alive enabled the countdown matches `CACHE_KEEPALIVE_TRIGGER`, and the fresh/warn boundary sits at 60% of that window — so a 270 s trigger turns yellow at 2:42, not 3:00.
- **`updated_ms` is your staleness check.** If your dashboard reads a file whose `updated_ms` is minutes old, the session it describes has likely stopped rendering (or you are watching a worktree's mirror copy while the live one is elsewhere).
- **Worktrees double-write.** Inside a worktree, both the worktree and the main repo get a copy of `stats.json` — point collectors at one, not both, to avoid double-counting.

---

## Related

- [Privacy Architecture and Zero Data Collection](data-privacy.md) — the zero-collection stance behind all of this.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the verbatim on-disk record of every request and response.
- [Configuration and Profiles](scorpiox-env.md) — where `USAGE_TRACKING`, `USAGE_SPEED_ESTIMATE`, and `CACHE_KEEPALIVE_TRIGGER` sit in the cascade.
- [Using the /keepalive Command](keepalive.md) — the mechanism behind the `T m:ss` countdown and the `KA` indicator.
