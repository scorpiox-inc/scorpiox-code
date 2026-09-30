# Privacy Policy & Data Architecture: Zero Collection Guarantee

**SCORPIOX, INC** publishes this policy for everything it ships: **SCORPIOX CODE**, the local-first coding agent, and **SCORPIO BOT**, the control layer that reaches SCORPIOX CODE sessions from anywhere.

It replaces, in full, the older app-store-era privacy page that described a commercial mobile app with third-party analytics, crash reporting, cloud file links, and cookie-based identifiers. That product and that posture are gone. This page describes what the current architecture actually does — and the one guarantee it makes everywhere:

> **Nothing about you, your prompts, or your code is collected. Where an optional path moves data at all, it exists only to do the work you asked for — never to learn from it, never to keep it, never to sell it.**

Docs for SCORPIOX CODE @ `13253cf`.

---

## The number we cannot give you

**We do not know how many people use SCORPIOX CODE.** We genuinely cannot tell you. There is no sign-in, no install counter, no heartbeat, no analytics ping, and no server that learns your machine exists. A tool that cannot see its users cannot report on them, cannot profile them, and cannot leak what it never received.

That is not a promise to be careful with data. It is the absence of a collection mechanism. "Zero data collection" here means there is no channel to delete data from — the strongest form of deletion is never having it.

---

## The two products, one rule

| | SCORPIOX CODE | SCORPIO BOT |
|---|---|---|
| **Runs where** | Your machine | Your hardware (default) or an optional SCORPIO+ tier you enable |
| **Account required** | No | No for self-hosted; only for optional SCORPIO+ tiers |
| **Data collection** | **Zero** — by construction | **Zero** in the default tier; the optional tiers are named, bounded, and execution-only |
| **Telemetry / analytics** | None | None |
| **Cookies / ad trackers** | None | None |

Both surfaces follow the same rule: **nothing is collected unless you deliberately choose a tier that moves data, and you always know which tier you are in.**

---

## SCORPIOX CODE: absolute zero

SCORPIOX CODE is a 100% local, filesystem-backed agent. This is not a setting you have to find — it is the shipping state.

- **Zero data collection.** Nothing about your identity, your prompts, your conversations, or your code is sent to SCORPIOX, INC.
- **Zero telemetry.** No usage pings, no launch counters, no crash reporter, no device fingerprint.
- **Zero analytics.** We do not measure, sample, or report how you use the tool.
- **Zero cloud dependencies.** There is no default endpoint baked in to "just work." Point it at a local inference server and unplug the network: it behaves exactly as it does online, because it was never designed to need anything else.

The consequences are checkable, not rhetorical:

- **We cannot inspect your prompts.** They never leave your filesystem.
- **We cannot read your code.** Files are read locally, on your machine, only to do what you asked.
- **We cannot count active users.** There is no channel that could report the number.
- **We cannot correlate a session to a person.** We do not have a name, an email, or an identifier to correlate with.

### Where your data lives

Everything a session records is written to a plain folder on your disk — history, event logs, transcripts, and the traffic capture you can read at any time:

```
.scorpiox/sessions/<session-id>/
  conversation.json     full conversation history
  agent.log             agent-level log
  events.jsonl          structured event stream
  traffic/              raw request/response capture of model calls
  meta.json             session summary
```

Those files are:

- **On your disk, not ours.** They are local artifacts, not uploads. There is no shadow copy, no sync, no backup we hold.
- **Yours to inspect.** Plain text and JSON, openable with any editor, `grep`, or `jq`.
- **Yours to delete.** Removing the session directory removes the record completely — there is no server-side copy to clean up, so nothing resurfaces later.

```bash
# List your local sessions
ls .scorpiox/sessions/

# Delete one session entirely (local-only, irreversible)
rm -rf .scorpiox/sessions/<session-id>
```

You can also set an automatic local retention window (`SESSION_RETENTION_DAYS`). Sessions older than the window are purged from your disk at startup; set it to `0` and nothing is ever deleted for you. Either way the deletion is real, because the only copy was the one you deleted.

### Credentials stay where they belong

Keys and tokens you configure — provider API keys, subscription tokens, SMTP credentials — are read from your local configuration and never transmitted to SCORPIOX, INC. When a session writes its configuration snapshot to disk for auditability, sensitive values are replaced with `****` before the file is written, so a snapshot you can read is one a stranger learns nothing from.

### The two switches that exist (both off by default)

For full transparency, exactly two outbound reporting paths exist in the code. Both ship **disabled**, and both send to a URL **you** control, not one we choose for you:

| Switch (in `scorpiox-env.txt`) | What it sends if you enable it | Default |
|---|---|---|
| `USAGE_TRACKING` | Token counts per model call (input, output, cache), plus machine fields such as hostname, username, OS, and the project name and branch | **Off** |
| `EMIT_SESSION_TRACKING` | Full conversation content — your prompts, the assistant's replies, tool calls and results | **Off** |

The second one is off by default for the obvious reason: it transmits actual conversation content. If you never touch these settings, neither ever runs — not once, not a single packet. Enabling one directs it at the endpoint you configure. The guarantee is precise: these paths are **off by default and yours to aim**, not "collection we promise to be careful with."

### Every model call is on disk, on your side

SCORPIOX CODE keeps a verbatim, local audit trail of the remote calls it makes on your behalf: the request before it leaves, the response when it arrives, the endpoint, the headers, and the token usage — under the session's `traffic/` folder.

- **Local by design.** The capture is written to your filesystem by your machine, for your eyes. Nothing in it is uploaded anywhere. Its purpose is transparency: if a byte left your machine, it is in these files, and you can see it.
- **Secrets are redacted in the capture.** Credential headers are written as `***REDACTED***` rather than stored in the clear, and capture files are written with owner-only permissions on systems that support them.
- **Replayable.** The captured requests can be re-issued locally with the bundled `scorpiox-executecurl` helper, which is what makes the trail a debugging tool rather than a diary.

See [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) for the file-by-file layout.

### The honest edges

A privacy page that hides its boundaries is worse than none. These are the edges, stated plainly:

- **Your model endpoint sees the model traffic.** If you point SCORPIOX CODE at a hosted provider, *that provider* receives your prompts and returns the responses — that is inherent to using a hosted model, not a property of this product. Running local inference removes even that. The traffic capture records exactly what crossed the boundary between your machine and theirs.
- **Distribution downloads contact the distribution server.** Installing, updating, or launching a containerized session fetches binaries, container images, and bridge executables from the distribution host. These are fetch-only downloads of public artifacts: no account, no identity, no payload from your machine, and nothing about your session travels with the request. There is no background update check while you work — updates happen when you run the update command, and a containerized session launch fetches the image it needs, once, when you ask for it.
- **Optional outbound CLIs are opt-in by use.** Web search, URL fetching, maps, weather, and mail sending are separate command-line tools the agent invokes when your task needs them. The query you ask for goes to the service that answers it. They are not running in the background, and they are not phoning home when idle.
- **Voice dictation transcribes where you point it.** Dictation sends recorded audio to the transcription endpoint configured in your settings, and to nothing else, only when you record. Point it at a local transcription server and no audio leaves your network.
- **Local files are your responsibility.** Because everything lives on your filesystem, deleting a session deletes it — permanently, with nothing on a server to restore. That is the flip side of ownership, and it is the same trade in every tier.

---

## SCORPIO BOT: three operational tiers

SCORPIO BOT is the layer that lets you reach SCORPIOX CODE sessions from anywhere — list them, stream them, type into them, and spawn new ones across machines. Its data posture depends entirely on the tier you are running, and you are always on a named tier. The default is zero collection; the two optional tiers each open exactly one bounded path and nothing more.

### Tier 1 — Default: Self-Hosted (zero collection)

This is what you get out of the box and what most people run.

- SCORPIO BOT runs on **your hardware**, talking to sessions on **your machines**.
- **Zero data collection.** No usage reporting, no analytics, no telemetry. The only traffic is the request flow between your clients and your nodes.
- No SCORPIOX, INC server sits in the path of your prompts, your code, or your fleet.

If you only ever run SCORPIO BOT self-hosted, nothing about you or your work ever touches our infrastructure. End of story. You can set it up with a local node password and a plain JSON node file — no account, no SQL, no network beyond your own.

### Tier 2 — Optional: SCORPIO+ Relay (ephemeral transport only)

When a node is not directly reachable from your client — a different network, behind NAT, on the road — the SCORPIO+ Relay tier moves messages to it.

- It is an **ephemeral device-to-device transport**: it carries the bytes from the sender to the receiver you chose, and stops.
- It is **never logged, never inspected, and never sold.** The relay does not persist your payloads, does not read them for our purposes, and is not a data source for anything we do.
- It is a **transport, not a store.** Turn it off and the path disappears; there is no durable artifact on our side to clean up later, because none was kept.

> **The boundary that matters:** the relay exists so your data can reach the destination you picked. It does not keep a copy. "Ephemeral" is load-bearing — the entire point is that there is nothing durable for anyone, including us, to hold.

The relay path authenticates with the credentials you signed in with. Those credentials are stored locally on your machine with owner-only permissions — created as private from the first byte, never world-readable — and a single logout command (`scorpiox-bot logout`) deletes the stored token outright.

### Tier 3 — Optional: SCORPIO+ Hosted Cloud Compute (isolated sandboxes)

For sessions you want to run on cloud hardware rather than your own machines, the SCORPIO+ Hosted Cloud Compute tier places them in isolated sandboxes.

- The sandboxes exist **solely to execute your work.** Your code and session logs run there to do the job you asked for.
- Customer code and session logs are **never used for model training.** Not for us, not for anyone, not in aggregate.
- They are **never inspected** by us for our own purposes, and **never sold.**
- They are **wipeable anytime** — by you, on demand. When you tell us to wipe a sandbox, its contents are gone.

> **The boundary that matters:** even in the most "cloudy" tier, our role is a landlord of compute, not a reader of your work. The sandbox runs your session; it does not feed it back into anything. No training, no inspection, no resale, and a delete button that actually deletes.

### Which tier am I on?

| You want to... | Tier | Account? | Data collection |
|---|---|---|---|
| Run and control sessions on your own machines | **Self-Hosted** (default) | No | **Zero** |
| Reach a node that is not directly reachable | **SCORPIO+ Relay** (opt-in) | Yes | Ephemeral transport; never logged, inspected, or sold |
| Run sessions on cloud hardware | **SCORPIO+ Hosted Cloud Compute** (opt-in) | Yes | Execution-only in isolated sandboxes; never trained on, never inspected, never sold; wipeable anytime |

You are on the **Self-Hosted** tier until you deliberately enable a SCORPIO+ tier. Enabling one does not switch on collection anywhere else in the product; it opens that one path and nothing else.

> See [SCORPIO BOT — Remote Agent Control & Fleet Management](scorpiox-bot.md) for how the self-hosted API, dashboard, node registry, and reverse-tunnel mesh work in practice.

---

## General standards (all products, all tiers)

These hold for SCORPIOX CODE, SCORPIO BOT, and every SCORPIO+ tier:

- **Zero tracking cookies.** We do not set tracking cookies, and we do not run third-party ad or analytics cookies on our properties.
- **Zero ad trackers.** No advertising pixels, no cross-site tracking, no data brokers.
- **No sale of data, ever.** There is no data to sell in the default tiers, and the optional tiers are contractually and architecturally barred from it.
- **No account required for the core.** You can run SCORPIOX CODE and self-hosted SCORPIO BOT with no account at all. Accounts exist only for the optional SCORPIO+ tiers, and only to authenticate the path you opted into.
- **Deletion is real.** Local data you delete is gone. Sandboxes you wipe are gone. Nothing is retained behind either action.
- **Children.** The products are developer tools not directed at children, and the default architecture collects nothing from anyone — of any age.

---

## What this replaces

The earlier privacy policy on this site (effective August 10, 2022) described a commercial mobile app that:

- collected personally identifiable information such as name and email,
- used third-party analytics and crash-reporting services,
- collected log data — IP address, device name, OS version, usage timestamps — on error,
- used third-party code that could set cookies, and
- linked cloud storage accounts, collecting access tokens.

That product and that policy are superseded. The current stack has **no third-party analytics or crash-reporting service in any default path**, collects **no personal information to operate**, and treats the optional cloud tiers as **ephemeral, execution-only, and user-wipeable** rather than as a data store. If a page or listing still describes the old behavior, that text is stale; this page is the operative policy for SCORPIOX CODE and SCORPIO BOT.

---

## Verify it yourself

You do not have to take any of this on faith. Every claim above is checkable on your own machine.

1. **Watch the network.** Run SCORPIOX CODE under a packet capture and confirm the only outbound connections are to the model endpoint you configured — and none at all if you have not started a turn.
2. **Look at the disk.** Confirm sessions appear under `.scorpiox/sessions/` and nowhere else, and that deleting one leaves nothing behind.
3. **Confirm the switches are off.** Check the built-in defaults and your configuration files: both reporting switches ship disabled. If you have not set them, nothing is being sent.
4. **Read the capture.** Open a session's `traffic/` folder and confirm what left your machine, byte for byte, with credential headers redacted.
5. **Choose your tier explicitly.** Self-host SCORPIO BOT and there is no network path to us at all. Only a Relay or Hosted Cloud Compute tier you deliberately enable introduces a named, bounded channel.
6. **Run air-gapped.** Point SCORPIOX CODE at a local inference server, disconnect from the internet, and work. It behaves exactly as it does online, because there is nothing to send in the first place.

If any step surprises you — an unexpected host, a file outside your filesystem, a tier that acts like a different one — that is a bug worth reporting, not expected behavior.

---

## Related

- [Privacy Architecture and Zero Data Collection Guarantee](data-privacy.md) — the technical deep-dive behind the SCORPIOX CODE guarantee.
- [SCORPIO BOT — Remote Agent Control & Fleet Management](scorpiox-bot.md) — the self-hosted fleet, node registry, and mesh tiers.
- [API Traffic Logging and Complete Remote Call Transparency](traffic-logging.md) — the local audit trail that proves what left your machine.
- [Configuration and Profiles](scorpiox-env.md) — where the switches above live and how the settings cascade resolves.

---

## TL;DR

- **SCORPIOX CODE is absolute zero:** 100% local filesystem sessions, zero collection, zero telemetry, zero analytics, zero cloud dependency. We cannot inspect your prompts, read your code, or count our users.
- **SCORPIO BOT defaults to self-hosted with zero collection.** The only two paths that touch our infrastructure are optional and both are execution-only:
  - **SCORPIO+ Relay** — ephemeral device-to-device transport; never logged, inspected, or sold.
  - **SCORPIO+ Hosted Cloud Compute** — isolated sandboxes for execution only; your code and logs are never trained on, never inspected, never sold, and wipeable anytime.
- **Zero tracking cookies, zero ad trackers, no data selling — in every tier.**

*Effective for the SCORPIOX, INC zero-collection architecture. Supersedes the prior commercial-app privacy page.*
