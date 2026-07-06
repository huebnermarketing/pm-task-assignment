# Schema: Run Log Database

## Purpose

Audit trail, debug surface, and trust signal. Every routine fire (Mode 1, Mode 2, Monthly Archival) writes one row to this database with a brief decision summary. Each row links to a `Run Log Detail` sub-page (see `schemas/run-log-detail-page.md`) holding the full decision trace. The PM uses the database to glance at "did the morning run go OK?" without opening every detail page; the operator uses it to spot patterns in failures.

The database is **brief by design**. Counts and statuses live here; reasoning lives in the linked detail page.

---

## Location

- **Parent:** the PM's Notion parent page (the same page that holds Preferences + dated queue pages).
- **Sub-page name:** `Run Log` (literal title).
- **Database type:** **inline database on the `Run Log` sub-page** (default). A full-page database would also work, but inline-on-subpage keeps the parent clean and lets the sub-page hold optional notes / views without colliding with the database itself.
- **Created on first preflight if missing.** Preflight Step 6 detects absence and creates the `Run Log` sub-page + inline database with the schema below before any routine writes.

---

## Columns

| Column           | Type                | Notes                                                                                       |
| ---------------- | ------------------- | ------------------------------------------------------------------------------------------- |
| `Run ID`         | Title               | Format `YYYY-MM-DD-mode<N>-HHMM` — e.g. `2026-04-29-mode1-0930`. Mode 2 → `mode2`. Monthly archival → `monthly`. |
| `Date`           | Date                | Fire date (no time component). Used for filters / monthly views.                            |
| `Mode`           | Select              | Options: `Mode 1`, `Mode 2`, `Monthly Archival`.                                            |
| `Status`         | Select              | Options: `OK`, `Partial`, `Failed`, `Escalated`. (See "Status meanings" below.)             |
| `Started`        | Date with time      | Run start timestamp, IST. Sort key.                                                         |
| `Duration`       | Number              | Whole seconds, end − start.                                                                 |
| `Signals`        | Number              | Mode 1: total signals collected across all sources. Mode 2 / Monthly: `0`.                  |
| `Items written`  | Number              | Mode 1: rows written to today's dated queue page. Mode 2: rows attempted (executed + failed + skipped). Monthly: pages archived. |
| `Items executed` | Number              | Mode 2 only — rows where the action succeeded. Mode 1 / Monthly: `0` or blank.              |
| `Errors`         | Number              | Connector failures + per-item execution errors. Counted, not described, here.               |
| `Detail`         | URL (preferred) or Relation | Link to the `Run Log Detail` sub-page for this run. URL keeps the schema simpler; relation is acceptable if a separate `Run Log Details` database is used later. |

**Note — `fixed_cost_stats`.** The Mode 1 fixed-cost lane stats object (`writers/run-log.md` input `fixed_cost_stats`) renders on the linked `Run Log Detail` sub-page only (see `schemas/run-log-detail-page.md` § Fixed-cost lane stats section). It does NOT get its own column on this database — the database stays brief-by-design; per-lane detail lives on the detail page.

### Status meanings

- `OK` — all sources reachable, no per-item errors, run completed normally.
- `Partial` — run completed but at least one source failed or at least one item errored. Detail page lists what.
- `Failed` — run aborted before completion. Detail page captures whatever was reached.
- `Escalated` — connector-failure tier 3 reached the "3+ consecutive unresolved incidents" threshold (per `connector-failure-notify.md`). Detail page captures the email sent to the backup person.

---

## Sort

- Default sort: **`Started` descending**. Newest run at top.

## Filters / Views

- **Default view:** `Last 30 days` — filter `Date` is within the past 30 days, sorted `Started` descending.
- **Optional view — By mode:** group by `Mode`, sort within group by `Started` descending. Useful when debugging a specific routine.
- **Optional view — Failures only:** filter `Status` is `Partial` or `Failed` or `Escalated`. The PM / operator triage queue.

Views beyond the default are nice-to-have; the writer creates only the default view on first run. Operator can add the others manually later.

---

## Retention

- The database **grows forever** in v1. At one row per routine fire and three fires per workday, that is approximately 750 rows per PM per year — well within Notion limits.
- **Future enhancement (out of v1 scope, document only):** the monthly archival routine could also nest Run Log rows older than the previous month under per-Year/per-Month sub-pages (e.g., `Run Log → 2026 → April`), mirroring the queue's `Year → Month → Date` hierarchy. Not implemented in v1.

---

## Write Rules

- **Only `writers/run-log.md` writes to this database.** No other writer, mode, or tool appends rows. This is enforced by convention; the writer file holds the single Notion call that creates the row.
- **Append-only.** Existing rows are never edited. If a routine retries, a new row is appended with a `(retry-N)` suffix on `Run ID` (see writer for idempotency rules).
- **Each row links to exactly one detail page** via the `Detail` column. The detail page is created **before** the row is appended, so the link is never null.
- **No row, no run.** If the writer itself fails (Notion outage), the writer falls back to a sent email to the PM with the run summary as a code block — the trace is never silently dropped.

---

## Loading verification footer

> If you are reading this file as part of skill execution, you have correctly loaded `schemas/run-log-database.md`. The schema above is the source of truth for the `Run Log` database structure on the PM's Notion parent page. The writer at `writers/run-log.md` enforces this schema; do not write to the database from anywhere else.
