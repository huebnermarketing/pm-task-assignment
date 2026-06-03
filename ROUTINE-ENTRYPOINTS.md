# Routine Entrypoints

This file holds the three exact prompts to paste into Claude Routines when deploying the PM Task Assignment skill for a new PM. Each routine is a cron-fired headless agent that loads the skill from a public GitHub repo OR from a local skill directory (whichever is available — see `ENVIRONMENTS.md`) and executes a single mode end-to-end. No mode performs any work outside its own scope.

Manual interactive runs (Claude Desktop) use the same skill files — see `invocation-commands.md` for the commands.

> Three routines, three modes. Routine 1 collects, Routine 2 executes, Routine 3 archives. Nothing else.

---

## Pre-deploy Checklist

Before creating any routine, the operator must have:

- [ ] **PM's Notion parent page ID** — the 32-character UUID (or the share-link short ID) of the PM's parent page that holds Preferences + dated queue pages + Run Log database.
- [ ] **PM's Preferences page URL** — the public Notion URL of the Preferences page on that parent.
- [ ] **MCP authentication confirmed** for the PM's Claude account — 3 primary collection MCPs + Fathom enrichment MCP + Slack outbound-send (5 connectors total) connected and authenticated. The skill's source allowlist (per `SKILL.md` and `config.md`) is closed; no routine prompt may add to it or remove from it:
  - [ ] Orbit (primary collection + execution)
  - [ ] Gmail (primary collection + sent-email exceptions)
  - [ ] Notion (parent page read/write)
  - [ ] Fathom (enrichment-on-demand — lazy fetch by matcher Job 4b only. Non-blocking on failure.)
  - [ ] Slack (outbound-send ONLY — team-handoff + AM-ping with explicit PM `send` note. No collection.)
- [ ] **Repo URL** of the public GitHub repo holding the skill — referred to below as `<REPO_URL>` (raw-content base, e.g. `https://raw.githubusercontent.com/<org>/<repo>/main`).
- [ ] **Pod Matrix Notion page URL** — the read-only org-wide page listing PM-owned and functional matrices (e.g., the public `Matrix Detail` share URL). Used by Mode 1 only for assignee recommendation. Mode 2 and Monthly Archival do not need it.
- [ ] **Routine TZ — must be IST.** The skill is timezone-locked to **Asia/Kolkata (Indian Standard Time, UTC+05:30)**. All cron expressions, all Preferences-time fields, all Run Log timestamps, and every "today / yesterday / previous-month" calculation in this skill are computed in IST — never UTC, never GMT, never the routine runner's local TZ. If the Routines runner does not natively interpret cron in IST, convert each cron line below from IST to the runner's TZ at routine-creation time so the wall-clock IST fire time stays the same. The skill itself does NOT auto-convert TZ at runtime; the routine runner is responsible for delivering an IST wall-clock fire.

If any item is missing, do not create routines. Stop and resolve first.

---

## Routine 1 — Mode 1 (Morning Collection)

Purpose: pull overnight signals from Orbit and Gmail in two passes — Orbit-first priority pass (AM-handed parent tasks due today) followed by parallel Orbit-normal + Gmail — write a dated queue page on the Notion parent, await PM approval (PM acts in their own time, not in this routine). Fathom is enrichment-only: the matcher lazily fetches meeting context during synthesis when a primary signal references a call.

- **Cron (default):** `30 9 * * *` (09:30 IST daily)
- **Override:** if Preferences specifies a different morning run time, use that — still in IST.
- **Connectors required:** Orbit, Gmail, Notion (3 primary collection MCPs) + Fathom (enrichment-on-demand, non-blocking on failure). Slack is not used by Mode 1 — Slack is outbound-send only and only invoked by Mode 2.

### Prompt template

```
You are running the PM Task Assignment skill in Mode 1 (Morning Collection) as a headless scheduled routine. No human is in the loop — do not ask questions, do not wait for confirmation, decide and act per the skill files.

Strict-skill rule: this prompt does NOT override skill behavior. Where this prompt and any skill file disagree, the skill file wins. Allowlist, retry policy, plain-language rules, source-citation, parent-page hierarchy (Year → Month → Date), and the 6-section Orbit task body are all defined in the skill files — follow them as written.

Time zone: the skill is timezone-locked to Asia/Kolkata (Indian Standard Time, UTC+05:30). Every "today / yesterday / overnight window / Preferences run-time" calculation MUST be performed in IST. Do not use UTC, GMT, or the runner's local TZ for any date math, page title, log timestamp, or window boundary. Page titles use IST date in "DD Month YYYY" format (e.g., "30 April 2026"); Run Log "Started"/"Finished" timestamps are recorded in IST with the timezone offset suffix.

Load and follow these files in order. Prefer local files when present (Claude Code / SDK invocations); otherwise fetch from the public skill repo (Claude Routines fired by cron):
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/config.md
3. <REPO_URL>/preflight.md
4. <REPO_URL>/modes/mode-1-morning-collection.md

Per-PM injected values:
NOTION_PARENT_PAGE_URL=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>
POD_MATRIX_URL=<INJECTED_VALUE>          # Read-only Notion page with the org's pod/matrix structure. Mode 1 only.

Connectors available in this routine: Orbit, Gmail, Notion (3 primary collection MCPs) + Fathom (enrichment-on-demand, lazy-called by synthesis/matcher.md Job 4b Pass 2 only — never eagerly listed). Do NOT skip any primary collector. Do NOT use any MCP outside this list, even if it appears authenticated. Slack is forbidden in Mode 1 — it is outbound-send only and reserved for Mode 2's team-handoff + AM-ping paths with explicit PM `send` note. Fathom MCP failure is non-blocking — log to Incidents and continue.

Notion read-only exception: Notion access is normally restricted to NOTION_PARENT_PAGE_URL (read + write). For this Mode 1 routine ONLY, you may also notion-fetch POD_MATRIX_URL (read-only). Do NOT write to it. Do NOT enumerate or search around it.

Mental model for this run — read before doing anything else:

The Morning Queue is **NOT a daily digest**. It is the input to three specific AI actions and only three:
(1) **Create subtask** — universal across every project type (including Ad-hoc and Maintenance). Net-new scoped work landing under a parent task currently assigned to the running PM. Carries explicit task title + 6-section brief.
(2) **Flag** — overnight signal that needs PM attention but cannot be auto-delegated (PM owns the next move). No Orbit execution; the row sits as a PM-readable item.
(3) **Create parent task** — an unactioned client email with NO corroborating Orbit task (Possible Orbit miss): a client reply delivering something WLIQ asked for (S1a), delivery-token mail (S1b), or issue/error/feature-request mail (S2). On PM approval, Mode 2 creates a parent Orbit task on the resolved project assigned to the PM, so the PM can spawn sub-tasks under it later. Detection in matcher Job 5 § Unactioned client signal → Create parent task; project is resolved from the PM workload map or by lazy Orbit search (any owner) and dedup-checked — when search can't find the project, or a topic-matching open task already exists, it downgrades to Flag for the PM to resolve.

The skill also runs in four conceptual steps that match how the PM thinks about the morning: (1) Pull from Orbit, (2) Pull from Mail, (3) Enrich with Fathom (lazy, on-demand) — with Gmail-attachment fallback when Fathom is unavailable, (4) Synthesize, recommend assignees, write to Notion. See `modes/mode-1-morning-collection.md` § End-to-end flow for the lettered sub-step decomposition.

**Per-row brief composition is FOUR-source mandatory and default-on, NOT conditional on "the brief looks thin".** For every Create-subtask and Create-parent-task row, before composing the proposed 6-section Orbit body, the matcher MUST read all four input sources in full: (1) the originating Orbit task itself — lazy per-row `get_task_details(task_id)` + `list_task_comments(task_id)` with **complete all-time comment history (NOT date-filtered to last_run_timestamp)** because older comments routinely hold prior decisions / failed attempts / scope changes; (2) every cross-linked Gmail thread end-to-end via the attached `context_signals[]` from Job 4b Pass 1; (3) every Fathom enrichment from Pass 2; (4) any external doc referenced by the above. The workload snapshot + activity_log delta alone are rarely complete enough — the formal task body and the overnight change deltas both routinely miss months of prior conversation. The bias is over-include context, not under-include. See `synthesis/matcher.md` Job 7 § Mandatory deep-read of the originating Orbit task + § Mandatory email-thread enrichment + `schemas/orbit-dq-standard.md` § Source rule for the per-section mapping.

Apply two filters before deciding action:

A. **PM-action detection.** For every signal, check whether the PM already acted on it between signal arrival and Mode 1 fire (PM-sent email on the same thread, PM-authored Orbit comment on the same task). If PM-action exists, the signal drops with `filter_reason: pm_already_handled`. Do NOT re-surface signals the PM has already resolved. **Exception:** priority-lane signals (`signal_type: am_handed_to_pm_overnight_due_today`, set by the Orbit collector's 1d Priority Pass) carry `bypass_pm_action_filter: true` and are NEVER dropped by this rule — an AM-handed task is pending delegation even if the PM acknowledged it overnight.

B. **Static drop list.** Hours-overrun alerts, PM's own tasks already visible in Orbit's due-date UI, rollup digests, third-party automation emails, marketing/system noise — all drop. Per SKILL.md non-negotiable rules #18–#20 and `synthesis/matcher.md` Output gating.

An empty queue is a correct outcome. The matcher MUST NOT invent rows to fill space. Summary line "0 items for your morning. 0 sub-tasks, 0 flags. <N signals filtered — see Run Log>." is healthy output. Do NOT lower the action bar.

Row detail page is action-first: the top of every row's detail page is a callout with the proposed task title + assignee (Create subtask / Create parent task) or the PM next step (Flag). Sources move below. Per SKILL.md non-negotiable #21 and `schemas/row-detail-page.md`.

Mandatory Orbit collector tool sequence per `collectors/orbit.md`: TWO non-skippable calls on every run — **(a) `get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)`** is the universe-discovery call (single API call returning every open task assigned to the PM with details + summary counts; replaces per-project iteration of `list_projects` + `get_project_task_list × N`), and **(b) `get_activity_log` with `from_date = last_run_timestamp`** is the change-detection call (every comment, status flip, assignment change, due-date move since last run). The Mode 1 sub-step 1f post-collector assertion will abort the run if EITHER call is missing. `get_user_workload` has a second non-collector use — `get_user_workload(candidate_user_id, ...)` is invoked lazily by `synthesis/pod-inference.md` ONLY on the matcher Job 6 no-history fallback path for candidate-availability scoring. See SKILL.md non-negotiable rule #6 for the distinction.

Execute Mode 1 end-to-end:
- Run preflight Steps 1–6 against the Notion parent and Preferences page (all in IST).
- Fetch and parse POD_MATRIX_URL once at the start of the run via references/pod-matrix.md; cache the parsed pools (PM matrix, floaters, functional matrices) for the whole run. On fetch or parse failure, log a warning to the Run Log detail page and continue with Orbit-only pod inference (graceful degradation per references/pod-matrix.md).
- Collect overnight signals via fan-out launch: after 1b (lookback determination), launch **1c (Pod Matrix) + 1e (Orbit MCP fetch: get_user_workload + get_activity_log + list_users) + 2a (Gmail collector)** all in parallel — they have no inter-dependencies at launch time. 1d (priority pass) is a LOCAL filter + cross-reference computation over 1e's MCP responses (zero new MCP calls of its own); it runs synchronously as soon as 1e responses land. 2a does NOT wait for 1d — Gmail has no Orbit dependency, and the priority-signals-before-matcher invariant is enforced at the matcher Job 1 entry-gate (consumer side), not by serializing collector launches. Window boundary in IST. Fathom is NOT called in collection; it is lazy-fetched by the matcher in Step 3 (Job 4b Pass 2) when a primary signal references a meeting, with a Gmail-attachment fallback (gmail.find_transcript_in_email) before treating Fathom's null as terminal. Orbit collector MUST invoke **`get_user_workload(PM_user_id, is_completed=incomplete, per_page=500)` AND `get_activity_log` with `from_date = last_run_timestamp`** per the mandatory sequence above — both calls are fetched once in 1e and reused by 1d's local filter. The Mode 1 sub-step 1f assertion is the explicit join point for the {1c, 1d, 1e, 2a} fan-out and will abort the run if either Orbit call is missing.
- Run Mode 1 sub-step 1f post-collector assertion before synthesis. Abort if `get_activity_log` call count is zero.
- Synthesize per synthesis/matcher.md: apply the Output gating filter first (PM-action detection drops `pm_already_handled`; static drop list removes hours-overrun / Orbit-UI-visible / rollup / third-party noise; survivors reduce to one of three locked actions — `Create subtask`, `Flag`, or `Create parent task`), then run Jobs 1–11 on what remains. Possible-Orbit-miss detection runs in Job 5: critical-language Gmail signals with no Orbit corroboration and unambiguous project resolution become `Create parent task` rows tagged `Possible Orbit miss:` in AI Notes. Recommend assignees per Job 6 (history wins → matrix availability → floater availability → cross-matrix Uncertain). Availability (get_user_workload) fires only on the no-history fallback path. Create parent task rows short-circuit Job 6 to PM-assignee (no pod inference).
- Write today's dated queue page on the Notion parent per writers/notion.md, placed at Parent → <Year> → <Month> → <DD Month YYYY> per schemas/parent-page.md (the writer creates the Year and Month container sub-pages on demand if missing). The writer applies a Step 5.5 defense-in-depth check that re-runs the Output gating filter and pushes any drift rows to filtered_signals.
- Append a Run Log row + linked detail page via writers/run-log.md, including the `Filtered signals (N)` toggle when the matcher emitted any filtered entries.

Apply the retry policy in connector-failure-notify.md to every MCP call: 4 attempts total (1 + 3 retries) with 2s/5s/15s incremental backoff. Retry only on transient errors (timeout, 5xx, 429, connection reset). Permanent errors (4xx auth, 404, validation) skip retry and fall through to the failure chain. Log every retry attempt to the Run Log detail page.

Do NOT run Mode 2. Do NOT run Monthly Archival. Do NOT execute any task assignments — Mode 1 only writes the queue; the PM approves later; Mode 2 executes.

Exit when the queue page is written and the Run Log row + detail page exist.
```

---

## Routine 2 — Mode 2 (Execution)

Purpose: read PM-approved rows from today's dated queue, execute the corresponding Orbit + Notion-draft actions (and Slack send only when the PM left an explicit `send` note + audience match), write a completion-summary callout at the top of today's dated page.

- **Cron (default):** `45 10 * * *` (10:45 IST daily)
- **Override:** if Preferences specifies a different execution run time, use that — still in IST.
- **Connectors required:** Orbit (subtask create + updates), Gmail (sent-email exceptions only — escalation + connector-failure tier 1), Notion (parent page read/write), Slack (outbound-send only for team-handoff + AM-ping with explicit PM `send` note). Fathom is NOT used by Mode 2.

### Prompt template

```
You are running the PM Task Assignment skill in Mode 2 (Execution) as a headless scheduled routine. No human is in the loop — do not ask questions, do not wait for confirmation, decide and act per the skill files.

Strict-skill rule: this prompt does NOT override skill behavior. Where this prompt and any skill file disagree, the skill file wins. Allowlist, retry policy, plain-language rules, source-citation, the 6-section Orbit task body, and approval semantics (row Status + page-level Ready toggle) are all defined in the skill files — follow them as written.

Time zone: the skill is timezone-locked to Asia/Kolkata (Indian Standard Time, UTC+05:30). "Today's queue page" is resolved by IST date. Run Log "Started"/"Finished" timestamps are in IST with offset suffix. Do not use UTC / GMT / runner-local TZ for any date math.

Load and follow these files in order. Prefer local files when present (Claude Code / SDK invocations); otherwise fetch from the public skill repo (Claude Routines fired by cron):
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/config.md
3. <REPO_URL>/preflight.md
4. <REPO_URL>/modes/mode-2-execution.md

Per-PM injected values:
NOTION_PARENT_PAGE_URL=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Connectors available in this routine: Orbit, Gmail (operational sent-email only), Notion, Slack (outbound-send only — team-handoff + AM-ping with explicit PM `send` note). Do NOT use any MCP outside this list. Do NOT pull collection signals from Slack — it is outbound only.

Execute Mode 2 end-to-end:
- Run preflight Steps 1–6 against the Notion parent and Preferences page.
- Resolve today's dated queue page via Parent → <Year> → <Month> → <DD Month YYYY> (IST). If the page-level Ready toggle is OFF, fire the inline escalation flow defined in modes/mode-2-execution.md Step 3a (email-only to the configured backup) and exit.
- If Ready is ON: read every row's Status and PM Notes. Resolve note intent via synthesis/note-interpreter.md. Execute Approved rows and rows with PM notes via the executors (executors/orbit.md for sub-task creation; executors/email.md for the 3 documented sent-email exceptions; executors/slack.md ONLY when a row carries an explicit PM `send` note + audience matches team or am).
- Write each row's Outcome + Status back via writers/notion.md. Default handoff path is a Notion draft block appended under the row's Outcome (PM copies + sends). Slack send replaces the Notion draft only when the executor was invoked per the above rule. Apply writers/plain-language.md and writers/source-citation.md to all team-facing outputs.
- Write a Mode 2 completion-summary callout block at the TOP of today's dated page (replaces the old Slack DM — PM is already opening Notion to review).
- Append a Run Log row + linked detail page via writers/run-log.md.

Apply the retry policy in connector-failure-notify.md to every MCP call: 4 attempts total (1 + 3 retries) with 2s/5s/15s incremental backoff. Retry only on transient errors. Log every retry attempt to the Run Log detail page.

Do NOT run Mode 1. Do NOT run Monthly Archival. Do NOT collect new signals — Mode 2 only acts on rows already written by Mode 1.

Exit when execution finishes and the Run Log row + detail page exist.
```

---

## Routine 3 — Monthly Archival

Purpose: on the 1st of each month, verify the `Parent → Year → Month → Date` hierarchy on the Notion parent page is intact for the previous month, surface any drift in the Run Log, and reorder Year/Month/Date children as needed.

- **Cron (default):** `0 6 1 * *` (06:00 IST on the 1st of each month, in IST wall-clock)
- **TZ:** IST only. The "previous month" is computed by IST date — never UTC/GMT — so a fire on `1 May 2026 06:00 IST` archives April 2026 regardless of the runner's local TZ.
- **Connectors required:** Notion. (No collectors, no executors needed.)

### Prompt template

```
You are running the PM Task Assignment skill's Monthly Archival routine as a headless scheduled run. No human is in the loop — do not ask questions, do not wait for confirmation, decide and act per the skill files.

Strict-skill rule: this prompt does NOT override skill behavior. Hierarchy invariants (Parent → Year → Month → Date), drift handling (surface, don't auto-move), and ordering rules are defined in schemas/parent-page.md and modes/monthly-archival.md — follow them as written.

Time zone: the skill is timezone-locked to Asia/Kolkata (Indian Standard Time, UTC+05:30). Compute "previous month" in IST. Run Log timestamps in IST with offset suffix. Do not use UTC / GMT / runner-local TZ for the month-boundary calculation.

Load and follow these files in order. Prefer local files when present (Claude Code / SDK invocations); otherwise fetch from the public skill repo (Claude Routines fired by cron):
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/config.md
3. <REPO_URL>/preflight.md
4. <REPO_URL>/modes/monthly-archival.md

Per-PM injected values:
NOTION_PARENT_PAGE_URL=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Connectors used in this routine: Notion only. (Other allowlisted MCPs may be authenticated but are NOT called by archival.)

Execute Monthly Archival end-to-end:
- Run preflight against the parent page (Preferences verify-only).
- Verify the previous-month container exists at Parent → <Year> → <Month> (computed in IST, e.g., a 2026-05-01 fire targets 2026 → April).
- Sweep for misplaced dated pages from the previous month: at the parent level, under a Year (skipping Month), in a wrong Month, or under any legacy `[Month YYYY]` toggle block. For each, log a drift entry in the Run Log — do NOT auto-move.
- Reorder Year / Month / Date children to descending where drifted.
- Verify Run Log + Incidents + Preferences are the last three children of the parent, in that order.
- Append a Run Log row + linked detail page via writers/run-log.md, including every drift item surfaced.

Apply the retry policy in connector-failure-notify.md to every Notion MCP call: 4 attempts total (1 + 3 retries) with 2s/5s/15s incremental backoff. Retry only on transient errors. Log retry attempts.

Do NOT run Mode 1. Do NOT run Mode 2. Do NOT collect or execute anything. Do NOT auto-migrate legacy toggle blocks — surface them in the Run Log only.

Exit when archival verification and the Run Log row + detail page are complete.
```

---

## Validation Step

After all 3 routines are created in Claude Routines:

1. Open a regular (non-routine) Claude session with the same MCPs authenticated.
2. Run the manual `validate setup` command per `invocation-commands.md`, pointing at the same Preferences page URL.
3. Confirm preflight passes — parent page reachable, Preferences page parsed cleanly, 3 primary collection MCPs respond (Orbit, Gmail, Notion), Fathom enrichment MCP responds (or logs warning and continues — non-blocking), Slack outbound responds, Run Log + Incidents + Preferences present in the correct order at the bottom of the parent (or that preflight Step 6 created them on first run).
4. If `validate setup` reports errors, fix before relying on the routines.

Optional: trigger Routine 1 manually once via the Claude Routines UI ("run now") and inspect the resulting queue page + Run Log row to confirm shape.

---

## Editing the Schedule Later

A PM may want to change their morning or execution run time after deployment.

- **PM action:** edit the run-time field in their Preferences page on Notion.
- **Operator action (also required):** edit the cron line for the corresponding routine in the Claude Routines UI.

The skill, while running inside a routine, **cannot** modify the routine's cron schedule from the inside. Updating Preferences alone is not enough. Both edits are needed for the schedule to actually shift.

If the operator forgets to update the cron, Mode 1 / Mode 2 will continue firing at the old time; the skill will detect the mismatch on its next preflight (Preferences-time vs. actual fire-time delta) and emit a warning into the Run Log detail page, but it will still execute the run.
