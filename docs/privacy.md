# Privacy Policy & Data Architecture: Zero Collection Guarantee

**SCORPIOX, INC** publishes this policy in place of the old commercial-app privacy page. The old page described a service that collects, logs, and analyzes. That is no longer accurate. This page describes what the current architecture actually does, and the guarantee it gives you.

The short version: **SCORPIOX CODE collects nothing by default, and cannot, by construction.** SCORPIO BOT gives you three explicit operational tiers, from fully self-hosted (zero collection) to optional, isolated, wipeable cloud compute. There are no tracking cookies, no ad trackers, no silent analytics, and no "we may collect log data in case of an error." If we say something is not collected, the code path that would collect it does not exist.

Docs for SCORPIOX CODE @ `e30b171`.

---

## The two products, one rule

There are two customer-facing surfaces, and they follow one rule: **nothing is collected unless you deliberately choose a tier that does it, and you always know which tier you are in.**

- **SCORPIOX CODE** is the local coding agent. It is absolute zero: 100% local filesystem sessions, zero data collection, zero telemetry, zero analytics, zero cloud dependencies.
- **SCORPIO BOT** is the remote-control and fleet layer. It has three explicit tiers, and the default one is self-hosted with zero collection. The two SCORPIO+ tiers are optional, named, and bounded.

We separate the two because the guarantee is different for each. SCORPIOX CODE has no optional collection tier at all. SCORPIO BOT's optional tiers move only what each tier is for, and nothing more.

---

## SCORPIOX CODE: absolute zero

SCORPIOX CODE does not phone home. It has no account to create, no license to activate, no registration, no heartbeat, and no update-check beacon that reports which machine is running. The only outbound network connections it makes are to the model endpoint you explicitly configure.

That has a consequence you can check: **we cannot count active users, inspect prompts, or access code.** There is no dashboard with your name on it, no telemetry number that includes your session, and no server that knows your machine exists. "Zero data collection" here is not a promise to delete what we collect. It is the absence of a collection mechanism to delete.

### What "zero" covers

- **No telemetry pings.** No metrics endpoint, no anonymous usage counter, no crash-reporting service. The binary has nowhere to send a number, so it never sends one.
- **No analytics.** No feature-usage tracking, no screen recording, no performance sampling.
- **No cloud dependency.** There is no default endpoint baked in to "just work." If you point it at a local inference server and unplug the network, it behaves the same as it does online. It never notices the difference.
- **100% local filesystem sessions.** Everything a session records — history, events, status, and (only when you explicitly turn on traffic capture) raw request/response pairs — lives in a plain folder on your disk that you own outright. No shadow copy, no sync, no backup we hold.

```
.scorpiox/sessions/<session-id>/
```

```bash
# List local sessions
ls .scorpiox/sessions/

# Delete a session entirely (local-only, irreversible)
rm -rf .scorpiox/sessions/<session-id>
```

Deleting that directory removes the session's record completely, because there is no server-side copy to remove.

### The one opt-in switch (off by default)

For full transparency: there is exactly one place in SCORPIOX CODE where a session-event network call is even possible, and it is gated behind an explicit setting that ships disabled.

- **Shipped default: off.** Out of the box it does nothing and sends nothing. The build behaves identically whether or not the switch is ever touched.
- **Only if you turn it on** does it send session event data — and then only to a URL you control.

```ini
# Shipped default — session event emission off, nothing sent
EMIT_SESSION_TRACKING=0
```

This is the single exception in an otherwise closed system, and it is closed by default. That is why the default build collects nothing, and why the guarantee above holds without an asterisk.

---

## SCORPIO BOT: three explicit operational tiers

SCORPIO BOT lets you drive SCORPIOX CODE without a keyboard — from a browser, a script, or across a fleet of machines. It is a thin supervisor that wraps a running SCORPIOX CODE host and its local API and web dashboard. It does not modify the agent, and it does not add a collection mechanism to SCORPIOX CODE. What it does is let you choose where and how the control surface runs. That choice is the tier.

| Tier | Where it runs | Data collection |
|------|---------------|-----------------|
| **Default (Self-Hosted)** | Your own hardware | Zero. No account, no SQL, no network to us. |
| **Optional SCORPIO+ Relay** | Ephemeral device-to-device transport | Ephemeral transport only. Never logged, inspected, or sold. |
| **Optional SCORPIO+ Hosted Cloud Compute** | Isolated SCORPIOX, INC sandboxes | Execution only. Never trained on, inspected, or sold. Wipeable anytime. |

### Tier 1 — Default (Self-Hosted)

The default deployment runs entirely on your hardware. The supervisor starts the API and the web dashboard as local processes with **zero cloud dependencies**, wires the local environment for you, and never leaves an orphaned daemon behind.

```bash
scorpiox-bot                  # run the API and web dashboard locally
```

In this tier there is no account, no SQL database, and no network call to us. Node records are plain JSON files on your disk. If you do not want the cloud at all, you never touch it. This is the tier that matches SCORPIOX CODE's zero collection exactly.

### Tier 2 — Optional SCORPIO+ Relay

When a machine sits behind NAT and has no inbound IP, it can **dial out** to a hub and hold an ephemeral tunnel open. The hub then forwards ordinary requests down that tunnel to the node, and the node's own identity token is forwarded so the hub refuses to route to a node owned by a different account. Because the connection is outbound, **no inbound port is ever opened** on your machine.

```bash
scorpiox-bot --connect <hub>  # join a hub over a reverse tunnel, no inbound port
```

The relay is **ephemeral device-to-device transport**. It exists only to carry your control traffic to the device you named. It is never logged, never inspected, and never sold. When the tunnel drops, it reconnects automatically with backoff; the fleet degrades to the nodes still up rather than hanging. The relay is a pipe, not a store — there is nothing persistent to hand over.

### Tier 3 — Optional SCORPIO+ Hosted Cloud Compute

When you choose hosted compute, work runs in **isolated SCORPIOX, INC sandboxes** whose sole purpose is execution. The boundaries are explicit:

- Customer code and session logs are **never used for model training**.
- They are **never inspected** by us.
- They are **never sold**.
- They are **wipeable anytime** — you can destroy a sandbox and everything in it on demand.

The sandbox is an execution environment, not a data lake. It holds your session only as long as you are running it, and it can be destroyed at will. Choosing this tier does not change SCORPIOX CODE's own behavior: the agent still does not phone home. The cloud is the compute you opted into, not a collector that was always watching.

---

## General standards (every tier, every surface)

- **Zero tracking cookies.** We do not set tracking cookies.
- **Zero ad trackers.** There are no advertising or third-party analytics scripts in any customer-facing surface.
- **No "log data in case of an error."** We do not collect device IP, OS, or app-configuration logs through third-party crash reporters. If an error is worth fixing, the path is the same as anywhere else: you report it.
- **No data resale.** Nothing we receive — in any tier, for any purpose — is sold.
- **You control deletion.** Local: delete the session directory. Hosted cloud compute: destroy the sandbox. Both are total, because there is no shadow copy we retain.

### Data protection

SCORPIOX, INC aims to let you correct, amend, delete, or limit the use of any personal data we hold in the tiers you choose. If you are a resident of the European Economic Area, the corresponding GDPR rights — access, rectification, erasure, restriction — apply to the personal data in those tiers, and the deletion paths above are how you exercise them. To exercise a right or ask what we hold, contact SCORPIOX, INC.

---

## Verify it yourself

You do not have to take any of this on faith.

1. **Watch the network.** Run SCORPIOX CODE under a packet capture and confirm the only outbound connections are to the endpoint you configured.
2. **Look at the disk.** Confirm sessions appear under `.scorpiox/sessions/` and nowhere else.
3. **Confirm the defaults.** Any diagnostic or usage emission ships disabled. If you have not turned it on, nothing is sent.
4. **Choose your tier explicitly.** Self-host SCORPIOX BOT and there is no network to us. Only a Relay or Hosted Cloud Compute tier you deliberately select introduces a named, bounded channel.
5. **Run air-gapped.** Point SCORPIOX CODE at a local inference server, disconnect from the internet, and use it. It works, because it was never designed to need anything else.

If any step surprises you — an unexpected host, a file outside your filesystem, a tier that acts like a different one — that is a bug worth reporting, not expected behavior.

---

## The bottom line

- **SCORPIOX CODE is absolute zero.** 100% local filesystem sessions, no telemetry, no analytics, no cloud dependency. We cannot count active users, inspect prompts, or access code.
- **SCORPIO BOT has three named tiers.** Default Self-Hosted is zero collection. The optional SCORPIO+ Relay is ephemeral transport, never logged, inspected, or sold. The optional SCORPIO+ Hosted Cloud Compute is isolated, execution-only, never trained on, never inspected, never sold, and wipeable anytime.
- **No tracking cookies, no ad trackers, no resale, and you control deletion** in every tier.

You can run SCORPIOX CODE fully air-gapped, against your own inference, and it behaves exactly as it does on the open internet — because it was built to have nothing to send in the first place.

*Effective for the SCORPIOX, INC zero-collection architecture. Supersedes the prior commercial-app privacy page.*
