# Schema — Fixed-Cost Registry Page

Sub-page of the Notion parent, sibling of Run Log / Incidents / Preferences. Created by
preflight if missing (like Preferences). Title: exactly `Fixed-Cost Registry`.

The registry is a **rendered snapshot + manual-override surface + ask ledger** — NOT the
source of truth for discovery. Source of truth is the daily filtered `list_projects` call
(collectors/orbit.md § Fixed-cost discovery). The skill refreshes this page every Mode 1 fire.

## Top-down layout

1. Header callout
2. H2 `Tracked Projects` + inline table
3. H2 `Client-Ask Ledger` + inline table
4. Toggle `Machine state — do not edit` containing one fenced `fc_state` JSON block

## 1. Header callout

Gray background, gear emoji. Five lines:

```
Fixed-cost tracking: <N> active projects
Last refresh: <YYYY-MM-DD HH:MM IST> · Discovery: <filtered | fallback-sweep>
Last successful discovery: <YYYY-MM-DD HH:MM IST>
Activity source: <shadow | mail-primary> · Clean audits: <n>
Managed by PM Task Assignment — edit only the Manual pin rows and the Ask column.
```

`discovery_mode` values: `filtered` (guard passed) or `fallback-sweep` (guard tripped,
Appendix-A sweep or last-known-good in use).

`Last refresh` is bumped every run regardless of outcome. `Last successful discovery` is
bumped only when `discovery_succeeded` — guard pass or completed sweep; never on snapshot
reuse (`collectors/orbit.md` § Fixed-cost extension returns this boolean explicitly, distinct
from `discovery_mode`: a guard-trip fallback that merely reuses the last-known tracked set
also reports `discovery_mode: fallback-sweep`, but `discovery_succeeded: false`, so it must
NOT bump this timestamp). It is the field `collectors/orbit.md` § Fixed-cost discovery reads
(via the `registry_snapshot` input) to decide whether a guard-trip fallback can reuse the
Tracked Projects set.

`Activity source` selects the fixed-cost lane's event feed (spec §11.3). `shadow` (the
initial value, and the value assumed when the line is missing on a pre-v4 page): BOTH the
unfiltered per-project `get_activity_log` loop AND Orbit-notification-mail parsing run;
Orbit is authoritative. `mail-primary`: the unfiltered `get_activity_log` loop is OFF; the
mail-derived signals are the fixed-cost event feed (roster loop and workload lane
unaffected). `Clean audits` counts consecutive gap-free `FC-0 — Friday coverage audit`
passes; it reaches **2** → the writer flips `Activity source: mail-primary` at registry
refresh. Any `mail_coverage_gap` → reset to 0 (and revert to `shadow` if in mail-primary).
Both fields are machine-maintained by the writer (`writers/notion.md` § Flow — refreshing
the Fixed-Cost Registry); the PM never edits them.

## 2. Tracked Projects table

One row per tracked project. Columns (exact names):

| Column | Content |
|---|---|
| `Project` | `<Title> (#<project_number>)` — rule 22 format |
| `Project ID` | internal Orbit `id` (tooling only; never quoted in queue text) |
| `Client` | `client_name` (+ ` / <sub_client_name>` when present) |
| `Due date` | `due_date` date part, or `—` (infinite/unset) |
| `Added by` | `filter` \| `manual` |
| `Added on` | date first seen |

Rules:
- `filter` rows are fully machine-managed: added when discovery first returns the project,
  removed when discovery stops returning it (project closed/on-hold) — removal only under
  `discovery_mode: filtered`; under fallback, prune per the sweep result instead.
- `manual` rows are PM-owned pins: never auto-removed. If discovery can't see the project AND
  its `get_project_details.status` is not Active, append ` — verify: closed?` to the Project
  cell instead of deleting.
- Manual pins join the daily activity union exactly like filtered projects.

## 3. Client-Ask Ledger table

One row per client ask on a tracked project. Columns (exact names):

| Column | Content |
|---|---|
| `Project` | `<Title> (#<project_number>)` |
| `Ask` | the specific thing requested — "logo assets + brand fonts", never "waiting on client" |
| `Asked on` | date of the originating ask signal |
| `Source` | link to the anchoring Orbit comment / task, or Gmail thread subject + date |
| `Status` | `Open` \| `Resolved` \| `Resolved-manual` |
| `Resolved on` | `YYYY-MM-DD`; set by FC-1 auto-resolution or the Mode 2 manual resolution when `Status` changes to `Resolved`/`Resolved-manual`; empty while `Open` |
| `Resolved by` | link/reference to the resolving signal (auto-resolution) OR the PM's note text (manual resolve via Mode 2); empty while Open |
| `Last reminder` | date the last follow-up Flag row was emitted, else `—` |

Rules:
- Rows are appended by FC-2 (ask detection) or by the PM manually (Project + Ask suffice;
  the skill backfills `Asked on` = today and `Source` = `manual` on next run).
- `Resolved` / `Resolved-manual` rows stay in the table (audit trail); FC-3 only evaluates
  `Status: Open` rows. Monthly archival moves rows with Status `Resolved`/`Resolved-manual`
  and `Resolved on` older than 60 days into an `Archived Asks` toggle at the bottom of this
  page (see `modes/monthly-archival.md`).
- The ledger is the authoritative resolution state. The `fc_state` block never stores
  open/resolved — only throttle timestamps — so a corrupt state block can cause at worst an
  early reminder, never a false one.

## 4. Machine state block

Inside a toggle titled `Machine state — do not edit`, one fenced code block:

```fc_state
{
  "asks":  { "#16915|2026-06-25|logo assets + brand fonts": { "last_reminder": "2026-07-02" } },
  "pings": { "<task_id>": { "last_ping": "2026-07-02", "stale_since": "2026-06-30" } }
}
```

- Ask keys: `#<project_number>|<Asked on date>|<first 40 chars of Ask>` — stable across runs.
  The ledger `Asked on` cell and the date segment of the key MUST be the identical
  `YYYY-MM-DD` string (key stability depends on it).
- Unreadable/corrupt block → treat as empty `{"asks":{},"pings":{}}` and rebuild over
  subsequent runs. Never abort on state-block failure.
- `Last reminder` ledger column mirrors `asks.*.last_reminder` for PM visibility; the JSON is
  what FC-3/FC-4 read.

## Date format

`Asked on`, `Added on`, `Resolved on`, and every date inside `fc_state` use `YYYY-MM-DD`
uniformly (no time component, no IST suffix — that's reserved for the header callout's
`HH:MM IST` timestamps). The ledger `Asked on` cell and the date segment of the `fc_state` ask
key MUST be the identical string — key stability depends on it.

## Failure posture

Registry page unreadable → non-blocking: log Incident, proceed with the day's discovery
result directly (manual pins and ask ledger are unavailable for that run; the state tracker
skips all four FC jobs this run — reminders/pings simply don't fire that morning; they resume
next run).

## What this page does NOT do

- Not the discovery source of truth (the filtered call is).
- Never holds queue rows, briefs, or handoff drafts.
- Not managed by Mode 2 — Mode 2 never refreshes the Tracked Projects table or runs
  discovery; its single touch is applying a PM note's resolve (`Status` → `Resolved-manual`,
  `Resolved by`, `Resolved on`) or snooze (`Last reminder` + `fc_state` `last_reminder` bump)
  to a Client-Ask Ledger row, via `writers/notion.md` § Flow — Mode 2 ledger touch.
