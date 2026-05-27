> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist:** Primary collection — Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand — Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand — Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever. The allowlist is enforced even under experimental scope or forced runs.

# Matcher

## Purpose

Takes every signal from the two primary collectors (Orbit, Gmail) — including the Orbit collector's priority-pass output (`signal_type: am_handed_to_pm_overnight_due_today`) — and turns them into a clean, ordered list of items the PM will see in today's Morning Queue. During Job 4b, the matcher additionally invokes the Fathom enrichment service (`collectors/fathom.md`) on-demand to attach meeting context to any primary signal that references a meeting. Fathom never originates a row.

## Identity matching is alias-aware

When the matcher classifies senders, recipients, or the running PM's identity, it matches the email address against the canonical email AND any aliases stored in the PM's Preferences (under the `Email aliases` field).

Examples:

- A WLIQ team member with canonical `aditis@whitelabeliq.com` and alias `aditi@whitelabeliq.com`: signals from EITHER address are treated as the same person.
- The running PM's self-noise filter: messages FROM the canonical email OR any alias of the running PM are filtered out (they're the PM talking to themselves or replying).
- AM identification: an AM listed in Preferences under one address but who emails from an alias is still identified as the AM.

This applies across all sender classification, all routing decisions, and all assignment recommendations. Same identity = same person, regardless of which alias appears in any given signal.

If a signal involves an email address that doesn't match the canonical OR any alias of any known WLIQ identity, the sender is classified as `unknown` and the matcher proceeds with reduced confidence (often flagging `Uncertain:` in AI Notes).

## Output gating — two actionable paths only

The Morning Queue exists to drive **two AI actions and only two**:

1. **Create sub-task under the PM's own parent task on that project** — universal across every project type (`Fixed Cost`, `SaaS`, `PPC`, `Hosting`, `Hourly`, `Repeat`, `Ad-hoc`, `Maintenance`, plus any new project type Orbit may add). Parent task **must be currently assigned to the running PM** (`assignee_id == PM_user_id`). Sub-task carries an explicit `task_title` AND the full 6-section `schemas/orbit-dq-standard.md` brief. There is no project-type bucket split — every actionable signal becomes a sub-task. (Previously Ad-hoc/Maintenance used a separate "Reassign" path; that path is retired — Ad-hoc/Maintenance signals now also become sub-tasks under the PM's parent so the team handover follows the same shape every time.)
2. **Flag overnight signal for PM attention** — for signals the PM needs to see and act on personally, where AI cannot create a sub-task (no clear dev work to delegate, or the next move is PM-owned: email a client, decide a scope question, reply to an AM). Flag rows carry summary + sources + suggested PM-next-step. They do NOT execute in Mode 2 — they sit as a PM-readable item that the PM resolves manually (then marks `Skip. No Action Needed`).

The "PM's own task on that project overnight" semantic is the load-bearing constraint for path 1: the PM holds a parent task for each active engagement (e.g., the engagement's umbrella "Discovery", "Reference", or named meeting/milestone task). Sub-tasks for that engagement land under the PM's parent so the PM remains the project's coordination anchor in Orbit. If no PM-owned task exists on a project that needs new work, the row downgrades to a Flag row asking the PM to seed a parent task — see Job 5 "Picking the parent task" for the fallback.

Every signal that cannot reduce to one of these two actions — AND that the PM has already handled overnight (see "PM-action detection" below) — is **dropped from the queue**. Dropped signals are recorded in the Run Log detail page under a `Filtered signals` section so the PM can audit what was suppressed and why (see Job 11).

### Drop list (do NOT emit a queue row for any of these)

The matcher checks the drop list BEFORE deciding Create-subtask vs Flag. If a signal matches a drop reason, no row is emitted; the signal goes to `filtered_signals`.

- Hours-overrun alerts (`no-reply@whitelabeliq.com` notifications). Already surfaced by Orbit hours flow per PM feedback.
- PM's own Orbit tasks due today / overdue / status changes that the PM will see in Orbit's own due-date UI.
- Rollup digests aggregating stale tasks for PM review (PM scans Orbit directly).
- Standup / Daily Status meeting recaps where every action item lands on someone else (no Orbit ownership change for the PM and no PM-action gap).
- Asana / third-party SaaS automation emails (`no-reply@asana.com`, etc.) outside the closed allowlist.
- Marketing / onboarding / system emails (Anthropic, Notion, Claude.com welcome emails, calendar invite acks).
- **Signals the PM already handled overnight** — see PM-action detection rule below. This is the most important new drop reason: yesterday the AI was overshooting by emitting Flag-shaped rows even for signals the PM had replied to / commented on / acted on hours earlier.

### PM-action detection (the no-flag-if-PM-acted rule)

For every signal that survives the static drop list, run a PM-action check before deciding Flag vs Drop:

1. Capture the signal's arrival timestamp (`signal.timestamp` — when the email arrived or when the Orbit comment was posted).
2. Look for PM-originated activity in the window `(signal.timestamp, mode1_fire_time)` that matches the signal's context:
   - **Gmail:** any thread message with `labelIds` containing `SENT` from the PM's canonical email or alias, on the same `threadId` as the signal.
   - **Orbit:** any task comment authored by the PM (`actor_id == PM_user_id`) on the same task or project, OR any task field change (status, due date, assignee) actored by the PM. From `get_activity_log`.
3. If matching PM activity is found, drop the signal with `filter_reason: pm_already_handled`. Capture the PM-action timestamp + a one-line note in `filtered_signals` so the audit trail shows what was already resolved.
4. If no matching PM activity, the signal is unhandled — proceed to Job 5 to decide Create-subtask vs Flag.

**Bypass for priority-lane signals.** Signals carrying `bypass_pm_action_filter: true` (set by the Orbit collector's Priority Pass on `am_handed_to_pm_overnight_due_today` signals) are NOT subject to this filter. They proceed directly to Job 5 regardless of whether the PM commented on the parent task overnight. Rationale: an AM-handed task is pending delegation even if the PM acknowledged it ("got it, will look in the morning"); the delegation work itself hasn't happened. Acknowledgement ≠ handled for this signal class.

This rule directly fixes the over-flagging observed in early runs. The PM action set (Sent emails, Orbit comments) IS the PM telling the system "I've already got this." The matcher must respect that signal — except where the priority-lane bypass overrides.

### Project-type lookup

`project_type` comes from the Orbit project metadata returned by `get_project_details` (already in the collector's relationship map). When `project_type` is missing or unresolved, log `project_type_unknown` in the filter trace and DROP the row (do not guess). With the bucket split retired, `project_type` is no longer load-bearing for action selection — the matcher uses it for project-shape context only (the brief mentions the project type so the assignee sees it).

## What the matcher produces

A list of items. Each item becomes one row in the Morning Queue database. For each item, the matcher sets:

- `summary` — one-line plain-English summary the PM reads (in normal professional English, not simplified)
- `project` — render-ready string for the Notion `Project` column: `<Orbit project name> (#<orbit_project_id>)`, plus `— Maintenance` / `— Ad-hoc` suffix when the project type warrants. `Standalone` when no project resolves.
- `project_name` and `orbit_project_id` — separate raw fields preserved alongside `project` for downstream consumers (writers, executors) that need them split
- `orbit_task_link` — render-ready URL for the Notion `Orbit Task Link` column. For `Create subtask` rows: the Orbit URL of the pinned `parent_task_id` (priority lane) OR the inferred PM-owned parent task. For `Flag` rows: the Orbit URL of the bound task if the flag is anchored to one; `—` (em-dash, written as plain text into the URL field, or left empty) when no Orbit task is bound (e.g., a Gmail-only flag for PM action). The sub-task that Mode 2 may create is NOT placed here — it goes into `outcome` only.
- `recommended_action` — short phrase, e.g., "Create task + handoff to Vijay"
- `recommended_assignee` — name + short reason
- `ai_notes` — anything unusual, uncertainty flags, split reasoning. Priority-lane rows always carry the `<AM> put this on your plate overnight, due today. Proposed delegate: <name> (<reason>).` line here.
- `source_signals` — the set of collector signals that contributed to this item
- `proposed_orbit_body` — 6-section task body (from `schemas/orbit-dq-standard.md`), in plain language (per `writers/plain-language.md`) — used by the Orbit Executor if approved
- `proposed_handoff` — plain-language message for the assignee, copied + sent by PM through whatever channel they use for that team member
- `proposed_email` — normal-English email draft if the action involves emailing an AM or client

## Jobs in order

### Job 1 — Group signals by project

Use the Orbit relationship map to connect each signal to a project.

For Orbit signals: project is already on the signal.

For Gmail signals:
- Check sender domain → map to a client → find that client's active projects for this PM
- Read subject and body for project-name keywords
- If multiple projects match, pick the most recently active one and flag `Uncertain:` with an AI Note

Fathom is not a signal source. Meeting context is fetched as enrichment during Job 4b — see that job for the trigger-detection + lazy-fetch flow.

For priority-lane Orbit signals (`signal_type: am_handed_to_pm_overnight_due_today`): `project_id` and `parent_task_id` are already on the signal — no inference needed. Skip the rest of Job 1 for these and pass straight to Job 4b.

### Job 2 — Deduplicate across sources

A single issue may show up in both Gmail and Orbit. Matcher merges them into one item.

Match signals as the same item when:
- Same project AND same topic/deliverable (keyword overlap in content)
- Same actors involved AND signals fall within the same ~24-hour window
- One signal explicitly cites the other (e.g., Orbit comment quotes an email subject)

When merging, preserve all source signals in `source_signals`. The row's detail page will cite each source separately. Fathom enrichment (if fetched in Job 4b) is attached to whichever primary signal triggered the fetch — it is never a co-signal to dedupe against.

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
| `Create subtask` | Net-new scoped work landing under the PM's own parent task on the project. Universal across project types — including Ad-hoc and Maintenance. |
| `Flag` | Signal needs PM attention but is not delegate-able (PM owns next move: client email, scope decision, AM reply). No auto-execution. |

These are the ONLY two starting verbs a queue row's Summary may use. If the matcher cannot frame the row with one of these two, the row was misclassified — re-run the Output gating filter and either drop or downgrade.

#### Summary patterns

Pattern A — Create subtask:
```
Create subtask on <project short name> #<parent_task_id> — <proposed sub-task title>. To <assignee first name>.
```

Pattern B — Flag:
```
Flag <project or topic short name> — <one-clause why PM needs to look>. PM action.
```

Examples:
- `Create subtask on ECP AI Visibility Audit #110464 — Run AI Visibility audit on approved competitor list. To Hitesh.`
- `Create subtask on Solstice WP #106447 — Swap Contact Form brochure PDF. To Atul.`
- `Create subtask on Brother Plesk #109958 — Investigate patch ETA with WP Maintenance. To Atul.`
- `Flag 2010 Solutions — Ellen needs dev names for 27 May call. PM action.`
- `Flag Conversant FTP — credentials still missing, blocking dev work today. PM action.`

Rules:
- First token is one of the two locked verbs. No other openers.
- Project short name = client-readable phrase, max 4 words. Strip the long Orbit project title.
- For `Create subtask`: parent task ID + the proposed sub-task title MUST appear in the summary. PM reads the title in the column without opening the row detail page.
- For `Flag`: no Orbit task ID required; suffix `PM action` so PM knows at-a-glance there is no auto-execute.
- Use the assignee's FIRST name only in the summary (full name lives in `Recommended Assignee` column).
- No emojis. No narrative context.
- Max 120 chars total (Notion title field stays scannable across the column).

### Job 4b — Context cross-link (Gmail → Orbit) + Fathom enrichment fetch

Two independent passes happen here. Both run before Job 5 row-type decisions so the row detail page renders the full backstory in one place. Priority-lane signals get both passes first; normal Orbit signals get them second.

#### Pass 1 — Gmail → Orbit cross-link

For each Orbit signal, scan the Gmail signal list and link any Gmail signal whose `context_link` corroborates the Orbit event. A Gmail signal corroborates an Orbit signal when ANY of the following hold:

1. **Project match.** `gmail.context_link.project_id_candidates` contains the Orbit signal's `project_id`.
2. **Actor match.** `gmail.context_link.actor_emails` intersects with the Orbit signal's AM identity (for priority-lane: `signal.am_actor_email` + aliases) or with the parent task's follower / assignee emails.
3. **Topic match.** `gmail.context_link.topic_keywords` overlaps with the parent task title or task body, above a simple threshold (≥ 2 keyword matches, case-insensitive). No embeddings — substring or token-overlap is sufficient.
4. **Time proximity.** `gmail.context_link.timestamp` is within ±24h of the Orbit signal's event timestamp (`event_timestamp` for priority-lane, `timestamp` for normal-pass).

Match strength: a Gmail signal that hits 2+ rules is high-confidence corroboration. A signal that hits exactly 1 rule is weak corroboration — still attach, but mark it weak so the row detail Sources section can render it under a `Possible context` subheading.

For each Orbit signal, populate `context_signals[]` with the matched Gmail signals, each annotated with `match_strength: "strong" | "weak"` and `match_rules: [<which rules fired>]`.

**Additivity.** Linking a Gmail signal as context does NOT consume it. The same Gmail signal may also surface as its own row (under Job 5 — Create subtask or Flag) IF it contains a fresh ask (new requirement, new client question, new commitment). The matcher decides that independently per signal in Job 5.

#### Pass 2 — Fathom enrichment fetch (lazy, on-demand)

Independently of Pass 1, scan EVERY primary signal (Orbit-priority + Orbit-normal + Gmail) for meeting-reference trigger phrases per the trigger-phrase list documented in `collectors/fathom.md`. For each detected reference, call the Fathom enrichment service:

```
result = fetch_enrichment(reference, signal_context)
```

If `result` is non-null, attach it as `signal.enrichment.fathom = result` on the originating primary signal. If `result` is null (Fathom unavailable, no matching meeting, or reference unresolved), continue without enrichment — the primary signal still flows through Job 5 normally.

Multiple primary signals may reference the same meeting; each gets its own `EnrichmentResult` attachment so the writer can cite the meeting under each row that referenced it. The enrichment service may cache duplicate lookups within a single Mode 1 run — that is an implementation detail and does not change the matcher contract.

**Critical:** Fathom enrichment never produces a `context_signal` and never originates a row. It only annotates an existing primary signal. If a meeting was referenced in a Gmail signal that is later filtered out by Job 5, the enrichment is discarded with it — there is no Fathom-only row.

**Writer impact.** `writers/notion.md` reads `context_signals[]` AND `enrichment.fathom` on each row. `context_signals[]` renders under the row's Sources H1 section per `schemas/row-detail-page.md`. `enrichment.fathom` renders under the Fathom H2 subsection of Sources (per `schemas/row-detail-page.md`), using the citation formats in `writers/source-citation.md`. Priority-lane rows see the AM-handed Orbit parent task at the top of Sources, the corroborating Gmail thread (if any) below, and the Fathom enrichment (if any) under the Fathom H2.

### Job 5 — Recommend the action (2 actions only)

The action set is closed and matches the Output gating section above. Pick exactly one:

- **Create subtask** — new sub-task is created in Orbit under the **PM-owned parent task** on the project. Universal across project types (Ad-hoc and Maintenance included). Output: `parent_task_id` (with `assignee_id == PM_user_id`), `task_title` (the actual sub-task title that will appear in Orbit), `assignee_id` for the dev, full 6-section body per `schemas/orbit-dq-standard.md` in plain language. The task title is generated explicitly and shown in the row Summary so the PM does not need to open the detail page to know what is being created.
- **Flag** — signal is recorded as a PM-attention row, no Orbit task created, no Mode 2 execution. Output: `pm_next_step` (one short clause describing the next move PM should take — e.g., "Reply to Ellen with the dev names", "Decide whether the FTP creds chase moves to client direct"), no `task_title`, no `proposed_orbit_body`, no `assignee_id`. The Status default is still `Recommended Action`; when the PM resolves the flagged item externally, they mark it `Skip. No Action Needed`.

If neither action fits, the signal does not become a row. Apply the Output gating filter and route to the `Filtered signals` log (Job 11).

#### Choosing Create subtask vs Flag

For each signal that survives the drop list AND the PM-action detection check, decide:

1. **Priority-lane override.** If the signal carries `signal_type: am_handed_to_pm_overnight_due_today`, it is ALWAYS `Create subtask`. Never `Flag`. The PM-owned parent pool is bypassed — the parent is pinned to `signal.parent_task_id` (the AM-assigned parent the PM is already the assignee on). Proceed to Job 6 to suggest the delegated assignee.
2. **Is there a clear dev task to delegate?** If yes — concrete deliverable, identifiable scope, a developer can pick it up and run — choose `Create subtask`. Examples: "swap brochure PDF on Contact Form thank-you emails", "run AI Visibility audit on the approved competitor list", "investigate patch ETA for the Plesk security warning and update the due date".
3. **Is the next move PM-owned?** If yes — reply to an AM, decide a scope question, brief the team for a meeting, pick which devs attend a call — choose `Flag`. Examples: "Ellen needs dev names for the 27 May Joe Warner call", "FTP credentials still missing — PM decides whether to chase client directly".
4. **Edge case — Create subtask path requires PM-owned parent.** If `Create subtask` was chosen but the project has NO PM-owned parent task (PM-owned parent pool empty), downgrade to `Flag`. The flag's `pm_next_step` becomes: "Seed a parent task on <project> so future sub-tasks have a home." Do NOT drop the signal; the PM should know they need to create the parent. (This edge case never applies to priority-lane signals — they always have a pinned parent.)

#### Picking the parent task for sub-task creation (Create subtask path)

The parent **must be a task currently assigned to the running PM** on the project. Procedure:

1. **Priority-lane override.** If the signal carries `signal_type: am_handed_to_pm_overnight_due_today`, set `parent_task_id = signal.parent_task_id` directly and skip the rest of this procedure. The Orbit collector's Priority Pass already determined the parent is on the PM's plate (`assignee_id == PM`) and an AM put it there with `due_date = today`. No inference needed — use the pinned parent exactly. The proposed sub-task nests under it when Mode 2 executes.
2. From the relationship map, list every open task on the project where `assignee_id == PM_user_id`. Call this the PM-owned parent pool for the project.
3. **If the PM-owned parent pool is empty** — downgrade the row to `Flag` per the edge case above. Do not drop.
4. **If the PM-owned parent pool has exactly one task** — that's the parent. Use it.
5. **If the PM-owned parent pool has multiple tasks** — match the signal to a parent by phase / feature / deliverable keyword overlap with each parent's title. Recency breaks ties (most recently updated parent wins). If two parents tie cleanly on score, set `parent_task_id = null` and write AI Notes: `Uncertain: sub-task could go under PM-owned parent #X (<title>) or #Y (<title>). Please pick.` Row is still emitted as Create subtask; parent axis flagged.

#### Generating the task title (Create subtask only)

The `task_title` is the actual string Orbit will use as the sub-task name AND it appears in the row Summary so the PM reads it at-a-glance. Rules:

- 6–12 words, action-led, verb-first (`Run`, `Build`, `Swap`, `Investigate`, `Draft`, `Review`, `Fix`, `Migrate`, etc.).
- Names the deliverable concretely. Not "follow up on Bowser things" — say "Build sitemap structure for bowsertrains.com per Angela's plan".
- Does not include client name (the project context covers that).
- Does not start with the project type. Start with the verb.
- Plain English. Role-specific technical terms preserved per `writers/plain-language.md`.

Sub-task `assignee_id` (different from `parent_task_id`) is picked via the Job 6 decision tree on the developer pool. The PM owns the parent; the dev owns the child.

#### Compose `orbit_task_link` and `project` for the row

After `parent_task_id` is resolved (priority-lane pin OR PM-owned parent inference), build the two render-ready fields the writer consumes for the Notion queue columns:

- **`orbit_task_link`** — the Orbit URL of the parent task. Pull from the source signal:
  - Priority-lane signal → `signal.parent_task_url` (Orbit collector Priority Pass already emits this).
  - Normal `Create subtask` signal → look up the PM-owned parent task's URL from the Orbit collector's relationship map (every task carries a `task_url`); when the parent was inferred (not directly on a signal), match by `parent_task_id` against the relationship map's task index.
  - **`Flag` rows**: if the flag is anchored to a specific Orbit task (e.g., `signal.task_id` present, no PM-action on the task), set `orbit_task_link = signal.task_url`. If the flag has no underlying Orbit task (Gmail-only signal, no project task created yet), set `orbit_task_link = "—"` (em-dash literal). Never null — writer expects the field.
  - **Null `task_url` on a signal**: cross-check the Orbit collector's relationship map (every task in the universe is indexed by `task_id` with its URL). If the URL is still missing there, set `orbit_task_link = "—"` and append an AI Note: `Orbit task URL not surfaced by MCP on this signal — open via task ID #<id>.` Do not block the row.
- **`project`** — render-ready string for the Notion `Project` column. Format: `<project_title> (#<project_id>)`. For Maintenance / Ad-hoc projects, append `— Maintenance` / `— Ad-hoc` (read project_type from the Orbit relationship map). For rows with no resolved Orbit project (Gmail-only flag with no project mapping), emit `Standalone`. Preserve the raw `project_name` and `orbit_project_id` separately for downstream consumers.

No Mode 2 step writes into `orbit_task_link` later — the column is frozen at row-create time and stays as the parent reference. Sub-task URLs (created by Mode 2) go into `Outcome` only, per `executors/orbit.md` Outcome format.

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

#### Priority-lane AI Notes wording

For priority-lane signals (`signal_type: am_handed_to_pm_overnight_due_today`), the AI Notes field on the row carries an explicit one-liner naming the AM actor and the matcher's chosen branch, so the PM sees at-a-glance who handed the work and why the delegate was picked. Format:

```
<AM full name> put this on your plate overnight, due today. Proposed delegate: <delegate name> (<branch reason>).
```

Examples:

- `Sarah Chen put this on your plate overnight, due today. Proposed delegate: Vijay Patel (FE) — primary FE on Agency X, 24 hrs on current homepage task.`
- `Sarah Chen put this on your plate overnight, due today. Proposed delegate: Atul (WP) — WordPress / PHP in Matrix A, no prior task on BrightPath, lightest workload (3 open tasks).`
- `Sarah Chen put this on your plate overnight, due today. Proposed delegate: Uncertain — no FE in Matrix A or Floaters. Cross-matrix candidates: Jay Panchal (Design Matrix). Please pick.`

The "put this on your plate overnight, due today" phrasing is intentional — it tells the PM (a) this is not a signal you generated, (b) the clock is today, (c) someone external to your team made the assignment decision. This framing matters because the row appears at the top of the queue and the PM should grok the context in under three seconds.

### Job 7 — Generate the proposed Orbit body (Create subtask path only)

The body content depends on the row's action:

**Create subtask path** — pre-write the full 6-section task body per `schemas/orbit-dq-standard.md`. Plain language (4th–5th grade English) per `writers/plain-language.md` since the delivery team reads it. Keep role-specific technical terms. Strip corporate English. The 6-section body lands in Orbit as the sub-task description when Mode 2 fires.

**Flag path** — does NOT carry a `proposed_orbit_body`. No Orbit write happens for a Flag row. Job 7 is skipped for Flag rows entirely. The row's detail page substitutes `Proposed Orbit Task Body` with `PM next step` (rendered from the `pm_next_step` clause set in Job 5).

### Job 8 — Generate the proposed handoff

The handoff message is the team handoff draft the PM copies-and-sends after Mode 2 fires. It lands on the row's Notion detail page under the `Proposed Handoff` H1 section and is also appended to the row's `Outcome` block when Mode 2 executes. The PM delivers it through whatever channel they use for that team member (direct message, in-person, email, etc.). Generate it for the **Create subtask** path only:

- Tell the assignee a new sub-task landed under the PM's parent task, the work in one short paragraph, the parent context, and the sub-task URL.

Plain language per `writers/plain-language.md`. Reminder to log hours if Preferences has that always-include rule.

For priority-lane rows specifically, the handoff body should additionally note that the parent task was handed down by the AM: `<AM name> assigned this task to <PM name> overnight; passing the dev work to you to start today.` This sets the assignee's expectation that the timeline is same-day.

Flag rows do NOT generate a handoff (no dev work to delegate; PM owns the next move).

NO handoff is generated for AMs or clients — those are PM-handled outside the queue per the Output gating filter.

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
  "source": <orbit | gmail>,
  "summary": <one-line description of what was dropped>,
  "filter_reason": <one of: pm_already_handled | hours_overrun_alert | pm_own_task_orbit_ui | rollup | standup_recap | third_party_automation | marketing_or_system_email | project_type_unknown>,
  "citations": [<source links — gmail thread URL, orbit task URL>]
}
```

Fathom never appears in `filtered_signals` — it is not a signal source. Enrichment failures (Fathom unavailable, reference unresolved) are logged separately under the Run Log's connector-status section, not under filtered signals.

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
0. **Priority-lane rows first** — items where `signal_type == am_handed_to_pm_overnight_due_today`. Within the priority lane, sort by `parent_task_due_time` ascending if available (Orbit due-date timestamp); fall back to `event_timestamp` ascending (earliest AM action first) when no due-time is set. These rows lead the queue every time so the PM cannot miss them on the way to the office.
1. Items with high-urgency signals first (overdue tasks, blockers, urgent language like "urgent", "blocking", "today")
2. Items from external clients ahead of internal
3. Items involving multiple sources (cross-system signals) ahead of single-source
4. Everything else by recency (newest first)

No scoring model, no scoring columns. Just a sensible order so the PM reads important things first. Rule 0 is non-negotiable — priority-lane rows always sit at the top of the queue regardless of what rules 1–4 would do to them.

## Output format

An array of item records ready for `writers/notion.md` to consume. Each item has everything listed in "What the matcher produces" at the top of this file, plus an ordered position in the array.
