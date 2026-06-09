> **This collector uses ONLY the Orbit MCP. Source allowlist — primary collection: Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand: Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever — including any that may seem relevant to a specific signal.**

> **Preflight (`preflight.md`) must have run before this collector is invoked. Do not call any tool until preflight has completed.**

# Orbit Collector

## Purpose

Pull overnight signals from Orbit that are relevant to **the PM's own todos** (every open task assigned to the PM). Return a structured list of signals ready for `synthesis/matcher.md`.

This collector runs in **two passes** during Mode 1:

- **Pass A — Priority pass** (Mode 1 sub-step 1d — local filter + cross-reference over Pass B's MCP responses; zero new MCP calls of its own). Narrow scope: parent tasks an AM created or reassigned to the running PM overnight, `due_date = today`. Output goes into a separate `priority_signals` list. Runs synchronously after Pass B's MCP responses land; does NOT block 2a (Gmail collector) from launching. See `## Priority Pass (Mode 1 sub-step 1d)` below.
- **Pass B — Normal pass** (Mode 1 sub-step 1e, parallel with the Gmail collector at 2a). Broad scope: every open task on the PM's plate (workload snapshot) + activity since `last_run_timestamp` (status changes, new comments, due-date moves, assignment changes). Output goes into the normal `orbit_signals` list. Behaves as documented in the sections below.

Mode 2 invokes neither pass — execution reads the morning queue from Notion, not from a fresh Orbit pull.

## Execution model — runs as a collection sub-agent (context isolation)

This collector runs as a **dispatched sub-agent** (the Task tool), launched by Mode 1 sub-step 1e. The point is **context isolation**: the bulk raw responses — the up-to-500-task `get_user_workload` dump and the per-project `get_activity_log` logs (each potentially hundreds of entries before/after date-bounding) — live and die inside the sub-agent's own context. They never enter the main routine's context window.

The sub-agent does, in its own context:
- `get_user_workload(PM)` (step 1 of the mandatory sequence),
- the per-project `get_activity_log` loop (step 2),
- `get_primary_account_manager_users` (step 3) + the priority-pass local filter (Pass A),
- HTML→plain-text cleaning of every signal's content/excerpt, the comment-tree flatten + newest-first sort, and `latest_signal` computation,
- relationship-map assembly.

The sub-agent returns to the main routine **only the distilled, structured outputs** (no raw dumps):
- `priority_signals[]` and `orbit_signals[]` (per § Output shape — plain text, not HTML),
- the `relationship_map` (projects + tasks + `latest_signal`; **no** raw bodies or comment trees),
- the `workload_summary` aggregates and resolved `AM_user_ids`,
- explicit **assertion flags** — `workload_fetched: true|false` and `activity_log_projects_queried: <N of by_project count>` — so Mode 1 sub-step 1f can verify collection completeness from the flags rather than by inspecting a tool trace.

**What stays in the MAIN context (NOT in this sub-agent):** the **per-row deep-read** (`get_task_details`, § Per-row deep-read) is fired by the matcher during Job 7, in the main routine context — so the 6-section body composition reads each surviving task's full body + comment thread end-to-end with no fidelity loss. The sub-agent supplies the `task_id`s (via the relationship map and signals); the matcher does the deep reads where the judgment happens. `get_user_workload(candidate_user_id)` availability scoring (pod-inference, lazy) likewise runs in the main context.

## Universe model — user-scoped, NOT project-scoped

Orbit has a direct user-scoped workload API: `get_user_workload(user_id, is_completed, per_page)` returns the **entire list of open tasks assigned to that user** in a single call, plus summary counts (overdue, due today, by project) and full per-task details. The PM's morning universe IS their workload — there is no need to enumerate "projects PM owns / follows / was active in for the last 6 months" and then iterate `get_project_task_list` per project. That iteration produced N API calls of mostly-irrelevant tasks (tasks PM follows but isn't assigned to). The single `get_user_workload(PM)` call replaces it.

**The universe of interest = `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)`.** Every primary collection signal flows from this set. Change history comes from `get_activity_log`, which is **project-scoped** (a `project_id` is required) — so it is called once per project in the workload's `workload_summary.by_project[]` set (date-bounded to `last_run_timestamp`), not as a single global call. See § MANDATORY tool call sequence step 2.

Tasks the PM follows but is NOT assigned to are intentionally out of scope here — if a follower-only task needs PM input, that ask reliably surfaces through Gmail (the AM emails the PM) and is captured by `collectors/gmail.md`. The morning queue is about the PM's own todos plus the items the PM hasn't yet acknowledged; Gmail is the safety net for everything outside the PM's direct task list (per Mode 1 Step 2 § Role of Gmail and matcher Job 5 § Unactioned client signal → Create parent task).

## Priority Pass (Mode 1 sub-step 1d — local filter, no MCP calls of its own)

This pass is a **local filter + cross-reference computation** over the MCP responses fetched by the normal pass (1e). It makes ZERO new MCP calls — `get_user_workload(PM)`, the per-project `get_activity_log` loop, and `get_primary_account_manager_users` are all shared with 1e (called once, reused here). The "sequential, BLOCKING" framing from earlier spec versions was a defensive measure to ensure `priority_signals[]` was populated before matcher Job 1 read it; that dependency is now encoded explicitly at the matcher consumer (Job 1 entry-gate), not at the Step 1 launch graph. As a result, 2a (Gmail collector) does NOT wait for 1d to complete — Gmail has no Orbit dependency.

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
2. Call `get_primary_account_manager_users` once (cached for the run). This returns the org's AM users directly — match each Preferences AM to a returned AM by canonical email or alias to get its Orbit `user_id`. (This replaces the older `list_users` + full-directory email scan for AM identity; `list_users` is still used by `references/pod-matrix.md` for matrix-member resolution, a separate concern.)
3. Build `AM_user_ids = [ids of resolved AMs]`. Cache on the collector run state.

Edge cases:

- **Preferences has zero AMs configured.** Log `priority_pass_no_ams_configured` to Run Log. Return an empty `priority_signals` list. Mode 1 still proceeds — the normal Orbit pass + Gmail + Fathom run as usual.
- **An AM identity does not resolve to an Orbit user.** Log `priority_pass_am_unresolved: <am_name>` to Run Log and continue with the AMs that did resolve. Do not abort.
- **AM canonical email matches multiple Orbit users.** Log `priority_pass_am_ambiguous: <am_name>` and use the first match. The PM can disambiguate by editing the AM email in Preferences (a unique alias).

### Tool sequence

For the priority pass only (the normal-pass mandatory sequence still applies separately and reuses the same workload + activity-log responses):

1. **`get_primary_account_manager_users`** — already called once per run for AM identity (cached). Reuse the cache to get `AM_user_ids`.
2. **`get_user_workload`** — already called in the normal pass with `user_id == PM_user_id, is_completed=incomplete, per_page=500`. The priority pass reuses that response. Filter the returned task list to tasks where `due_date == today (IST)`. This is the candidate set — typically a handful of tasks. (The precomputed `workload_summary.due_today` count is a cross-check.)
3. **`get_activity_log`** — already pulled in the normal pass as the **per-project, date-bounded loop** over `workload_summary.by_project[]` (`startdate = last_run_timestamp`). The priority pass reuses those aggregated entries. For each candidate task from step 2, cross-reference the entries to confirm: (a) `created_by_id ∈ AM_user_ids` AND `created_at >= last_run_timestamp`, OR (b) an assignee-change entry exists for this task with the **actor (`user.id`) ∈ AM_user_ids** AND the change names the PM as the new assignee AND `timestamp >= last_run_timestamp`. Because activity entries identify the task and the new value in **HTML** (not structured fields), read the entry's `field_type` (assignee/reassign) + actor id + the `<b>…</b>` name in `description` to make the determination.
4. **`get_task_details`** — called per surviving candidate (a tiny set) to load the full parent task body for the matcher's sub-task brief. **The workload returns `description: null`, so this IS required to get the body** — it is the same single-call super-read used in Job 7 (§ Per-row deep-read), reused here so the matcher does not refetch later.

Tool budget for the priority pass: the `get_primary_account_manager_users`, `get_user_workload`, and per-project `get_activity_log` calls are all shared with the normal pass and cached (0 new calls for detection). `get_task_details` fires only on the typically-tiny surviving candidate set (workload carries no body).

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

This block overrides any inferred "fast path" the runtime might take. Mode 1's normal pass MUST invoke the following Orbit MCP tools in this exact order on every run. Skipping any of them is a Mode 1 failure and the assertion in `modes/mode-1-morning-collection.md` Step 1f will abort the run. The Priority Pass above runs FIRST (sequentially) and reuses the workload, per-project activity-log, and `get_primary_account_manager_users` results emitted by this normal-pass sequence; the two passes share tool calls, they do not duplicate them.

1. **`get_user_workload` with `user_id == PM_user_id, is_completed=incomplete, per_page=500`** — the **non-skippable** universe-discovery call. Returns the full list of every open task assigned to the PM, with per-task details (`project_id`, `project_title`, `due_date`, `status`, `severity`, `parent_id`) and a precomputed `workload_summary` (`overdue_tasks` / `due_today` / `due_this_week` / `by_status` / `by_severity` / **`by_project[]`** — the deduped set of projects the PM has open tasks in, each with a count). This is the canonical PM-todo universe for the run; every downstream signal flows from it. Replaces the prior `list_projects` → `get_project_task_list × N` per-project iteration entirely. **The workload returns `description: null` per task — task bodies are NOT included here**; the full description + comments arrive later via the per-row deep-read (§ Per-row deep-read), not in this call. PM identity is confirmed by preflight's `get_user_details` ping (`preflight.md` Step 5) plus the PM `user_id` from Preferences — the collector does **NOT** re-call `get_user_details` (one fewer round-trip; the preflight result is reused).
2. **`get_activity_log` — looped per project, date-bounded** — the **non-skippable** change-history pull. Every comment, status flip, due-date move, new assignment, AM action landing inside the lookback window arrives ONLY via this call. Workload returns the static snapshot; activity_log returns the deltas since last run. **`get_activity_log` REQUIRES a `project_id` — there is NO user-only / global feed.** Iterate over the deduped project set from step 1's `workload_summary.by_project[]` (typically 5–15 projects) and call, per project:
   `get_activity_log(project_id=<p>, user_id=PM_user_id, startdate=<last_run_timestamp as YYYY-MM-DD>, timezone="Asia/Kolkata", per_page=500)` — page on `pagination.has_more`. The `user_id` filter returns activities where the PM is actor OR assignee; **`startdate` date-bounding is mandatory** (an unbounded single-project log can exceed 900 entries / 190+ pages). Activity entries identify their task by **task name embedded in the `description` / `details` HTML** (e.g. `"… in <b>Task22</b> task"`), NOT a structured `task_id` field — map each entry back to a workload `task_id` by matching that name against the step-1 task list. **Name-collision rule** — normalise the extracted name before matching (strip HTML tags, lowercase, collapse whitespace, strip punctuation), then: (a) **exactly one match** → map directly; (b) **multiple matches** → disambiguate by status — prefer `status ∈ { "In Progress", "Active", "Open" }` over terminal states (`"Done"`, `"Closed"`, `"Archive"`) — if that yields one, use it; if still multiple after disambiguation, attach the activity entry to **all matching `task_id`s** (each gets the delta signal independently; matcher Job 4 deduplicates into separate rows so the PM sees both candidates — do NOT pick arbitrarily or drop); (c) **zero matches** → the task was likely completed, archived, or belongs to another user — log `activity_log_unmatched_entry: { project_id, extracted_name, entry_excerpt }` to the run's Incidents and skip; do NOT drop silently. **Co-issued in the same per-project loop:** `get_project_details(project_id)` — one call per project in `by_project[]`, issued in parallel with each `get_activity_log` call (adds zero serial latency — same project set, same loop iteration). Returns `project_number` + `followers[]` + project metadata, folded into the relationship map for that project. See § `project_number` enrichment for why this runs eagerly at collection time rather than lazily post-filter. This per-project loop is **NOT** the retired per-project iteration: it never calls `list_projects` or `get_project_task_list`, and the project set comes free from the single workload call.
3. **`get_primary_account_manager_users`** — enumerate the org's Account Managers directly for the priority pass. Match the returned AM users against the PM's configured AM(s) in Preferences (canonical email + aliases) to build `AM_user_ids`. Replaces the older `list_users` + manual email-matching path for AM identity. Called once per run, cached.
4. **`get_task_details`** — the per-row deep-read **super-call** (synthesis-phase, mandatory on the post-Job-5 survivor set). A single call returns the full task `description`, the **complete threaded comment tree** (`comments[]` with nested `replies[]`), `subtasks[]`, `followers[]`, and `task_attachments[]` (each with a pre-extracted `summary`). One call covers everything Job 7 needs — see § Per-row deep-read. (Also serves as a collection-phase fallback for any single task whose body is needed before synthesis.)
5. **`list_task_comments`** — **fallback only.** `get_task_details` (step 4) already bundles the full comment tree, so this is NOT a default per-row call. Use `list_task_comments(task_id)` only when the bundled tree appears paginated / capped (very long histories) or when the `comment_filter` / `group_by_type` split is needed.
6. **`get_asset_attachment_summary_with_download_url`** — call ONLY when a task/project `task_attachments[].summary` is empty or insufficient. The `summary` field is pre-extracted in the `get_task_details` / `get_project_details` payloads, so most attachments need no separate call.
7. **`list_clients` / `list_sub_clients` / `list_users`** — relationship-map enrichment for client / sub-client / actor identification on the workload's per-task project info. `list_users` is also used by `references/pod-matrix.md` for matrix-member name→id resolution (cached once per run).

**`list_projects` and `get_project_task_list` are no longer in the mandatory sequence.** The PM's task universe comes from `get_user_workload(PM)`, not from project enumeration. These tools remain available for narrow secondary use cases (e.g., the PM's owned-project list, if needed for non-collection purposes), but are NOT called during the normal collection pass.

`get_user_workload` now has TWO uses in this skill, with different `user_id` targets:

- **PM-workload (this collector, mandatory)** — `get_user_workload(PM_user_id, ...)` is the morning-collection universe-discovery call. Always called in Mode 1 Step 1d/1e.
- **Candidate-availability (pod-inference, lazy)** — `get_user_workload(candidate_user_id, ...)` is called by `synthesis/pod-inference.md` ONLY on the matcher Job 6 no-history fallback path, to score availability of a small candidate subset. Per SKILL.md non-negotiable rule #6.

The Mode 1 Step 1f assertion checks BOTH that `get_user_workload(PM)` was called (step 1 above) AND that `get_activity_log` was called at least once per project in `workload_summary.by_project[]` (step 2) — a tool trace missing the workload call, or showing zero `get_activity_log` calls when the workload had projects, is a hard abort. A trace that shows only `get_user_workload(PM)` and no `get_activity_log` is a SPEC VIOLATION — change detection requires both calls. (Under the collection-sub-agent model below, the sub-agent returns these as explicit flags; the main routine's 1f reads the flags rather than re-inspecting a tool trace.)

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
6. **Today's-due signal** — tasks where `due_date == today (IST)` AND status is not `Done` AND not already in `priority_signals[]` (the priority pass takes precedence) emit a standalone `pm_task_due_today` signal. **First-class signal, not just a sort heuristic** — the single-pane mandate (the PM should never have to open Orbit to learn what's due today) requires every due-today task to surface even when nothing changed overnight. The signal carries `bypass_pm_action_filter: true` (a standing due date is not "handled" by a prior PM comment) and an `orbit_due_today` anchor (see § Per-task `latest_signal`). The matcher merges this signal by `task_id` with any other signal on the same task: if an actionable signal (new comment / Gmail ask / status change) also exists, that verb wins and due-today rides along as an AI-Notes annotation; if this is the only signal on the task, the matcher emits a `Flag` row anchored on the due date (matcher Job 5 due-today branch). Overdue tasks (item 5) and passive status changes are NOT promoted this way — they remain matcher drop-list candidates per the explicit scope decision (due-today only).
7. **New attachments** — flagged by activity log on the task. The filename + a pre-extracted `summary` already arrive bundled in `task_attachments[]` (from `get_task_details` / `get_project_details`); only call `get_asset_attachment_summary_with_download_url` when that bundled `summary` is empty/insufficient and the PM may need to review the file content.

## What to skip

- Field changes the PM made themselves (self-noise — actor_id == PM_user_id on field changes).
- Automated bot comments (e.g., "system updated due date").
- Tasks already surfaced in `priority_signals[]` (deduplicate by task_id between priority and normal passes).
- Tasks where status is `Archive` (already complete from PM's perspective).

## Comment ordering — flatten the thread tree, then sort descending (newest first)

**Comments are a threaded TREE, not a flat list.** Both `get_task_details` (the per-row deep-read super-call) and `list_task_comments` return comments as a nested tree: each top-level node carries a recursive `replies[]` array of nested replies at any depth. Top-level nodes are ordered **oldest first**, but a reply posted today can sit nested under a top-level comment from weeks ago — so **the newest activity on a task is frequently a deep reply, NOT the last top-level node.** Reading `comments[0]` (or even the last top-level comment) as "newest" is wrong for a tree.

**This collector flattens, then sorts on the way out.** After the MCP response lands and before relationship-map assembly:

1. **Flatten** the tree — walk every node and every nested `replies[]` recursively into a single flat list, preserving each node's `id`, `parent_id`, `comment`, `comment_type`, `user_id`, `user_name`, `created_at`.
2. **Sort that flat list descending by `created_at` — newest first.** Only after this flatten-then-sort is `comments[0]` guaranteed to be the newest comment anywhere in the thread.

Reasons:

- The matcher's mental model on every Create-subtask / Reopen-subtask / Hand-off / Flag row is "what just triggered this?" — answering that requires `comments[0]` to be the genuinely newest node, replies included.
- Long, deeply-threaded histories used to surface a stale comment at index 0; the matcher's attention drifted before it reached the actual trigger. Flatten-then-sort eliminates that by construction.
- Job 5.5 (existing-subtask reuse, see `synthesis/matcher.md`) reads sibling subtasks' last non-PM commenter — also computed off the flattened, newest-first list.

Apply uniformly:

- Per-task `comments[]` in the relationship map (flattened + newest-first).
- Anywhere a list of comments is returned to the matcher.
- The cached `get_task_details(task_id).comments` payload that survives across Job 7 deep-read consumers (and any `list_task_comments` fallback payload).

`latest_signal` (§ Per-task `latest_signal`) is computed off this flattened, sorted list — `comments[0]` is the `orbit_comment` candidate. If the MCP changes its source ordering in a future version, only this collector adjusts — downstream consumers continue to read `comments[0]` as newest.

## Per-task `latest_signal` field — the deep-read anchor

Every task entry in the relationship map carries a structured `latest_signal` field that names the single most-recent event on the task. The matcher (Job 7) picks the newest across the row's input sources and passes it through as `row.latest_signal_anchor`; the writer renders it as the row-detail `**Triggered by:** ...` line. Per SKILL.md non-negotiable rule #24 — every row must carry an anchor or be dropped to `filtered_signals`.

Selection rule for `latest_signal`. Compute across these candidate sources for the task and pick the entry with the most-recent `timestamp_iso`:

1. **`orbit_comment`** — newest comment on the task (`comments[0]` after the flatten-then-sort above — may be a nested reply, not a top-level comment). Source id = comment id, author = comment's `user_id` / `user_name`, excerpt = first 240 chars of the comment text (HTML stripped to plain text).
2. **`orbit_task_body_update`** — the task's own `updated_at` if it postdates every comment AND the corresponding activity_log entry on this task surfaces a `field_change` event affecting `title`, `description`, `due_date`, or `severity`. Source id = task_id, author = the actor that performed the update (from activity_log), excerpt = the field-change description or the first 240 chars of the body if a body update.
3. **`orbit_status_change`** — newest activity_log `status_change` event on this task. Source id = activity_log entry id (or task_id if entry id absent), author = the actor, excerpt = `"<from_status> → <to_status>"`.
4. **`orbit_due_date_change`** — newest activity_log `due_date_change` event on this task. Same shape as status_change; excerpt = `"<from_date> → <to_date>"`.
5. **`orbit_due_today`** — synthetic anchor for a task whose `due_date == today (IST)` (the `pm_task_due_today` signal, item 6 above). Source id = task_id, author = the task's assignee (the PM), excerpt = `"Due today (<due_date>)"`. **Timestamp = today 00:00 IST** — deliberately the start of the day so any *real* same-day event (a comment posted at 07:00, a status flip) outranks it and keeps the row actionable; this anchor only wins when nothing else happened on the task, producing a bare due-today `Flag`. Emitted only when item 6's conditions hold.

Tiebreaker: if two candidates share an identical timestamp, pick in the priority order above (1 > 2 > 3 > 4 > 5 — comments win over body updates win over status changes win over due-date changes win over the bare due-today anchor). Excerpt strips HTML tags, collapses whitespace, and caps at 240 chars.

The `latest_signal` field lives on every task entry in the relationship map regardless of whether the task surfaces as a queue row. The matcher reads it during Job 7 to anchor the row. Shape:

```
latest_signal: {
  source: "orbit_comment" | "orbit_task_body_update" | "orbit_status_change" | "orbit_due_date_change" | "orbit_due_today",
  id: <int — comment id, activity_log entry id, or task_id depending on source>,
  timestamp_iso: <ISO 8601>,
  author_id: <int>,
  author_name: <string>,
  excerpt: <string, ≤ 240 chars, plain text>
}
```

If the task has zero comments AND no activity_log events AND no body updates since creation (rare — fresh task with no activity beyond creation), set `latest_signal` to the creation event: `source: "orbit_task_body_update"`, `id: task_id`, `timestamp_iso: created_at`, author = `created_by_id` / name, excerpt = the task title. Never null — the matcher and writer assume the field is present on every task.

## Per-row deep-read (mandatory, fired by matcher Job 7)

The collection-phase calls above return the universe + the deltas since `last_run_timestamp`, which is enough for the matcher to decide which tasks become rows. But composing the proposed 6-section body for a row requires **deeper context than the workload snapshot or the activity-log delta provides** — specifically, the full task description and the complete comment history (including comments older than `last_run_timestamp`).

For every task that becomes a `Create subtask` or `Create parent task` row (typically 10-30 rows per morning), the matcher's Job 7 lazily fires a **single-call** deep-read. `get_task_details(task_id)` is a super-call — it returns everything the body composition needs in one response:

1. **`get_task_details(task_id)`** — full task `description`, the **complete threaded comment tree** (`comments[]` with nested `replies[]`, all-time — NOT date-filtered to `last_run_timestamp`), `subtasks[]`, `followers[]`, and `task_attachments[]` (each with a pre-extracted `summary`). Default-on, NOT gated on whether the workload returned a description (the workload always returns `description: null`). The bundled comment tree is the same shape `list_task_comments` returns; older decisions, prior client feedback, AM clarifications, failed attempts, and scope changes from earlier weeks/months all live here.
2. **`list_task_comments(task_id)` — fallback only.** Fire it for a row **only** when step 1's bundled `comments[]` looks paginated / capped (a very long history that may have been truncated) — then page back through the full all-time history. For the typical task, step 1 already returns the complete tree and this call does NOT fire. Calling it by default would re-fetch the same comments the super-call already returned (doubling comment tokens in context for no gain).

**`pm_task_due_today` bare Flags fire a *lighter* variant** — `get_task_details(task_id)` only (its bundled newest comments + `subtasks[]` are enough), no `list_task_comments` fallback, used solely to compose the row's task_brief and open-subtask note (see `synthesis/matcher.md` Job 7 § Light deep-read for due-today Flags).

**Why the deep-read fires during synthesis, not collection.** Firing at collection time would mean calling `get_task_details` for every task in the workload (typically 30-80), most of which won't become rows after Job 5 filtering — wasted calls and wasted context. Firing during Job 7 bounds it to the post-filter survivor set (typically 10-30 rows). This mirrors the lazy patterns already established for `fetch_enrichment()` (Fathom, Job 4b Pass 2) and `get_user_workload(candidate_user_id)` (pod-inference, Job 6 no-history fallback).

**Issuance is a parallel batch, not serial per-row.** Matcher Job 7 issues `get_task_details(task_id)` for EVERY row in the post-Job-5 survivor set as parallel tool calls in one LLM turn — Claude Code supports multi-tool-use per turn. Batch cap: 25 parallel tool calls per turn; with one call per row the matcher chunks survivor sets larger than 25 rows into multiple batches (rows 1–25, 26–50, etc.). Any `list_task_comments` fallback for a truncated row issues in a follow-up batch. Per-row failure semantics (one row's deep-read failure doesn't block other rows) are defined in `synthesis/matcher.md` Job 7 § Mandatory deep-read of the originating Orbit task.

**Mandatory, not conditional.** Same default-on rule as the Gmail-thread enrichment (see `synthesis/matcher.md` Job 7 § Mandatory email-thread enrichment): do NOT skip the `get_task_details` call because the workload snapshot or the Orbit task title looks "complete enough". The runtime cannot judge completeness without reading; reading first is the only way to know.

**Flag rows skip the deep-read — except `pm_task_due_today` bare Flags.** Ordinary Flag rows do not produce an Orbit body (per matcher Job 7 § Flag path), so no per-row deep-read fires; the PM resolves them manually via the Orbit task URL in the row's `orbit_task_link` column. The one exception is a `pm_task_due_today` bare Flag: it fires the *light* read (`get_task_details` only) so its task_brief can tell the PM where the task stands without opening Orbit (matcher Job 7 § Light deep-read for due-today Flags). Still no body, no assignee, no handoff.

## Output shape — per signal

Each signal is a structured record:

```
{
  "source": "orbit",
  "signal_type": "activity_log_entry" | "overdue_task" | "new_task" | "status_change" | "new_comment" | "new_attachment" | "pm_task_due_today",
  "bypass_pm_action_filter": <bool — true only on pm_task_due_today signals (and priority-pass signals); absent/false otherwise>,
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
      "id": <int — internal Orbit PK (NEVER user-facing per SKILL.md non-negotiable #22). Used only inside URLs and for MCP calls.>,
      "project_number": <string — user-visible Orbit project code (e.g. "16915"). Source: `get_project_details.project_number` or `list_projects.project_number`. THIS is the value rendered in Notion / Slack / handoff drafts as `#<project_number>`.>,
      "url_slug": <string — URL-encoded id (e.g. "51083298598"). Used by `url` field; NEVER user-facing as a project reference.>,
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
          "is_pm_owned": <bool — assignee_id == PM_user_id at collection time>,
          "latest_signal": {
            "source": <"orbit_comment" | "orbit_task_body_update" | "orbit_status_change" | "orbit_due_date_change" | "orbit_due_today">,
            "id": <int>,
            "timestamp_iso": <ISO 8601>,
            "author_id": <int>,
            "author_name": <string>,
            "excerpt": <string ≤ 240 chars, plain text>
          }
        }
      ]
    }
  ]
}
```

The relationship map is now **derived from `get_user_workload(PM)` response**, not from per-project iteration. Each task in the workload carries its project info (project_id, title, client, sub-client, AM, owner, followers, project_type). The collector deduplicates the workload's per-task project info into project-level entries for this map.

**`project_number` enrichment (eager, per-workload-project, multi-purpose).** The `get_user_workload` response does **not** carry `project_number` per task (confirmed against the live MCP). The sub-agent fetches `get_project_details(project_id)` for **every project in `workload_summary.by_project[]`** — one call per project, issued in parallel with each `get_activity_log` call in the mandatory step-2 per-project loop (adds zero serial latency), cached for the run. **Why eager, not lazy post-filter:** this sub-agent completes and hands off the relationship map to the main context before the matcher runs Job 5 filtering — the sub-agent has no visibility into which projects survive that filter. Waiting to fetch would mean the relationship map ships without `project_number` on any project, leaving every downstream consumer (matcher, writer) unable to render SKILL.md rule #22 (`#<project_number>`) without a separate MCP round-trip. Fetching for all workload projects (typically 5–15) is a bounded, predictable cost co-located with an existing loop. **This call is multi-purpose:** alongside `project_number` it returns the project `followers[]` (folded into the relationship map's project entry) and project-level metadata — so it doubles as relationship-map enrichment, not a project_number-only cost. (`list_projects` also returns `project_number` inline and can resolve it by `search_value` when a single project must be looked up off-workload.)

The `tasks` array is populated from the workload response directly — every open task assigned to the PM appears here with `is_pm_owned = true` (by definition, since workload only returns PM-assigned tasks). For projects where workload returned multiple tasks, all of them appear in the array; the project entry consolidates them.

It serves three downstream needs:

1. `synthesis/matcher.md` Job 5 — finding the PM-owned parent task to nest a sub-task under (filter `is_pm_owned == true`, which is true for every task in this map by definition).
2. `synthesis/matcher.md` — composing the row's `orbit_task_link` column by looking up the parent task's `url` by `parent_task_id` when the signal itself doesn't carry the parent URL.
3. `synthesis/pod-inference.md` — computing `has_history_on_project` for candidate assignees. Note: under the user-scoped universe, the relationship map's project list is bounded by projects where the PM has at least one assigned open task. Projects where the PM is a follower-only contribute to history scoring only through Gmail-routed signals (no Orbit follower-only signal reaches this map).

This map is used by `synthesis/matcher.md` to take a signal from Gmail (e.g., "email from jane@agencyx.com") and figure out it's about project 8426 because Agency X is the client for that project and Jane is a client contact. When the candidate project is NOT in the map (the PM has no open task there), the matcher does not give up — it resolves the project via lazy Orbit search (`list_clients` / `list_projects` / `get_client_details`) per matcher Job 5 § Unactioned client signal → Create parent task; only when search also fails does the signal become a project-not-found Flag for the PM.

It's also used by `synthesis/pod-inference.md` to compute candidate assignees per project that appears in the map.

## Tool calls

Use the following Orbit MCP tools (the `mcp__...orbit.` prefix matches whichever Orbit MCP namespace the user has installed):

- `mcp__...orbit.get_user_details` — PM identity is confirmed by **preflight** (`preflight.md` Step 5 ping); this collector does NOT re-call it. Still used lazily by `synthesis/pod-inference.md` to read a *candidate's* department.
- **`mcp__...orbit.get_user_workload`** — **the primary universe-discovery call.** Two distinct uses, distinguished by `user_id`:
  - **PM-workload (THIS collector, mandatory):** `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)`. Called once per Mode 1 run in step 1 of the mandatory sequence. Returns every open task assigned to the PM plus the precomputed `workload_summary` (including the deduped `by_project[]` set).
  - **Candidate-availability (`synthesis/pod-inference.md`, lazy):** `get_user_workload(candidate_user_id, ...)`. Invoked ONLY on the matcher Job 6 no-history fallback path to score availability of a small candidate subset. Per SKILL.md non-negotiable rule #6.
- `mcp__...orbit.get_activity_log` — for changes since last run. **`project_id` is required** — loop once per project in `workload_summary.by_project[]` with `user_id=PM_user_id`, `startdate=<last_run YYYY-MM-DD>`, `timezone="Asia/Kolkata"`, `per_page=500`; page on `has_more`. Map entries back to workload `task_id`s by the task name in the entry HTML.
- `mcp__...orbit.get_primary_account_manager_users` — enumerate the org's AMs for priority-pass `AM_user_ids` resolution (matched against the PM's Preferences AM). Replaces `list_users` email-matching for AM identity.
- `mcp__...orbit.get_task_details` — the per-row deep-read super-call: full `description` + threaded `comments[]` + `subtasks[]` + `followers[]` + `task_attachments[]`(with `summary`) in ONE call. Fired per surviving row in Job 7.
- `mcp__...orbit.list_task_comments` — **fallback only**, when `get_task_details.comments` looks paginated/capped, or for the `comment_filter`/`group_by_type` split.
- `mcp__...orbit.get_asset_attachment_summary_with_download_url` — only when a bundled `task_attachments[].summary` is empty/insufficient (note: unreliable for non-txt — default to download-and-read per `writers/source-citation.md`).
- `mcp__...orbit.list_clients`, `mcp__...orbit.list_sub_clients` — for client/sub-client enrichment in the relationship map.
- `mcp__...orbit.list_users` — for matrix-name → user_id resolution. Called once per Mode 1 run by `references/pod-matrix.md`; the user list is cached for the duration of the run. (AM identity now resolves via `get_primary_account_manager_users` above.)
- **`mcp__...orbit.get_project_details`** — **mandatory collection call**, one per project in `workload_summary.by_project[]`, co-issued in parallel with `get_activity_log` during the step-2 per-project loop. Fetches `project_number` + `followers[]` + project metadata for the relationship map. Runs inside the sub-agent at collection time — NOT lazy or post-filter (see § `project_number` enrichment).

**Not part of the standing relationship-map build — but called LAZILY by the matcher for off-workload project resolution + dedup** (`synthesis/matcher.md` § "Unactioned client signal → Create parent task"). These fire only when a Possible-Orbit-miss Gmail signal resolves to no project in the PM's workload map — a low-volume escalation, not part of normal collection:
- `mcp__...orbit.list_clients(search_value=…)` / `mcp__...orbit.get_client_details(company_name=… | client_id=…)` — resolve the sender's CLIENT (id, AM, `contact_people` emails, `website_link` domain) when the workload map has no candidate. Note: `get_client_details` returns project *counts*, not the project list.
- `mcp__...orbit.list_projects(client_id=… | search_value=…)` — the client's actual project list (each with `id`, `title`, `project_number`, `owner_id`, `account_manager_id`) for topic disambiguation; or search projects by name/number directly.
- `mcp__...orbit.get_project_task_list(project_id, search=…, is_completed="incomplete")` — dedup check ("if not already") before proposing a new parent on the resolved project.
- `mcp__...orbit.get_project_details(project_id)` — fetch `project_number` (and follower/owner info) for a **search-resolved off-workload project** so it renders correctly. Workload projects are enriched eagerly at collection time via the mandatory step-2 per-project loop — not via this lazy path.

All four are read-only and auto-`allow`; `create_task` (the only write in this flow) stays `ask`-gated and runs in Mode 2 after PM approval.

## Performance

Single primary API call (`get_user_workload(PM)`) returns the entire universe — typically 30–80 open tasks for a working PM, capped at `per_page=500`. Per-project fan-out in the step-2 loop: **one `get_activity_log` + one `get_project_details` per project** (typically 5–15 projects; both calls parallelizable within the same loop — zero extra serial latency for `get_project_details`). Synthesis phase: **one `get_task_details` pre-batch in Job 5.5** for all `Create subtask` survivors (typically 10–30 rows), with Job 7 reusing those cached results — no double-fetch. `list_task_comments`, `list_task_comments` fallback, and `get_asset_attachment_summary_with_download_url` fire only as fallbacks (truncated comment tree / empty bundled summary). Compared to the prior project-iteration model (which fired N task-list calls for N=30-50 projects PLUS a two-call deep-read per row), the collection step is bounded by project count, not task count, and the synthesis deep-read is halved.

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
- Does not iterate per project **for universe discovery**. The universe is user-centric (`get_user_workload(PM)`) — `list_projects` and `get_project_task_list` are NOT called during normal collection; the old per-project task-list iteration (N API calls of mostly-irrelevant tasks) is fully retired. **Note the one legitimate per-project loop:** `get_activity_log` requires a `project_id`, so change detection loops it once per project in the workload's `by_project[]` set — that loop reads only the PM's own activity (`user_id=PM` filter, date-bounded) and never enumerates projects the PM has no task in.
- Does not detect follower-only tasks (tasks the PM follows but is not assigned to). If a follower-only task needs PM input, the ask reliably surfaces through Gmail and is captured by `collectors/gmail.md`; matcher Job 5 routes it via the standard or Possible-Orbit-miss paths.
- Does not detect unassigned project orphans. Same rationale — if an unassigned task genuinely needs the PM, an AM will email about it.
- Does not collect candidate-workload proactively. `get_user_workload(candidate_user_id)` (different `user_id` than the PM-workload call) is invoked lazily by `synthesis/pod-inference.md` on the matcher Job 6 no-history fallback path only, per `SKILL.md` non-negotiable rule #6. The PM-workload call IS made proactively every Mode 1 run — that is its intended use.
