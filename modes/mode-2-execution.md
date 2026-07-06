> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes routine triggers — preflight runs even when invoked by a scheduled cloud routine.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Notion. Enrichment-on-demand — Fathom (lazy fetch via `collectors/fathom.md`). Slack is **outbound-send only** (no collection, no reads) and only invoked from the 2 documented send paths: team-handoff send and AM-ping send, each gated by an explicit PM `send` note on the row + audience match. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Mode 2 — Execution Run

## When this runs

- Scheduled: daily at the PM's configured execution time (default 10:45 AM IST), fired by a Claude Routine.
- Manual: `PM Task Assignment, run execution now`.

## What Mode 2 does

Reads today's dated page in Notion. Checks the page-level `Ready for Execution` toggle at the top. If ON, executes every approved row and every row with a PM note. Writes outcomes back. Writes a completion-summary callout block at the top of today's dated page (replaces the old Slack DM — the PM is already opening Notion). If OFF, fires the inline escalation flow (Step 3a) and exits — routines cannot wait for the PM to flip a toggle.

Mode 2 is identical across the two execution surfaces (headless vs interactive — see `ENVIRONMENTS.md`). The only difference: in interactive mode, when a PM Note is genuinely ambiguous and `synthesis/note-interpreter.md` returns `confidence = low`, the skill may ask the PM one clarifying question before falling back to `HELD`. In headless mode it always falls back to `HELD`.

## End-to-end flow

### Step 1 — Read Preferences

Identity, AM list, communication defaults, always-include rules, tone samples. All used downstream.

### Step 2 — Locate today's dated page

Search the PM's Notion parent for a sub-page titled with today's date in "DD Month YYYY" format. If it doesn't exist, either Mode 1 never ran today or the PM deleted it. Abort with a sent email to the PM (operational exception per `executors/email.md`): subject `[PM Task Assignment] Today's morning queue is missing`, body `I don't see today's morning queue. Did Mode 1 run? Try the manual command 'PM Task Assignment, run morning'.`

### Step 3 — Check the Ready for Execution toggle

The toggle is a checkbox block at the top of the dated page. Fetch the page content and find the to-do block labeled `Ready for Execution`.

- **If unchecked (OFF):** fire the inline escalation flow (Step 3a below), append a run-log entry via `writers/run-log.md` recording outcome `escalated` (with reason `ready_toggle_off`), and exit. Do NOT prompt the PM to flip the toggle. Do NOT wait. Routines cannot block on human input — if the PM never approved by execution time, the backup owns it.
- **If checked (ON):** proceed to Step 4.

### Step 3a — Escalation flow (only when Ready=OFF)

Replaces the former separate `mode-3-escalation.md` file. Runs inline in Mode 2.

#### 3a.1 — Pull Preferences fields

- PM identity (for the backup message)
- Escalation backup: name, email (the only channel — Slack option was removed), ping time
- Today's dated page URL (the page just fetched in Step 2)

#### 3a.2 — Edge cases (handle before composing)

- **No backup configured.** Email the PM (sent, operational exception): subject `[PM Task Assignment] Escalation backup not configured`, body `I noticed the Ready toggle isn't on and there's no escalation backup configured. Today's queue was not executed. Add a backup via 'PM Task Assignment, change my preference: add [name] as my escalation backup'.` Skip 3a.3–3a.5 and exit.
- **Backup is the PM themselves.** Email the PM: subject `[PM Task Assignment] Escalation backup misconfigured`, body `Your escalation backup is set to yourself. That won't work. Set someone else via 'PM Task Assignment, change my preference: change my escalation backup to [name]'.` Skip 3a.3–3a.5 and exit.
- **Backup email is invalid or missing.** Email the PM that escalation failed and why; exit.

#### 3a.3 — Compose the escalation message

Email-only (the only channel):

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

Use `executors/email.md` to send the escalation email directly to the backup. (Operational escalation is one of the two documented send-not-draft exceptions in `executors/email.md`.)

#### 3a.5 — Log + notify the PM

- Add a callout block at the top of today's dated page: `⚠️ Escalation fired at [time] IST. [Backup name] notified via email at [backup_email].`
- Send the PM an operational email (second send-not-draft exception): subject `[PM Task Assignment] Today's queue was escalated`, body `I escalated today's morning queue to [backup name] at [time] IST because the Ready for Execution toggle was still off. If this was unintentional, flip the toggle now and type 'PM Task Assignment, run execution now' to proceed.`
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
| Outcome starts with `FAILED — deep-read incomplete` (any Status) | Skip execution even if Status = `Approved`. The row's body was not composed from the full task + comment history (Job 7 per-row deep-read failed in Mode 1); executing it would write incomplete content to Orbit. Replace Outcome with `Skipped Mode 2 execution — Mode 1 deep-read failed; manual review needed. Re-fire Mode 1 with retry to refresh this row.` Log: `mode2_skip_failed_deep_read`. |

### Step 6 — Execute per row

For each row that needs action:

1. Identify row type from the **`Recommended Action` column's leading verb** (NOT the Summary text, which is now topic-style per `synthesis/matcher.md` Job 4 rollback). The verb is one of: `Create subtask`, `Reopen subtask`, `Hand off parent task`, `Flag`, `Create parent task`.
2. **Flag rows do not execute — UNLESS they carry an actionable PM note.** A Flag row with no PM note (or a note that resolves to skip) is skipped: Status unchanged, PM marks `Skip. No Action Needed` once resolved externally. Log: `Flag row — no Mode 2 action.` **Before skipping, check for a PM note** and route it to the note-interpreter (step 7) first — THREE documented promotions can turn a Flag into a write: (a) a **Possible-Orbit-miss** Flag with a `create it on… / create anyway` note → `Create parent task`; (b) a **`pm_task_due_today` bare Flag** with a delegation note (`delegate to… / hand to <pod> / give to <dev>`) → `Create subtask` / `Reopen subtask` / `Hand off parent task` (see `synthesis/note-interpreter.md` § Delegate a due-today Flag); (c) a **`fc_row_type: stale_work_ping`** Flag with a `delegate to <dev>` note → `Create subtask` / `Reopen subtask` / `Hand off parent task`, promoting exactly like (b). These are the ONLY verb-changing promotions. If the interpreter returns `confidence: low`, the row is HELD with its clarification (not skipped). Any other note on a Flag (or any note on a non-promotable Flag class) → the row stays a Flag, note recorded in AI Notes, no execution.

   **Fixed-cost carve-out (not a verb change).** A note on a Flag row carrying `fc_row_type: follow_up_reminder` that **resolves** (`resolved` / `client sent it` / `no longer needed`) or **snoozes** (`snooze <period>`) IS executed — the row stays a `Flag` (no verb change), but the note-interpreter emits a `fc_ledger_update` action_plan that Mode 2 routes to `writers/notion.md` § Flow — Mode 2 ledger touch (see Step 6 executor dispatch below). This is a non-verb-changing ledger touch, distinct from the three verb-changing promotions above.
3. **`Create subtask` rows** — load the row's detail page content (Action Block callout at top, Task Brief, Sources, Recommended Action reasoning, proposed Orbit task body, proposed Handoff).
4. **`Reopen subtask` rows** — load the row's detail page content (Action Block, Task Brief, Sources, Recommended Action, Proposed New-Work Comment, proposed Handoff). Route to `executors/orbit.md` § Reopen an existing subtask. The 6-section body is NOT used here (the existing subtask already has one); the comment + reassign + status flip is the entire write.
5. **`Hand off parent task` rows** — load the row's detail page content (Action Block, Task Brief, Sources, Recommended Action reasoning, proposed Handoff). Route to `executors/orbit.md` § Hand off parent task. The 6-section body is NOT used (the existing parent already has one); only the reassign + audit comment fires.
6. **`Create parent task` rows** — load the row's detail page content (proposed Orbit task body for the parent, sources). These rows do NOT have a proposed Handoff (no delegate; parent is PM-assigned) and do NOT participate in the Pod Daily Task or AM Daily Ping blocks (Step 9). Route to `executors/orbit.md` § Create a parent task. PM-note overrides apply the same way (severity, due-date, follower add) via post-create `update_task` / `change_task_due_date` calls.
7. If there's a PM note, feed it to `synthesis/note-interpreter.md` to resolve the intent and override parts of the recommendation. Notes may change the assignee, the parent task, severity, or due date. Notes generally **cannot change the row verb** — with exactly THREE documented promotion exceptions: a **Possible-Orbit-miss** Flag → `Create parent task` (`create it on…`); a **`pm_task_due_today` bare Flag** → `Create subtask` / `Reopen subtask` / `Hand off parent task` (`delegate to… / hand to <pod>`; see `synthesis/note-interpreter.md` § Delegate a due-today Flag); and a **`fc_row_type: stale_work_ping`** Flag → `Create subtask` / `Reopen subtask` / `Hand off parent task` (`delegate to <dev>`), which promotes exactly like the due-today delegate. No other verb change is permitted (e.g. a `Create parent task` row cannot be re-cast as `Create subtask` — the PM deletes it and lets it re-emerge with corrected matcher input next morning). **On a promotion, the row is reclassified to its new verb for ALL downstream steps** — executor dispatch (step 8), handoff-draft generation, the Pod Daily Task block (step 9), and the summary counts (step 10) treat it as the new verb, and the row's `Recommended Action` + Outcome are updated to reflect what was actually executed.
8. Route to the appropriate executors:
   - **Orbit** — `executors/orbit.md`. Dispatch by verb:
     - `Create subtask` → Create-subtask path: `parent_id = parent_task_id`, `assignee_id = recommended_assignee`, `description = proposed_orbit_body`. PM-note overrides (severity, due date, follower add) are applied as additional `update_task` or `change_task_due_date` calls AFTER the subtask is created. Priority-lane rows pass `parent_id = signal.parent_task_id` from the matcher (the AM-assigned parent) — the sub-task nests under that parent.
     - `Reopen subtask` → Reopen-subtask path: `update_task(existing_subtask_id, assignee=last_dev_user_id, task_status_id=24)` + `add_task_comment(existing_subtask_id, new_work_description)`. See `executors/orbit.md` § Reopen an existing subtask for guards (inactive-dev fallback, subtask-closed-since-Mode-1 rollback to standard Create-subtask).
     - `Hand off parent task` → Hand-off path: `update_task(parent_task_id, assignee=recommended_assignee_user_id, add_followers=[PM_user_id])` + `add_task_comment(parent_task_id, handoff audit comment)`. See `executors/orbit.md` § Hand off parent task.
     - `Create parent task` → Create-parent-task path: `project_id = row.project_id`, `assignee_id = PM_user_id`, `parent_id = null`, `description = proposed_orbit_body`. Optional `due_date` derived from urgency tokens per `executors/orbit.md` § Create a parent task. After creation, add `Orbit` to the row's Source Systems multi-select (the row was Gmail-origin).
   - **Notion (default handoff path)** — every `Create subtask`, `Reopen subtask`, and `Hand off parent task` row produces a Handoff draft block (per `writers/notion.md` Flow — updating rows after Mode 2) appended under the row's Outcome. PM copies + sends it themselves. This is the default and applies to every team handoff and every AM ping unless the PM explicitly opts into the Slack send path. **`Create parent task` rows skip the handoff block entirely** (no delegate). **Un-promoted `Flag` rows skip too** (no executor fired) — but a `pm_task_due_today` Flag promoted to `Create subtask` / `Hand off parent task` via PM note is, by step 7, reclassified to that verb and DOES get a handoff draft like any other delegated row.
   - **Slack (outbound-send only, narrow exception)** — `executors/slack.md` sends the handoff/ping via Slack only when the PM left an explicit `send` note on the row AND the audience matches a documented send path: (a) `audience = team` → team-handoff send to the delegate's Slack handle (resolved via the Slack handles reference block on the dated page — see `writers/notion.md`), (b) `audience = am` → AM-ping send to the AM's Slack handle (resolved via Preferences `AM Slack handle` field, also rendered on the dated page reference block). Without an explicit `send` note, the handoff stays as a Notion draft for the PM to copy. `Create parent task` and `Flag` rows are NEVER eligible for Slack send (no handoff exists).
   - **Notion (fixed-cost ledger touch)** — a `fc_row_type: follow_up_reminder` Flag with a resolve/snooze note produces an `fc_ledger_update` action_plan (see `synthesis/note-interpreter.md`). Route it to `writers/notion.md` § Flow — Mode 2 ledger touch: it writes the Client-Ask Ledger row on the Fixed-Cost Registry sub-page (`Status` → `Resolved-manual`, `Resolved by` = PM note text, `Resolved on` = today for a resolve; on snooze, bumps the ask's `fc_state` `last_reminder` and mirrors `Last reminder`). This is Mode 2's ONLY registry write. Queue row `Outcome` → `Done — ask closed by PM` (resolve) or `Done — snoozed` (snooze). Not a verb change — the row stays `Flag`.
7. Before calling any executor, pass every string the assignee (delivery team) will read through `writers/plain-language.md` to enforce 4th–5th grade English.
8. Pass every source reference through `writers/source-citation.md` to ensure proper citation format.

Note: Email drafting paths (`executors/email.md`) are no longer triggered by queue rows — client and AM emails are PM-handled under the new gating rule. The email executor remains for operational sends (escalation backup ping, connector-failure tier-1 ping to PM) only — see `executors/email.md` for the 2 documented send exceptions.

### Step 7 — Update the row after execution

For each row, after all its actions execute:

1. Flip `Status` to `Done`.
2. Write the `Outcome` column with a short line describing what was done. Format: concise, specific, include Orbit links where applicable.
   - Examples (default Notion draft path):
     - `Subtask #110890 created under parent #110464. Handoff draft for Hitesh appended below.`
     - `Subtask #110918 created under parent #109958. Handoff draft for Atul appended below.`
     - `Subtask created → Orbit #110523. Severity bumped to Important per your note. Handoff draft appended below.`
   - Examples (Create parent task path):
     - `Created parent task #111002 on Agency X → Orbit [link]. Assigned to you. You can spawn sub-tasks under this in future runs.`
     - `Created parent task #111034 on BrightPath → Orbit [link]. Assigned to you. Due 2026-05-27 per signal urgency tokens. You can spawn sub-tasks under this in future runs.`
   - Examples (Slack send path, PM left explicit `send` note):
     - `Subtask #110890 created under parent #110464. Sent handoff to Hitesh on Slack (@hitesh).`
     - `Subtask #110918 created under parent #109958. Sent handoff to Atul on Slack (@atul) per your 'send' note.`
   - Failure:
     - `FAILED — Orbit subtask create returned 409 duplicate. Will retry if re-approved.`
     - `Subtask created. Slack send FAILED (@hitesh handle not found); draft appended for you to copy.`
3. For the default Notion-draft path, the writer appends the handoff body under the row's detail page (per `writers/notion.md` Flow — updating rows after Mode 2). The PM copies it from Notion and delivers it through whatever channel they use. For the Slack-send path, the executor sends the message directly and the Notion draft is omitted (the Outcome line records the send).

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
- Every row whose Outcome string records an Orbit subtask create (`Subtask #... created under parent #...`).
- For each qualifying row, pull from the executor return values: `client_name`, `project_name`, `task_title`, `task_id`, `orbit_task_url`, `assignee_first_name`, `assignee_full_name`.
- Preserve matcher order (the same order the rows appear in the Morning Queue database).
- Un-promoted Flag rows are NEVER included in the Pod Daily Task block (no Orbit task was created). **A `pm_task_due_today` Flag promoted to `Create subtask` via PM note (step 7) DID create a subtask — it qualifies via the Outcome-string criterion above, exactly like any other subtask-create row.**
- **Create parent task rows are NEVER included** either — the parent is PM-assigned, not a team handoff. Future sub-tasks under this parent will surface in subsequent days' Pod Daily Task blocks via the standard Create-subtask path.

Skip silently if zero rows qualify or if `pod_daily_task_enabled = false`.

#### Step 9b — AM Daily Ping block

Hand to `writers/notion.md` — Flow — appending the AM Daily Ping block.

Inputs the builder collects:
- The same qualifying-row set as Step 9a (subtask-create rows only — including a `pm_task_due_today` Flag promoted to `Create subtask` via PM note; un-promoted Flag rows AND Create parent task rows both excluded).
- Grouping: map each qualifying row to its AM via the Preferences Account Managers → Projects associations. Drop rows with no AM. Drop AMs listed in `quiet_ams`.
- For each AM group: the AM identity fields from Preferences, plus the bundle of row contexts (summaries, recommended actions, resolved assignees) needed for the writer's drafting call.

Skip silently if zero AMs qualify or if `am_ping_enabled = false`.

The writer handles per-AM body drafting via `writers/plain-language.md`. Mode 2 does not draft any text itself for this block — it only assembles inputs.

### Step 10 — Post-execution summary callout on today's dated page

After the digest blocks are appended (or skipped), write a single Notion callout block at the very TOP of today's dated page summarizing the Mode 2 run. This replaces the old Slack DM to the PM — the PM is already opening Notion to review approved rows, so the summary lives where their eyes are.

Hand the inputs to `writers/notion.md` § Flow — Mode 2 PM self-summary callout. Body template:

```
✅ Mode 2 ran at [HH:MM IST]. [N] approved, [K] executed, [F] failed, [S] skipped.
Priority lane: [N_priority] sub-tasks created under AM-handed parents (if any).

Executed:
  • Solstice WP brochure swap → Subtask #110890 under parent #106447. Handoff draft for Atul appended.
  • Brother Plesk patch ETA → Subtask #110918 under parent #109958. Sent handoff to Atul on Slack per your 'send' note.
  • ECP AI Visibility audit → Subtask #110945 under parent #110464. Handoff draft for Hitesh appended.

Parent tasks created (Possible Orbit miss):
  • Agency X contact form → Parent #111002 created on Agency X. Assigned to you. You can spawn sub-tasks under this in future runs.

Flagged (no auto-action):
  • 2010 Solutions — Ellen needs dev names for 27 May call. PM reply pending.

Failures:
  • BrightPath proposal subtask — Orbit create_subtask returned 409 duplicate.

Pod Daily Task block ready at the bottom of today's page — copy and paste into your pod's daily task channel.
AM Ping Drafts block ready below it — copy each (or, when you leave a 'send' note + audience=am, the skill auto-sends to the AM's Slack handle).
```

Closing-line variants — append independently based on each block's outcome. If both are skipped, omit both lines.

Pod Daily Task line variants:
- Success → `Pod Daily Task block ready at the bottom of today's page — copy and paste into your pod's daily task channel.`
- Layout parsed but has no `for_each_task:` loop → `Pod Daily Task block rendered but has no task loop — see preferences.`
- Render failed → `Pod Daily Task block failed to render — see run-log.`
- Zero qualifying rows OR digest disabled in preferences → omit.

AM Ping Drafts line variants:
- Success → `AM Ping Drafts block ready below it — copy each (or auto-send via the 'send' note + audience=am).`
- Layout parsed but has no `for_each_am:` loop → `AM Ping Drafts block rendered but has no AM loop — see preferences.`
- Render failed (block-level or any per-AM draft) → `AM Ping Drafts block had errors — see run-log.`
- Zero qualifying AMs OR AM ping disabled in preferences → omit.

Idempotency: on a same-day Mode 2 rerun, replace any existing summary callout in place (find the first block on the page that opens with `Mode 2 ran at`). Do not stack callouts.

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

If there's no dated page for today, exit cleanly with an operational email to the PM (subject `[PM Task Assignment] No morning queue today`, body `No morning queue found for today. Mode 1 either didn't run or was skipped. Run 'PM Task Assignment, run morning' to generate one.`).

## Special case — manual fire of Mode 2 when Ready is OFF

Manual fires follow the same deterministic path as routine fires: if Ready is OFF, Step 3 fires escalation and exits. No interactive confirmation, no waiting. The PM can flip the toggle and re-run the command — that next run will see Ready ON and proceed.
