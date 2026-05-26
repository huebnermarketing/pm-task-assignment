# connector-failure-notify.md — Failure Fallback Chain

> **When any MCP connector in the primary allowlist fails (Orbit, Gmail, Fathom, Notion), the skill records and surfaces the failure through this fallback chain. Slack is outbound-send only (team-handoff + AM-ping per Mode 2) and failures of the Slack executor route through the same chain. Routines fire unattended, so each tier is fire-and-forget — no tier waits for human input. Read-only reference MCPs (Google Drive/Docs/Sheets, SharePoint per `references/external-doc-access.md`) follow the same retry-with-backoff policy but their failures are non-blocking — they degrade row context but never abort a run.**

## When this fires

- Preflight step 5 detects a connector that doesn't respond
- A collector hits an auth error mid-run
- An executor's Orbit / Email / Slack call fails
- The escalation run can't reach the configured backup channel
- The monthly archival can't move pages

Any connector failure routes here. Routine fires never block on the failure — they record it via the tiers below and proceed (or exit cleanly if proceeding is unsafe).

## Retry policy (runs before the fallback chain)

Most MCP failures are transient — a cold-start delay, a brief rate-limit, a network blip between Claude and the MCP server, or a momentary 5xx from the upstream API. Routing every first-failure straight to the PM-paging fallback chain produces noise and trains the PM to ignore alerts. Retry-with-backoff catches the bulk of these without paging anyone; the fallback chain below now fires only when retries genuinely exhaust or the error is not retryable.

### Retry rules

- **Total attempts: 4** (1 initial + 3 retries).
- **Backoff between retries: 2s, 5s, 15s** (incremental; cumulative max wait ≈ 22s before giving up).
- **Retry on:** timeouts, 5xx errors, 429 rate-limit (honor the `Retry-After` header if it specifies a wait longer than the next backoff step), connection reset, "service unavailable", generic network errors.
- **Do NOT retry on:** 4xx auth (401/403), 404 not-found, validation errors, malformed requests. These are permanent — fall through to the fallback chain immediately with `retry_skipped=true` flag on the run-log entry.

### Per-run safety cap

If 3+ distinct connector calls in the same routine fire each enter retry loops, abort the run early — that pattern almost always indicates a broader outage rather than per-connector flake. Log `early_abort=outage_suspected` on the run-log entry and trigger fallback chain Tier 1 + Tier 2 (email PM + Notion red callout) with the full list of connectors that hit retry loops. Do not continue executing the rest of the routine logic.

### Logging during retry

Each retry attempt is recorded in-memory with timestamp + error message as it happens. After exhaustion (or on a non-retryable error that skips retry), all attempts are appended to the run-log detail page under a `Connector failures` section. Format example:

```
Notion: notion-fetch failed 4/4 attempts → 2026-04-29 09:30:14 timeout / 09:30:16 timeout / 09:30:21 timeout / 09:30:36 503 → fallback Tier reached: 1
```

For a non-retryable error (no retry attempted), the same line records `1/4 attempts (retry_skipped: <reason>)` so the audit trail is consistent.

### Then continues

Only after retry exhaustion (or a non-retryable error) does the fallback chain below begin. A successful retry mid-loop returns control to the calling step as if the original call had succeeded — the fallback chain is not invoked at all.

## The 3-tier fallback chain

Tiers below fire only AFTER the retry policy above has exhausted (or been skipped due to a non-retryable error). The skill tries each tier in order. If a tier itself fails (because the connector for that tier is also down), it falls to the next tier.

### Tier 1 — Email from the PM's Gmail to themselves (sent, not drafted)

The PM's Gmail account sends an email TO themselves (their canonical email or any alias). This is a **sent** email, not a draft — it lands in their inbox like a normal incoming message. This is one of the documented exceptions where the Email Executor sends rather than drafts (see `executors/email.md`).

Template:

```
Subject: 🚨 PM Task Assignment — Connector Failure

Connector: Fathom
Time: 25 April 2026, 9:32 AM IST
Error: MCP server disconnected

What this means: I couldn't pull yesterday's meeting action items into your morning queue.
What I did: I built the queue without Fathom signals. Today's queue is missing meeting-derived items.
What you need to do: Re-authenticate the Fathom MCP. The next scheduled routine fire will pick it back up automatically; or run `PM Task Assignment, run morning` to re-collect immediately.

— PM Task Assignment skill
```

If the email succeeds, stop here. Done. (The failure is also written to the Run Log and the Incidents page — see Tier 3.) Tier 2 still runs as a redundant Notion-side trace when the failure occurred on today's Mode 1 / Mode 2 dated page; Tier 3 always runs.

### Tier 2 — Bold-red callout on today's dated Notion page

Used when Gmail itself is the failed connector OR Gmail is unreachable. Also runs as a redundant secondary trace any time the PM is likely to open Notion before checking their inbox (Mode 1 / Mode 2 failures, where Notion is the PM's primary view of the morning).

If today's dated page exists, prepend a callout block to the very top of the page:

```
🚨 **CONNECTOR FAILURE — ATTENTION REQUIRED**

[Connector] is not responding. [Brief impact and remediation step].

You may not have received the email notification for this. Please fix the connector before relying on tomorrow's run.
```

The callout uses red color formatting if Notion supports it, plus a 🚨 emoji and bold text for visibility.

If today's dated page doesn't exist yet (e.g., Mode 1 was the run that failed before writing it), create a minimal page titled with today's date that contains only this callout.

### Tier 3 — Incidents page on Notion parent

Used as a last resort, AND always used in addition to the above tiers as a safety net. This is the routine-friendly replacement for "surface at next manual invocation" — routines have no manual invocation, so the trace lives on Notion where the next fire (and the PM) can read it.

Append an incident row to a dedicated `Incidents` sub-page on the Notion parent. Create the sub-page if it does not yet exist (one-time, on first incident). Each incident row captures:

- Timestamp (ISO + local IST)
- Mode that was running (Mode 1 / Mode 2 / Mode 2 Step 3a escalation / monthly archival / preflight)
- MCP that failed
- Error message (verbatim if available, summary otherwise)
- `retry_on_next_fire` flag (boolean — `true` if the skill thinks a retry next fire is worth attempting; `false` if the failure looks structural)
- `resolved` flag (defaults `false`; the next successful run for that connector flips it to `true`)
- Run-log row link (the corresponding Run Log row for the run during which this incident fired)

The next routine fire reads this page in **preflight Step 6** (immediately after the Run Log check) and surfaces every unresolved incident in its run-log entry, so post-mortems and trend-spotting are in one place.

**Repeat-failure escalation:** if 3+ consecutive runs hit the same incident (same connector, same error class, all unresolved), escalate via **email** to the configured backup (the one configured in Preferences Escalation Backup section) — NOT the PM. Assume the PM's email may itself be the failing connector or that the PM is unreachable. Backup escalation message:

```
Subject: 🚨 PM Task Assignment — repeat connector failure for [PM name]

Connector: [name]
Failures: [N] consecutive routine fires
First seen: [timestamp]
Last seen: [timestamp]
Latest error: [message]

The PM may not have seen prior alerts. Please check on them or the connector.

— PM Task Assignment skill
(Automated message on behalf of [PM name])
```

Sent via `executors/email.md` (operational send exception). Once a successful fire writes to that connector again, mark all matching incident rows `resolved = true` and reset the consecutive-failure counter.

## Behavior matrix

| Failed connector | Tier 1 (Email) | Tier 2 (Notion) | Tier 3 (Incidents page) |
|---|---|---|---|
| Orbit | works | works | always |
| Gmail | tier 1 fails — skip | works | always |
| Fathom | works | works | always |
| Notion | works | tier 2 also fails — skip | tier 3 also fails — record in run-log only |
| Slack (outbound-send executor) | works | works | always |
| All MCPs except Claude | tier 1 fails | tier 2 fails | tier 3 fails — run-log entry is the only trace; backup-escalation email also blocked |

The "always" column for Tier 3 is the safety net. Even if all other tiers succeed, Tier 3 still records the incident on the Notion parent so the next routine fire — which has no memory of this fire — can surface it. This is how a stateless routine system maintains visibility across fires.

## Aggregation when multiple connectors fail in one run

If preflight or a single run detects multiple connector failures, do not send multiple separate notifications. Aggregate into one:

```
Subject: 🚨 PM Task Assignment — Multiple Connector Failures

2 of 4 connectors failed at 9:32 AM IST today:
  • Fathom — MCP server disconnected
  • Orbit — auth expired

What this means: today's morning queue is missing Fathom and Orbit signals.
What you need to do: re-authenticate Fathom and Orbit. The next scheduled routine fire will pick them back up automatically.
```

One email (or Notion callout, or Incidents-page entry) summarizing all failures. One Run Log line per failure (so the per-connector audit trail stays granular even when the PM-facing message is aggregated).

## Rate limiting

If the same connector has failed in 3 of the last 5 routine fires (read from the Run Log + Incidents page during preflight), escalate the next routine's Tier 1 / Tier 3 messages to leadership-flagged tone:

> "⚠️ This connector has failed repeatedly. Please prioritize fixing it — the skill is partially blind without it."

This complements the 3-consecutive-failure backup escalation in Tier 3. The "3 of last 5" rule is for non-consecutive but persistent flake; the consecutive rule is for sustained outage.

## Run-log integration

Every connector failure also appends a line to the run-log detail page for the current run, so post-mortems are clear. Format:

```
[connector] failed at [step] after [N]/4 attempts — [error] — fallback tier: [M]
```

Examples:

```
Fathom failed at Mode 1 / Step 3b (collector) after 4/4 attempts — MCP server disconnected — fallback tier: 1
Slack failed at Mode 2 / Step 6 (team-handoff send executor) after 4/4 attempts — auth expired (final) — fallback tier: 1
Notion failed at Mode 1 / Step 5 (writer) after 1/4 attempts (retry_skipped: 403 permission denied) — fallback tier: 3
```

`writers/run-log.md` reads these lines from the per-run failure list and renders them in the run's decision-trace detail page. `[N]` is the highest tier that successfully recorded the failure (1 if the email landed; 3 if everything except the Incidents-page write failed).

## Self-test

The skill can run a connector self-test on demand if the PM types something like `PM Task Assignment, check my connectors`. This re-runs preflight step 5 in isolation and reports the status of each MCP without firing any other logic. (Future addition; not in v1 unless you call it out.)

## What this file does NOT do

- Does NOT retry beyond the 4-attempt policy defined above. Once retries exhaust, the fallback chain takes over — the skill does not loop back into another retry cycle from inside the chain.
- Does NOT modify Preferences (the Incidents page is a separate sub-page on the Notion parent).
- Does NOT change the morning queue or execution behavior — those flows handle their own degraded-mode logic in the modes/* files.
- Does NOT block subsequent routine fires. The next fire reads the Incidents page in preflight Step 6 and decides for itself whether to proceed, degrade, or skip.
- Does NOT prompt the PM for confirmation. Routines fire unattended; every tier is fire-and-forget.
