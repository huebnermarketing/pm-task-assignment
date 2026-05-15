# connector-failure-notify.md — Failure Fallback Chain

> **When any MCP connector in the 6-MCP allowlist fails (Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks), the skill notifies the running PM through this fallback chain.**

## When this fires

- Preflight step 5 detects a connector that doesn't respond
- A collector hits an auth error mid-run
- An executor's Orbit / Email / Slack call fails
- The escalation run can't reach the configured backup channel
- The monthly archival can't move pages

Any connector failure routes here.

## The 4-tier fallback chain

The skill tries each tier in order. If a tier itself fails (because the connector for that tier is also down), it falls to the next tier.

### Tier 1 — Slack DM to the PM themselves

The PM's Slack identity is in Preferences. Send a DM with:

- Specific connector that failed
- The error message (verbatim if available)
- The remediation step (re-authenticate, check Cowork settings, etc.)
- Time of failure
- What the skill did or didn't do as a result

Template:

```
🚨 PM Task Assignment — Connector Failure

Connector: Fathom
Time: 25 April 2026, 9:32 AM IST
Error: MCP server disconnected
What this means: I couldn't pull yesterday's meeting action items into your morning queue.
What I did: I built the queue without Fathom signals. Today's queue is missing meeting-derived items.
What you need to do: Reconnect Fathom in your Cowork settings. Once it's back, run `PM Task Assignment, run morning` to re-collect, or wait for tomorrow's scheduled run.
```

If the Slack DM succeeds, stop here. Done.

### Tier 2 — Email from the PM's Gmail to themselves (sent, not drafted)

Used when Slack is the failed connector OR Slack is otherwise unreachable.

The PM's Gmail account sends an email TO themselves (their canonical email or any alias). This is a **sent** email, not a draft — it lands in their inbox like a normal incoming message. This is one of the three documented exceptions where the Email Executor sends rather than drafts.

Template:

```
Subject: 🚨 PM Task Assignment — Connector Failure

Connector: Slack
Time: 25 April 2026, 9:32 AM IST
Error: [error message]

What this means: [explanation of impact]
What I did: [what the skill did or didn't do]
What you need to do: [remediation step]

— PM Task Assignment skill
(I'm sending this to you because Slack wasn't reachable for the standard notification.)
```

If the email succeeds, stop here. Done. Optionally also leave a Notion note (Tier 3) for redundancy on the dated page.

### Tier 3 — Bold-red callout on today's dated Notion page

Used when both Slack and Gmail are unreachable.

If today's dated page exists, prepend a callout block to the very top of the page:

```
🚨 **CONNECTOR FAILURE — ATTENTION REQUIRED**

[Connector] is not responding. [Brief impact and remediation step].

You couldn't be reached via Slack or email automatically. Please fix this connector before relying on tomorrow's run.
```

The callout uses red color formatting if Notion supports it, plus a 🚨 emoji and bold text for visibility.

If today's dated page doesn't exist yet (e.g., Mode 1 was the run that failed before writing it), create a minimal page titled with today's date that contains only this callout.

### Tier 4 — Surface at the next manual invocation

Used as a last resort, AND always used in addition to the above tiers as a safety net.

The skill writes the failure to a small `failures-log.txt` section inside the Preferences page (or as a new sub-page if that gets unwieldy). The next time the PM types ANY skill command, the very first thing the skill says is:

```
⚠️ Heads up — your last [run/escalation] failed.

When: [date and time]
Connector: [name]
Error: [verbatim or summary]
Remediation: [step]

Do you want me to [proceed anyway / wait until you fix it / something else]?
```

Wait for the PM's reply before doing anything. If they say proceed, log the acknowledgment and move on. If they say wait, log and exit.

Once the failure has been acknowledged this way, remove it from the failures log so it doesn't surface again on the run after that.

## Behavior matrix

| Failed connector | Tier 1 (Slack) | Tier 2 (Email) | Tier 3 (Notion) | Tier 4 (next run) |
|---|---|---|---|---|
| Orbit | works | works | works | always |
| Gmail | works | tier 2 also fails — skip | works | always |
| Slack | tier 1 fails — skip | works | works | always |
| Fathom | works | works | works | always |
| Notion | works | works | tier 3 also fails — skip | always |
| scheduled-tasks | works | works | works | always |
| All MCPs except Claude | n/a — manual run only | n/a | n/a | always (the only viable path) |

The "always" column for Tier 4 is the safety net. Even if all other tiers succeed, Tier 4 still logs the failure for the next-run warning. This way the PM has redundant visibility.

## Aggregation when multiple connectors fail in one run

If preflight or a single run detects multiple connector failures, do not send multiple separate notifications. Aggregate into one:

```
🚨 PM Task Assignment — Multiple Connector Failures

3 of 6 connectors failed at 9:32 AM IST today:
  • Fathom — MCP server disconnected
  • Slack — auth expired
  • Pipedrive — wait, this isn't in our allowlist [should never appear; if it does, file a bug]

What this means: today's morning queue is missing meeting and Slack signals.
What you need to do: reconnect Fathom and Slack in your Cowork settings.
```

One Slack DM (or email, or Notion note, or next-run warning) summarizing all failures.

## Rate limiting

If the same connector has failed in 3 of the last 5 runs, escalate the next-run warning to leadership-flagged tone:

> "⚠️ This connector has failed repeatedly. Please prioritize fixing it — the skill is partially blind without it."

## Self-test

The skill can run a connector self-test on demand if the PM types something like `PM Task Assignment, check my connectors`. This re-runs preflight step 5 in isolation and reports the status of each MCP without firing any other logic. (Future addition; not in v1 unless you call it out.)

## What this file does NOT do

- Does NOT retry failed connectors automatically.
- Does NOT modify Preferences (except writing to the failures-log section).
- Does NOT change the morning queue or execution behavior — those flows handle their own degraded-mode logic in the modes/* files.
- Does NOT block subsequent runs unless the PM explicitly says wait.
