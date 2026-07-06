> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand — Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

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
  - Source signals (email, Orbit) + any Fathom enrichment fetched by matcher Job 4b Pass 2
  - Proposed Orbit task body (pre-drafted)
  - Proposed handoff (pre-drafted; copied + sent by PM through whatever channel they use)
  - Proposed email (pre-drafted if any)

## Output

A revised action plan:

```
{
  "action_plan": [
    {
      "type": "create_subtask" | "create_parent_task" | "update_task" | "add_comment" | "change_due_date" | "regenerate_handoff" | "skip",
      "executor": "orbit" | "email" | "notion" | "none",
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

- Action: change the assignee for the sub-task being created
- Look up X in the pod (via `synthesis/pod-inference.md`) — if found, use that user ID
- If X is not in the pod, still honor the override and log `AI Notes: Assigned to X per your note — outside inferred pod.`
- If X is ambiguous (multiple people match), flag clarification: `Multiple people match "X": [list]. Please specify.`

### "save as draft" / "draft for later" / "don't send yet"

- If the original action involved sending an email, convert to draft
- Email Executor creates a Gmail draft instead of sending
- Team handoff drafts are already draft-only by default (PM copies + sends manually), so this note is a no-op for them

### "mark as high priority" / "mark urgent"

- Set `severity_id` to Critical (15) or Important (16) in the Orbit task creation
- Add to the handoff draft: "**Priority:** High"

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

### "create it on [project / #number / client]" / "create anyway" / "new task on [X]"

Resolves a Possible-Orbit-miss **Flag** row (project not found, or duplicate-suspected) into an actual parent-task creation. Two entry shapes:

- **Project named** (`create it on Agency X` / `new task on #16915` / `create under Boyar Miller`) — resolve the named project:
  - First the Orbit relationship map. If absent, `list_projects(search_value=<name/number>)` and/or `list_clients(search_value=<name>)` → `get_client_details(company_name=<name>, include_projects_summary=true)`. Pick the single match.
  - Then run the dedup check `get_project_task_list(project_id, search=<topic keywords>, is_completed="incomplete")` UNLESS the note also says "anyway" (PM has overridden dedup). If a topic match still exists and the PM did not say "anyway", set `confidence: low`, `clarification_needed: "#<id> '<title>' on <project> may already cover this — reply 'create anyway' to add a new task."`
- **"create anyway"** on a duplicate Flag (the row already names the resolved project in its context) — skip dedup, use the row's resolved `project_id`.

On success, emit:
```
{
  "type": "create_parent_task",
  "executor": "orbit",
  "parameters": {
    "project_id": <resolved>,
    "title": <row.task_title, or regenerated from the signal subject>,
    "description": <row.proposed_orbit_body, regenerated for the resolved project if the project changed>,
    "assignee": <PM_user_id>,
    "parent_id": null
  },
  "why": "PM directed parent-task creation on <project> per note."
}
```

If the named project can't be resolved: `confidence: low`, `clarification_needed: "Which project? I couldn't find '<name>' in Orbit."` See `executors/orbit.md` § Create a parent task for the create_task call.

### "delegate to X" / "hand to <pod>" / "give this to <dev>" / "needs a dev" (Delegate a due-today Flag)

**Entry condition — widened.** This promotion fires when the row verb is `Flag` AND EITHER (a) the row's signal is `pm_task_due_today` (a bare due-today Flag — a task due today that the PM has now decided needs delegating), OR (b) the row carries `fc_row_type: stale_work_ping` (a fixed-cost stale-work ping the PM has decided needs a dev). Cases (a) and (b) are two of only THREE verb-changing promotions allowed in the whole skill (the third is the Possible-Orbit-miss create path above). For any other Flag class, a delegation note does NOT change the verb — return `confidence: low`, `clarification_needed: "This row is flagged for your own action; I can't auto-delegate it. If you want a sub-task, tell me the project and pod."`

When the entry condition holds and the note expresses delegation intent, resolve it like the matcher would have if there had been a real ask:

1. **Upgrade the light read to a full deep-read.** The Mode-1 light read fired `get_task_details(task_id)` but only used its newest bundled comments. Now read the **full** bundled comment tree from that same `get_task_details` response (flattened, all-time — it already contains the complete history), and fire `list_task_comments(task_id)` only as a fallback if that tree looks truncated, so the sub-task body carries complete context. Attach any `context_signals[]` / Fathom already on the row (usually none for a bare due-today task).
2. **Classify work_type** (matcher Job 5a) from the task body + comments + **the PM note itself** (the note often names the work or pod — `hand to FE` → `HTML_CSS`, `QA this` → `QA`, `needs a quote` → `QUOTE`).
3. **Pick the verb** via the Job 5 work-type → verb table: pod work (`HTML_CSS` / `PHP_BACKEND` / `QA`) → `Create subtask` under the row's task (the due-today task, or — for a `stale_work_ping` promotion — the stale task named on the row; either way it is the PM-owned parent, so `parent_id = task_id`); non-pod work (`AUDIT` / `QUOTE` / `SEO` / `DESIGN` / `CONTENT` / `BA`) → `Hand off parent task` (reassign the task itself to the pool lead).
4. **Existing-subtask check** (matcher Job 5.5): if an open subtask of the matching work_type already sits under the task (from the deep-read `subtasks[]`), emit `Reopen subtask` instead of a duplicate — reassign to the PM-named dev, else the last non-PM commenter; inactive dev → clarification.
5. **Resolve the assignee.** PM named a specific dev → honor it (look up via `pod-inference.md`; ambiguous or inactive → clarification, same rule as "assign to X"). No name (generic `delegate this` / `needs a dev`) → run the Job 6 pod-boundary tree; if it lands on `Uncertain`, return `confidence: low`, `clarification_needed: "Which pod or person should own <task>? I can delegate but couldn't infer it from the note."` — never guess a delegate.
6. **Compose the 6-section body** (matcher Job 7) from the full deep-read.

On success, emit the executor action for the resolved verb (`create_subtask`, or `update_task` + `add_comment` for `Reopen subtask` / `Hand off parent task` per `executors/orbit.md`), plus a `regenerate_handoff` action so a fresh handoff draft is appended under the row Outcome. Example (pod work, named dev):

```
{
  "action_plan": [
    {
      "type": "create_subtask",
      "executor": "orbit",
      "parameters": { "parent_id": <due-today task_id>, "assignee": <Vijay's user ID>, "description": "<6-section body from full deep-read>", "work_type": "HTML_CSS" },
      "why": "PM delegated this quiet due-today task to the FE pod per note 'hand to FE'; promoted Flag → Create subtask."
    },
    {
      "type": "regenerate_handoff",
      "executor": "notion",
      "parameters": { "to": "Vijay", "message": "<plain-language handoff>" },
      "why": "Handoff draft for the promoted sub-task, appended under the row Outcome for the PM to copy + send."
    }
  ],
  "confidence": "high",
  "clarification_needed": null
}
```

If the note is delegation-shaped but the work_type cannot be inferred AND no dev/pod is named → `confidence: low`, `clarification_needed: "I can delegate <task>, but tell me the pod or person — I couldn't tell from 'delegate this' what kind of work it is."`

### "CC [person]" / "add [person] as CC" / "include [person]"

- For email actions: add the person to CC
- For team handoff drafts: name the person in the handoff body and add their email below the draft so the PM can include them when copying + sending
- Look up the person in Preferences' AM list first, then the Orbit user list

### "remind me later" / "snooze"

- Row stays at current state
- Do NOT execute
- Add note to Orbit task (if one exists): "PM snoozed this on [date] — will re-evaluate"

For Flag rows carrying `fc_row_type: follow_up_reminder`, the fixed-cost snooze section below governs instead (it bumps the ask's reminder clock rather than leaving a follow-up note).

### "resolved" / "client sent it" / "got it" / "no longer needed" (fixed-cost reminder rows)

Applies to Flag rows with `fc_row_type: follow_up_reminder`. Mode 2 sets the matching
Client-Ask Ledger row (on the Fixed-Cost Registry page) to `Status: Resolved-manual`, notes
the PM note text in `Resolved by`, and marks the queue row `Done — ask closed by PM`. No
Orbit write. `no longer needed` resolves identically (the ask is moot — same end state).

**Where `ask_key` comes from.** The interpreter does not derive `ask_key` itself — it reads
the `ask_key: <value>` line off the triggering row's detail page, inside its bottom
Reference Context toggle (`schemas/row-detail-page.md` § Bottom toggle), per
`writers/notion.md` § Flow — Mode 2 ledger touch. That line was written at Mode 1 render
time from the row candidate's own `ask_key` field (`synthesis/fixed-cost-state.md` § FC-3).

Emits:
```
{
  "type": "fc_ledger_update",
  "ask_key": "#<project_number>|<Asked on>|<first 40 chars>",
  "status": "Resolved-manual",
  "resolved_by": "<exact PM note text>",
  "resolved_on": "<today, YYYY-MM-DD>"
}
```
Executed via `writers/notion.md` § Flow — Mode 2 ledger touch, which writes `Status`,
`Resolved by`, and the `Resolved on` column (YYYY-MM-DD) on the matching Client-Ask Ledger row.

### "snooze 2 weeks" / "remind me in <period>" (fixed-cost reminder rows)

Bump the ask's `fc_state` `last_reminder` forward so the next reminder fires after the
requested period (e.g. `snooze 2 weeks` → `last_reminder = today + 14d −
follow_up_reminder_days`). Ledger `Last reminder` mirrors it. Row → `Done — snoozed`.
`ask_key` is sourced the same way as the resolve pattern above — the row detail page's
bottom Reference Context toggle.

Emits:
```
{
  "type": "fc_ledger_update",
  "ask_key": "#<project_number>|<Asked on>|<first 40 chars>",
  "snooze_until_shift": "<period, e.g. '2 weeks'>"
}
```
Executed via `writers/notion.md` § Flow — Mode 2 ledger touch, which bumps the ask's
`fc_state` `last_reminder` and mirrors the ledger's `Last reminder` field. No `Resolved on`
write on a snooze (the ask is still open).

On `fc_row_type: stale_work_ping` rows, the existing verbs apply unchanged: `delegate to
<dev>` promotes per § Delegate a due-today Flag; `skip` / `no` marks Skip. `resolved` on a
ping row just marks the row Skip (there is no ledger entry behind a ping).

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
- Pass the task's current `due_date` state into the executor so it can pick Path A (initial set, null → date, no reason needed) vs Path B (date change, fresh reason + category required). See `executors/orbit.md` § Change task due date and `references/due-date-categories.md`.

## Parsing rules

Use Claude's natural-language understanding — no regex or hard-coded rules. The note is free-form. Resolve intent against the row's full context.

**Always preserve the PM's original wording** in AI Notes so there's an audit trail:
> `AI Notes: PM note honored — "assign to Ravi instead, he knows their codebase"`

## When to ask for clarification (rarely)

The skill only asks for clarification in two cases:

1. **The note is genuinely ambiguous.** Example: `assign this to the new person` — who's "the new person"?
2. **The note contradicts itself.** Example: `mark urgent but save as draft` — urgency usually implies send.

In those cases, set `confidence = low` and populate `clarification_needed` with a specific question. Mode 2 will leave the row untouched and include the question in the post-execution Notion callout summary at the top of today's dated page.

## What the interpreter does NOT do

- Does not write to any system — it just produces the action plan.
- Does not apply notes to rows not marked `Approved` AND not having a note (i.e., doesn't second-guess the PM's decision to skip).
- Does not interpret notes as "do something different from what you recommended" unless the PM specifically says so. The default is still the original recommendation.

## Examples — full input/output

### Example 1 — assignee override on a sub-task

**Input:**
- Row Summary (topic-style): `Swap Contact Form brochure PDF — Solstice WP #16720`
- Row Recommended Action: `Create subtask under #106447, assign to Atul (WP) + handoff draft`
- PM Note: `assign to Ravi instead, he knows their codebase from Phase 1`

**Output:**
```
{
  "action_plan": [
    {
      "type": "create_subtask",
      "executor": "orbit",
      "parameters": { "parent_id": 106447, "assignee": <Ravi's user ID>, "... rest same as original ..." },
      "why": "PM overrode the assignee to Ravi based on Phase 1 codebase familiarity."
    },
    {
      "type": "regenerate_handoff",
      "executor": "notion",
      "parameters": { "to": "Ravi", "message": "<regenerated plain-language handoff for Ravi>" },
      "why": "Handoff draft re-targeted to Ravi; appended under the row Outcome for PM to copy + send."
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
