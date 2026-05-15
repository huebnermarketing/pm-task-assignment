> **This executor uses ONLY the relevant MCP from the 6-MCP allowlist. The allowlist is closed: Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever.**

> **Preflight (`preflight.md`) must have run before this executor is invoked.**

# Orbit Executor

## Purpose

All write operations against Orbit. Called from Mode 2 (execution) when a row's action plan involves Orbit changes. All operations validated in production during the MVP build phase (side quest 3, 2026-04-25).

## Supported operations

### Create a task

Use `mcp__...orbit.create_task`.

Required parameters:
- `project_id` — from the row's matched project
- `title` — plain-language task title (processed through `writers/plain-language.md`)
- `description` — 6-section body per `schemas/orbit-dq-standard.md`, in plain language, with source citations per `writers/source-citation.md`

Optional parameters:
- `assignee` — Orbit user ID (from the recommended assignee or PM note override)
- `internal_followers` — list of user IDs (always include the PM; add the AM if relevant)
- `severity_id` — 17 (Normal) by default; 15 (Critical) or 16 (Important) per PM note
- `estimated_hours` — set if the task has a clear estimate from the source signal
- `task_status_id` — typically 24 (To Do) for new tasks
- `start_date` — today's date in YYYY-MM-DD

After creating, capture the returned `task_id` and Orbit task URL for the row's Outcome.

### Create a subtask

Same as create_task, but add `parent_id` pointing to the parent task.

Subtask titles should include the parent context. Example: `[Agency X Homepage] QA: verify mobile breakpoints`.

### Assign or reassign a task

For NEW tasks: pass `assignee` at creation time.

For EXISTING tasks (reassignments): use `mcp__...orbit.update_task` with `task_id` and new `assignee`.

When reassigning:
- Add an Orbit comment explaining the reassignment (general/internal comment)
- Example comment: `Reassigned from [old assignee] to [new assignee] on [date]. Reason: [brief reason from source signal or PM note]. Original assignment context is above.`

### Update a task

Use `mcp__...orbit.update_task`. Partial update — only pass fields that are changing.

Common update patterns:
- Move status to In Review: `task_status_id: <ID for In Review>`
- Update severity: `severity_id: <ID>`
- Update estimated hours: `estimated_hours: <number>`
- Add or remove followers: `add_followers: [...]`, `remove_followers: [...]`

### Change task due date

Use `mcp__...orbit.change_task_due_date`.

**Strict requirement** from the tool: fresh `reason` + `category_id` on every call. Do not reuse from a prior call.

- `reason` — from the PM's note or the source signal's reason. Be specific. Example: `Client requested revised deadline via email on 25 April 2026.`
- `category_id` — pick from `references/due-date-categories.md` mapping. Default fallback: 3 (Other critical task/priority).

### Add a comment to a task

Use `mcp__...orbit.add_task_comment`.

- `comment` — HTML content (supports paragraphs, bold, lists, links). Follows the Orbit comment standards from `schemas/orbit-dq-standard.md`.
- `comment_type` — `general` (internal, default) or `client` (client-visible, use carefully).

Standard comment templates:

**When starting work:**
> "Starting work on this. My understanding of the requirement: [1-2 sentence summary]. Will update with progress."

**When reassigning:**
> "Reassigned from [old] to [new] on [date]. Reason: [reason]."

**When marking complete:**
> "Completed. Here's what I did: [summary]. Staging URL: [link]. Self-QA: [what I checked]. Notes: [deviations or things to watch for in QA]."

**When blocked:**
> "Blocked: [specific reason]. What I need to proceed: [specific request]. Who can help: [person]. What I've tried: [what was already attempted]."

**When not completed on time:**
> "Not completed today. Reason: [specific reason]. Expected completion: [date/time]. Risk: [what's affected by the delay]."

### Create a project (rare)

Use `mcp__...orbit.create_project`. Only for true new-project intakes where the PM explicitly approves.

Required parameters:
- `title` — client name + project description
- `client_name` — matched from Orbit's client list
- `project_type` — inferred from the source signal (Fixed Cost, Ad-hoc, etc.)
- `account_manager_id` — from Preferences or inferred
- `project_owner_id` — the PM (895 for Ishant, etc.)
- `followers` — include PM + AM + Nishant

After creating, immediately create the first task under it (scoping / discovery task).

## Execution order

When multiple Orbit operations need to happen for one row:

1. **Create project** (if needed) — first, so the task has somewhere to live
2. **Create task** (or update existing task)
3. **Create subtasks** — after the parent exists
4. **Assign** (if not done at creation time)
5. **Change due date** — if different from the initial value
6. **Add comment** — last, so the comment can reference the now-existing task

## Plain-language enforcement

Every string that lands in Orbit AND will be read by the delivery team goes through `writers/plain-language.md`:

- Task title
- Task description (all 6 sections)
- Task comments

Strings in Orbit that are only for PM / AM eyes (internal admin notes) stay in normal English.

## Source citation

Every Orbit task body's `📎 REFS` section includes citations for every source that contributed to the task, per `writers/source-citation.md`.

If the task was derived from a document (PDF, image, PPT) that the skill read via download-and-native-read, the citation explicitly says so:
> "Sourced from `homepage_revision_feedback.pdf` attached to Orbit task #105892 — content read by the skill on 25 April 2026."

## Error handling

| Failure | Behavior |
|---|---|
| `create_task` fails | Log error in row Outcome: `FAILED — task creation failed: [error]`. Continue with other rows. |
| `change_task_due_date` fails (bad category_id or missing reason) | Retry once with fallback category 3 and an explicit test reason. If still fails, skip the due-date change but preserve the rest of the row's actions. Log the partial success. |
| `update_task` fails on a field | Retry with fewer fields. If still fails, log and move on. |
| `add_task_comment` fails | Non-blocking. Log the failure but do not fail the whole row. Try once more. |
| MCP auth expired mid-run | Abort the current row. Log: `FAILED — Orbit auth expired. Please re-authenticate and run execution again.` Continue with other rows that don't need Orbit. |

## After all operations for a row

Build the `Outcome` string for the Morning Queue row. Format is concise and specific:
- `Task created → Orbit [link]`
- `Reassigned [old] → [new] in Orbit [link]`
- `Status updated → [new status] in Orbit [link]`
- `Due date changed → [new date] in Orbit [link] (category: [category])`
- `Subtask created → Orbit [link]`

Multiple operations combine with periods: `Task created → Orbit [link]. Slack sent to Vijay.`

## What this executor does NOT do

- Does not auto-send client-facing comments. `comment_type: client` requires explicit PM intent (via note).
- Does not change `project_owner_id` or `account_manager_id`.
- Does not delete tasks, projects, or comments.
- Does not bulk-update.
- Does not touch V3-related pages or projects.
