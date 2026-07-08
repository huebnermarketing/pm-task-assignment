# Signal Provenance Classification — Design

**Date:** 2026-07-08
**Status:** Approved by Krishna, pending implementation
**Target surface:** Claude skill (`pm-task-assignment`) only. n8n port inherits later.
**Source of ask:** Anomaly found while auditing the 2026-07-07 Mode 1 run — a direct client-to-PM
email thread (Naz Lax Site — Givebutter widget, #8081) was mislabeled in the run narrative as an
"overnight AM handoff from Jessica," when the source Gmail thread showed no AM involvement at all
and was over a week stale, not overnight. Root cause: the skill has no data field that records who
actually originated a signal — framing is inferred ad hoc from `sender.category` alone, with no
guard against conflating AM-relay with direct-client contact.

## 1. Problem

Two channels feed the Morning Queue — Gmail and Orbit — and neither carries an explicit,
machine-checkable record of **who really originated a signal** and **how it reached the PM**.
This causes two distinct gaps:

1. **Mislabeling risk (data exists, framing is wrong).** A Gmail signal's `sender.category`
   (`collectors/gmail.md`) is the only classification available, and nothing captures who else
   was on the thread (To/CC). A client emailing the PM directly, with no AM anywhere in the
   thread, can be — and was — narrated as an AM handoff. On the Orbit side, comment/activity
   entries carry `actor_id`/`actor_name` but no role classification at all, so a client or AM
   commenting directly on a task the PM owns renders identically to any other actor.
2. **Visibility gap (data never arrives).** `collectors/orbit.md`'s universe model
   (`get_user_workload(PM)`) intentionally excludes tasks where the PM is a **follower**, not
   assignee. Today's stated mitigation — "the ask reliably surfaces through Gmail" — is only
   half-true: it relies on a *person* emailing the PM, not on Orbit's own automatic
   follower-notification emails, which the skill currently only parses for the fixed-cost lane
   (`collectors/gmail.md` § Orbit-notification mails). Outside that lane, a comment on a
   followed task — from an AM or a client with direct Orbit access — never reaches the PM's
   queue at all.

## 2. Scope

**Framing-only.** This design adds a provenance classification to every signal so
Summary/AI-Notes phrasing is always correct about who originated an ask and how. It does **not**
change the 5 locked verbs, priority-lane logic, Job 5/5.5 branching, or introduce any new
escalation/priority tier. A client bypassing the AM does not jump the queue — it is simply
described accurately.

## 3. The five provenance classes

| Tag | Meaning |
|---|---|
| `am_relay` | AM emails the client; PM is cc'd. AM-initiated, client-facing, PM watching. |
| `am_direct` | AM emails the PM directly; no client on the thread. AM's own ask. |
| `client_direct` | Client emails the PM directly; no AM on the thread. |
| `client_orbit_comment` | Client comments directly on an Orbit task (some clients have direct Orbit access). |
| `am_orbit_comment` | AM comments directly on an Orbit task (no email involved). |

`provenance: null` is the common case — most signals are ordinary internal/team activity and
render exactly as they do today. The five tags only matter when non-null.

## 4. Data model changes

### 4.1 Gmail collector (`collectors/gmail.md`)

Add a `recipients` object to every signal, classified with the same category enum
`sender.category` already uses (`client | am | team | leadership | self | unknown`) — the
collector already has the Orbit relationship map on hand for this classification, so extending it
to recipients is not new capability, just wider application of existing logic:

```
"recipients": {
  "to": [{"name", "email", "category"}],
  "cc": [{"name", "email", "category"}]
}
```

### 4.2 Orbit collector (`collectors/orbit.md`)

**Actor classification.** Every comment/activity entry gains `actor_category`, computed against
data the collector already resolves in-run:
- Match against `AM_user_ids` (already built for the priority pass) → `am`.
- Match against the workload/task assignee pool or pod-matrix roster → `team`.
- Match against the project's `client_contacts[]` (already in the relationship map) by email,
  falling back to name-substring match → `client`.
- No match → `unknown`. Never guess a role.

**Mail-parser gate removed.** The `## Orbit-notification mails` section (currently gated to
`registry_snapshot.tracked_projects[]`, i.e. fixed-cost only) drops that gate and runs for every
Orbit notification mail in the lookback window, resolving `project_id` the same way (subject/body
`#<project_number>` match) regardless of fixed-cost status. The parsed `actor` field gets the same
`actor_category` classification as above.

Fixed-cost-tagged mails (`origin: fixed_cost_mail`) keep their existing dedicated dedup/Job-4c
treatment, completely unchanged. Newly-unlocked general mails get a new tag,
`origin: orbit_notification_mail`, and enter Job 2's **normal** dedup path (no fixed-cost
shadow-mode exemption applies to them).

## 5. Matcher — provenance composition (new Job 1.5)

A new step, right after Job 1 (grouping), before Job 4 (Summary), Job 4b (context-link), and Job
10 (AI Notes) — all of which need the tag to render correctly. Pure lookup over
collector-supplied categories; no new identity resolution happens here.

| Signal source | Condition | `provenance` |
|---|---|---|
| Gmail | `sender.category == am` AND any of `recipients.to/cc` is `client` | `am_relay` |
| Gmail | `sender.category == am` AND no `client` in recipients | `am_direct` |
| Gmail | `sender.category == client` | `client_direct` |
| Gmail | sender is `team`/`leadership`/`self`/`unknown` | `null` |
| Orbit (activity/comment, incl. generalized mail-parser) | `actor_category == client` | `client_orbit_comment` |
| Orbit (activity/comment, incl. generalized mail-parser) | `actor_category == am` | `am_orbit_comment` |
| Orbit | actor is `team`/`leadership`/`unknown` | `null` |

**Row-level provenance.** When Job 2 merges multiple signals into one row, `row.provenance` is
the provenance of whichever signal is `row.latest_signal_anchor` — the same signal already chosen
as "what triggered this row" (SKILL.md rule #24). Not a blend, not a vote — one consistent source
of truth, reusing the anchor mechanic that already exists.

## 6. Framing rule updates

- **`references/am-context.md`** — new subsection, "Provenance-tag framing," mapping each tag to
  required phrasing, following the page's existing right/wrong table pattern:
  - `am_relay` → existing relay framing — now triggered by a confirmed tag, not inferred from
    "sender happens to be AM."
  - `am_direct` → existing "§ When an AM IS the originator" framing — same tag-driven
    confirmation.
  - `client_direct` → **new required phrasing**: "Client (`<name>`) emailed you directly — no AM
    involved." Explicitly forbidden: "AM put this on your plate" / "AM handoff" language. This is
    the direct fix for the Naz Lax mislabel.
  - `client_orbit_comment` → "Client (`<name>`) commented directly on Orbit task #`<id>`."
  - `am_orbit_comment` → same relay-vs-originated judgment as `am_relay`/`am_direct`, anchored on
    an Orbit comment instead of an email.
- **`synthesis/matcher.md` Job 4 (Summary) + AI Notes composition** — read `row.provenance` and
  apply the matching phrasing rule instead of inferring framing from `sender.category` alone
  (today's implicit path — the gap that let the mislabel through).

**Display is unaffected.** The commenter/sender's name (`actor_name` / `sender.name`) is already
always shown on every row, unchanged by this design. `actor_category`/`provenance` is a new
*internal* field used only to select phrasing — it never replaces or gates name display. When
classification is ambiguous or unresolved, the row still shows the name; it simply falls back to
today's default (unmarked) phrasing rather than asserting a role.

## 7. Error handling & edge cases

- **Actor doesn't resolve** to AM/team/client → `unknown`. `provenance` stays `null`; existing
  default framing applies. Never guess a role — matches the house style already used elsewhere
  (`project_type_unknown`, `activity_log_unmatched_entry`).
- **Ambiguous actor** (matches more than one roster) → `provenance` stays `null`, AI Notes gets
  `Uncertain: actor <name> matched more than one role; provenance not set.` No arbitrary pick.
- **Generalized mail parser fails to parse a mail** → same handling as today's
  `fc_mail_parse_failure`, generalized to `orbit_mail_parse_failure` for non-fixed-cost mails —
  skip that mail, log incident, continue. No new failure-handling pattern.
- **Dedup risk for owned-task mail signals.** The generalized mail parser doesn't always resolve
  a hard `task_id` (only `task_title` from subject/body, sometimes null for project-level events).
  For tasks the PM already owns, a mail-derived comment signal could fail to dedupe cleanly
  against the same comment already captured via `get_activity_log`. Mitigation: reuse the
  existing name-normalization/match logic from `collectors/orbit.md`'s activity-log task-mapping
  (strip HTML, lowercase, collapse whitespace) for this dedup check in Job 2 — no new matching
  mechanism invented.
- **Follower-only signal, PM not assignee.** Never dev-shaped work — routes straight to `Flag`
  (the Create-subtask/Hand-off attempt is skipped entirely, since there's no PM-owned parent to
  act under by definition). `pm_next_step: "Review comment on a task you follow — no delegation
  assumed."`
- **Volume risk (flagged, not solved now).** Generalizing the mail parser to all projects means
  more notification-mail volume for PMs following many active projects → potentially more Flag
  rows. Noted as a risk to observe post-launch; no throttling logic added preemptively.

## 8. Explicit out-of-scope

- No verb/routing changes. All 5 locked verbs, priority-lane logic, Job 5/5.5 branching —
  untouched. Provenance only changes wording, never the recommended action.
- No new escalation/priority tier for `client_direct` or `client_orbit_comment` — framing-only,
  confirmed. A client bypassing the AM does not jump the queue or skip the PM-action filter.
- No `cc_am_aware` flag, no separate Sources-section provenance marker — considered during design
  and cut per YAGNI; the Summary/AI-Notes phrasing fix is sufficient on its own.
- No throttling/volume-control logic for the generalized mail parser.
- The fixed-cost lane's existing state machine (FC-0..FC-4, Client-Ask Ledger, shadow dedup)
  stays completely as-is. This design only removes the mail parser's scope *gate* for
  non-fixed-cost projects; fixed-cost-tagged mail signals keep their current dedicated path
  untouched.
