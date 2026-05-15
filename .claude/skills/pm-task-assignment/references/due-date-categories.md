# Reference — Due Date Change Categories

Orbit's `change_task_due_date` tool requires a `category_id` from a fixed list. Validated in production during side quest 3 (2026-04-25).

## The 9 categories

| ID | Text |
|---|---|
| 1 | Waiting for access |
| 2 | Waiting for assets |
| 3 | Other critical task/priority |
| 4 | Dev on leave |
| 5 | Server/Infra issue |
| 6 | Waiting for Feedback |
| 7 | In development (QA) |
| 8 | Dev/Html Bug fixes (QA) |
| 9 | Received Client Feedback |

Default fallback: **3** (Other critical task/priority) — use when no other category clearly matches.

## Mapping PM notes to categories

When the skill's Note Interpreter (`synthesis/note-interpreter.md`) processes a PM note that includes a due-date change, it maps the PM's wording to a category.

Common patterns and mappings:

| PM note language | Category |
|---|---|
| "waiting on access / credentials / login / admin panel" | 1 — Waiting for access |
| "don't have the brief / assets / images / design / logo yet" | 2 — Waiting for assets |
| "had to pull [person] onto something more urgent" | 3 — Other critical task/priority |
| "critical fire somewhere else" | 3 — Other critical task/priority |
| "dev is out sick / on vacation / on leave" | 4 — Dev on leave |
| "server down / hosting issue / infrastructure problem" | 5 — Server/Infra issue |
| "waiting on client feedback / client hasn't replied / waiting for the PM's approval" | 6 — Waiting for Feedback |
| "in QA / QA is testing / QA has it" | 7 — In development (QA) |
| "HTML bug fix / needs bug fix / QA found issues" | 8 — Dev/Html Bug fixes (QA) |
| "client gave feedback / client sent revisions / got feedback from the client" | 9 — Received Client Feedback |
| No clear signal from the note | 3 — Other critical task/priority (default) |

## Reason text rules

The `reason` parameter must be a fresh, specific string every time. The tool explicitly rejects reused reasons.

Good reasons:
- `Client requested revised deadline via email on 25 April 2026.`
- `Ravi is on planned leave 26-28 April; pushing to 29 April.`
- `Server migration caused 2-day delay in staging environment.`
- `Received client feedback with new scope items; restructuring timeline.`

Not good:
- `[TEST]` (only appropriate during side quest)
- `Change due date`
- `Per PM request`
- Empty string

The skill should always include:
- The PM's rationale (paraphrased from their note, or inferred from the source signal)
- A specific date if the PM mentioned one
- A specific person if the PM mentioned one

## Calling the tool

```javascript
change_task_due_date({
  task_id: <int>,
  due_date: "<YYYY-MM-DD>",
  reason: "<fresh, specific string>",
  category_id: "<string — one of '1' through '9'>"
})
```

Note: `category_id` is a STRING in the tool spec, not a number.

## Error handling

If the tool rejects a call (e.g., due to a malformed reason or invalid category), the skill:

1. Retries ONCE with fallback category 3 and a more conservative reason
2. If still fails, logs in the row Outcome: `FAILED — due date change rejected by Orbit. Please change manually.` and continues with other operations for that row

Do NOT retry more than once, to avoid tool abuse.

## When to NOT change the due date

- If the task doesn't have a due date to start with, changing it via this tool may fail. Use `update_task` with `start_date` / other fields instead.
- If the PM note is ambiguous about whether to change the date, skip the date change and note it in the Outcome.

## What this reference does NOT cover

- Project due date changes (use `change_project_due_date`, same category list but different tool).
- Retroactive date changes (same tool, same categories — no special handling).
- Bulk date updates across multiple tasks (not supported by the skill in v1).
