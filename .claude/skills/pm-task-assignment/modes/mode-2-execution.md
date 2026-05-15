> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

# Mode 2 — Execution Run

## When this runs

- Scheduled: daily at the PM's configured execution time (default 10:45 AM IST).
- Manual: `PM Task Assignment, run execution now`.

## What Mode 2 does

Reads today's dated page in Notion. Checks the page-level `Ready for Execution` toggle at the top. If ON, executes every approved row and every row with a PM note. Writes outcomes back. Slacks the PM a completion summary. If OFF, fires escalation per `modes/mode-3-escalation.md`.

## End-to-end flow

### Step 1 — Read Preferences

Identity, AM list, communication defaults, always-include rules, tone samples. All used downstream.

### Step 2 — Locate today's dated page

Search the PM's Notion parent for a sub-page titled with today's date in "DD Month YYYY" format. If it doesn't exist, either Mode 1 never ran today or the PM deleted it. Abort with a Slack to the PM: "I don't see today's morning queue. Did Mode 1 run? Try `PM Task Assignment, run morning`."

### Step 3 — Check the Ready for Execution toggle

The toggle is a checkbox block at the top of the dated page. Fetch the page content and find the to-do block labeled `Ready for Execution`.

- **If unchecked (OFF):** route to `modes/mode-3-escalation.md` and stop.
- **If checked (ON):** proceed.

### Step 4 — Read the Morning Queue database

Fetch every row. For each row, read:
- Summary
- Status (`Recommended Action` / `Approved` / `Done` / `Skip. No Action Needed`)
- Recommended Action
- Recommended Assignee
- PM Notes
- Project
- Row ID (for updating later)
- Row page content (for context the executors will need)

### Step 5 — Classify each row

For each row, decide what to do:

| Row state | What Mode 2 does |
|---|---|
| `Skip. No Action Needed` | Skip. Log: "Skipped per PM." Move on. |
| `Recommended Action` + no note | Skip. PM didn't approve. Log: "Not approved — skipped." |
| `Recommended Action` + note | Honor the note. Route to `synthesis/note-interpreter.md`. |
| `Approved` + no note | Execute the skill's original recommendation as-is. |
| `Approved` + note | Honor the note (note wins over status). Route to `synthesis/note-interpreter.md`. |
| `Done` | Already executed in a prior run. Skip. |

### Step 6 — Execute per row

For each row that needs action:

1. Load the row's detail page content to understand full context (Sources, proposed Orbit task body, proposed Slack, proposed email).
2. If there's a PM note, feed it to `synthesis/note-interpreter.md` to resolve the intent and override parts of the recommendation.
3. Route to the appropriate executors based on the action type:
   - **Orbit operations** (create task, assign, update status, change due date, comment, create project, subtask) → `executors/orbit.md`
   - **Email** (drafts only, never auto-send) → `executors/email.md`
   - **Slack** (team handoff, AM ping, PM self-summary) → `executors/slack.md`
4. Before calling any executor, pass every string the assignee (delivery team) will read through `writers/plain-language.md` to enforce 4th–5th grade English.
5. Pass every source reference through `writers/source-citation.md` to ensure proper citation format.

### Step 7 — Update the row after execution

For each row, after all its actions execute:

1. Flip `Status` to `Done`.
2. Write the `Outcome` column with a short line describing what was done. Format: concise, specific, include Orbit links where applicable.
   - Examples:
     - `Task created → Orbit #105892. Slack sent to Vijay. Caitlin emailed.`
     - `Reassigned from Amit → Rohit. Slack sent to Rohit with context.`
     - `Email drafted (saved to Gmail drafts — not sent per your note).`
     - `Task status updated in Orbit. Mannan notified.`

### Step 8 — Handle failures per row

If execution of a row fails mid-way (MCP unavailable, note un-interpretable, Orbit API error):

- Do NOT set `Status` to `Done`. Leave it at whatever state it was in.
- Write a failure note in the `Outcome` column: `FAILED — [reason]. Will retry if re-approved.`
- Log the failure. Continue with remaining rows — don't abort the whole run for one row's failure.

### Step 9 — Post-execution Slack summary to the PM

After all rows are processed (whether success or fail), send a Slack DM to the PM summarizing:

```
Your morning queue executed.

✅ 7 of 9 actions taken:
  • Agency X homepage task → Vijay
  • DigitalFirst 3 action items → Ravi + Amit
  • GrowthLab blog → Ravi (per your note)
  • CloudBase QA handoff → Mannan
  • Reply drafted to Jane (Agency X, saved as draft)
  • AM email sent to Caitlin

⚠️ 1 held back:
  • TechCo budget alert — your note said "save as draft, check with Caitlin first". Draft saved in Gmail. No other action taken.

❌ 1 failed:
  • BrightPath proposal — Gmail MCP timed out. Retry by typing `PM Task Assignment, run execution now` or wait for tomorrow.

All Orbit links + Slack thread links in today's Notion page.
```

Use `executors/slack.md` to send this DM.

### Step 10 — Update Preferences

Set `last_execution_run` timestamp in Preferences.

### Step 11 — Exit

Mode 2 is complete. No further action.

## What Mode 2 does NOT do

- Does not re-pull signals from collectors. It only acts on what Mode 1 already gathered.
- Does not ask the PM questions mid-run.
- Does not flip a row to `Done` until its execution succeeded.
- Does not auto-send client-facing emails. Emails are drafts only — even when `Approved`.
- Does not execute rows that are `Recommended Action` without a note.
- Does not override `Skip. No Action Needed` rows under any circumstance.

## Special case — row with a note but ambiguous intent

If `synthesis/note-interpreter.md` can't resolve the PM's note with confidence, do not execute. Leave the row at its current state, write to the `Outcome` column: `HELD — I couldn't interpret your note: "[exact PM note]". Rephrase and run execution again, or edit the row.` Include this in the post-execution summary.

## Special case — Mode 2 fires but Mode 1 never ran today

If there's no dated page for today, exit cleanly with a Slack to the PM: "No morning queue found for today. Mode 1 either didn't run or was skipped. Run `PM Task Assignment, run morning` to generate one."

## Special case — manual fire of Mode 2 when Ready is OFF

If the PM manually fires Mode 2 via the command but Ready is OFF, don't auto-escalate. Instead, confirm: "Ready for Execution is still off. Flip it in Notion and I'll continue, or say 'cancel'." Wait for the PM's reply before doing anything.
