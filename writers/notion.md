> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand — Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Notion Writer

## Purpose

All Notion write operations. Creates and reuses **Year and Month heading-toggle blocks** on the parent page body (NOT sub-pages — see `schemas/parent-page.md` for the hybrid hierarchy). Places dated sub-pages with parent = the parent page, and inserts each dated sub-page's `child_page` block inside the relevant Month toggle. Builds the inline Morning Queue database on each dated sub-page, populates row detail sub-pages, updates status and outcome columns, ensures Preferences stays at the bottom, and verifies the structure on monthly archival.

## Tools used

- `mcp__...notion.notion-fetch` — read pages / databases
- `mcp__...notion.notion-create-pages` — create pages (sub-pages, row detail pages)
- `mcp__...notion.notion-create-database` — create inline databases
- `mcp__...notion.notion-update-page` — update page content or properties
- `mcp__...notion.notion-update-data-source` — update database schema if needed
- `mcp__...notion.notion-move-pages` — re-parents a page/database only. No position argument — cannot place a block at a specific spot. Used for monthly archival sub-page re-rooting, never for ordering.

**No block-reorder primitive exists.** `notion-update-page` exposes `update_content` (search-and-replace via `content_updates: [{old_str, new_str}]`), `insert_content` (prepend/append at literal page start/end only), and `replace_content` (full rewrite) — nothing that reorders an existing block in place. Every "move to the top," "reorder," or "insert at position X" instruction below targets a position relative to other content (below the header callout, above earlier toggles, inside a specific toggle's children) — never the page's literal start/end — so `insert_content` doesn't apply; the mechanism is always: re-fetch the current markdown → locate the exact block's string → one `update_content` call whose `old_str`/`new_str` pair removes the block from its old spot and reinserts it at the new spot **in the same call**. Splitting removal and insertion across two calls (or a `new_str` that momentarily omits the child page/database) trips the `allow_deleting_content` safeguard, since Notion sees a referenced child page vanish from the content. Verified live 2026-07-07.

## Flow — writing today's page (Mode 1)

Called after the matcher has produced the ordered list of items.

### Step 1 — Confirm Preferences at bottom of parent

Fetch the parent page block tree. Confirm the `Preferences` `child_page` block is the last block on the parent body. If not, move it to the end via the atomic `update_content` mechanism (see Tools used note above): one call whose `old_str`/`new_str` removes the `Preferences` block from its current spot and reinserts it after the last other block, in the same call.

### Step 2 — Resolve / create the Year toggle block

- Compute current `YYYY` (4-digit, e.g., `2026`).
- Fetch the parent's body block tree.
- Look for a Heading-1 toggle block (`heading_1` with `is_toggleable: true`) on the parent body whose plain-text content is exactly `YYYY` (no prefix, no suffix).
- If absent: insert one immediately below the header callout, above any earlier Year toggles. Via the atomic `update_content` mechanism: one call whose `old_str` matches the header callout's markdown (plus whatever currently follows it) and whose `new_str` is the same text with the new toggle block inserted right after the callout. Default state: expanded.
- If present: reuse — never create a duplicate. If multiple match, log to run-log, use the topmost, leave the duplicates alone for PM review.
- **Legacy detection:** if a sub-page (`child_page` block) titled `YYYY` exists on the parent body instead of a toggle block, log to run-log as `Legacy Year sub-page detected: <year>`. Create the new toggle block in the correct position; the legacy sub-page is left untouched (no auto-migration).

### Step 3 — Resolve / create the Month toggle block

- Compute current `Month` spelled out (e.g., `April`).
- Fetch the Year toggle's children blocks.
- Look for a Heading-2 toggle block (`heading_2` with `is_toggleable: true`) inside the Year toggle whose plain-text content is exactly `Month` (spelled out, no year suffix).
- If absent: insert one at the TOP of the Year toggle's children, above earlier Month toggles. Same atomic `update_content` mechanism as Step 2 — `old_str` matches the Year toggle's opening + its current first child, `new_str` inserts the new Month toggle between them, in one call. Default state: expanded.
- If present: reuse — same dup + legacy rule as Year.

### Step 4 — Create today's dated sub-page

- **Title:** today's date in "DD Month YYYY" format. Example: `25 April 2026`.
- **Icon:** 📅
- **Notion-tree parent:** the parent page (use `notion-create-pages` with `parent` = `DEFAULT_NOTION_PARENT_PAGE_ID`). The dated sub-page is a direct child of the parent in the Notion sidebar tree.
- **Body-block placement on parent:** insert the resulting `child_page` block at the TOP of the Month toggle's children (above older dated `child_page` blocks from the same month). `notion-create-pages` lands the new page at the bottom of whatever it's appended to by default — use the atomic `update_content` mechanism to relocate its `child_page` block: one call whose `old_str` matches the Month toggle's opening + current first child and whose `new_str` inserts the new `child_page` block between them.
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
   - One line: `N items for your morning. X sub-tasks, Y flags. <M signals filtered — see Run Log if you want to audit>.`

4. **Inline Morning Queue database**

   > **HARD RULE — the queue is a Notion DATABASE, never page-body markdown. NON-NEGOTIABLE.**
   > The Morning Queue MUST be built as a real inline database (`notion-create-database` + one DB row per item). Rendering items as Heading / paragraph blocks in the dated page body — e.g. `### Row 1 — Hand off…`, `**Project:** …`, `**Triggered by:** …` written straight into the page content — is **FORBIDDEN** and constitutes a **FAILED run**, not a stylistic variant. Mode 2 reads the `Status` column and the `Ready for Execution` toggle off DB rows; a markdown dump leaves Mode 2 with nothing to iterate, so the queue is dead even if the PM flips the toggle.
   > There is **NO fallback** to inline markdown if the database flow feels longer, the `notion-create-pages` content path looks simpler, or the DB tool errors. If `notion-create-database` fails, retry once, then abort per the Error-handling table — do NOT route the rows into the page body to "save the run". A markdown queue is worse than no queue: it looks complete while being non-executable, and it silently breaks Mode 2.
   > **Mandatory tool sequence (every Mode 1 run, in this order):** (a) `notion-create-database` on the dated page → capture `morning_queue_db_id`; (b) per surviving item, a DB-row create against `morning_queue_db_id` (Step 6.1) → capture each `row_id`; (c) per item, `notion-create-pages` for the row-detail sub-page (Step 6.2). The long per-row content (Task Brief, Triggered-by, Sources, Recommended Action reasoning, AI Notes) lives ONLY on the row-detail sub-page — never on the dated page body. If `notion-create-database` did not fire this run, the queue was NOT written, regardless of what the dated page looks like. The Step 6.5 gate verifies this before the run is allowed to report `OK`.

   - Create via `notion-create-database` with `parent.page_id` = this dated page. **Unverified:** the tool schema exposes no `is_inline` / placement flag — nothing in its signature confirms this produces an inline database block versus a full-page one. Before relying on this in production, create a test database this way and `notion-fetch` the dated page to confirm it renders inline in the block tree rather than as a separate full-page object.
   - Schema per `schemas/morning-queue-database.md` — 10 columns, all visible in the default view, in order: Summary, AI Notes, Orbit Task Link, Project, Recommended Action, Recommended Assignee, Outcome, PM Notes, Source Systems, Status.
   - `Orbit Task Link` is Notion's native URL property — single clickable URL per row. For `Create subtask` / `Reopen subtask` / `Hand off parent task` rows it holds the parent task's Orbit URL; for `Flag` rows it holds the bound Orbit task URL, or `—` / empty when no Orbit task is bound (e.g., Gmail-only flag); for `Create parent task` rows it holds `—` (the parent doesn't exist yet). The sub-task URL that Mode 2 creates is written into `Outcome` only — never into `Orbit Task Link` (which keeps the parent reference stable through the row's lifetime).
   - `Project` format is `<Project Name> (#<project_number>)` — matcher supplies both name and Orbit user-visible project code from `get_project_details.project_number` (NEVER the internal `id` field, per SKILL.md non-negotiable #22). Maintenance / Ad-hoc style projects append a type suffix (`— Maintenance`).

### Step 5.5 — Enforce output gating (defense in depth)

Before iterating over the matcher's output array, scan it once and verify every item conforms to the 5-verb rule from `synthesis/matcher.md` Output gating + Job 4. The `summary` field is topic-style (NOT verb-prefixed per the rolled-back rule); the verb lock applies to `recommended_action` only.

Per-verb required fields (the writer rejects malformed rows by moving them to `filtered_signals`, NOT by trying to repair them):

- **`Create subtask`** — MUST carry `parent_task_id`, `task_title`, `assignee_id`, `proposed_orbit_body`, `work_type`. Recommended assignee may be `—` only when an `Uncertain:` AI Note explains why.
- **`Reopen subtask`** — MUST carry `parent_task_id`, `existing_subtask_id`, `existing_subtask_title`, `new_work_description`, `work_type`. May carry `last_dev_user_id` (or null when Job 5.5 flagged inactive-dev edge — in which case AI Notes carries the `Uncertain:`).
- **`Hand off parent task`** — MUST carry `parent_task_id`, `recommended_assignee_user_id` (the pool leader), `work_type` (which must be in `{AUDIT, QUOTE, SEO, DESIGN, CONTENT, BA}`). MUST NOT carry `task_title` or `proposed_orbit_body`.
- **`Flag`** — MUST carry `pm_next_step`. MUST NOT carry `parent_task_id`, `task_title`, `assignee_id`, or `proposed_orbit_body`.
- **`Flag` with `fc_row_type: stale_work_ping`** — additionally MUST carry the structured fields `stale_task_title`, `stale_assignee_name`, and `last_movement_excerpt` on the row candidate. The gate checks for the presence of these three fields directly — it does not parse the Summary/Task Brief prose for them. This mirrors the tracker-side generic-ping ban (`synthesis/fixed-cost-state.md` § FC-4) as defense in depth: any field missing/null → do NOT render the row (drop it, don't repair it), and write run-log audit line `ping dropped at render — insufficient specifics`. The `latest_signal_anchor` requirement above (rule 24) applies to this row exactly as to any other row, including `fc_row_type: follow_up_reminder` — no v3 carve-out from the Triggered-by gate.
- **`Create parent task`** — MUST carry `project_id`, `task_title`, `proposed_orbit_body`, `assignee_id == PM_user_id`, `parent_task_id == null`.

Cross-row consistency checks:
- Every item's `recommended_action` text must start with one of the five locked verbs.
- Every item must carry a `task_brief` (Job 7b output), non-null, ≤ 600 chars.
- Every item's `project` field (if not `Standalone`) must render as `<name> (#<project_number>)` where `project_number` is the string Orbit user-visible code, not the integer `id`.
- Every item must carry `latest_signal_anchor` (non-null, all sub-fields populated). A null anchor means Job 7 should have dropped the row to `filtered_signals` (`filter_reason: no_trigger_signal_identifiable`) rather than emit it; if one slips through, route to `filtered_signals` here with `filter_reason: writer_gating_caught_missing_anchor` and surface in the Run Log.

Any item failing these checks is moved from the items array to the `filtered_signals` array with `filter_reason: writer_gating_caught_drift` before row creation. Log to Run Log. This is a safety net; the matcher should have already prevented these — but if it didn't, the writer catches the drift rather than polluting the queue.

### Step 5.6 — Render the row-detail Action Block

For each item that survives Step 5.5, the writer renders the row's detail page with the Action Block callout at the very top (per `schemas/row-detail-page.md`). The callout is plain text — no emoji, no decorative glyphs. The verb word (`Create subtask`, `Reopen subtask`, `Hand off parent task`, `Flag for PM`, `Create parent task`) opens the first line in bold; subsequent lines carry structured fields. The action callout appears BEFORE the H1 Summary. The PM should not have to scroll past Sources or Recommended Action to find the proposed task title and assignee — that information is in the top callout.

### Step 5.7 — Fixed-Cost Pulse toggle

Below the Morning Queue database — the first block after the database (the Slack-handles reference block keeps its usual place above the database), render ONE toggle block:

`Fixed-Cost Pulse — <N> projects, <M> with overnight movement`

**Source selection.** When the run carries the Orbit fixed-cost activity loop's output
(`fixed_cost_activity_loop_ran: true` — shadow mode, the monthly re-audit, or a
Gmail-failure fallback), compose the Pulse from the Orbit collector's `activity_summary[]`
exactly as below. Otherwise (mail-primary, ordinary day), compose it from the Gmail
collector's `mail_activity_summary[]` instead: movement lines are built from its
per-project event-type counts, still in rule-22 project format; tracked projects absent
from the rollup (no parsed mail this run) collapse into the same no-movement tail line.
Never sum the two sources on a day both are present — that double-counts; the Orbit copy
wins (same precedence as the matcher's shadow dedup, `synthesis/matcher.md` § Job 4c).

Inside, one line per project WITH movement (from whichever source this run selected;
counts only, no deep-reads), rule-22 project format:

`1828 Eng Landing (#15942) — 2 tasks completed, 3 comments, due Sep 10`

Include only non-zero facts (skip "0 comments"). Then ONE collapsed tail line for the rest:
`<K> projects — no overnight activity`. Zero tracked projects → render the toggle with the
single line `No fixed-cost projects tracked` (or the header shows `0 projects`).
`discovery_mode: fallback-sweep` this run → append ` · discovery: fallback` to the toggle
title so the PM can see degraded discovery at a glance.

The Pulse is passive visibility (spec §4.5): it never contains action verbs, assignees, or
checkboxes — actionable items are queue rows, not pulse lines.

### Step 6 — Populate each row

For each item in the matcher's output array (after Step 5.5 gating):

**Step 6.1 — Create the database row.** Against `morning_queue_db_id` from Step 4 (NOT as page-body blocks), with (in schema DDL order — Summary, AI Notes, Orbit Task Link, Project, Recommended Action, Recommended Assignee, Outcome, PM Notes, Source Systems, Status):
   - `Summary` (title) — matcher's topic-style one-liner (NOT verb-prefixed; verb lives in `Recommended Action` only)
   - `AI Notes` — any uncertainty flags, split reasoning, matcher Job 6 delegate reasoning, or Job 5.5 last-dev edge cases; for priority-lane rows always populated with `<AM> put this on your plate overnight, due today. Proposed delegate: <name> (<reason>).`; empty otherwise
   - `Orbit Task Link` (URL property) — parent task URL for `Create subtask` / `Reopen subtask` / `Hand off parent task` rows; bound task URL for `Flag` rows that have one; `—` or empty for Flag rows with no Orbit task and for `Create parent task` rows
   - `Project` — `<name> (#<project_number>)` (user-visible string code, not internal `id` per SKILL.md #22) or `Standalone`
   - `Recommended Action` — short phrase starting with one of the five locked verbs (`Create subtask`, `Reopen subtask`, `Hand off parent task`, `Flag`, `Create parent task`)
   - `Recommended Assignee` — name + role + short reason; `—` for advisory items
   - `Outcome` — empty (filled by Mode 2; Mode 2 writes new sub-task URL or reassignment confirmation here, NOT into Orbit Task Link)
   - `PM Notes` — empty
   - `Source Systems` — multi-select of which sources contributed (Orbit, Gmail — primary; Fathom only if matcher Job 4b Pass 2 fetched enrichment for this row)
   - `Status` = `Recommended Action` (default)

**Step 6.2 — Populate the row-detail sub-page** per `schemas/row-detail-page.md` (block order is load-bearing — see schema). This sub-page is the ONLY place the long per-row content lives:
   - Top callout — Action Block (no heading) — per the 5 verb variants in `schemas/row-detail-page.md` Top callout section.
   - `Summary` heading + matcher's topic-style summary
   - **`Task Brief` heading + Triggered-by line + matcher's `row.task_brief` paragraph (Job 7b output)** — placed BETWEEN Summary and Sources so the PM reads what the work is + what's new before scanning citations. The Triggered-by line is THE FIRST LINE of this block (above the Job 7b narrative), rendered from `row.latest_signal_anchor`: bold-label + source + author + timestamp + 1-line excerpt + source link. Format per `schemas/row-detail-page.md` Task Brief section. Required — Step 5.5 already rejects rows with a null anchor, so render unconditionally.
   - `Sources` heading + full citations per `writers/source-citation.md`. Two cases:
     - **Job 4b Pass 1 (Gmail → Orbit cross-link).** When `context_signals[]` is populated on this row, render each linked Gmail signal as an H2 subsection under Sources. High-confidence cross-links go under the primary Sources list; weak cross-links (match_strength="weak") go under an H2 `Possible context` subsection.
     - **Job 4b Pass 2 (Fathom enrichment).** When `enrichment.fathom` is populated on this row (the matcher fetched a meeting context from `collectors/fathom.md`), render it under an H2 `Fathom enrichment` subsection per `schemas/row-detail-page.md`. Include the trigger phrase that caused the fetch, the scoped summary excerpt, relevant action items, attendees, and the recording link. Skip this subsection entirely if `enrichment.fathom` is null.
   - `Recommended Action` heading + reasoning (assignee-pick logic for Create/Reopen, work_type → pool routing for Hand off, signal-weighing for Flag)
   - `Proposed Orbit Task Body` heading + 6-section body (Create subtask + Create parent task only)
   - `Proposed New-Work Comment` heading + matcher's `new_work_description` (Reopen subtask only)
   - `PM Next Step` heading + paragraph (Flag only)
   - `Proposed Handoff` heading + plain-language message (Create subtask / Reopen subtask / Hand off parent task)
   - `AI Notes` heading (if any notes). For priority-lane rows, AI Notes carries the matcher's `<AM> put this on your plate overnight, due today. Proposed delegate: <name> (<reason>).` line.
   - Bottom toggle: `Reference Context for the Skill — working memory, not for review`, containing the full raw signals and pod inference reasoning. For a row carrying `fc_row_type: follow_up_reminder`, this toggle additionally includes a machine-readable `ask_key: <value>` line (the row candidate's `ask_key` field, `synthesis/fixed-cost-state.md` § FC-3) — the PM never needs to read or understand it; Mode 2's note-interpreter reads it back off this toggle when resolving/snoozing the ask (see § Flow — Mode 2 ledger touch below).

### Step 6.5 — Mandatory post-write structure verification (assertion gate)

After all rows and row-detail pages are written, the writer **re-fetches the dated page and proves the queue is a real database before the run is allowed to report `OK`.** This is the gate that catches the markdown-dump failure mode (the writer skipping `notion-create-database` and dumping rows as page-body headings). Do NOT skip this gate even when the writer "knows" it created the database — verify against Notion, not against intent.

1. `notion-fetch` the dated page block tree.
2. Compute and record the following assertions into a `notion_write_assertions` object (passed to `writers/run-log.md` in Mode 1 Step 4e):

   | Assertion key | Pass condition |
   |---|---|
   | `db_created` | Exactly one `<database>` block titled `Morning Queue` exists on the dated page body. |
   | `db_row_count_matches` | The database's row count equals `items_written` (the count of items that survived Step 5.5 gating). |
   | `row_detail_pages_created` | A row-detail sub-page exists for every DB row (count equals `items_written`). |
   | `no_markdown_row_dump` | The dated page body contains **no** queue-row content rendered as heading/paragraph blocks. Heuristic: no body Heading block whose text matches `Row \d+ —`, and no body paragraph leading with `**Project:**` / `**Triggered by:**` / `**Recommended action:**`. The only headings allowed on the dated page body are the structural ones (Morning Queue, Slack handles reference, Fixed-Cost Pulse toggle, Pod/AM digest anchors added in Mode 2). |
   | `single_db` | No duplicate `Morning Queue` database on the page (Idempotency rule). |

3. **On any assertion FAIL:** the run status is `Failed` (not `OK`, not `Partial`). 
   - If `db_created == false` or `no_markdown_row_dump == false`: this is the markdown-dump bug. Retry the database build **once** — create the database, move the row content off the page body into DB rows + detail pages, and delete the offending page-body blocks. Re-run this gate.
   - If the retry still fails, abort per the Error-handling table (`Database creation fails` / `Queue rendered as page-body markdown` rows) — send the PM the failure email, and pass the failed assertions to `writers/run-log.md` so the Run Log row records `Status = Failed` with the specific assertion(s) that failed. **Never report `OK` with a failed structure assertion** — a falsely-green Run Log is what let the 04 June markdown-dump run pass silently.

4. **On all assertions PASS:** proceed to the Slack-handles reference block and exit normally. The passing assertion set is still recorded in `notion_write_assertions` and surfaced in the Run Log (so a healthy run leaves positive evidence the DB was built, not just an absence of errors).

## Flow — Mode 2 ledger touch (fixed-cost resolve/snooze notes)

Fires when Mode 2's note-interpreter (`synthesis/note-interpreter.md` § fixed-cost
resolved/snooze sections) emits an `fc_ledger_update` action_plan against a Flag row
carrying `fc_row_type: follow_up_reminder`. This is Mode 2's ONLY write to the Fixed-Cost
Registry.

1. Locate the Fixed-Cost Registry sub-page under the Notion parent.
2. Find the Client-Ask Ledger row matching the action_plan's `ask_key`
   (`#<project_number>|<Asked on>|<first 40 chars>`). The note-interpreter reads this
   `ask_key` from the triggering row's detail page — the `ask_key: <value>` line inside its
   bottom Reference Context toggle (`schemas/row-detail-page.md` § Bottom toggle), written
   there at Mode 1 render time (§ Step 6.2 above). **Fallback** — if that toggle is missing
   the line (older row, render drift): recompute `ask_key` by matching the row's `Project`
   against the ledger's `Project` column and the row's ask text against the ledger's `Ask`
   column (topic-match, same judgment as FC-1's resolution scan). If more than one open
   ledger row plausibly matches, do NOT guess — set `confidence: low` and flag Uncertain
   rather than touching the wrong ask.
3. Apply the action_plan:
   - **Resolve** (`status: Resolved-manual`) — set `Status` → `Resolved-manual`, `Resolved by`
     → the action_plan's `resolved_by` (the PM note text), `Resolved on` → the action_plan's
     `resolved_on` (today, YYYY-MM-DD).
   - **Snooze** (`snooze_until_shift`) — bump the ask's `fc_state` `last_reminder` forward by
     the requested period and mirror the new value into the ledger row's `Last reminder`
     column. Do not touch `Status` or `Resolved on`.
4. Write the queue row's `Outcome`: `Done — ask closed by PM` (resolve) or `Done — snoozed`
   (snooze). The row `Status` flips to `Done` per the normal Step 7 rule in
   `modes/mode-2-execution.md`; the row's `Recommended Action` stays `Flag` (no verb change).
5. **Write failure** (ledger row not found, Notion API error): log an Incident, and the queue
   row's Outcome notes the failure (`FAILED — couldn't update Client-Ask Ledger row for
   <ask_key>. [error]`). The run continues — one ledger-touch failure does not abort Mode 2.

## Flow — updating rows after Mode 2

For each row Mode 2 executed:

- Flip `Status` to `Done` (or leave at prior state if failure; see `modes/mode-2-execution.md`)
- Write the `Outcome` column with the concise result string
- If the row produced a team / AM handoff draft (the default path for every Create-subtask row), append a `Handoff draft (copy + paste)` block to the row's detail page under the existing Outcome content. Block contents:
  - First line — `Audience: <Team | AM | Client>` and `Recipient: <name> (<canonical email>)`
  - The full plain-language draft body produced by the matcher's `proposed_handoff` and applied through `writers/plain-language.md`
- No other changes to row detail page content — source-of-truth stays intact.

## Flow — Slack handles reference block on the dated page (end of Mode 1)

Append a small reference block to today's dated page, **directly above the Morning Queue inline database**, listing the Slack handles of every recipient the day's rows could potentially auto-send to via `executors/slack.md`. Purpose: (a) gives the executor a single in-page lookup to resolve handles at Mode 2 send time without re-fetching Preferences / Pod Matrix; (b) lets the PM glance at every relevant handle when deciding whether to leave a `send` note on a row.

Sourcing for each entry:

- **Team members** — for each row's `recommended_assignee`, resolve the Slack handle from the Pod Matrix (`references/pod-matrix.md` carries handles alongside names and Orbit user_ids). If a delegate is not in the matrix, fall back to `slack_search_users` against the delegate's canonical email at render time; cache the resolved handle for the run.
- **AMs** — for each AM matched to a row's project, read the `Slack handle` field from the Preferences AM entry. Missing field renders as `(no handle — Notion draft only)`.

Block shape:

```
─────────────────────────────────
## Slack handles for today (reference for executor + PM)

Team:
  • Vijay Patel — @vijaypatel
  • Atul Mehra — @atulmehra
  • Hitesh Asnani — (no handle — Notion draft only)

AMs:
  • Caitlin Sims — @caitlins
  • Ellen Thomas — (no handle — Notion draft only)
```

The block uses a Divider + Heading-2 anchor (`Slack handles for today (reference for executor + PM)`) for same-day rerun idempotency, exactly like the Pod Daily Task and AM Daily Ping blocks. On Mode 1 rerun, find the anchor and replace the body in place.

This block is purely a reference — the writer never reads back from it. The executor reads from it as a fallback when `slack_search_users` returns no match (preferring the pre-resolved handle to a runtime search).

If a row carries zero potential Slack send targets (priority-lane row where the delegate is not in the matrix AND has no canonical email match, plus the AM has no handle), the block omits that row entirely. If every row falls into that case, skip the block — log `slack_handles_block_empty` to the Run Log.

## Flow — refreshing the Fixed-Cost Registry (end of Mode 1)

After the dated page is written (registry data already in hand from the collector +
Job 4c):

1. Update the header callout: count, `Last refresh`, `Discovery` mode. Bump
   `Last successful discovery` ONLY when the collector's returned `discovery_succeeded`
   flag (`collectors/orbit.md` § Fixed-cost extension) is `true` — a guard pass or a
   completed Appendix-A sweep. **Never bump it on `discovery_succeeded: false`** — that
   covers both the guard-trip fallback that merely reused the `registry_snapshot` and a
   fully-skipped lane; both report `discovery_mode: fallback-sweep` (or absent) but neither
   is a real discovery, so keying the bump on `discovery_mode` alone would let a bare
   snapshot reuse falsely renew the ≤7-day freshness check every run. `Last refresh` bumps
   on every run regardless.
2. **Activity source / Clean audits (spec §11.3).** Render the header line `Activity
   source: <value> · Clean audits: <n>` from `fc_output.fc_state_patch`: apply
   `activity_source_next` / `clean_audits_next` when present, otherwise carry the current
   values forward. When the applied patch CHANGES `Activity source` (shadow → mail-primary
   cutover, or mail-primary → shadow revert), append a flip audit string to the audit
   strings returned to the calling mode (same pattern as the suppressed-reminder /
   dropped-ping audit strings — this writer NEVER touches the Run Log database;
   `writers/run-log.md` is its only writer). The run-log writer renders it as a
   decision-trace line at Mode 1 step 4e, format `[subject: Fixed-Cost Registry] →
   [action: "activity-source flip"] → [reason: <shadow → mail-primary | mail-primary →
   shadow>, <clean_audits reached 2 | mail_coverage_gap on re-audit>]`.
   A pre-v4 page missing the line entirely → write it as `Activity source: shadow · Clean
   audits: 0` this refresh (schema default).
3. Reconcile the Tracked Projects table per schemas/fixed-cost-registry.md rules
   (`filter` rows synced to today's discovery set; `manual` rows never auto-removed —
   annotate ` — verify: closed?` when appropriate). For any Tracked Projects row
   appearing for the first time this run, set `Added on = today` and fill `Client`
   from `client_name` + `sub_client_name`.
4. Apply the `fc_output.ledger_mutations` + `fc_output.fc_state_patch` returned by
   matcher Job 4c (held in memory since synthesis): append FC-2 asks; set FC-1/FC-3
   resolutions (`Status`, `Resolved by`, `Resolved on`); mirror `Last reminder` dates.
5. Rewrite the `fc_state` block (asks/pings timestamps) inside its toggle.
6. Any write fails after retries → Incident, continue (registry is a mirror; next run
   heals it).

## Flow — Mode 2 PM self-summary callout (end of Mode 2)

After every row is processed and per-row Handoff drafts are appended, write a single Notion **callout block** at the very top of today's dated page summarizing the Mode 2 run. This replaces what used to be a Slack DM to the PM. The PM is already opening Notion to review approved rows — the summary lives where their eyes are.

Block format:

- Callout block with icon `✅` if every approved row executed successfully, `⚠️` if one or more failed.
- First line: `Mode 2 ran at <HH:MM IST>. <N> approved, <K> executed, <F> failed, <S> skipped.`
- If priority-lane rows existed, second line: `Priority lane: <N_priority> sub-tasks created under AM-handed parents.`
- Third line (only if F > 0): `Failures: <one-line per failure with row title + reason>.`
- Last line: link to today's dated page (self-reference is fine — the callout anchors the PM's eye).

Position: AT THE TOP of today's dated page, ABOVE the `Ready for Execution` toggle. PM sees it the moment they open the page.

Idempotency: on a same-day Mode 2 rerun, find any prior summary callout (first block on the page if it carries the `✅` or `⚠️` icon AND opens with `Mode 2 ran at`) and replace it in place. Do not stack callouts.

## Flow — appending the Pod Daily Task block (end of Mode 2)

After the Mode 2 summary callout is written and the per-row `Handoff draft (copy + paste)` blocks are appended, append a **single Pod Daily Task block** to today's dated page. The PM copies this whole block in one shot and pastes it into whatever channel they use for daily team delivery (direct message, email, in-person, etc.).

**Placement:** at the very bottom of the dated page, BELOW the inline Morning Queue database. Never above it, never inside the database, never on the row detail pages.

The block has two parts: a fixed **anchor** (skill-managed, supports rerun detection) and a customizable **body** (rendered from the PM's `pod_block_layout` DSL). Both are described below.

### Step 1 — Read templates, switches, and layout from Preferences

Pull the **Pod Daily Task Block** section from the cached Preferences page (already loaded in Mode 2 Step 1). Read every field:

- `pod_daily_task_enabled` (bool) — if `false`, skip the entire flow, no run-log entry needed.
- `pod_daily_task_destination` (string, reference only — free-text reminder of where the PM pastes the block; never used to send).
- `caption_template` (string).
- `date_label_template` (string).
- `task_heading_template` (string).
- `task_line_template` (string).
- `pod_block_layout` (string — the block layout DSL, stored in a Notion code block).

If any template or layout field is missing or empty on the Preferences page, fall back to the documented default for that field — see `schemas/preferences-page.md` — Pod Daily Task Block. Each field is resolved independently.

**Token substitution rule (applies to every rendered string).** Substitute every recognized `{token}` with its value. Leave unrecognized tokens verbatim in the output — never strip them, never error. This makes typos visible to the PM in Notion.

**Template-field reference rule.** Inside the layout DSL, `{{field_name}}` (double braces) resolves to the rendered value of the named template field (its content is itself token-substituted in the current context first). Single-brace `{token}` always means a token; double-brace `{{name}}` always means a template field reference.

### Step 2 — Qualify rows

Include a row only if Mode 2 executed an Orbit subtask create for it in this run. Specifically:
- Include rows whose Outcome string records a new Orbit subtask ID (`Subtask #... created under parent #...`).
- Exclude `Flag` rows entirely (no Orbit write happened).
- Exclude rows with `Status = Skip. No Action Needed`, `Status = Recommended Action` (PM didn't approve), `HELD`, `FAILED`, and any row whose Outcome doesn't record a subtask create.
- If zero rows qualify, do NOT append an empty block (anchor included). Skip silently and log `pod_digest_skipped: no_qualifying_rows` in the run-log.

Preserve matcher order for the included rows.

### Step 3 — Parse the layout DSL

Parse `pod_block_layout` into an in-memory tree of block specs. Rules per the Preferences schema:

- One block per line, format `<type>` or `<type>: <content>`.
- Two-space indentation = nesting. Establish each block's parent by indentation level.
- `for_each_task:` marks a sub-tree that is repeated per qualifying row.
- `#`-prefixed lines and blank lines are ignored.
- Validation:
  - Unknown block type → log `pod_layout_unknown_block: <type>`, skip the line, keep parsing.
  - Indentation under a non-nestable block (`heading_1/2/3`, `paragraph`, `quote`, `divider`) → log `pod_layout_illegal_nesting: <parent>`, render the child as a sibling at the parent's level, keep parsing.
  - Malformed indentation (e.g., 3-space indent, or jumping levels) → log `pod_layout_parse_failed: <reason>` and fall back to the default layout for this run.
  - No `for_each_task:` directive anywhere in the parsed tree → render the layout as-is (no task list) and log `pod_layout_no_task_loop`. The PM self-summary closing line changes to: `Pod Daily Task block rendered but has no task loop — see preferences.`
- A successfully parsed empty layout (zero block lines after comments) → fall back to default.

### Step 4 — Resolve the per-row token map

For each qualifying row, capture the executor-return values: `client_name`, `project_name`, `task_title`, `task_id`, `orbit_task_url`, `assignee_first_name`, `assignee_full_name`. Build the date token map once for the run (`{date_dd}`, `{date_mm}`, `{date_yyyy}`, `{date_month_name}`, `{date_iso}`).

**Available tokens by context.** Date tokens are always available. Task tokens are available ONLY inside a `for_each_task:` sub-tree. If a task token appears outside that sub-tree, leave it verbatim (so a PM mistake surfaces in Notion rather than silently rendering empty).

### Step 5 — Emit blocks: anchor first, then body

Emit blocks in this order. The anchor is fixed and never reads from the DSL.

**Anchor (always emitted, in this order, NEVER customizable):**

1. **Divider block** — separates the digest from the morning queue database above.
2. **Heading-2 block** — plain text exactly `Pod Daily Task — copy + paste`. This text is the rerun match key. Do not let any template or DSL override it.

**Body (rendered from the parsed `pod_block_layout` tree):**

Walk the parsed tree depth-first. For each block spec:
- Resolve template-field references (`{{name}}`) by looking up the named field's current value and substituting tokens in it in the current context.
- Resolve tokens (`{name}`) in the resulting string, then in any of the spec's other text properties.
- When the walker enters a `for_each_task:` sub-tree, iterate over the qualifying rows in matcher order. For each row, emit a fresh copy of the sub-tree's blocks with task tokens bound to that row's values.
- Map each block spec to its Notion block type. For toggle blocks, descend into the spec's children and emit them as Notion `children` of the toggle. For `to_do`, `bulleted_list_item`, `numbered_list_item`, and `callout`, children are also legal — emit them as Notion children. For non-nestable types, ignore any (already-warned-during-parse) children.

The default layout produces, for a 3-task day on 11 May 2026:

```
─────────────────────────────────
## Pod Daily Task — copy + paste
Copy everything inside the date toggle below and paste into your pod's daily task channel (set the destination in Preferences to whatever you use). Each line is one task you assigned today.

▼ 11/05/2026
  ## Wick MarketingWick Marketing PHP Upgrade16315
  ☐ https://app.whitelabeliq.com/93640173/project/45021837663/96642 - Aagna

  ## Markit360 llcTru-ConnectTru-Connect AI Audit16380
  ☐ https://app.whitelabeliq.com/93640173/project/25976347756/96632 - Hitesh

  ## Nex-TechNex Tech Google Data Studio16341
  ☐ https://app.whitelabeliq.com/93640173/project/20748967704/96442 - Manan
```

### Step 6 — Idempotency

**Block-level (rerun same day).** If today's dated page already has a `Pod Daily Task — copy + paste` Heading-2 anchor from a prior Mode 2 fire:
- Find the anchor (the Heading-2 with that exact plain-text).
- Treat every block between the anchor and the next anchor (or the end of the page) as the digest's body. Delete those body blocks.
- Re-emit the body from the freshly-parsed layout. The anchor (Divider + Heading-2) stays in place — do not delete or recreate it.
- If the layout DSL changed between runs (different block types, different nesting), the new layout fully replaces the old body. Tick-state preservation (below) is best-effort across structural changes.
- If multiple anchors exist on the page (drift — should not happen), use the last one and log it in the run-log.

**Checkbox state preservation.** Before deleting old body blocks, scan them for to-do blocks whose rendered text contains an Orbit task URL (`https://app.whitelabeliq.com/.../<task_id>` pattern). Build a `set<ticked_task_id>` of ones that were checked. After emitting the new body, walk the freshly-emitted to-do blocks and, for any whose rendered text contains a URL with a `task_id` in the ticked set, flip them to checked. URLs not in the qualifying set are dropped naturally because they don't appear in the new body. New URLs default unchecked.

This preservation works only if the PM kept an Orbit URL token in their `task_line_template` (or some other template the to-do references). If the layout has no URL anywhere, tick state cannot be matched across runs and all to-dos render unchecked — log `pod_digest_tick_preservation_skipped: no_task_id_in_layout`.

### Step 7 — Tooling and failure handling

Use `notion-update-page` block-append for first-time creation. Use targeted child-block delete + append for body refresh on rerun. No new pages are created. Batch the body appends in one `notion-update-page` call where the block count permits.

If any part of the digest append fails, log to run-log and continue. Do NOT block the PM self-summary — the digest is a copy-aid, not a blocker. The PM self-summary message appends one line on failure: `Pod Daily Task block failed to render — see run-log.`

## Flow — appending the AM Daily Ping block (end of Mode 2)

Runs immediately after the Pod Daily Task block flow completes. Same shape: fixed anchor + body rendered from a layout DSL (`am_block_layout`). Lives directly below the Pod Daily Task block on the dated page.

**Placement:** at the bottom of the dated page, BELOW the Pod Daily Task block (or where the Pod block would have been if it was skipped). Never above the Morning Queue database, never on row detail pages.

### Step 1 — Read AM ping config from Preferences

Pull the **AM Daily Ping Block** section. Read every field:

- `am_ping_enabled` (bool) — if `false`, skip the entire flow, no run-log entry needed.
- `quiet_ams` (list of AM names to skip).
- `am_ping_caption_template` (string).
- `am_heading_template` (string).
- `am_ping_body_guidance` (string — free-text guidance, NOT a token-substituted template).
- `am_block_layout` (string — layout DSL).

Fall back to defaults per `schemas/preferences-page.md` for any missing or empty field. Each field resolved independently.

### Step 2 — Qualify and group rows by AM

Walk the same set of rows used by the Pod Daily Task qualification (Step 2 of that flow): subtask-create rows only, Flag rows excluded.

- For each qualifying row, determine which AM owns it by matching the row's `Project` value against the AM-to-Projects associations in the Preferences Account Managers section. A row may map to zero AMs (no AM owns the project) or one AM. Rows that map to zero AMs are not part of any per-AM ping.
- After grouping, drop any AM listed in `quiet_ams`.
- After grouping, drop any AM with zero qualifying rows.

If zero AMs remain after grouping, do NOT append the block (no anchor emitted). Log `am_digest_skipped: no_qualifying_ams` in the run-log.

Order AMs alphabetically by `am_name` for deterministic rendering across runs.

### Step 3 — Parse the AM layout DSL

Same parser rules as Pod Daily Task layout. The only difference: the loop directive is `for_each_am:` instead of `for_each_task:`. A `for_each_task:` directive inside `am_block_layout` is rejected at parse time with `am_layout_unknown_directive: for_each_task` and the line is skipped.

### Step 4 — Draft the per-AM body via `writers/plain-language.md`

For each AM in the grouped set:

1. Gather row context for the AM's qualifying rows: the row summaries, recommended actions, resolved assignees, project names, and any relevant details from the row detail pages (Sources, Proposed Orbit Task Body sections).
2. Build the AM context object: `{am_name}`, project list, task list, assignee list.
3. Call `writers/plain-language.md` with:
   - The current `am_ping_body_guidance` text as the drafting instruction.
   - The row context bundle.
   - PM tone samples from Preferences (for voice calibration).
4. Receive a 3-line draft body. This becomes the value of `{{am_ping_body}}` for this AM in the loop iteration.

**Plain-language enforcement.** The AM ping body passes through the same 4th-5th grade English screen as team handoff drafts ONLY for AMs marked in Preferences as needing it. By default, AM pings stay in normal professional English. The drafting call should pass an `audience: am` flag so the plain-language writer skips the simplification pass.

### Step 5 — Resolve the per-AM token map

Build the AM token map: `{am_name}`, `{am_first_name}`, `{am_last_name}`, `{am_email}` from Preferences, `{am_tasks_count}` and `{am_projects_csv}` from the grouped row set. Add the run's date token map for blocks outside the loop.

### Step 6 — Emit blocks: anchor first, then body

**Anchor (always emitted when at least one AM qualifies, NEVER customizable):**

1. **Divider block.**
2. **Heading-2 block** — plain text exactly `AM Ping Drafts — copy + paste`. This is the rerun match key.

**Body** — walk the parsed `am_block_layout` tree depth-first, same rules as Pod Daily Task layout. When the walker enters a `for_each_am:` sub-tree, iterate over the grouped AMs in alphabetical order; for each, emit the sub-tree with AM tokens and `{{am_ping_body}}` bound to that AM's draft.

The default layout produces, for a 2-AM day:

```
─────────────────────────────────
## AM Ping Drafts — copy + paste
One short ping per AM, wrapping today's work on their projects. DM each AM at their handle below. The skill never auto-sends these — copy and send yourself.

### Sarah Chen (@sarahc)
Two tasks picked up for Agency X today — Vijay is on the homepage revisions and Rohit is owning the QA pass.
Target: preview ready before Thursday's board review.
Will keep you posted.

### Caitlin Park (@caitlinp)
DigitalFirst's three action items from this morning's call are now in Orbit — Ravi has the homepage tweaks and Amit owns the analytics fix.
Ravi is targeting end of week for the first pass.
Ping me if priorities shift.
```

### Step 7 — Idempotency

**Block-level (rerun same day).** Same pattern as Pod Daily Task. Match by the anchor's exact Heading-2 plain-text `AM Ping Drafts — copy + paste`. Find the anchor, delete every block between it and the next anchor (or end of page), re-emit the body from the freshly-parsed layout. Anchor stays in place.

**Body regeneration on rerun.** The 3-line ping body is re-drafted on every Mode 2 fire because the underlying rows may have changed. No tick-state to preserve here (AM pings are paragraphs, not to-dos).

**If the rerun produces a different AM set** (e.g., a quiet AM was just added, or new rows surfaced new AMs), the body fully reflects the new set. Removed AMs are dropped.

### Step 8 — Tooling and failure handling

Same as Pod Daily Task. If any per-AM `writers/plain-language.md` call fails (timeout, MCP error), emit a placeholder paragraph for that AM: `Couldn't draft ping for {am_name} on this run — try rerunning Mode 2 or write manually.` Continue with the remaining AMs. Log each failure per AM. The PM self-summary appends one line on failure: `AM Ping Drafts block had errors — see run-log.`

## Flow — monthly archival (1st of month)

See `modes/monthly-archival.md` for the orchestration. Because every dated page is created inside its Year-toggle → Month-toggle nesting on the parent body, archival is **verification**, not migration. The Notion Writer's role on archival day:

1. Verify the previous-month Month-toggle exists inside its Year-toggle on the parent body, and that all dated `child_page` blocks for that month sit inside it.
2. If any dated `child_page` from a prior month is found at the parent body level outside any Month toggle OR inside the wrong Month toggle OR inside a legacy Year sub-page, log it in the run-log entry but DO NOT auto-move (placement drift implies manual edits — surface for PM review).
3. Verify Year-toggle ordering on the parent body: newest Year on top, descending. If drifted, reorder via the atomic `update_content` mechanism (Tools used note above) — one call per swap, `old_str`/`new_str` removing and reinserting the out-of-place toggle in the same call.
4. Verify Month-toggle ordering inside each Year-toggle: newest Month on top. Same mechanism if drifted.
5. Verify dated `child_page` block ordering inside each Month-toggle: newest date on top. Same mechanism if drifted.
6. Ensure `Preferences` `child_page` block is still the last block on the parent body.
7. Ensure `Run Log`, `Incidents`, and `Fixed-Cost Registry` `child_page` blocks sit immediately above `Preferences`, in that order, after all Year-toggle blocks.

`notion-move-pages` is used only when sub-pages (Day, Run Log, Incidents, Fixed-Cost Registry, Preferences) need to be re-rooted across parent pages — which should never happen in normal operation. It only changes a page's parent; it cannot reorder. All toggle-block and child-page reordering above uses the atomic `notion-update-page` `update_content` mechanism instead.

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
- Don't append a second Pod Daily Task block on the same dated page. Replace the existing toggle's children instead.
- Don't append a second AM Daily Ping block on the same dated page. Find the anchor and replace the body blocks below it.
- Don't append a second Fixed-Cost Pulse toggle on the same dated page. Find by its `Fixed-Cost Pulse —` title prefix and replace it in place.

## Error handling

| Failure | Behavior |
|---|---|
| Page creation fails | Retry once. If fails, abort Mode 1 with sent email to PM: `Couldn't write today's morning queue. [error]` |
| Database creation fails | Retry once. If fails, abort Mode 1 with sent email to PM: `Couldn't create the Morning Queue database. [error]`. **Do NOT fall back to rendering rows as page-body markdown** — abort instead. Record `Status = Failed` + `db_created: false` in the Run Log. |
| Queue rendered as page-body markdown (Step 6.5 `no_markdown_row_dump` or `db_created` FAILS) | This is a writer bug, not a Notion error. Retry the database build once: create the DB, migrate row content into DB rows + detail pages, delete the offending page-body heading/paragraph blocks, re-run the Step 6.5 gate. If still failing, abort with sent email to PM: `Morning queue was written as plain text instead of a database — aborted to avoid a non-executable queue. [assertion]`. Record `Status = Failed` + the failed assertion keys in the Run Log. Never report `OK`. |
| Row creation fails for a single item | Log in AI Notes on the summary block, continue with remaining rows |
| Block reorder fails during archival | Log and continue. Order drift can be corrected on the next fire. |
| `notion-move-pages` fails during archival sub-page reorder (Run Log / Incidents / Fixed-Cost Registry / Preferences) | Log and continue. |
| Year or Month toggle creation fails | Retry once. If still fails, abort Mode 1 with sent email to PM: `Couldn't create the Year/Month toggle block. [error]` — do NOT fall back to creating the dated page flat at the parent body level. |
| Pod Daily Task block append fails | Log to run-log. Continue. Append `Pod Daily Task block failed to render — see run-log.` to the PM self-summary. Do not abort Mode 2. |
| AM Daily Ping block append fails | Log to run-log. Continue. Append `AM Ping Drafts block failed to render — see run-log.` to the PM self-summary. Do not abort Mode 2. |
| Per-AM draft generation fails | Render a placeholder paragraph for that AM (`Couldn't draft ping for {am_name} on this run — try rerunning Mode 2 or write manually.`). Continue with remaining AMs. |
| Preferences page is missing | Route to `first-run-setup.md` |

## Performance

- Batch row creation where the tool supports it (up to 100 rows per call per Notion's limit)
- Create row detail page content in parallel for the 10-30 rows of a typical morning
- Target total Notion-write time: under 60 seconds for a 20-item morning

## What this writer does NOT do

- **Never renders queue rows as page-body markdown.** The Morning Queue is always a real Notion database (`notion-create-database` + DB rows). Heading/paragraph blocks like `### Row N — …`, `**Project:** …`, `**Triggered by:** …` on the dated page body are forbidden and fail the Step 6.5 gate. See the HARD RULE in Step 5 item 4.
- Does not write to V3 pages (see `references/v3-context.md`)
- Does not modify the PM's Preferences page autonomously (only via explicit preference-edit commands)
- Does not delete pages or databases
- Does not change the parent page's page-level properties (title, icon)
- Does not use toggles inside row detail pages except the one bottom Reference Context toggle
