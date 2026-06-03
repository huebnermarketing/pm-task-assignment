---
name: pm-task-assignment
description: Automates a WLIQ Project Manager's pre-office morning hour. Mode 1 runs in four conceptual steps matching how the PM thinks about the morning — (1) Pull from Orbit (priority pass for AM-handed parent tasks due today, then normal pass), (2) Pull from Mail (Gmail enrichment + safety-net for critical signals missed from Orbit), (3) Enrich with Fathom (lazy fetch when a primary signal references a meeting; Gmail-attachment fallback if Fathom is unavailable), (4) Synthesize + recommend assignees + write. Writes a plain-language morning queue to a Notion page with three row types — `Create subtask` (delegated dev work under a parent task currently assigned to the PM, with the 6-section brief), `Flag` (PM-attention items the matcher cannot auto-delegate), and `Create parent task` (Possible Orbit miss — Gmail-only critical signal with no corroborating Orbit task; Mode 2 creates the parent on PM approval). Executes the approved sub-task and parent-task creations in Orbit (handoff drafts appended to Notion under each row's Outcome) after the PM approves. Runs as 3 separate Claude Routines (Mode 1 morning collection, Mode 2 execution, monthly archival) on cron; same skill also runs interactively in Claude Desktop or programmatically via Claude Code / SDK (see `ENVIRONMENTS.md`). Manual out-of-routine commands also accepted — e.g. "PM Task Assignment, run morning", "PM Task Assignment, run execution now", "PM Task Assignment, validate setup" — documented in `invocation-commands.md`. Lenient about phrasing — variations like "skill PM task assignment, run my morning" or "PM task assignment run morning" all route correctly.
---

# PM Task Assignment

> **MANDATORY FIRST STEP on every invocation, scheduled or manual: load `config.md`, then run the preflight sequence in `preflight.md`. Do not proceed to mode logic, collectors, executors, or any tool call until preflight has completed successfully. This rule is non-negotiable and applies to every code path that reaches this skill.**

## Source allowlist — read this before touching any tool

This skill uses these MCPs as **primary collection sources**: **Orbit, Gmail, Notion**.

It uses **Fathom** as an **enrichment-on-demand MCP**: never collected eagerly, never a row of its own. The matcher (`synthesis/matcher.md` Job 4b) detects meeting references inside primary signals (mail / Orbit) using the trigger-phrase list in `collectors/fathom.md`, then lazily fetches the meeting summary, attendees, and relevant action items as enrichment on the originating primary signal. Fathom MCP failure is non-blocking — primary signals still flow.

It uses these MCPs as **on-demand read-only references** when an allowed primary signal links to a file there: **Google Drive, Google Docs, Google Sheets, SharePoint**. Read-only. Never write. Trigger condition: a primary signal (Gmail / Orbit / Notion) contains a link or attachment pointing at the document; the skill MAY fetch the doc's content for context. See `references/external-doc-access.md`.

It uses **Slack** as an **outbound-send-only MCP**, scoped to exactly two paths in Mode 2: (a) team handoff send when the row's PM Note explicitly says `send` AND audience = `team`; (b) AM ping send when the row's PM Note explicitly says `send` AND audience = `am`. Slack is NEVER used for collection (the Slack collector was removed), NEVER for PM self-summary (now a Notion callout block), NEVER for escalation backup (now email-only), NEVER for connector-failure notification (now email-sent → Notion red callout → Incidents). All Slack details live in `executors/slack.md`.

Any other MCP is forbidden — including any added to the user's Cowork after installation, and any the running user explicitly asks the skill to use.

This rule applies even under experimental scope, forced runs, sandbox runs, or any kind of override. If a signal seems relevant from a forbidden source, ignore it.

The clause is repeated in `config.md`, in `preflight.md`, and at the top of every collector / executor file. The repetition is deliberate.

## What this skill does

Eliminates the one hour WLIQ Project Managers currently spend from home every morning before the India office opens at 11:00 AM. The PM currently scans Gmail and Orbit for overnight signals (and occasionally a Fathom recording when an email or comment references one), figures out which project each signal belongs to, decides who should pick up each piece of work, writes Orbit tasks with enough context for the assignee to start, then walks desk to desk in the office to brief each team member.

This skill replaces all of that except the PM's approval moment. Each PM runs the skill on their own Claude account. Their MCP connections (Orbit, Gmail, Notion as primary; Fathom as enrichment-on-demand) provide the data. The skill writes the morning's work to a Notion page identified in `config.md`. The PM reviews, approves, and the skill executes.

**Execution model.** This skill runs under two surfaces: (1) headless — Claude Routines on cron, or programmatic Claude Code / SDK invocations — never asks questions; (2) interactive — a PM types a manual command in Claude Desktop — may ask one clarifying question per ambiguous item before falling back to `Uncertain:`. Both surfaces share the same preflight, allowlist, writes, plain-language rules, and run-log behavior. The only difference is question-asking. See `ENVIRONMENTS.md` for the surface-detection contract and what is shared. See `ROUTINE-ENTRYPOINTS.md` for the three baked routine prompts; `invocation-commands.md` for the interactive commands.

## The two modes

**Mode 1 — Scheduled morning run.** Fires automatically at the time configured in Preferences (default 9:30 AM IST). After preflight, pulls overnight signals. Synthesizes them into items. Writes today's dated sub-page at the top of the PM's Notion parent (the one identified in `config.md`). No PM input required during this run.

**Mode 2 — Scheduled execution run.** Fires automatically at the configured execution time (default 10:45 AM IST). After preflight, reads the page-level `Ready for Execution` toggle at the top of today's dated page.

- If **ON**, executes every row the PM marked `Approved` and every row with a PM note. Writes `Done` outcomes back. Writes a completion summary as a Notion callout at the top of today's dated page (the PM is already opening Notion to review — no email noise on normal mornings).
- If **OFF**, runs the inline escalation flow (Mode 2 Step 3a) — emails the backup person configured in Preferences and exits. Escalation is a fallback branch of Mode 2, not a separate mode.

## Invocation commands (manual overrides)

**These commands are for out-of-routine manual use only.** Routines do not parse user messages — they fire by cron with a baked prompt that loads this skill from the repo. See `ROUTINE-ENTRYPOINTS.md` for the 3 routine prompts. Manual commands below are used only when an operator runs the skill in a regular Claude session for catch-up or preference edits.

The PM types one of these in Claude when they want to override the schedule or edit configuration. The skill is **lenient** about phrasing — case-insensitive, comma-optional, "skill" prefix optional, "my" or "the" qualifiers fine. Any reasonable variation routes correctly.

- `PM Task Assignment, run morning` — manually fire Mode 1 (scheduled run missed, or on-demand morning read)
- `PM Task Assignment, run execution now` — manually fire Mode 2 (PM flipped Ready after the scheduled time passed)
- `PM Task Assignment, change my preference: [instruction]` — edit the Preferences page only; does not fire the morning flow
- `PM Task Assignment, update tone samples` — add or replace tone calibration samples for the plain-language writer

Variants Claude must accept and route correctly:

| Variant | Routes to |
|---|---|
| `PM Task Assignment, run morning` | Mode 1 |
| `pm task assignment, run morning` | Mode 1 |
| `Skill PM Task Assignment, run morning` | Mode 1 |
| `PM Task Assignment run morning` (no comma) | Mode 1 |
| `PM Task Assignment, run my morning` | Mode 1 |
| `PM Task Assignment, run the morning` | Mode 1 |
| `PM Task Assignment, run execution` | Mode 2 |
| `PM Task Assignment, run execution now` | Mode 2 |
| `PM Task Assignment, execute` | Mode 2 |
| `PM Task Assignment, change my preference: ...` | Preference edit |
| `pm task assignment change preference: ...` | Preference edit |

Internal logic for routing: any message that contains the skill name + an action verb (run, execute, change, update) + a context word (morning, execution, preference, tone) routes correctly. See `invocation-commands.md` for full routing rules.

## First run

The skill detects first run during preflight (step 3) by checking whether a `Preferences` sub-page exists under the Notion parent identified in `config.md`. If no Preferences page exists, the skill runs the first-run setup flow conversationally (see `first-run-setup.md`) and asks 9 setup questions.

Once Preferences is populated, every subsequent run reads it silently and proceeds.

## Architecture overview — what calls what

```
EVERY ENTRY POINT (scheduled or manual):
  → Load config.md
  → Run preflight.md (5 steps)
  → If preflight succeeds → continue to mode/command logic
  → If preflight fails → connector-failure-notify.md or first-run-setup.md or abort

Mode 1 (scheduled collection — 4 conceptual steps, lettered sub-steps below):
  → preflight
  → Step 1 — Pull from Orbit (1c + 1d + 1e + 2a fan-out launch after 1b; 1f is the join point):
    → 1a. Read Preferences (already loaded by preflight)
    → 1b. Determine lookback window (default 12h, Monday override = max(now − last_run, 72h))
    → 1c. references/pod-matrix.md — fetch + parse Pod Matrix from POD_MATRIX_URL (parallel; cached for run; gracefully degrades to Orbit-only on absence/failure)
    → 1d. Orbit priority pass (LOCAL filter over 1e's MCP responses — zero new MCP calls; runs synchronously after 1e responses land):
      → collectors/orbit.md with priority_pass=true
      → Detects parent tasks an AM created or reassigned to the running PM overnight with due_date=today (filter PM-workload by due_date=today, cross-ref activity_log for AM-actor events)
      → Output: priority_signals[] each carrying parent_task_id + am_actor + bypass_pm_action_filter=true
      → Matcher Job 1 entry-gates on this — dependency encoded at consumer, NOT by blocking 2a
    → 1e. Orbit normal pass (parallel with 1c/1d/2a — the ONLY Orbit MCP-fetching step in Step 1):
      → collectors/orbit.md normal pass: get_user_workload(PM) + get_activity_log + list_users (shared with 1d), plus comments/status/overdue/new tasks not in priority pass
      [Fathom NOT called here — lazy-fetched by matcher in Step 3 only when a primary signal references a meeting]
    → 1f. Post-collector assertion (MANDATORY — explicit join point for the {1c, 1d, 1e, 2a} fan-out) — aborts if get_user_workload(PM) OR get_activity_log were skipped
  → Step 2 — Pull from Mail:
    → 2a. collectors/gmail.md (parallel with 1c/1d/1e — launched at the same fan-out point; no Orbit dependency) — Gmail signals + context-link metadata (project_id_candidates, actor_emails, topic_keywords, timestamp)
    → 2b. Possible-Orbit-miss safety-net (cross-reference to matcher Job 5; executed during Step 4)
  → Step 3 — Enrich with Fathom (lazy, on-demand):
    → 3a. Matcher Job 4b Pass 2 — scan every primary signal for meeting-reference trigger phrases per collectors/fathom.md
    → 3b. fetch_enrichment(reference, signal_context) per detected reference — attach EnrichmentResult as signal.enrichment.fathom
    → 3c. Gmail-attachment fallback (NEW) — when fetch_enrichment returns null, call gmail.find_transcript_in_email() against the already-collected Gmail signal list; if found, attach as signal.enrichment.fathom with enrichment_source: "gmail_attachment_fallback"
    [Failure of both Fathom AND fallback is non-blocking — logged to Incidents only]
  → Step 4 — Synthesize, recommend, write & log:
    → 4a. synthesis/matcher.md Jobs 1, 2, 3, 4, 4b Pass 1 (group / dedup / uncertainty / summary / Gmail → Orbit cross-link)
    → 4b. synthesis/matcher.md Jobs 5, 5a, 5.5, 6, 7, 7b, 8, 9, 10, 11:
      → Job 5: action classification — one of FIVE locked verbs: Create subtask, Reopen subtask, Hand off parent task, Flag, or Create parent task (per `synthesis/matcher.md` Job 4 + Job 5 + 5a + 5.5)
      → Job 5a: work-type classifier (HTML_CSS | PHP_BACKEND | QA | AUDIT | QUOTE | SEO | DESIGN | CONTENT | BA | OTHER) — drives Job 5 verb branch and Job 6 pod-boundary filter
      → Job 5.5: existing-subtask check — flips `Create subtask` to `Reopen subtask` when an open subtask of matching work_type already sits under the parent (per the cached `get_task_details(parent_id).subtasks[]`)
      → Job 6: assignee routing — pod-boundary 4-branch tree (history → matrix → floater → cross-matrix Uncertain) for Create/Reopen subtask; pool-leader lookup for Hand off parent task; short-circuits for Flag (—) and Create parent task (PM)
      → Job 7: compose the 6-section Orbit body (Create subtask + Create parent task + as Reopen-fallback) — MANDATORY per-row deep-read of FOUR sources (unconditional, default-on):
        ① Originating Orbit task in full — per-row get_task_details + list_task_comments (full all-time comment history, NOT date-filtered — older comments hold prior decisions / failed attempts / scope changes). **Issued as parallel batch (cap 25 calls/turn, chunked if S > 12 rows); per-row failure → Uncertain AI Note or FAILED-deep-read flag.** `pm_task_due_today` bare Flags join the same batch with a *light* read (details + newest-page comments only, no full history) to feed their task_brief — no body/assignee/handoff (matcher Job 7 § Light deep-read for due-today Flags).
        ② Every attached Gmail context_signal — thread end-to-end, including long multi-day threads
        ③ Every signal.enrichment.fathom — meeting summary + action items
        ④ External docs referenced by any of the above (per references/external-doc-access.md)
        Weave facts from all four into DO/WHY/CONTEXT/DONE-WHEN/SELF-QA/REFS via the source-agnostic per-section mapping in matcher.md Job 7 + orbit-dq-standard.md § Source rule. The workload snapshot + activity_log delta alone are rarely complete enough.
      → Job 7b: compose row.task_brief (2-4 sentence narrative for the row detail page H1 Task Brief block between Summary and Sources) — pulled from the most recent input source (top of email thread, latest Orbit comment, AM clarification)
      → PM-action filter: bypass for signals with bypass_pm_action_filter=true
    → synthesis/pod-inference.md — compute pod per project (matrix members ∪ Orbit followers/recent-assignees)
    → 4c. writers/notion.md — write today's dated sub-page with inline database + Ready toggle at top
      → Priority-lane rows sort to the top (matcher sort rule 0)
      → Row detail Sources section renders linked Gmail context signals + Fathom enrichment (when fetched) on priority-lane rows
      → Enforces parent-page structure per schemas/parent-page.md
      → Enforces Morning Queue schema per schemas/morning-queue-database.md
      → Enforces row detail layout per schemas/row-detail-page.md
    → 4d. Update last-run timestamp in Preferences
    → 4e. writers/run-log.md — append run entry to Run Log database (with Fathom enrichment counts: N attached, M references unresolved, K resolved via gmail-attachment fallback)
    → 4f. Exit silently

Mode 2 (scheduled execution):
  → preflight
  → Read today's dated page — check Ready toggle
  → If OFF: run inline escalation flow (Step 3a) → email backup → stop
  → If ON:
    → For each row: read Status + PM Notes
    → synthesis/note-interpreter.md — resolve PM note intent
    → Route by row verb to executors:
      ├─ executors/orbit.md — Create subtask path (parent_id = parent_task_id, assignee = recommended_assignee; priority-lane sub-tasks pin parent_task_id to the AM-assigned parent)
      ├─ executors/orbit.md — Create parent task path (Possible Orbit miss; parent_id = null, assignee = PM, project_id = matcher-resolved; due_date derived from urgency tokens if present)
      ├─ Flag rows — no executor; PM resolves externally and marks Skip manually
      └─ executors/email.md (drafts only, plus the 2 documented send-not-draft exceptions: connector-failure tier 1, escalation backup ping)
    → writers/notion.md — write Done state + Outcome per row; team/AM handoffs always draft to the row's Outcome block (PM copies + sends). Create parent task rows generate no handoff (no delegate)
    → writers/plain-language.md — enforce language rule on outputs
    → writers/source-citation.md — cite every source in Orbit task bodies and elsewhere
    → writers/run-log.md — append run entry
  → Write completion summary as a Notion callout at the top of today's dated page (no email DM to PM on normal mornings)

Monthly archival (scheduled on 1st of month at 6:00 AM IST):
  → preflight
  → modes/monthly-archival.md — move previous month's dated pages into a named toggle
  → Enforces parent-page structure per schemas/parent-page.md
  → writers/run-log.md — append run entry

First run (no Preferences page exists):
  → first-run-setup.md — ask 9 questions, write Preferences page, register scheduled tasks

Connector failure at any step:
  → connector-failure-notify.md — 3 tiers: email PM (sent, not drafted) → Notion red callout on today's dated page → Incidents sub-page (repeat-failure escalation = email to backup)
```

## Data sources (all via MCP, scoped per the allowlist)

| Source | Scope | Tool family |
|---|---|---|
| **Orbit** | The PM's open task list — every task where `assignee_id == PM_user_id` returned by a single `get_user_workload(PM)` call. Plus `get_activity_log` for changes since last run (status flips, new comments, assignment changes, due-date moves) scoped to the workload task IDs. Plus attachments for tasks with new attachment activity. The Mode 1 sub-step 1d priority pass filters the workload to `due_date == today` and cross-references the activity log for AM-created / AM-reassigned events overnight. **Every** task with `due_date == today` (not just the AM-handed ones) also emits a `pm_task_due_today` signal in the normal pass so it surfaces in the queue even with zero overnight activity — the single-pane mandate means the PM never opens Orbit to find today's deadlines (see `collectors/orbit.md` § What to pull, item 6 + matcher Job 5 due-today branch). **User-centric universe — no per-project iteration.** Tasks the PM follows but is not assigned to are out of scope here (they surface via Gmail if relevant). See `collectors/orbit.md` § Universe model. | `mcp__...orbit.*` |
| **Gmail** | PM's `@whitelabeliq.com` inbox ONLY (single account, even if other accounts are authenticated). Aliases respected. Overnight window. Filtered for client/AM/team/leadership. Skips Orbit notification emails. Each signal emits context-link metadata (project_id_candidates, actor_emails, topic_keywords, timestamp) so matcher Job 4b can corroborate Orbit signals. | `mcp__...gmail.*` |
| **Fathom** *(enrichment-on-demand, not primary collection)* | Fetched lazily by `synthesis/matcher.md` Job 4b Pass 2 when a primary signal references a meeting (per trigger-phrase list in `collectors/fathom.md`). Returns scoped meeting summary excerpt, relevant action items, attendees, recording URL. Never originates a row; never scanned eagerly. Failure is non-blocking — primary signals still flow. | `mcp__...fathom.search_meetings`, `get_summary`, `get_transcript` only |
| **Notion** | Read Preferences. Write dated sub-pages, inline database, row detail pages. On 1st of month: move previous month into named toggle. ONLY the Notion parent page identified in `config.md`. **Read-only exception:** the Pod Matrix page identified by the runtime-injected `POD_MATRIX_URL` (Mode 1 routine prompt only — see `ROUTINE-ENTRYPOINTS.md`) is allowlisted for `notion-fetch` only; never written. Mode 2 and Monthly Archival do not receive `POD_MATRIX_URL` and do not read the Pod Matrix. | `mcp__...notion.*` |

Explicitly forbidden: Pipedrive, Apollo, Common Room, Hex, Calendar, Keka, Atlassian, Linear, Intercom, Figma, Klaviyo, Ahrefs, Canva, ClickUp, Monday, Fireflies, Pendo, Amplitude, Quickbooks, or any other connector. Google Drive / Docs / Sheets and SharePoint are allowed read-only on-demand only as defined in `references/external-doc-access.md` — never as primary collection sources.

## Non-negotiable rules

1. **Preflight runs first.** Every invocation, every code path. No exceptions.
2. **V3 stays sealed.** Do not edit any page under the V3 Notion space. See `references/v3-context.md` for what V3 is and why.
3. **Plain language only for India delivery team outputs.** Handoff drafts written for team members (rendered in the row Outcome block for the PM to copy) and Orbit task bodies use 4th–5th grade general English with role-specific technical terms preserved. PMs, AMs, and leadership get normal professional English. See `writers/plain-language.md`.
4. **Source citation on everything sourced from a document.** When the skill reads a PDF, image, PPT, or doc for context, it cites the filename in the output. See `writers/source-citation.md`.
5. **Nothing client-facing or team-facing is auto-sent by default.** Emails are drafts, with the documented send exceptions in `executors/email.md`: connector-failure tier 1 email to PM, and Mode 2 Step 3a escalation email to backup. Slack is outbound-send only for two explicit-opt-in paths: team handoff send (PM Note = `send` + audience = `team`) and AM ping send (PM Note = `send` + audience = `am`) — see `executors/slack.md`. Without those explicit notes, team and AM handoffs are drafted into today's dated Notion page under the row's Outcome — the PM copies and sends from there. The PM self-summary at end of Mode 2 is written as a Notion callout block at the top of today's dated page (no auto-send to PM).
6. **`get_user_workload` has TWO uses, distinguished by `user_id` target.** (a) **PM-workload universe discovery (always-on, this collector):** `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)` is the primary universe-discovery call in `collectors/orbit.md`. It runs on every Mode 1 fire — the PM's open task list IS the morning universe. Per-project iteration (`list_projects` + `get_project_task_list × N`) is retired; the single workload call replaces it. (b) **Candidate-availability scoring (lazy):** `get_user_workload(candidate_user_id, ...)` is invoked by `synthesis/pod-inference.md` ONLY on the matcher Job 6 no-history fallback path. When at least one role-fit candidate has prior task history on the project, familiarity wins (no availability check). When no role-fit candidate has history (brand-new project, or all role-fit pod members are new to the project), the matcher calls `get_user_workload` for the role-fit candidates from the running PM's matrix and picks the lightest-loaded. If the PM's matrix has no role-fit member, the matcher checks the Floater matrix; if Floaters also lack the role, it surfaces cross-matrix candidates as `Uncertain:`. The PM still overrides via note. See `collectors/orbit.md` § MANDATORY tool call sequence, `synthesis/matcher.md` Job 6, and `synthesis/pod-inference.md` Step 5. No Keka / leave data — that connector is forbidden.
7. **Mode 1 never asks the PM questions.** If unsure about an item, list it as its own row with an `Uncertain:` note in AI Notes. Never block waiting for input.
8. **PM notes are interpreted as natural language.** See `synthesis/note-interpreter.md`. Short notes like `assign to Vijay`, `save as draft`, `mark as high priority` must resolve correctly.
9. **Row-level approval is explicit per row.** No bulk-approve. No status sweep. Each row either gets flipped to `Approved`, gets a note, gets marked `Skip. No Action Needed`, or stays at `Recommended Action` (= no action).
10. **Page-level approval is the `Ready for Execution` toggle at the TOP of each dated page.** Mode 2 only executes if the toggle is ON.
11. **All Orbit task bodies follow the 6-section standard.** Sections in order: DO · WHY · CONTEXT · DONE WHEN · SELF-QA · REFS. Header style defaults to bold professional headers; PM may opt into `emoji` style via Preferences `orbit_task_header_style`. See `schemas/orbit-dq-standard.md`.
12. **Source allowlist is closed.** See top of file.
13. **Notion parent is hardcoded in `config.md`.** No discovery, no asking the PM where to write. The page is fixed per installation.
14. **Connector failures notify the PM via the 3-tier fallback chain** in `connector-failure-notify.md`: email-sent → Notion red callout on today's dated page → Incidents sub-page. Repeat-failure escalation routes via email to the configured backup. Never silently fail.
15. **Email aliases are respected.** Identity matching across all collectors and executors uses the canonical email plus any aliases stored in Preferences.
16. **Every routine fire writes a run-log entry** to the Run Log database on the Notion parent. Brief reason traces (one line per decision: subject → action → reason). See `writers/run-log.md` and `schemas/run-log-database.md`.
17. **All MCP calls retry with backoff before failing.** Every MCP tool call (any of the 3 primary collection MCPs — Orbit, Gmail, Notion — plus the Fathom enrichment MCP, plus the Slack outbound-send MCP) wraps in the retry policy defined in `connector-failure-notify.md`: 4 attempts total (1 + 3 retries), 2s/5s/15s incremental backoff, retry only on transient errors (timeout, 5xx, 429, connection reset). Permanent errors (4xx auth, 404, validation) skip retry. After exhaustion, the failure chain fires for primary MCPs; Fathom exhaustion is non-blocking (logged to Incidents page only, no PM callout). The run-log records all retry attempts.

18. **The Morning Queue drives exactly five AI actions, picked per row by `synthesis/matcher.md` Job 5 + Job 5a (work-type classifier) + Job 5.5 (existing-subtask check):**
    - **`Create subtask`** — net-new pod-resource work (work_type ∈ HTML_CSS, PHP_BACKEND, QA) landing under a PM-owned parent task. Carries explicit sub-task title + 6-section brief.
    - **`Reopen subtask`** — an existing open subtask of the same `work_type` already sits under the parent; instead of creating a duplicate, the executor flips status, appends a comment with the new work, and reassigns to the last non-PM developer who worked it (extracted from `list_task_comments`). When that last dev is inactive in Orbit, the row drops to `Uncertain:` for PM to pick.
    - **`Hand off parent task`** — work that does NOT touch HTML/PHP/QA pod resources (work_type ∈ QUOTE, SEO, AUDIT, DESIGN, CONTENT, BA). The executor reassigns the existing PM-owned parent directly to the respective pod leader; no subtask created. PM stays on as internal follower.
    - **`Flag`** — PM-attention signal with no auto-execution. PM owns the next move. Also the resting verb for a **due-today** task with no actionable overnight signal (`pm_task_due_today`, anchored on the `orbit_due_today` latest_signal) — it surfaces as a Flag (`Due today — review/delegate/close`) so the deadline is visible in the queue without opening Orbit. These bare due-today Flags fire a **light deep-read** (task details + newest comments only) to compose a "here's where this stands" task_brief + open-subtask note — but carry NO assignee recommendation and NO handoff (the skill never invents work for a task with no new ask). If the PM decides delegation IS needed, a PM note (`delegate to <dev>` / `hand to <pod>`) **promotes** the Flag to `Create subtask` / `Reopen subtask` / `Hand off parent task` in Mode 2 — the one due-today verb-change exception (upgrades to full deep-read + work_type + assignee + body at promotion time; see `synthesis/note-interpreter.md` § Delegate a due-today Flag). If a due-today task ALSO carries an actionable signal, that verb wins and `due_today` is annotated, not double-emitted (one task → one row; Job 5 due-today branch).
    - **`Create parent task`** — Possible Orbit miss; a client email that should have become an Orbit task but did not, with NO corroborating Orbit task and no PM action. Three trigger sub-classes (any one fires): **S1a** client reply delivering something WLIQ asked for, **S1b** delivery-token mail ("here is / as requested / attached"), **S2** issue / error / feature-request mail (critical-language). Project is resolved from the PM workload map first, then by lazy Orbit search (`list_clients` / `list_projects` / `get_client_details`); a topic-matching open task on the resolved project suppresses the create (dedup, `get_project_task_list`). Project found (any owner) → propose `Create parent task` for PM approval; project not found OR duplicate-suspected → `Flag` the PM, who names the project / says "create anyway" and Mode 2 creates it. On PM approval, Mode 2 creates the parent on the resolved project assigned to the PM. See `synthesis/matcher.md` § "Unactioned client signal → Create parent task".

    Every other class of signal is either dropped (already handled by the PM overnight, **overdue** or bare status-change visible in Orbit's own UI, hours-overrun alert, third-party automation) or downgraded to Flag. Note: due-today tasks are NO LONGER in the drop set — they surface per the Flag verb above. Only `due_date < today` (overdue) and passive status changes remain dropped. Priority-lane rows (AM handed parent to PM overnight, due today — Mode 1 sub-step 1d) follow the same Job 5 → 5a → 5.5 tree as any other Orbit signal; parent_task_id is pinned to the AM-assigned parent. See `synthesis/matcher.md` Output gating + Job 4 locked-verb table + Job 5 + Job 5a + Job 5.5 + Job 6 pod-boundary table.

    **Role/pod boundaries (Job 5a + Job 6):** HTML/CSS → FE pod only (matrix `HTML`); PHP backend → WP pod only (matrix `WordPress / PHP`); QA → QA matrix; Audit (including AI audit, SEO audit) → Marketing/SEO pool; Quote/SEO/Design/Content/BA → their named functional pools. Routing is pool-based, not person-based — Pod Matrix resolves the assignee.

19. **PM-action detection — never re-flag a signal the PM already handled overnight.** Before deciding Flag vs Drop, the matcher checks whether the PM took action on the signal between its arrival timestamp and Mode 1 fire (PM-sent email on the thread, PM-authored Orbit comment). If PM-action exists, the signal drops with `filter_reason: pm_already_handled`. This prevents the over-flagging that produced 13-row queues from PM-handled work. **Exception:** signals with `bypass_pm_action_filter == true` (set by the Orbit-first priority pass on AM-handed-to-PM tasks) are never dropped by this filter. Rationale: an AM-handed task is pending delegation even if the PM acknowledged it overnight. See `synthesis/matcher.md` PM-action detection.

20. **An empty Morning Queue is a correct, expected outcome.** Many mornings produce zero rows because all overnight signals reduced to PM-personal work, AM-led threads, or status reviews that the PM scans in Orbit directly. The matcher MUST NOT invent rows to fill the queue. The summary line `0 items for your morning. 0 sub-tasks, 0 flags. <N signals filtered — see Run Log if you want to audit>.` is a valid, healthy run. Do not lower the action bar to produce a non-zero count.

21. **Row detail page is action-first AND brief-before-sources.** A Notion callout block sits at the very top of every row's detail page with the proposed action (task title + assignee for Create subtask / Reopen subtask, pod leader for Hand off parent task, PM next step for Flag, proposed parent title for Create parent task). Directly below the H1 Summary heading comes an H1 `Task Brief` heading — what the work is about + the new update / latest signal content that triggered this row. H1 `Sources` sits below the brief, not directly under Summary. The PM must never have to scroll past Sources to find the actionable proposal OR the new context. See `schemas/row-detail-page.md` Action Block + Task Brief sections.

22. **Project identifiers user-facing → use Orbit's `project_number`, never `id` or `url_slug`.** Orbit's `get_project_details` returns three project identifiers: `id` (internal numeric PK, e.g. `8598`), `url_slug` (URL-encoded id, e.g. `"51083298598"`), and `project_number` (the user-visible code shown in the Orbit UI, e.g. `"16915"`). Every user-facing rendering of a project — Notion `Project` column, Summary text, Sources block citations, Slack drafts, handoff drafts — uses `project_number` rendered as `#<project_number>` (with the leading hash). Internal `id` and `url_slug` are NEVER user-facing; they only appear inside URLs (where Orbit needs them) and inside the relationship map (where downstream tooling reads them). Format: `<Project Name> (#<project_number>)` for the Project column; `Process Barron Change Order #16915` for inline Summary mentions.

23. **AM-framing rule — AMs are client-relay, never teammates.** Every text the skill renders that mentions an AM (Summary, Task Brief, Recommended Action, Outcome handoff bodies, PM Next Step, AI Notes, AM Ping Drafts) frames the AM as the client-relay channel — never as an internal teammate, never as the developer-picker, never as the scope-approver, never as the delivery-owner. See `references/am-context.md` for the canonical role description and the right-vs-wrong framing table. The 1 PM ↔ 1 AM cardinality is locked: Preferences carries the single AM. Existing priority-lane phrasings (`<AM> put this on your plate overnight, due today`) already comply and stay verbatim.

24. **Latest-signal anchor required on every row.** Every queue row MUST carry `latest_signal_anchor` — the single most-recent signal across all deep-read sources that triggered the row (source, id, timestamp, author, excerpt, source link). The collector emits a per-task / per-thread `latest_signal` field; the matcher selects the newest across sources as the row's anchor and passes it through; the writer renders it as the first line of the row's H1 Task Brief block (`**Triggered by:** ...`). The render requirement IS the gate — a row with a null anchor cannot render. Structural defense against silent missed-comment failures: collector comment lists are sorted newest-first so `comments[0]` is always the trigger by construction, and the required-field flow makes "matcher silently skipped the newest comment" impossible without the row visibly failing to render its Triggered-by line.

25. **Off-workload project must be resolved before any create — never create on a guessed project.** For a Possible-Orbit-miss `Create parent task`, when the email's project is NOT in the PM's workload map the matcher resolves it by lazy Orbit search (`list_clients` / `list_projects` / `get_client_details`). If search returns exactly one clean match, propose the create (any project owner — PM still approves in Mode 2). If search returns zero or an ambiguous set, the matcher emits a `Flag` informing the PM, NOT a `Create parent task` — the PM names the project and Mode 2 creates it via `synthesis/note-interpreter.md`. Before any create the matcher runs the dedup check (`get_project_task_list(project_id, search=…, is_completed="incomplete")`); a topic-matching open task suppresses the create and Flags it instead. The only write (`create_task`) is `ask`-gated and runs after PM approval. See `synthesis/matcher.md` § "Unactioned client signal → Create parent task".

## File map

Everything below this file provides the detailed behavior. Load the specific file when that behavior is needed.

- `ENVIRONMENTS.md` — the two execution surfaces (headless routine/code vs interactive Claude Desktop) and how the skill adapts.
- `config.md` — hardcoded Notion parent + operational rules. Loaded FIRST every invocation.
- `preflight.md` — the 6-step preflight sequence. Runs after config, before any mode logic.
- `connector-failure-notify.md` — 3-tier failure fallback chain (email → Notion callout → Incidents page).
- `first-run-setup.md` — operator-facing per-PM Preferences-page setup. Use this before the first routine fire (or for any new PM rollout).
- `invocation-commands.md` — exact interactive command syntax (`PM Task Assignment, run morning` / `run execution now` / `validate setup`) and lenient routing rules. Used by Claude Desktop sessions; routines never reach this routing.
- `setup-template.md` — operator-facing one-time per-PM Preferences template. Reference companion to `first-run-setup.md`.
- `ROUTINE-ENTRYPOINTS.md` — the 3 baked routine prompts (Mode 1 morning collection, Mode 2 execution, monthly archival). Each routine fires its prompt by cron and loads this skill from the public GitHub repo.
- `schemas/run-log-database.md` — Run Log database schema (one row per routine fire) on the Notion parent.
- `schemas/run-log-detail-page.md` — per-row detail page layout with the decision trace (subject → action → reason lines).
- `writers/run-log.md` — appends a Run Log entry (and detail page) on every routine fire.
- `DEPLOYMENT.md` — how to deploy this skill to a new PM (edit config, ship, install).
- `modes/mode-1-morning-collection.md` — Mode 1 end-to-end orchestration.
- `modes/mode-2-execution.md` — Mode 2 end-to-end orchestration.
- `modes/monthly-archival.md` — 1st-of-month archival at 6:00 AM IST.
- `collectors/orbit.md`, `gmail.md` — primary data collection (called by Mode 1 Step 3a/3b). `collectors/orbit.md` documents both the normal pass and the Mode 1 Step 3a priority pass (AM-handed-to-PM detection).
- `collectors/fathom.md` — **enrichment service** (not a primary collector). Provides `fetch_enrichment(reference, signal_context)` interface called lazily by `synthesis/matcher.md` Job 4b Pass 2 when a primary signal references a meeting.
- `synthesis/matcher.md` — signal grouping, Job 4b Pass 1 (Gmail → Orbit context cross-link) + Pass 2 (Fathom enrichment fetch), and summary generation, alias-aware.
- `synthesis/pod-inference.md` — how to compute pod per project from Orbit.
- `synthesis/note-interpreter.md` — natural-language PM note resolution.
- `executors/orbit.md`, `email.md`, `slack.md` — per-action-type write operations. `slack.md` is outbound-send only (team-handoff + AM-ping with explicit PM `send` note).
- `writers/notion.md` — Notion page and database creation. Enforces parent-page structure.
- `writers/plain-language.md` — language-splitter rules.
- `writers/source-citation.md` — citation formats.
- `schemas/parent-page.md` — parent Notion page structure: header callout + Year/Month heading-toggle blocks on parent body containing dated sub-page links + Run Log + Incidents + Preferences last. Day pages stay as sub-pages (Notion-tree parent = parent page); Year/Month are toggle blocks, not sub-pages.
- `schemas/preferences-page.md` — Preferences layout. AM identities (canonical email + aliases) drive the Mode 1 Step 3a priority pass.
- `schemas/morning-queue-database.md` — inline database schema (10 columns all visible: Summary, AI Notes, Orbit Task Link, Project, Recommended Action, Recommended Assignee, Outcome, PM Notes, Source Systems, Status; 4 Status options, 3 Source Systems options).
- `schemas/row-detail-page.md` — row detail page layout (headings + reference toggle at bottom).
- `schemas/orbit-dq-standard.md` — 6-section Orbit task body template.
- `references/due-date-categories.md` — PM note → category ID mapping.
- `references/status-values.md` — Status enum semantics.
- `references/pod-matrix.md` — read-only Pod Matrix (Notion) parsing, name → Orbit user_id resolution, fallback when matrix unavailable.
- `references/am-context.md` — Account Manager role description + sentence framing (right-vs-wrong examples). Read once per session so AM mentions in Summary / Task Brief / Recommended Action / handoff bodies frame the AM as client-relay, not as an internal teammate.

## Build status

**Version:** 1.1.0 (post-test-run rewrite)
**Built for:** White Label IQ Project Managers
**First installer:** Aditi Singh (subject to change based on availability)
**Distribution:** Editable per-PM via `config.md`. Ship the folder, the receiving PM swaps the Notion page ID, installs on their own Claude account.
**Parallel run expected:** 4–6 weeks alongside the PM's manual morning hour, with accuracy compared before full rollout.

## Source of truth for design decisions

All design decisions that shape this skill are locked in the MVP project workspace in Notion:
[PM Automation MVP 1](https://www.notion.so/3493846840c8805b9424f63dcdc9b9ce)

Never re-invent rules. If a behavior is underspecified in these files, consult the Notion workspace decisions and open items lists.
