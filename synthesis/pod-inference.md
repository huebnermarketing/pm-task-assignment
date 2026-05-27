> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand — Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Pod Inference

## Purpose

Given a project ID, compute the candidate pool of team members who could plausibly take on new work for that project. Used by `synthesis/matcher.md` when recommending an assignee.

## The computation

The pod for a project is computed fresh every morning from two complementary sources:

1. **Top-down truth** — the org's Pod Matrix (Notion page parsed by `references/pod-matrix.md`), which lists each PM's matrix and the functional matrices (Floaters, Design, QA, Maintenance, AI, SaaS, Marketing, WIC, Hosting, Quoting). The running PM's matrix is the primary candidate pool.
2. **Bottom-up signal** — Orbit followers + recent task assignees on the project (last 6 months). Surfaces people who actually work on this project regardless of formal matrix membership.

Both sources are merged. Matrix membership marks "primary" candidates; Orbit-only history marks "secondary" candidates; people in both rank highest.

Input: a project ID from the Orbit relationship map.

Output: an ordered list of candidate team members, each with a role hint, familiarity score, matrix-membership flag, and (lazily computed) availability score.

## Algorithm

### Step 0 — Load the Pod Matrix (NEW)

Pull the cached matrix output from `references/pod-matrix.md`. Three fallback conditions trigger pure-Orbit pod inference (skip Step 0, run Steps 1–4 against Orbit signals only); the calling matcher logs a one-line note in the Run Log Decisions trace:

| Condition | Reason logged |
|---|---|
| `POD_MATRIX_URL` not in runtime context (interactive surface or routine misconfiguration) | `Pod Matrix not loaded — interactive surface or URL not injected. Using Orbit follower/history inference only.` |
| `notion-fetch` failed after retries | `Pod Matrix fetch failed (<reason>) — using Orbit follower/history inference only.` |
| Page parsed but expected structure missing | `Pod Matrix parse failed — using Orbit follower/history inference only.` |

When the matrix IS available, capture three pools for use in later steps:

- `matrix_pm_pool` — the running PM's matrix members (e.g., Matrix A members for PM Abhishek).
- `matrix_floater_pool` — the Floater / Shared Resources matrix members.
- `matrix_functional_pools` — Design, QA, Maintenance, AI, SaaS, Marketing, WIC, Hosting, Quoting (used only as Uncertain-flagged cross-matrix candidates when Step 6 of `synthesis/matcher.md` falls all the way through).

### Step 1 — Pull the project's people

From the Orbit relationship map (produced by `collectors/orbit.md`), for this project, collect:

- `followers` — people tagged as followers on the project
- `recent_task_assignees` — people who've been assigned tasks on this project in the last 6 months, with their task counts
- `project_owner`, `development_owner`, `account_manager` — for reference, NOT included as candidates (they're oversight, not doers)

Then **union with `matrix_pm_pool`** from Step 0 (when available). Every candidate carries a `source` tag:

- `source: matrix` — listed in the running PM's matrix but no Orbit follower/history signal on this project
- `source: orbit-history` — Orbit follower or recent task assignee but not in the running PM's matrix
- `source: both` — appears in both (highest rank — formally on the pod AND already active on this project)

If the matrix is unavailable, every candidate is implicitly `source: orbit-history` and the union step is a no-op.

### Step 2 — Filter out leadership and AMs

From Preferences, read the `exclude_from_pod` list. Default exclusions:
- Brian Gerstner
- Nishant Rana
- Anyone in the PM's AM list
- The PM themselves

Plus anyone flagged in Preferences' always-exclude. Remove these people from the candidate list.

### Step 3 — Classify each candidate's role

For each remaining candidate, infer their role from three signals (in priority order):

1. **Matrix `role_hint`** (when `source: matrix` or `source: both`) — the role bucket the candidate sits in within their matrix table. Strongest signal — explicit org assignment.
2. **Orbit department** — from `get_user_details`, the user's department (PHP, HTML, QA, Design, Content, etc.). Used as primary when matrix hint absent, as tiebreaker when matrix hint is `Unknown` or `Lead` / `Mentor`.
3. **Task history on this project** — what types of tasks they've been assigned. If someone has 50 FE tasks across projects and 5 BE tasks, they're an FE dev. Useful for floaters and cross-functional contributors.

Map matrix hint + department + task history to a role:
- `FE` — front-end developer (matrix `HTML`)
- `BE` — back-end developer
- `WP` — WordPress specialist (matrix `WordPress / PHP`)
- `Design` — visual / UI designer (matrix `Design` matrix or `(designer)` parenthetical)
- `QA` — QA engineer (matrix `QA`)
- `Content` — content writer / copy (matrix `Marketing` content writer / `Content` department)
- `BA` — business analyst (matrix `Business Analyst`)
- `Full-stack` — broad history across FE + BE (matrix SaaS `Development Team`)
- `Unknown` — can't infer

### Step 4 — Rank by project familiarity

For each candidate, compute a familiarity score based on:

- **Currently active on this project** (assigned to an open task right now) → high weight
- **Has been assigned tasks on this project in the last 3 months** → moderate weight
- **Has been assigned tasks on this project within 3–6 months** → low weight
- **Follower only, no task history** → minimal weight
- **Total task count across all projects** (shows general activity) → small tiebreaker

Higher score = more familiar = more likely pick.

Also set a per-candidate boolean `has_history_on_project` — `true` iff the candidate has at least one task assignment on this project in the last 6 months (i.e., `task_count_on_project_last_6mo > 0`). The matcher uses this boolean to decide whether to take the familiarity-wins path or fall through to the availability-based path.

### Step 5 — Availability score (lazy, on-demand) (NEW)

**Do NOT compute by default.** Availability is checked only on the matcher's no-history fallback path — when no role-fit candidate has `has_history_on_project = true`. Computing for every candidate every run would burn `get_user_workload` calls on candidates the matcher will never need.

Expose a single hook the matcher calls when it hits the fallback:

```
compute_availability(candidate_user_ids: list<int>) → { user_id: availability_score }
```

Implementation:

1. For each `user_id` in the input list, call `mcp__...orbit.get_user_workload`.
2. Apply the standard retry policy from `connector-failure-notify.md`.
3. Convert workload to a normalized `availability_score` ∈ [0, 1] — lower workload = higher score. Concrete formula: `score = 1 / (1 + open_task_count)` (monotonic, bounded, never zero — so a tied highest-loaded candidate still gets a deterministic pick by familiarity tiebreaker).
4. If `get_user_workload` fails for a specific user after retries, set their score to `null` and surface in the matcher's reason as `availability unknown — workload check failed`. The matcher picks among candidates with non-null scores; if all are null, it flags `Uncertain:`.

### Step 6 — Return the ranked list

Output format:

```
{
  "project_id": <int>,
  "project_title": <string>,
  "matrix_available": <bool>,
  "running_pm_matrix": <string or null>,   // e.g., "Matrix A" — null when matrix unavailable
  "candidates": [
    {
      "user_id": <int>,
      "name": <string>,
      "role": "FE" | "BE" | "WP" | "Design" | "QA" | "Content" | "BA" | "Full-stack" | "Unknown",
      "source": "matrix" | "orbit-history" | "both",
      "in_pm_matrix": <bool>,
      "matrix_role_hint": <string or null>,    // verbatim matrix label, e.g., "WordPress / PHP"
      "familiarity_score": <float 0-1>,
      "has_history_on_project": <bool>,
      "active_on_project": <bool>,
      "task_count_on_project_last_6mo": <int>,
      "availability_score": <float 0-1 or null>,   // null until matcher invokes compute_availability
      "reasoning": <string — a one-line human-readable reason>,
      "department": <string — from Orbit>,
      "total_task_load": <int — across all projects, for context>
    }
  ],
  "floater_pool": [<candidate object>, ...],          // matrix-derived; empty when matrix unavailable
  "functional_pools": {                                // for cross-matrix Uncertain surfacing
    "Design": [...], "QA": [...], "Maintenance": [...], "AI": [...],
    "SaaS": [...], "Marketing": [...], "WIC": [...], "Hosting": [...], "Quoting": [...]
  },
  "notes": [<string>, ...]
}
```

When `matrix_available: false`, `running_pm_matrix`, `floater_pool`, and `functional_pools` are all null/empty and every candidate has `source: orbit-history` and `in_pm_matrix: false`.

`reasoning` examples:
- `primary FE on Agency X, 24 hrs on current homepage task, active now`
- `back-end dev with 8 BE tasks on Agency X in the last 3 months, active on 2 tasks now`
- `QA on Agency X, last involved 4 months ago — may be rusty`

## How the matcher uses this output

`synthesis/matcher.md` § Job 6 runs a 4-branch decision tree:

- **(a) History wins.** If at least one role-fit candidate has `has_history_on_project = true`, pick the one with highest `familiarity_score`. No availability check fired.
- **(b) Matrix availability fallback.** If no role-fit candidate has history but the running PM's matrix has at least one role-fit member, the matcher calls `compute_availability` on those members and picks the highest score.
- **(c) Floater fallback.** If the PM's matrix has no role-fit member, the matcher checks `floater_pool`, calls `compute_availability` on Floater role-fit members, and picks the highest score.
- **(d) Cross-matrix Uncertain.** If neither PM-matrix nor Floaters fit, `recommended_assignee = null` and AI Notes lists role-fit candidates from `functional_pools` (Design / QA / Maintenance / AI / SaaS / Marketing / WIC) for the PM to pick.

The PM's note always wins over any of these picks (existing rule).

## Edge cases

### Brand-new project with no task history

If the project has zero task assignees (newly created, no one's been assigned yet), candidates list will be empty or just followers. In that case:

- When matrix is available: return the running PM's matrix members as candidates with `has_history_on_project: false` and `source: matrix`. Matcher's branch (b) fires — calls `compute_availability` and picks lightest-loaded role-fit member. AI Note: `New project — picked by lightest workload from <PM matrix>.`
- When matrix is unavailable: return the project's followers (minus leadership/AMs) as candidates with minimal scores, add a note `New project — no task history, no Pod Matrix. Candidates based only on followers.` Matcher flags `Uncertain:` since no signal can decide.

### PM's matrix has no role-fit member

When the PM's matrix lacks the required role for a task (e.g., Internal Projects matrix gets an HTML task; the matrix has no HTML row):

- Return matrix candidates from `floater_pool` whose `role_hint` matches, marked with note `From Floater matrix — PM's matrix has no <role>`.
- Matcher's branch (c) fires.
- If Floaters also lack the role, return candidates from `functional_pools` for the matcher to surface as Uncertain (branch d).

### Pod member recently left the company

If someone shows up in task history but their Orbit user is marked inactive:

- Exclude them from candidates automatically
- Add a note: `[Name] had task history on this project but is no longer active.`

### Everyone on the project is excluded (all AMs + leadership only)

Rare but possible. Return an empty candidates list with a note: `All observed people on this project are AMs or leadership. No delivery-team candidates found. Please tell me who should pick this up.` Matcher flags `Uncertain:` and PM resolves.

### PM wants to assign outside the pod

The PM's note always wins. If the PM writes `assign to Mannan` and Mannan isn't in the inferred pod, honor it anyway. Log in AI Notes: `Assigned to Mannan per your note — they weren't in the inferred pod for this project.`

## Performance

Should be fast — data is already in memory from the Orbit collector's pass. No new API calls needed unless the user details cache is stale.

## Caching

Between runs, the inferred pod for frequently-used projects can be cached. Invalidate the cache:
- When the Orbit collector's next run sees a new task assignment on that project
- When the PM edits Preferences' exclusion list
- After 7 days (hard TTL)

## Configuration via Preferences

The Preferences page includes these knobs (from first-run setup or later edits):

- `exclude_from_pod` — list of user IDs or names to always exclude (leadership, AMs, ex-employees). Applied AFTER matrix membership is resolved — a person listed in the Pod Matrix but also in `exclude_from_pod` is still excluded.
- `always_consider` — list of user IDs the PM wants always considered even if familiarity is low (e.g., a new hire who needs more work).
- `role_overrides` — per-user role overrides (e.g., "Treat Amit as FE even though Orbit says Back-end — he's transitioning"). Wins over the Step 3 priority order (matrix hint > department > history).

## What this does NOT do

- Does not check availability or capacity proactively. Availability is computed lazily via `compute_availability` only on the matcher's no-history fallback path (per non-negotiable rule #6 in `SKILL.md`).
- Does not consult Keka or any leave data — that connector is not in the source allowlist.
- Does not recommend multiple people per task — only returns the ranked candidate list. The matcher picks one.
- Does not write to the Pod Matrix Notion page — read-only via `references/pod-matrix.md`.
