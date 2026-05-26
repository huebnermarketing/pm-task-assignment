# Invocation Commands

> **Out-of-routine commands only.** Routines fire by cron with a baked prompt and never parse user messages. The commands below are for an operator running this skill manually in a Claude.ai session — typically for catch-up runs after a missed schedule, or to validate setup.

## When you'd use these commands

- A scheduled routine missed its fire window (Claude Routines outage, infra hiccup, model maintenance).
- The PM wants an ad-hoc morning read after returning from leave, before tomorrow's regular fire.
- The operator is validating a fresh setup before the first scheduled fire — a dry run to confirm Preferences and the setup checklist are wired correctly.
- The PM flipped the `Ready for Execution` toggle after the Mode 2 cron already passed and wants execution to proceed today rather than waiting until tomorrow.

In all four cases the operator opens a regular Claude.ai chat session with the skill loaded and types one of the commands below. Routines never use this path.

## The commands

### `PM Task Assignment, run morning`

Manually fires Mode 1 (morning collection) for today.

Loads: `preflight.md`, then `modes/mode-1-morning-collection.md`.

Behavior:
- Run preflight against the Preferences page. Abort on any failure exactly as a routine would.
- If today's dated page already exists on the parent, ask the operator: "A morning queue for today already exists. Overwrite or skip?" (this is the only place a manual run pauses for input — routines never reach this branch because they fire before the page exists.)
- Otherwise run Mode 1 end-to-end and write the dated page.

### `PM Task Assignment, run execution now`

Manually fires Mode 2 (execution) for today.

Loads: `preflight.md`, then `modes/mode-2-execution.md`.

Behavior:
- Run preflight. Abort on failure.
- Read today's dated page. If it doesn't exist, abort with "no morning queue for today — run morning first."
- Check the `Ready for Execution` toggle. If ON, execute approved rows + rows with notes and write the completion summary as a Notion callout at the top of today's dated page. If OFF, ask the operator: "Ready toggle is off. Flip it now and reply `go`, or reply `cancel`."

### `PM Task Assignment, validate setup`

Read-only setup check. Does not write anywhere.

Loads: `preflight.md` only.

Behavior:
- Run preflight against the Preferences page.
- Report each validation rule with a pass/fail line — checklist boxes, required fields, time formats, escalation-not-self, canonical email domain, AM rows, schema parseability.
- Print a final `READY` or `NOT READY: <reasons>`.
- Do not write a Run Log entry, do not touch the dated page, do not fire any mode.

Use this immediately after deploying a new PM, before the first scheduled fire, to confirm the Notion side is wired.

## Removed from v1

These were in earlier drafts and have been **removed**:

- `PM Task Assignment, change my preference: ...` — preferences are now edited directly in Notion on the Preferences page. The next routine fire picks up the change. No chat command needed.
- `PM Task Assignment, update tone samples` — tone samples are pasted directly into the `Tone Samples` toggle block on the Preferences page. The next routine fire reads them. No chat command needed.

If an operator types either of these, respond: "That command was removed in v1 — edit the Preferences page in Notion directly. Changes apply on the next routine fire."

## Routing rules

For manual-session routing only:

1. Match the skill name `PM Task Assignment` followed by an action verb (`run morning`, `run execution now`, `validate setup`) anywhere in the user message. Be lenient: case-insensitive, the comma is optional, extra whitespace is fine.
2. On match, route to the corresponding mode/preflight file as listed above.
3. If the message mentions the skill name but doesn't match a known command, list the three valid commands and ask which one the operator wants.
4. If the message doesn't mention the skill name at all, do not auto-fire anything — answer conversationally.

## What routines do instead

Routines fire on cron from the Claude Routines runner. Each routine has a baked prompt that loads this skill from the public GitHub repo and directly invokes its assigned mode file (`modes/mode-1-morning-collection.md`, `modes/mode-2-execution.md`, or `modes/monthly-archival.md`) after preflight passes. Routines do not read the routing tables in this file, do not parse user messages, and never reach the interactive branches above. The full per-routine prompt structure lives in `ROUTINE-ENTRYPOINTS.md`.

---

> **If invoked from a routine context, ignore the routing tables here — the routine prompt directly invokes the appropriate mode file.**
