# Fixed-Cost Project Tracking — Design

**Date:** 2026-07-02 (v2 — same day, discovery model revised)
**Status:** Approved by Krishna (brainstorming session)
**Target surface:** Claude skill (`pm-task-assignment`) only. n8n port inherits later.
**Source of ask:** PM team feedback — "Track fixed cost projects and add them in todos based on activities and updates; Active projects + Fixed cost project type."

> **v2 change:** v1 assumed no PM filter on `list_projects` and designed a weekly 77-call
> discovery sweep. Per Krishna's direction, v2 **assumes Orbit will support
> `list_projects(development_owner=<PM_user_id>)`** and makes filtered discovery the primary
> path — with a mandatory runtime guard (the param is *silently ignored* today, so the skill
> must detect non-application, never trust it) and the v1 sweep retained as the automatic
> fallback (Appendix A).

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

`list_projects(type_name="Fixed Cost", status_name="Active", development_owner=<PM_user_id>)`
will return the PM's 20–30 projects in 1–2 pages.

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
  `discovery_mode: filtered | fallback-sweep`.
- Inline table, one row per project: `Project` (`<Title> (#<project_number>)`), `project_id`
  (internal, for tooling), `Client`, `Due date`, `Added by` (filter | manual), `Added on`.
- **Manual rows are honored**: a PM may pin a project the filter doesn't return (e.g. they
  co-manage it). Manual rows join the daily union set and are never auto-removed — if discovery
  stops returning them AND they're Closed, they get annotated `verify — closed?`, not deleted.
- Refreshed by Mode 1 on every fire (cheap — the discovery call already ran).

Read/write path failure → non-blocking: log Incident, proceed with discovery result directly
(registry is a mirror, so losing it loses only manual pins for that run).

### 4.2 Component: Daily discovery + filter guard (inside collection sub-agent)

Every Mode 1 fire, before the activity loop:

1. `list_projects(type_name="Fixed Cost", status_name="Active",
   development_owner=<PM_user_id>)` — page until `has_more` false (expected 1–2 pages).
2. **Filter guard** (mandatory, cheap):
   a. **Plausibility check:** result `total` ≤ 40 expected. `total` ≈ org-wide count
      (≥ ~2× the registry's last-known count AND > 40) → treat as filter-ignored.
   b. **Spot check:** `get_project_details` on ONE result project (rotate daily) → assert
      `development_owner == PM`. Mismatch → filter-ignored.
   c. Filter-ignored → discard result, run Appendix A sweep (if last sweep > 7 days old;
      else reuse registry last-known-good), write Incident
      (`fixed-cost discovery filter not applied`), set registry `discovery_mode:
      fallback-sweep`.
3. Guard passed → project set = filtered result ∪ registry manual pins. Refresh registry
   snapshot.

New projects appear same-day (the filter call is fresh every morning); Closed projects drop
same-day. No weekly maintenance job, no Gmail fast-lane — both were latency/correctness
workarounds for the missing filter and are **deleted from this design** (they survive only
inside Appendix A as fallback machinery).

### 4.3 Component: Daily activity sweep (extension of collector, one merged loop)

There is exactly **one** daily loop. No second sweep.

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
- Runs inside the Lever-2 collection sub-agent; raw logs die there.

### 4.4 Component: Matcher integration (no new jobs)

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

- One line per project **with movement**: `#15942 1828 Eng Landing — 2 tasks completed,
  3 comments, due Sep 10`. Composed from activity-log data already collected. No deep-reads.
- Projects without movement collapse to a single tail line: `14 projects — no overnight
  activity`.
- Purpose: the "PM wants updates on ongoing projects" half of the feedback, served passively
  while promotion criteria remain unknown. Future iteration may promote recurring pulse
  patterns to rows once PMs articulate what matters.

### 4.6 Data flow

```
DAILY Mode 1 (one sweep):
  collection sub-agent:
    list_projects(Fixed Cost, Active, development_owner=PM) ×1–2 pages
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
| Daily activity | +20–30 `get_activity_log` (date-bounded, mostly near-empty, parallel, minus overlap savings) | Raw dies in sub-agent; main context gains only surviving-signal text + pulse lines |
| Deep-read | Only for surviving fixed-cost rows | Same per-row cost as today |
| Fallback (guard failure only) | ~81-call sweep, ≤ weekly | Throwaway sub-agent |

## 6. Error handling

| Failure | Behavior |
|---|---|
| Discovery filter silently ignored (guard trips) | Fallback sweep or last-known-good registry; Incident logged; `discovery_mode: fallback-sweep` |
| Discovery call fails outright | Registry last-known-good list stands; Incident; existing retry policy first |
| Registry page unreadable | Proceed with discovery result (lose manual pins for the run); Incident |
| Activity-log failure on a project | Existing 4-attempt backoff; project skipped after exhaustion; noted in run log |
| Registry empty + discovery empty (new PM) | Valid state — zero fixed-cost calls, `0 projects` pulse header |

## 7. Files to touch (implementation plan input)

- `SKILL.md` — universe description (§ Data sources Orbit row), architecture overview Step 1,
  new rule: fixed-cost layer non-blocking + filter guard mandatory; five-verb section untouched.
- `collectors/orbit.md` — discovery call + filter guard, union loop, unfiltered activity_log
  for fixed-cost projects, overlap rule, `origin` tag, registry snapshot refresh.
- `synthesis/matcher.md` — fixed-cost signal gating notes (`fixed_cost_routine_activity` drop
  reason); no new jobs.
- `writers/notion.md` — Fixed-Cost Pulse toggle block; registry snapshot write.
- `schemas/parent-page.md` — Fixed-Cost Registry sub-page in the parent structure.
- NEW `schemas/fixed-cost-registry.md` — registry page schema (snapshot + manual pins +
  discovery_mode header).
- `preflight.md` — registry existence check (create-if-missing on first run, like Preferences).
- `writers/run-log.md` — fixed-cost stats in run entry (projects discovered, guard result,
  signals surfaced, pulse counts).
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

- **Filter param final name/shape** — `development_owner`? `development_owner_id`?
  `dev_lead_id`? Confirm with Orbit team before implementation; guard logic is name-agnostic
  but the call site needs the real param.
- Does `get_activity_log` without `user_id` include comment **bodies** or only field-change
  entries? (Probe during build; if bodies absent, pulse lines still work off counts; row
  gating may need a light `get_task_details` on candidate tasks.)
- Guard plausibility threshold (40) — sanity-check against production PM slices before lock.

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
