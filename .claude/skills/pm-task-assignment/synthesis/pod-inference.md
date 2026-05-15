> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

# Pod Inference

## Purpose

Given a project ID, compute the candidate pool of team members who could plausibly take on new work for that project. Used by `synthesis/matcher.md` when recommending an assignee.

## The computation

The pod for a project is NOT stored anywhere. It's computed fresh every morning from Orbit data.

Input: a project ID from the Orbit relationship map.

Output: an ordered list of candidate team members, each with a role hint and a short reasoning string.

## Algorithm

### Step 1 — Pull the project's people

From the Orbit relationship map (produced by `collectors/orbit.md`), for this project, collect:

- `followers` — people tagged as followers on the project
- `recent_task_assignees` — people who've been assigned tasks on this project in the last 6 months, with their task counts
- `project_owner`, `development_owner`, `account_manager` — for reference, NOT included as candidates (they're oversight, not doers)

### Step 2 — Filter out leadership and AMs

From Preferences, read the `exclude_from_pod` list. Default exclusions:
- Brian Gerstner
- Nishant Rana
- Anyone in the PM's AM list
- The PM themselves

Plus anyone flagged in Preferences' always-exclude. Remove these people from the candidate list.

### Step 3 — Classify each candidate's role

For each remaining candidate, infer their role from two signals:

1. **Orbit department** — from `get_user_details`, the user's department (PHP, HTML, QA, Design, Content, etc.).
2. **Task history on this project** — what types of tasks they've been assigned. If someone has 50 FE tasks across projects and 5 BE tasks, they're an FE dev.

Map department + task history to a role:
- `FE` — front-end developer
- `BE` — back-end developer
- `WP` — WordPress specialist
- `Design` — visual / UI designer
- `QA` — QA engineer
- `Content` — content writer / copy
- `Full-stack` — broad history across FE + BE
- `Unknown` — can't infer

### Step 4 — Rank by project familiarity

For each candidate, compute a familiarity score based on:

- **Currently active on this project** (assigned to an open task right now) → high weight
- **Has been assigned tasks on this project in the last 3 months** → moderate weight
- **Has been assigned tasks on this project within 3–6 months** → low weight
- **Follower only, no task history** → minimal weight
- **Total task count across all projects** (shows general activity) → small tiebreaker

Higher score = more familiar = more likely pick.

### Step 5 — Return the ranked list

Output format:

```
{
  "project_id": <int>,
  "project_title": <string>,
  "candidates": [
    {
      "user_id": <int>,
      "name": <string>,
      "role": "FE" | "BE" | "WP" | "Design" | "QA" | "Content" | "Full-stack" | "Unknown",
      "familiarity_score": <float 0-1>,
      "active_on_project": <bool>,
      "task_count_on_project_last_6mo": <int>,
      "reasoning": <string — a one-line human-readable reason>,
      "department": <string — from Orbit>,
      "total_task_load": <int — across all projects, for context>
    }
  ],
  "notes": [<string>, ...]
}
```

`reasoning` examples:
- `primary FE on Agency X, 24 hrs on current homepage task, active now`
- `back-end dev with 8 BE tasks on Agency X in the last 3 months, active on 2 tasks now`
- `QA on Agency X, last involved 4 months ago — may be rusty`

## How the matcher uses this output

`synthesis/matcher.md` takes the top candidate matching the task's role. If the task is FE, pick the top FE candidate. If the task role is unclear, flag `Uncertain:` and leave `recommended_assignee = null`.

## Edge cases

### Brand-new project with no task history

If the project has zero task assignees (newly created, no one's been assigned yet), candidates list will be empty or just followers. In that case:

- Return the project's followers (minus leadership/AMs) as candidates with minimal scores
- Add a note: `New project — no task history. Candidates based only on followers.`
- Matcher will see this and flag AI Notes: `Uncertain: No one has been assigned to this project yet. Please tell me who should pick this up.`

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

- `exclude_from_pod` — list of user IDs or names to always exclude (leadership, AMs, ex-employees)
- `always_consider` — list of user IDs the PM wants always considered even if familiarity is low (e.g., a new hire who needs more work)
- `role_overrides` — per-user role overrides (e.g., "Treat Amit as FE even though Orbit says Back-end — he's transitioning")

## What this does NOT do

- Does not check availability or capacity.
- Does not estimate the person's current workload beyond a coarse "active on project" signal.
- Does not consult Keka or any leave data.
- Does not recommend multiple people per task — only returns the ranked candidate list. The matcher picks one.
