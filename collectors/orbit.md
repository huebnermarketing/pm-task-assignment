> **This collector uses ONLY the Orbit MCP. Source allowlist — primary collection: Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand: Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever — including any that may seem relevant to a specific signal.**

> **Preflight (`preflight.md`) must have run before this collector is invoked. Do not call any tool until preflight has completed.**

# Orbit Collector

## Purpose

Pull overnight signals from Orbit that are relevant to the PM's projects. Return a structured list of signals ready for `synthesis/matcher.md`.

This collector runs in **two passes** during Mode 1:

- **Pass A — Priority pass** (Mode 1 Step 3a, sequential, blocking). Narrow scope: parent tasks an AM created or reassigned to the running PM overnight, `due_date = today`. Output goes into a separate `priority_signals` list. See `## Priority Pass (Mode 1 Step 3a)` below.
- **Pass B — Normal pass** (Mode 1 Step 3b, parallel with Gmail + Fathom). Broad scope: activity log + overdue + new tasks + status changes + new comments + attachments since `last_run_timestamp`. Output goes into the normal `orbit_signals` list. Behaves as documented in the original sections of this file.

Mode 2 invokes neither pass — execution reads the morning queue from Notion, not from a fresh Orbit pull.

## Priority Pass (Mode 1 Step 3a)

This pass runs FIRST in Mode 1, before any other collector. It produces the `priority_signals` list that lands in the top-of-queue priority lane. Triggered when Mode 1 invokes the collector with `priority_pass = true`.

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

For the priority pass only (the normal-pass mandatory sequence still applies separately):

1. **`list_users`** — already called once per run (cached by `references/pod-matrix.md`). Reuse the cache.
2. **`get_project_task_list`** with `due_date_filter: "today"` AND `assignee_id == PM_user_id` — run once per project in the PM's universe (owner + follower + active-in-last-6-months, same scope as the normal pass). Returns the small set of today's-due tasks currently on the PM's plate.
3. **`get_activity_log`** — already called in the normal pass with `from_date = last_run_timestamp`. The priority pass reuses the same response. For each candidate task from step 2, cross-reference the activity log to confirm: (a) `created_by_id ∈ AM_user_ids` AND `created_at >= last_run_timestamp`, OR (b) an `assignee_change` entry exists for this task with `new_assignee_id == PM`, `actor_id ∈ AM`, and `timestamp >= last_run_timestamp`.
4. **`get_task_details`** — called per surviving candidate to enrich the priority signal with the full parent task body (so the matcher can render the proposed sub-task brief without an extra fetch later).

Tool budget for the priority pass: ≤ 2 extra MCP calls per project (`get_project_task_list` + `get_task_details` on the typically-tiny survivor set). The cached `list_users` and activity log are free. On a typical morning where most projects have zero today's-due tasks on PM's plate, only a handful of `get_task_details` calls fire.

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
- Does not call `get_user_workload`. That's reserved for Job 6's no-history fallback path.
- Does not run in Mode 2. Mode 2 reads the morning queue from Notion, not Orbit.
- Does not double-emit signals into the normal pass. A task surfaced as a priority signal is excluded from the normal pass's `new_task` signal type (deduplicate by `task_id`).

---

## MANDATORY tool call sequence — no shortcuts (Normal pass)

This block overrides any inferred "fast path" the runtime might take. Mode 1's normal pass MUST invoke the following Orbit MCP tools in this exact order on every run. Skipping any of them is a Mode 1 failure and the assertion in `modes/mode-1-morning-collection.md` will abort the run. The Priority Pass above runs FIRST (sequentially) and reuses the activity-log and `list_users` results emitted by this normal-pass sequence; the two passes share tool calls, they do not duplicate them.

1. `get_user_details` — PM identity + assigned projects list.
2. `list_projects` filtered by `project_owner_id` — owned-project enrichment.
3. **`get_activity_log` with `from_date = last_run_timestamp`** — the **non-skippable** comment + change-history pull. Every comment, status flip, new task, new assignment landing inside the lookback window arrives ONLY via this call. `get_user_workload` does NOT return comments and is not a substitute. Run this call on every project in the universe; do not pre-filter.
4. `list_task_comments` — fallback for any task flagged by activity log that needs comment-body detail (activity log entries may include comment IDs but not full text).
5. `get_project_task_list` — for overdue / unassigned / new-task filters per the steps below.
6. `get_task_details` — for context on flagged tasks.
7. `get_asset_attachment_summary_with_download_url` — for new attachments.
8. `list_clients` / `list_sub_clients` / `list_users` — relationship-map enrichment.

`get_user_workload` is **lazy-only** — invoked by `synthesis/pod-inference.md` on the no-history fallback path, never as a substitute for steps 3–6. A Mode 1 run whose Orbit tool trace is `[get_user_workload × N]` and nothing else is a SPEC VIOLATION.

If `get_activity_log` returns an MCP error, apply the retry policy from `connector-failure-notify.md` (4 attempts, 2s/5s/15s backoff). After 4 failures, log `orbit_activity_log_unavailable` in the Run Log and continue — but the Mode 1 assertion will surface the gap to the PM in the page summary.

## Scope

Signals scoped to the PM's projects. "PM's projects" means:

1. Projects where the PM is the owner (from `list_projects` with `project_owner_id` filter — but note this is a small set)
2. Projects where the PM is a follower (from `get_user_details` with `include_assigned_projects: true` — the big set)
3. Projects active in the last 6 months where the PM has been a task assignee or commenter

Combine all three and deduplicate by project ID. This is the "universe" for the morning run.

## What to pull per project

For each project in the universe:

1. **Activity log since the PM's last run** — `get_activity_log` with `from_date = last_run_timestamp`. This surfaces every field change, status transition, new comment, new task, new assignment.
2. **Overdue tasks where the PM is owner or follower** — `get_project_task_list` with `due_date_filter: "overdue"`. Flag if any.
3. **New tasks created since last run** — filter `get_project_task_list` results by `created_at >= last_run_timestamp`.
4. **Unassigned tasks in the PM's projects** — `get_project_task_list` with `assignee_id: 0`. These are orphans needing assignment.
5. **Task status changes that matter** — any task where status moved into `Waiting for Feedback`, `Client Review`, `Blocked`, or `Done` since last run. PM needs to know.
6. **New comments on tasks** — from activity log. Especially ones mentioning the PM (@ in comment content).
7. **New attachments on tasks or projects** — flag for PM's attention. Include filename and download URL from `get_asset_attachment_summary_with_download_url` if user may need to review.

## What to skip

- Field changes the PM made themselves (self-noise).
- Automated bot comments (e.g., "system updated due date").
- Projects closed more than 30 days ago (historical).
- Tasks in the "Archive" project status.
- Projects where the PM is a follower but hasn't been a task assignee or commenter in the last 6 months (cold projects).

## Output shape — per signal

Each signal is a structured record:

```
{
  "source": "orbit",
  "signal_type": "activity_log_entry" | "overdue_task" | "new_task" | "status_change" | "new_comment" | "new_attachment" | "unassigned_task",
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

The `tasks` array is populated from the `get_project_task_list` call the collector already makes per project (one call per project in the universe) — no new MCP call. It serves three downstream needs:

1. `synthesis/matcher.md` Job 5 — finding the PM-owned parent task to nest a sub-task under (filter `is_pm_owned == true`).
2. `synthesis/matcher.md` — composing the row's `orbit_task_link` column by looking up the parent task's `url` by `parent_task_id` when the signal itself doesn't carry the parent URL.
3. `synthesis/pod-inference.md` — computing `has_history_on_project` for candidate assignees via task-count rollups.

This map is used by `synthesis/matcher.md` to take a signal from Gmail (e.g., "email from jane@agencyx.com") and figure out it's about project 8426 because Agency X is the client for that project and Jane is a client contact.

It's also used by `synthesis/pod-inference.md` to compute candidate assignees per project.

## Tool calls

Use the following Orbit MCP tools (the `mcp__...orbit.` prefix matches whichever Orbit MCP namespace the user has installed):

- `mcp__...orbit.get_user_details` — for PM identity + assigned projects list
- `mcp__...orbit.list_projects` — filtered by `project_owner_id`
- `mcp__...orbit.get_project_details` — for per-project metadata
- `mcp__...orbit.get_activity_log` — for changes since last run
- `mcp__...orbit.get_project_task_list` — for overdue / unassigned / new tasks
- `mcp__...orbit.get_task_details` — for full context on flagged tasks
- `mcp__...orbit.list_task_comments` — for recent comments
- `mcp__...orbit.get_asset_attachment_summary_with_download_url` — for attachment summaries (but note: unreliable for non-txt — default to download-and-read per `writers/source-citation.md`)
- `mcp__...orbit.list_clients`, `mcp__...orbit.list_sub_clients` — for client/sub-client enrichment in the relationship map
- `mcp__...orbit.list_users` — for matrix-name → user_id resolution. Called once per Mode 1 run by `references/pod-matrix.md`; the user list is cached for the duration of the run.
- `mcp__...orbit.get_user_workload` — invoked **on-demand** by `synthesis/pod-inference.md` (NOT in the bulk per-run pull). Used only when `synthesis/matcher.md` Job 6 hits the no-history fallback path and needs an availability score for a small subset of role-fit candidates. Subject to the same retry policy.

## Performance

Projects universe is capped at \~400 (the typical WLIQ follower count). Most days only 10–50 of those have activity since the last run. Filter aggressively using the last-run timestamp before paginating through task lists.

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
- Does not collect workload proactively. `get_user_workload` is called lazily by pod-inference on a small candidate subset only when the matcher requests an availability check (per `SKILL.md` non-negotiable rule #6).
