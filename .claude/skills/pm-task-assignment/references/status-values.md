# Reference — Status Values

## Morning Queue row Status (the skill's own enum)

Four states. Strict progression.

| Value | Color | Meaning | Who sets it |
|---|---|---|---|
| `Recommended Action` | gray | Default state. Skill recommended something, PM hasn't decided. If left at this state when Mode 2 runs, **nothing happens** for this row. | Mode 1 sets on creation |
| `Approved` | blue | PM approved the recommendation (or note-based override). Mode 2 will execute. | PM flips manually |
| `Done` | green | Mode 2 successfully executed this row. Final state. | Mode 2 sets after execution |
| `Skip. No Action Needed` | orange | PM explicitly chose to skip. Mode 2 does nothing. Final state. | PM flips manually |

### State transitions

```
Recommended Action ─┬─► Approved ──► Done         (happy path)
                    │
                    ├─► Approved + PM note ──► Done   (PM overrode)
                    │
                    ├─► Skip. No Action Needed       (PM skipped)
                    │
                    └─► [stays at Recommended Action] (PM didn't act)
```

### Rules

- Mode 2 only executes rows in `Approved` state (including those with PM notes).
- Mode 2 also executes rows at `Recommended Action` IF the PM added a note (note implies intent).
- `Skip. No Action Needed` is a final state — Mode 2 never touches it.
- `Done` is set by Mode 2 only after all operations for that row succeeded.
- A failed row stays in its prior state (`Approved`) with a failure note in the Outcome column. Running Mode 2 again will retry it.

## Orbit task statuses (the other system)

Orbit has its own task statuses, separate from the skill's Morning Queue status. These are read/written when the skill creates or updates Orbit tasks.

Reference the canonical list via `mcp__...orbit.list_task_status`. Validated values during side quest 3 include:

| Orbit ID | Orbit status name | Typical use |
|---|---|---|
| 0 | Unassigned | Task created without assignee |
| 24 | To Do | Ready to start |
| (varies) | Doing / In Progress | Work has started |
| (varies) | Blocked | Can't proceed — external dependency |
| (varies) | Waiting for Feedback | PM/client response needed |
| (varies) | Client Review | Client is reviewing deliverable |
| (varies) | In Review | Internal review (QA or PM) |
| 29 | Done | Complete |
| (varies) | On Hold | Paused by PM |

Call `mcp__...orbit.list_task_status` on first use to get exact IDs and cache them.

### Rules for the skill

- New Orbit tasks created by the skill default to status `To Do` (ID 24) unless the source signal clearly implies another state (e.g., if reassigning an already-in-progress task, don't reset its status).
- The skill updates task status only when explicitly requested:
  - PM note says "move to In Review" or similar
  - The recommended action itself involves a status change (e.g., "Move CloudBase task to In Review")
- The skill doesn't auto-advance Orbit statuses based on time or activity.

## Orbit task severity (related enum)

Orbit has severity levels. Reference via `mcp__...orbit.list_task_severity`.

Validated values:

| Orbit ID | Severity name | When |
|---|---|---|
| 15 | Critical | Urgent, blocking others |
| 16 | Important | High priority |
| 17 | Normal | Default |
| (varies) | Minor | Low priority |
| (varies) | MVP | Scoped for minimum viable product |

### Rules for the skill

- Default new tasks to `Normal` (17).
- Apply `Critical` (15) or `Important` (16) only when the source signal or PM note implies urgency ("urgent", "critical", "today", "blocker").
- The skill doesn't downgrade severity — PM can do that manually if needed.

## Orbit project statuses

Less commonly needed, but when creating projects:

| Orbit ID | Project status | Use |
|---|---|---|
| 14 | Active | Default for new production projects |
| 21 | Quote | Scoping phase |
| (varies) | On Hold | Paused |
| (varies) | Closed | Finished |
| (varies) | Archived | Historical |

The skill typically creates new projects as `Active` unless the source signal clearly indicates a quote phase.

## Source Systems multi-select values (Morning Queue column)

Four values, one per collector:

- `Orbit` (green)
- `Gmail` (red)
- `Slack` (purple)
- `Fathom` (orange)

Each row's Source Systems field is populated with all collectors that contributed at least one signal to the item.

## What this reference does NOT cover

- Custom Orbit fields or per-project status additions
- Pipedrive statuses (Pipedrive is out of scope for the MVP)
- Keka statuses (Keka is out of scope for the MVP)
- Calendar event statuses (Calendar is out of scope for the MVP)
