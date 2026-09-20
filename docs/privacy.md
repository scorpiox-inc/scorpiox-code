# Privacy Policy & Data Architecture: Zero Collection Guarantee

**Entity:** SCORPIOX, INC
**Last updated:** September 20, 2026

This is the privacy policy for everything SCORPIOX, INC ships: **SCORPIOX CODE**, the local-first AI coding tool, and **SCORPIO BOT**, the control layer that reaches SCORPIOX CODE sessions from anywhere. It replaces the earlier, app-store-era privacy policy in full. Where the old policy described a mobile app that leaned on third-party analytics, crash reporting, and cloud file links, that is gone. What replaces it is a single, checkable claim:

> **We collect zero data by default, and every optional path is a pipe you open, not a pipe we open on you.**

You can run the entire product stack with no account, no telemetry, no analytics, and no cloud. The optional cloud features exist only to do work for you, never to learn from you.

---

## The one-line proof

**We genuinely cannot count our active users.**

Most AI coding tools track every single user. They phone home on launch, count active installations, and report usage to a central dashboard. That is what lets them say "1.2 million users this month."

SCORPIOX CODE has no phone-home mechanism, no install counter, and no mandatory account. There is no backend that receives a "hello, I am running" from your machine, so there is no dashboard counting heads. If we cannot count you, we cannot track you. That is not a marketing claim — it is an architectural constraint. The channel simply does not exist.

> **Why does this matter to you?** The tools that count their users have, by design, a pipe from your machine to their servers. SCORPIOX CODE does not open that pipe. What is not sent cannot be logged, aggregated, or leaked.

---

## The two products, the guarantee

| | SCORPIOX CODE | SCORPIO BOT |
|---|---|---|
| **Runs where** | Your machine | Your machine (self-hosted) or an optional SCORPIO+ tier you enable |
| **Account required** | No | No for self-hosted; only for optional SCORPIO+ tiers |
| **Default data collection** | **Zero** | **Zero** (self-hosted) |
| **Optional cloud** | None | SCORPIO+ Relay and SCORPIO+ Hosted Cloud Compute — both opt-in, both execution-only |
| **Cookies / ad trackers** | None | None |

Both products share the same default: **zero data collection, zero telemetry, zero analytics, zero cloud dependency.** Everything below is the same promise stated per product, plus the exact boundaries of the optional tiers.

---

## SCORPIOX CODE — absolute zero

SCORPIOX CODE is a 100% local, filesystem-backed tool. This is not a setting you have to find and turn on; it is the shipping state.

- **Zero data collection.** Nothing about your identity, your prompts, your conversations, or your code is sent to SCORPIOX, INC.
- **Zero telemetry.** There are no usage pings, no launch counters, no crash reporters, and no install beacons.
- **Zero analytics.** We do not measure or report how you use the tool.
- **Zero cloud dependency.** It does not need the internet to work. Point it at a local inference endpoint, disconnect from the network, and it behaves exactly as it does online.

What that means in practice:

- **We cannot inspect your prompts.** They never leave your filesystem.
- **We cannot read your code.** Files are read locally, on your machine, only to do what you ask.
- **We cannot count your active users.** There is no channel to report it.

### Where your data lives

Every session artifact — conversation history, event files, and any traffic capture you explicitly enable — is written to a directory under your local filesystem:

```
.scorpiox/sessions/<session-id>/
```

Those files are:

- **On your disk, not on ours.** They are local artifacts, not uploads.
- **Yours to inspect.** Open them, diff them, or delete them. They are plain files.
- **Yours to delete.** Remove the session directory and the record is gone.

```bash
# List your local sessions
ls .scorpiox/sessions/

# Delete a session entirely (local-only, irreversible)
rm -rf .scorpiox/sessions/<session-id>
```

Because there is no server-side copy, deletion is total and trivial: there is nothing remote to revoke, no retention window to wait out, no "we keep a copy for 30 days." Wipe the directory and it is gone.

### The only network it makes

The **only** outbound network connections SCORPIOX CODE produces are the ones you explicitly point it at — the inference endpoint in your config. No default endpoint is baked in, no update-check beacon rides along, and no secondary "telemetry" host is hidden in the background. The network surface of the tool is exactly the endpoint you wrote down.

> See the [Data Privacy Architecture](data-privacy.md) page for the verification steps (packet capture, air-gap runs) if you want to prove this for yourself on your own machine.

---

## SCORPIO BOT — three explicit operational tiers

SCORPIO BOT is the control layer: dispatch prompts, watch live terminal streams, peek at state, and drive a fleet of machines from one screen. It does not change what a session is — it is a thin control surface over the session files SCORPIOX CODE already writes. What it adds is a question the raw tool does not face: **where does a session run, and does anything about reaching it pass through our infrastructure?**

The answer is always one of three tiers. The default is zero collection. The other two are optional, and both are **execution-only** — they exist to run your work, never to learn from it.

### Tier 1 — Default: Self-Hosted (zero collection)

This is what you get out of the box and what most people run.

- SCORPIO BOT runs on **your hardware**, talking to sessions on **your machines**.
- **Zero data collection.** No usage reporting, no analytics, no telemetry. The only traffic is the request flow between your clients and your nodes.
- No SCORPIOX, INC server sits in the path of your prompts or your code.

If you only ever run SCORPIO BOT self-hosted, then nothing about you or your work ever touches our infrastructure. End of story.

### Tier 2 — Optional: SCORPIO+ Relay (ephemeral transport only)

When a node is not directly reachable from your client — different network, behind a NAT, on the road — you can enable the SCORPIO+ Relay tier so messages reach it.

- It is an **ephemeral device-to-device transport**: it moves the bytes from the sender to the receiver and stops.
- It is **never logged, never inspected, and never sold.** The relay does not persist your payloads, does not read them for our purposes, and is not a data source for anything we do.
- It is a **transport, not a service with your data in it.** Turn it off and the path disappears; nothing is retained on our side to clean up later.

> **The boundary that matters:** the Relay moves your data so it can reach the destination you chose. It does not keep a copy. "Ephemeral" here is load-bearing — the whole point is that there is no durable artifact for anyone, including us, to hold.

### Tier 3 — Optional: SCORPIO+ Hosted Cloud Compute (isolated sandboxes)

For sessions you want to run on cloud hardware rather than your own, the SCORPIO+ Hosted Cloud Compute tier places them in isolated sandboxes.

- The sandboxes exist **solely to execute your work.** Your code and session logs run there to do the job you asked for.
- Customer code and session logs are **never used for model training.** Not for us, not for anyone, not in aggregate.
- They are **never inspected** by us for our own purposes and **never sold.**
- They are **wipeable anytime** — by you, on demand. When you tell us to wipe a sandbox, its contents are gone.

> **The boundary that matters:** even in the most "cloudy" tier, our role is a landlord of compute, not a reader of your work. The sandbox runs your session; it does not feed it back into anything. No training, no inspection, no resale, and a delete button that actually deletes.

### Which tier am I on?

| You want to... | Tier | Account? | Data collection |
|---|---|---|---|
| Run and control sessions on your own machines | **Self-Hosted** (default) | No | **Zero** |
| Reach a node that is not directly reachable | **SCORPIO+ Relay** (opt-in) | Yes | Ephemeral transport; never logged, inspected, or sold |
| Run sessions on cloud hardware | **SCORPIO+ Hosted Cloud Compute** (opt-in) | Yes | Execution-only in isolated sandboxes; never trained on, inspected, or sold; wipeable anytime |

You are on the **Self-Hosted** tier until you deliberately enable a SCORPIO+ tier. Enabling one does not turn on collection across the product; it only opens that one path.

> See [SCORPIO BOT — Remote Agent Control & Fleet Management](scorpiox-bot.md) for how the self-hosted API and web dashboard work and how nodes are registered.

---

## General standards (all products)

These hold for SCORPIOX CODE, SCORPIO BOT, and every SCORPIO+ tier:

- **Zero tracking cookies.** We do not set tracking cookies, and we do not run third-party ad or analytics cookies on our properties in a way that profiles you.
- **Zero ad trackers.** No advertising pixels, no cross-site tracking, no data brokers.
- **No account required for the core.** You can run SCORPIOX CODE and self-hosted SCORPIO BOT with no account at all. Accounts exist only for the optional SCORPIO+ tiers.
- **No data selling, ever.** In no tier is your code, your prompts, or your session data sold, licensed, or shared with a third party as a data product.
- **Deletion is real.** Local data you delete is gone. Optional cloud sandboxes you wipe are gone. We do not keep hidden retention copies.

### Your rights

Where applicable law applies (including GDPR for residents of the EEA), you retain the standard rights — access, rectification, erasure, and objection. But notice the shape of this policy: for the default, self-hosted experience there is simply **nothing of yours to hold**, so most data-subject requests resolve to "we have no copy." For the optional SCORPIO+ tiers, you can exercise erasure directly by wiping the sandbox or disabling the tier.

---

## What changed from the previous policy

The earlier policy on scorpiox.net described a mobile-era app with third-party analytics (App Center, Application Insights), crash reporting, cloud file links, and cookie-based identifiers. That product and that posture are gone. The current stack:

- Has **no third-party analytics or crash-reporting service** in the default path.
- Collects **no Personal Information** to operate — no name, email, or account is required to run it.
- Treats any optional cloud tier as **ephemeral, execution-only, and user-wipeable**, rather than a data store.

If a previous version of this page told you we collected log data or analytics, that is superseded in full by this one.

---

## TL;DR

- **SCORPIOX CODE is absolute zero:** 100% local filesystem sessions, zero collection, zero telemetry, zero analytics, zero cloud dependency. We cannot inspect your prompts, read your code, or count your users.
- **SCORPIO BOT defaults to self-hosted with zero collection.** The only two paths that touch our infrastructure are optional and both are execution-only:
  - **SCORPIO+ Relay** — ephemeral device-to-device transport; never logged, inspected, or sold.
  - **SCORPIO+ Hosted Cloud Compute** — isolated sandboxes for execution only; your code and logs are never trained on, never inspected, never sold, and wipeable anytime.
- **Zero tracking cookies, zero ad trackers, no data selling — in every tier.**
- **Deletion is real:** local data you remove is gone, and optional cloud sandboxes you wipe are gone.

You can run the whole stack air-gapped, against your own inference, with no account — and it behaves exactly as it does online, because there is nothing to send in the first place.
