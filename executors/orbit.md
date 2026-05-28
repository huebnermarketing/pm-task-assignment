> **This executor uses ONLY the Orbit MCP. Source allowlist — primary collection: Orbit, Gmail, Notion. Enrichment-on-demand: Fathom (lazy fetch via `collectors/fathom.md`). Slack is outbound-send only via `executors/slack.md` (team-handoff + AM-ping with explicit PM `send` note). Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever.**

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
- `description` — 6-section body per `schemas/orbit-dq-standard.md`, in plain language, with source citations per `writers/source-citation.md`. Read `orbit_task_header_style` from Preferences to pick header glyphs (`professional` default, or `emoji` if the PM opted in).

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

### Reopen an existing subtask (Reopen subtask path)

Triggered by rows whose `recommended_action` matches `Reopen subtask under #<existing_subtask_id> — ...`. These rows are emitted by `synthesis/matcher.md` Job 5.5 when an open subtask of the same `work_type` already exists under the PM-owned parent. The executor flips status, appends a new-work comment, and reassigns to the developer who last worked the subtask. No `create_task` call fires.

Required row fields (set by matcher Job 5.5):
- `row.existing_subtask_id` — the subtask to reopen.
- `row.last_dev_user_id` — the Orbit user ID of the last non-PM commenter (or fallback per Job 5.5 step 6).
- `row.new_work_description` — plain-language comment describing what the new signal adds (composed by matcher Job 7 Reopen-path rules).

Executor sequence (per row, all calls applied via the retry policy in `connector-failure-notify.md`):

1. **Guard: inactive last-dev.** If `row.last_dev_user_id` is null (matcher Job 5.5 step 7 flagged inactive dev or no-non-PM-history edge), skip the reassign + status flip entirely. Append only the comment via `add_task_comment(existing_subtask_id, new_work_description)` so the existing subtask carries an audit trail, and write Outcome: `HELD — last dev on subtask #<existing_subtask_id> is inactive; comment posted for audit. Please reassign manually.` Do not error; the row holds for PM.
2. **Reassign.** Call `update_task(existing_subtask_id, assignee=last_dev_user_id)`. If the subtask's current assignee is already `last_dev_user_id` (e.g. the dev never got formally unassigned when the subtask went idle), skip this call — no change needed.
3. **Status flip.** Call `update_task(existing_subtask_id, task_status_id=24)` to flip back to `To Do`. If the subtask is already at `To Do`, skip. If it's at a non-Done active state (`In Progress`, `In Review`, `Waiting for Feedback`), also skip — moving to `To Do` would lose progress. Append a note to the Outcome: `Status preserved at <current_status>; only comment + reassign applied.`
4. **Comment.** Call `add_task_comment(existing_subtask_id, comment=<header + new_work_description>)` where the header reads `Reopened — new work from <signal_date IST>:` and the body is `row.new_work_description` rendered as HTML (paragraphs, bold, lists allowed per `mcp__...orbit.add_task_comment` content rules). The comment cites the originating signal (Gmail thread URL or new Orbit comment URL).
5. **Outcome string.** Write Outcome: `Reopened subtask #<existing_subtask_id> under parent #<parent_id> → Orbit [link]. Reassigned to <last_dev name>. Comment posted. Handoff draft for <last_dev first_name> appended below.`

Edge cases:
- **Existing subtask was closed between Job 5.5 detection and Mode 2 execution.** If `get_task_details(existing_subtask_id)` returns `is_completed == 1` at execution time, do NOT reopen the closed subtask — fall back to the standard `create_task` Create-subtask path using the matcher's pre-composed 6-section body (matcher Job 7 always composes one even on Reopen rows, for exactly this rollback path). Write Outcome: `Existing subtask #<existing_subtask_id> was closed since Mode 1 — created new subtask #<new_id> under parent #<parent_id> instead.`
- **`update_task` reassign returns 403** (Orbit refuses reassign to the picked user — e.g. user lost project access). Skip the reassign, post the comment + status flip, and write Outcome: `HELD — could not reassign to <last_dev name> (403). Comment posted; please reassign manually.`
- **All Orbit calls succeed but the comment add fails.** Non-blocking. Log the failure and continue — the reassign + status flip already happened, so the dev sees the row in their queue; the PM can post the context comment manually.

### Hand off parent task (Hand off path)

Triggered by rows whose `recommended_action` matches `Hand off parent task to <pod leader> — ...`. These rows are emitted by `synthesis/matcher.md` Job 5 when `work_type ∈ {AUDIT, QUOTE, SEO, DESIGN, CONTENT, BA}` — work that does not touch HTML/PHP/QA pod resources. The executor reassigns the existing PM-owned parent task directly to the functional pool leader; no subtask is created.

Required row fields:
- `row.parent_task_id` — the PM-owned parent to hand off.
- `row.recommended_assignee_user_id` — the pod leader's Orbit user ID (set by matcher Job 6 Hand-off path).
- `row.work_type` — for the audit-trail comment.

Executor sequence:

1. **Reassign.** Call `update_task(parent_task_id, assignee=recommended_assignee_user_id, add_followers=[PM_user_id])`. The PM stays on as a follower for visibility — they remain the coordination anchor even though the named assignee is now the pool leader. If `recommended_assignee_user_id` matches the parent's current `assignee_id`, skip (no change).
2. **Audit-trail comment.** Call `add_task_comment(parent_task_id, comment=<header + handoff body>)` where the header reads `Handed off to <pool> lead per PM's morning queue (work_type: <work_type>).` and the body briefly cites the originating signal that triggered the handoff. This is internal-audit only — `comment_type: general`, never `client`.
3. **Outcome string.** Write Outcome: `Parent #<parent_task_id> reassigned to <pool leader name> (<pool> lead). PM kept as follower. Handoff comment posted.`

No new task is created. No 6-section body is written. The Notion row's `Proposed Handoff` section carries the plain-language handoff text the PM can copy if they want to ping the pool leader externally; the executor's only Orbit write is the reassignment + audit comment.

Edge cases:
- **Pool leader not resolved.** If matcher Job 6 surfaced `Uncertain:` (empty functional_pool / matrix unavailable), the row's `recommended_assignee_user_id` is null. The executor MUST NOT auto-pick a fallback. Write Outcome: `HELD — no pool leader resolved for work_type <work_type>. Please reassign manually.` PM resolves via note in a follow-up morning.
- **Pool leader matches PM.** If the resolved leader is the PM themselves (degenerate case — e.g. the PM also leads the Quoting pool), skip the reassign (no change in assignee) and write Outcome: `Handoff target resolved to you (PM) — parent stays on your plate. No reassign fired.`

### Create a parent task (Possible Orbit miss path)

Triggered by rows whose `recommended_action` matches `Create parent task on <project> assigned to you` — i.e., `Create parent task` verb rows emitted by `synthesis/matcher.md` Job 5's Possible-Orbit-miss detection. These rows surface a Gmail-only critical-language signal that had no corroborating Orbit task; on PM approval, this executor creates the missing parent.

Use `mcp__...orbit.create_task`.

Required parameters:
- `project_id` — `row.project_id` (resolved by the matcher; row will not have reached this path if project was ambiguous — see the matcher's project-uncertainty rule)
- `title` — `row.task_title` (already plain-language per the matcher's title-generation rules)
- `description` — `row.proposed_orbit_body`, the 6-section body. REFS section cites the originating Gmail thread by sender, subject, and thread URL — that citation is the entire basis for this parent's existence, so it must be present.
- `assignee` — PM's Orbit user ID. Parent goes on the PM's plate so future sub-tasks can nest under it via the standard `Create subtask` path in later runs.
- `parent_id` — **null / omitted**. This row creates a top-level parent, not a sub-task.
- `internal_followers` — include the PM (always); add the AM if the row's `signal_context.actor_emails` matches an AM identity from Preferences.
- `task_status_id` — 24 (To Do).
- `start_date` — today's date in YYYY-MM-DD.

Optional `due_date` derivation from the originating signal's urgency tokens:
- Tokens `today`, `eod`, `end of day`, `before tomorrow`, `cannot wait` → set `due_date = today`.
- Token `asap` → set `due_date = today` (treat as same-day).
- Token `urgent`, `critical`, `blocker`, `blocking`, `escalation`, `client is waiting`, `please do` (without an explicit date token) → leave `due_date` unset; the PM will set it after triaging.
- Multiple tokens → most aggressive wins (today over unset).

The due-date path goes through `change_task_due_date` Path A (initial set, no reason required) when an urgency token forces today — the create_task call itself does not always carry a due date, so the executor calls `create_task` first, captures the returned `task_id`, then calls `change_task_due_date` with the today date.

After creating, capture the returned `task_id` and Orbit task URL. Write to row Outcome:

```
Created parent task #<task_id> on <project_name> → Orbit [link]. Assigned to you. You can spawn sub-tasks under this in future runs.
```

If the matcher set a due date, append: ` Due <YYYY-MM-DD>.`

**Source Systems multi-select on the row.** Add `Orbit` to the multi-select (the row's original Source Systems was `Gmail` only). After execution, the row reflects both: `Gmail` (the originating signal) AND `Orbit` (the executor-touched system).

Edge cases:
- **`row.project_id` null at execution time.** Defensive check; the matcher's project-uncertainty rule should prevent this path. If it happens, skip execution and write Outcome: `Skipped — project unresolved at execution time. Please create the parent task manually on the right project.`
- **`create_task` MCP fails.** Standard retry-with-backoff per `connector-failure-notify.md` (4 attempts, 2s/5s/15s backoff). On exhaustion, write Outcome: `FAILED — parent task creation failed: <error>. Retry manually.`
- **Project exists but PM is not a follower.** Add PM as follower in `internal_followers` per the parameter rule above. Orbit will accept this even if PM is new to the project.

### Assign a new (sub)task

Pass `assignee` at create_subtask time.

Reassignment now has TWO queue-driven paths (per the 5-verb action set):
- **`Reopen subtask`** — executor reassigns the existing subtask to its last non-PM dev (see Reopen subtask path above).
- **`Hand off parent task`** — executor reassigns the existing PM-owned parent to the functional pool leader (see Hand off path above).

Pre-execution reassignment via PM note (e.g., `PM Notes: actually assign to Ravi instead`) is still resolved by `synthesis/note-interpreter.md` BEFORE any executor fires, so the relevant call (create_task, update_task on existing subtask, update_task on parent) uses the corrected assignee from the start. There is no post-execute reassignment path in Mode 2.

### Update a task

Use `mcp__...orbit.update_task`. Partial update — only pass fields that are changing.

Common update patterns:
- Move status to In Review: `task_status_id: <ID for In Review>`
- Update severity: `severity_id: <ID>`
- Update estimated hours: `estimated_hours: <number>`
- Add or remove followers: `add_followers: [...]`, `remove_followers: [...]`

### Change task due date

Use `mcp__...orbit.change_task_due_date`.

Two paths, depending on whether the task already has a due date:

#### Path A — Initial due date set (existing `due_date` is null)

For authorized users, Orbit now allows null → date without `reason` or `category_id`. Call the tool with the date only. Skip `reason` and `category_id`. Log on the row: `Initial due date set — no reason required.`

Use this path when `get_task_details` returns the task with `due_date == null` (i.e., the task was created without one and we are setting it for the first time).

#### Path B — Due date change (existing `due_date` is non-null)

The tool requires fresh `reason` + `category_id` on every call. Do not reuse from a prior call.

- `reason` — from the PM's note or the source signal's reason. Be specific. Example: `Client requested revised deadline via email on 25 April 2026.`
- `category_id` — pick from `references/due-date-categories.md` mapping. Default fallback: 3 (Other critical task/priority).

Detection: before calling the tool, fetch the task (or read the value already loaded by Mode 2 from the row's reference context) and branch on `due_date == null`. If the field state is unknown, default to Path B (safer — the tool will accept reason/category even on null→date for non-authorized users).

### Add a comment to a task

Use `mcp__...orbit.add_task_comment`.

- `comment` — HTML content (supports paragraphs, bold, lists, links). Follows the Orbit comment standards from `schemas/orbit-dq-standard.md`.
- `comment_type` — `general` (internal, default) or `client` (client-visible, use carefully).

Standard comment templates:

**When starting work:**
> "Starting work on this. My understanding of the requirement: [1-2 sentence summary]. Will update with progress."

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

Every Orbit task body's REFS section (last of the 6 sections) includes citations for every source that contributed to the task, per `writers/source-citation.md`.

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
- `Subtask #<id> created under parent #<parent_id> → Orbit [link]`
- `Created parent task #<id> on <project_name> → Orbit [link]. Assigned to you. You can spawn sub-tasks under this in future runs.` (Create parent task path)
- `Status updated → [new status] in Orbit [link]` (PM-note override only)
- `Due date changed → [new date] in Orbit [link] (category: [category])` (PM-note override only)
- `Severity bumped → [new severity] in Orbit [link]` (PM-note override only)

- `Reopened subtask #<id> under parent #<parent_id> → Orbit [link]. Reassigned to <last_dev_name>. Comment posted.` (Reopen subtask path)
- `Parent #<parent_id> reassigned to <pool leader name> (<pool> lead). PM kept as follower.` (Hand off parent task path)

Multiple operations combine with periods: `Subtask #110890 created under parent #110464 → Orbit [link]. Handoff draft for Hitesh appended below.`

## What this executor does NOT do

- Does not auto-send client-facing comments. `comment_type: client` requires explicit PM intent (via note).
- Does not change `project_owner_id` or `account_manager_id`.
- Does not delete tasks, projects, or comments.
- Does not bulk-update.
- Does not touch V3-related pages or projects (see `references/v3-context.md`).
- **Does not modify the `Orbit Task Link` column on the Morning Queue row.** That column is the matcher-frozen parent task URL, written once at Mode 1 row-create time. The newly-created sub-task URL goes into the `Outcome` column only (per the Outcome format above), preserving the parent reference for audit.
