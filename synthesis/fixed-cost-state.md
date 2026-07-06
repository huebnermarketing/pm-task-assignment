# Synthesis — Fixed-Cost State Tracker

Runs in the MAIN context (judgment, not plumbing), invoked by the matcher AFTER Job 4b
(grouping + Gmail cross-link done) and BEFORE Job 5 (verb classification). If the registry
was unreadable this run, skip ALL FOUR jobs silently (reminders/pings resume next run).

**Inputs — widened universe.** Signals tagged `origin: fixed_cost_registry` (Orbit-side)
AND signals tagged `origin: fixed_cost_mail` (parsed Orbit-notification mails, pre-resolved
to tracked projects — the fixed-cost event feed in mail-primary mode)
AND Gmail signals Job 4b Pass 1 cross-linked to an `origin: fixed_cost_registry` Orbit
signal AND Gmail signals Job 4b **Pass 1b** resolved standalone (`fixed_cost_linked: true`,
`synthesis/matcher.md` Job 4b § Pass 1b — Fixed-cost standalone Gmail resolution) against the
tracked fixed-cost project set — Pass 1b is what actually delivers the standalone case:
these signals join the fixed-cost universe even when no Orbit-side signal co-exists on the
same ask/task, plus the Client-Ask Ledger rows, `fc_state` block, Preferences
`follow_up_reminder_days` (default 7) and `fixed_cost_stale_days` (default 2, business
days). The Ledger rows and `fc_state` block arrive on the same `registry_snapshot` object
the main routine read before collector dispatch — the main routine passes that snapshot to
the matcher when invoking Job 4c (the collectors receive it too but ignore
`client_ask_ledger[]`/`fc_state`). Both FC-1 and FC-2 walk the Gmail-linked signals, not just the Orbit-origin ones: a
purely-email client delivery can resolve an open ask (FC-1), and a purely-email AM-relayed
request can open one (FC-2) — a client can deliver or be asked for something entirely over
email, with no corroborating Orbit-side signal that morning.

Two lanes (from the PM interview that produced spec §10):
- **Waiting on client** — WLIQ asked the client for something; PM needs a weekly reminder
  naming the SPECIFIC ask.
- **In development** — assigned work in motion; PM needs a SPECIFIC status ping when it
  stalls — never a generic "any update?".

A project can be in both lanes at once (waiting on assets for phase 2 while phase 1 dev
continues). State is per-ask and per-task, never per-project.

Jobs run STRICTLY in this order. FC-1 before FC-3 is the zero-false-alarm ordering: a
same-morning delivery must resolve its ask before reminders are evaluated. FC-1..FC-4 key
off the PRE-run `activity_source` (`registry_snapshot.activity_source` — the label that
governed this run's collection); `fc_state_patch` values take effect from the next run's
registry read, never mid-run.

## FC-0 — Friday coverage audit (dual-feed runs only)

Spec §11.3. Runs BEFORE FC-1, only when ALL of: today (IST) is a Friday; the collector
returned `fixed_cost_activity_loop_ran: true`; the run collected mail-derived signals
(`origin: fixed_cost_mail` universe member) — i.e. both feeds are present side by side.
This covers every shadow-mode Friday AND the mail-primary monthly re-audit (first Friday —
the Friday whose day-of-month ≤ 7 (IST), per `collectors/orbit.md` § Activity-source gate —
the only mail-primary day the loop runs). On non-Friday shadow runs, FC-0 is a no-op (both
feeds still run; only the compare is weekly). A Gmail-failure fallback run has no mail
feed, so FC-0 skips.

**Compare.** Per tracked project, over this run's lookback window, build two event sets
from the Job 4c input universe: ORBIT = `origin: fixed_cost_registry` events; MAIL =
`origin: fixed_cost_mail` events (including mail signals the shadow dedup dropped as
duplicates — dedup removes them from row consideration, not from this audit). For every
ORBIT event of an actionable class (comment, status change, assignment, due-date move,
task completion, task creation, attachment) with NO MAIL counterpart (same project, same task, same event
class, timestamp within the window): record a coverage gap `{project_number, task_title,
event_type, orbit_timestamp}` and emit a `mail_coverage_gap` incident line naming all
four. MAIL events with no ORBIT counterpart are NOT gaps (mail can precede the
lookback-bounded log view). Exclude from the comparison entirely any project NOT present
in `registry_snapshot.tracked_projects[]` at dispatch time (i.e. first discovered by THIS
run's Orbit discovery): the Gmail collector's scope gate was dispatched with the
pre-discovery snapshot, so the mail feed structurally could not have been scoped to that
project — its Orbit-only events are a dispatch artifact, not a mail-reliability signal.

**Verdict → state.**
- Zero gaps → `clean_audits_next = registry_snapshot.clean_audits + 1`. If that reaches 2
  and `activity_source == shadow` → `activity_source_next = "mail-primary"` (cutover; the
  writer applies the header flip and returns the flip audit string, which the run-log
  writer renders as a decision-trace line at end-of-run).
- Any gap → `clean_audits_next = 0`; if `activity_source == mail-primary` (monthly
  re-audit found drift) → `activity_source_next = "shadow"` (auto-revert; incident already
  emitted per gap).
- FC-0 did not run (not Friday / single feed) → both fields pass through unchanged.

Results travel in `fc_output.fc_state_patch` (§ Output & downstream) — FC-0 performs NO
Notion writes itself, same as FC-1..FC-4.

## FC-1 — Resolution scan (kills false alarms at the source)

For each ledger row with `Status: Open`, judge every signal in the Job 4c input universe
(Orbit activity signals and/or `origin: fixed_cost_mail` signals — whichever feeds this run
carries per `activity_source`), and its cross-linked Gmail context: **does this signal
deliver, answer, or make moot the ask?**
Topic-match judgment — same pattern as the Create-parent-task dedup (matcher § "Dedup").
Delivery forms include: client reply or attachment on the ask's thread/task, a new comment
noting receipt ("client sent the fonts"), the blocked task resuming (status change off
hold), or the ask's task completing.

Match → set ledger `Status: Resolved`, fill `Resolved by` with the resolving signal
reference, and `Resolved on: <today YYYY-MM-DD>`. The delivery signal itself still flows to
Job 5 like any other signal (it will usually earn its own row — that's gate 3; do NOT
suppress it).

No match → leave open. Never resolve on vague activity ("any progress?" comments do not
resolve an ask).

## FC-2 — Ask detection (opens the ledger)

Scan the Job 4c input universe (Orbit activity signals and/or `origin: fixed_cost_mail`
signals — whichever feeds this run carries per `activity_source`) for OUTBOUND
client-directed requests: a PM- or AM-authored
Orbit comment or AM-relayed email asking the client to provide/approve/decide something.
The ask must be concrete enough to name — extract the specific object ("staging approval",
"product CSV", "logo assets + brand fonts"). Can't name it → not an ask; skip (do not
ledger vague nudges).

Before appending: topic-match against existing OPEN asks on the same project. Same thing
re-asked → update that row's `Asked on` to the newer date (clock restarts; no duplicate).
This changes the `fc_state` ask key — delete the old key's entry; the reminder clock
deliberately restarts under the new key rather than inheriting `last_reminder` from the
stale one.
New ask → append ledger row: Project (rule-22 format), Ask, `Asked on` = signal date,
`Source` = signal anchor, `Status: Open`, `Last reminder: —`.

FC-2 runs before FC-3, and the ≥7-day threshold guarantees a new ask never reminds the day
it's created.

## FC-3 — Follow-up reminders (weekly, verified, fail-closed)

For each ledger row still `Status: Open` after FC-1, where
`today − max(Asked on, fc_state.asks[key].last_reminder) ≥ follow_up_reminder_days`.
**Missing-key semantics:** a missing `fc_state.asks[key]` entry for an open ask is treated
as "no prior reminder" — `max(Asked on)` alone governs the threshold (this is the normal
case for a freshly-opened ask, and also the case right after FC-2's re-ask key rotation,
which deliberately deletes the old key):

1. **Re-verify (gate 2).** Deep-read the ask's anchor NOW: Orbit source →
   `get_task_details(task_id)` (bundled comments); Gmail source → re-read the thread; mail
   source (`latest_signal.source: "orbit-mail"` — a parsed Orbit-notification mail, which
   carries no `task_id` and is not a re-readable client thread) → resolve the task via the
   current run's `open_task_roster[]` for that `project_id` (title match against the
   signal's `task_title`). Resolved → `get_task_details(roster task_id)` exactly as the
   Orbit branch; unresolved (task absent from the roster, or ambiguous title match) →
   **fail closed**: emit NOTHING, write run-log audit line `reminder suppressed —
   unverified: <ask key>` (same pattern as the unreadable-anchor case below). Also
   re-read any Gmail context signals cross-linked (Job 4b Pass 1) to that anchor/task in
   TODAY's collection — a client may have replied on a thread the anchor's task never
   mirrored, so the anchor alone can miss a same-morning answer. Confirm nothing in either
   the anchor or its linked context answers the ask (the daily scan can miss deliveries —
   filtered inbox, attachment-only replies).
   - Verification says answered → resolve the row (as FC-1 would), emit NOTHING.
   - Anchor unreadable (404/permission/timeout after retries) → **fail closed**: emit
     NOTHING, write run-log audit line `reminder suppressed — unverified: <ask key>`.
2. Still open → emit a **Flag** row candidate (`fc_row_type: follow_up_reminder`):
   - Summary pattern: `Waiting on client <N> days: <ask> — follow up via <AM name> —
     <Project Title> (#<project_number>)`. N = days since `Asked on`. Worked example:
     `Waiting on client 9 days: staging approval — follow up via Sarah Chen — Board &
     Staff Page Layout Update (#16046)`.
   - `ask_key` — structured field carried on the row candidate, same string as the
     `fc_state` key (`#<project_number>|<Asked on>|<first 40 chars of Ask>`). This is how
     Mode 2's note-interpreter recovers the ask identity from a row it never derived
     itself — the writer renders it into the row detail page's bottom reference toggle
     (`writers/notion.md` § Step 6.2; `schemas/row-detail-page.md` § Bottom toggle); PM
     never needs to see or understand it.
   - `latest_signal_anchor` = the ORIGINAL ask signal (source, id, timestamp, author,
     excerpt, link) — rule 24.
   - Task Brief: what was asked, when, on which thread/task, and what it blocks (from the
     anchor's surrounding context). AM framing per rule 23 — the AM relays to the client;
     never instruct the PM to "ping the client" directly, never frame the AM as a teammate.
   - Recommended Assignee: `—` (Flag). PM Next Step: `Ask <AM> to nudge the client re:
     <ask>`.
3. Update `fc_state.asks[key].last_reminder = today` AND mirror to the ledger
   `Last reminder` column. One reminder per ask per window — a still-open ask reminds again
   only after another full `follow_up_reminder_days`.

## FC-4 — Stale-work pings (specific by construction)

For each in-development tracked fixed-cost project, walk the collector's `open_task_roster[]`
(`collectors/orbit.md` § Fixed-cost extension § Fixed-cost signal handling) and filter to its
OPEN tasks assigned to a non-PM dev. A task is **stale** when its roster `last_activity_at` is
older than `fixed_cost_stale_days` BUSINESS days (Sat/Sun don't count — Friday-assigned work
is not stale on Monday) AND the current run's event feed shows no newer event for it
(comments, status changes, time-track entries, attachments) — the unfiltered `get_activity_log`
in shadow mode / the monthly re-audit / a Gmail-failure fallback, or the `origin:
fixed_cost_mail` signals in mail-primary mode otherwise (same feed-selection rule as FC-1/FC-2,
per `activity_source`). The roster call is lookback-independent, but the feed's own window
still guards against a same-morning event the roster snapshot predates. `last_activity_at: null` (roster payload had no
updated/last-activity timestamp) → run the light deep-read (below) FIRST to establish last
real movement before judging staleness — do not assume stale or fresh on a null.

In mail-primary mode, time-track movement is not observable in the mail feed — Orbit sends
no time-track notification mails, so `origin: fixed_cost_mail` signals can never carry it.
The roster's own `last_activity_at` (which already reflects time-track entries) remains the
authoritative staleness input regardless of mode; the same-morning guard above only ever
covers the mail-visible classes (comments, status changes, attachments) when mail-primary
is the active feed.

Throttle: skip if `fc_state.pings[task_id].last_ping` is younger than
`fixed_cost_stale_days` business days. A task that stays stale re-pings at that cadence
with an escalating day count. Throttled pings are counted (`pings_throttled`) for the run
summary but do NOT get a per-item decision-trace line — only suppressions and drops do.

For each ping: run the LIGHT deep-read (same mechanics as the due-today Flag light read,
SKILL.md rule 18): one `get_task_details(task_id)` — newest bundled comments only, no
fallback call — to get last real movement + what the task was assigned for.

Emit a **Flag** row candidate (`fc_row_type: stale_work_ping`):
- Summary pattern: `No movement <N> days: "<task title>" — <assignee>, due <date> —
  <Project Title> (#<project_number>)`. Worked example: `No movement 4 days: "Rebuild
  contact form validation" — Vijay Patel, due 2026-07-08 — Board & Staff Page Layout
  Update (#16046)`.
- `latest_signal_anchor` = the task's `latest_signal` (its last real activity) — rule 24.
- **Structured fields** — each ping row candidate carries `stale_task_title`,
  `stale_assignee_name`, and `last_movement_excerpt` as discrete fields alongside the
  rendered Summary/Task Brief (not just embedded in prose). Task Brief: what the task is
  for (from description/comments) + last known movement line + assignee + due date.
  **The generic-ping ban is structural, and the gate checks field presence, not prose
  parsing:** the emission gate (and the writer's render gate) checks that
  `stale_task_title`, `stale_assignee_name`, and `last_movement_excerpt` are all
  populated. A candidate missing any of the three fields MUST NOT be emitted — drop it and
  write a run-log audit line `ping dropped — insufficient specifics: task <id>` (the
  Notion writer enforces the same field-presence gate at render time; this is defense in
  depth).
- PM Next Step: `Check with <assignee> on "<task title>"`.

Update `fc_state.pings[task_id] = {last_ping: today, stale_since: <first stale date>}`.

## Output & downstream

This tracker returns `fc_output = {row_candidates[], ledger_mutations[], fc_state_patch}`.

- `row_candidates[]` flow into matcher Job 5, which classifies both types as `Flag` (they
  carry `fc_row_type`, no delegation target, no subtask; per matcher Job 6 they also
  short-circuit past pod-inference). Existing machinery applies unchanged: rule-24 anchor
  gate, row detail pages, PM notes.
- `ledger_mutations[]` (FC-1 resolutions, FC-2 new/updated asks, FC-3 reminder timestamps,
  FC-4 ping timestamps) and `fc_state_patch` (the updated `fc_state.asks[]` /
  `fc_state.pings[]` block) are carried in memory — this file performs NO Notion writes
  itself. The Notion writer applies them to the Fixed-Cost Registry's Client-Ask Ledger and
  `fc_state` block in its registry-refresh flow at end of Mode 1.
- `fc_state_patch` additionally carries `activity_source_next` (`"shadow"` |
  `"mail-primary"` | absent = unchanged), `clean_audits_next` (int | absent = unchanged),
  and `coverage_gaps[]` — the array of full FC-0 gap records `{project_number, task_title,
  event_type, orbit_timestamp}`. The run-log `fixed_cost_stats.coverage_gaps` stat is this
  array's length; the calling mode passes the records themselves to the run-log writer
  alongside the audit strings so it renders one decision-trace line per gap
  (`writers/run-log.md` Step 3). The header fields are applied by the writer to the
  registry header (`schemas/fixed-cost-registry.md` §1) at the end-of-Mode-1 registry
  refresh.

A PM note `delegate to <dev>` on a stale-work ping promotes it exactly like a due-today
Flag (note-interpreter § Delegate — entry condition includes `fc_row_type:
stale_work_ping`), per the locked promotion gate: Flag AND (`pm_task_due_today` OR
`fc_row_type: stale_work_ping`). Follow-up reminder rows (`fc_row_type:
follow_up_reminder`) do NOT promote via delegate — a reminder isn't dev-assignable work, it
resolves or snoozes. Notes `resolved` / `client sent it` / `no longer needed` / `snooze
<period>` on a reminder row resolve or defer the ledger ask (note-interpreter § Fixed-cost
resolution notes).

Rule 20 holds: no open asks + no stale work → this file emits nothing.

## What this tracker does NOT do

- Never emits Create subtask / Reopen / Hand off / Create parent task — Flags only.
- Never auto-sends anything to the client or AM (rule 5 — Flags have no executor).
- Never resolves an ask on vague activity; never reminds without re-verifying the anchor.
- Never pings tasks assigned to the PM (those are the PM's own; due-today logic covers them).
- Never runs on non-fixed-cost signals.
