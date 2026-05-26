> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Fathom, Notion (Slack forbidden). Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

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
   - One line: `N items for your morning. X sub-tasks, Y flags. <M signals filtered — see Run Log if you want to audit>.`

4. **Inline Morning Queue database**
   - Create via `notion-create-database` with the parent as this dated page.
   - Schema per `schemas/morning-queue-database.md` — 10 columns, all visible in the default view, in order: Summary, AI Notes, Orbit Task Link, Project, Recommended Action, Recommended Assignee, Outcome, PM Notes, Source Systems, Status.
   - `Orbit Task Link` is Notion's native URL property — single clickable URL per row. For `Create subtask` rows it holds the parent task's Orbit URL; for `Flag` rows it holds the bound Orbit task URL, or `—` / empty when no Orbit task is bound (e.g., Gmail-only flag). The sub-task URL that Mode 2 creates is written into `Outcome` only — never into `Orbit Task Link` (which keeps the parent reference stable through the row's lifetime).
   - `Project` format is `<Project Name> (#<orbit_project_id>)` — matcher supplies both name and Orbit project code. Maintenance / Ad-hoc style projects append a type suffix (`— Maintenance`).

### Step 5.5 — Enforce output gating (defense in depth)

Before iterating over the matcher's output array, scan it once and verify every item conforms to the 2-action rule from `synthesis/matcher.md` Output gating:

- Every item's `summary` must start with `Create subtask on` or `Flag `. No other openers.
- `Create subtask` items MUST carry `parent_task_id`, `task_title`, `assignee_id`, and `proposed_orbit_body`. Missing any of these = malformed.
- `Flag` items MUST carry `pm_next_step`. They MUST NOT carry `parent_task_id`, `task_title`, `assignee_id`, or `proposed_orbit_body`.
- Items with `recommended_assignee = —` AND row action is `Create subtask` are malformed (subtask always has an assignee or an `Uncertain:` AI Note explaining why).

Any item failing these checks is moved from the items array to the `filtered_signals` array with `filter_reason: writer_gating_caught_drift` before row creation. Log to Run Log. This is a safety net; the matcher should have already prevented these — but if it didn't, the writer catches the drift rather than polluting the queue.

### Step 5.6 — Render the row-detail Action Block

For each item that survives Step 5.5, the writer renders the row's detail page with the Action Block callout at the very top (per `schemas/row-detail-page.md`). The callout uses the structural emoji 🎯 (Create subtask) or 🚩 (Flag) — these are on the writer-emoji allowlist per `writers/plain-language.md`. The action callout appears BEFORE the H1 Summary. The PM should not have to scroll past Sources or Recommended Action to find the proposed task title and assignee — that information is in the top callout.

### Step 6 — Populate each row

For each item in the matcher's output array (after Step 5.5 gating):

1. Create a row in the database with (in schema DDL order — Summary, AI Notes, Orbit Task Link, Project, Recommended Action, Recommended Assignee, Outcome, PM Notes, Source Systems, Status):
   - `Summary` (title) — matcher's verb-first one-liner
   - `AI Notes` — any uncertainty flags, split reasoning, or matcher Job 6 delegate reasoning; for priority-lane rows always populated with `<AM> put this on your plate overnight, due today. Proposed delegate: <name> (<reason>).`; empty otherwise
   - `Orbit Task Link` (URL property) — parent task URL for `Create subtask` rows; bound task URL for `Flag` rows that have one; `—` or empty for Flag rows with no Orbit task
   - `Project` — `<name> (#<orbit_project_id>)` or `Standalone`
   - `Recommended Action` — short phrase
   - `Recommended Assignee` — name + role + short reason; `—` for advisory items
   - `Outcome` — empty (filled by Mode 2; Mode 2 writes the new sub-task URL here, NOT into Orbit Task Link)
   - `PM Notes` — empty
   - `Source Systems` — multi-select of which collectors contributed (Orbit, Gmail, Fathom)
   - `Status` = `Recommended Action` (default)

2. Populate the row's page content per `schemas/row-detail-page.md`:
   - `Summary` heading + the matcher's summary
   - `Sources` heading + full citations per `writers/source-citation.md`. When matcher Job 4b produced a `context_signals[]` array on this row (Gmail/Fathom signals linked to a primary Orbit signal as corroborating context), render each linked signal as an H2 subsection under Sources. High-confidence cross-links go under the primary Sources list; weak cross-links (match_strength="weak") go under an H2 `Possible context` subsection.
   - `Recommended Action` heading + reasoning
   - `Proposed Orbit Task Body` heading + 6-section body in plain language
   - `Proposed Handoff` heading + plain-language message
   - `Proposed Email` heading + email draft (if applicable)
   - `AI Notes` heading (if any notes). For priority-lane rows, AI Notes carries the matcher's `<AM> put this on your plate overnight, due today. Proposed delegate: <name> (<reason>).` line.
   - Bottom toggle: `Reference Context for the Skill — working memory, not for review`, containing the full raw signals and pod inference reasoning

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
- Don't append a second Pod Daily Task block on the same dated page. Replace the existing toggle's children instead.
- Don't append a second AM Daily Ping block on the same dated page. Find the anchor and replace the body blocks below it.

## Error handling

| Failure | Behavior |
|---|---|
| Page creation fails | Retry once. If fails, abort Mode 1 with sent email to PM: `Couldn't write today's morning queue. [error]` |
| Database creation fails | Retry once. If fails, abort. |
| Row creation fails for a single item | Log in AI Notes on the summary block, continue with remaining rows |
| Block reorder fails during archival | Log and continue. Order drift can be corrected on the next fire. |
| `notion-move-pages` fails during archival sub-page reorder (Run Log / Incidents / Preferences) | Log and continue. |
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

- Does not write to V3 pages (see `references/v3-context.md`)
- Does not modify the PM's Preferences page autonomously (only via explicit preference-edit commands)
- Does not delete pages or databases
- Does not change the parent page's page-level properties (title, icon)
- Does not use toggles inside row detail pages except the one bottom Reference Context toggle
