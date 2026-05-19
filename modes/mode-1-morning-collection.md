> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes routine triggers — preflight runs even when invoked by a scheduled cloud routine.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Mode 1 — Morning Collection Run

## When this runs

- Scheduled: daily at the PM's configured morning time (default 9:30 AM IST), fired by a Claude Routine.
- Manual: `PM Task Assignment, run morning`.

## PM input during this run — surface-dependent

Mode 1's question behavior depends on the `is_interactive` flag set during preflight (see `ENVIRONMENTS.md`).

- **Headless surface (`is_interactive = false`):** never asks questions. Never blocks waiting for input. Never confirms with the PM. Routines fire unattended — there is no human in the loop at fire time. If something is uncertain, it becomes its own row with an `Uncertain:` note in AI Notes. The PM resolves it when they review.
- **Interactive surface (`is_interactive = true`):** may ask **one** clarifying question per ambiguous item before falling back to `Uncertain:`. Use this sparingly; the PM's morning attention is precious. Everything else (write order, plain-language enforcement, Notion structure, run-log) is identical across surfaces.

## End-to-end flow

### Step 1 — Read Preferences

Read the Preferences sub-page under the PM's Notion parent. If it doesn't exist, route to `first-run-setup.md` instead and stop.

Extract:
- PM identity (name, Orbit user ID)
- Last-run timestamp (if any)
- Morning run time, execution run time, escalation time + backup
- Account manager list with preferred channels
- Default email/Slack preferences, always-include rules, tone samples

### Step 2 — Determine lookback window

- Default: 12 hours (to catch overnight signals).
- If Preferences has a last-run timestamp: lookback = (now − last_run). This handles PM coming back from leave automatically.
- Cap lookback at 7 days to avoid overwhelming runs.

### Step 2.5 — Load Pod Matrix (cached for the run)

Call `references/pod-matrix.md` to fetch and parse the org Pod Matrix from `POD_MATRIX_URL` (injected by the Mode 1 routine prompt — see `ROUTINE-ENTRYPOINTS.md`).

- On success: cache the parsed pools (PM matrix, floaters, functional matrices) plus the resolved Orbit user-id mapping for the rest of the run. `synthesis/pod-inference.md` reads from this cache.
- On `POD_MATRIX_URL` absent (interactive surface or routine misconfiguration), fetch failure, or parse failure: log a one-line warning to the Run Log Decisions trace and continue. Pod-inference will gracefully degrade to Orbit-only inference (matrix-unavailable code path).

This step is non-blocking: a matrix outage must never block the morning queue.

### Step 3 — Fire collectors in parallel

Call all four collectors. Do not wait for one before starting the next — they're independent.

- `collectors/orbit.md` — scoped to PM's projects (owner + follower + active in last 6 months)
- `collectors/gmail.md` — scoped to PM's inbox, last <lookback_window>
- `collectors/slack.md` — scoped to client/AM/project direct asks, PM DMs
- `collectors/fathom.md` — scoped to calls PM attended or missed in <lookback_window>

Each collector returns a list of raw signals with source metadata and full context.

If a collector fails (MCP unavailable, auth expired), do not abort Mode 1. Log a note and proceed:
> "Gmail was unavailable this morning — you may want to check manually."

This note is included in the summary section at the top of the dated page.

### Step 3.5 — Post-collector assertion (MANDATORY)

Before passing signals to the matcher, verify the Orbit collector actually invoked its non-skippable tool sequence. This guards against the runtime LLM taking a "fast path" that pulls workload metadata only and silently drops every comment + activity-log entry inside the lookback window.

Run the following checks on the Mode 1 tool trace and Orbit signals list:

1. **`get_activity_log` call count must be ≥ 1.** If zero, abort Mode 1 with a hard error. Write the dated page with a single callout block at the top: `Mode 1 aborted — Orbit collector skipped get_activity_log. No queue generated. See Run Log for trace.` Log to Run Log with code `mode1_abort: orbit_activity_log_skipped`. Do NOT write a queue database. Do NOT proceed to synthesis. The PM sees a clear failure on the page instead of a misleadingly-clean queue built on incomplete data.

2. **Orbit signals list contains ≥ 1 entry with `signal_type` in {`activity_log_entry`, `new_comment`, `status_change`, `new_task`}** — OR — the Orbit activity log call returned a documented empty result (`{}` / `data: []`) for every project in the universe. If the call count is 1+ but signals contain zero of those types AND at least one project returned non-empty activity, log a warning `orbit_collector_signals_dropped` to Run Log and continue. The PM will see "0 Orbit signals captured this morning" in the page summary as a soft flag.

3. **Tool trace must show NO `get_user_workload` calls outside the no-history fallback path.** `get_user_workload` is reserved for `synthesis/pod-inference.md` per `SKILL.md` non-negotiable rule #6. If `get_user_workload` was called before any `get_activity_log` call (i.e., used as a substitute), log warning `orbit_collector_used_workload_as_substitute` and continue — the assertion in #1 has already caught the deeper miss.

Output of this step: either a hard abort (case 1) or a clean signals list with warnings logged (cases 2–3). Only on clean pass does synthesis run.

### Step 4 — Synthesize

Feed all collected signals into `synthesis/matcher.md`. The matcher:
- Groups signals by client + project using the Orbit relationship map
- Generates a one-line plain-language summary per item (normal English — PM reads this)
- Flags items as `Uncertain:` when it can't confidently group or classify
- Recommends an action per item
- Calls `synthesis/pod-inference.md` to compute the candidate pool per project (matrix members ∪ Orbit followers/recent-assignees)
- Picks the assignee via the 4-branch decision tree in `synthesis/matcher.md` Job 6: history wins → matrix availability → floater availability → cross-matrix Uncertain. Availability calls (`get_user_workload`) fire only on the no-history fallback path.
- Writes AI Notes as needed (including any matrix-unavailable degradation note)

### Step 5 — Write to Notion

Call `writers/notion.md` to:
1. Archive last month's dated pages if today is the 1st (route to `modes/monthly-archival.md` first, then return)
2. Ensure `Preferences` sub-page is positioned at the very bottom of the parent
3. Resolve / create the Year heading-toggle block on the parent body (e.g., `2026`). Resolve / create the Month heading-toggle block inside it (e.g., `April`). Create today's dated sub-page (Notion-tree parent = parent page) and insert its `child_page` block at the TOP of the Month toggle's children. Title format: `DD Month YYYY` (e.g., `25 April 2026`). Per `schemas/parent-page.md` for the hybrid Year/Month-toggle + Day-sub-page structure. **If a `child_page` block matching today's title already exists in that Month toggle, do NOT overwrite and do NOT prompt.** Append a numeric rerun suffix and create a new sub-page: `25 April 2026 (rerun 2)`, `25 April 2026 (rerun 3)`, etc. Pick the lowest unused suffix. The original page is left untouched.
4. Write content into today's page:
   a. **Top of page: `Ready for Execution` toggle** — a to-do-style checkbox block, unchecked by default, labeled clearly
   b. **Summary line** — "N items for your morning. X reassignments, Y new sub-tasks. <M signals filtered — see Run Log if you want to audit>."
   c. **Inline Morning Queue database** — schema from `schemas/morning-queue-database.md`, one row per item
5. For each row, populate the detail page with the heading-based layout from `schemas/row-detail-page.md`:
   - Summary heading
   - Sources heading (with citations from `writers/source-citation.md`)
   - Recommended Action heading
   - Proposed Orbit Task Body heading (from `schemas/orbit-dq-standard.md`, in plain language from `writers/plain-language.md`)
   - Proposed Slack Handoff heading (plain language)
   - Proposed Email heading (if applicable, normal English)
   - AI Notes heading
   - Reference Context toggle at the bottom (labeled — skill's working memory)

### Step 6 — Update last-run timestamp

Update the Preferences page's `last_morning_run` field to now.

> **Note:** Mode 2 (execution) and the escalation check are pre-scheduled as separate Claude Routines. Mode 1 does NOT register them in-skill. The `scheduled-tasks` MCP is no longer in the allowlist — routines themselves are the scheduler.

### Step 7 — Append run-log entry

Call `writers/run-log.md` with the run summary:
- Timestamp range (start → end of this Mode 1 fire)
- Source counts per collector (Orbit / Gmail / Slack / Fathom signal counts)
- Item count written to today's queue
- Decisions list (key synthesis decisions, especially `Uncertain:` flags and assignee picks)
- Connector status (which MCPs were healthy, which degraded, which failed)
- Page title actually written (including any `(rerun N)` suffix)

The writer creates a row in the Run Log database on the Notion parent and a linked decision-trace detail page. This is how a stateless routine fire leaves a trace for the next fire and for the PM's audit.

### Step 8 — Exit silently

Mode 1 does not notify the PM on completion. The PM will open Notion on their own schedule. No Slack ping, no email. Silent.

Exception: if an MCP source failed, log the note on the page summary so the PM sees it when they open.

## What each row looks like after Mode 1

- **Summary** column: plain-language one-liner in normal English (PM reads this)
- **Status** column: set to `Recommended Action` by default
- **Recommended Action** column: short phrase like "Create task + Slack Vijay"
- **Recommended Assignee** column: person name + short reason ("Vijay Patel (FE) — primary FE on Agency X")
- **PM Notes** column: empty (PM fills this)
- **Outcome** column: empty (Mode 2 fills this after execution)
- **Row page body**: full heading-based detail per `schemas/row-detail-page.md`

## Error handling

| Failure | Behavior |
|---|---|
| Preferences page missing | Route to `first-run-setup.md` and stop |
| Individual collector fails | Log note on page summary, continue with other collectors |
| Notion write fails | Retry once. If still fails, stop and Slack the PM: "Couldn't write today's morning queue — [error]. I'll retry on next scheduled run." |
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
