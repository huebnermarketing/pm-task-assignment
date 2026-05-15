# Invocation Commands

All user-facing commands start with the skill name so Claude routes correctly.

## The commands

### `PM Task Assignment, run morning`

Manually fires Mode 1 (morning collection). Use when:
- The scheduled morning run didn't fire
- The PM wants an ad-hoc morning read (e.g., coming back from leave, or it's mid-day and they want to re-collect)

Loads: `modes/mode-1-morning-collection.md`

Behavior:
- Read Preferences. If not found, route to `first-run-setup.md` instead.
- If today's dated page already exists, confirm with the PM before overwriting: "You already have a morning queue for today. Overwrite or skip?"
- Otherwise, run the full Mode 1 flow.

### `PM Task Assignment, run execution now`

Manually fires Mode 2 (execution). Use when:
- The PM flipped the Ready toggle after the scheduled Mode 2 time passed
- The scheduled run didn't fire
- The PM wants to execute early

Loads: `modes/mode-2-execution.md`

Behavior:
- Read Preferences.
- Read today's dated page.
- Check the Ready toggle.
- If ON: execute approved rows + rows with notes. Slack the PM the summary.
- If OFF: confirm with the PM — "The Ready toggle is still off. Flip it and I'll continue, or say 'cancel'."

### `PM Task Assignment, change my preference: [instruction]`

Edits Preferences only. Does not fire any morning flow.

Loads: Preferences page via `schemas/preferences-page.md`.

Behavior:
- Parse the instruction (free-form natural language).
- Confirm back to the PM what you're about to change: "You want me to change [X] to [Y]. Sound right?"
- On confirmation, update the Preferences sub-page.
- If the change affects the scheduled tasks (e.g., "change my morning run to 10:00 AM"), update the registered scheduled-task via `mcp__scheduled-tasks__update_scheduled_task`.

Examples:

- `PM Task Assignment, change my preference: run morning at 10:00 AM instead of 9:30`
- `PM Task Assignment, change my preference: always CC Nishant on emails to leadership`
- `PM Task Assignment, change my preference: my escalation backup is Hiten now, Slack him at 11:30 AM`
- `PM Task Assignment, change my preference: add a rule to always link the related Orbit task when I reassign work`

### `PM Task Assignment, update tone samples`

Adds or replaces tone calibration samples used by the plain-language writer.

Loads: `writers/plain-language.md` (for context) and Preferences page (for storage).

Behavior:
- Ask: "Paste 3 to 5 real Slack messages or emails you've recently sent to your team. I'll use them to match your tone."
- Confirm sample count and replace or append (PM picks).
- Update the Tone Samples section of Preferences.

## Routing rules

When Claude receives a user message, check for these routing conditions in order:

1. Does the message start with `PM Task Assignment,` followed by one of the known command phrases? → route directly.
2. Is Claude being invoked by a scheduled-tasks trigger that registered with this skill? → route to the appropriate mode based on the scheduled task's metadata.
3. Does the message match intent patterns like "run my morning queue", "generate today's assignments", "handle my actions"? → ask: "Did you mean `PM Task Assignment, run morning` or `PM Task Assignment, run execution now`? I'll run whichever you confirm."
4. Does the message mention preferences, setup, or first run? → route to `first-run-setup.md` if Preferences doesn't exist, otherwise route to the `change my preference` flow and ask what they want to change.

## Command phrases that are NOT in v1 (but on the roadmap)

These were deferred to post-pilot:

- `PM Task Assignment, handle my actions` — alternate Mode 2 trigger phrase (future muscle-memory option)
- `PM Task Assignment, approve all` — bulk-approve all Recommended Action rows at once
- `PM Task Assignment, skip tomorrow` — planned-leave skip, disables next day's run

If a PM types one of these in v1, respond: "That command isn't in v1 — it's on the roadmap. For now, [here's the workaround]."

## Non-command mentions

If a PM mentions the skill casually ("how's the PM Task Assignment skill going?" or "what does the skill do?") without an action verb, answer conversationally. Don't auto-fire anything. Ask what they want: "Looking for the morning run? Execution? A preference change? Or just info?"
