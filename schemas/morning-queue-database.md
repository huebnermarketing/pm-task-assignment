# Schema — Morning Queue Database

The inline database that sits inside each dated sub-page. One row per item. The skill creates a fresh Morning Queue database every Mode 1 run (one per dated page).

---

## Database identity

| Field | Value |
|---|---|
| Title | `Morning Queue` |
| Parent | The dated sub-page (e.g., the page titled `25 April 2026`) |
| Inline | Yes (rendered inline within the dated page, not as a standalone database) |
| Default view | `Today's Queue` (defined below) |

---

## Full column list (in display order, left to right)

The database has **9 columns total**. The first 6 are visible in the default view. The last 3 (Project, Source Systems, AI Notes) exist in the schema but are hidden from the default view — they appear only when a row is opened to its detail page.

### Column 1 — `Summary`

| Property | Value |
|---|---|
| Type | **Title** (the row's primary identifier — Notion's required title field) |
| Visible in default view | YES (column 1 — leftmost) |
| Set by | Mode 1 (the matcher), via `synthesis/matcher.md` |
| Editable by PM | Yes, but rare — the matcher writes a good summary, PM usually doesn't change it |
| Default value | A one-line verb-first action sentence per `synthesis/matcher.md` Job 4. Starts with one of TWO locked verbs: `Create subtask` or `Flag`. No other openers permitted. For `Create subtask` rows, the proposed sub-task title appears inside the Summary so the PM reads it without opening the row detail page. |
| Maximum length | Capped at 120 chars (the matcher enforces this — see Job 4 rules). |
| Sample values | `Create subtask on ECP AI Visibility Audit #110464 — Run AI Visibility audit on approved competitor list. To Hitesh.` <br> `Create subtask on Solstice WP #106447 — Swap Contact Form brochure PDF. To Atul.` <br> `Flag 2010 Solutions — Ellen needs dev names for 27 May call. PM action.` <br> `Flag Conversant FTP — credentials still missing, blocking dev work today. PM action.` |

### Column 2 — `Status`

| Property | Value |
|---|---|
| Type | **Select** (single-choice dropdown) |
| Visible in default view | YES (column 2) |
| Set by | Mode 1 sets default; PM flips to `Approved` or `Skip. No Action Needed`; Mode 2 finalizes to `Done` |
| Editable by PM | Yes — primary PM interaction |
| Default value | `Recommended Action` |

**Dropdown options (4 total):**

| Order | Option name (exact text) | Color | Meaning |
|---|---|---|---|
| 1 | `Recommended Action` | gray | Default state. Skill recommended something, PM hasn't decided yet. If left here when Mode 2 fires, **nothing happens** for this row. |
| 2 | `Approved` | blue | PM flipped manually. Mode 2 will execute the recommendation. |
| 3 | `Done` | green | Mode 2 successfully executed this row. Final state. |
| 4 | `Skip. No Action Needed` | orange | PM explicitly chose to skip. Mode 2 does nothing. Final state. |

**State transitions allowed:**

```
Recommended Action ─► Approved (by PM)         ─► Done (by Mode 2)
                  ─► Skip. No Action Needed (by PM)
                  ─► [stays at Recommended Action] (no action)
```

`Approved` can be flipped back to `Recommended Action` if the PM changes their mind before Mode 2 fires.

`Done` is final. Skill never edits a `Done` row.

### Column 3 — `Recommended Action`

| Property | Value |
|---|---|
| Type | **Rich Text** (short text — paragraph styling allowed but rarely used) |
| Visible in default view | YES (column 3) |
| Set by | Mode 1 (the matcher) |
| Editable by PM | Yes, but rare — PM usually drops a note in `PM Notes` instead of editing this column |
| Default value | A short phrase describing one of the two locked actions (`Create subtask` or `Flag`) plus what Mode 2 will do. Per `synthesis/matcher.md` Job 5 the action set is closed to these two. |
| Maximum length | Recommend under 100 chars for scannability |
| Sample values | `Create subtask under #110464, assign to Hitesh (SEO/AI) + handoff draft` <br> `Create subtask under #106447, assign to Atul (WP) + handoff draft` <br> `Flag — PM owns next step (reply to Ellen). No Mode 2 action.` |

### Column 4 — `Recommended Assignee`

| Property | Value |
|---|---|
| Type | **Rich Text** |
| Visible in default view | YES (column 4) |
| Set by | Mode 1 (the matcher, via `synthesis/pod-inference.md`) |
| Editable by PM | Yes — but typically the PM uses `PM Notes` ("assign to Vijay instead") rather than editing this column directly |
| Default value | `Name (Role) — short reason` |
| Allowed empty value | `—` (em-dash, indicating advisory item with no assignee) |
| Sample values | `Vijay Patel (FE) — primary FE on Agency X, 24 hrs on current homepage task` <br> `Mannan Kapoor (QA)` <br> `Rohit Mehra (BE) — handles maintenance security` <br> `—` (advisory only) |

### Column 5 — `PM Notes`

| Property | Value |
|---|---|
| Type | **Rich Text** |
| Visible in default view | YES (column 5) |
| Set by | The PM (manually) |
| Editable by PM | Yes — primary PM input alongside Status |
| Default value | Empty |
| Maximum length | No hard cap; recommend under 500 chars |
| How it's interpreted | By `synthesis/note-interpreter.md` during Mode 2. Free-form natural language. PM note always wins over the original recommendation. |
| Sample values | `assign to Ravi instead, he knows their codebase` <br> `save as draft — want to run this past Caitlin first` <br> `mark as high priority` <br> `split into two — homepage and about page` <br> `add Nishant as CC` <br> *(empty — PM accepts the recommendation)* |

### Column 6 — `Outcome`

| Property | Value |
|---|---|
| Type | **Rich Text** |
| Visible in default view | YES (column 6 — rightmost) |
| Set by | Mode 2 (after execution) |
| Editable by PM | Technically yes, but they shouldn't — this is the skill's audit trail |
| Default value | Empty |
| Populated when | Mode 2 finishes processing this row (success or failure) |
| Format | Concise, specific, with Orbit links where applicable |
| Sample values | `Subtask #110890 created under parent #110464. Handoff draft for Hitesh appended below.` <br> `Subtask #110918 created under parent #109958. Severity bumped to Important per your note. Handoff draft appended below.` <br> `Flag row — no Mode 2 action.` <br> `FAILED — Orbit create_subtask returned 409. Will retry if re-approved.` |

### Column 7 — `Project` (HIDDEN from default view)

| Property | Value |
|---|---|
| Type | **Rich Text** |
| Visible in default view | NO (hidden) |
| Visible in row detail page | Yes (as metadata) |
| Set by | Mode 1 (the matcher) |
| Editable by PM | Yes — but typically the matcher gets it right |
| Default value | Inferred Orbit project name |
| Allowed empty value | `Standalone` (item not tied to a specific project) |
| Sample values | `Agency X — Homepage Redesign` <br> `BrightPath Phase 2` <br> `CloudBase Platform Build` <br> `Standalone` |

### Column 8 — `Source Systems` (HIDDEN from default view)

| Property | Value |
|---|---|
| Type | **Multi-Select** (zero or more values) |
| Visible in default view | NO (hidden) |
| Visible in row detail page | Yes (as tags at top) |
| Set by | Mode 1 (the matcher) |
| Editable by PM | No (skill-managed) |
| Default value | At least one source — items always have a contributing collector |

**Multi-select options (3 total):**

| Option name (exact text) | Color |
|---|---|
| `Orbit` | green |
| `Gmail` | red |
| `Fathom` | orange |

The skill picks any combination of the three. A row that came from a single Gmail thread gets `[Gmail]`. A row aggregated from email + Orbit gets `[Gmail, Orbit]`. Priority-lane rows (Mode 1 Step 3a) always include `Orbit` and may also include `Gmail` and/or `Fathom` if matcher Job 4b cross-linked corroborating context signals.

> Historical note: previous schema versions also offered a `Slack` (purple) option. Existing rows with that tag are preserved as an immutable audit trail; new runs cannot produce it.

### Column 9 — `AI Notes` (HIDDEN from default view)

| Property | Value |
|---|---|
| Type | **Rich Text** |
| Visible in default view | NO (hidden) |
| Visible in row detail page | Yes (under the AI Notes heading) |
| Set by | Mode 1 (the matcher); Mode 2 may append after execution |
| Editable by PM | Yes (PM can read and add their own notes; skill won't overwrite manual additions) |
| Default value | Empty (only populated when there's something notable) |
| Format | Plain text, multi-line OK |
| Common patterns | `Uncertain: I don't know if this email relates to Agency X's homepage or a new request.` <br> `PM note changed assignee Vijay → Ravi. Honored.` <br> `New project — no task history. Pod inference based only on followers.` <br> `Gmail collector failed this morning — this row's source list is incomplete.` |

---

## Database default view configuration

| Property | Value |
|---|---|
| View name | `Today's Queue` |
| View type | Table |
| Visible columns (in order) | Summary, Status, Recommended Action, Recommended Assignee, PM Notes, Outcome |
| Hidden columns | Project, Source Systems, AI Notes |
| Sort | By creation order ascending (matches matcher's intentional ordering — urgency first, external client first, multi-source first, recency for the rest) |
| Filter | None (show all rows) |
| Group by | None |

---

## Optional secondary views

The skill can create these additional views if useful in the future. None are created in v1; they're listed here for forward compatibility.

### View: `Open items only`

- Filters out rows with `Status = Done` or `Status = Skip. No Action Needed`
- Useful on busy mornings to focus on pending items

### View: `Attention flagged`

- Filters: rows where `AI Notes` contains `Uncertain:`
- Useful for spotting items the skill wasn't sure about

### View: `By project`

- Group by `Project`
- Useful when one client has multiple items

---

## Row creation rules

Every row created by Mode 1 must have:

- `Summary` populated — never empty
- `Status` set to `Recommended Action` (the default)
- `Recommended Action` populated (or a clear placeholder if there's truly no action — rare)
- `Recommended Assignee` populated OR set to `—` for advisory items
- `Project` populated OR set to `Standalone`
- `Source Systems` populated with at least one value
- `PM Notes` — empty
- `Outcome` — empty
- `AI Notes` — populated only if there's something notable; empty otherwise

---

## Schema enforcement during Mode 1

When `writers/notion.md` creates the Morning Queue database, it uses these exact column names, types, and option values. The SQL DDL the skill generates for `notion-create-database` looks like this:

```sql
CREATE TABLE (
  "Summary" TITLE,
  "Status" SELECT('Recommended Action':gray, 'Approved':blue, 'Done':green, 'Skip. No Action Needed':orange),
  "Recommended Action" RICH_TEXT,
  "Recommended Assignee" RICH_TEXT,
  "PM Notes" RICH_TEXT,
  "Outcome" RICH_TEXT,
  "Project" RICH_TEXT,
  "Source Systems" MULTI_SELECT('Orbit':green, 'Gmail':red, 'Fathom':orange),
  "AI Notes" RICH_TEXT
)
```

The order in the SQL determines the default left-to-right column order. Hidden columns (Project, Source Systems, AI Notes) are hidden via the database's view configuration after creation.

If the database creation tool returns an error (e.g., a schema conflict), abort Mode 1 with a notification to the PM via `connector-failure-notify.md`.

When the schema needs to evolve (e.g., adding a column in a future version), use `notion-update-data-source` with ALTER statements to migrate live databases without losing data. Document the migration in a CHANGELOG.

---

## Sample row — Create subtask (project work)

| Column | Value |
|---|---|
| Summary | `Create subtask on ECP AI Visibility Audit #110464 — Run AI Visibility audit on approved competitor list. To Hitesh.` |
| Status | `Approved` |
| Recommended Action | `Create subtask under #110464, assign to Hitesh (SEO/AI) + handoff draft` |
| Recommended Assignee | `Hitesh Asnani (SEO/AI) — Marketing/SEO Matrix, in followers, Ellen named him in approval comment` |
| PM Notes | *(empty — PM accepted the recommendation)* |
| Outcome | `Subtask #110890 created under parent #110464 (assigned to Abhishek). Handoff draft for Hitesh appended below (copy from row page).` |
| Project | `ECP AI Visibility Audit (#8545)` |
| Source Systems | `Gmail, Orbit` |
| AI Notes | `Parent picked: #110464 (Signed SOW) — only PM-owned task open on this project.` |

## Sample row — Create subtask (Ad-hoc / Maintenance work, also Create subtask now)

| Column | Value |
|---|---|
| Summary | `Create subtask on Brother Plesk #109958 — Investigate patch ETA with WP Maintenance. To Atul.` |
| Status | `Approved` |
| Recommended Action | `Create subtask under #109958, assign to Atul (WP) + handoff draft` |
| Recommended Assignee | `Atul Mehra (WP) — WP Maintenance lead, 2 prior tasks on Brother last 3mo` |
| PM Notes | *(empty)* |
| Outcome | `Subtask #110918 created under parent #109958. Handoff draft for Atul appended below (copy from row page).` |
| Project | `Brother Site Issues (#8461) — Maintenance` |
| Source Systems | `Orbit, Fathom` |
| AI Notes | `Maintenance project type — previously this would have been a Reassign row; new rule unifies on Create subtask under the PM-owned parent.` |

## Sample row — Flag (PM action needed)

| Column | Value |
|---|---|
| Summary | `Flag 2010 Solutions — Ellen needs dev names for 27 May call. PM action.` |
| Status | `Recommended Action` |
| Recommended Action | `Flag — PM owns next step (reply to Ellen). No Mode 2 action.` |
| Recommended Assignee | `— (PM action)` |
| PM Notes | *(empty)* |
| Outcome | *(empty — Mode 2 does not execute Flag rows. PM marks Skip when resolved.)* |
| Project | `2010 Solutions FTE Works (#6994)` |
| Source Systems | `Gmail` |
| AI Notes | `PM-action detection: no PM-sent reply on the thread between signal arrival 2026-05-15 21:00 IST and Mode 1 fire. Flag emitted.` |

---

## What this schema does NOT include

- An "Estimated hours" column — hours live on the Orbit task, not in the queue.
- A "Due date" column — dates live on the Orbit task. The queue is about TODAY's actions.
- A "Priority" column — captured in the row sort order, not as a visible column.
- An "Uncertain?" checkbox — uncertainty is captured as text in `AI Notes` instead.
- An "Availability" column — no availability check in the recommendation logic per locked decision.
- A "Last edited by" column — Notion tracks this automatically; not exposed in the schema.
- A "Created at" column — Notion tracks this automatically; sorted by it but not exposed as a visible column.
- A "Source link" column — sources are cited in the row detail page (per `writers/source-citation.md`), not in the database row.
