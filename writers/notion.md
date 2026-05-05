> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Notion Writer

## Purpose

All Notion write operations. Creates and reuses **Year and Month heading-toggle blocks** on the parent page body (NOT sub-pages — see `schemas/parent-page.md` for the hybrid hierarchy). Places dated sub-pages with parent = the parent page, and inserts each dated sub-page's `child_page` block inside the relevant Month toggle. Builds the inline Morning Queue database on each dated sub-page, populates row detail sub-pages, updates status and outcome columns, ensures Preferences stays at the bottom, and verifies the structure on monthly archival.

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

Fetch the parent page block tree. Confirm the `Preferences` `child_page` block is the last block on the parent body. If not, move it to the end via `notion-update-page` block-reorder.

### Step 2 — Resolve / create the Year toggle block

- Compute current `YYYY` (4-digit, e.g., `2026`).
- Fetch the parent's body block tree.
- Look for a Heading-1 toggle block (`heading_1` with `is_toggleable: true`) on the parent body whose plain-text content is exactly `YYYY` (no prefix, no suffix).
- If absent: insert one at the top of the parent body, immediately below the header callout, above any earlier Year toggles. Use `notion-update-page` to append the new block, then move it into position. Default state: expanded.
- If present: reuse — never create a duplicate. If multiple match, log to run-log, use the topmost, leave the duplicates alone for PM review.
- **Legacy detection:** if a sub-page (`child_page` block) titled `YYYY` exists on the parent body instead of a toggle block, log to run-log as `Legacy Year sub-page detected: <year>`. Create the new toggle block in the correct position; the legacy sub-page is left untouched (no auto-migration).

### Step 3 — Resolve / create the Month toggle block

- Compute current `Month` spelled out (e.g., `April`).
- Fetch the Year toggle's children blocks.
- Look for a Heading-2 toggle block (`heading_2` with `is_toggleable: true`) inside the Year toggle whose plain-text content is exactly `Month` (spelled out, no year suffix).
- If absent: insert one at the TOP of the Year toggle's children, above earlier Month toggles. Default state: expanded.
- If present: reuse — same dup + legacy rule as Year.

### Step 4 — Create today's dated sub-page

- **Title:** today's date in "DD Month YYYY" format. Example: `25 April 2026`.
- **Icon:** 📅
- **Notion-tree parent:** the parent page (use `notion-create-pages` with `parent` = `DEFAULT_NOTION_PARENT_PAGE_ID`). The dated sub-page is a direct child of the parent in the Notion sidebar tree.
- **Body-block placement on parent:** insert the resulting `child_page` block at the TOP of the Month toggle's children (above older dated `child_page` blocks from the same month). Use `notion-update-page` block-append + reorder.
- **Idempotency:** before creating, scan the Month toggle's children for an existing `child_page` block with the same title. If one exists and the run is intentional (PM-fired re-run), append a numeric rerun suffix per the existing flow: `25 April 2026 (rerun 2)`, `(rerun 3)`, etc. Pick the lowest unused suffix.

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

See `modes/monthly-archival.md` for the orchestration. Because every dated page is created inside its Year-toggle → Month-toggle nesting on the parent body, archival is **verification**, not migration. The Notion Writer's role on archival day:

1. Verify the previous-month Month-toggle exists inside its Year-toggle on the parent body, and that all dated `child_page` blocks for that month sit inside it.
2. If any dated `child_page` from a prior month is found at the parent body level outside any Month toggle OR inside the wrong Month toggle OR inside a legacy Year sub-page, log it in the run-log entry but DO NOT auto-move (placement drift implies manual edits — surface for PM review).
3. Verify Year-toggle ordering on the parent body: newest Year on top, descending. Reorder via `notion-update-page` block-reorder if drifted.
4. Verify Month-toggle ordering inside each Year-toggle: newest Month on top. Reorder if drifted.
5. Verify dated `child_page` block ordering inside each Month-toggle: newest date on top. Reorder if drifted.
6. Ensure `Preferences` `child_page` block is still the last block on the parent body.
7. Ensure `Run Log` and `Incidents` `child_page` blocks sit immediately above `Preferences`, in that order, after all Year-toggle blocks.

`notion-move-pages` is used only when sub-pages (Day, Run Log, Incidents, Preferences) need to be re-rooted across parent pages — which should never happen in normal operation. Toggle-block reordering uses `notion-update-page` instead.

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

- Don't create a duplicate Year toggle block. Match by exact 4-digit plain-text title on the parent body.
- Don't create a duplicate Month toggle block. Match by exact spelled-out month name inside the relevant Year toggle.
- Don't create a dated sub-page if a `child_page` block matching today's title already exists in the resolved Month toggle. The rerun-suffix flow (Step 4) handles intentional re-fires.
- Don't create a duplicate Preferences page. If one exists and first-run setup re-fires accidentally, route to first-run-setup.md's early-exit logic.
- Don't duplicate Morning Queue databases within a dated page. Each dated page has exactly one.
- Don't auto-migrate legacy Year/Month sub-pages into toggle blocks — log drift, leave alone.

## Error handling

| Failure | Behavior |
|---|---|
| Page creation fails | Retry once. If fails, abort Mode 1 with Slack to PM: `Couldn't write today's morning queue. [error]` |
| Database creation fails | Retry once. If fails, abort. |
| Row creation fails for a single item | Log in AI Notes on the summary block, continue with remaining rows |
| Block reorder fails during archival | Log and continue. Order drift can be corrected on the next fire. |
| `notion-move-pages` fails during archival sub-page reorder (Run Log / Incidents / Preferences) | Log and continue. |
| Year or Month toggle creation fails | Retry once. If still fails, abort Mode 1 with Slack to PM: `Couldn't create the Year/Month toggle block. [error]` — do NOT fall back to creating the dated page flat at the parent body level. |
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
