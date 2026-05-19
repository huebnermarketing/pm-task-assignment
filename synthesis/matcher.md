> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Matcher

## Purpose

Takes every signal from all four collectors (Orbit, Gmail, Slack, Fathom) and turns them into a clean, ordered list of items the PM will see in today's Morning Queue.

## Identity matching is alias-aware

When the matcher classifies senders, recipients, or the running PM's identity, it matches the email or Slack handle against the canonical email AND any aliases stored in the PM's Preferences (under the `Email aliases` field).

Examples:

- A WLIQ team member with canonical `aditis@whitelabeliq.com` and alias `aditi@whitelabeliq.com`: signals from EITHER address are treated as the same person.
- The running PM's self-noise filter: messages FROM the canonical email OR any alias of the running PM are filtered out (they're the PM talking to themselves or replying).
- AM identification: an AM listed in Preferences under one address but who emails from an alias is still identified as the AM.

This applies across all sender classification, all routing decisions, and all assignment recommendations. Same identity = same person, regardless of which alias appears in any given signal.

If a signal involves an email address that doesn't match the canonical OR any alias of any known WLIQ identity, the sender is classified as `unknown` and the matcher proceeds with reduced confidence (often flagging `Uncertain:` in AI Notes).

## Output gating — two actionable paths only

The Morning Queue exists to drive **two AI actions and only two**:

1. **Reassign existing Orbit task to a developer** — applies to projects whose Orbit `project_type` is in the **reassign bucket**: `Ad-hoc`, `Maintenance` ONLY. Existing task already sits with a dev; signal indicates current owner needs to change.
2. **Create sub-task under the PM's own parent task on that project** — applies to ALL other project types (`Fixed Cost`, `SaaS`, `PPC`, `Hosting`, `Hourly`, `Repeat`, plus any new project type Orbit may add). Parent task **must be currently assigned to the running PM** (`assignee_id == PM_user_id`). Sub-task gets the full 6-section `schemas/orbit-dq-standard.md` body.

The "PM's own task on that project overnight" semantic is the load-bearing constraint for path 2: the PM holds a parent task for each active engagement (e.g., the engagement's umbrella "Discovery", "Reference", or named meeting/milestone task). Sub-tasks for that engagement land under the PM's parent so the PM remains the project's coordination anchor in Orbit. If no PM-owned task exists on a project that needs new work, the matcher cannot create the sub-task — see Job 5 "Picking the parent task" for the fallback.

Every signal that cannot reduce to one of these two actions is **dropped from the queue**. The PM scans Orbit directly for status reviews, FYI items, and AM coordination. Dropped signals are not lost — they are recorded in the Run Log detail page under a `Filtered signals` section so the PM can audit what was suppressed and why (see Job 11).

### Drop list (explicit — do NOT emit a queue row for any of these)

- Pure FYI rows (no action the PM personally must take inside Orbit on the assignee axis).
- "PM coordination" or "PM-managed wait state" rows (`recommended_assignee = —`).
- Rows where the next move belongs to an AM or the client (chase email, AM Slack ping, status nudge).
- Rollup rows aggregating multiple stale tasks for PM review.
- Status updates the PM would do themselves on Orbit (move-to-In-Review, due-date bump, close).
- Comment-only follow-ups the PM would post themselves.
- Email drafts to clients / AMs (PM writes those manually in Gmail).
- Hours-overrun alerts (`no-reply@whitelabeliq.com` notifications). Already handled by Orbit hours flow per PM feedback.
- Standup / Daily Status recap rows where no Orbit assignment changes.

If the matcher finds itself drafting a row that doesn't end in either `Reassign task #X to <name>` or `Create subtask under #X with brief`, the row is filtered and logged.

### Project-type lookup

`project_type` comes from the Orbit project metadata returned by `get_project_details` (already in the collector's relationship map). When `project_type` is missing or unresolved, log `project_type_unknown` in the row's filter trace and DROP the row (do not guess). The PM sees the dropped signal in the Run Log; the next morning's run will retry once project metadata is populated.

## What the matcher produces

A list of items. Each item becomes one row in the Morning Queue database. For each item, the matcher sets:

- `summary` — one-line plain-English summary the PM reads (in normal professional English, not simplified)
- `project` — the Orbit project name (best match)
- `recommended_action` — short phrase, e.g., "Create task + Slack Vijay"
- `recommended_assignee` — name + short reason
- `ai_notes` — anything unusual, uncertainty flags, split reasoning
- `source_signals` — the set of collector signals that contributed to this item
- `proposed_orbit_body` — 6-section task body (from `schemas/orbit-dq-standard.md`), in plain language (per `writers/plain-language.md`) — used by the Orbit Executor if approved
- `proposed_slack_handoff` — plain-language message for the assignee
- `proposed_email` — normal-English email draft if the action involves emailing an AM or client

## Jobs in order

### Job 1 — Group signals by project

Use the Orbit relationship map to connect each signal to a project.

For Orbit signals: project is already on the signal.

For Gmail signals:
- Check sender domain → map to a client → find that client's active projects for this PM
- Read subject and body for project-name keywords
- If multiple projects match, pick the most recently active one and flag `Uncertain:` with an AI Note

For Slack signals:
- Check channel name for project-name or client-name match
- Check sender against AM list → map to their assigned projects
- For DMs, scan message content for project keywords
- If no match, leave `project = null` and flag as `Uncertain:`

For Fathom signals:
- Check meeting title and attendee list
- Match external attendees to client contacts
- Match meeting title against client/project names
- If a meeting covered multiple projects, split into multiple items (one per project)

### Job 2 — Deduplicate across sources

A single issue may show up in Gmail, Slack, and Orbit. Matcher merges them into one item.

Match signals as the same item when:
- Same project AND same topic/deliverable (keyword overlap in content)
- Same client contact sent the email AND AM discussed it in Slack within the same \~24-hour window
- Fathom meeting's action item overlaps with a subsequent email or Slack ask

When merging, preserve all source signals in `source_signals`. The row's detail page will cite each source separately.

### Job 3 — Uncertainty handling — NO probable-match grouping

If the matcher isn't confident two signals are about the same thing, DO NOT group them. List them as separate items.

For each item where the matcher has any uncertainty, add a line to AI Notes that starts with `Uncertain:` and explains what's uncertain. Examples:

- `Uncertain: I don't know if this relates to Agency X's homepage task or a new request.`
- `Uncertain: Sender's domain doesn't match any Orbit client. Guessed project based on subject keywords.`
- `Uncertain: Meeting covered multiple topics; I split this into three items but they may overlap.`

The PM resolves uncertainty by reviewing and either splitting further, writing a PM Note, or leaving it at Recommended Action.

### Job 4 — Generate the one-line summary (verb-first, locked vocabulary)

Normal professional English. This is what the PM reads when scanning the queue. The summary is **action-led** — the first token is a locked verb, the rest is the smallest context clause needed to identify the work.

#### Locked verb list (exactly 2 verbs — no others permitted)

| Verb | Used when |
|---|---|
| `Reassign` | Project type = Ad-hoc OR Maintenance. Existing dev-owned Orbit task is changing hands. |
| `Create subtask` | Project type = everything else (Fixed Cost / SaaS / PPC / Hosting / Hourly / Repeat / any other). Net-new scoped work landing under the PM's own parent task on the project. |

These are the ONLY two starting verbs a queue row's Summary may use. If the matcher cannot frame the row with one of these two, the row was misclassified — re-run the Output gating filter and drop it.

#### Summary patterns

Pattern A — Reassign:
```
Reassign <project short name> #<task_id> — <one-clause why>. To <role>.
```

Pattern B — Create subtask:
```
Create subtask under <project short name> #<parent_task_id> — <one-clause what>. To <role>.
```

Examples:
- `Reassign Brother Plesk #109958 — patch ETA needs WP Maintenance ownership. To WP dev.`
- `Reassign Process Barron Hours #15949 — backend overrun handoff. To BE dev.`
- `Create subtask under Solstice WP #8393 — swap Contact Form brochure PDF. To WP dev.`
- `Create subtask under BoyarMiller Practice Area #103623 — draft Real Estate page outline. To Content.`

Rules:
- First token is one of the two locked verbs. No other openers.
- Project short name = client-readable phrase, max 4 words. Strip the long Orbit project title.
- Task ID always present (the row is anchored to a specific Orbit task).
- Why / what clause = 6–10 words, identifying the work, not narrating context.
- Role at the end = the role-hint label the matcher resolved in Job 6.
- No emojis. No leading client name without verb. No narrative.
- Max 120 chars total (Notion title field stays scannable across the column).

### Job 5 — Recommend the action (2 actions only)

The action set is closed and matches the Output gating section above. Pick exactly one:

- **Reassign** — existing Orbit task moves from current assignee to a new one. Reassign bucket only (`project_type` ∈ {`Ad-hoc`, `Maintenance`}). Output: target `task_id` (existing) + new `assignee_id`. No new task is created. No 6-section body needed (a short Orbit comment captures the reason, drafted in Job 7).
- **Create subtask with brief** — new sub-task is created in Orbit under the **PM-owned parent task** on the project. Applies to every project type NOT in the reassign bucket. Output: `parent_task_id` (which must be a task with `assignee_id == PM_user_id`), new sub-task title, full 6-section body per `schemas/orbit-dq-standard.md` written in plain language, `assignee_id` for the sub-task.

If neither action fits, the signal does not become a row. Apply the Output gating filter and route to the `Filtered signals` log (Job 11).

#### Picking the existing task to reassign (Reassign path only)

For ad-hoc / maintenance / hosting / hourly / repeat projects, the matcher must select WHICH existing Orbit task the signal targets. Procedure:

1. Pull all open tasks on the project from the relationship map.
2. Score each task by signal-to-task match: subject keyword overlap with task title, file/feature reference, mentioned task ID in the signal body.
3. If exactly one task scores meaningfully higher than the rest, pick it. Row Summary cites that task ID.
4. If two or more tasks tie on score, set `recommended_assignee = null` and write AI Notes: `Uncertain: signal could apply to task #X (<title>) or task #Y (<title>). Please pick which to reassign.` Row is still emitted (one of the two valid actions) but assignee axis flagged.
5. If no open task on the project meaningfully matches the signal, DROP the row (this is not net-new for an ad-hoc project — most likely the signal is operational chatter the dev already sees in Orbit). Log to Filtered signals.

#### Picking the parent task for sub-task creation (Create subtask path only)

The parent **must be a task currently assigned to the running PM** on the project. Procedure:

1. From the relationship map, list every open task on the project where `assignee_id == PM_user_id`. Call this the PM-owned parent pool for the project.
2. **If the PM-owned parent pool is empty** — the sub-task cannot be created under this rule. Drop the row and log to `Filtered signals` with `filter_reason: no_pm_owned_parent_task`. The PM sees the dropped signal in the Run Log and can seed a parent task themselves (the next morning's run will then surface it).
3. **If the PM-owned parent pool has exactly one task** — that's the parent. Use it.
4. **If the PM-owned parent pool has multiple tasks** — match the signal to a parent by phase / feature / deliverable keyword overlap with each parent's title. Recency breaks ties (most recently updated parent wins). If two parents tie cleanly on score, set `parent_task_id = null` and write AI Notes: `Uncertain: sub-task could go under PM-owned parent #X (<title>) or #Y (<title>). Please pick.` Row is still emitted; parent axis flagged.

Sub-task `assignee_id` (different from `parent_task_id`) is picked via the Job 6 decision tree on the developer pool — same rules as before. The PM owns the parent; the dev owns the child.

### Job 6 — Recommend the assignee

Call `synthesis/pod-inference.md` with the item's project ID. It returns the candidate pool — matrix members ∪ Orbit followers/recent-assignees — with role hints, familiarity scores, `has_history_on_project` booleans, matrix membership flags, plus the `floater_pool` and `functional_pools` for fallback paths.

Pick the single best assignee using a 4-branch decision tree. Familiarity wins by default; availability is checked only on the no-history fallback path (per `SKILL.md` non-negotiable rule #6).

#### Role-fit heuristics

Filter the candidate list by role first, using the task's domain:

- FE / HTML / front-end / homepage → role `FE` (matrix label `HTML`)
- WordPress / WP / PHP / theme / plugin → role `WP` (matrix label `WordPress / PHP`)
- BE / API / database / server-side → role `BE` (no matrix label — Orbit-history only)
- QA / testing / regression → role `QA`
- Design / mockup / Figma → role `Design`
- Content / copy / blog → role `Content`
- Business analysis / requirements / scoping → role `BA`
- Project coordination / AM-liaison → suggest the PM themselves (existing behavior)

Apply matrix `role_hint` first (strongest signal); fall back to Orbit `department` and task history when the candidate has `source: orbit-history` (no matrix hint).

#### Decision tree

1. **Branch (a) — History wins.** If at least one role-fit candidate has `has_history_on_project = true`:
   - Pick the candidate with highest `familiarity_score`.
   - **No `compute_availability` call.** Familiarity is the deciding signal.
   - Reason format: `<role> on <project>, <N> tasks last 3mo<, active now if applicable>`.

2. **Branch (b) — Matrix availability fallback.** Else if the PM's matrix (`source: matrix` or `source: both`) has at least one role-fit candidate (none with project history):
   - Call `pod-inference.compute_availability` on those candidates.
   - Pick the candidate with highest `availability_score`.
   - Reason format: `<role> in <PM matrix>, no prior task on <project>, lightest workload (<N> open tasks)`.
   - If all availability scores come back null (workload check failed for everyone), pick the highest `familiarity_score` and append ` — availability unknown` to the reason.

3. **Branch (c) — Floater fallback.** Else if `floater_pool` has at least one role-fit candidate:
   - Call `compute_availability` on the floater role-fit candidates.
   - Pick the highest `availability_score`.
   - Reason format: `Floater <role>, no <role> in <PM matrix>, lightest workload (<N> open tasks)`.

4. **Branch (d) — Cross-matrix Uncertain.** Else:
   - Set `recommended_assignee = null`.
   - In AI Notes: `Uncertain: No <role> in <PM matrix> or Floaters. Cross-matrix candidates: <comma-separated list of names + matrix from functional_pools>. Please pick.`
   - Example: `Uncertain: No Design in Matrix A or Floaters. Cross-matrix candidates: Jay Panchal (Design Matrix), Vijay Jadav (Design Matrix), Rinkal (Design Matrix). Please pick.`

#### Matrix-unavailable degradation

When `pod-inference` returns `matrix_available: false` (URL not injected, fetch failed, or parse failed), only branches (a) and a degraded (d) apply:

- Branch (a) still works — Orbit history is sufficient.
- Branches (b) and (c) cannot fire (no matrix pool to draw from). Skip them.
- Branch (d) becomes: `recommended_assignee = null`, AI Notes `Uncertain: No prior task on <project> and Pod Matrix unavailable. Please pick the assignee.`

#### Assignee write format

Write the assignee as `Name (Role) — short reason`. Examples:

- Branch (a): `Vijay Patel (FE) — primary FE on Agency X, 24 hrs on current homepage task`
- Branch (b): `Atul (WP) — WordPress / PHP in Matrix A, no prior task on BrightPath, lightest workload (3 open tasks)`
- Branch (c): `Vijay Salvi (FE) — Floater HTML, no HTML in Matrix A, lightest workload (2 open tasks)`
- Branch (d): `null` with AI Notes Uncertain message above.

If the PM knows better and wants to assign outside the pod, their note will override (PM note always wins — existing rule).

### Job 7 — Generate the proposed Orbit body (action-specific)

The body content depends on which of the two paths the row took:

**Create subtask path (project bucket)** — pre-write the full 6-section task body per `schemas/orbit-dq-standard.md`. Plain language (4th–5th grade English) per `writers/plain-language.md` since the delivery team reads it. Keep role-specific technical terms. Strip corporate English. This is what lands in Orbit as the sub-task description when Mode 2 fires.

**Reassign path (ad-hoc bucket)** — pre-write a short **Orbit reassignment comment** (NOT a full 6-section body). The comment goes on the existing task at reassignment time and tells the new assignee what changed. Format:

```
Reassigning to <new assignee first name>. Reason: <one-clause why the handover>. <Optional single sentence of context the assignee won't already see on the task>. Please continue from where the previous assignee left off and log your hours.
```

Plain language. Max 3 sentences. Cite a source link only when the context line references something the assignee can't see in Orbit (e.g., a Fathom moment or a client email thread).

A reassignment row does NOT carry a `proposed_orbit_body` field. It carries a `proposed_orbit_reassign_comment` field instead. Mode 2 executor reads whichever field is present based on the row's action.

### Job 8 — Generate the proposed Slack handoff

The Slack handoff message is the team-Slack DM the PM copies-and-sends after Mode 2 fires (per `executors/slack.md`). Generate it for BOTH paths:

- **Reassign path** — tell the new assignee what's moving to them, why, and the task URL.
- **Create subtask path** — tell the assignee a new sub-task landed, why, the parent context, and the task URL.

Plain language per `writers/plain-language.md`. Reminder to log hours if Preferences has that always-include rule.

NO Slack handoff is generated for AMs or clients — those are PM-handled outside the queue per the Output gating filter.

### Job 9 — Generate the proposed email

**Skipped.** Email drafting is outside the queue's scope under the 2-action gating rule. Client and AM emails are PM-handled in Gmail directly. No row produces an email artifact.

If a signal's primary value was an email reply, the matcher Filtered-signals-logs it and moves on.

### Job 10 — Write AI Notes

Include only things worth the PM knowing:
- `Uncertain:` flags (always)
- Why a PM note was honored differently than the recommendation
- When a single signal was split into multiple items
- When the recommended assignee isn't obvious
- When a collector failed and this item has partial context

Leave empty if nothing notable.

### Job 11 — Emit the Filtered signals trace

For every signal (or grouped signal-set) the Output gating filter dropped, append an entry to a `filtered_signals` array on the matcher's return payload. Each entry:

```
{
  "source": <orbit | gmail | slack | fathom>,
  "summary": <one-line description of what was dropped>,
  "filter_reason": <one of: pm_coordination | wait_state | rollup | status_update_self_drive | comment_only_self_drive | client_email_pm_owns | am_ping_pm_owns | hours_overrun_alert | standup_recap | project_type_unknown | no_matching_open_task | no_pm_owned_parent_task>,
  "citations": [<source links — gmail thread URL, orbit task URL, fathom recording URL>]
}
```

`writers/run-log.md` consumes this array and writes a `Filtered signals (N)` collapsible section in the Run Log detail page. Standard Notion toggle, closed by default. The PM opens it only when they want to audit what was suppressed.

This is the safety valve: the queue stays tight, but nothing the collector saw is silently lost.

## What the matcher does NOT do

- Does not check availability or capacity proactively. Availability is checked only on the no-history fallback path (Job 6 branches b/c) per `SKILL.md` non-negotiable rule #6.
- Does not estimate hours.
- Does not pick a due date — the PM decides via the Recommended Action execution or via an explicit note.
- Does not group probable matches. Uncertain = separate items.
- Does not execute anything — its output is consumed by `writers/notion.md` to write the dated page.

## Ordering the items on the page

Sort by a simple heuristic:
1. Items with high-urgency signals first (overdue tasks, blockers, urgent language like "urgent", "blocking", "today")
2. Items from external clients ahead of internal
3. Items involving multiple sources (cross-system signals) ahead of single-source
4. Everything else by recency (newest first)

No scoring model, no scoring columns. Just a sensible order so the PM reads important things first.

## Output format

An array of item records ready for `writers/notion.md` to consume. Each item has everything listed in "What the matcher produces" at the top of this file, plus an ordered position in the array.
