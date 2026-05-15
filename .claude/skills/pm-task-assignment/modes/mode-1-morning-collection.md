> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

# Mode 1 — Morning Collection Run

## When this runs

- Scheduled: daily at the PM's configured morning time (default 9:30 AM IST).
- Manual: `PM Task Assignment, run morning`.

## Zero PM input during this run

Mode 1 never asks questions. Never blocks waiting for input. If something is uncertain, it becomes its own row with an `Uncertain:` note in AI Notes. The PM resolves it when they review.

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

### Step 4 — Synthesize

Feed all collected signals into `synthesis/matcher.md`. The matcher:
- Groups signals by client + project using the Orbit relationship map
- Generates a one-line plain-language summary per item (normal English — PM reads this)
- Flags items as `Uncertain:` when it can't confidently group or classify
- Recommends an action per item
- Calls `synthesis/pod-inference.md` to compute the candidate pool per project
- Calls the recommendation logic to pick the single best assignee (no availability check — role fit only)
- Writes AI Notes as needed

### Step 5 — Write to Notion

Call `writers/notion.md` to:
1. Archive last month's dated pages if today is the 1st (route to `modes/monthly-archival.md` first, then return)
2. Ensure `Preferences` sub-page is positioned at the very bottom of the parent
3. Create today's dated sub-page at the TOP of the parent, titled `[DD Month YYYY]` (e.g., `25 April 2026`)
4. Write content into today's page:
   a. **Top of page: `Ready for Execution` toggle** — a to-do-style checkbox block, unchecked by default, labeled clearly
   b. **Summary line** — "N items for your morning. X new assignments, Y reassignments, Z FYI."
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

### Step 6 — Register execution and escalation

Via `mcp__scheduled-tasks`:
- Mode 2 (execution run) at the configured execution time on this same date
- Escalation check at the configured escalation time

If these are already daily recurring tasks, just confirm they're active. Don't re-register if nothing changed.

### Step 7 — Update last-run timestamp

Update the Preferences page's `last_morning_run` field to now.

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

- Target end-to-end: under 3 minutes for a typical morning (4 collectors in parallel, 20-50 signals, 10-30 items after synthesis, Notion write).
- Upper bound: 5 minutes if an unusually large Fathom meeting or 7-day lookback expands the work.

## What Mode 1 does NOT do

- Does not send any messages to anyone.
- Does not write to Orbit.
- Does not draft emails.
- Does not ask the PM anything.
- Does not touch V3 pages.
- Does not check availability or Keka.
- Does not use toggles in row detail pages (except the one bottom reference-context toggle per row).
