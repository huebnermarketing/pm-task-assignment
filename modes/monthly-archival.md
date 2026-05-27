> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Notion. Enrichment-on-demand — Fathom (lazy fetch via `collectors/fathom.md`). Slack is outbound-send only (team-handoff + AM-ping with explicit PM `send` note) — not used by this mode. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Monthly Archival

## When this runs

Scheduled on the 1st of each month at 6:00 AM IST (30 minutes before the earliest possible Mode 1 run). Registered during first-run setup as a separate recurring scheduled task.

Can also be triggered indirectly if Mode 1 fires on the 1st and notices the previous-month container is missing, misordered, or has stray pages — in that case, Mode 1 calls this flow before creating today's new page.

## Purpose

Dated sub-pages are already created with their `child_page` blocks placed inside the appropriate Year-toggle → Month-toggle nesting on the parent body from the moment Mode 1 places them (see `schemas/parent-page.md` and `writers/notion.md`). Archival is **verification on a calendar boundary**, not migration. It surfaces drift so the PM knows about it.

The hierarchy invariants archival enforces (against the parent body block tree):

1. Every dated `child_page` block is inside a Month-toggle block.
2. Every Month-toggle block is inside a Year-toggle block.
3. Every Year-toggle block is a direct block on the parent body (below the header callout, above operational sub-page blocks).
4. Year-toggle blocks are ordered descending on the parent body (newest year on top).
5. Month-toggle blocks are ordered descending inside their Year-toggle (newest month on top).
6. Dated `child_page` blocks are ordered descending inside their Month-toggle (newest date on top).
7. `Run Log`, `Incidents`, `Preferences` `child_page` blocks are the last three blocks on the parent body, in that order.

Note: dated sub-pages' Notion-tree parent is the parent page itself (sidebar shows them flat under parent). Archival does NOT verify sidebar tree position — only the parent body block placement.

## End-to-end flow

### Step 1 — Read Preferences

Pull the Notion parent page ID.

### Step 2 — Verify the previous-month toggle exists

Compute the previous month and its year. Example: running on 1 May 2026 → previous month is `April`, year is `2026`.

- Fetch the parent body block tree.
- Confirm a Heading-1 toggle block titled `2026` exists on the parent body. If missing but dated `child_page` blocks from `2026` exist somewhere else (legacy sub-page, flat at parent body), log to run-log as drift; do not auto-create-and-move (PM resolves manually).
- Confirm a Heading-2 toggle block titled `April` exists inside the `2026` toggle. Same drift rule.

### Step 3 — Sweep for misplaced dated `child_page` blocks from the previous month

Walk the parent body block tree. For any `child_page` block whose title matches `DD Month YYYY` format and whose date falls in the previous month, confirm it sits inside the expected Month toggle. Possible drift locations:

- Flat at the parent body level (outside any toggle).
- Inside the wrong Month toggle.
- Inside a legacy Year sub-page (`child_page` block at parent body whose title is the year — pre-toggle schema).

For each misplaced page found: log to the run-log entry with its current parent-body path. Do NOT auto-move. Surface in the run-log summary as: `Drift: <page title> at <current path>, expected inside 2026 → April toggle`.

### Step 4 — Reorder Year-toggle blocks

Walk the parent body. Among the Year-toggle blocks found (Heading-1 toggles whose plain-text content is a 4-digit number), confirm descending order. If drifted, use `notion-update-page` block-reorder to fix.

### Step 5 — Reorder Month-toggle blocks within each Year-toggle

For each Year-toggle, walk its children, identify Month-toggle blocks (Heading-2 toggles whose plain-text content is a spelled-out month name), confirm descending order within that year. Reorder if drifted.

### Step 6 — Reorder dated `child_page` blocks within each Month-toggle

For each Month-toggle reachable from any Year-toggle, walk its children, identify dated `child_page` blocks (`DD Month YYYY` titles), confirm descending order. Reorder if drifted.

### Step 7 — Verify operational sub-page block positions

Confirm `Run Log`, `Incidents`, `Preferences` `child_page` blocks are the last three blocks on the parent body, in that order, after all Year-toggle blocks. Reorder if drifted (use block-reorder via `notion-update-page`; do not move the underlying sub-pages with `notion-move-pages` — they're already correctly parented).

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
| Previous-month Month-toggle not found and no drift items either | Skip silently — that month had zero activity. |
| Block reorder fails on a toggle or `child_page` block | Log and continue. Order drift can be corrected on the next fire. |
| Year-toggle is missing but drift items exist | Log the drift, do NOT auto-create the Year/Month toggles and move the pages. Surface to PM via run-log. |
| Legacy Year/Month sub-page detected on parent body | Log as drift `Legacy Year sub-page: <year>` (or Month). Do NOT auto-migrate. PM resolves. |
| Notion API rate-limits | Retry each failed call with exponential backoff per the policy in `connector-failure-notify.md`. |

## Special case — first archival run

If the skill has been installed for less than a month, there are no completed-month containers to verify. Run Steps 4, 5, 7 anyway — they are cheap and confirm the structure is healthy. Skip Steps 2, 3, 6 silently.

## Special case — PM manually re-triggers archival

If the PM manually triggers archival, only verify previous-and-older months. Do NOT touch the current Month or today's page.

## Edge cases

- **Time zones.** Month boundary is calculated in IST.
- **Pages with the wrong title format.** If a dated `child_page` block doesn't parse as `DD Month YYYY`, leave it alone — don't guess. Log it as `Unparseable title at <path>` in the run-log.
- **Legacy `[Month YYYY]` single-toggle blocks** (e.g., `April 2026` flat toggle directly on parent body from a pre-Year/Month-nesting schema): if found, log as drift `Legacy toggle block at parent: April 2026 — contents need migration to Year-toggle → Month-toggle nesting`. Do NOT auto-migrate.
- **Legacy Year/Month sub-pages** (from the pre-toggle schema where Year and Month were each their own sub-page): if found, log as drift `Legacy Year sub-page: <year>` or `Legacy Month sub-page: <year>/<month>`. Do NOT auto-migrate; the PM moves dated pages out manually before deleting the legacy sub-page.

## What archival does NOT do

- Does not delete any pages.
- Does not modify dated page content.
- Does not auto-move drifted dated pages — only surfaces them for PM review.
- Does not auto-migrate legacy `[Month YYYY]` toggle blocks.
- Does not trigger any collectors or executors.
- Does not notify the PM directly (drift surfaces in the run-log; persistent drift escalates via the standard incident path if it recurs).
- Does not modify Preferences content (only its position).
