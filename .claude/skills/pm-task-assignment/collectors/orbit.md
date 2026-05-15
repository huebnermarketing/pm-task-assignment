> **This collector uses ONLY the relevant MCP from the 6-MCP allowlist. The allowlist is closed: Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal.**

> **Preflight (`preflight.md`) must have run before this collector is invoked. Do not call any tool until preflight has completed.**

# Orbit Collector

## Purpose

Pull overnight signals from Orbit that are relevant to the PM's projects. Return a structured list of signals ready for `synthesis/matcher.md`.

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
      "client_name": <string>,
      "sub_client_name": <string or null>,
      "project_owner_name": <string>,
      "project_owner_id": <int>,
      "account_manager_name": <string>,
      "account_manager_id": <int>,
      "followers": [{"id": <int>, "name": <string>}, ...],
      "recent_task_assignees": [{"id": <int>, "name": <string>, "task_count": <int>}, ...],
      "department_usage": [{"department_name": <string>, "hours_used": <string>}, ...],
      "client_contacts": [{"name": <string>, "email": <string>}, ...]
    }
  ]
}
```

This map is used by `synthesis/matcher.md` to take a signal from Gmail (e.g., "email from jane@agencyx.com") and figure out it's about project 8426 because Agency X is the client for that project and Jane is a client contact.

It's also used by `synthesis/pod-inference.md` to compute candidate assignees per project.

## Tool calls

Use the following Orbit MCP tools:

- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__get_user_details` — for PM identity + assigned projects list
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__list_projects` — filtered by `project_owner_id`
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__get_project_details` — for per-project metadata
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__get_activity_log` — for changes since last run
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__get_project_task_list` — for overdue / unassigned / new tasks
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__get_task_details` — for full context on flagged tasks
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__list_task_comments` — for recent comments
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__get_asset_attachment_summary_with_download_url` — for attachment summaries (but note: unreliable for non-txt — default to download-and-read per `writers/source-citation.md`)
- `mcp__a5f9e856-d0f8-45b9-bb59-a9633796f0bd__list_clients`, `list_sub_clients` — for client/sub-client enrichment in the relationship map

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
