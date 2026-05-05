> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes routine triggers — preflight runs even when invoked by a scheduled cloud routine.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `external-doc-access.md`). The Pod Matrix Notion page (this file's subject) is also read-only — see "Notion scope exception" below. No other MCP, ever.

# Reference — Pod Matrix (read-only)

The org maintains a top-down team-structure page on Notion called **Matrix Detail**. It lists every matrix (PM-owned and functional), the role buckets in each matrix, and the team members in each role. The skill reads this page to ground assignee recommendations in real org structure rather than inferring pods only from Orbit follower / task-history bottom-up signals.

This file defines:

- Where the URL comes from (runtime injection — not committed in this repo).
- Three fallback conditions that drop the skill back to Orbit-only pod inference.
- How the page is parsed.
- How matrix-member names are resolved to Orbit user IDs.
- The output shape that `synthesis/pod-inference.md` consumes.

## URL source

The Matrix Detail page URL is **injected by the Mode 1 routine prompt at fire time** as the variable `POD_MATRIX_URL` — same pattern as `NOTION_PARENT_PAGE_URL` and `PREFERENCES_PAGE_URL`. See `ROUTINE-ENTRYPOINTS.md` § Routine 1.

The URL is **not** stored in `config.md`. It is per-installation runtime context, not committed code.

Mode 2 (Execution) and Monthly Archival do **not** receive `POD_MATRIX_URL` and do **not** read this page. Mode 2 acts on assignees already written to queue rows by Mode 1; Archival never touches pod data.

## Notion scope exception

The skill's general rule (per `SKILL.md` and `config.md`) is that Notion access is restricted to the PM's Notion parent page. The Pod Matrix page is the **only** read-only exception. It is allowlisted for `notion-fetch` only; never written to, never enumerated, never used for any other purpose.

## When this file runs

- **Always called by `modes/mode-1-morning-collection.md`** at the start of the run, immediately after Preferences is read and before collectors fire.
- **Never called by Mode 2 or Monthly Archival.**
- **May be called by interactive `PM Task Assignment, run morning`** invocations. In that case `POD_MATRIX_URL` is typically absent and the loader returns the "matrix unavailable" fallback shape (see below) — `synthesis/pod-inference.md` then proceeds with Orbit-only inference.

## Fallback — three "matrix unavailable" conditions

The loader returns `{ matrix_unavailable: true, reason: <string> }` (and nothing else) in these three cases. Pod-inference treats this exactly the same in all three: skip Step 0, run Steps 1–4 against Orbit signals only, log a one-line note in the Run Log Decisions trace.

| Condition | `reason` value |
|---|---|
| `POD_MATRIX_URL` not in runtime context (interactive surface or routine misconfiguration) | `Pod Matrix not loaded — interactive surface or URL not injected` |
| `notion-fetch` failed (network / 404 / permission) after the standard retry policy | `Pod Matrix fetch failed — <one-line error>` |
| Page fetched but parser could not find the expected `### <Matrix Name>` headings + table rows | `Pod Matrix parse failed — unexpected page structure` |

Graceful degradation is the whole point — a matrix outage must never block the morning queue.

## Parse rules

1. Fetch the page via `notion-fetch` with `id: <POD_MATRIX_URL>`. Apply the standard retry policy from `connector-failure-notify.md`.
2. The page body contains an ordered list of matrix names at the top, then `## Matrix Details`, then one `### <Matrix Name>` heading per matrix, each followed by a `<table>` with two columns:
   - Column 1: `Role / Function`
   - Column 2: `Team Members` (comma-separated names; may include parenthetical role hints like `Jay (designer)` or `Astha (content writer)`)
3. **Yellow-highlighted rows** (`<tr color="yellow">`) mean "to be hired" or vacant — skip them entirely.
4. Section headings without a `<table>` (e.g., `### Night Shift` in the current page) are ignored — emit a parse note but do not fail.

## PM → matrix lookup

Determined from Preferences `Identity.Name` (case-insensitive first-name match against PM rows in each matrix). Current mapping — embedded as a fallback if the matrix page somehow lists multiple PMs in a matrix:

| PM first name | Matrix |
|---|---|
| `Abhishek` | Matrix A |
| `Hiten`    | Matrix B |
| `Dhruvi`   | Matrix C |
| `Juhi`     | Matrix D |
| `Aditi`    | Internal Projects |

If the running PM's first name does not match any matrix's PM row, set `running_pm_matrix = null` and emit a note: `PM <name> not found in any Matrix Detail PM row — using Orbit-only pod inference for this run`. Pod-inference treats this the same as the matrix-unavailable fallback.

## Role normalization

Matrix uses the org's terminology. Pod-inference uses the existing role enum (`FE | BE | WP | Design | QA | Content | Full-stack | BA | Unknown`). Map:

| Matrix label | Pod-inference role |
|---|---|
| `HTML` | `FE` |
| `WordPress / PHP` (any spelling — `WordPress · PHP`, `WP/PHP`) | `WP` |
| `QA` | `QA` |
| `Business Analyst` / `BA` | `BA` (new — treated as `Unknown` for non-BA tasks) |
| `Special Allocation` | role-by-context — read the member's Orbit `department` to classify |
| `Marketing`, `SEO` | `Content` (when task domain matches) |
| `Design` (matrix or row label) | `Design` |
| `Development Team` (SaaS Matrix) | `Full-stack` |
| `Server / Infrastructure`, `Hosting / Infrastructure` | `Unknown` (rarely targeted by pod-inference; keep for completeness) |
| `Lead`, `Mentor`, `Head`, `Owner / Lead`, `Core Team`, `Team` | role-by-context — read the member's Orbit `department` |
| `Account Managers aligned` | excluded from pod (existing rule #2 in `synthesis/pod-inference.md`) |

`PMs` / `PM` rows are also excluded from pod (the running PM themselves never gets recommended as their own assignee per the existing `exclude_from_pod` rule).

## Name → Orbit user_id resolution

For each non-skipped matrix member, resolve to an Orbit `user_id`:

1. Call `mcp__...orbit.list_users` once at the start of the Mode 1 run; cache the user list for the whole run.
2. Strip parenthetical role hints (`Jay (designer)` → `Jay`).
3. Match by first name + (optional) last name. Prefer last-name-included matches when the matrix gives one (`Vijay Salvi` not just `Vijay`; `Jay Panchal` not just `Jay`; `Ashish Gaur` not just `Ashish`).
4. **Disambiguation when multiple Orbit users share the matrix name** — prefer the one whose Orbit `department` aligns with the matrix role for that row. Example: "Atul" appears in Matrix A's `WordPress / PHP` row → match the Atul whose Orbit department is `WordPress / PHP`, not a different Atul in another department.
5. **Still ambiguous** — drop the name from candidates, surface in Run Log Decisions as `name-ambiguous: <name> in <matrix> (<role row>)`.
6. **Not found in Orbit** — drop, surface as `name-unresolved: <name> in <matrix>`. Common cause: new joiner whose Orbit user has not been provisioned yet, or a contractor not in Orbit.

### Cross-matrix duplicates — known list

These names appear in multiple matrices. The disambiguation rule above handles them, but record them here so future operators understand the parse is deliberate, not buggy:

| Name | Appears in | Resolution |
|---|---|---|
| `Atul` | Matrix A WP, Maintenance WordPress Support | Same Orbit user — WP dept |
| `Kunjan` | Matrix D WP, Maintenance WordPress Support | Same Orbit user — WP dept |
| `Aagna` | Quoting, Maintenance | Same Orbit user — context-dependent role |
| `Pravin` | Quoting, WIC Dedicated | Same Orbit user — context-dependent role |
| `Jay` | AI Matrix Team, Marketing Matrix (designer), Design Matrix Lead (`Jay Panchal`) | **Three different people** — disambiguate by full name + matrix dept |
| `Astha` | Marketing Matrix (content writer), Sales Matrix | **Two different people** — disambiguate by Orbit dept (Content vs Sales) |
| `Kishan` | Marketing/SEO Matrix, SaaS Development Team | **Two different people** — disambiguate by Orbit dept (SEO vs Dev) |
| `Tarang` | AI Matrix mentor, SaaS Lead | Same Orbit user |
| `Vijay` | Floater (`Vijay Salvi`), Design Matrix support (`Vijay Jadav`) | **Two different people** — last-name disambiguates |

## Output shape

Returned to `synthesis/pod-inference.md` and cached for the run.

```
{
  "matrix_unavailable": false,
  "running_pm_matrix": "<matrix name or null>",
  "running_pm_matrix_members": [
    {
      "name": "<canonical full name from matrix>",
      "orbit_user_id": <int>,
      "role_hint": "FE | WP | QA | Design | Content | BA | Full-stack | Unknown",
      "matrix_role_label": "<verbatim label from matrix table — e.g. 'WordPress / PHP'>"
    }
  ],
  "floater_members": [
    { "name": "<full name>", "orbit_user_id": <int>, "role_hint": "FE", "matrix_role_label": "Floaters (Shared)" }
  ],
  "functional_matrices": {
    "Design":      [<member objects>],
    "QA":          [<member objects>],
    "Maintenance": [<member objects>],
    "AI":          [<member objects>],
    "SaaS":        [<member objects>],
    "Marketing":   [<member objects>],
    "WIC":         [<member objects>],
    "Hosting":     [<member objects>],
    "Quoting":     [<member objects>]
  },
  "unresolved": [
    { "name": "<matrix name>", "matrix": "<matrix>", "row": "<role label>", "reason": "ambiguous | not-found-in-orbit" }
  ],
  "fetched_at": "<ISO timestamp IST>"
}
```

When the matrix is unavailable, the shape collapses to `{ matrix_unavailable: true, reason: "<string>", fetched_at: "<ISO>" }`. All other fields absent.

## Caching

- Fetched **once per Mode 1 run**, at the start. Reused for every project's pod inference within that run.
- **No cross-run cache.** Each Mode 1 fire refetches. Matrix changes (new joiner, role move, exit) get picked up next morning, no manual cache bust required.
- The `list_users` call (used for name resolution) is also cached for the duration of the run only.

## What this file does NOT do

- Does not write to the Matrix Detail page. Read-only.
- Does not infer roles beyond what the matrix table states + the Orbit `department` disambiguator.
- Does not call `get_user_workload` — availability is computed lazily by `synthesis/pod-inference.md` when the matcher requests it. This file only resolves membership.
- Does not enrich missing matrix entries from Orbit followers. That's pod-inference's existing Step 1.
- Does not modify Preferences. The PM's `exclude_from_pod` list is applied later in pod-inference Step 2.

## Loading verification

> If you are reading this file as part of skill execution, you have correctly loaded `references/pod-matrix.md`. The contract above is the source of truth for how the Pod Matrix is fetched, parsed, and exposed to pod-inference. Fallback to Orbit-only pod inference is a normal degraded mode, not an error — log a note in the Run Log Decisions trace and continue.
