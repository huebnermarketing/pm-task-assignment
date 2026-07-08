# Signal Provenance Classification — Design

**Date:** 2026-07-08
**Status:** Approved by Krishna, pending implementation
**Target surface:** Claude skill (`pm-task-assignment`) only. n8n port inherits later.
**Source of ask:** Anomaly found while auditing the 2026-07-07 Mode 1 run — a direct client-to-PM
email thread (Naz Lax Site — Givebutter widget, #8081) was mislabeled in the run narrative as an
"overnight AM handoff from Jessica," when the source Gmail thread showed no AM involvement at all
and was over a week stale, not overnight. This is a **compound bug**: the wrong actor role AND a
false freshness claim, and the two need separate guards — see Problem below.

## 1. Problem

Two channels feed the Morning Queue — Gmail and Orbit — and neither carries an explicit,
machine-checkable record of **who really originated a signal**, **how it reached the PM**, or
**whether it's actually fresh**. This causes three distinct gaps:

1. **Mislabeling risk (data exists, framing is wrong).** A Gmail signal's `sender.category`
   (`collectors/gmail.md`) is the only classification available, and nothing captures who else
   was on the thread (To/CC). A client emailing the PM directly, with no AM anywhere in the
   thread, can be — and was — narrated as an AM handoff. On the Orbit side, comment/activity
   entries carry `actor_id`/`actor_name` but no role classification at all, so a client or AM
   commenting directly on a task the PM owns renders identically to any other actor.
2. **Stale-thread / false-freshness risk (root cause of the observed anomaly).**
   `synthesis/matcher.md` Job 4b Pass 1 corroborates a Gmail signal to an Orbit signal when **any**
   of 4 rules hold (project match, actor match, topic match, time proximity) — only the 4th rule
   checks recency, and the four are OR'd, not AND'd. A thread that matches purely on actor or topic
   overlap can corroborate — and, worse, if such a thread is the *only* signal in a row (no
   genuinely fresh event exists), it becomes that row's `latest_signal_anchor` by default with no
   check that it's actually within this run's lookback window. That's exactly how a week-stale
   thread gets narrated with "overnight"/"handed to you" language it never earned.
3. **Visibility gap (data never arrives).** `collectors/orbit.md`'s universe model
   (`get_user_workload(PM)`) intentionally excludes tasks where the PM is a **follower**, not
   assignee. Today's stated mitigation — "the ask reliably surfaces through Gmail" — is only
   half-true: it relies on a *person* emailing the PM, not on Orbit's own automatic
   follower-notification emails, which the skill currently only parses for the fixed-cost lane
   (`collectors/gmail.md` § Orbit-notification mails). Those notification emails are
   **system-generated** (sent from Orbit's own notification address, e.g.
   `notifications@orbit...`) — the actual commenter is never the `sender`, so this class of signal
   can never be classified via `sender.category` at all; the actor has to come from parsing the
   mail's content (the existing fixed-cost mail-parser already does this correctly — § 4.2 reuses
   that approach rather than inventing a new one). Outside the fixed-cost lane, a comment on a
   followed task — from an AM or a client with direct Orbit access — never reaches the PM's queue
   at all.

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

### 4.2 Orbit collector (`collectors/orbit.md`) — actor classification, and Gmail collector (`collectors/gmail.md`) — mail-parser generalization

**Actor classification** (`collectors/orbit.md`). Every comment/activity entry gains `actor_category`, computed against
data the collector already resolves in-run:
- Match against `AM_user_ids` (already built for the priority pass) → `am`.
- Match against the workload/task assignee pool or pod-matrix roster → `team`.
- Match against the project's `client_contacts[]` (already in the relationship map) by email,
  falling back to name-substring match → `client`.
- No match → `unknown`. Never guess a role.

**Mail-parser gate removed** (`collectors/gmail.md` — this parser reads Orbit's own notification
emails through the Gmail MCP, so it lives in the Gmail collector, not the Orbit collector). The
`## Orbit-notification mails` section (currently gated to
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
| Orbit | actor is `team`/`unknown` | `null` |

**Row-level provenance.** When Job 2 merges multiple signals into one row, `row.provenance` is
the provenance of whichever signal is `row.latest_signal_anchor` — the same signal already chosen
as "what triggered this row" (SKILL.md rule #24). Not a blend, not a vote — one consistent source
of truth, reusing the anchor mechanic that already exists.

### 5.1 Recency guard — freshness framing requires an in-window anchor

Separate from *who* (provenance) is *when* (freshness) — a signal can be correctly labeled
`client_direct` and still be wrongly described as "overnight" if nothing checks its age. Fix:

- **`row.is_fresh`** (new boolean, computed alongside `row.latest_signal_anchor` in Job 7): true
  only when the anchor signal's own timestamp falls within **this run's actual lookback window**
  (`[last_run_timestamp, now]` — the same dynamic window `collectors/gmail.md` § Window already
  computes per run: 12–18h normal, up to 7 days extended, 72h forced on Monday). Reuses the
  existing value; no new constant, no fixed hour count — a fixed cutoff (e.g. "last 12-14h")
  would incorrectly reject valid same-run corroboration on Monday/extended-lookback runs.
- **Time-relative phrasing** ("overnight", "just now", "handed to you overnight", "today") is
  permitted in Summary/AI-Notes/framing **only when `row.is_fresh == true`**. When `false`, the
  row still renders its correct `provenance` tag (who) but drops all freshness language in favor
  of a plain date reference (e.g. "Client asked about this on Jun 30" instead of "Client asked
  overnight").
- **Job 4b Pass 1 unaffected for context-attachment.** The existing actor-match/topic-match rules
  (with their wider, non-recency-bound reach) still work exactly as today for populating
  `context_signals[]` — attaching genuinely old backstory to a fresh row is legitimate and
  intentional (per Job 4b's existing rationale: "email threads about a task routinely span days
  before the Orbit task surfaces overnight"). The recency guard applies at the anchor/framing
  layer only — an old signal can still be attached as context; it just can never *itself* claim
  to be the fresh, triggering event unless it actually is.

## 6. Framing rule updates

- **`references/am-context.md`** — new subsection, "Provenance-tag framing," mapping each tag to
  required phrasing, following the page's existing right/wrong table pattern:
  - `am_relay` → existing relay framing — now triggered by a confirmed tag, not inferred from
    "sender happens to be AM."
  - `am_direct` → existing "§ When an AM IS the originator" framing — same tag-driven
    confirmation.
  - `client_direct` → **new required phrasing**: "Client (`<name>`) emailed you directly — no AM
    involved." Explicitly forbidden: "AM put this on your plate" / "AM handoff" language. Combined
    with § 5.1's recency guard (no "overnight" wording unless `row.is_fresh`), this is the direct
    fix for the Naz Lax mislabel — it was both the wrong actor role and a false freshness claim.
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
- **Row's only signal is stale (the Naz Lax case).** `row.is_fresh` computes `false`. The row
  still emits with correct `provenance` and its normal verb — this is not a drop reason — but
  Job 4/10 phrasing must fall back to plain-date wording (§ 5.1), never "overnight"/"today"
  language.

## 8. Explicit out-of-scope

- No verb/routing changes. All 5 locked verbs, priority-lane logic, Job 5/5.5 branching —
  untouched. Provenance only changes wording, never the recommended action.
- No new escalation/priority tier for `client_direct` or `client_orbit_comment` — framing-only,
  confirmed. A client bypassing the AM does not jump the queue or skip the PM-action filter.
- No `cc_am_aware` flag, no separate Sources-section provenance marker — considered during design
  and cut per YAGNI; the Summary/AI-Notes phrasing fix is sufficient on its own.
- No throttling/volume-control logic for the generalized mail parser.
- No change to Job 4b Pass 1's context-attachment reach (`context_signals[]` backstory can still
  span days, unchanged) — § 5.1's recency guard applies only to anchor/framing eligibility, not
  to what may be attached as context.
- No fixed-hour recency constant (e.g. a hardcoded "12-14h" cutoff) — would silently break
  Monday/extended-lookback runs. The guard always reuses the run's own already-computed
  lookback window.
- The fixed-cost lane's existing state machine (FC-0..FC-4, Client-Ask Ledger, shadow dedup)
  stays completely as-is. This design only removes the mail parser's scope *gate* for
  non-fixed-cost projects; fixed-cost-tagged mail signals keep their current dedicated path
  untouched.
