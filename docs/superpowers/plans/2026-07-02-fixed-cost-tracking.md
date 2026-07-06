# Fixed-Cost Project Tracking (v3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add project-centric tracking of Active + Fixed Cost projects (managed by the running PM) to the pm-task-assignment skill: filtered discovery with guard, one merged daily activity sweep, a Client-Ask Ledger with weekly follow-up reminders, specific stale-work pings, and a Project Pulse digest — with a hard zero-false-alarm rule.

**Architecture:** This is a **prompt-programmed skill** — implementation = writing/editing markdown behavior files that Claude executes at runtime. There is no compilable code and no test runner; each task's "test" is a structural verification (grep for required anchors, cross-file name consistency). Spec: `docs/superpowers/specs/2026-07-02-fixed-cost-project-tracking-design.md` (read the referenced §§ per task).

**Tech Stack:** Markdown behavior files; Orbit MCP (`list_projects`, `get_activity_log`, `get_project_details`, `get_task_details`); Notion MCP (registry page, dated pages).

## Global Constraints

Copied from the spec + SKILL.md non-negotiables. Every task inherits these:

- **DO NOT COMMIT.** No `git add`, no `git commit`, anywhere. Krishna reviews and commits manually. A task ends with edited files in the working tree, verified.
- **Five locked verbs only** (SKILL.md rule 18). All v3 rows are `Flag`. Never invent a verb.
- **Rule 20:** empty queue is valid. No lane may invent rows.
- **Rule 22:** user-facing project references use `project_number` as `#<project_number>`, never internal `id`/`url_slug`. Format: `<Project Name> (#<project_number>)`.
- **Rule 23:** AM = client relay. Follow-up copy says "follow up via <AM>", never "ping the client" as if the PM owns the client channel, never frames AM as teammate/dev-picker.
- **Rule 24:** every emitted row carries `latest_signal_anchor`; the writer's Triggered-by line is the gate.
- **Zero-false-alarm rule (spec §10.3):** resolution scan before reminder evaluation in the same run; re-verify anchor before emitting a reminder; fail-closed (cannot verify → suppress + run-log audit line).
- **Non-blocking posture:** any fixed-cost component failure degrades to today's workload-only behavior + Incident. Never abort the morning run for this layer.
- **Naming (exact, used across all files):** signal tag `origin: fixed_cost_registry`; drop reason `filter_reason: fixed_cost_routine_activity`; Preferences knobs `follow_up_reminder_days` (default 7) and `fixed_cost_stale_days` (default 2, business days); registry sub-page title `Fixed-Cost Registry`; machine state block fence tag `fc_state`; discovery modes `filtered` / `fallback-sweep`; state-tracker jobs `FC-1 Resolution scan`, `FC-2 Ask detection`, `FC-3 Follow-up reminders`, `FC-4 Stale-work pings`.
- Repo root for all paths: `/home/Krishna/Projects/PM Task Automation/pm-task-assignment/`.
- Match each file's existing voice: imperative second-person behavior instructions, `##`/`###` sections, tables for enums, explicit "does NOT do" lists.

---

### Task 1: Registry page schema (new file)

**Files:**
- Create: `schemas/fixed-cost-registry.md`

**Interfaces:**
- Produces: the page structure every other task references — header callout fields (`count`, `last_refresh`, `discovery_mode`), `Tracked Projects` table columns, `Client-Ask Ledger` table columns, `fc_state` JSON block shape. Later tasks MUST use these exact column names and JSON keys.

- [ ] **Step 1: Write the file** with this full content:

````markdown
# Schema — Fixed-Cost Registry Page

Sub-page of the Notion parent, sibling of Run Log / Incidents / Preferences. Created by
preflight if missing (like Preferences). Title: exactly `Fixed-Cost Registry`.

The registry is a **rendered snapshot + manual-override surface + ask ledger** — NOT the
source of truth for discovery. Source of truth is the daily filtered `list_projects` call
(collectors/orbit.md § Fixed-cost discovery). The skill refreshes this page every Mode 1 fire.

## Top-down layout

1. Header callout
2. H2 `Tracked Projects` + inline table
3. H2 `Client-Ask Ledger` + inline table
4. Toggle `Machine state — do not edit` containing one fenced `fc_state` JSON block

## 1. Header callout

Gray background, gear emoji. Three lines:

```
Fixed-cost tracking: <N> active projects
Last refresh: <YYYY-MM-DD HH:MM IST> · Discovery: <filtered | fallback-sweep>
Managed by PM Task Assignment — edit only the Manual pin rows and the Ask column.
```

`discovery_mode` values: `filtered` (guard passed) or `fallback-sweep` (guard tripped,
Appendix-A sweep or last-known-good in use).

## 2. Tracked Projects table

One row per tracked project. Columns (exact names):

| Column | Content |
|---|---|
| `Project` | `<Title> (#<project_number>)` — rule 22 format |
| `Project ID` | internal Orbit `id` (tooling only; never quoted in queue text) |
| `Client` | `client_name` (+ ` / <sub_client_name>` when present) |
| `Due date` | `project_due_date` date part, or `—` (infinite/unset) |
| `Added by` | `filter` \| `manual` |
| `Added on` | date first seen |

Rules:
- `filter` rows are fully machine-managed: added when discovery first returns the project,
  removed when discovery stops returning it (project closed/on-hold) — removal only under
  `discovery_mode: filtered`; under fallback, prune per the sweep result instead.
- `manual` rows are PM-owned pins: never auto-removed. If discovery can't see the project AND
  its `get_project_details.status` is not Active, append ` — verify: closed?` to the Project
  cell instead of deleting.
- Manual pins join the daily activity union exactly like filtered projects.

## 3. Client-Ask Ledger table

One row per client ask on a tracked project. Columns (exact names):

| Column | Content |
|---|---|
| `Project` | `<Title> (#<project_number>)` |
| `Ask` | the specific thing requested — "logo assets + brand fonts", never "waiting on client" |
| `Asked on` | date of the originating ask signal |
| `Source` | link to the anchoring Orbit comment / task, or Gmail thread subject + date |
| `Status` | `Open` \| `Resolved` \| `Resolved-manual` |
| `Resolved by` | link/reference to the resolving signal (auto-resolution), else empty |
| `Last reminder` | date the last follow-up Flag row was emitted, else `—` |

Rules:
- Rows are appended by FC-2 (ask detection) or by the PM manually (Project + Ask suffice;
  the skill backfills `Asked on` = today and `Source` = `manual` on next run).
- `Resolved` / `Resolved-manual` rows stay in the table (audit trail); FC-3 only evaluates
  `Status: Open` rows. Archive resolved rows older than 60 days during monthly archival.
- The ledger is the authoritative resolution state. The `fc_state` block never stores
  open/resolved — only throttle timestamps — so a corrupt state block can cause at worst an
  early reminder, never a false one.

## 4. Machine state block

Inside a toggle titled `Machine state — do not edit`, one fenced code block:

```fc_state
{
  "asks":  { "<ledger row: Project #number + Asked on>": { "last_reminder": "2026-07-02" } },
  "pings": { "<task_id>": { "last_ping": "2026-07-02", "stale_since": "2026-06-30" } }
}
```

- Ask keys: `#<project_number>|<Asked on date>|<first 40 chars of Ask>` — stable across runs.
- Unreadable/corrupt block → treat as empty `{"asks":{},"pings":{}}` and rebuild over
  subsequent runs. Never abort on state-block failure.
- `Last reminder` ledger column mirrors `asks.*.last_reminder` for PM visibility; the JSON is
  what FC-3/FC-4 read.

## Failure posture

Registry page unreadable → non-blocking: log Incident, proceed with the day's discovery
result directly (manual pins and ask ledger are unavailable for that run; FC-3/FC-4 skip —
reminders/pings simply don't fire that morning; they resume next run).

## What this page does NOT do

- Not the discovery source of truth (the filtered call is).
- Never holds queue rows, briefs, or handoff drafts.
- Never edited by Mode 2 (Mode 2 only reads note-interpreter resolutions, which Mode 2
  writes back to the ledger `Status` column — the single Mode 2 touch).
````

- [ ] **Step 2: Verify** — run and expect all greps to hit:

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -c "Client-Ask Ledger" schemas/fixed-cost-registry.md   # >= 2
grep -c "fc_state" schemas/fixed-cost-registry.md            # >= 2
grep -n "Resolved-manual\|fallback-sweep\|verify: closed?" schemas/fixed-cost-registry.md
```

- [ ] **Step 3: DO NOT COMMIT** — leave in working tree.

---

### Task 2: Orbit collector — discovery, filter guard, merged union loop

**Files:**
- Modify: `collectors/orbit.md`

**Interfaces:**
- Consumes: registry schema (Task 1) — `Tracked Projects` columns, `discovery_mode` values.
- Produces: `fixed_cost_projects[]` (discovery output: `{project_id, project_number, title, client_name, due_date, added_by}`), signals tagged `origin: fixed_cost_registry`, per-project `activity_summary` counts for the Pulse (`{project_number, title, due_date, tasks_completed, comments, status_changes, new_tasks, hours_logged_entries}`), and `discovery_mode` flag returned to the main context.

- [ ] **Step 1: Read** `collectors/orbit.md` §§ "Universe model", "MANDATORY tool call sequence", "Output shape — per signal", "Relationship map", "Tool calls", "Error handling" in full.

- [ ] **Step 2: Insert new section** after `## Universe model — user-scoped, NOT project-scoped` (extend, don't replace — the user-scoped model stays for everything non-fixed-cost). New section content:

````markdown
## Fixed-cost extension — ONE project-scoped lane inside the same sweep

The user-scoped universe above stays authoritative for everything except **Active + Fixed
Cost projects managed by this PM** (`development_owner == PM`). For those projects only, the
collector widens from "tasks assigned to the PM" to "the whole project" — activity by ANY
actor. Everything below runs inside this same collection sub-agent, in the same single pass.

### Fixed-cost discovery (runs before the activity loop)

1. Call `list_projects(type_name="Fixed Cost", status_name="Active",
   development_owner=<PM_user_id>)`; page until exhausted (expected 1–2 pages / 20–30
   projects).
2. **Filter guard — MANDATORY, never trust the response shape.** The param is silently
   ignored by backends that don't support it (returns the org-wide list, no error):
   a. *Plausibility:* `total` > 40 OR `total` ≥ 2× the registry's last-known count → suspect.
   b. *Spot check:* pick ONE returned project (rotate daily by date-mod-N), call
      `get_project_details`, assert `development_owner` == this PM. Mismatch → guard trips.
   c. Guard trips → discard the result. If registry `last_refresh` ≤ 7 days old → reuse the
      registry's Tracked Projects as today's set, `discovery_mode: fallback-sweep`. Older →
      run the Appendix-A sweep (spec): unfiltered `list_projects(type_name="Fixed Cost",
      status_name="Active")` all pages + `get_project_details` per project, keep
      `development_owner == PM`. Either way: write an Incident
      (`fixed-cost discovery filter not applied`).
   d. Guard passes → `discovery_mode: filtered`.
3. Union in the registry's `manual` pin rows (Task-1 schema). Output: `fixed_cost_projects[]`
   with `{project_id, project_number, title, client_name, due_date, added_by}` +
   `discovery_mode`.

### Merged activity loop — one loop, no second sweep

Loop set = `workload_summary.by_project[] ∪ fixed_cost_projects[]`, deduped by `project_id`:

- **Fixed-cost projects** (in the union, whether or not also in workload):
  `get_activity_log(project_id, startdate=<lookback>, timezone="Asia/Kolkata", per_page=500,
  page on has_more)` — **NO `user_id` param** → all actors.
- **Workload-only projects:** the existing call, unchanged (`user_id=PM`).
- Overlap rule: a project in both sets gets ONE unfiltered call, never two. The unfiltered
  result is a strict superset of the PM-filtered result — zero signal loss, one call saved.
- Monday lookback override applies to both identically.

### Fixed-cost signal handling

- Every signal originating from a fixed-cost project's unfiltered log carries
  `origin: fixed_cost_registry` in its output shape (add the field to § Output shape).
- `latest_signal` per task: same flatten-newest-first rule as everywhere else.
- Additionally emit per-project `activity_summary` counts for the Pulse writer:
  `{project_number, title, due_date, tasks_completed, comments, status_changes, new_tasks,
  hours_logged_entries}` — zeros allowed (Pulse renders "no movement" from zeros). Counts
  only; no bodies (the Pulse never deep-reads).

### Failure posture (this lane only)

Discovery call fails after retries → proceed workload-only (today's behavior), Incident.
A single project's activity call fails after retries → skip that project, note in run log,
continue. This lane never aborts the run.
````

- [ ] **Step 3: Update the collector's return payload** — in the section describing what the sub-agent returns to the main context (§ "Execution model" and/or § "Output shape"), add: `fixed_cost_projects[]`, `discovery_mode`, per-project `activity_summary[]`, and the `origin: fixed_cost_registry` field on signals. Also extend the § "Tool calls" table with the discovery calls (`list_projects` ×1–2, spot-check `get_project_details` ×1) and the § "MANDATORY tool call sequence" so the Step 1f assertion covers "discovery ran (or consciously fell back)".

- [ ] **Step 4: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "origin: fixed_cost_registry" collectors/orbit.md          # >= 2 (section + output shape)
grep -n "development_owner" collectors/orbit.md | head             # discovery + guard
grep -n "NO .user_id.\|no .user_id." collectors/orbit.md           # unfiltered rule present
grep -n "activity_summary" collectors/orbit.md                     # pulse feed
grep -n "fallback-sweep" collectors/orbit.md                       # guard fallback
```

- [ ] **Step 5: DO NOT COMMIT.**

---

### Task 3: Fixed-cost state tracker (new file — the v3 core)

**Files:**
- Create: `synthesis/fixed-cost-state.md`

**Interfaces:**
- Consumes: signals tagged `origin: fixed_cost_registry` (Task 2), Client-Ask Ledger + `fc_state` block (Task 1), Preferences knobs `follow_up_reminder_days` / `fixed_cost_stale_days` (Task 8), matcher cross-linked Gmail signals (existing Job 4b output).
- Produces: ledger mutations (new asks, resolutions), `fc_state` mutations (`last_reminder`, `last_ping`), and **Flag-verb row candidates** handed to matcher Job 5 with `latest_signal_anchor` set. Row candidate shape identical to any other pre-Job-5 item plus `fc_row_type: follow_up_reminder | stale_work_ping`.

- [ ] **Step 1: Write the file** with this full content:

````markdown
# Synthesis — Fixed-Cost State Tracker

Runs in the MAIN context (judgment, not plumbing), invoked by the matcher AFTER Job 4b
(grouping + Gmail cross-link done) and BEFORE Job 5 (verb classification). Only processes
signals tagged `origin: fixed_cost_registry` plus the Client-Ask Ledger. If the registry was
unreadable this run, skip all four jobs silently (reminders/pings resume next run).

Inputs: fixed-cost signals, cross-linked Gmail context signals, Client-Ask Ledger rows,
`fc_state` block, Preferences `follow_up_reminder_days` (default 7) and
`fixed_cost_stale_days` (default 2, business days).

Two lanes (from the PM interview that produced spec §10):
- **Waiting on client** — WLIQ asked the client for something; PM needs a weekly reminder
  naming the SPECIFIC ask.
- **In development** — assigned work in motion; PM needs a SPECIFIC status ping when it
  stalls — never a generic "any update?".

A project can be in both lanes at once (waiting on assets for phase 2 while phase 1 dev
continues). State is per-ask and per-task, never per-project.

Jobs run STRICTLY in this order. FC-1 before FC-3 is the zero-false-alarm ordering: a
same-morning delivery must resolve its ask before reminders are evaluated.

## FC-1 — Resolution scan (kills false alarms at the source)

For each ledger row with `Status: Open`, judge every incoming fixed-cost signal (and its
cross-linked Gmail context): **does this signal deliver, answer, or make moot the ask?**
Topic-match judgment — same pattern as the Create-parent-task dedup (matcher § "Dedup").
Delivery forms include: client reply or attachment on the ask's thread/task, a new comment
noting receipt ("client sent the fonts"), the blocked task resuming (status change off
hold), or the ask's task completing.

Match → set ledger `Status: Resolved`, fill `Resolved by` with the resolving signal
reference. The delivery signal itself still flows to Job 5 like any other signal (it will
usually earn its own row — that's gate 3; do NOT suppress it).

No match → leave open. Never resolve on vague activity ("any progress?" comments do not
resolve an ask).

## FC-2 — Ask detection (opens the ledger)

Scan today's fixed-cost signals for OUTBOUND client-directed requests: a PM- or AM-authored
Orbit comment or AM-relayed email asking the client to provide/approve/decide something.
The ask must be concrete enough to name — extract the specific object ("staging approval",
"product CSV", "logo assets + brand fonts"). Can't name it → not an ask; skip (do not
ledger vague nudges).

Before appending: topic-match against existing OPEN asks on the same project. Same thing
re-asked → update that row's `Asked on` to the newer date (clock restarts; no duplicate).
New ask → append ledger row: Project (rule-22 format), Ask, `Asked on` = signal date,
`Source` = signal anchor, `Status: Open`, `Last reminder: —`.

FC-2 runs before FC-3, and the ≥7-day threshold guarantees a new ask never reminds the day
it's created.

## FC-3 — Follow-up reminders (weekly, verified, fail-closed)

For each ledger row still `Status: Open` after FC-1, where
`today − max(Asked on, fc_state.asks[key].last_reminder) ≥ follow_up_reminder_days`:

1. **Re-verify (gate 2).** Deep-read the ask's anchor NOW: Orbit source →
   `get_task_details(task_id)` (bundled comments); Gmail source → re-read the thread.
   Confirm nothing in it answers the ask (the daily scan can miss deliveries — filtered
   inbox, attachment-only replies).
   - Verification says answered → resolve the row (as FC-1 would), emit NOTHING.
   - Anchor unreadable (404/permission/timeout after retries) → **fail closed**: emit
     NOTHING, write run-log audit line `reminder suppressed — unverified: <ask key>`.
2. Still open → emit a **Flag** row candidate (`fc_row_type: follow_up_reminder`):
   - Summary pattern: `Waiting on client <N> days: <ask> — follow up via <AM name>
     (#<project_number> <Project Title>)`. N = days since `Asked on`.
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

For each tracked fixed-cost project, walk its OPEN tasks assigned to a non-PM dev (from the
relationship map / workload universe). A task is **stale** when the unfiltered activity log
shows ZERO events for it (comments, status changes, time-track entries, attachments) for
≥ `fixed_cost_stale_days` BUSINESS days (Sat/Sun don't count — Friday-assigned work is not
stale on Monday).

Throttle: skip if `fc_state.pings[task_id].last_ping` is younger than
`fixed_cost_stale_days` business days. A task that stays stale re-pings at that cadence
with an escalating day count.

For each ping: run the LIGHT deep-read (same mechanics as the due-today Flag light read,
SKILL.md rule 18): one `get_task_details(task_id)` — newest bundled comments only, no
fallback call — to get last real movement + what the task was assigned for.

Emit a **Flag** row candidate (`fc_row_type: stale_work_ping`):
- Summary pattern: `No movement <N> days: "<task title>" — <assignee>, due <date>
  (#<project_number> <Project Title>)`.
- `latest_signal_anchor` = the task's `latest_signal` (its last real activity) — rule 24.
- Task Brief: what the task is for (from description/comments) + last known movement line +
  assignee + due date. **The generic-ping ban is structural:** a ping row missing any of
  {task title, assignee, last-movement line} MUST NOT be emitted — drop it and write a
  run-log audit line `ping dropped — insufficient specifics: task <id>` (the Notion writer
  enforces the same gate at render time; this is defense in depth).
- PM Next Step: `Check with <assignee> on "<task title>"`.

Update `fc_state.pings[task_id] = {last_ping: today, stale_since: <first stale date>}`.

## Output & downstream

Row candidates flow into matcher Job 5, which classifies both types as `Flag` (they carry
`fc_row_type`, no delegation target, no subtask). Existing machinery applies unchanged:
rule-24 anchor gate, row detail pages, PM notes. A PM note `delegate to <dev>` promotes a
ping row exactly like a due-today Flag (note-interpreter § Delegate). Notes `resolved` /
`client sent it` / `no longer needed` / `snooze <period>` on a reminder row resolve or
defer the ledger ask (note-interpreter § Fixed-cost resolution notes).

Rule 20 holds: no open asks + no stale work → this file emits nothing.

## What this tracker does NOT do

- Never emits Create subtask / Reopen / Hand off / Create parent task — Flags only.
- Never auto-sends anything to the client or AM (rule 5 — Flags have no executor).
- Never resolves an ask on vague activity; never reminds without re-verifying the anchor.
- Never pings tasks assigned to the PM (those are the PM's own; due-today logic covers them).
- Never runs on non-fixed-cost signals.
````

- [ ] **Step 2: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "FC-1\|FC-2\|FC-3\|FC-4" synthesis/fixed-cost-state.md | head
grep -n "fail closed\|fail-closed" synthesis/fixed-cost-state.md
grep -n "follow_up_reminder_days\|fixed_cost_stale_days" synthesis/fixed-cost-state.md
grep -n "fc_row_type" synthesis/fixed-cost-state.md
grep -n "BUSINESS days" synthesis/fixed-cost-state.md
```

- [ ] **Step 3: DO NOT COMMIT.**

---

### Task 4: Matcher hook + gating

**Files:**
- Modify: `synthesis/matcher.md`

**Interfaces:**
- Consumes: `origin: fixed_cost_registry` tag (Task 2); `synthesis/fixed-cost-state.md` (Task 3).
- Produces: the invocation point (after Job 4b, before Job 5) every mode file references; drop reason `fixed_cost_routine_activity`.

- [ ] **Step 1: Read** matcher §§ "Output gating — five locked verbs", "Drop list", "Job 4b", "Job 5" in full.

- [ ] **Step 2: Insert a new subsection** between Job 4b and Job 5 (heading level matching siblings, e.g. `### Job 4c — Fixed-cost state tracker (delegated)`):

````markdown
### Job 4c — Fixed-cost state tracker (delegated)

After Job 4b completes (grouping + Gmail cross-links done) and BEFORE Job 5, run
`synthesis/fixed-cost-state.md` (FC-1 resolution scan → FC-2 ask detection → FC-3 follow-up
reminders → FC-4 stale-work pings) over the signals tagged `origin: fixed_cost_registry`.
It mutates the Fixed-Cost Registry's Client-Ask Ledger + `fc_state` block and returns Flag
row candidates (`fc_row_type: follow_up_reminder | stale_work_ping`) that join the Job 5
input set. Registry unavailable this run → skip silently (non-blocking).

**Gating for fixed-cost signals in Job 5:** a fixed-cost signal earns a row ONLY if
actionable-for-PM — blockers/questions aimed at the PM, AM- or client-authored comments,
task completions that unblock a next step, due-date slips, new unassigned tasks. Routine
dev progress on dev-assigned tasks (WIP comments, self-status notes, time logging) drops
with `filter_reason: fixed_cost_routine_activity` — it is represented in the Fixed-Cost
Pulse, not the queue. Rule 20 applies: quiet fixed-cost projects add zero rows.
````

- [ ] **Step 3: Extend the Drop list** (§ "Drop list") with one entry: `Routine dev activity on fixed-cost registry projects (dev WIP comments / self-status / time logs on their own assigned tasks) → drop with filter_reason: fixed_cost_routine_activity — surfaces in the Fixed-Cost Pulse instead.` And in Job 5, note that `fc_row_type` candidates classify as `Flag` (no delegation target at emission; PM note can promote a ping like a due-today Flag).

- [ ] **Step 4: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "Job 4c" synthesis/matcher.md
grep -n "fixed_cost_routine_activity" synthesis/matcher.md    # >= 2 (Job 4c + drop list)
grep -n "fc_row_type" synthesis/matcher.md
```

- [ ] **Step 5: DO NOT COMMIT.**

---

### Task 5: Notion writer — Pulse block, registry writes, ping gate

**Files:**
- Modify: `writers/notion.md`

**Interfaces:**
- Consumes: `activity_summary[]` + `discovery_mode` + `fixed_cost_projects[]` (Task 2), ledger/`fc_state` mutations (Task 3), registry schema (Task 1).
- Produces: Pulse toggle on the dated page; registry refresh procedure; render-time required-field gate for ping rows.

- [ ] **Step 1: Read** notion.md §§ "Step 5 — Write the page body", "Step 5.5 — Enforce output gating", "Step 6.5 — post-write structure verification".

- [ ] **Step 2: Insert new step** after Step 5.6 (or adjacent to the Slack-handles block flow, matching numbering style) — `### Step 5.7 — Fixed-Cost Pulse toggle`:

````markdown
### Step 5.7 — Fixed-Cost Pulse toggle

Below the Morning Queue database (and above the Slack-handles reference block), render ONE
toggle block:

`Fixed-Cost Pulse — <N> projects, <M> with overnight movement`

Inside, one line per project WITH movement (from the collector's `activity_summary[]`;
counts only, no deep-reads), rule-22 project format:

`#15942 1828 Eng Landing — 2 tasks completed, 3 comments, due Sep 10`

Include only non-zero facts (skip "0 comments"). Then ONE collapsed tail line for the rest:
`<K> projects — no overnight activity`. Zero tracked projects → render the toggle with the
single line `No fixed-cost projects tracked` (or the header shows `0 projects`).
`discovery_mode: fallback-sweep` this run → append ` · discovery: fallback` to the toggle
title so the PM can see degraded discovery at a glance.

The Pulse is passive visibility (spec §4.5): it never contains action verbs, assignees, or
checkboxes — actionable items are queue rows, not pulse lines.
````

- [ ] **Step 3: Insert registry-refresh flow** as a new flow section (sibling of "Flow — writing today's page"): `## Flow — refreshing the Fixed-Cost Registry (end of Mode 1)`:

````markdown
## Flow — refreshing the Fixed-Cost Registry (end of Mode 1)

After the dated page is written (registry data already in hand from the collector +
Job 4c):

1. Update the header callout: count, `Last refresh`, `Discovery` mode.
2. Reconcile the Tracked Projects table per schemas/fixed-cost-registry.md rules
   (`filter` rows synced to today's discovery set; `manual` rows never auto-removed —
   annotate ` — verify: closed?` when appropriate).
3. Apply Job 4c ledger mutations: append FC-2 asks; set FC-1/FC-3 resolutions
   (`Status`, `Resolved by`); mirror `Last reminder` dates.
4. Rewrite the `fc_state` block (asks/pings timestamps) inside its toggle.
5. Any write fails after retries → Incident, continue (registry is a mirror; next run
   heals it).
````

- [ ] **Step 4: Extend Step 5.5 (output gating)** with the ping required-field gate: a row whose `fc_row_type` is `stale_work_ping` MUST contain task title + assignee + last-movement line in its Summary/Task Brief; missing any → do NOT render the row; write run-log audit line `ping dropped at render — insufficient specifics`. (Mirror of the tracker-side gate; and the rule-24 Triggered-by gate applies to both v3 row types as to any row.)

- [ ] **Step 5: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "Fixed-Cost Pulse" writers/notion.md            # >= 2 (step + flow refs)
grep -n "refreshing the Fixed-Cost Registry" writers/notion.md
grep -n "fc_state" writers/notion.md
grep -n "insufficient specifics" writers/notion.md
```

- [ ] **Step 6: DO NOT COMMIT.**

---

### Task 6: Note-interpreter — resolution/snooze vocabulary

**Files:**
- Modify: `synthesis/note-interpreter.md`

**Interfaces:**
- Consumes: ledger `Status` semantics (Task 1); `fc_row_type` (Task 3).
- Produces: Mode 2 behavior for PM notes on v3 Flag rows.

- [ ] **Step 1: Read** note-interpreter §§ "Common note patterns", the existing "remind me later / snooze" and "delegate to X" sections (to match format: quoted-pattern heading + resolution steps).

- [ ] **Step 2: Add one new pattern section** (same style as siblings), after the "remind me later / snooze" section:

````markdown
### "resolved" / "client sent it" / "got it" / "no longer needed" (fixed-cost reminder rows)

Applies to Flag rows with `fc_row_type: follow_up_reminder`. Mode 2 sets the matching
Client-Ask Ledger row (on the Fixed-Cost Registry page) to `Status: Resolved-manual`, notes
the PM note text in `Resolved by`, and marks the queue row `Done — ask closed by PM`. No
Orbit write. `no longer needed` resolves identically (the ask is moot — same end state).

### "snooze 2 weeks" / "remind me in <period>" (fixed-cost reminder rows)

Bump the ask's `fc_state` `last_reminder` forward so the next reminder fires after the
requested period (e.g. `snooze 2 weeks` → `last_reminder = today + 14d −
follow_up_reminder_days`). Ledger `Last reminder` mirrors it. Row → `Done — snoozed`.

On `fc_row_type: stale_work_ping` rows, the existing verbs apply unchanged: `delegate to
<dev>` promotes per § Delegate a due-today Flag; `skip` / `no` marks Skip. `resolved` on a
ping row just marks the row Skip (there is no ledger entry behind a ping).
````

- [ ] **Step 3: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "fc_row_type" synthesis/note-interpreter.md      # >= 2
grep -n "Resolved-manual" synthesis/note-interpreter.md
```

- [ ] **Step 4: DO NOT COMMIT.**

---

### Task 7: Parent-page schema + preflight — registry sub-page exists

**Files:**
- Modify: `schemas/parent-page.md`
- Modify: `preflight.md`

**Interfaces:**
- Consumes: registry page title `Fixed-Cost Registry` (Task 1).
- Produces: guaranteed page existence before any Mode 1 logic touches it.

- [ ] **Step 1:** In `schemas/parent-page.md` § "Static elements (always present)", add the registry as a new static sub-page entry between Incidents and Preferences (matching the existing numbered-entry format): title `Fixed-Cost Registry`, one-line purpose ("fixed-cost tracking snapshot + Client-Ask Ledger — see `schemas/fixed-cost-registry.md`"), and update the "Top-down layout" listing to include it in order (… Run Log → Incidents → **Fixed-Cost Registry** → Preferences last).

- [ ] **Step 2:** In `preflight.md` § "Step 6 — Confirm parent-page operational sub-pages exist", add `Fixed-Cost Registry` to the checked set with create-if-missing behavior (same pattern as the other operational pages; on creation, build the skeleton per `schemas/fixed-cost-registry.md`: header callout with `0 active projects` / `Discovery: filtered`, both empty tables, empty `fc_state` block). Missing-and-uncreatable → non-blocking for Mode 1 (fixed-cost lane skips, Incident) — do NOT abort the run for this page (unlike Preferences).

- [ ] **Step 3: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "Fixed-Cost Registry" schemas/parent-page.md preflight.md
grep -n "non-blocking" preflight.md
```

- [ ] **Step 4: DO NOT COMMIT.**

---

### Task 8: Preferences schema — the two knobs

**Files:**
- Modify: `schemas/preferences-page.md`

**Interfaces:**
- Produces: `follow_up_reminder_days` (default 7) and `fixed_cost_stale_days` (default 2, business days) — the names FC-3/FC-4 (Task 3) read.

- [ ] **Step 1: Read** preferences-page.md § "H2: Default Communication Preferences" (or the most fitting existing H2) to match the key/value documentation format.

- [ ] **Step 2: Add a new H2 section** (before "H2: Always-Include Rules", matching sibling format): `### H2: Fixed-Cost Tracking`:

````markdown
### H2: Fixed-Cost Tracking

Controls the fixed-cost project lane (collectors/orbit.md § Fixed-cost extension +
synthesis/fixed-cost-state.md).

| Key | Default | Meaning |
|---|---|---|
| `follow_up_reminder_days` | `7` | Days an open client ask waits before a follow-up Flag row (and between repeat reminders). |
| `fixed_cost_stale_days` | `2` | BUSINESS days (Sat/Sun excluded) of zero activity before an assigned task on a tracked project earns a stale-work ping; also the re-ping throttle. |

Absent section or keys → use defaults silently (first-run setup may omit it; the skill
never blocks on these).
````

- [ ] **Step 3: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "follow_up_reminder_days\|fixed_cost_stale_days" schemas/preferences-page.md
```

- [ ] **Step 4: DO NOT COMMIT.**

---

### Task 9: Run-log — fixed-cost stats + audit lines

**Files:**
- Modify: `writers/run-log.md`

**Interfaces:**
- Consumes: audit-line strings produced by Tasks 3/5 (`reminder suppressed — unverified`, `ping dropped — insufficient specifics`), `discovery_mode`, FC counters.
- Produces: run-entry fields the verification (§8/§10.6 of spec) inspects.

- [ ] **Step 1: Read** run-log.md §§ "Inputs" and "Step 3 — Build the detail-page body".

- [ ] **Step 2: Extend Inputs + detail body** with a fixed-cost stats line for Mode 1 entries (matching the existing terse one-line-per-decision style):

````markdown
Fixed-cost lane (Mode 1 only): `discovery: <filtered|fallback-sweep>, projects: <N>,
signals: <S>, asks opened: <A>, asks resolved: <R>, reminders: <E> emitted / <X> suppressed
(unverified), pings: <P> emitted / <T> throttled / <D> dropped (insufficient specifics)`.
Every suppression/drop also gets its own decision-trace line (subject → action → reason),
e.g. `ask "logo assets" (#16046) → reminder suppressed → anchor unreadable (fail-closed)`.
````

- [ ] **Step 3: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "asks opened\|reminders:" writers/run-log.md
grep -n "fail-closed" writers/run-log.md
```

- [ ] **Step 4: DO NOT COMMIT.**

---

### Task 10: SKILL.md — architecture, data sources, new rule, file map

**Files:**
- Modify: `SKILL.md`

**Interfaces:**
- Consumes: everything above — this task writes the top-level contract that names it all.

- [ ] **Step 1:** In the § "Architecture overview" Mode 1 block: extend Step 1e's description with the fixed-cost discovery + guard + merged union loop (one line each, referencing `collectors/orbit.md § Fixed-cost extension`), and add to Step 4 a line for Job 4c (`synthesis/fixed-cost-state.md — FC-1..FC-4`) plus the writer's Pulse + registry refresh (4c/4e area, matching lettering style).

- [ ] **Step 2:** In the § "Data sources" table, Orbit row: append 2–3 sentences — Active+Fixed-Cost projects with `development_owner == PM` are additionally tracked project-wide (all actors) via daily filtered discovery with a mandatory filter guard + registry manual pins; one merged activity loop (unfiltered superset call for these projects); tasks the PM follows on OTHER project types remain out of scope (the existing sentence stands, scoped).

- [ ] **Step 3:** Append new numbered rule after rule 25 (§ "Non-negotiable rules"):

````markdown
26. **Fixed-cost lane: non-blocking, Flag-only, zero false alarms.** The fixed-cost tracking
    layer (registry discovery + guard, Client-Ask Ledger, follow-up reminders, stale-work
    pings, Pulse — `collectors/orbit.md` § Fixed-cost extension,
    `synthesis/fixed-cost-state.md`, `schemas/fixed-cost-registry.md`) is strictly additive:
    any failure in it degrades the run to the workload-only universe (Incident logged),
    never aborts. Its rows are always `Flag` verb. Reminder emission is
    resolution-scan-first, re-verified against the ask's anchor at emission time, and
    fail-closed (cannot verify → suppress + run-log audit). Stale-work pings must be
    specific (task title + assignee + last movement) or they are dropped — a generic
    "any update?" row is forbidden. Discovery never trusts the `development_owner` filter
    without the guard (the param is silently ignored by servers that lack it).
````

- [ ] **Step 4:** Add to § "File map": `synthesis/fixed-cost-state.md` and `schemas/fixed-cost-registry.md` entries (one line each, matching format), and mention the registry in the `schemas/parent-page.md` line.

- [ ] **Step 5: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "^26\." SKILL.md
grep -n "fixed-cost-state.md\|fixed-cost-registry.md" SKILL.md   # file map + rule
grep -n "Fixed-cost" SKILL.md | head
```

- [ ] **Step 6: DO NOT COMMIT.**

---

### Task 11: Cross-file consistency verification (no edits expected)

**Files:**
- Read-only across all files touched above.

- [ ] **Step 1: Name consistency** — every grep must return ≥1 hit in EVERY listed file:

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
for t in "origin: fixed_cost_registry" "fc_row_type"; do echo "== $t"; grep -l "$t" collectors/orbit.md synthesis/matcher.md synthesis/fixed-cost-state.md; done
grep -l "fixed_cost_routine_activity" synthesis/matcher.md synthesis/fixed-cost-state.md
grep -l "follow_up_reminder_days" schemas/preferences-page.md synthesis/fixed-cost-state.md
grep -l "fixed_cost_stale_days" schemas/preferences-page.md synthesis/fixed-cost-state.md
grep -l "fc_state" schemas/fixed-cost-registry.md synthesis/fixed-cost-state.md writers/notion.md
grep -l "Fixed-Cost Registry" schemas/parent-page.md preflight.md writers/notion.md SKILL.md
grep -l "Fixed-Cost Pulse" writers/notion.md synthesis/matcher.md
```

(Note: `fc_row_type` in collectors/orbit.md is NOT expected — only matcher + state tracker + note-interpreter + notion writer carry it. Adjust the loop accordingly: check `fc_row_type` in `synthesis/matcher.md synthesis/fixed-cost-state.md synthesis/note-interpreter.md writers/notion.md`.)

- [ ] **Step 2: Ordering invariants** — confirm by reading: matcher Job 4c sits between Job 4b and Job 5; fixed-cost-state.md declares FC-1 before FC-3; notion.md registry refresh happens end-of-Mode-1 (after dated page).

- [ ] **Step 3: Spec-coverage sweep** — open the spec's §7 files-to-touch list and §10.6 verification items 8–13; for each, name the file+section that implements it. Any gap → fix in the owning task's file now.

- [ ] **Step 4: Report** — list every modified/created file with a one-line summary for Krishna's review. **DO NOT COMMIT — final state is an uncommitted working tree.**

---

## Self-review notes (done at plan time)

- **Spec coverage:** §4.1→T1/T5/T7, §4.2→T2, §4.3→T2, §4.4→T4, §4.5→T5, §10.2→T1/T3/T5, §10.3→T3 (gates 1,2) + T5/T9 (audit), §10.4→T3/T4, §10.5→T3/T8, §7 knobs→T8, run-log stats→T9, SKILL contract→T10, Appendix A fallback→T2 (guard branch c). Mode 2 ledger write-back→T6. Monthly-archival ledger pruning is mentioned in T1's schema (60-day archive note) and intentionally NOT given its own task — it's one line in an existing monthly flow; the archival mode file reads the registry schema. YAGNI.
- **Type/name consistency:** locked in Global Constraints; Task 11 enforces.
- **No placeholders:** every insert step carries full content or an exact, bounded instruction anchored to a named existing section.
