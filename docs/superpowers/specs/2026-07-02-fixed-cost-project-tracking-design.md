# Fixed-Cost Project Tracking — Design

**Date:** 2026-07-02 (v3 — adds two-lane state model: client-ask follow-ups + stale-work pings; v4 2026-07-03 — mail-primary activity lane)
**Status:** v2 approved by Krishna; v3 additions from PM interview (relayed by Krishna), pending review; v4 approved by Krishna 2026-07-03, pending implementation
**Target surface:** Claude skill (`pm-task-assignment`) only. n8n port inherits later.
**Source of ask:** PM team feedback — "Track fixed cost projects and add them in todos based on activities and updates; Active projects + Fixed cost project type."

> **v2 change:** v1 assumed no PM filter on `list_projects` and designed a weekly 77-call
> discovery sweep. Per Krishna's direction, v2 **assumes Orbit will support
> `list_projects(development_owner=<PM_user_id>)`** and makes filtered discovery the primary
> path — with a mandatory runtime guard (the param is *silently ignored* today, so the skill
> must detect non-application, never trust it) and the v1 sweep retained as the automatic
> fallback (Appendix A).
>
> **v3 change (PM interview):** fixed-cost projects split into two operational lanes —
> (a) **waiting on client** (we asked the client for something; work partly on hold; PM needs
> a weekly follow-up reminder) and (b) **in development** (assigned work in motion; PM needs a
> *specific*, never generic, status ping when assigned work goes stale). Adds the Client-Ask
> Ledger, the fixed-cost state tracker (`synthesis/fixed-cost-state.md`), and a hard
> **zero-false-alarm rule**: a delivered ask must auto-resolve from activity and never remind.
> See §10.
>
> **v4 change (2026-07-03, PM feedback round 2):** Orbit emails the PM a notification for
> every event on their projects. v4 makes those notification mails the **primary daily
> event source for the fixed-cost lane**, replacing the 20–30 unfiltered per-project
> `get_activity_log` calls after an audited shadow period. Discovery (§4.2) and the daily
> roster loop (§4.3) are unchanged. See §11.

---

## 1. Problem

The skill's Mode 1 universe is user-centric: `get_user_workload(PM)` returns only tasks
**assigned to the PM**. A fixed-cost project the PM manages can move overnight — dev comments,
task completions, blockers, due-date slips — on tasks assigned to *other people*, and none of it
surfaces in the Morning Queue. PMs currently open Orbit and check each fixed-cost project by hand.

This design adds **project-centric tracking for one project type** (Fixed Cost, status Active,
managed by the running PM) without changing behavior for anything already collected.

## 2. Probed API facts (uat_orbit, 2026-07-02)

All verified live this session.

1. `list_projects(type_name="Fixed Cost", status_name="Active")` works — 77 org-wide in UAT,
   20/page (4 pages). Payload includes `project_number`, owner, AM, dates, `quoted_hours`.
2. The dedicated project-manager field is **`development_owner`** — today returned ONLY by
   `get_project_details`. It is `dev_lead` in KPI-report filters.
3. **As of probe date, `list_projects` cannot filter by `development_owner`.** Exhaustively
   tested: `account_manager_id` filters the true AM (not PM, despite its description);
   `project_owner_id` is the US-side owner; `search_value` matches title/number only;
   undocumented `development_owner` passed by name AND by ID is **silently ignored** — the
   backend drops the unknown param and returns the full unfiltered 77-project list.
   **UPDATE 2026-07-06 — filter is LIVE.** Re-probed on UAT: `list_projects(type_name="Fixed
   Cost", status_name="Active", development_owner_id=587)` returned exactly Hiten Upadhyay's
   9 active fixed-cost projects, all owner-verified. Param name is **`development_owner_id`**
   (numeric user id). `account_manager_id` remains the AM (unchanged). Item 3's "cannot
   filter" is superseded; the §4.2 guard stays as the regression tripwire.
4. `get_fixed_cost_project_kpi_report(dev_lead_ids=[PM])` filters correctly by dev lead BUT
   returns ~515K chars and is documented Super Admin / Admin only — unusable on PM tokens.
   **Rejected as a discovery source.**
5. Per-PM slice with the UI's three filters (type + PM + active) is **20–30 projects** (PM team
   observation). The Orbit UI already filters by project manager, so the backend supports it —
   this design assumes that filter gets exposed on the MCP `list_projects` tool.
6. `get_activity_log(project_id=...)` requires `project_id`; date-bounding via
   `startdate`/`enddate` (Y-m-d) is mandatory (988 entries/project unbounded). Omitting
   `user_id` returns all actors' activity.

## 3. Working assumption (directed by Krishna)

`list_projects(type_name="Fixed Cost", status_name="Active", development_owner_id=<PM_user_id>)`
will return the PM's 20–30 projects in 1–2 pages. *(No longer an assumption — validated live
2026-07-06, see §2 item 3 UPDATE.)*

**Consequence of the silent-ignore behavior (fact 2.3):** if the filter is absent or broken,
the call does NOT error — it returns the org-wide list. The skill can therefore never trust
the response shape alone. Every discovery call runs the **filter guard** (§4.2). Guard failure
= automatic fallback to the Appendix A sweep, plus an Incident so the operator knows the
assumption broke.

## 4. Architecture

### 4.1 Component: Fixed-Cost Registry (new Notion sub-page)

Sub-page under the Notion parent (sibling of Run Log / Incidents / Preferences). With live
filtered discovery, the registry is a **rendered snapshot + manual-override surface**, not the
source of truth.

Contents:
- Header callout: active fixed-cost project **count** + `last_refresh` timestamp +
  `discovery_mode: filtered | fallback-sweep` + `last_successful_discovery` timestamp.
  `last_refresh` bumps on every run regardless of outcome; `last_successful_discovery` bumps
  ONLY when this run's returned `discovery_succeeded` boolean (§4.2) is `true` — a guard pass
  or a completed Appendix-A sweep — and NEVER when the fallback merely reuses the last-known
  tracked set (that branch also reports `discovery_mode: fallback-sweep`, but
  `discovery_succeeded: false`, distinguishing the two so a bare reuse can't self-renew the
  ≤7-day freshness check forever). It is the field the guard-trip fallback (§4.2) reads to
  decide whether a last-known-good reuse is still fresh.
- Inline table, one row per project: `Project` (`<Title> (#<project_number>)`), `project_id`
  (internal, for tooling), `Client`, `Due date`, `Added by` (filter | manual), `Added on`.
- **Manual rows are honored**: a PM may pin a project the filter doesn't return (e.g. they
  co-manage it). Manual rows join the daily union set and are never auto-removed — if discovery
  stops returning them AND `get_project_details.status` is not Active (not just Closed), they
  get annotated ` — verify: closed?`, not deleted.
- Refreshed by Mode 1 on every fire (cheap — the discovery call already ran).
- Client-Ask Ledger — second inline table on this page; see §10.2.

Read/write path failure → non-blocking: log Incident, proceed with discovery result directly
(registry is a mirror, so losing it loses only manual pins for that run).

### 4.2 Component: Daily discovery + filter guard (inside collection sub-agent)

**Registry read is the main routine's job, not the collector's.** After preflight, the MAIN
ROUTINE reads the Fixed-Cost Registry page once and builds a `registry_snapshot` object —
`{tracked_projects[] (incl. manual pins), last_successful_discovery, tracked_count, ledger,
fc_state}` — then passes it INTO the collection sub-agent at dispatch time. The collector's
allowlist stays Orbit-only; it never calls Notion itself. Every reference below to "the
registry" resolves against this passed-in snapshot, not a live fetch.

Every Mode 1 fire, before the activity loop:

1. `list_projects(type_name="Fixed Cost", status_name="Active",
   development_owner_id=<PM_user_id>)` — page until `has_more` false (expected 1–2 pages).
2. **Filter guard** (mandatory, cheap):
   a. **Plausibility check:** result `total` > 40 AND `total` ≥ 2× the registry snapshot's
      last-known tracked count → treat as filter-ignored. An empty or unknown registry (first
      run, or no last-known count) makes `total` > 40 alone sufficient.
   b. **Spot check:** `get_project_details` on ONE result project (rotate daily) → assert
      `development_owner == PM`. Mismatch → filter-ignored.
   c. Filter-ignored → discard result. If the snapshot's `last_successful_discovery` is ≤ 7
      days old AND its `tracked_projects[]` is non-empty → reuse that tracked set for today
      (`discovery_mode: fallback-sweep`, **`discovery_succeeded: false`** — a mere reuse, not
      a real discovery). Otherwise (stale, or empty because this is a first run /
      freshly-created registry — which always sweeps) → run the Appendix A sweep
      (`discovery_mode: fallback-sweep`, **`discovery_succeeded: true`** — a completed sweep
      IS a real discovery). Either way, write an Incident (`fixed-cost discovery filter not
      applied`).
3. Guard passed → project set = filtered result ∪ registry manual pins. `discovery_mode:
   filtered`, **`discovery_succeeded: true`**. Refresh registry snapshot.

New projects appear same-day (the filter call is fresh every morning); Closed projects drop
same-day. No weekly maintenance job, no Gmail fast-lane — both were latency/correctness
workarounds for the missing filter and are **deleted from this design** (they survive only
inside Appendix A as fallback machinery).

### 4.3 Component: Daily activity sweep (extension of collector, one merged loop)

> **v4 note:** the fixed-cost half of this loop (the unfiltered `get_activity_log` calls) is
> superseded by mail-primary mode after the shadow audit passes — see §11. The roster calls
> and the workload half stay exactly as written here.

There is exactly **one** daily loop. No second sweep.

- Discovery output shape: `fixed_cost_projects[]` items carry `{project_id, project_number,
  title, client_name, sub_client_name, due_date, added_by}` — `sub_client_name` comes straight
  off the `list_projects` payload (null allowed). `added_by: filter | manual` is stamped by the
  collector; `Added on` first-seen bookkeeping (the registry's date-first-seen column) is the
  WRITER's job at registry refresh, not the collector's.
- Loop set = `workload.by_project[] ∪ discovery set (incl. manual pins)` (deduped union).
- Fixed-cost projects (whether or not also in workload): `get_activity_log(project_id,
  startdate=lookback, timezone=Asia/Kolkata, per_page=500, page-on-has_more)` — **no `user_id`
  param** → all actors.
- Workload-only projects: unchanged existing call (`user_id=PM`).
- Overlap rule: one unfiltered call per project, never two. The unfiltered result is a strict
  superset of the PM-filtered result — zero signal loss, one call saved.
- Monday lookback override applies identically (≥72h).
- Output: fixed-cost signals tagged `origin: fixed_cost_registry`; per-task `latest_signal`
  computed the same way (flatten, newest-first).
- **Open-task roster (feeds FC-4 staleness).** The activity loop above is lookback-bounded, so
  it cannot see a task with zero events inside the window — exactly the tasks FC-4 needs to
  find. For each project in the fixed-cost discovery set, ALSO call
  `get_project_task_list(project_id, is_completed="incomplete")` — one call per project,
  parallel with that project's `get_activity_log` call. Output: per-project
  `open_task_roster[]`: `{task_id, title, assignee, assignee_is_pm, due_date,
  last_activity_at}` (`last_activity_at: null` when the payload lacks a timestamp).
- Runs inside the Lever-2 collection sub-agent; raw logs die there.

### 4.4 Component: Matcher integration (no new jobs)

- **Input universe (widened).** The fixed-cost signal set is Orbit signals tagged
  `origin: fixed_cost_registry` PLUS any Gmail signal that Job 4b Pass 1 cross-links to an
  Orbit signal carrying `origin: fixed_cost_registry` PLUS any Gmail signal Job 4b **Pass
  1b** (§10.4) resolves standalone — no new Orbit calls, just a match against the tracked
  fixed-cost project set already in memory — marking it `fixed_cost_linked: true`. Pass 1b
  is the mechanism that actually closes the Gmail-blindness gap: Pass 1 only links Gmail
  signals TO Orbit signals, so a Gmail-only client delivery (no same-morning Orbit activity
  to link against) would otherwise never resolve. With Pass 1b, an email-only client
  delivery can resolve an open ask (FC-1, §10.4) and an email-only AM-relayed request can
  open one (FC-2, §10.4), even with no corroborating Orbit-side signal that morning.
- Fixed-cost signals enter Jobs 1–11 through the same gate as workload signals.
- Bar: **actionable-for-PM only** — blockers/questions aimed at the PM, AM- or client-authored
  comments, task completions that unblock a next step, due-date slips, new unassigned tasks.
  Routine dev chatter drops (`filter_reason: fixed_cost_routine_activity`).
- Verbs unchanged (5 locked verbs). Tasks assigned to other devs that are merely progressing
  produce **no row**.
- Rule 20 intact: quiet fixed-cost projects add zero rows.
- Deep-read (Job 7) applies only to surviving rows — unchanged mechanics, `get_task_details`
  once per row.

### 4.5 Component: Project Pulse (new writer block, zero extra calls)

Rendered on today's dated page **below the Morning Queue database**, as a toggle:
`Fixed-Cost Pulse — N projects, M with overnight movement`.

- One line per project **with movement**, rule-22 project format (title before `#`):
  `1828 Eng Landing (#15942) — 2 tasks completed, 3 comments, due Sep 10`. Composed from
  activity-log data already collected. No deep-reads.
- Projects without movement collapse to a single tail line: `14 projects — no overnight
  activity`.
- Purpose: the "PM wants updates on ongoing projects" half of the feedback, served passively
  while promotion criteria remain unknown. Future iteration may promote recurring pulse
  patterns to rows once PMs articulate what matters.

### 4.6 Data flow

```
DAILY Mode 1 (one sweep):
  collection sub-agent:
    list_projects(Fixed Cost, Active, development_owner_id=PM) ×1–2 pages
      → FILTER GUARD (plausibility + 1-project spot check)
         ├─ pass → discovery set (+ registry manual pins) → refresh Registry snapshot
         └─ fail → Appendix A sweep or last-known-good registry + Incident
    get_user_workload(PM) ──────────────┐
    discovery set ──────────────────────┤→ union project set
    get_activity_log × (union):         │   fixed-cost projects: no user_id (all actors)
                                        │   workload-only:       user_id=PM (unchanged)
    → distilled signals (fixed-cost ones tagged origin: fixed_cost_registry)
  main context:
    matcher Jobs 1–11 (unchanged bar) → rows ──→ Morning Queue
    pulse composer (from already-collected activity) → Fixed-Cost Pulse toggle
```

## 5. Cost profile

| Path | Added calls | Context impact |
|---|---|---|
| Daily discovery | 1–2 `list_projects` + 1 spot-check `get_project_details` | Dies in sub-agent |
| Daily activity | +20–30 `get_activity_log` + 20–30 `get_project_task_list` (both date-/scope-bounded, parallel, in sub-agent). **v4:** the `get_activity_log` half drops to 0 in mail-primary mode (§11.6) | Raw dies in sub-agent; main context gains only surviving-signal text + pulse lines |
| Deep-read | Only for surviving fixed-cost rows | Same per-row cost as today |
| Fallback (guard failure only) | ~81-call sweep, ≤ weekly | Throwaway sub-agent |

## 6. Error handling

| Failure | Behavior |
|---|---|
| Discovery filter silently ignored (guard trips) | Fallback sweep or last-known-good registry; Incident logged; `discovery_mode: fallback-sweep` |
| Discovery call fails outright | Registry snapshot's last-known tracked set stands (fallback-sweep mode); Incident; workload-only only when the snapshot is empty/unavailable |
| Registry page unreadable | Proceed with discovery result (lose manual pins for the run); Incident |
| Activity-log failure on a project | Existing 4-attempt backoff; project skipped after exhaustion; noted in run log |
| Open-task-roster call failure on a project | Existing retry policy; that project skips FC-4 (stale-work pings) this run — reminders (FC-1..FC-3) unaffected; noted in run log |
| Registry empty + discovery empty (new PM) | Valid state — zero fixed-cost calls, `0 projects` pulse header |

## 7. Files to touch (implementation plan input)

- `SKILL.md` — universe description (§ Data sources Orbit row), architecture overview Step 1
  + Step 4 (state tracker), new rule: fixed-cost layer non-blocking + filter guard mandatory
  + zero-false-alarm rule; five-verb section untouched (v3 rows are Flags).
- `collectors/orbit.md` — discovery call + filter guard, union loop, unfiltered activity_log
  for fixed-cost projects, overlap rule, `origin` tag, registry snapshot refresh.
- `synthesis/matcher.md` — fixed-cost signal gating notes (`fixed_cost_routine_activity` drop
  reason); hook: run `synthesis/fixed-cost-state.md` after Job 4b, before Job 5; no new
  matcher jobs.
- NEW `synthesis/fixed-cost-state.md` — FC-1..FC-4 (resolution scan, ask detection,
  follow-up reminders, stale-work pings) per §10.4.
- `synthesis/note-interpreter.md` — resolution/snooze vocabulary on v3 Flag rows
  (`resolved` / `client sent it` / `no longer needed` / `snooze <period>`) + existing
  delegate-promotion applies.
- `writers/notion.md` — Fixed-Cost Pulse toggle block; registry snapshot write; Client-Ask
  Ledger writes; machine state block; ping required-field gate (task title + assignee +
  last-movement or drop-to-audit).
- `schemas/parent-page.md` — Fixed-Cost Registry sub-page in the parent structure.
- NEW `schemas/fixed-cost-registry.md` — registry page schema (snapshot table + Client-Ask
  Ledger + machine state block + discovery_mode header).
- `schemas/preferences-page.md` — new knobs: `follow_up_reminder_days` (default 7),
  `fixed_cost_stale_days` (default 2 business days).
- `preflight.md` — registry existence check (create-if-missing on first run, like Preferences).
- `writers/run-log.md` — fixed-cost stats in run entry (projects discovered, guard result,
  `signal_count`, pulse counts incl. `pulse_projects_with_movement`, asks opened/resolved,
  reminders emitted/suppressed, pings emitted/throttled).
- `modes/mode-1-morning-collection.md` — orchestration narrative for the fixed-cost lane
  (registry read after preflight, `registry_snapshot` pass-through into the 1e collection
  sub-agent, discovery + activity-loop dispatch, fixed-cost lane stats incl. `signal_count`
  and `pulse_projects_with_movement` folded into the Mode 1 run-log summary).
- `modes/mode-2-execution.md` — the fixed-cost ledger-touch carve-out (resolve/snooze notes on
  a `follow_up_reminder` Flag route to `writers/notion.md` § Flow — Mode 2 ledger touch without
  a verb change) and the third verb-changing promotion (`fc_row_type: stale_work_ping` +
  `delegate to <dev>`).
- `modes/monthly-archival.md` — the four-page operational tail (Run Log, Incidents,
  Fixed-Cost Registry, Preferences) and Step 8 (Client-Ask Ledger archival: Resolved /
  Resolved-manual rows with `Resolved on` older than 60 days move into an `Archived Asks`
  toggle).
- `ROUTINE-ENTRYPOINTS.md` — baked prompts updated for the fixed-cost lane.
- `schemas/run-log-database.md` + `schemas/run-log-detail-page.md` — home for the fixed-cost
  stats block (`signal_count`, `pulse_projects_with_movement`, and the rest of
  `fixed_cost_stats`) and its decision-trace lines.
- `README.md`, `ENVIRONMENTS.md` — one-line mentions of the fixed-cost lane.
- (Appendix A fallback, only if guard failure becomes chronic: NEW
  `modes/weekly-registry-reconcile.md` + `ROUTINE-ENTRYPOINTS.md` 4th prompt — deferred,
  not built up front. The inline fallback in `collectors/orbit.md` covers occasional failure.)

## 8. Verification

1. **Guard test first (the assumption test):** run discovery on UAT as-is (filter not yet
   shipped) → guard MUST trip (77 ≈ org-wide, spot-check mismatch) → fallback engages. This
   proves the design safe even if the filter never ships.
2. When the filter ships: discovery returns the same 20–30 set the Orbit UI shows with
   type+PM+active filters; guard passes; registry snapshot matches.
3. Dry-run Mode 1: ONE activity call per union project; fixed-cost projects show unfiltered
   actor coverage; workload-only unchanged.
4. Dev-comment-only night on a fixed-cost project → zero rows + a pulse line.
5. AM comment on a non-PM-assigned fixed-cost task → row through the normal verb tree with
   `latest_signal_anchor` intact.
6. Delete the registry page → run proceeds on discovery alone + Incident (non-blocking proof).
7. Queue diff for workload-only signals before/after the change — must be byte-identical
   (additive guarantee).

## 9. Open items

- ~~**Filter param final name/shape** — `development_owner`? `development_owner_id`?
  `dev_lead_id`? Confirm with Orbit team before implementation; guard logic is name-agnostic
  but the call site needs the real param.~~ **RESOLVED 2026-07-06:** `development_owner_id`
  (numeric user id) — live-validated on UAT, see §2 item 3 UPDATE.
- Does `get_activity_log` without `user_id` include comment **bodies** or only field-change
  entries? (Probe during build; if bodies absent, pulse lines still work off counts; row
  gating may need a light `get_task_details` on candidate tasks.)
- Guard plausibility threshold (40) — sanity-check against production PM slices before lock.
- `fixed_cost_stale_days` semantics — calendar vs business days (weekend gap shouldn't ping
  Monday for Friday-assigned work; recommend business days, confirm with PM). **RESOLVED:
  business days** (Sat/Sun excluded) — shipped in `schemas/preferences-page.md` and
  `synthesis/fixed-cost-state.md` FC-4.
- Ask-detection precision — validate FC-2 on a sample of real fixed-cost project comment
  threads before rollout (false ledger entries are visible/deletable, but measure anyway).

---

## 10. v3 — Two-lane state model (client-ask follow-ups + stale-work pings)

### 10.1 The two lanes (from PM interview)

| Lane | Situation | PM need | Cadence |
|---|---|---|---|
| **Waiting on client** | WLIQ asked the client for something (assets, approval, content, access); some or all work is on hold behind it | Reminder naming the *specific ask*: "it's been a week since we asked for X — follow up" | Weekly per open ask |
| **In development** | Assigned work in motion on the project | Status ping naming the *specific assigned work* — NEVER a generic "any update?": task title + assignee + due date + last known movement | When assigned work goes stale (threshold, throttled) |

A project is NOT exclusively one lane. It can wait on the client for one deliverable while dev
work continues elsewhere. Therefore state is tracked **per ask**, not per project — the
project's lane is derived (has open asks → waiting-lane rows possible; has stale assigned
tasks → ping-lane rows possible; both can coexist).

### 10.2 Client-Ask Ledger (registry page extension)

Second inline table on the Fixed-Cost Registry page: one row per **open client ask**.

Columns: `Project` (`<Title> (#<project_number>)`) · `Ask` (specific, e.g. "logo assets +
brand fonts") · `Asked on` · `Source` (link to the Orbit comment / Gmail thread that made the
ask — this is the ask's anchor) · `Status` (`Open` / `Resolved` / `Resolved-manual`) ·
`Resolved on` (`YYYY-MM-DD`; set by FC-1 auto-resolution or the Mode 2 manual resolution when
`Status` changes to `Resolved`/`Resolved-manual`; empty while `Open`) ·
`Resolved by` (link to resolving signal, when auto-resolved) · `Last reminder`.

- **Ask creation (automatic):** during the daily sweep, the state tracker (§10.4) detects
  outbound client-directed requests in fixed-cost project signals — a PM/AM Orbit comment or
  AM-relayed Gmail message asking the client for something. New ask → ledger row with the
  originating signal as `Source`. Dedup: topic-match against existing open asks for the same
  project (same LLM-judgment pattern as the Create-parent-task dedup); a refreshed ask (re-ask
  of the same thing) updates that row's `Asked on` to the newer date instead of duplicating —
  the clock restarts under the same row.
- **Ask creation (manual):** PM may add a ledger row by hand (fills Project + Ask; skill
  backfills what it can). Manual rows behave identically.
- **Machine state block:** a collapsed toggle at the bottom of the registry page holds a
  fenced JSON block (machine-owned): per-ask `last_reminder`, per-task `last_ping`
  for ping throttling (§10.5) — both `YYYY-MM-DD`, no `_iso` suffix, no time component. Human
  tables stay human; timestamps that drive logic live here. Block unreadable/corrupt → treat
  as empty (worst case: one early reminder/ping, never a missed resolution — resolution state
  lives in the ledger `Status` column, not the block).
- **Date-format rule.** `Asked on`, `Added on`, `Resolved on`, and every date inside the
  `fc_state` block use `YYYY-MM-DD` uniformly. The ledger `Asked on` cell and the date segment
  of the `fc_state` ask key (`#<project_number>|<Asked on date>|<first 40 chars of Ask>`) MUST
  be the identical string — key stability depends on it.
- **Re-ask key semantics.** When FC-2 detects the same ask re-raised on an already-open ledger
  row, it updates `Asked on` to the newer date AND deletes the OLD `fc_state.asks[]` key for
  that row — the reminder clock deliberately restarts under the new key rather than inheriting
  `last_reminder` from the stale one. A missing `fc_state.asks[key]` entry for an open ask
  means "no prior reminder" (the normal case for a freshly-opened ask, and also the case right
  after a re-ask key rotation) — `max(Asked on)` alone governs the `follow_up_reminder_days`
  threshold.
- **60-day archival.** Monthly archival (`modes/monthly-archival.md` Step 8) moves ledger rows
  with `Status: Resolved` or `Resolved-manual` and `Resolved on` older than 60 days into an
  `Archived Asks` toggle at the bottom of the registry page. Still-open rows, or rows resolved
  more recently than 60 days, stay in the live ledger.

### 10.3 Zero-false-alarm rule (hard requirement)

**A resolved ask must never remind.** Scenario to prevent: we asked the client for assets
Monday; client delivered Wednesday; next Monday the PM gets "waiting on assets — follow up".
Forbidden. Three independent gates:

1. **Daily resolution scan.** Every Mode 1 fire, the state tracker matches ALL incoming
   fixed-cost signals (Orbit comments/attachments/status changes + cross-linked Gmail
   signals, per matcher Job 4b) against open ledger asks — topic-match judgment: does this
   signal deliver, answer, or make moot the ask? Match → `Status: Resolved`, `Resolved by`
   linked, ask exits the reminder pool the same morning the delivery lands. Resolution runs
   BEFORE reminder evaluation in the same run — a delivery and a due reminder on the same
   morning resolves, never reminds.
2. **Re-verify at reminder time.** Before emitting any reminder row, deep-read the ask's
   anchor (its Orbit task via `get_task_details` / its Gmail thread) and confirm the ask is
   still unanswered. This catches deliveries the daily scan missed (e.g. client replied in a
   thread the PM's inbox filter dropped, delivery arrived as a new attachment with no
   comment). Verification fails-closed: cannot confirm still-open → do NOT remind; emit
   nothing; log `reminder suppressed — unverified` to run log for audit.
3. **The delivery is its own morning row.** Unchanged existing behavior: a client delivery is
   an actionable signal (Gmail safety net / activity row) and surfaces in the queue the
   morning it arrives — the PM sees "we got it," and that same signal is what gate 1 consumed.

PM note override: `resolved` / `client sent it` / `no longer needed` on a reminder row →
note-interpreter marks the ledger row `Resolved-manual`. `snooze 2 weeks` → bumps
`last_reminder` forward.

### 10.4 Component: fixed-cost state tracker (NEW `synthesis/fixed-cost-state.md`)

Runs in the MAIN context (it is judgment, not plumbing), after matcher Job 4/4b grouping and
cross-linking, before Job 5 verb classification (matcher Job 4c). Inputs — widened universe:
signals tagged `origin: fixed_cost_registry` PLUS Gmail signals that Job 4b Pass 1
cross-linked to an `origin: fixed_cost_registry` Orbit signal PLUS Gmail signals Job 4b
**Pass 1b — Fixed-cost standalone Gmail resolution** resolved directly against the tracked
fixed-cost project set (`fixed_cost_linked: true`, no new Orbit calls — Pass 1b walks
Gmail signals Pass 1 left unlinked and matches them via `context_link.project_id_candidates`
+ topic/keyword match against tracked project titles/numbers) (§4.4) — Pass 1b is what
actually delivers the standalone case, since Pass 1 alone only cross-links Gmail signals TO
Orbit signals and never resolves a Gmail-only signal on its own — so an email-only client
delivery can resolve an open ask (FC-1) and an email-only AM-relayed request can open one
(FC-2), even with no corroborating Orbit-side signal that morning — plus ledger state and
the machine state block. Four jobs, in order:

- **FC-1 Resolution scan** (gate 1 above) — match incoming signals (Orbit + cross-linked
  Gmail) against open asks; resolve + link. On match: `Status: Resolved`, `Resolved by` set to
  the resolving signal, `Resolved on` set to today (`YYYY-MM-DD`).
- **FC-2 Ask detection** — detect new outbound client asks in today's signals (Orbit +
  cross-linked Gmail); append ledger rows (with topic-dedup).
- **FC-3 Follow-up reminders** — for each still-open ask where
  `today − max(asked_on, last_reminder) ≥ follow_up_reminder_days` (Preferences, default 7):
  re-verify anchor (gate 2) — deep-read the ask's anchor AND re-read any Gmail context signals
  Job 4b Pass 1 cross-linked to that anchor in TODAY's collection (a client may have replied on
  a thread the anchor's task never mirrored) — then emit a **Flag** row:
  `Summary:` "Waiting on client 8 days: logo assets + brand fonts — follow up via <AM> —
  Board & Staff Page Layout Update (#16046)" (rule-22 project format: title before `#`).
  `latest_signal_anchor` = the original ask signal. Task Brief carries the ask context + what
  it blocks. AM-framing rule 23 applies — the follow-up routes *via the AM as client relay*,
  never "ping the client directly". Update `last_reminder`.
- **FC-4 Stale-work pings** — for in-development fixed-cost projects: walk the collector's
  `open_task_roster[]` (§4.3 — one `get_project_task_list` per tracked project, since the
  lookback-bounded activity log alone cannot see a task with zero events inside the window)
  and filter to open tasks assigned to a non-PM dev. A task is stale when its roster
  `last_activity_at` is ≥ `fixed_cost_stale_days` (Preferences, default 2 business days) old
  AND the current run's unfiltered activity log shows no newer event for it (comments, status,
  time-track, attachments); `last_activity_at: null` → run the light deep-read first to
  establish last movement before judging staleness. AND `last_ping` for that task ≥
  `fixed_cost_stale_days` ago → emit a **Flag**
  row that is *specific by construction*:
  `Summary:` "No movement 3 days: \"Homepage stats counter markup\" — Nikunj Bhalodia, due
  Jul 4 — Homepage Stats Counter (#15927)" (rule-22 project format). Row candidates carry
  three **structured fields** alongside the rendered Summary/Task Brief — `stale_task_title`,
  `stale_assignee_name`, `last_movement_excerpt` — not just embedded prose. Task Brief = last
  known movement (from `latest_signal`) + what the task was assigned for. A light
  `get_task_details` (same mechanics as the due-today light deep-read, rule 18 Flag) feeds the
  brief when the activity log alone is too thin — **the generic-ping ban is structural: the
  emission gate (and the writer's render gate) check that `stale_task_title`,
  `stale_assignee_name`, and `last_movement_excerpt` are all populated as fields, not by
  parsing prose** (same enforcement pattern as rule 24's Triggered-by line). A candidate
  missing any of the three MUST NOT be emitted. Update `last_ping`.

**Output.** The tracker returns `fc_output = {row_candidates[], ledger_mutations[],
fc_state_patch}`, held in memory. `row_candidates[]` join Job 5's input set (both types
pre-classified `Flag`); `ledger_mutations[]` and `fc_state_patch` are NOT written to Notion by
this file — the Notion writer applies them to the Client-Ask Ledger and `fc_state` block in its
registry-refresh flow at the end of Mode 1 (no mid-synthesis Notion writes).

**Matcher Job 6 short-circuit.** Rows carrying `fc_row_type` skip Job 6 (assignee
recommendation) entirely — the same way a bare `pm_task_due_today` Flag does. Recommended
Assignee stays `—`; no pod-inference, no candidate pool, no decision tree. For
`stale_work_ping` rows, the CURRENT assignee named in `stale_assignee_name` is roster fact from
this tracker, not a Job 6 recommendation, and must not be overwritten.

Both row types are **Flag** verb — rule 18's five verbs untouched; Flags have no executor, so
Mode 2 is untouched except for note-interpreter's new resolution/snooze vocabulary and the
ledger-touch carve-out below. A PM note `delegate to <dev>` on a **stale-work ping**
(`fc_row_type: stale_work_ping`) promotes it exactly like a due-today Flag — note-interpreter's
delegate-promotion entry condition now includes `fc_row_type: stale_work_ping` alongside
`pm_task_due_today`, and Mode 2 now documents THREE verb-changing promotions in total (the
third being this one; the other two are the due-today delegate and the Possible-Orbit-miss
create path). **Follow-up reminder rows (`fc_row_type: follow_up_reminder`) do NOT promote** —
a reminder isn't dev-assignable work; it resolves or snoozes instead. Those notes
(`resolved` / `client sent it` / `no longer needed` / `snooze <period>`) route through
`writers/notion.md` § Flow — Mode 2 ledger touch: `Status → Resolved-manual`, `Resolved by`,
`Resolved on` set on resolve; `fc_state.asks[key].last_reminder` bumped forward on snooze. This
is Mode 2's ONLY write to the Fixed-Cost Registry — the row itself stays `Flag` (no verb
change), `Status` flips to `Done`, and `Outcome` records the ledger-touch result.

Ordering note: FC-1 strictly before FC-3 within a run (same-morning delivery must resolve
before reminders evaluate). FC-2 before FC-3 so a brand-new ask starts its clock today (a
new ask never reminds on day 0 — threshold ≥ 7 days guarantees it).

### 10.5 Noise & cost control

- Reminders: at most one per ask per `follow_up_reminder_days` window (ledger
  `last_reminder`).
- Pings: at most one per task per `fixed_cost_stale_days` window (state block `last_ping`);
  a task that stays stale keeps re-pinging at the throttled cadence with an escalating count
  ("no movement 6 days") — visible pressure without daily spam. A throttled ping is counted in
  the run-log `pings_throttled` stat only — unlike suppressions (FC-3 gate 2 fail-closed) and
  drops (FC-4 generic-ping ban), a throttled ping gets NO per-item decision-trace line, since
  it's routine rate-limiting rather than a content or verification decision worth auditing.
- Extra calls: FC-3 re-verify = 1 `get_task_details` (or Gmail thread read) per due reminder
  (typically 0–3/morning); FC-4 light read = 1 `get_task_details` per NEW stale ping
  (typically 0–5/morning). Daily collection additionally adds one `get_project_task_list` per
  tracked fixed-cost project (§4.3 — 20–30 small calls, inside the collection sub-agent) to
  build the `open_task_roster[]` FC-4 needs; everything else in FC-1..FC-4 reuses that roster
  plus the already-collected sweep data.
- Rule 20 intact: no open asks + no stale work → zero v3 rows. The Pulse (§4.5) still gives
  passive daily visibility either way.

### 10.6 v3 verification additions

8. Seed an open ask; inject a client-delivery signal (comment with attachment on the ask's
   task); run Mode 1 → ledger row auto-resolves with `Resolved by` link; NO reminder row;
   delivery surfaces as its own row. Run again 7+ days later (simulated) → still no reminder.
9. Seed an open ask 8 days old with no delivery → exactly one Flag reminder row, correct ask
   text + AM-relay framing + anchor = original ask signal; `last_reminder` updated; next-day
   run emits nothing new.
10. Same-morning collision: delivery signal AND reminder due on the same run → resolves,
    never reminds (FC-1-before-FC-3 ordering proof).
11. Re-verify fail-closed: make the anchor task unreadable (permission/404) → reminder
    suppressed + run-log audit line, no row.
12. Stale assigned task (no activity ≥ threshold) → ping row contains task title, assignee,
    due date, last-movement line (required-field gate); re-run next day → no duplicate ping
    (throttle); day after threshold → re-ping with escalated day count.
13. Generic-ping ban: force a ping candidate with missing assignee/last-movement → row fails
    the writer gate and is dropped to run-log audit, not rendered generic.

---

## 11. v4 — Mail-primary activity lane (2026-07-03 PM feedback)

### 11.1 Feedback and decisions

PM feedback round 2 (relayed by Krishna):

1. *"Slow lane: populate proper stuff once a week (Friday); daily just track the project
   count — 19 → 20 means the list changed."*
2. *"Orbit mail notifications are enabled for all PMs — every project can be tracked from
   mail: client-dependent holds AND daily progress. PMs take updates on mostly all projects
   daily; a light, context-rich flag helps."*

Decisions (Krishna, 2026-07-03):

- **Filter WILL be fixed** *(came true — live-validated 2026-07-06 as `development_owner_id`, see §2 item 3 UPDATE)* — proceed on the §3 assumption. Daily filtered discovery already
  refreshes the list (and its count) every day, which supersedes feedback item 1 outright:
  the weekly-populate + daily-count-delta scheme was a workaround for the broken filter, and
  the §4.2 guard + Appendix A remain the safety net if the filter regresses. **Discovery is
  unchanged in v4.**
- **Mail coverage confirmed**: the PM receives an Orbit notification email for every event
  on their projects (including tasks assigned to others and client comments). Confirmed
  verbally — the notification **format** is still unproven, hence the shadow audit (§11.3).
  This mail-covers-all-actionable-classes assumption is exactly what FC-0 audits: if Orbit
  permanently sends no notification mail for some actionable event class, that class's
  events surface as recurring coverage gaps, which keep (or return) the lane to shadow —
  the safe direction.
- **Flags/digest unchanged**: FC-1..FC-4 + the Pulse toggle stay as built. Feedback item 2's
  "light context-rich flag" is served by the existing FC-4 ping + Pulse; no new row type.
- **Friday** is the audit anchor day (Krishna's choice; simple, precedes the weekend gap).

### 11.2 Component: Orbit-notification mail parsing (Gmail collector extension)

`collectors/gmail.md` currently EXCLUDES Orbit notification emails ("already covered by the
Orbit collector"). That exclusion assumed workload-lane coverage; for the fixed-cost lane the
equivalent Orbit coverage costs 20–30 unfiltered multi-page `get_activity_log` calls daily.
v4 lifts the exclusion **for tracked fixed-cost projects only**:

- **Scope gate.** The main routine already reads the registry once per run and passes
  `registry_snapshot` into the Orbit collection sub-agent. v4 passes the same snapshot into
  the **Gmail collector** too. A notification mail is collected iff it resolves to a project
  in the tracked set; all other Orbit notification mails stay excluded exactly as today.
- **Detection**: existing rule (sender matches Orbit's notification-from address) — already
  specified in gmail.md's exclusion list; v4 reuses it as the inclusion test.
- **Parse per mail** (subject + body HTML): `{project (name and/or #number), task title,
  event_type (comment | status | assignment | due-date | attachment | task-created |
  task-completed), actor, excerpt, mail timestamp}`. Project resolution against
  `registry_snapshot` by `#project_number` first, title substring second.
- **Signal shape**: mirrors an Orbit activity signal, tagged `origin: fixed_cost_mail`, with
  `latest_signal` anchor built from the mail (actor + timestamp + excerpt) — rule 24 holds.
- **Parse failure** on an individual mail → Incident line (`fc_mail_parse_failure`, with
  subject) + skip that mail. Never guess a project match.
- Signals with `origin: fixed_cost_mail` join the matcher exactly where
  `origin: fixed_cost_registry` signals enter (§4.4): same Jobs 1–11 bar, same FC-1/FC-2
  consumption in Job 4c, same Pulse composition input.

### 11.3 Activity-source state machine (shadow → mail-primary)

Registry header gains two machine fields: `Activity source: shadow | mail-primary` and
`Clean audits: <n>` (both maintained by the writer at registry refresh, like the existing
header fields).

- **shadow** (initial): BOTH sources run — the §4.3 unfiltered `get_activity_log` loop AND
  §11.2 mail parsing. Orbit signals remain the authoritative feed into the matcher (mail
  signals are collected but deduplicated against Orbit signals by project + task + event
  type + timestamp proximity; the Orbit copy wins). Zero behavior change for the PM.
- **Friday audit** (shadow mode only): runs as a step inside the state tracker
  (`synthesis/fixed-cost-state.md`, before FC-1 — it sees both distilled signal sets in
  matcher Job 4c's input). Compare the two event sets per tracked project over the run's
  lookback window. Every activity-log event whose class the
  matcher could act on (comment, status change, assignment, due-date move, task completion,
  task creation, attachment) that has NO corresponding mail-derived event → `mail_coverage_gap` Incident
  naming project, task, event. Projects first discovered by the same run are excluded from
  the compare (the mail feed's scope gate was dispatched with the pre-discovery
  `registry_snapshot`, so it structurally could not cover them — not a mail-reliability
  signal). Any gap → `Clean audits` resets to 0. No gaps → increment.
- **Cutover**: `Clean audits` reaches 2 → writer flips `Activity source: mail-primary` at
  registry refresh and returns a flip audit string; the run-log writer renders it as a
  decision-trace line (sole Run Log writer invariant holds).
- **mail-primary**: the fixed-cost half of the §4.3 activity loop is OFF (no unfiltered
  `get_activity_log` calls). Mail signals are the fixed-cost event feed. The roster loop
  (`get_project_task_list` per tracked project) and the workload lane (user_id=PM activity
  calls for workload projects) run unchanged.
- **Re-audit** (mail-primary mode): on the **first Friday of each month**, run the shadow
  loop once (activity_log + mail side by side) and audit. Clean → stay mail-primary. Gap →
  auto-revert to `shadow`, reset `Clean audits: 0`, Incident. This catches silent
  notification-settings drift.
- **Gmail collector failure** in mail-primary mode (collector-level, after existing
  retries): that run falls back to the shadow-mode Orbit loop for the fixed-cost lane
  (fail-closed — never run blind), Incident logged; mode is NOT flipped by a one-off
  failure.

### 11.4 What each FC job consumes post-cutover

| Job | Source in mail-primary mode | Change from v3 |
|---|---|---|
| FC-1 resolution scan | mail-derived events (client replies/deliveries land as notification mails AND as direct client emails via Pass 1b) | source swap only; gates unchanged |
| FC-2 ask detection | mail-derived events + Gmail Pass 1b as today | source swap only |
| FC-3 follow-up reminders | ledger + re-verify at emission (1 `get_task_details` or Gmail read per due reminder) | **unchanged** — re-verify was already a live lazy call |
| FC-4 stale-work pings | `open_task_roster[]` from the daily roster loop, exactly as v3, PLUS a same-morning-event guard that reads `origin: fixed_cost_mail` signals instead of the (now-off) unfiltered activity log | roster source **unchanged**; the secondary same-morning guard becomes source-conditional (same feed-selection rule as FC-1/FC-2, § 11.4 note below) |
| Pulse | composed from mail-derived movement, via a NEW `mail_activity_summary[]` rollup the Gmail collector emits (§11.2) — the existing `activity_summary[]` from the unfiltered activity loop is not populated in mail-primary mode | source swap, with a dedicated mail-side rollup (writer picks source by `fixed_cost_activity_loop_ran`; Orbit wins if both present) |

**Correction (v4 sweep, 2026-07-06).** `synthesis/fixed-cost-state.md`'s FC-4 spec text
(originally written in §10.4, before v4 existed) hard-coded "the current run's unfiltered
activity log" as the secondary guard against a same-morning event the roster snapshot
predates. That guard is unreachable in `mail-primary` mode on non-audit days, since the
unfiltered activity log doesn't run then. The implementation now conditions the guard on
`activity_source`, same as FC-1/FC-2's input-universe rule: the unfiltered activity log in
shadow mode / the monthly re-audit / a Gmail-failure fallback, or `origin: fixed_cost_mail`
signals in mail-primary mode otherwise. This section is amended to match.

**Why the roster loop stays daily (rejected alternative).** FC-4 staleness is task-granular;
project-level mail silence would miss a stale task inside an otherwise-noisy project. A
weekly Friday roster snapshot corrected by mail events (completions remove, creations add)
was considered and rejected: stateful, drift-prone, and it trades ~20–30 small daily calls
for a new class of false-alarm risk — against the §10.3 hard rule. The expensive calls were
the multi-page all-actor activity logs; those are what v4 eliminates.

### 11.5 Data flow (mail-primary)

```
DAILY Mode 1 (mail-primary):
  Gmail collector (2a):                       Orbit collection sub-agent (1e):
    registry_snapshot (from main routine)       registry_snapshot (as v3)
    orbit-notification mails, tracked set  →     list_projects(…dev_owner=PM) + guard   (unchanged)
    parse → origin: fixed_cost_mail signals      get_user_workload(PM)                  (unchanged)
    rollup → mail_activity_summary[] (Pulse)     get_activity_log × workload projects   (user_id=PM, unchanged)
                                                 get_project_task_list × tracked set → open_task_roster[]
                                                 [NO unfiltered get_activity_log × tracked set]
  matcher: Jobs 1–11 + Job 4c (FC-1..FC-4) — fixed_cost_mail signals enter where
           fixed_cost_registry signals did; FC-4 walks the roster as v3
  writer:  rows + Pulse + registry refresh (Activity source / Clean audits fields)
```

### 11.6 Cost profile delta

| Path | shadow | mail-primary |
|---|---|---|
| Fixed-cost activity | 20–30 `get_activity_log` (multi-page) + mail parsing | **0 Orbit calls** + mail parsing |
| Roster | 20–30 `get_project_task_list` (small) | same |
| FC-3/FC-4 lazy verifies | 0–8 `get_task_details` | same |
| Audit overhead | Friday compare (in-context, no extra calls) | monthly first-Friday shadow run (~20–30 calls, 1×/month) |

Mail parsing cost lives in the Gmail collector sub-agent (already dispatched daily); raw
mail bodies die there.

### 11.7 Error handling additions

| Failure | Behavior |
|---|---|
| Individual mail unparseable | `fc_mail_parse_failure` Incident + skip; never guess |
| Mail names project not in tracked set | Excluded (scope gate) — not an error |
| Audit finds coverage gap | Incident + `Clean audits: 0`; revert to shadow if in mail-primary |
| Gmail collector down in mail-primary | One-run fallback to shadow-mode Orbit loop; Incident; no mode flip |
| Registry header fields missing (pre-v4 page) | Treat as `shadow` / `Clean audits: 0`; writer adds fields at next refresh |

### 11.8 Files to touch (v4 delta)

- `collectors/gmail.md` — exclusion lift + scope gate + parse spec + signal shape (§11.2);
  NEW `mail_activity_summary[]` Pulse rollup (§11.4)
- `collectors/orbit.md` — activity loop conditional on `Activity source`; audit event-set
  export in shadow mode
- `modes/mode-1-morning-collection.md` — pass `registry_snapshot` to Gmail collector too;
  Friday-audit step; mode-flip narrative; note Gmail collector's `mail_activity_summary[]`
  output for the Pulse
- `synthesis/matcher.md` — accept `origin: fixed_cost_mail` in Job 4c universe + dedup rule
  (shadow mode)
- `synthesis/fixed-cost-state.md` — source-agnostic wording for FC-1/FC-2 inputs; NEW
  Friday-audit step (before FC-1, shadow mode only)
- `writers/notion.md` — registry header: `Activity source`, `Clean audits`; flip/revert
  logic; § Step 5.7 Pulse source-selection rule (`activity_summary[]` vs
  `mail_activity_summary[]`, §11.4)
- `schemas/fixed-cost-registry.md` — two new header fields
- `writers/run-log.md` — `fixed_cost_stats`: `activity_source`, `clean_audits`,
  `mail_signals`, `coverage_gaps`
- `schemas/run-log-detail-page.md` — **omitted from the original list; added at the v4
  sweep (2026-07-06)** — the Fixed-cost lane stats one-line summary example needed the
  `source`/`audits`/`mail signals`/`gaps` fields to match `writers/run-log.md`'s new keys
- `SKILL.md` — data-source table row (Gmail gains the fixed-cost notification-mail role);
  architecture step note
- `ROUTINE-ENTRYPOINTS.md` — baked-prompt sentence for the mail lane + Friday audit

### 11.9 Verification additions

14. Shadow run with seeded Orbit activity + matching notification mails → no duplicate rows
    (dedup proof); Friday audit reports 0 gaps; `Clean audits` increments.
15. Shadow Friday with one activity-log event lacking a mail counterpart →
    `mail_coverage_gap` Incident, `Clean audits` resets to 0, no cutover.
16. Two clean Fridays → `Activity source: mail-primary`; next run makes zero unfiltered
    `get_activity_log` calls (tool-trace assert) and FC-1/Pulse still populate from mail.
17. Mail-primary: unparseable notification mail → `fc_mail_parse_failure` Incident, run
    completes, no invented project match.
18. Mail-primary: Gmail collector hard-fails → run falls back to Orbit loop for that run,
    mode unchanged.
19. First-Friday monthly re-audit with an injected gap → auto-revert to shadow + Incident.

---

## Appendix A — Fallback discovery sweep (v1 design, kept as guard-failure path)

Used only when the filter guard trips (filter missing/broken) AND the registry's
last-known-good list is > 7 days stale.

Throwaway sub-agent:
1. `list_projects(type_name="Fixed Cost", status_name="Active")` — all pages (~4).
2. `get_project_details` per project (~77 org-wide, parallel) → read `development_owner`.
3. Filter to `development_owner == PM` → canonical 20–30 list.
4. Reconcile registry (add missing, prune Closed/On-Hold, annotate — never silently delete —
   manual pins it can't find), set `discovery_mode: fallback-sweep`, update `last_refresh`.

~81 cheap calls, raw dump dies in the sub-agent. This was v1's primary path; v2 demotes it to
insurance. The v1 Gmail project-notification fast-lane is dropped entirely — daily filtered
discovery makes it redundant, and under chronic fallback the sweep cadence (≤ weekly) matches
v1's original latency anyway.
