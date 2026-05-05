# Schema — Parent Notion Page

The PM's Notion parent page (set in `config.md` as `DEFAULT_NOTION_PARENT_PAGE_ID`) follows a fixed structure. Every PM's parent page looks identical in shape and order. This is enforced by `writers/notion.md` on every Mode 1 run and on every monthly archival, and re-confirmed by `preflight.md` Step 6 on every routine fire.

See also:
- `schemas/run-log-database.md` — schema for the inline database on the `Run Log` sub-page
- `schemas/run-log-detail-page.md` — schema for the per-fire detail pages nested under `Run Log`
- `connector-failure-notify.md` — Tier 4 fallback that writes to the `Incidents` sub-page
- `preflight.md` — Step 6 creates `Run Log` and `Incidents` sub-pages if missing and re-asserts order

## Top-down layout

```
[Page header — controlled by Notion, just shows the page title]

┌─────────────────────────────────────────────────────────────┐
│ HEADER CALLOUT (static, updated daily by Mode 1)            │
│                                                              │
│ "PM Task Assignment skill writes here every morning at      │
│  [PM's morning run time] IST. Approved items execute at     │
│  [PM's execution run time] IST. Last run: [timestamp]."     │
└─────────────────────────────────────────────────────────────┘

▼ 2026                                              ← H1 heading-toggle block on parent body
   ▼ April                                          ← H2 heading-toggle nested inside Year toggle
      📅 25 April 2026                              ← page-link block (Day sub-page lives here)
      📅 24 April 2026
      ...
      📅 1 April 2026
   ▼ March                                          ← H2 heading-toggle (collapsed by default)
      📅 dated page links
   ▶ February (collapsed)
   ▶ January (collapsed)

▶ 2025 (collapsed)                                  ← Older years collapsed by default
   ▶ December
   ...

📒 Run Log         ← sub-page (unchanged) — routine-fire log + linked detail pages
🚨 Incidents       ← sub-page (unchanged) — append-only connector-failure log (Tier 4 fallback)
⚙️ Preferences     ← sub-page (unchanged) — always last
```

## The hybrid hierarchy — read this carefully

Year and Month containers are **heading-toggle blocks on the parent's body content**, not sub-pages. The Year/Month structure is purely visual organization on the parent page; it does NOT create sub-pages in the Notion sidebar tree.

Day pages are still **sub-pages**. Their Notion-tree parent is the parent page itself (so they show up in the parent's sidebar children list). Their visual location on the parent page body is inside the Year toggle → Month toggle nesting.

| Layer | Notion type | Notion-tree parent | Where it appears |
|---|---|---|---|
| Year (e.g., `2026`) | Heading-1 toggle block | n/a — block, not a page | Inside parent body, descending year order |
| Month (e.g., `April`) | Heading-2 toggle block | n/a — block, not a page | Inside its Year toggle, descending month order |
| Date (e.g., `25 April 2026`) | Sub-page (`child_page` block) | Parent page | Sidebar: flat under parent. Body: page-link block inside the relevant Month toggle |
| Row detail | Sub-page | Date sub-page | Sidebar: under Date. Body: opened from row in inline Morning Queue |
| Run Log | Sub-page | Parent page | Sidebar: under parent. Body: bottom of parent above Incidents |
| Run Log detail | Sub-page | Run Log sub-page | Sidebar: under Run Log. Body: opened from Run Log inline DB row |
| Incidents | Sub-page | Parent page | Sidebar: under parent, below Run Log |
| Preferences | Sub-page | Parent page | Sidebar: under parent. Body: bottom of parent (always last) |

**Why hybrid:** toggle blocks let the PM scan the whole year on a single page (open the right Year/Month, see all the dates) without clicking into nested sub-pages. Day pages stay as sub-pages because (a) Mode 2 needs a stable URL per day, (b) row detail pages live as children of the day page, (c) the day page contains an inline database (Morning Queue) which renders better as a full page than nested deep in toggles.

**Sidebar consequence:** the parent's sidebar children list will accumulate one entry per day plus Run Log / Incidents / Preferences. After a year of use, ~365 day sub-pages live flat under the parent in the sidebar. This is by design — sidebar is not the primary navigation surface; the parent page body's Year/Month toggle structure is.

## Static elements (always present)

### 1. Header callout — at the very top of parent body

A Notion callout block. Updated by Mode 1 every morning (the timestamp at minimum). Content:

```
PM Task Assignment skill writes here every morning at [TIME] IST.
Approved items execute at [TIME] IST.
Last morning run: [ISO timestamp]
Last execution run: [ISO timestamp]
```

If the skill detected connector failures or other issues at the last run, the header includes a brief warning line:

```
⚠️ Last run had issues — see today's page for details.
```

The header is the only static, persistent block at the very top of the parent body.

### 2. Run Log sub-page — second-to-last static sub-page

- **Title:** exactly `Run Log` (no emoji, no suffix — the writer matches by exact title).
- **Notion-tree parent:** the parent page (sidebar shows it as a direct child of parent).
- **Body location on parent:** below all Year toggle blocks, immediately above `Incidents`. Represented on parent body as a `child_page` block.
- **Contents:**
  1. A one-paragraph header callout at the top of the page reading approximately:
     > "This page is auto-written by the PM Task Assignment skill on every routine fire. Do not edit rows manually — manual edits will be overwritten or ignored. To investigate a specific fire, click into its row to open the detail page."
  2. An inline Notion database below the callout. Schema is in `schemas/run-log-database.md`.
- **Sort:** the inline database default view sorts by `Started` descending (newest fire at the top).
- **Detail pages:** each row in the inline database opens a child page whose schema is in `schemas/run-log-detail-page.md`. Detail pages are nested as children of THIS sub-page (not of the parent), which keeps the parent tidy.
- **Lifecycle:** created on first preflight if missing (per `preflight.md` Step 6). Once it exists, the writer (`writers/run-log.md`) appends a row on every fire.

### 3. Incidents sub-page — between Run Log and Preferences

- **Title:** exactly `Incidents`.
- **Notion-tree parent:** the parent page.
- **Body location on parent:** between Run Log and Preferences.
- **Contents:** an append-only inline Notion database. No header callout required — the title is self-explanatory. Columns:

  | Column              | Type                       | Notes                                                                                       |
  | ------------------- | -------------------------- | ------------------------------------------------------------------------------------------- |
  | `Timestamp`         | Date (with time)           | When the incident occurred (ISO 8601, Asia/Kolkata IST with `+05:30` offset).               |
  | `Mode`              | Select                     | One of: `Mode 1`, `Mode 2`, `Monthly Archival`.                                             |
  | `MCP`               | Select                     | One of: `Orbit`, `Gmail`, `Slack`, `Fathom`, `Notion`.                                      |
  | `Step`              | Text                       | Which step in the mode failed (free text, e.g., "fetch overdue tasks").                     |
  | `Error`             | Text                       | The error message captured from the connector.                                              |
  | `Retry on next fire`| Checkbox                   | If checked, the next fire of the same mode will retry the failed step before normal flow.   |
  | `Resolved`          | Checkbox                   | Set manually by the PM (or by the skill once a later fire succeeds on the same MCP/step).   |
  | `Run Log link`      | URL (or relation)          | Points at the corresponding row's detail page in `Run Log`.                                 |

- **Sort:** by `Timestamp` descending (most recent incident first).
- **Lifecycle:** created on first preflight if missing (per `preflight.md` Step 6), OR on first incident if it doesn't exist yet (the connector-failure path in `connector-failure-notify.md` Tier 4 will create-or-append).
- **Read on every fire:** preflight loads unresolved incidents and surfaces them in the new run-log entry's "Pre-existing unresolved incidents" field, so each fire is aware of standing failures.
- **Escalation:** 3+ consecutive unresolved incidents on the same `MCP` trigger a Slack escalation to the backup person — already documented in `connector-failure-notify.md`.

### 4. Preferences sub-page — always last

The `Preferences` sub-page is the very last child of the parent (sidebar tree) and the last block on the parent body. Position is enforced by:

- `writers/notion.md` Step 1 of the Mode 1 flow: confirm Preferences is at the bottom; move it if not.
- `preflight.md` Step 6: re-confirm Preferences is last after creating/positioning `Run Log` and `Incidents`.
- Monthly archival: re-confirm position as part of the structure-verification pass.

## Dynamic elements (created and managed by the skill)

### Year toggle block — top of parent body

A Year toggle block (e.g., `2026`) is the outermost dated container on the parent body. The current Year toggle sits at the TOP of the parent body (just below the header callout), above earlier Year toggles. Title is the bare 4-digit year — no prefix, no suffix.

- **Notion type:** Heading-1 toggle (`heading_1` block with `is_toggleable: true`). Falls back to a regular toggle block if the writer cannot create heading-toggles via the MCP — schema is permissive, but heading-toggle is preferred for visual hierarchy.
- **Default state:** current year expanded; older years collapsed.
- **Lifecycle:** created on demand by `writers/notion.md` (or any writer that needs to place a dated page) when no Year toggle for the current year exists. Idempotent — never created twice.

### Month toggle block — inside Year

A Month toggle block (e.g., `April`) lives inside its Year toggle on the parent body. Title is the spelled-out month name only (e.g., `April`, not `04`, `Apr`, or `April 2026`). The year context is implicit from the Year toggle wrapping it.

- **Notion type:** Heading-2 toggle (`heading_2` block with `is_toggleable: true`).
- **Order inside the Year toggle:** newest month at the top, descending. So inside `2026` you see `April → March → February → January` top-to-bottom.
- **Default state:** current month expanded; older months collapsed.
- **Lifecycle:** created on demand when no Month toggle for the current month exists inside the current Year toggle. Idempotent.

### Dated sub-page — page-link inside Month toggle

Each Mode 1 run creates (or refreshes) a dated sub-page titled with the date in `DD Month YYYY` format (e.g., `25 April 2026`).

- **Notion-tree parent:** the parent page (NOT a Month sub-page — there are no Month sub-pages).
- **Page-link block placement on parent body:** as a `child_page` block, placed at the TOP of the relevant Month toggle's children. Newest date at top within the month.
- **Sidebar appearance:** under the parent page in the Notion sidebar, alongside other day sub-pages.
- **Page contents:** the inline `Morning Queue` database — schema in `schemas/morning-queue-database.md`. Each row in that database opens to a row detail page — schema in `schemas/row-detail-page.md`.

### Resolving "where does today's dated page go?"

On every Mode 1 fire (and any manual write that creates a dated page):

1. Compute current `YYYY`, `Month` (spelled out), and `DD Month YYYY` title.
2. Fetch the parent's body block tree.
3. Look for a Heading-1 toggle block on the parent body whose plain-text content is exactly `YYYY`. If absent, create it at the top of the parent body (immediately below the header callout). If multiple match, log to run-log and use the topmost.
4. Inside that Year toggle's children, look for a Heading-2 toggle block whose plain-text content is exactly `Month`. If absent, create it at the top of the Year toggle. If multiple match, log + use topmost.
5. Create the dated sub-page (`notion-create-pages` with parent = the Parent page). Insert its `child_page` block at the top of the Month toggle's children. If a `child_page` block matching today's title already exists in that Month toggle, reuse — do not create a duplicate (the rerun-suffix flow in `writers/notion.md` handles intentional reruns).

This sequence is the single source of truth for placement. Placement is correct from creation — the skill does NOT lazily flatten dated pages and archive them later. Monthly archival (`modes/monthly-archival.md`) only verifies nothing has drifted out of place.

After several months of use, the parent body looks like:

```
HEADER CALLOUT
▼ 2026
   ▼ April
      📅 25 April 2026
      📅 24 April 2026
      ...
      📅 1 April 2026
   ▶ March
   ▶ February
   ▶ January
▶ 2025
   ▶ December
   ...
📒 Run Log         (child_page block — sub-page)
🚨 Incidents       (child_page block — sub-page)
⚙️ Preferences     (child_page block — sub-page)
```

## What the parent must NOT contain (on the body)

- No other content blocks at the parent body level outside the structure above (no extra databases, no orphan blocks, no PM-added text content)
- **No flat dated `child_page` blocks at the parent body level** — every dated page-link must sit inside a Month toggle inside a Year toggle. Day sub-pages whose Notion-tree parent is the parent are expected, but their `child_page` block on the parent body must be inside the right Year/Month toggle.
- **No flat Month toggles at the parent body level** — every Month toggle must sit inside a Year toggle.
- **No legacy Year/Month sub-pages** — if the parent contains older `2025` / `April` sub-pages from a prior schema version, they are migration drift. Surface in run-log; the writer does NOT auto-migrate.
- **No `[Month YYYY]` toggle blocks at the parent level** (legacy structure superseded by Year/Month toggle nesting)
- No `Run Log` detail pages at the parent level — those must be nested under the `Run Log` sub-page

The skill enforces this on every routine fire by:

1. Verifying the header callout exists (creating or updating it if needed)
2. Verifying the current Year toggle block exists at the top of the parent body (creating it if missing)
3. Verifying the current Month toggle exists inside the Year toggle (creating it if missing)
4. Verifying today's dated `child_page` block is at the top of the Month toggle (creating or moving it if needed)
5. Verifying `Run Log` exists as a sub-page and its `child_page` block sits below all Year toggles, above `Incidents` (creating the sub-page if missing — see `preflight.md` Step 6)
6. Verifying `Incidents` exists as a sub-page and its `child_page` block sits below `Run Log` (creating if missing — see `preflight.md` Step 6)
7. Verifying `Preferences` `child_page` block is the last block on the parent body
8. Logging (but not removing) any unexpected content at the parent level so the PM can investigate
9. Logging (but not auto-moving) any flat dated page-link found at the parent body level outside the toggle structure — surfaces it in the run-log so the PM can confirm before relocation

## Visual hierarchy and order — strict rules

The parent body has these ordered top-level blocks:

| #  | Block                            | Type                              | Created/refreshed by                              | Notes                                                  |
| -- | -------------------------------- | --------------------------------- | ------------------------------------------------- | ------------------------------------------------------ |
| 1  | Header callout                   | Callout block                     | Mode 1 daily; preflight on every fire             | Always first.                                          |
| 2  | Current Year toggle              | Heading-1 toggle                  | Mode 1 / writers/notion.md on demand              | Newest year at top. Contains Month toggles.            |
| 3  | Earlier Year toggles             | Heading-1 toggle                  | Prior Mode 1 runs in those years                  | Descending year order.                                 |
| 4  | `Run Log` sub-page link          | `child_page` block                | `preflight.md` Step 6 (creates if missing)        | Below all Year toggles, above `Incidents`.             |
| 5  | `Incidents` sub-page link        | `child_page` block                | `preflight.md` Step 6 OR Tier 4 of failure path   | Between `Run Log` and `Preferences`.                   |
| 6  | `Preferences` sub-page link      | `child_page` block                | PM-curated; position enforced                     | Always last.                                           |

**Hierarchy nesting on parent body:** Year toggles contain Month toggles (descending), Month toggles contain dated `child_page` blocks (descending). The parent body itself only ever has the blocks listed in the table above — nothing else.

**Sidebar tree under the parent (informational):** all dated sub-pages flat (one per day), plus the three operational sub-pages (Run Log, Incidents, Preferences). Run Log detail pages are children of Run Log (not parent). Row detail pages are children of their respective dated sub-page (not parent). The sidebar is informational; navigation happens via the parent body.

Mode 1's Notion writer enforces this order. `preflight.md` Step 6 creates `Run Log` and `Incidents` in the correct slot if either is missing on a routine fire. Monthly archival verifies (rather than reshuffles) the structure since placement is correct from creation. If a PM manually rearranges, the next routine fire silently re-sorts top-level blocks and toggle children; flat dated/month pages found at the parent level are flagged in the run-log for PM review rather than auto-moved.

## Why the structure is fixed

- **Predictability:** every PM's parent looks the same. Onboarding new PMs is faster because the layout is recognizable.
- **Skill reliability:** the skill always knows where to find things. It doesn't have to guess.
- **Audit-friendly:** anyone (PM, leadership, fellow PM) can open a parent page and immediately see the whole year by expanding/collapsing toggles.
- **Scan-on-one-page:** Year/Month toggles let the PM see the entire history without clicking into sub-pages — major UX improvement over the prior all-sub-page schema.
- **Stable URLs preserved:** day sub-pages keep their stable URL contract for Mode 2 and for any manual link-shares the PM does.

## What this schema does NOT include

- Per-PM customization of parent layout (none allowed in v1)
- Multiple parents per PM (one parent per PM, hardcoded in config)
- Cross-PM sharing of parent pages (each PM has their own)
- A "drafts" section or "ideas" section at the parent level (everything goes through dated pages)

If a PM wants additional content at the parent level, they should add it INSIDE the Preferences page under a custom heading. The skill won't touch their custom content there.

## Migration note (from prior all-sub-page schema)

Earlier installs created Year and Month as sub-pages instead of toggle blocks. If the writer encounters legacy Year/Month sub-pages on a parent during a Mode 1 fire:

- Log the drift in the run-log detail page (`Legacy Year sub-page found at parent: <year>` or `Legacy Month sub-page found at parent: <year>/<month>`).
- Do NOT auto-migrate the dated pages out of the legacy sub-page hierarchy.
- Surface to the PM via the run-log so they can decide whether to re-home dated pages manually.
- New dated pages going forward use the toggle-based structure regardless of legacy presence.

The PM completes the migration manually (or via a one-time operator script) — the skill stays read-only on legacy structure to prevent accidental data movement.
