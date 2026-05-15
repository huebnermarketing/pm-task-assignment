> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

# Monthly Archival

## When this runs

Scheduled on the 1st of each month at 6:00 AM IST (30 minutes before the earliest possible Mode 1 run). Registered during first-run setup as a separate recurring scheduled task.

Can also be triggered indirectly if Mode 1 fires on the 1st and sees un-archived dated pages from the previous month — in that case, Mode 1 calls this flow before creating today's new page.

## Purpose

The PM's Notion parent page would get cluttered with 30+ dated sub-pages per month. Archival collapses each completed month into a single named toggle so the parent stays clean — today's page on top, this month's dated pages below, previous months as named toggles going further back, and `Preferences` always at the very bottom.

## End-to-end flow

### Step 1 — Read Preferences

Pull the Notion parent page ID.

### Step 2 — Determine what to archive

Get all dated sub-pages under the parent. Filter to those dated in the PREVIOUS month. Example: running on 1 May 2026 → archive all pages dated 1 April through 30 April 2026.

If zero pages match, skip archival and exit silently.

### Step 3 — Create the month toggle

Create a new toggle block under the parent, placed just above the earliest-month existing toggle (or just above Preferences if no month toggles exist yet).

- Toggle label: `April 2026` (the month name + year that's being archived — use the format matching the locked date format: "Month YYYY")
- Toggle should be closed by default.

### Step 4 — Move dated pages into the toggle

For each dated sub-page from the previous month:

1. Move the page to be a child of the new month toggle (not a direct child of the parent).
2. Preserve the page's content, database, and any row detail pages — nothing gets deleted.
3. Preserve the order within the month: newest date at the top of the toggle, oldest at the bottom.

Note: moving pages in Notion via MCP uses the `notion-move-pages` tool. Use it for each page.

### Step 5 — Confirm Preferences is still at the bottom

After all moves, fetch the parent page and verify that `Preferences` is the very last child block. If not, move it to the end.

### Step 6 — Log the archival

Add a brief note inside the new month toggle (at the top of the toggle's content, before the dated pages):

> "Archived on [date of archival run]. N pages moved from the parent page on [archival date]. Dated pages sorted newest-top. Any row detail pages within each dated page were preserved."

### Step 7 — Exit

The archival is complete. Mode 1 may now continue to create today's new dated page on top of the newly-organized parent.

## Error handling

| Failure | Behavior |
|---|---|
| Zero previous-month pages found | Skip archival. Do nothing. |
| Move fails for a single page | Log the failure in the month toggle note. Continue with the rest. Report to PM in the next Mode 1 run's summary: "Monthly archival on [date] had [N] failures. Manual cleanup may be needed." |
| Notion API rate-limits | Retry each failed move with exponential backoff, max 3 attempts per page. |

## Special case — first archival run

If the skill has been installed for less than a month, there are no completed-month pages to archive. Skip silently.

## Special case — PM manually re-triggers archival

If the PM types a command to trigger archival (not in v1 but possible later), only archive the previous full month. Never archive the current month or today's page.

## Edge cases

- **Time zones.** The month boundary is calculated in IST. A page dated `30 April 2026` is archived when running on `1 May 2026 06:00 IST`, regardless of where the PM is in the world.
- **Pages with the wrong date format.** If a dated page doesn't parse as a date in "DD Month YYYY" format, leave it alone. Don't guess.
- **`Preferences` positioning.** After every archival, re-position `Preferences` at the end of the parent. This is also enforced on every Mode 1 run (for defense in depth).

## What archival does NOT do

- Does not delete any pages.
- Does not modify dated page content.
- Does not trigger any collectors or executors.
- Does not Slack the PM.
- Does not modify Preferences.
