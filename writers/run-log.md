# Writer: run-log

## Purpose & When Called

Appends one row to the `Run Log` Notion database and creates the linked detail sub-page that holds the brief decision trace. Called at the **end of every routine run** — Mode 1, Mode 2, and Monthly Archival — just before the routine exits.

Also called by `connector-failure-notify.md` when a run aborts mid-flow, so failed runs are still recorded with whatever partial data the mode collected before the abort. **Failures must be logged**; silent drops are a bug.

This writer is the **only** code path that writes to the `Run Log` database. See `schemas/run-log-database.md` for schema; see `schemas/run-log-detail-page.md` for detail-page layout.

---

## Inputs

The calling mode passes a structured `run-summary` object with the fields below. All fields are required unless noted.

```
run_summary = {
  mode:               "Mode 1" | "Mode 2" | "Monthly Archival",
  started:            ISO-8601 datetime, IST,
  ended:              ISO-8601 datetime, IST,
  status:             "OK" | "Partial" | "Failed" | "Escalated",
  sources:            [ { name, count, status }, ... ],            // Mode 1 only; [] for others
  items_written:      integer,
  items_executed:     integer,                                      // Mode 2 only; 0 otherwise
  decisions:          [ { item, subject, action, reason }, ... ],
  skipped:            [ { source, subject, reason }, ... ],
  uncertain:          [ { item, subject, candidates, reason }, ... ],
  connector_failures: [ { connector, step, error, tier }, ... ],
  filtered_signals:   [ { source, summary, filter_reason, citations }, ... ],  // Mode 1 only — from matcher Job 11
  execution_outcomes: [ { row, task, outcome, pm_note_interp }, ... ],  // Mode 2 only
  links: {
    queue_page_url:   string | null,        // Mode 1 / Mode 2
    archived_toggle:  string | null,        // Monthly Archival
  }
}
```

If a list field has no entries, pass `[]` (the writer omits the corresponding section heading).

---

## Steps

### Step 1 — Verify Run Log database exists

Use `notion-fetch` (Notion MCP) to confirm the `Run Log` sub-page + inline database exist on `NOTION_PARENT_PAGE_ID`. If missing, this is a preflight bug — preflight Step 6 should have created them. Surface the error via the failure-mode path below; do not silently create the database here.

### Step 2 — Compute the Run ID

Format: `YYYY-MM-DD-mode<N>-HHMM` from `started` (IST). Examples:

- `2026-04-29-mode1-0930`
- `2026-04-29-mode2-1045`
- `2026-05-01-monthly-0600`

Then check idempotency (Step 6 below) and adjust the ID if a duplicate exists.

### Step 3 — Build the detail-page body

Render the page body per `schemas/run-log-detail-page.md`. Sections in order:

1. Header callout — one line: `<Mode> · <HH:MM>–<HH:MM> IST · <Status> · <signals> signals · <items_written> items · <errors> errors` (errors = `len(connector_failures) + count of execution_outcomes where outcome == "failed"`).
2. `Sources` — one bullet per `sources[i]` (omit section if `sources == []`).
3. `Decisions` — one bullet per `decisions[i]`, formatted `Item N — [subject] → [action] → [reason]`.
4. `Skipped` — one bullet per `skipped[i]` (omit if empty).
5. `Uncertain` — one bullet per `uncertain[i]` (omit if empty).
6. `Connector failures` — one bullet per `connector_failures[i]` (omit if empty).
7. `Filtered signals (N)` — Mode 1 only. Notion toggle block, **closed by default**. Inside: one bullet per `filtered_signals[i]`, formatted `<source> · <summary> · <filter_reason> · <citations>`. Omit the toggle entirely if `filtered_signals == []`. This is where the PM audits what the matcher dropped under the 2-action output-gating rule (`synthesis/matcher.md` Output gating + Job 11).
8. `Execution outcomes` — Mode 2 only; one bullet per `execution_outcomes[i]`.
9. Footer — `Run Log database` link + queue-page-or-archived-toggle link.

Apply the **brevity rules** (next section) to every reason field before rendering.

### Step 4 — Create the detail sub-page

Call `mcp__...notion.notion-create-pages` with:

- Parent: the `Run Log` sub-page (preferred — keeps detail pages co-located) on `NOTION_PARENT_PAGE_ID`.
- Title: the Run ID from Step 2.
- Body: the rendered content from Step 3.

Capture the returned page URL — it goes into the database row's `Detail` column.

**Do this before writing the database row** so the row's `Detail` link is never null.

### Step 5 — Append the database row

Call `mcp__...notion.notion-create-pages` (or the database-row variant) targeting the `Run Log` inline database. Column values:

| Column           | Value                                                                |
| ---------------- | -------------------------------------------------------------------- |
| `Run ID`         | the ID from Step 2                                                   |
| `Date`           | `started` date portion                                               |
| `Mode`           | `run_summary.mode`                                                   |
| `Status`         | `run_summary.status`                                                 |
| `Started`        | `run_summary.started` (datetime IST)                                 |
| `Duration`       | `floor((ended − started) seconds)`                                   |
| `Signals`        | `sum(sources[i].count)` for Mode 1; `0` for Mode 2 / Monthly         |
| `Items written`  | `run_summary.items_written`                                          |
| `Items executed` | `run_summary.items_executed` for Mode 2; `0` otherwise               |
| `Errors`         | `len(connector_failures) + count(execution_outcomes where failed)`   |
| `Detail`         | URL of the detail page from Step 4                                   |

### Step 6 — Update calling-mode state

Return the Run ID and detail-page URL to the caller so the mode's exit log records them. The caller does not write further to Notion; this writer is the terminal Notion write of the run.

---

## Brevity Rules

The writer is the **last line of defense** for keeping the trace brief. Even if a calling mode passes verbose content, the writer enforces:

- **Reasons (decisions, skipped, uncertain, connector_failures):** truncate to 15 words. If truncated, append `…` (single ellipsis character). Word count = whitespace-split tokens.
- **No newlines inside a single decision/skip/uncertain entry.** Replace any embedded newline with `; `.
- **Subjects:** truncate to 80 characters; append `…` if truncated.
- **PM-note interpretations (Mode 2):** truncate to 12 words; same `…` suffix rule.
- **No code fences inside reasons.** Strip backticks; replace with single quotes.

These are dumb truncations, not summarization. The mode is responsible for passing reasonable content; the writer guarantees brevity even when the mode misbehaves.

---

## Failure Mode

If Step 4 or Step 5 fails (Notion API error, network outage, schema drift):

1. Retry once after a 5-second wait.
2. If still failing, **fall back to sent email**. Use the Gmail MCP to send the PM an email (their canonical email from Preferences) with:

   ```
   Subject: [PM Task Assignment] Run Log write FAILED for <Run ID>

   Body:
   Mode: <mode>
   Status: <status>
   Started: <started> IST
   Duration: <duration>s
   Signals: <signals>  Items: <items_written>  Errors: <errors>

   Decision summary:
   <one line per decision, brevity rules applied>

   Skipped:
   <one line per skipped>

   Connector failures:
   <one line per failure>

   Notion error: <error one-liner>
   ```

   Wrap the body inside `<pre>` tags (or a triple-backtick code block if the Gmail MCP renders Markdown) so it stays monospace and unwrapped.

3. If email also fails, write the summary to the routine's stdout/log so the operator finds it in the Routines run history. **Do not silently drop the trace.**

---

## Idempotency

Routines may retry on transient errors. If a row with the same `Run ID` already exists in the `Run Log` database (detected via `notion-search` or pre-write probe):

- Append `(retry-N)` to the Run ID, where `N` is the next integer (start at `1`). Example: `2026-04-29-mode1-0930 (retry-1)`.
- Use the same suffix on the detail-page title.
- Set `Status` to whatever the current run actually was — do not infer from the prior row.

This means a flaky day may produce `2026-04-29-mode1-0930`, `2026-04-29-mode1-0930 (retry-1)`, `2026-04-29-mode1-0930 (retry-2)` — all visible in the database, all linked to their own detail pages. The operator can then triage why the retries happened.

---

## Allowlist Reminder

This writer uses **only the Notion MCP** for its primary path: `notion-fetch`, `notion-create-pages`. On the failure-mode fallback path, it additionally uses the **Gmail MCP** (as a sent-email exception per `executors/email.md`) to email the PM. No Orbit or Fathom calls are ever made by this writer — those connectors are collector / executor concerns, not log concerns. Slack is forbidden.

---

## Loading verification footer

> If you are reading this file as part of skill execution, you have correctly loaded `writers/run-log.md`. This is the **only** writer that touches the `Run Log` database; modes must call this writer at end-of-run, not write the database directly. Brevity rules and idempotency rules above are enforced here so calling modes do not have to.
