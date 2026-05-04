> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Notion Writer

## Purpose

All Notion write operations. Creates and reuses Year and Month container pages, places dated sub-pages inside `Parent → Year → Month → Date` hierarchy, builds the inline Morning Queue database, populates row detail pages, updates status and outcome columns, ensures Preferences stays at the bottom, and verifies the structure on monthly archival.

## Tools used

- `mcp__...notion.notion-fetch` — read pages / databases
- `mcp__...notion.notion-create-pages` — create pages (sub-pages, row detail pages)
- `mcp__...notion.notion-create-database` — create inline databases
- `mcp__...notion.notion-update-page` — update page content or properties
- `mcp__...notion.notion-update-data-source` — update database schema if needed
- `mcp__...notion.notion-move-pages` — for monthly archival and structural drift correction

## Flow — writing today's page (Mode 1)

Called after the matcher has produced the ordered list of items.

### Step 1 — Confirm Preferences at bottom of parent

Fetch the parent page. Get the list of child blocks. If `Preferences` isn't the last child block, move it to the end.

### Step 2 — Resolve / create the Year container

- Compute current `YYYY` (4-digit, e.g., `2026`).
- Look for a child of the parent titled exactly `YYYY` (no prefix/suffix).
- If absent: create it via `notion-create-pages` with parent = PM's parent page. Title = `YYYY`. Icon = 📂. Position at the TOP of the parent's children, above any earlier Year pages.
- If present: reuse — never create a duplicate. If somehow more than one exists, log to run-log; use the topmost and skip duplicates (PM resolves manually).

### Step 3 — Resolve / create the Month container

- Compute current `Month` spelled out (e.g., `April`).
- Look for a child of the Year page titled exactly `Month` (spelled out, no year suffix).
- If absent: create it via `notion-create-pages` with parent = the Year page. Title = `Month`. Icon = 📂. Position at the TOP of the Year page's children, above earlier Month pages.
- If present: reuse — same dup rule as Year.

### Step 4 — Create today's dated sub-page

- **Title:** today's date in "DD Month YYYY" format. Example: `25 April 2026`.
- **Icon:** 📅
- **Parent:** the Month page resolved in Step 3 (NOT the PM's parent page).
- **Position:** at the TOP of the Month page's children, above older dated pages from the same month.

### Step 5 — Write the page body

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

### Step 6 — Populate each row

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
- If the row produced a team / AM Slack draft (the default path for non-`send` rows), append a `Slack draft (copy to send)` block to the row's detail page under the existing Outcome content. Block contents:
  - First line — `Audience: <Team | AM | Client>` and `Recipient: <name> (<slack handle>)`
  - The full plain-language draft body returned by `executors/slack.md`
- No other changes to row detail page content — source-of-truth stays intact.

## Flow — monthly archival (1st of month)

See `modes/monthly-archival.md` for the orchestration. Because every dated page is created inside `Parent → Year → Month → Date`, archival is **verification**, not migration. The Notion Writer's role on archival day:

1. Verify the previous-month container exists at `Parent → Year → Month` and contains all dated pages from that month.
2. If any dated page from a prior month is found floating at the parent level OR directly under a Year page (without a Month parent) OR misfiled in the wrong Month, log it in the run-log entry but DO NOT auto-move (placement drift implies manual edits — surface for PM review).
3. Verify Year ordering: newest Year on top, descending. Reorder via `notion-move-pages` if drifted.
4. Verify Month ordering inside each Year: newest Month on top. Reorder if drifted.
5. Ensure `Preferences` is still the last child of the parent.
6. Ensure `Run Log` and `Incidents` sit immediately above `Preferences`, in that order.

## Flow — Preferences page updates

Triggered by `PM Task Assignment, change my preference: ...` commands.

1. Locate the `Preferences` sub-page under the parent
2. Parse the instruction (free-form natural language from the PM)
3. Locate the relevant section of the Preferences page
4. Update the specific field(s)
5. If the change affects run times (morning, execution), update the Preferences fields only — note in the operator's response that the corresponding Claude Routine cron must also be updated externally (the skill does not modify routines from inside).
6. Confirm back to the PM: `Updated [section] to [new value].`

Use `notion-update-page` with `command: update_content` and targeted find-and-replace for clean edits. Avoid full rewrites that could disrupt the page structure.

## Idempotency

- Don't create a duplicate Year page. Match by exact 4-digit title under the parent.
- Don't create a duplicate Month page. Match by exact spelled-out month name under the relevant Year page.
- Don't create a dated page if one already exists for today (under the resolved Month). Confirm with the PM first if they're manually re-firing Mode 1.
- Don't create a duplicate Preferences page. If one exists and first-run setup re-fires accidentally, route to first-run-setup.md's early-exit logic.
- Don't duplicate Morning Queue databases within a dated page. Each dated page has exactly one.

## Error handling

| Failure | Behavior |
|---|---|
| Page creation fails | Retry once. If fails, abort Mode 1 with Slack to PM: `Couldn't write today's morning queue. [error]` |
| Database creation fails | Retry once. If fails, abort. |
| Row creation fails for a single item | Log in AI Notes on the summary block, continue with remaining rows |
| `notion-move-pages` fails during archival reorder | Log and continue. Order drift can be corrected on the next fire. |
| Year or Month container creation fails | Retry once. If still fails, abort Mode 1 with Slack to PM: `Couldn't create the Year/Month container. [error]` — do NOT fall back to creating the dated page flat under the parent. |
| Preferences page is missing | Route to `first-run-setup.md` |

## Performance

- Batch row creation where the tool supports it (up to 100 rows per call per Notion's limit)
- Create row detail page content in parallel for the 10-30 rows of a typical morning
- Target total Notion-write time: under 60 seconds for a 20-item morning

## What this writer does NOT do

- Does not write to V3 pages (see `references/v3-context.md`)
- Does not modify the PM's Preferences page autonomously (only via explicit preference-edit commands)
- Does not delete pages or databases
- Does not change the parent page's page-level properties (title, icon)
- Does not use toggles inside row detail pages except the one bottom Reference Context toggle
