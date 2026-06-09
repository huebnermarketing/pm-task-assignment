> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes routine triggers — preflight runs even when invoked by a scheduled cloud routine.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand — Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Mode 1 — Morning Collection Run

## When this runs

- Scheduled: daily at the PM's configured morning time (default 9:30 AM IST), fired by a Claude Routine.
- Manual: `PM Task Assignment, run morning`.

## PM input during this run — surface-dependent

Mode 1's question behavior depends on the `is_interactive` flag set during preflight (see `ENVIRONMENTS.md`).

- **Headless surface (`is_interactive = false`):** never asks questions. Never blocks waiting for input. Never confirms with the PM. Routines fire unattended — there is no human in the loop at fire time. If something is uncertain, it becomes its own row with an `Uncertain:` note in AI Notes. The PM resolves it when they review.
- **Interactive surface (`is_interactive = true`):** may ask **one** clarifying question per ambiguous item before falling back to `Uncertain:`. Use this sparingly; the PM's morning attention is precious. Everything else (write order, plain-language enforcement, Notion structure, run-log) is identical across surfaces.

## End-to-end flow

> **PM mental model.** Mode 1 runs in four conceptual steps matching how a PM thinks about the morning: **(1) Pull from Orbit** (todos due today + open items needing attention), **(2) Pull from Mail** (enrichment + critical-miss safety-net), **(3) Enrich with Fathom** for any meeting references (with email-attachment fallback when Fathom is unavailable), **(4) Synthesize, recommend assignees, write to Notion**. The lettered sub-steps below preserve the technical sequencing (preferences, lookback, assertions, run-log, exit) without altering this conceptual flow.

### Step 1 — Pull from Orbit

The four-step PM flow's first conceptual step. Covers: Preferences load, lookback determination, Pod Matrix cache, Orbit priority pass, Orbit normal pass, and the post-collector assertion that protects against silent collector skips.

#### 1a. Read Preferences

Read the Preferences sub-page under the PM's Notion parent. If it doesn't exist, route to `first-run-setup.md` instead and stop.

Extract:
- PM identity (name, Orbit user ID)
- Last-run timestamp (if any)
- Morning run time, execution run time, escalation time + backup
- Account manager list with preferred channels
- Default email and handoff-channel preferences, always-include rules, tone samples

#### 1b. Determine lookback window

- Default: 12 hours (to catch overnight signals).
- If Preferences has a last-run timestamp: lookback = (now − last_run). This handles PM coming back from leave automatically.
- **Monday override (IST):** if today is Monday, set `lookback = max(now − last_run, 72 hours)`. Cron is weekday-only, so the previous run was Friday — Monday must always capture the full Fri/Sat/Sun window even if a manual weekend run shrank `last_run`.
- Cap lookback at 7 days to avoid overwhelming runs.

#### 1c. Load Pod Matrix (cached for the run, fan-out launch)

Launch in parallel with 1d (priority-pass filter), 1e (Orbit MCP fetch), and 2a (Gmail collector). Pod Matrix has no dependency on Orbit or Gmail responses — it is consumed by matcher Job 6 in Step 4, not by Step 1 collection. There is no reason to block the other Step 1 launches behind it.

Call `references/pod-matrix.md` to fetch and parse the org Pod Matrix from `POD_MATRIX_URL` (injected by the Mode 1 routine prompt — see `ROUTINE-ENTRYPOINTS.md`).

- On success: cache the parsed pools (PM matrix, floaters, functional matrices) plus the resolved Orbit user-id mapping for the rest of the run. `synthesis/pod-inference.md` reads from this cache.
- On `POD_MATRIX_URL` absent (interactive surface or routine misconfiguration), fetch failure, or parse failure: log a one-line warning to the Run Log Decisions trace and continue. Pod-inference will gracefully degrade to Orbit-only inference (matrix-unavailable code path).

This step is non-blocking: a matrix outage must never block the morning queue, and the matrix fetch never blocks any other Step 1 sub-step from launching.

#### 1d. Orbit priority pass (local filter, NOT a blocking fetch)

The priority pass is a **local filter + cross-reference computation** over the MCP responses fetched by 1e (Orbit normal pass). It makes ZERO new MCP calls of its own — it reuses 1e's `get_user_workload(PM)`, the per-project `get_activity_log` loop, and the cached `get_primary_account_manager_users` response (AM identity). The "sequential, BLOCKING" framing from earlier spec versions was a defensive correctness measure for the dependency between this pass and matcher Job 1; that dependency is now encoded explicitly at the matcher entry point, not at the Step 1 launch point.

**Launch model:** 1c (Pod Matrix), 1e (Orbit MCP fetch), and 2a (Gmail collector) fan out together right after 1b completes. 1d runs as a synchronous local computation as soon as 1e's MCP responses land — typically a few hundred milliseconds of filtering, no network. 2a does NOT wait for 1d to complete; it has no dependency on Orbit responses. The join point is 1f (post-collector assertion), which gates on `{1c, 1e (with 1d filtered output), 2a}` all completing.

**Matcher entry-gate:** Matcher Job 1 reads `priority_signals[]` first (per Sort rule 0). It will not start until 1d's local-filter computation has completed and populated `priority_signals[]`. This dependency is encoded at the consumer (matcher entry), not enforced by artificially blocking 2a.

Procedure (full details in `collectors/orbit.md` § Priority Pass):

1. Resolve AM identities from Preferences (canonical email + aliases) to Orbit `user_id`s via the cached `get_primary_account_manager_users` response (the org's AM list — match each Preferences AM by email/alias). Cached once per run.
2. From the PM-workload response (`get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)`, fetched by 1e — call once and reuse), filter to tasks where `due_date == today (IST)` AND `assignee_id == PM_user_id`. This is the candidate set — typically a handful of tasks.
3. Cross-reference against the per-project `get_activity_log` entries (also from 1e) for the AM actor (`user.id ∈ AM_user_ids`) on either a task-creation or an assignee-change naming the PM, since `last_run_timestamp`. Activity entries carry the task name + new value in HTML, so read the entry's `field_type` + actor id + `<b>…</b>` name to make the determination.
4. Return `priority_signals[]`, each carrying `signal_type: am_handed_to_pm_overnight_due_today`, `parent_task_id`, `am_actor_id`, `bypass_pm_action_filter: true`.

If Preferences has zero AMs configured, the priority pass returns an empty list. Log `priority_pass_no_ams_configured` to Run Log and continue. Mode 1 still completes — only the priority lane is empty for this run.

#### 1e. Orbit normal pass (fan-out launch)

Launches in parallel with 1c, 1d (which is a local filter over this pass's MCP responses), and 2a (Gmail collector). 1e is the **only** Orbit MCP-fetching step in Step 1 — 1d reuses its responses without making new calls.

**Dispatch as a collection sub-agent (context isolation).** 1e runs `collectors/orbit.md` as a **sub-agent** (Task tool). The bulk raw responses — the up-to-500-task `get_user_workload` dump and the per-project `get_activity_log` logs — stay inside the sub-agent's context and never enter the main routine's context window. The sub-agent returns only the distilled `priority_signals[]` + `orbit_signals[]` (plain text), the `relationship_map` (no raw bodies/comments), the `workload_summary` aggregates, resolved `AM_user_ids`, and explicit assertion flags (`workload_fetched`, `activity_log_projects_queried`). See `collectors/orbit.md` § Execution model. The per-row deep-read (`get_task_details`) is NOT done here — it fires later in the matcher's Job 7, in the main context, on the post-filter survivor set.

- `collectors/orbit.md` (normal pass — `priority_pass = false`) — universe is `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)` returning every open task assigned to the PM plus the precomputed `workload_summary` (including the deduped `by_project[]` set). Activity since `last_run_timestamp` comes from `get_activity_log` — which **requires a `project_id`**, so it is looped once per project in `workload_summary.by_project[]` (`user_id=PM`, `startdate=last_run` as YYYY-MM-DD, `timezone=Asia/Kolkata`, paged on `has_more`). Tasks surfaced in `priority_signals[]` by 1d's filter are deduplicated by `task_id` from the normal-pass output. **No per-project task-list iteration** — the workload call replaces the old `list_projects` + `get_project_task_list × N` loop entirely; the only per-project loop is the date-bounded `get_activity_log` over the workload's own project set. See `collectors/orbit.md` § Universe model for the rationale and § MANDATORY tool call sequence for the exact tool order.

**Fathom is NOT called in this step.** Fathom is enrichment-only and is invoked lazily by the matcher during Step 3 (Enrich with Fathom) whenever it detects a meeting reference inside a primary signal. See `collectors/fathom.md` for the enrichment interface and trigger-phrase list.

Each collector returns a list of raw signals with source metadata and full context.

If the Orbit normal pass fails (MCP unavailable, auth expired), do not abort Mode 1. Log a note and proceed:
> "Orbit was unavailable this morning — you may want to check manually."

This note is included in the summary section at the top of the dated page.

#### 1f. Post-collector assertion (MANDATORY — explicit join point for the fan-out)

**This sub-step is the join point for the Step 1 + Step 2 fan-out launches.** 1c (Pod Matrix), 1d (priority-pass filter, which depends on 1e MCP responses), 1e (Orbit MCP fetch), and 2a (Gmail collector) all run in parallel; 1f does not start until ALL of them have completed (or failed-and-logged). Once 1f passes (or aborts), Mode 1 advances to Step 3 (Fathom enrichment) and Step 4 (synthesis). Matcher Job 1 entry then reads `priority_signals[]` first (per Sort rule 0) — that dependency is encoded at the matcher consumer, not at the producer.

Before passing signals to the matcher (Step 4), verify the Orbit collector actually invoked its non-skippable tool sequence in 1e (which 1d also depends on). Because 1e runs as a collection sub-agent, the main routine reads the **assertion flags the sub-agent returns** (`workload_fetched`, `activity_log_projects_queried`, and the `workload_summary.by_project[]` count) rather than re-inspecting a raw tool trace. This guards against the sub-agent taking a "fast path" that pulls workload metadata only and silently drops every comment + activity-log entry inside the lookback window.

Additional priority-pass assertion: if `priority_signals[]` is non-empty, every entry must carry `bypass_pm_action_filter: true`. The matcher reads this flag to skip the PM-action drop rule (non-negotiable rule #19) for priority-lane signals. If the flag is missing on any priority signal, log `priority_pass_missing_bypass_flag` to Run Log and patch the flag to `true` before passing to the matcher.

Run the following checks on the Mode 1 tool trace and Orbit signals list:

1. **`workload_fetched` must be `true`.** `get_user_workload(PM)` is the universe-discovery call — every Orbit signal flows from its response. If the sub-agent reports `workload_fetched: false` (or omits the flag), abort Mode 1 with a hard error. Write the dated page with a single callout block at the top: `Mode 1 aborted — Orbit collector skipped get_user_workload(PM). No queue generated. See Run Log for trace.` Log to Run Log with code `mode1_abort: orbit_user_workload_skipped`. Do NOT write a queue database. Do NOT proceed to synthesis.

2. **`activity_log_projects_queried` must cover the workload's projects.** `get_activity_log` is the change-detection call — workload returns the static snapshot; activity_log returns the deltas since `last_run_timestamp`, looped once per project in `workload_summary.by_project[]`. If the workload had ≥ 1 project but `activity_log_projects_queried == 0`, abort Mode 1 with a hard error. Write the dated page with a single callout block at the top: `Mode 1 aborted — Orbit collector skipped get_activity_log. No queue generated. See Run Log for trace.` Log to Run Log with code `mode1_abort: orbit_activity_log_skipped`. Do NOT write a queue database. Do NOT proceed to synthesis. The PM sees a clear failure on the page instead of a misleadingly-clean queue built on incomplete data. (A partial loop — some but not all projects queried, e.g. a per-project MCP error — is NOT a hard abort: log `orbit_activity_log_partial: <queried>/<total>` to Run Log and continue with the projects that returned.)

3. **Orbit signals list contains ≥ 1 entry with `signal_type` in {`activity_log_entry`, `new_comment`, `status_change`, `new_task`, `overdue_task`}** — OR — the PM's workload was empty (PM has zero open tasks today, rare but legitimate) — OR — the workload was non-empty but `get_activity_log` returned a documented empty result (no changes since last_run). If the workload is non-empty AND activity_log returned non-empty results AND signals contain zero of those types, log a warning `orbit_collector_signals_dropped` to Run Log and continue. The PM will see "0 Orbit signals captured this morning" in the page summary as a soft flag.

4. **Tool trace MUST show `get_user_workload(PM_user_id, ...)` — and SHOULD show `get_user_workload(candidate_user_id, ...)` only on the no-history fallback path.** Per `SKILL.md` non-negotiable rule #6 there are two valid uses of `get_user_workload`: (a) PM-workload universe discovery in this collector (always fires, checked by assertion #1), (b) candidate-availability scoring in `synthesis/pod-inference.md` on the matcher Job 6 no-history fallback path (lazy). If `get_user_workload` was called with a `user_id` other than the PM AND no matcher Job 6 no-history fallback path actually fired this run, log warning `orbit_workload_candidate_call_unexpected` and continue. This catches accidental candidate-availability fan-out.

Output of this step: either a hard abort (cases 1 or 2) or a clean signals list with warnings logged (cases 3–4). Only on clean pass does synthesis run.

### Step 2 — Pull from Mail

The four-step PM flow's second conceptual step. Covers: Gmail collection, plus a cross-reference note for the matcher's Possible-Orbit-miss detection.

#### 2a. Gmail collector

Fires in parallel with 1e (Orbit normal pass) — they are independent, do not wait. Same lookback window as 1e (`<lookback_window>` from 1b).

- `collectors/gmail.md` — scoped to PM's WLIQ inbox, last <lookback_window>. Also emits `context_link` metadata per signal so matcher Job 4b can cross-link to Orbit signals.

**Role of Gmail — three duties, all default-on.** Gmail serves three purposes simultaneously, all of which apply to every Mode 1 run regardless of how the Orbit signals look:

1. **Emit standalone signals** where a fresh ask, client question, or action item exists that the matcher will turn into its own row.
2. **Carry context-link metadata** that `synthesis/matcher.md` Job 4b Pass 1 attaches to EVERY Orbit signal — priority-lane and normal-pass alike. This is the default context-enrichment path: the matcher must check whether a related email thread exists for each Orbit signal and link the thread end-to-end via the Gmail collector's full-thread pull. **The bias is over-include context, not under-include.** Emails carry nuance — AM clarifications, prior decisions, client constraints, scope tweaks, deadline reasoning — that the Orbit task body rarely captures. The proposed Orbit sub-task body composed in matcher Job 7 MUST read through the attached email threads (even long, multi-day threads) and weave anything load-bearing into the DO / WHY / CONTEXT / DONE-WHEN / SELF-QA / REFS sections.
3. **Safety net for missed Orbit critical items** (Possible-Orbit-miss detection, executed in matcher Job 5 — see 2b below).

The same Gmail thread between the AM and PM about a priority-lane parent task gets ALL THREE treatments: it surfaces as Sources context under the priority-lane row, it enriches the proposed sub-task body in Job 7, AND (if it contains a fresh ask) may also surface as its own row.

If Gmail fails (MCP unavailable, auth expired), do not abort Mode 1. Log a note and proceed:
> "Gmail was unavailable this morning — you may want to check manually."

This note is included in the summary section at the top of the dated page.

#### 2b. Possible-Orbit-miss safety-net (cross-reference to matcher)

The matcher (`synthesis/matcher.md` Job 5 § Unactioned client signal → Create parent task) inspects every Gmail signal for an unactioned client email that should have become an Orbit task: empty `context_signals[]` after Job 4b Pass 1 AND no PM action on the thread AND external sender, plus any one of three trigger sub-classes — S1a (client reply delivering something WLIQ asked for), S1b (delivery tokens: "here is / as requested / attached"), or S2 (issue / error / feature-request mail, the original critical-language class). It then resolves the project (workload map → lazy Orbit search) and dedup-checks it: resolved, non-duplicate → `Create parent task` tagged `Possible Orbit miss:`; project not found or a topic-matching open task already exists → `Flag` for the PM. Mode 2 creates the parent on approval (or after the PM names the project).

This detection is executed inside the matcher (Step 4), not by the Gmail collector. Documented here under Step 2 as a cross-reference for the PM mental model: pulling mail also serves as the safety net that catches items the PM may have missed on the Orbit side — including deliveries the client returned that nobody turned into a task.

### Step 3 — Enrich with Fathom (lazy, on-demand)

The four-step PM flow's third conceptual step. Covers: matcher trigger-phrase detection, lazy Fathom enrichment fetch, and Gmail-attachment fallback when Fathom is unavailable. Executed inside the matcher's Job 4b Pass 2 — documented here under its own parent step because it is conceptually independent of Job 5+ (which is the assignment/recommendation work in Step 4).

#### 3a. Matcher Job 4b Pass 2 trigger-phrase detection

For each primary signal (Orbit-priority + Orbit-normal + Gmail), scan for meeting-reference trigger phrases per the trigger-phrase list in `collectors/fathom.md`: direct-callback phrases (`"per our call"`, `"as discussed"`, etc.), meeting-title cites, attendee cites, `fathom.video/<id>` URLs, date+meeting references. Each detected reference becomes a candidate for enrichment fetch.

#### 3b. `fetch_enrichment()` calls per detected meeting reference

For each detected reference, call `collectors/fathom.md` `fetch_enrichment(reference, signal_context)`. The service picks the cheapest tool call per the lookup-strategy table (direct ID lookup, title search, attendee+date search, etc.). When a meeting is found, the service returns a scoped `EnrichmentResult` (summary excerpt scoped to the primary signal's project/actors; relevant action items only). The matcher attaches the result as `signal.enrichment.fathom` on the originating signal.

#### 3c. Gmail-attachment fallback when Fathom returns null (new)

When `fetch_enrichment()` returns `null` (Fathom MCP down OR no matching meeting found), Job 4b Pass 2 calls `gmail.find_transcript_in_email()` against the already-collected Gmail signal list (`collectors/gmail.md` § Transcript fallback). The helper searches Fathom-sender / attendee-sender emails ±2 days around the meeting reference for transcript-shaped attachments — no new Gmail MCP calls. If found, an `EnrichmentResult` with `enrichment_source: "gmail_attachment_fallback"` is attached as `signal.enrichment.fathom`. If the fallback also returns null, only then does the primary signal flow into Step 4 without any enrichment, and the Incidents log entry fires once per Mode 1 run with format: `Fathom enrichment unavailable AND no email-attachment fallback found for N primary signals.`

Fathom enrichment failure (and fallback failure) is **non-blocking** and is logged to the Incidents page only (no PM callout) — see `connector-failure-notify.md`.

### Step 4 — Synthesize, recommend, write & log

The four-step PM flow's fourth conceptual step. Covers: the rest of the matcher's jobs (group / dedup / uncertainty / summary / cross-link / action / assignee / body / handoff / email / notes / filtered trace), Notion write, Preferences last-run timestamp update, run-log append, and silent exit.

#### 4a. Matcher Jobs 1, 2, 3, 4, 4b Pass 1

Feed all collected signals (Orbit priority-lane + Orbit normal-pass + Gmail, each with any Fathom/gmail-attachment enrichment attached from Step 3) into `synthesis/matcher.md`. Jobs 1 through 4b Pass 1:

- **Job 1 — Group by project.** Use the Orbit relationship map to connect each signal to a project.
- **Job 2 — Dedup across sources.** Merge signals that are about the same item (same project + same topic + same actors + ±24h window OR explicit cite).
- **Job 3 — Uncertainty handling.** No probable-match grouping — list separately with `Uncertain:` AI Note.
- **Job 4 — Generate the one-line summary.** Topic-style — names the deliverable / scope, NOT the PM action. Verb lives in `Recommended Action` column only. Per the matcher's locked-vocabulary rules (5 verbs in `Recommended Action`: `Create subtask`, `Reopen subtask`, `Hand off parent task`, `Flag`, `Create parent task`).
- **Job 4b Pass 1 — Gmail → Orbit cross-link (default-on enrichment).** For EVERY Orbit signal (priority-lane and normal-pass), scan Gmail signals for corroborating context via project_id / actor_emails / topic_keywords / ±7-day time proximity, and attach matched Gmail signals as `context_signals[]` on the Orbit signal. The Gmail collector pulls the full thread end-to-end per `collectors/gmail.md` § Full thread context, so each attached signal carries every message in the thread — Job 7 reads through them when composing the proposed Orbit body. The bias is over-include — even weak matches attach. Cross-linking is additive — the same Gmail signal may also become its own row if it contains a fresh ask.

#### 4b. Matcher Jobs 5, 6, 7, 8, 9, 10, 11

- **Job 5 — Recommend action.** Pick exactly one of five locked verbs via Job 5 + 5a + 5.5: `Create subtask` (pod-resource work_type, no existing open subtask of same type), `Reopen subtask` (existing subtask of matching work_type found by Job 5.5), `Hand off parent task` (non-pod work_type: AUDIT, QUOTE, SEO, DESIGN, CONTENT, BA — parent reassigned to pool leader; no subtask), `Flag` (PM-attention only), `Create parent task` (Possible Orbit miss). Priority-lane signals always become `Create subtask` or `Reopen subtask` rows with `parent_task_id` PINNED to `signal.parent_task_id`.
- **Job 5a — Work-type classifier.** Inspects signal + originating task content; assigns `work_type ∈ HTML_CSS | PHP_BACKEND | QA | AUDIT | QUOTE | SEO | DESIGN | CONTENT | BA | OTHER`. Drives Job 5 verb branch and Job 6 pod-boundary filter.
- **Job 5.5 — Existing-subtask check.** For `Create subtask` survivors, inspects parent's `subtasks[]` (bundled in Job 7's `get_task_details` deep-read). If an open subtask of matching `work_type` exists, flips verb to `Reopen subtask` and sets `last_dev_user_id` from that subtask's `get_task_details(subtask_id).comments` (flattened newest-first; last non-PM commenter) — the same one super-call, no separate `list_task_comments`. Inactive-dev edge → `Uncertain:` AI Note.
- **Job 6 — Recommend assignee.** Call `synthesis/pod-inference.md` for the candidate pool. For `Create subtask` rows, pick via the 4-branch decision tree (history → matrix → floater → cross-matrix Uncertain) with pod-boundary filter driven by `work_type` (HTML_CSS → FE only, PHP_BACKEND → WP only, QA → QA only). For `Reopen subtask`, short-circuit to `last_dev_user_id`. For `Hand off parent task`, look up the pool leader from `functional_pools[<pool>]`. For `Create parent task`, short-circuit to PM. Availability calls (`get_user_workload`) fire only on the no-history fallback path for `Create subtask` rows.
- **Job 7 — Generate proposed Orbit body** (Create subtask + Create parent task paths). Full 6-section body per `schemas/orbit-dq-standard.md`, plain language. **Mandatory per-row deep-read of FOUR input sources** before composing the body — all default-on, none gated on whether the workload snapshot or the Orbit task title looks "complete":
  1. **Originating Orbit task in full** — lazy per-row `get_task_details(task_id)`, the single super-call that bundles the full description + the complete threaded comment tree (all-time, NOT date-filtered to `last_run_timestamp` — older comments hold prior decisions / failed attempts / scope changes) + `subtasks[]` + `task_attachments[]`. `list_task_comments(task_id)` fires only as a fallback when the bundled comment tree looks truncated. See `collectors/orbit.md` § Per-row deep-read.
  2. **Every attached Gmail `context_signal`** — thread end-to-end, including long multi-day threads.
  3. **Every `signal.enrichment.fathom`** — meeting summary + action items.
  4. **External docs** referenced by any of the above, per `references/external-doc-access.md` (Drive / Docs / Sheets / SharePoint, read-only).
  
  Weave extracted facts from all four sources into DO / WHY / CONTEXT / DONE-WHEN / SELF-QA / REFS per the source-agnostic mapping in `synthesis/matcher.md` Job 7 + `schemas/orbit-dq-standard.md` § Source rule. Flag rows skip body generation (and the full per-row Orbit deep-read) but still render every enrichment in row detail Sources for PM scanning. **Exception:** `pm_task_due_today` bare Flags fire a *light* deep-read (task details + newest comments only) to feed their "where this stands" task_brief — no body, no assignee, no handoff. See `synthesis/matcher.md` Job 7 § Light deep-read for due-today Flags.
- **Job 8 — Generate proposed handoff** (Create subtask path only). Plain-language team handoff draft. Flag + Create parent task rows skip this (no delegate).
- **Job 9 — Generate proposed email.** Skipped under current Output gating (PM handles emails outside the queue).
- **Job 10 — Write AI Notes.** Including `Uncertain:` flags and `Possible Orbit miss:` tags as applicable.
- **Job 11 — Emit Filtered signals trace.** For every signal the Output gating filter dropped, record source / summary / filter_reason / citations into the `filtered_signals` array. Writer renders this as a collapsible section on the Run Log detail page.
- **Sort rule 0** surfaces priority-lane rows at the top of the queue, ahead of all other ordering heuristics.
- **Sort rule 0.5** places due-today rows immediately below the priority-lane band, ahead of all non-dated rows. A row is in this band if it is a `pm_task_due_today` Flag OR any row with `row.due_today == true` (a task due today that also carried an actionable signal). The queue thus reads top-down as: AM-handed-due-today → other due-today → everything else. This is what lets the PM clear "what's due today" without opening Orbit.

#### 4c. Write to Notion

Call `writers/notion.md` to:
1. Archive last month's dated pages if today is the 1st (route to `modes/monthly-archival.md` first, then return)
2. Ensure `Preferences` sub-page is positioned at the very bottom of the parent
3. Resolve / create the Year heading-toggle block on the parent body (e.g., `2026`). Resolve / create the Month heading-toggle block inside it (e.g., `April`). Create today's dated sub-page (Notion-tree parent = parent page) and insert its `child_page` block at the TOP of the Month toggle's children. Title format: `DD Month YYYY` (e.g., `25 April 2026`). Per `schemas/parent-page.md` for the hybrid Year/Month-toggle + Day-sub-page structure. **If a `child_page` block matching today's title already exists in that Month toggle, do NOT overwrite and do NOT prompt.** Append a numeric rerun suffix and create a new sub-page: `25 April 2026 (rerun 2)`, `25 April 2026 (rerun 3)`, etc. Pick the lowest unused suffix. The original page is left untouched.
4. Write content into today's page:
   a. **Top of page: `Ready for Execution` toggle** — a to-do-style checkbox block, unchecked by default, labeled clearly
   b. **Summary line** — "N items for your morning. X sub-tasks, Y flags. <M signals filtered — see Run Log if you want to audit>."
   c. **Inline Morning Queue database** — schema from `schemas/morning-queue-database.md`, one row per item. **The queue is a real Notion database (`notion-create-database` + DB rows), NEVER page-body markdown headings.** Rendering rows as `### Row N — …` / `**Project:** …` blocks in the dated page body is a FAILED run — see `writers/notion.md` Step 5 item 4 HARD RULE.
   d. **Verification gate** — `writers/notion.md` Step 6.5 re-fetches the dated page and proves the database exists with one row per item before the run may report `OK`. The gate emits `notion_write_assertions` (db_created, db_row_count_matches, row_detail_pages_created, no_markdown_row_dump, single_db), passed to 4e below. A failed assertion forces `Status = Failed`.
5. For each row, populate the detail page with the heading-based layout from `schemas/row-detail-page.md`:
   - Summary heading
   - Sources heading (with citations from `writers/source-citation.md`)
   - Recommended Action heading
   - Proposed Orbit Task Body heading (from `schemas/orbit-dq-standard.md`, in plain language from `writers/plain-language.md`)
   - Proposed Handoff heading (plain language)
   - Proposed Email heading (if applicable, normal English)
   - AI Notes heading
   - Reference Context toggle at the bottom (labeled — skill's working memory)

#### 4d. Update last-run timestamp

Update the Preferences page's `last_morning_run` field to now.

> **Note:** Mode 2 (execution) and the escalation check are pre-scheduled as separate Claude Routines. Mode 1 does NOT register them in-skill. The `scheduled-tasks` MCP is no longer in the allowlist — routines themselves are the scheduler.

#### 4e. Append run-log entry

Call `writers/run-log.md` with the run summary:
- Timestamp range (start → end of this Mode 1 fire)
- Source counts per primary collector (Orbit / Gmail signal counts) + Fathom enrichments fetched (N enrichments attached, M references unresolved, K resolved via gmail-attachment fallback)
- Item count written to today's queue
- Decisions list (key synthesis decisions, especially `Uncertain:` flags, `Possible Orbit miss:` flags, and assignee picks)
- Connector status (which MCPs were healthy, which degraded, which failed)
- Page title actually written (including any `(rerun N)` suffix)
- `notion_write_assertions` — the queue-structure assertion set from the 4c.d verification gate (`writers/notion.md` Step 6.5). **Required on every Mode 1 run.** The run-log writer renders these as a `Queue structure assertions` section and overrides `Status` to `Failed` if any assertion failed — so a markdown-dump run can never be recorded as `OK`.

The writer creates a row in the Run Log database on the Notion parent and a linked decision-trace detail page. This is how a stateless routine fire leaves a trace for the next fire and for the PM's audit. The Run Log is the audit backstop: a green row must mean a real database was written, verified — not merely that no MCP call threw.

#### 4f. Exit silently

Mode 1 does not notify the PM on completion. The PM will open Notion on their own schedule. No email, no push. Silent.

Exception: if an MCP source failed, log the note on the page summary so the PM sees it when they open.

## What each row looks like after Mode 1

- **Summary** column: plain-language one-liner in normal English (PM reads this)
- **Status** column: set to `Recommended Action` by default
- **Recommended Action** column: short phrase like "Create task + handoff to Vijay"
- **Recommended Assignee** column: person name + short reason ("Vijay Patel (FE) — primary FE on Agency X")
- **PM Notes** column: empty (PM fills this)
- **Outcome** column: empty (Mode 2 fills this after execution)
- **Row page body**: full heading-based detail per `schemas/row-detail-page.md`

## Error handling

| Failure | Behavior |
|---|---|
| Preferences page missing | Route to `first-run-setup.md` and stop |
| Individual collector fails | Log note on page summary, continue with other collectors |
| Notion write fails | Retry once. If still fails, stop and email the PM (sent, not drafted): "Couldn't write today's morning queue — [error]. I'll retry on next scheduled run." |
| No signals at all | Still create today's page with summary "0 items for your morning. You're all caught up." and Ready toggle. Scheduled Mode 2 will see no work and exit cleanly. |
| Zero projects match the PM's Orbit scope | Write page summary "No projects found under your Orbit user. Check your Orbit account or update your identity in Preferences." |

## Performance expectations

Accuracy and clean output are primary. Speed is secondary.

- Wall-clock target: complete within 10–15 minutes for a typical morning (4 collectors in parallel, 20–50 signals, 10–30 items after synthesis, Notion write).
- No hard upper bound. If volume of signals, depth of synthesis, or external-doc fetches demand more time, take it. Better a slow, accurate queue than a fast, half-baked one.
- The Run Log records actual duration on every fire, so trends are observable and the PM can flag a regression if runs start consistently going beyond 15 minutes.

## What Mode 1 does NOT do

- Does not send any messages to anyone.
- Does not write to Orbit.
- Does not draft emails.
- Does not ask the PM anything.
- Does not touch V3 pages. See `references/v3-context.md`.
- Does not consult Keka or any leave data.
- Does not check availability proactively. Availability (`get_user_workload`) is called lazily by `synthesis/pod-inference.md` only on the matcher's no-history fallback path per `SKILL.md` non-negotiable rule #6.
- Does not write to the Pod Matrix Notion page — read-only via `references/pod-matrix.md`.
- Does not use toggles in row detail pages (except the one bottom reference-context toggle per row).
