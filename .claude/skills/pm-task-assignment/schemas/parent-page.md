# Schema — Parent Notion Page

The PM's Notion parent page (set in `config.md` as `DEFAULT_NOTION_PARENT_PAGE_ID`) follows a fixed structure. Every PM's parent page looks identical in shape and order. This is enforced by `writers/notion.md` on every Mode 1 run and on every monthly archival.

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

📅 [Today's dated page] — created/refreshed daily by Mode 1
📅 [Yesterday's dated page]
📅 [Day before's dated page]
...
📅 [Older dated pages from this month]

▸ [Month YYYY] (toggle, e.g., "April 2026")    ← created on 1st of next month
   📅 [30 dated pages from that month]

▸ [Earlier Month YYYY] (toggle)
   📅 [dated pages]

... older months ...

⚙️ Preferences (always last)
```

## Static elements (always present)

### 1. Header callout — at the very top

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

The header is the only static, persistent block on the parent. Everything else under it is either dated pages or the Preferences page.

### 2. Preferences sub-page — always last

The `Preferences` sub-page is the very last child of the parent. Position is enforced by:

- `writers/notion.md` Step 1 of the Mode 1 flow: confirm Preferences is at the bottom; move it if not.
- Monthly archival: re-confirm position after moving prior-month pages into a toggle.

## Dynamic elements (created and managed by the skill)

### Dated sub-pages — top of parent

Each Mode 1 run creates a new dated sub-page titled with the current date in `DD Month YYYY` format (e.g., `25 April 2026`). New pages go at the TOP of the parent (above the previous day's).

Layout of each dated page is defined in this file (parent-page.md) by reference, but the actual structure is in:
- The dated page itself contains the inline `Morning Queue` database — schema in `schemas/morning-queue-database.md`
- Each row in that database opens to a row detail page — schema in `schemas/row-detail-page.md`

### Month toggles — middle of parent

On the 1st of each month, `monthly-archival.md` runs and:

1. Identifies all dated pages from the previous month
2. Creates a toggle block titled `[Month YYYY]` (e.g., `April 2026`)
3. Moves all those dated pages inside the toggle, preserving newest-on-top order within the toggle
4. Positions the toggle below the current month's dated pages and above earlier-month toggles
5. Re-confirms `Preferences` is still last

After several months of use, the parent looks like:

```
HEADER CALLOUT
📅 25 April 2026
📅 24 April 2026
📅 23 April 2026
...
📅 1 April 2026
▸ March 2026
▸ February 2026
▸ January 2026
⚙️ Preferences
```

## What the parent must NOT contain

- No other content blocks at the parent level (no extra databases, no extra sub-pages, no orphan blocks)
- No PM-added arbitrary content at the parent level (PMs can add content INSIDE Preferences or inside dated pages, but not at the parent level itself)
- No orphan dated pages outside the toggle structure once their month has been archived

The skill enforces this on every Mode 1 run by:

1. Verifying the header callout exists (creating or updating it if needed)
2. Verifying today's dated page is at the top (creating or moving it if needed)
3. Verifying Preferences is at the bottom
4. Logging (but not removing) any unexpected content at the parent level so the PM can investigate

## Visual hierarchy and order — strict rules

The order of children is:

1. Header callout (always first)
2. Today's dated page (created by current run if it doesn't exist)
3. This month's earlier dated pages (newest first)
4. Last month's toggle (if exists)
5. Earlier months' toggles (newest first)
6. Preferences (always last)

Mode 1's Notion writer enforces this order. Monthly archival re-establishes it. If a PM manually rearranges, the next Mode 1 run silently re-sorts.

## Why the structure is fixed

- **Predictability:** every PM's parent looks the same. Onboarding new PMs is faster because the layout is recognizable.
- **Skill reliability:** the skill always knows where to find things. It doesn't have to guess.
- **Audit-friendly:** anyone (PM, leadership, fellow PM) can open a parent page and immediately understand what they're looking at.
- **Archival is clean:** monthly toggles keep the page from growing unbounded.

## What this schema does NOT include

- Per-PM customization of parent layout (none allowed in v1)
- Multiple parents per PM (one parent per PM, hardcoded in config)
- Cross-PM sharing of parent pages (each PM has their own)
- A "drafts" section or "ideas" section at the parent level (everything goes through dated pages)

If a PM wants additional content at the parent level, they should add it INSIDE the Preferences page under a custom heading. The skill won't touch their custom content there.
