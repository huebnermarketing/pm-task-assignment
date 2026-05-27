> **This collector uses ONLY the Orbit MCP. Source allowlist — primary collection: Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand: Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever — including any that may seem relevant to a specific signal.**

> **Preflight (`preflight.md`) must have run before this collector is invoked. Do not call any tool until preflight has completed.**

# Orbit Collector

## Purpose

Pull overnight signals from Orbit that are relevant to **the PM's own todos** (every open task assigned to the PM). Return a structured list of signals ready for `synthesis/matcher.md`.

This collector runs in **two passes** during Mode 1:

- **Pass A — Priority pass** (Mode 1 sub-step 1d — local filter + cross-reference over Pass B's MCP responses; zero new MCP calls of its own). Narrow scope: parent tasks an AM created or reassigned to the running PM overnight, `due_date = today`. Output goes into a separate `priority_signals` list. Runs synchronously after Pass B's MCP responses land; does NOT block 2a (Gmail collector) from launching. See `## Priority Pass (Mode 1 sub-step 1d)` below.
- **Pass B — Normal pass** (Mode 1 sub-step 1e, parallel with the Gmail collector at 2a). Broad scope: every open task on the PM's plate (workload snapshot) + activity since `last_run_timestamp` (status changes, new comments, due-date moves, assignment changes). Output goes into the normal `orbit_signals` list. Behaves as documented in the sections below.

Mode 2 invokes neither pass — execution reads the morning queue from Notion, not from a fresh Orbit pull.

## Universe model — user-scoped, NOT project-scoped

Orbit has a direct user-scoped workload API: `get_user_workload(user_id, is_completed, per_page)` returns the **entire list of open tasks assigned to that user** in a single call, plus summary counts (overdue, due today, by project) and full per-task details. The PM's morning universe IS their workload — there is no need to enumerate "projects PM owns / follows / was active in for the last 6 months" and then iterate `get_project_task_list` per project. That iteration produced N API calls of mostly-irrelevant tasks (tasks PM follows but isn't assigned to). The single `get_user_workload(PM)` call replaces it.

**The universe of interest = `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)`.** Every primary collection signal flows from this set, with `get_activity_log` providing the change history for tasks in the set since `last_run_timestamp`.

Tasks the PM follows but is NOT assigned to are intentionally out of scope here — if a follower-only task needs PM input, that ask reliably surfaces through Gmail (the AM emails the PM) and is captured by `collectors/gmail.md`. The morning queue is about the PM's own todos plus the items the PM hasn't yet acknowledged; Gmail is the safety net for everything outside the PM's direct task list (per Mode 1 Step 2 § Role of Gmail and matcher Job 5 § Possible Orbit miss detection).

## Priority Pass (Mode 1 sub-step 1d — local filter, no MCP calls of its own)

This pass is a **local filter + cross-reference computation** over the MCP responses fetched by the normal pass (1e). It makes ZERO new MCP calls — `get_user_workload(PM)`, `get_activity_log`, and `list_users` are all shared with 1e (called once, reused here). The "sequential, BLOCKING" framing from earlier spec versions was a defensive measure to ensure `priority_signals[]` was populated before matcher Job 1 read it; that dependency is now encoded explicitly at the matcher consumer (Job 1 entry-gate), not at the Step 1 launch graph. As a result, 2a (Gmail collector) does NOT wait for 1d to complete — Gmail has no Orbit dependency.

Triggered when Mode 1 invokes the collector with `priority_pass = true`. It produces the `priority_signals` list that lands in the top-of-queue priority lane. The 1d filter runs synchronously as soon as 1e's MCP responses land — typically a few hundred milliseconds of in-memory filtering, no network.

### What it detects

Parent tasks where ALL of the following hold:

1. `assignee_id == PM_user_id` (the parent task is currently on the PM's plate)
2. `due_date == today` (in IST)
3. EITHER `created_by_id ∈ AM_user_ids` AND `created_at >= last_run_timestamp` (an AM created a new task and put it on PM directly during the lookback window)
4. OR an `assignee_change` event exists with `timestamp >= last_run_timestamp`, `new_assignee_id == PM_user_id`, AND `actor_id ∈ AM_user_ids` (an AM reassigned an existing parent task to the PM during the lookback window)

These are tasks the PM did not pick up themselves. The PM is the current assignee because an AM handed it to them, and the AM wants delegation work done today. The skill's job: surface the task at the top of the queue, suggest a delegate via the existing `synthesis/matcher.md` Job 6 4-branch tree (history → matrix availability → floater → cross-matrix Uncertain), and propose a sub-task **nested under this specific parent**.

### AM identity resolution

At the start of the priority pass:

1. Read the Account Managers section of the Preferences page (already loaded by preflight Step 4). Extract canonical email + aliases for each AM.
2. Call `list_users` once (the call is already cached for the run by `references/pod-matrix.md`, so this is a free lookup). For each AM, find the Orbit `user_id` whose email matches the AM canonical email or any alias.
3. Build `AM_user_ids = [ids of resolved AMs]`. Cache on the collector run state.

Edge cases:

- **Preferences has zero AMs configured.** Log `priority_pass_no_ams_configured` to Run Log. Return an empty `priority_signals` list. Mode 1 still proceeds — the normal Orbit pass + Gmail + Fathom run as usual.
- **An AM identity does not resolve to an Orbit user.** Log `priority_pass_am_unresolved: <am_name>` to Run Log and continue with the AMs that did resolve. Do not abort.
- **AM canonical email matches multiple Orbit users.** Log `priority_pass_am_ambiguous: <am_name>` and use the first match. The PM can disambiguate by editing the AM email in Preferences (a unique alias).

### Tool sequence

For the priority pass only (the normal-pass mandatory sequence still applies separately and reuses the same workload + activity-log responses):

1. **`list_users`** — already called once per run (cached by `references/pod-matrix.md`). Reuse the cache.
2. **`get_user_workload`** — already called in the normal pass with `user_id == PM_user_id, is_completed=incomplete, per_page=500`. The priority pass reuses that response. Filter the returned task list to tasks where `due_date == today (IST)`. This is the candidate set — typically a handful of tasks.
3. **`get_activity_log`** — already called in the normal pass with `from_date = last_run_timestamp` scoped to the PM's task universe. The priority pass reuses the same response. For each candidate task from step 2, cross-reference the activity log to confirm: (a) `created_by_id ∈ AM_user_ids` AND `created_at >= last_run_timestamp`, OR (b) an `assignee_change` entry exists for this task with `new_assignee_id == PM`, `actor_id ∈ AM`, and `timestamp >= last_run_timestamp`.
4. **`get_task_details`** — called per surviving candidate to enrich the priority signal with the full parent task body (so the matcher can render the proposed sub-task brief without an extra fetch later). Often `get_user_workload` already returns enough detail to skip this — only call when the body or description is missing or truncated.

Tool budget for the priority pass: 0 new MCP calls in most mornings (the `get_user_workload`, `get_activity_log`, and `list_users` calls are shared with the normal pass and cached). `get_task_details` fires only on the typically-tiny survivor set when extra body content is needed beyond what workload returned.

### Output shape — per priority signal

```
{
  "source": "orbit",
  "signal_type": "am_handed_to_pm_overnight_due_today",
  "event_kind": "created" | "reassigned",
  "project_id": <int>,
  "project_title": <string>,
  "project_url": <string>,
  "client_name": <string>,
  "sub_client_name": <string or null>,
  "account_manager": <string>,
  "project_owner": <string>,
  "parent_task_id": <int>,
  "parent_task_title": <string>,
  "parent_task_url": <string>,
  "parent_task_body": <string — full body, for matcher to derive sub-task brief>,
  "parent_task_due_date": <ISO date — should equal today>,
  "am_actor_id": <int>,
  "am_actor_name": <string>,
  "am_actor_email": <string>,
  "event_timestamp": <ISO datetime>,
  "bypass_pm_action_filter": true,
  "raw_source_data": <full source object>,
  "citation": {
    "type": "orbit",
    "label": "Orbit task #<parent_task_id> (handed to you by <am_actor_name>)",
    "url": <parent_task_url>
  }
}
```

Two fields are load-bearing for downstream matcher logic:

- `bypass_pm_action_filter: true` — instructs `synthesis/matcher.md` to skip the PM-action drop rule (SKILL.md non-negotiable #19) for this signal. An AM-handed task is pending delegation even if the PM commented or acknowledged it overnight.
- `parent_task_id` — instructs `synthesis/matcher.md` Job 5 to PIN the proposed sub-task under this exact parent rather than inferring a parent from PM-owned tasks. Executors then create the sub-task in Orbit under this parent.

### What the priority pass does NOT do

- Does not pick the delegated assignee. PM did not choose one (the parent task sits with the PM, not a delegate). Assignee suggestion runs in `synthesis/matcher.md` Job 6 via the existing 4-branch tree — same logic as any other Create-subtask row.
- Does not make a separate `get_user_workload` call — it reuses the one fired by the normal pass (step 2 of the mandatory sequence). The priority pass is a filter + cross-reference over that shared response, not an independent fetch.
- Does not block 2a (Gmail collector) from launching. 2a fans out in parallel with 1c/1d/1e. The matcher Job 1 entry-gate enforces the priority-signals-must-be-populated invariant at the consumer, not by serializing collector launches.
- Does not run in Mode 2. Mode 2 reads the morning queue from Notion, not Orbit.
- Does not double-emit signals into the normal pass. A task surfaced as a priority signal is excluded from the normal pass's `new_task` signal type (deduplicate by `task_id`).

---

## MANDATORY tool call sequence — no shortcuts (Normal pass)

This block overrides any inferred "fast path" the runtime might take. Mode 1's normal pass MUST invoke the following Orbit MCP tools in this exact order on every run. Skipping any of them is a Mode 1 failure and the assertion in `modes/mode-1-morning-collection.md` Step 1f will abort the run. The Priority Pass above runs FIRST (sequentially) and reuses the workload, activity-log, and `list_users` results emitted by this normal-pass sequence; the two passes share tool calls, they do not duplicate them.

1. **`get_user_details`** — PM identity confirmation (id, name, email).
2. **`get_user_workload` with `user_id == PM_user_id, is_completed=incomplete, per_page=500`** — the **non-skippable** universe-discovery call. Returns the full list of every open task assigned to the PM, with per-task details (project, due_date, status, severity, parent_id, description if present) and summary counts (overdue / due-today / by-project). This is the canonical PM-todo universe for the run; every downstream signal flows from it. Replaces the prior `list_projects` → `get_project_task_list × N` per-project iteration entirely.
3. **`get_activity_log` with `from_date = last_run_timestamp`** — the **non-skippable** change-history pull. Every comment, status flip, due-date move, new assignment, AM action landing inside the lookback window arrives ONLY via this call. Workload returns the static snapshot; activity_log returns the deltas since last run. Scope: pass the task IDs from step 2 (workload) if the MCP supports task-id filtering; otherwise call user-scoped (events involving PM as actor or target) and cross-reference against the workload task IDs locally. Do NOT iterate per project — that was the old model.
4. **`list_task_comments`** — has TWO call patterns: (a) collection-phase fallback for any task flagged by activity log that references a comment by id without the body text; (b) **synthesis-phase per-row deep-read, mandatory** — see § Per-row deep-read below.
5. **`get_task_details`** — has TWO call patterns: (a) collection-phase fallback for any workload task whose description / body is missing or truncated in the workload response; (b) **synthesis-phase per-row deep-read, mandatory** — see § Per-row deep-read below.
6. **`get_asset_attachment_summary_with_download_url`** — for any new attachments flagged by activity log on PM-workload tasks.
7. **`list_clients` / `list_sub_clients` / `list_users`** — relationship-map enrichment for client / sub-client / actor identification on the workload's per-task project info.

**`list_projects` and `get_project_task_list` are no longer in the mandatory sequence.** The PM's task universe comes from `get_user_workload(PM)`, not from project enumeration. These tools remain available for narrow secondary use cases (e.g., the PM's owned-project list, if needed for non-collection purposes), but are NOT called during the normal collection pass.

`get_user_workload` now has TWO uses in this skill, with different `user_id` targets:

- **PM-workload (this collector, mandatory)** — `get_user_workload(PM_user_id, ...)` is the morning-collection universe-discovery call. Always called in Mode 1 Step 1d/1e.
- **Candidate-availability (pod-inference, lazy)** — `get_user_workload(candidate_user_id, ...)` is called by `synthesis/pod-inference.md` ONLY on the matcher Job 6 no-history fallback path, to score availability of a small candidate subset. Per SKILL.md non-negotiable rule #6.

The Mode 1 Step 1f assertion checks BOTH that `get_user_workload(PM)` was called (step 2 above) AND that `get_activity_log` was called (step 3) — a tool trace missing either is a hard abort. A trace that shows only `get_user_workload(PM)` and no `get_activity_log` is also a SPEC VIOLATION — change detection requires both calls.

If `get_user_workload(PM)` returns an MCP error, apply the retry policy from `connector-failure-notify.md` (4 attempts, 2s/5s/15s backoff). After 4 failures, log `orbit_user_workload_unavailable` in the Run Log and abort the Orbit collector — no universe means no Orbit signals this run. If `get_activity_log` returns an MCP error, apply the same retry policy. After 4 failures, log `orbit_activity_log_unavailable` and continue with workload-only data — the PM will see "no change detection this morning, only static workload" in the page summary, and the Mode 1 Step 1f assertion will surface the gap.

## Scope — the PM's open task list

The "universe" for the morning run is the set of open tasks returned by `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)`. Each task in the response carries its project, due_date, status, severity, assignee (= PM), parent_id, and (typically) description.

What is NOT in scope:

- **Tasks where the PM is a follower but not the assignee.** These intentionally drop out of the Orbit universe. If a follower-only task needs PM input, the ask reliably surfaces through Gmail — captured by `collectors/gmail.md` and routed via matcher Job 5 (possibly as Possible-Orbit-miss if no Orbit corroboration).
- **Unassigned project orphans.** Previously detected via `get_project_task_list(assignee_id: 0)` per project. Out of scope under the user-centric model — if an unassigned task genuinely needs the PM's attention, an AM will email about it.
- **Cold tasks the PM is no longer involved with.** Workload naturally excludes them (only `is_completed=incomplete` tasks return).

## What to pull from the workload

For each task in the workload response:

1. **Static snapshot** — task_id, title, project info, due_date, status, severity, parent_id, description. This is the row's foundational data and the matcher's relationship map seed.
2. **Activity since `last_run_timestamp`** — from the `get_activity_log` call. Every field change, status transition, new comment, new assignment, due-date move on this task ID lands as an `activity_log_entry` signal.
3. **New comments** — from activity log entries with `type: comment` on this task ID. Especially ones mentioning the PM (@ in comment content).
4. **Status changes that matter** — any task where status moved into `Waiting for Feedback`, `Client Review`, `Blocked`, or `Done` since last run.
5. **Overdue flag** — tasks where `due_date < today (IST)` AND status is not `Done` get an `overdue_task` signal type.
6. **Today's-due flag** — tasks where `due_date == today (IST)` AND status is not `Done` AND not already in `priority_signals[]` (the priority pass takes precedence). Used by sort rule heuristics, not as a standalone signal.
7. **New attachments** — flagged by activity log on the task; pull filename + download URL via `get_asset_attachment_summary_with_download_url` if PM may need to review.

## What to skip

- Field changes the PM made themselves (self-noise — actor_id == PM_user_id on field changes).
- Automated bot comments (e.g., "system updated due date").
- Tasks already surfaced in `priority_signals[]` (deduplicate by task_id between priority and normal passes).
- Tasks where status is `Archive` (already complete from PM's perspective).

## Per-row deep-read (mandatory, fired by matcher Job 7)

The collection-phase calls above return the universe + the deltas since `last_run_timestamp`, which is enough for the matcher to decide which tasks become rows. But composing the proposed 6-section body for a row requires **deeper context than the workload snapshot or the activity-log delta provides** — specifically, the full task description and the complete comment history (including comments older than `last_run_timestamp`).

For every task that becomes a `Create subtask` or `Create parent task` row (typically 10-30 rows per morning), the matcher's Job 7 lazily fires:

1. **`get_task_details(task_id)`** — full task body, description, all fields. Default-on, NOT gated on whether the workload returned a description. The workload's truncation behavior is opaque; trust nothing about completeness.
2. **`list_task_comments(task_id)`** — full all-time comment history, NOT date-filtered to `last_run_timestamp`. The collection-phase `get_activity_log` returns deltas since last_run only; older decisions, prior client feedback, AM clarifications, failed attempts, scope changes live in comments from earlier weeks/months that the activity log won't surface.

**Why both calls fire during synthesis, not collection.** Firing at collection time would mean calling `get_task_details` + `list_task_comments` for every task in the workload (typically 30-80), most of which won't become rows after Job 5 filtering — wasted API calls. Firing during Job 7 bounds the calls to the post-filter survivor set (typically 10-30 rows). This mirrors the lazy patterns already established for `fetch_enrichment()` (Fathom, Job 4b Pass 2) and `get_user_workload(candidate_user_id)` (pod-inference, Job 6 no-history fallback).

**Issuance is a parallel batch, not serial per-row.** Matcher Job 7 issues `get_task_details(task_id)` + `list_task_comments(task_id)` for EVERY row in the post-Job-5 survivor set as parallel tool calls in one LLM turn — Claude Code supports multi-tool-use per turn. Batch cap: 25 parallel tool calls per turn; the matcher chunks larger survivor sets into multiple batches (rows 1–12, 13–24, etc.). Wall-clock estimate: serial would be ~8s for S=20 rows at ~200ms MCP latency × 40 calls; parallel one-batch issuance lands in ~250ms + overhead. Per-row failure semantics (one row's deep-read failure doesn't block other rows) are defined in `synthesis/matcher.md` Job 7 § Mandatory deep-read of the originating Orbit task.

**Mandatory, not conditional.** Same default-on rule as the Gmail-thread enrichment (see `synthesis/matcher.md` Job 7 § Mandatory email-thread enrichment): do NOT skip these calls because the workload snapshot or the Orbit task title looks "complete enough". The runtime cannot judge completeness without reading; reading first is the only way to know.

**Flag rows skip these calls.** Flag rows do not produce an Orbit body (per matcher Job 7 § Flag path), so the per-row deep-read does not fire. The PM resolves Flag rows manually — they have the Orbit task URL in the row's `orbit_task_link` column and can click through to see comment history themselves.

## Output shape — per signal

Each signal is a structured record:

```
{
  "source": "orbit",
  "signal_type": "activity_log_entry" | "overdue_task" | "new_task" | "status_change" | "new_comment" | "new_attachment",
  "project_id": <int>,
  "project_title": <string>,
  "project_url": <string>,
  "client_name": <string>,
  "sub_client_name": <string or null>,
  "account_manager": <string>,
  "project_owner": <string>,
  "task_id": <int or null>,
  "task_title": <string or null>,
  "task_url": <string or null>,
  "actor_name": <string>,
  "actor_id": <int>,
  "timestamp": <ISO datetime>,
  "content": <string — the exact change, comment text, or description>,
  "raw_source_data": <full source object for downstream reference>,
  "citation": {
    "type": "orbit",
    "label": "Orbit task #<task_id>" | "Orbit project #<project_id>",
    "url": <task_url or project_url>
  }
}
```

## Relationship map

In addition to the signals, emit an Orbit relationship map that the matcher uses to connect cross-source signals to the right project.

Structure:

```
{
  "pm": {
    "id": <int>,
    "name": <string>,
    "email": <string>
  },
  "projects": [
    {
      "id": <int>,
      "title": <string>,
      "url": <string>,
      "project_type": <string — "Fixed Cost" | "SaaS" | "PPC" | "Hosting" | "Hourly" | "Repeat" | "Ad-hoc" | "Maintenance" | ...>,
      "client_name": <string>,
      "sub_client_name": <string or null>,
      "project_owner_name": <string>,
      "project_owner_id": <int>,
      "account_manager_name": <string>,
      "account_manager_id": <int>,
      "followers": [{"id": <int>, "name": <string>}, ...],
      "recent_task_assignees": [{"id": <int>, "name": <string>, "task_count": <int>}, ...],
      "department_usage": [{"department_name": <string>, "hours_used": <string>}, ...],
      "client_contacts": [{"name": <string>, "email": <string>}, ...],
      "tasks": [
        {
          "id": <int>,
          "title": <string>,
          "url": <string — Orbit task URL>,
          "assignee_id": <int or null>,
          "assignee_name": <string or null>,
          "parent_task_id": <int or null — null for top-level parents, set for sub-tasks>,
          "status": <string>,
          "due_date": <ISO date or null>,
          "is_pm_owned": <bool — assignee_id == PM_user_id at collection time>
        }
      ]
    }
  ]
}
```

The relationship map is now **derived from `get_user_workload(PM)` response**, not from per-project iteration. Each task in the workload carries its project info (project_id, title, client, sub-client, AM, owner, followers, project_type). The collector deduplicates the workload's per-task project info into project-level entries for this map.

The `tasks` array is populated from the workload response directly — every open task assigned to the PM appears here with `is_pm_owned = true` (by definition, since workload only returns PM-assigned tasks). For projects where workload returned multiple tasks, all of them appear in the array; the project entry consolidates them.

It serves three downstream needs:

1. `synthesis/matcher.md` Job 5 — finding the PM-owned parent task to nest a sub-task under (filter `is_pm_owned == true`, which is true for every task in this map by definition).
2. `synthesis/matcher.md` — composing the row's `orbit_task_link` column by looking up the parent task's `url` by `parent_task_id` when the signal itself doesn't carry the parent URL.
3. `synthesis/pod-inference.md` — computing `has_history_on_project` for candidate assignees. Note: under the user-scoped universe, the relationship map's project list is bounded by projects where the PM has at least one assigned open task. Projects where the PM is a follower-only contribute to history scoring only through Gmail-routed signals (no Orbit follower-only signal reaches this map).

This map is used by `synthesis/matcher.md` to take a signal from Gmail (e.g., "email from jane@agencyx.com") and figure out it's about project 8426 because Agency X is the client for that project and Jane is a client contact. The candidate project must still appear in the map (i.e., PM has at least one open task there) for the routing to work — Gmail-only signals about projects with NO PM-assigned task become Possible-Orbit-miss candidates per matcher Job 5 § Possible Orbit miss detection.

It's also used by `synthesis/pod-inference.md` to compute candidate assignees per project that appears in the map.

## Tool calls

Use the following Orbit MCP tools (the `mcp__...orbit.` prefix matches whichever Orbit MCP namespace the user has installed):

- `mcp__...orbit.get_user_details` — for PM identity confirmation (id, name, email)
- **`mcp__...orbit.get_user_workload`** — **the primary universe-discovery call.** Two distinct uses, distinguished by `user_id`:
  - **PM-workload (THIS collector, mandatory):** `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)`. Called once per Mode 1 run in step 2 of the mandatory sequence. Returns every open task assigned to the PM with full details + summary counts.
  - **Candidate-availability (`synthesis/pod-inference.md`, lazy):** `get_user_workload(candidate_user_id, ...)`. Invoked ONLY on the matcher Job 6 no-history fallback path to score availability of a small candidate subset. Per SKILL.md non-negotiable rule #6.
- `mcp__...orbit.get_activity_log` — for changes since last run. Pass workload task IDs if MCP supports task-id filtering; otherwise call user-scoped and cross-reference locally.
- `mcp__...orbit.get_task_details` — for full body context on any workload task whose description is missing or truncated. Skip when workload already returned sufficient detail.
- `mcp__...orbit.list_task_comments` — fallback for activity-log entries that reference a comment by id without the body text.
- `mcp__...orbit.get_asset_attachment_summary_with_download_url` — for new attachments flagged on workload tasks (note: unreliable for non-txt — default to download-and-read per `writers/source-citation.md`).
- `mcp__...orbit.list_clients`, `mcp__...orbit.list_sub_clients` — for client/sub-client enrichment in the relationship map.
- `mcp__...orbit.list_users` — for matrix-name → user_id resolution + AM identity resolution in the priority pass. Called once per Mode 1 run by `references/pod-matrix.md`; the user list is cached for the duration of the run.

**Removed from the mandatory sequence (no longer called during normal collection):**
- `mcp__...orbit.list_projects` — was used to enumerate the PM's project universe; replaced by user-centric workload discovery. Remains available if a downstream component genuinely needs the PM's owned-project list, but the collector itself does not call it.
- `mcp__...orbit.get_project_task_list` — was used to iterate per-project task lists; replaced entirely by `get_user_workload(PM)`. Not called during normal collection.
- `mcp__...orbit.get_project_details` — was used for per-project metadata; project info now comes inline with each task in the workload response.

## Performance

Single primary API call (`get_user_workload(PM)`) returns the entire universe — typically 30–80 open tasks for a working PM, capped at `per_page=500`. Add one `get_activity_log` call for change detection, plus a small number of `get_task_details` / `list_task_comments` / `get_asset_attachment_summary_with_download_url` calls for the subset of tasks with new activity. Compared to the prior project-iteration model (which fired N task-list calls for N=30-50 projects in the universe), the new model is roughly an order of magnitude faster for the collection step.

Wall-clock target for Orbit collection alone: under 30 seconds on a typical morning. The full Mode 1 wall-clock target (10–15 minutes) is dominated by Notion writes and synthesis, not Orbit calls.

## Error handling

| Failure | Behavior |
|---|---|
| Orbit MCP unavailable | Return an empty signals list with an error note. Mode 1 will log the failure on the page summary and continue. |
| Individual project fetch fails | Skip that project. Do not abort the collector. |
| Attachment summary fails | Fall back to raw filename reference. Matcher still knows the file exists, just without a text summary. |

## What this collector does NOT do

- Does not write to Orbit. Read-only.
- Does not synthesize or group signals. That's the matcher's job.
- Does not dedup against other sources. Each source collector is independent.
- Does not filter by urgency or importance. Every relevant signal is returned.
- Does not iterate per project. The universe-discovery model is user-centric (`get_user_workload(PM)`) — `list_projects` and `get_project_task_list` are NOT called during normal collection. Per-project iteration was the old model and produced N API calls of mostly-irrelevant tasks; the single workload call replaces it.
- Does not detect follower-only tasks (tasks the PM follows but is not assigned to). If a follower-only task needs PM input, the ask reliably surfaces through Gmail and is captured by `collectors/gmail.md`; matcher Job 5 routes it via the standard or Possible-Orbit-miss paths.
- Does not detect unassigned project orphans. Same rationale — if an unassigned task genuinely needs the PM, an AM will email about it.
- Does not collect candidate-workload proactively. `get_user_workload(candidate_user_id)` (different `user_id` than the PM-workload call) is invoked lazily by `synthesis/pod-inference.md` on the matcher Job 6 no-history fallback path only, per `SKILL.md` non-negotiable rule #6. The PM-workload call IS made proactively every Mode 1 run — that is its intended use.
