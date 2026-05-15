> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

# Notion Writer

## Purpose

All Notion write operations. Creates dated sub-pages under the PM's parent, builds the inline Morning Queue database, populates row detail pages, updates status and outcome columns, ensures Preferences stays at the bottom, and handles the monthly archival move.

## Tools used

- `mcp__...notion.notion-fetch` — read pages / databases
- `mcp__...notion.notion-create-pages` — create pages (sub-pages, row detail pages)
- `mcp__...notion.notion-create-database` — create inline databases
- `mcp__...notion.notion-update-page` — update page content or properties
- `mcp__...notion.notion-update-data-source` — update database schema if needed
- `mcp__...notion.notion-move-pages` — for monthly archival

## Flow — writing today's page (Mode 1)

Called after the matcher has produced the ordered list of items.

### Step 1 — Confirm Preferences at bottom of parent

Fetch the parent page. Get the list of child blocks. If `Preferences` isn't the last child block, move it to the end.

### Step 2 — Create today's dated sub-page

- **Title:** today's date in "DD Month YYYY" format. Example: `25 April 2026`.
- **Icon:** 📅
- **Parent:** the PM's designated Notion parent page.
- **Position:** at the TOP of the parent's children, above any older dated pages.

### Step 3 — Write the page body

Content order:

1. **Ready for Execution toggle at the TOP**
   - A to-do-style checkbox block at the very top of the page content (per locked decision 2026-04-25).
   - Label: `Ready for Execution`
   - Initial state: unchecked.
   - Followed by a short instruction: `Flip this to On when you've reviewed all items. The skill's scheduled execution run reads this toggle at [execution time] IST. If it's still off, I escalate to your backup.`

2. **Run metadata callout** — a callout block with:
   - Time Mode 1 fired
   - Lookback window used
   - Any collector failures (if any)

3. **Summary line**
   - One line: `N items for your morning. X new assignments, Y reassignments, Z FYI.`

4. **Inline Morning Queue database**
   - Create via `notion-create-database` with the parent as this dated page.
   - Schema per `schemas/morning-queue-database.md` — 6 main columns visible: Summary, Status, Recommended Action, Recommended Assignee, PM Notes, Outcome.
   - Additional properties (Project, Source Systems, AI Notes) exist in the schema but are hidden from the default view (they appear only when a row is opened).

### Step 4 — Populate each row

For each item in the matcher's output array:

1. Create a row in the database with:
   - `Summary` (title)
   - `Status` = `Recommended Action` (default)
   - `Recommended Action` — short phrase
   - `Recommended Assignee` — name + role + short reason
   - `PM Notes` — empty
   - `Project` — the matched project name
   - `Source Systems` — multi-select of which collectors contributed (Orbit, Gmail, Slack, Fathom)
   - `AI Notes` — any uncertainty flags or split reasoning; empty if none
   - `Outcome` — empty (filled by Mode 2)

2. Populate the row's page content per `schemas/row-detail-page.md`:
   - `Summary` heading + the matcher's summary
   - `Sources` heading + full citations per `writers/source-citation.md`
   - `Recommended Action` heading + reasoning
   - `Proposed Orbit Task Body` heading + 6-section body in plain language
   - `Proposed Slack Handoff` heading + plain-language message
   - `Proposed Email` heading + email draft (if applicable)
   - `AI Notes` heading (if any notes)
   - Bottom toggle: `Reference Context for the Skill — working memory, not for review`, containing the full raw signals and pod inference reasoning

## Flow — updating rows after Mode 2

For each row Mode 2 executed:

- Flip `Status` to `Done` (or leave at prior state if failure; see `modes/mode-2-execution.md`)
- Write the `Outcome` column with the concise result string
- No changes to row detail page content — source-of-truth stays intact

## Flow — monthly archival (1st of month)

See `modes/monthly-archival.md` for the orchestration. Notion Writer handles the actual moves via `notion-move-pages`.

Steps:
1. Identify all dated pages from the previous month under the parent
2. Create a new toggle block under the parent labeled `[Month YYYY]` — e.g., `April 2026`
3. Move each prior-month dated page to be a child of that toggle
4. Ensure `Preferences` is still the last child of the parent after the moves

## Flow — Preferences page updates

Triggered by `PM Task Assignment, change my preference: ...` commands.

1. Locate the `Preferences` sub-page under the parent
2. Parse the instruction (free-form natural language from the PM)
3. Locate the relevant section of the Preferences page
4. Update the specific field(s)
5. If the change affects scheduled tasks (e.g., run time changed), call `mcp__scheduled-tasks__update_scheduled_task` with the new time
6. Confirm back to the PM: `Updated [section] to [new value].`

Use `notion-update-page` with `command: update_content` and targeted find-and-replace for clean edits. Avoid full rewrites that could disrupt the page structure.

## Idempotency

- Don't create a dated page if one already exists for today. Confirm with the PM first if they're manually re-firing Mode 1.
- Don't create a duplicate Preferences page. If one exists and first-run setup re-fires accidentally, route to first-run-setup.md's early-exit logic.
- Don't duplicate Morning Queue databases within a dated page. Each dated page has exactly one.

## Error handling

| Failure | Behavior |
|---|---|
| Page creation fails | Retry once. If fails, abort Mode 1 with Slack to PM: `Couldn't write today's morning queue. [error]` |
| Database creation fails | Retry once. If fails, abort. |
| Row creation fails for a single item | Log in AI Notes on the summary block, continue with remaining rows |
| `notion-move-pages` fails during archival | Log and continue. Missing pages can be moved manually later. |
| Preferences page is missing | Route to `first-run-setup.md` |

## Performance

- Batch row creation where the tool supports it (up to 100 rows per call per Notion's limit)
- Create row detail page content in parallel for the 10-30 rows of a typical morning
- Target total Notion-write time: under 60 seconds for a 20-item morning

## What this writer does NOT do

- Does not write to V3 pages
- Does not modify the PM's Preferences page autonomously (only via explicit preference-edit commands)
- Does not delete pages or databases
- Does not change the parent page's page-level properties (title, icon)
- Does not use toggles inside row detail pages except the one bottom Reference Context toggle
