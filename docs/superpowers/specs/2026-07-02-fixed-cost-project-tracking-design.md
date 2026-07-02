# Fixed-Cost Project Tracking — Design

**Date:** 2026-07-02
**Status:** Approved by Krishna (brainstorming session)
**Target surface:** Claude skill (`pm-task-assignment`) only. n8n port inherits later.
**Source of ask:** PM team feedback — "Track fixed cost projects and add them in todos based on activities and updates; Active projects + Fixed cost project type."

---

## 1. Problem

The skill's Mode 1 universe is user-centric: `get_user_workload(PM)` returns only tasks
**assigned to the PM**. A fixed-cost project the PM manages can move overnight — dev comments,
task completions, blockers, due-date slips — on tasks assigned to *other people*, and none of it
surfaces in the Morning Queue. PMs currently open Orbit and check each fixed-cost project by hand.

This design adds **project-centric tracking for one project type** (Fixed Cost, status Active,
managed by the running PM) without changing behavior for anything already collected.

## 2. Probed API facts (uat_orbit, 2026-07-02)

These constrain the design. All verified live this session.

1. `list_projects(type_name="Fixed Cost", status_name="Active")` works — 77 org-wide in UAT,
   20/page (4 pages). Payload includes `project_number`, owner, AM, dates, `quoted_hours`.
2. The dedicated project-manager field is **`development_owner`** — returned ONLY by
   `get_project_details`. It is `dev_lead` in KPI-report filters.
3. `list_projects` **cannot** filter by `development_owner`. Its `account_manager_id` param
   filters the true AM field (tested: returns AM matches, not dev-lead matches), despite its
   description saying "project manager".
4. `get_fixed_cost_project_kpi_report(dev_lead_ids=[PM])` filters correctly by dev lead BUT
   returns ~515K chars (nested department hours + every timetrack line) and is documented
   Super Admin / Admin only — unusable on PM tokens. **Rejected as a discovery source.**
5. Per-PM slice with the UI's three filters (type + PM + active) is **20–30 projects** (PM team
   observation). The Orbit UI *can* filter by project manager, so the backend supports it — the
   filter just isn't exposed in the MCP `list_projects` tool. No fix expected from the Orbit team.
6. `get_activity_log(project_id=...)` requires `project_id`; date-bounding via
   `startdate`/`enddate` (Y-m-d) is mandatory (988 entries/project unbounded). Omitting
   `user_id` returns all actors' activity. (Per existing ORBIT-OPTIMIZATION-PLAN.md probe +
   this session.)

## 3. Decisions locked in brainstorming

| Question | Decision |
|---|---|
| Surface | Claude skill only, for now |
| PM identity field | `development_owner` (validated; not owner, not AM) |
| Universe discovery | Weekly cached sweep (KPI report rejected; Orbit API fix not happening) |
| Registry storage | Dedicated `Fixed-Cost Registry` Notion sub-page (user-visible separate record) |
| Queue output | Actionable rows via existing matcher + passive **Project Pulse** digest |
| Signal bar | Actionable-for-PM only (rule 20 intact); "what counts as an update" criteria unknown → deferred into the Pulse, not guessed into row logic |
| New-project detection | Gmail fast-lane (Orbit project-level notification mails) + weekly reconcile as the correctness guarantee |
| Sweep count | ONE daily sweep (merged loop, union set); ONE weekly maintenance job (registry rebuild, collects no signals) |
| Failure posture | Non-blocking additive layer — registry/sweep failure degrades to today's behavior, logged to Incidents |

## 4. Architecture

### 4.1 Component: Fixed-Cost Registry (new Notion sub-page)

Sub-page under the Notion parent (sibling of Run Log / Incidents / Preferences).

Contents:
- Header callout: active fixed-cost project **count** + `last_full_sweep` timestamp +
  `last_fast_lane_add` timestamp.
- Inline table, one row per project: `Project` (`<Title> (#<project_number>)`), `project_id`
  (internal, for tooling), `Client`, `Due date`, `Added by` (sweep | fast-lane | manual),
  `Added on`.
- PM may manually add/remove rows; the weekly sweep reconciles but **never removes a manual
  row without marking it** (`sweep: not found — verify` annotation instead of silent delete).

Read path: every Mode 1 fire, after Preferences, the collection sub-agent reads the registry
list. Registry unreadable → non-blocking: log Incident, proceed with workload-only universe
(today's behavior).

### 4.2 Component: Daily activity sweep (extension of collector, inside collection sub-agent)

There is exactly **one** daily loop. No second sweep.

- Loop set = `workload.by_project[] ∪ registry.project_ids` (deduped union).
- Registry projects (whether or not also in workload): `get_activity_log(project_id,
  startdate=lookback, timezone=Asia/Kolkata, per_page=500, page-on-has_more)` — **no `user_id`
  param** → all actors.
- Workload-only projects: unchanged existing call (`user_id=PM`).
- Overlap rule: one unfiltered call per project, never two calls. The unfiltered result is a
  strict superset of the PM-filtered result — zero signal loss, one call saved.
- Monday lookback override applies to registry projects identically (≥72h).
- Output: registry-project signals tagged `origin: fixed_cost_registry` join `orbit_signals[]`;
  per-task `latest_signal` computed the same way (flatten, newest-first).
- Runs inside the Lever-2 collection sub-agent; raw logs die there.

### 4.3 Component: Matcher integration (no new jobs)

- Registry signals enter Jobs 1–11 through the same gate as workload signals.
- Bar: **actionable-for-PM only** — blockers/questions aimed at the PM, AM- or client-authored
  comments, task completions that unblock a next step, due-date slips, new unassigned tasks.
  Routine dev chatter drops (`filter_reason: fixed_cost_routine_activity`).
- Verbs unchanged (5 locked verbs). A registry signal on a task NOT assigned to the PM that
  needs delegation resolves the same way any signal does; tasks assigned to other devs that are
  merely progressing produce **no row**.
- Rule 20 intact: quiet fixed-cost projects add zero rows.
- Deep-read (Job 7) applies only to surviving rows — unchanged mechanics, `get_task_details`
  once per row.

### 4.4 Component: Project Pulse (new writer block, zero extra calls)

Rendered on today's dated page **below the Morning Queue database**, as a toggle:
`Fixed-Cost Pulse — N projects, M with overnight movement`.

- One line per project **with movement**: `#15942 1828 Eng Landing — 2 tasks completed,
  3 comments, due Sep 10`. Composed from activity-log data already collected. No deep-reads.
- Projects without movement collapse to a single tail line: `14 projects — no overnight
  activity`.
- Purpose: the "PM wants updates on ongoing projects" half of the feedback, served passively
  while the promotion criteria remain unknown. Future iteration may promote recurring pulse
  patterns to rows once PMs articulate what matters.

### 4.5 Component: Gmail fast-lane (new-project detection, best-effort)

- Narrow carve-out to the existing "skip Orbit notification emails" rule in
  `collectors/gmail.md`: **project-level** notifications only (project created / you were added
  to project). Task-level Orbit notifications remain skipped (double-signal risk with
  activity_log).
- On detection: one `get_project_details(project_id)` → confirm `project_type_name == "Fixed
  Cost"` AND `status == "Active"` AND `development_owner == PM` → append registry row
  (`Added by: fast-lane`). Non-matching → ignore.
- Explicitly **best-effort**: PM notification settings may be off; templates may change. A miss
  costs nothing — the weekly reconcile catches it. No Incident on fast-lane silence.

### 4.6 Component: Weekly reconcile sweep (registry maintenance job)

- Cadence: weekly (own cron slot or piggybacked on an existing routine; NOT on the morning
  critical path). Collects zero signals; produces zero queue rows.
- Runs in its own throwaway sub-agent:
  1. `list_projects(type_name="Fixed Cost", status_name="Active")` — all pages (~4).
  2. `get_project_details` per project (~77 org-wide, parallel) → read `development_owner`.
  3. Filter to `development_owner == PM` → canonical 20–30 list.
  4. Reconcile registry: add missing, prune projects now Closed/On-Hold/archived, annotate
     (never silently delete) manual rows the sweep can't find.
  5. Update `last_full_sweep` + count in the registry header.
- Failure: non-blocking; registry keeps last-known-good list; Incident logged; repeat-failure
  escalation per existing chain.
- Swap-ready: if Orbit ever exposes a dev-lead filter on `list_projects`, steps 1–3 collapse to
  1–2 calls; registry page and everything downstream unchanged.

### 4.7 Data flow

```
WEEKLY (maintenance, no signals):
  reconcile sub-agent: list_projects ×4 → get_project_details ×~77
      → filter development_owner == PM → reconcile Fixed-Cost Registry (Notion)

DAILY Mode 1 (one sweep):
  collection sub-agent:
    get_user_workload(PM) ─────────────┐
    read Fixed-Cost Registry ──────────┤→ union project set
    get_activity_log × (union):        │    registry projects: no user_id (all actors)
                                       │    workload-only:    user_id=PM (unchanged)
    gmail collector (existing) + project-level Orbit-mail carve-out
        → get_project_details ×0–2 → registry append (fast-lane)
    → distilled signals (registry ones tagged origin: fixed_cost_registry)
  main context:
    matcher Jobs 1–11 (unchanged bar) → rows ──→ Morning Queue
    pulse composer (from already-collected activity) → Fixed-Cost Pulse toggle
```

## 5. Cost profile

| Path | Added calls | Context impact |
|---|---|---|
| Daily morning | +20–30 `get_activity_log` (date-bounded, mostly near-empty, parallel, minus overlap savings) + 0–2 fast-lane `get_project_details` | Raw dies in sub-agent; main context gains only surviving-signal text + pulse lines |
| Weekly | ~81 calls in throwaway sub-agent | Zero — returns reconciled list only |
| Deep-read | Only for surviving fixed-cost rows | Same per-row cost as today |

## 6. Error handling

| Failure | Behavior |
|---|---|
| Registry page unreadable | Proceed workload-only (today's behavior); Incident logged |
| Activity-log failure on a registry project | Per-project retry policy (existing 4-attempt backoff); project skipped after exhaustion; noted in run log |
| Fast-lane parse miss / notifications off | Silent (by design); weekly reconcile is the guarantee |
| Weekly sweep failure | Last-known-good registry stands; Incident; repeat-failure escalation |
| Registry empty (new PM, no fixed-cost projects) | Valid state — zero registry calls, zero pulse block or `0 projects` header line |

## 7. Files to touch (implementation plan input)

- `SKILL.md` — universe description (§ Data sources Orbit row), architecture overview Step 1,
  new rule for registry non-blocking posture; five-verb section untouched.
- `collectors/orbit.md` — union loop, unfiltered activity_log for registry projects, overlap
  rule, `origin` tag, registry read step.
- `collectors/gmail.md` — project-level Orbit-notification carve-out (task-level still skipped).
- `synthesis/matcher.md` — registry-signal gating notes (`fixed_cost_routine_activity` drop
  reason); no new jobs.
- `writers/notion.md` — Fixed-Cost Pulse toggle block on dated page.
- `schemas/parent-page.md` — Fixed-Cost Registry sub-page in the parent structure.
- NEW `schemas/fixed-cost-registry.md` — registry page schema.
- NEW `modes/weekly-registry-reconcile.md` — the maintenance job.
- `ROUTINE-ENTRYPOINTS.md` — 4th routine prompt (weekly reconcile) or piggyback note.
- `preflight.md` — registry existence check (create-if-missing on first run, like Preferences).
- `writers/run-log.md` — registry stats in run entry (projects swept, signals surfaced, pulse
  counts).

## 8. Verification

1. Dry-run Mode 1 with a populated registry: confirm ONE activity call per union project;
   registry projects show unfiltered actor coverage; workload-only unchanged.
2. Confirm a dev-comment-only night on a registry project yields zero rows + a pulse line.
3. Confirm an AM comment on a non-PM-assigned registry task yields a row through the normal
   verb tree with `latest_signal_anchor` intact.
4. Delete the registry page → run degrades to today's behavior + Incident (non-blocking proof).
5. Weekly sweep on UAT: registry converges to the same 20–30 set the Orbit UI shows with
   type+PM+active filters.
6. Queue diff for workload-only signals before/after the change — must be byte-identical
   (additive guarantee).

## 9. Open items

- Does `get_activity_log` without `user_id` include comment **bodies** or only field-change
  entries? (Probe during build; if bodies absent, pulse lines still work off counts; row
  gating may need a light `get_task_details` on candidate tasks.)
- Exact Orbit project-notification email subjects/templates for the fast-lane matcher (collect
  samples from a live PM inbox during build).
- Weekly reconcile cron slot: own routine vs piggyback on monthly archival pattern — decide at
  implementation.
