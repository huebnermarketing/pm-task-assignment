> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

# Note Interpreter

## Purpose

Take a short, free-form PM note from a Morning Queue row and resolve it into a concrete action plan the executors can run. Used during Mode 2 execution.

## Input

- The PM's exact note text (string)
- The full row context:
  - Original summary
  - Original recommended action
  - Original recommended assignee
  - Project
  - Source signals (email, Slack, Orbit, Fathom)
  - Proposed Orbit task body (pre-drafted)
  - Proposed Slack handoff (pre-drafted)
  - Proposed email (pre-drafted if any)

## Output

A revised action plan:

```
{
  "action_plan": [
    {
      "type": "create_task" | "reassign" | "update_task" | "add_comment" | "change_due_date" | "create_subtask" | "draft_email" | "send_email" | "send_slack" | "skip",
      "executor": "orbit" | "email" | "slack" | "none",
      "parameters": { ... },
      "why": <string — one-line justification tied to the note>
    }
  ],
  "confidence": "high" | "low",
  "clarification_needed": <string or null>
}
```

If `confidence = low`, the executors do NOT run. Instead, the row's Outcome column is filled with:
> `HELD — I couldn't interpret your note: "[exact note]". Rephrase and run execution again.`

## Common note patterns and how to resolve them

### "assign to X" / "X should do it" / "give it to X"

- Action: change the assignee for the task creation or reassignment
- Look up X in the pod (via `synthesis/pod-inference.md`) — if found, use that user ID
- If X is not in the pod, still honor the override and log `AI Notes: Assigned to X per your note — outside inferred pod.`
- If X is ambiguous (multiple people match), flag clarification: `Multiple people match "X": [list]. Please specify.`

### "save as draft" / "draft for later" / "don't send yet"

- If the original action involved sending an email, convert to draft
- Email Executor creates a Gmail draft instead of sending
- If the original action involved a Slack message, do NOT apply — Slack doesn't have drafts in the skill's scope

### "mark as high priority" / "mark urgent"

- Set `severity_id` to Critical (15) or Important (16) in the Orbit task creation
- Add to Slack handoff: "**Priority:** High"

### "split into two" / "split into X, Y" / "split this"

- Decompose the single task into multiple. The skill should:
  - Parse what the split looks like from the note (if specified)
  - Create multiple Orbit tasks instead of one
  - If split details aren't clear, flag clarification: `Please specify how to split: "[note]" isn't detailed enough.`

### "wait till tomorrow" / "hold for now" / "push to next week"

- Action is: skip for today, but leave a note for tomorrow's Mode 1 to re-surface
- Write a comment on the relevant Orbit task (if it exists): "Held per PM note: [note]. Re-evaluate [date]."
- Row Outcome: `Held per your note. Will re-surface in tomorrow's queue.`

### "this is about project Y, not project X" / "wrong project"

- Re-scope: change the target project for the action
- Look up project Y in the Orbit relationship map
- Regenerate the proposed Orbit task body with Y's context
- Execute against Y instead of X
- Log: `Re-scoped to Project Y per your note.`

### "CC [person]" / "add [person] as CC" / "include [person]"

- For email actions: add the person to CC
- For Slack actions: @-mention the person in the message (or send a parallel message to them)
- Look up the person in Preferences' AM list first, then the Orbit user list

### "remind me later" / "snooze"

- Row stays at current state
- Do NOT execute
- Add note to Orbit task (if one exists): "PM snoozed this on [date] — will re-evaluate"

### "no" / "skip" / "don't do this" / "never mind"

- Convert status to `Skip. No Action Needed`
- Do not execute

### "approved, do it" / "go ahead" / "yes" / "do what you recommended"

- Treat as if the status were `Approved` with no note
- Execute the original recommendation exactly

### "change due date to [date]" / "push to [date]"

- Update the Orbit task's due date to the parsed date
- Parse relative dates (`tomorrow`, `next Friday`) against the current date
- If the note doesn't specify a clear date, flag clarification

## Parsing rules

Use Claude's natural-language understanding — no regex or hard-coded rules. The note is free-form. Resolve intent against the row's full context.

**Always preserve the PM's original wording** in AI Notes so there's an audit trail:
> `AI Notes: PM note honored — "assign to Ravi instead, he knows their codebase"`

## When to ask for clarification (rarely)

The skill only asks for clarification in two cases:

1. **The note is genuinely ambiguous.** Example: `assign this to the new person` — who's "the new person"?
2. **The note contradicts itself.** Example: `mark urgent but save as draft` — urgency usually implies send.

In those cases, set `confidence = low` and populate `clarification_needed` with a specific question. Mode 2 will leave the row untouched and include the question in the post-execution Slack summary.

## What the interpreter does NOT do

- Does not write to any system — it just produces the action plan.
- Does not apply notes to rows not marked `Approved` AND not having a note (i.e., doesn't second-guess the PM's decision to skip).
- Does not interpret notes as "do something different from what you recommended" unless the PM specifically says so. The default is still the original recommendation.

## Examples — full input/output

### Example 1 — simple reassign

**Input:**
- Original recommended assignee: `Vijay Patel (FE) — 12 hrs free this week`
- PM Note: `assign to Ravi instead, he knows their codebase from Phase 1`

**Output:**
```
{
  "action_plan": [
    {
      "type": "create_task",
      "executor": "orbit",
      "parameters": { "assignee": <Ravi's user ID>, "... rest same as original ..." },
      "why": "PM overrode the assignee to Ravi based on Phase 1 codebase familiarity."
    },
    {
      "type": "send_slack",
      "executor": "slack",
      "parameters": { "to": "Ravi", "message": "<regenerated plain-language handoff for Ravi>" },
      "why": "Handoff message re-targeted to Ravi."
    }
  ],
  "confidence": "high",
  "clarification_needed": null
}
```

### Example 2 — save as draft

**Input:**
- Original recommended action: Email Caitlin about Agency Y's scope question
- PM Note: `save as draft — want to run this past Caitlin first`

**Output:**
```
{
  "action_plan": [
    {
      "type": "draft_email",
      "executor": "email",
      "parameters": { "to": "Caitlin's email", "subject": "...", "body": "..." },
      "why": "PM requested draft-only per note."
    }
  ],
  "confidence": "high",
  "clarification_needed": null
}
```
Note the subtle contradiction the PM baked in — they want to run it past Caitlin first, but the email IS to Caitlin. Interpreter still just saves it as a draft. PM will notice in Gmail and decide.

### Example 3 — ambiguous

**Input:**
- PM Note: `handle it`

**Output:**
```
{
  "action_plan": [],
  "confidence": "low",
  "clarification_needed": "Your note 'handle it' is ambiguous. Do you mean: execute as recommended, or do you have specific changes in mind?"
}
```
