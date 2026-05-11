> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes routine triggers — preflight runs even when invoked by a scheduled cloud routine.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Mode 2 — Execution Run

## When this runs

- Scheduled: daily at the PM's configured execution time (default 10:45 AM IST), fired by a Claude Routine.
- Manual: `PM Task Assignment, run execution now`.

## What Mode 2 does

Reads today's dated page in Notion. Checks the page-level `Ready for Execution` toggle at the top. If ON, executes every approved row and every row with a PM note. Writes outcomes back. Slacks the PM a completion summary. If OFF, fires the inline escalation flow (Step 3a) and exits — routines cannot wait for the PM to flip a toggle.

Mode 2 is identical across the two execution surfaces (headless vs interactive — see `ENVIRONMENTS.md`). The only difference: in interactive mode, when a PM Note is genuinely ambiguous and `synthesis/note-interpreter.md` returns `confidence = low`, the skill may ask the PM one clarifying question before falling back to `HELD`. In headless mode it always falls back to `HELD`.

## End-to-end flow

### Step 1 — Read Preferences

Identity, AM list, communication defaults, always-include rules, tone samples. All used downstream.

### Step 2 — Locate today's dated page

Search the PM's Notion parent for a sub-page titled with today's date in "DD Month YYYY" format. If it doesn't exist, either Mode 1 never ran today or the PM deleted it. Abort with a Slack to the PM: "I don't see today's morning queue. Did Mode 1 run? Try `PM Task Assignment, run morning`."

### Step 3 — Check the Ready for Execution toggle

The toggle is a checkbox block at the top of the dated page. Fetch the page content and find the to-do block labeled `Ready for Execution`.

- **If unchecked (OFF):** fire the inline escalation flow (Step 3a below), append a run-log entry via `writers/run-log.md` recording outcome `escalated` (with reason `ready_toggle_off`), and exit. Do NOT prompt the PM to flip the toggle. Do NOT wait. Routines cannot block on human input — if the PM never approved by execution time, the backup owns it.
- **If checked (ON):** proceed to Step 4.

### Step 3a — Escalation flow (only when Ready=OFF)

Replaces the former separate `mode-3-escalation.md` file. Runs inline in Mode 2.

#### 3a.1 — Pull Preferences fields

- PM identity (for the backup message)
- Escalation backup: name, channel (Slack or email), ping time
- Today's dated page URL (the page just fetched in Step 2)

#### 3a.2 — Edge cases (handle before composing)

- **No backup configured.** Slack the PM: "I noticed the Ready toggle isn't on and there's no escalation backup configured. Today's queue was not executed. Add a backup via `PM Task Assignment, change my preference: add [name] as my escalation backup`." Skip 3a.3–3a.5 and exit.
- **Backup is the PM themselves.** Slack the PM: "Your escalation backup is set to yourself. That won't work. Set someone else via `PM Task Assignment, change my preference: change my escalation backup to [name]`." Skip 3a.3–3a.5 and exit.
- **Backup Slack handle invalid.** Fall back to email if email is configured. If neither works, Slack the PM that escalation failed and why; exit.

#### 3a.3 — Compose the escalation message

For Slack:

```
[PM name] hasn't marked today's morning queue ready for execution by [execution time] IST.

This means one of two things:
  • They're unavailable today (sick, emergency, meeting).
  • They forgot to flip the Ready toggle.

Can you cover? Review the queue and either:
  • Flip the Ready toggle yourself if everything looks good — the next scheduled execution check picks it up.
  • Act on the items manually if you'd rather handle them yourself.

Morning queue: [link to today's dated page]
Items count: [N]
```

For email (when backup channel is email):

```
Subject: Morning queue not marked ready — can you cover for [PM name]?

Hi [Backup name],

[PM name] hasn't marked today's morning queue ready for execution by [execution time] IST. They may be unavailable or forgot to flip the toggle.

Would you mind covering? Open the queue and either:
  - Flip the "Ready for Execution" toggle at the top of the dated page if the recommendations look right — the next scheduled execution check picks it up.
  - Act on any items manually if you'd rather handle them yourself.

Queue: [link to today's dated page]
Items count: [N]

Thanks,
PM Task Assignment skill
(Automated message on behalf of [PM name])
```

#### 3a.4 — Send

- If channel = Slack: use `executors/slack.md` to send to the backup's Slack handle. (Escalation Slack send is one of the few Slack send paths still allowed — it is operational, not a team handoff.)
- If channel = email: use `executors/email.md` to send directly. (Operational escalation is one of the documented send-not-draft exceptions in `executors/email.md`.)

#### 3a.5 — Log + notify the PM

- Add a note to today's dated page, under the summary line: `Escalation fired at [time] IST. [Backup name] notified via [channel].`
- Slack DM the PM: `I escalated today's morning queue to [backup name] at [time] IST because the Ready for Execution toggle was still off. If this was unintentional, flip the toggle now and type \`PM Task Assignment, run execution now\` to proceed.`
- Append a Run Log entry via `writers/run-log.md` with `outcome: escalated`, `reason: ready_toggle_off`, target backup recorded.
- Exit Mode 2. The backup or PM owns the next move.

#### 3a.6 — Hourly re-check (light)

For 3 hours after escalation, every hour: re-check the Ready toggle silently. If someone flips it ON, fire Mode 2 from Step 4 automatically. After 3 hours, stop re-checking; the rest of the day is manual via `PM Task Assignment, run execution now`. Send the escalation message at most once per day.

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
   - **Orbit operations** (create task, assign, update status, change due date, comment, create project, subtask) → `executors/orbit.md`. Due date paths A/B per `references/due-date-categories.md`.
   - **Email** (drafts only, never auto-send except the documented exceptions) → `executors/email.md`
   - **Slack** — team and AM handoffs default to `Slack draft appended to Notion` (no Slack API send). Real send only when allowed: PM self-summary, escalation backup ping, or explicit PM `send` note with audience = team. See `executors/slack.md`.
4. Before calling any executor, pass every string the assignee (delivery team) will read through `writers/plain-language.md` to enforce 4th–5th grade English.
5. Pass every source reference through `writers/source-citation.md` to ensure proper citation format.

### Step 7 — Update the row after execution

For each row, after all its actions execute:

1. Flip `Status` to `Done`.
2. Write the `Outcome` column with a short line describing what was done. Format: concise, specific, include Orbit links where applicable.
   - Examples:
     - `Task created → Orbit #105892. Slack draft for Vijay appended below. Email draft to Caitlin saved to Gmail.`
     - `Reassigned from Amit → Rohit. Slack draft for Rohit appended below.`
     - `Email drafted (saved to Gmail drafts — not sent per your note).`
     - `Task status updated in Orbit. AM draft appended below.`
3. If a Slack team / AM handoff was produced, the writer appends the draft body under the row's detail page (per `writers/notion.md` Flow — updating rows after Mode 2). The PM copies it from Notion into Slack on their own.

### Step 8 — Handle failures per row

If execution of a row fails mid-way (MCP unavailable, note un-interpretable, Orbit API error):

- Do NOT set `Status` to `Done`. Leave it at whatever state it was in.
- Write a failure note in the `Outcome` column: `FAILED — [reason]. Will retry if re-approved.`
- Log the failure. Continue with remaining rows — don't abort the whole run for one row's failure.

### Step 9 — Build the end-of-day copy blocks

After the per-row loop completes (success and failure rows both processed), assemble the two end-of-day copy blocks and hand each to `writers/notion.md`. Both blocks are non-blocking — if either reports failure, surface it in the self-summary and continue.

#### Step 9a — Pod Daily Task block

Hand to `writers/notion.md` — Flow — appending the Pod Daily Task block.

Inputs the builder collects:
- Every row whose Outcome string records an Orbit task create or reassign (new `Task created → Orbit #...` or `Reassigned from X → Y`).
- For each qualifying row, pull from the executor return values: `client_name`, `project_name`, `task_title`, `task_id`, `orbit_task_url`, `assignee_first_name`, `assignee_full_name`.
- Preserve matcher order (the same order the rows appear in the Morning Queue database).

Skip silently if zero rows qualify or if `pod_daily_task_enabled = false`.

#### Step 9b — AM Daily Ping block

Hand to `writers/notion.md` — Flow — appending the AM Daily Ping block.

Inputs the builder collects:
- The same qualifying-row set (filtered by this section's own `include_reassignments` flag).
- Grouping: map each qualifying row to its AM via the Preferences Account Managers → Projects associations. Drop rows with no AM. Drop AMs listed in `quiet_ams`.
- For each AM group: the AM identity fields from Preferences, plus the bundle of row contexts (summaries, recommended actions, resolved assignees) needed for the writer's drafting call.

Skip silently if zero AMs qualify or if `am_ping_enabled = false`.

The writer handles per-AM body drafting via `writers/plain-language.md`. Mode 2 does not draft any text itself for this block — it only assembles inputs.

### Step 10 — Post-execution Slack summary to the PM

After the digest block is appended (or skipped), send a Slack DM to the PM summarizing:

```
Your morning queue executed.

✅ 7 of 9 actions taken:
  • Agency X homepage task created → Slack draft for Vijay appended in today's queue
  • DigitalFirst 3 action items created → Slack drafts for Ravi + Amit appended
  • GrowthLab blog reassigned to Ravi (per your note) → Slack draft appended
  • CloudBase QA handoff → Slack draft for Mannan appended
  • Reply drafted to Jane (Agency X, saved to Gmail drafts)
  • AM email draft for Caitlin saved to Gmail

⚠️ 1 held back:
  • TechCo budget alert — your note said "save as draft, check with Caitlin first". Draft saved in Gmail. No other action taken.

❌ 1 failed:
  • BrightPath proposal — Gmail MCP timed out. Retry by typing `PM Task Assignment, run execution now` or wait for tomorrow.

Open today's Notion queue page to copy the Slack drafts and send them to your team / AMs.

Pod Daily Task block ready at the bottom of today's page — copy and paste into the pod's daily task channel.
AM Ping Drafts block ready below it — copy each into a DM to the AM.
```

Closing-line variants — append independently based on each block's outcome. If both are skipped, omit both lines.

Pod Daily Task line variants:
- Success → `Pod Daily Task block ready at the bottom of today's page — copy and paste into the pod's daily task channel.`
- Layout parsed but has no `for_each_task:` loop → `Pod Daily Task block rendered but has no task loop — see preferences.`
- Render failed → `Pod Daily Task block failed to render — see run-log.`
- Zero qualifying rows OR digest disabled in preferences → omit.

AM Ping Drafts line variants:
- Success → `AM Ping Drafts block ready below it — copy each into a DM to the AM.`
- Layout parsed but has no `for_each_am:` loop → `AM Ping Drafts block rendered but has no AM loop — see preferences.`
- Render failed (block-level or any per-AM draft) → `AM Ping Drafts block had errors — see run-log.`
- Zero qualifying AMs OR AM ping disabled in preferences → omit.

Use `executors/slack.md` to send this DM.

### Step 11 — Update Preferences

Set `last_execution_run` timestamp in Preferences.

### Step 12 — Append run-log entry

Call `writers/run-log.md` with the execution summary:
- Timestamp range (start → end of this Mode 2 fire)
- Per-row execution outcome list: row ID, action type, result (`executed` / `skipped` / `failed` / `held`), and — if the row had a PM note — the PM-note interpretation produced by `synthesis/note-interpreter.md`
- Aggregate counts (executed / skipped / failed / held)
- Connector status (which MCPs were healthy, degraded, failed)
- Whether escalation was fired (no, in this branch — Step 3 handles that case before reaching here)
- Pod Daily Task block outcome: `appended` / `skipped_no_qualifying_rows` / `skipped_disabled` / `failed: <reason>`, plus the count of qualifying rows
- AM Daily Ping block outcome: `appended` / `skipped_no_qualifying_ams` / `skipped_disabled` / `failed: <reason>` / `partial: <N>_drafts_failed`, plus the count of AMs in the block and the count of per-AM draft failures (if any)

The writer creates a row in the Run Log database on the Notion parent and a linked decision-trace detail page. This trace is what the post-mortem and the next fire's preflight read.

### Step 13 — Exit

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

Manual fires follow the same deterministic path as routine fires: if Ready is OFF, Step 3 fires escalation and exits. No interactive confirmation, no waiting. The PM can flip the toggle and re-run the command — that next run will see Ready ON and proceed.
