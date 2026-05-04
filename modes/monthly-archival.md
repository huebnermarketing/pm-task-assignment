> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Monthly Archival

## When this runs

Scheduled on the 1st of each month at 6:00 AM IST (30 minutes before the earliest possible Mode 1 run). Registered during first-run setup as a separate recurring scheduled task.

Can also be triggered indirectly if Mode 1 fires on the 1st and notices the previous-month container is missing, misordered, or has stray pages — in that case, Mode 1 calls this flow before creating today's new page.

## Purpose

Dated pages are already created inside `Parent → Year → Month → Date` from the moment Mode 1 places them (see `schemas/parent-page.md` and `writers/notion.md`). Archival no longer collapses flat pages into a toggle — that legacy job is gone. Archival now does one thing: **verify the hierarchy is still correct on a calendar boundary** and surface drift so the PM knows about it.

The hierarchy invariants archival enforces:

1. Every dated page is a child of a Month page.
2. Every Month page is a child of a Year page.
3. Every Year page is a direct child of the parent.
4. Year pages are ordered descending under the parent (newest year on top).
5. Month pages are ordered descending under their Year (newest month on top).
6. Date pages are ordered descending under their Month (newest date on top).
7. `Run Log`, `Incidents`, `Preferences` are the last three children of the parent, in that order.

## End-to-end flow

### Step 1 — Read Preferences

Pull the Notion parent page ID.

### Step 2 — Verify the previous-month container exists

Compute the previous month and its year. Example: running on 1 May 2026 → previous month is `April`, year is `2026`.

- Confirm a Year page titled `2026` exists as a direct child of the parent. If missing but dated pages from `2026` exist somewhere else, log to run-log as drift; do not auto-create-and-move (PM resolves manually).
- Confirm a Month page titled `April` exists as a child of `2026`. Same drift rule.

### Step 3 — Sweep for misplaced dated pages from the previous month

Search for any pages titled in `DD Month YYYY` format dated in the previous month that are NOT children of the expected Month page. Possible drift locations:

- Direct children of the parent (flat, legacy structure).
- Direct children of a Year page (skipped Month nesting).
- Inside the wrong Year or Month (manual misfile).

For each misplaced page found: log to the run-log entry with its current path. Do NOT auto-move. Surface in the run-log summary as: `Drift: <page title> at <current path>, expected at 2026/April`.

### Step 4 — Reorder Year pages

Fetch the parent's children. Among the Year pages found (those with 4-digit numeric titles), confirm descending order. If drifted, use `notion-move-pages` to reorder.

### Step 5 — Reorder Month pages within each Year

For each Year page, fetch children, identify Month pages (titles matching a spelled-out month name), confirm descending order within that year. Reorder if drifted.

### Step 6 — Reorder Date pages within each Month

For each Month page reachable from any Year, fetch children, identify dated pages (`DD Month YYYY` format), confirm descending order. Reorder if drifted.

### Step 7 — Verify operational sub-pages position

Confirm `Run Log`, `Incidents`, `Preferences` are the last three children of the parent, in that order. Reorder if drifted.

### Step 8 — Append archival summary to Run Log

Append a Run Log entry with:

- Trigger = `monthly-archival`
- Counts: years checked, months checked, dated pages checked, drift items surfaced, reorders performed
- Drift items listed (path each)

### Step 9 — Exit

The archival is complete. Mode 1 may now continue to create today's new dated page inside the correctly-structured hierarchy.

## Error handling

| Failure | Behavior |
|---|---|
| Previous-month Month page not found and no drift items either | Skip silently — that month had zero activity. |
| `notion-move-pages` fails on a reorder | Log and continue. Order drift can be corrected on the next fire. |
| Year page is missing but drift items exist | Log the drift, do NOT auto-create the Year/Month and move pages. Surface to PM via run-log. |
| Notion API rate-limits | Retry each failed call with exponential backoff per the policy in `connector-failure-notify.md`. |

## Special case — first archival run

If the skill has been installed for less than a month, there are no completed-month containers to verify. Run Steps 4, 5, 7 anyway — they are cheap and confirm the structure is healthy. Skip Steps 2, 3, 6 silently.

## Special case — PM manually re-triggers archival

If the PM manually triggers archival, only verify previous-and-older months. Do NOT touch the current Month or today's page.

## Edge cases

- **Time zones.** Month boundary is calculated in IST.
- **Pages with the wrong title format.** If a dated page doesn't parse as `DD Month YYYY`, leave it alone — don't guess. Log it as `Unparseable title at <path>` in the run-log.
- **Legacy `[Month YYYY]` toggle blocks** (e.g., `April 2026` toggle directly under the parent from before this skill version): if found, log as drift `Legacy toggle block at parent: April 2026 — contents need migration to Year/Month sub-page hierarchy`. Do NOT auto-migrate; PM must confirm before the skill restructures historical pages.

## What archival does NOT do

- Does not delete any pages.
- Does not modify dated page content.
- Does not auto-move drifted dated pages — only surfaces them for PM review.
- Does not auto-migrate legacy `[Month YYYY]` toggle blocks.
- Does not trigger any collectors or executors.
- Does not Slack the PM (drift surfaces in the run-log; persistent drift escalates via the standard incident path if it recurs).
- Does not modify Preferences content (only its position).
