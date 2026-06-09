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

## Output gating — five locked verbs

The Morning Queue exists to drive **exactly five AI actions and only five**:

1. **`Create subtask`** — pod-resource work (work_type ∈ HTML_CSS, PHP_BACKEND, QA) landing under the PM-owned parent task. No matching open subtask of the same work_type exists. Sub-task carries an explicit `task_title` AND the full 6-section `schemas/orbit-dq-standard.md` brief.
2. **`Reopen subtask`** — same as Create subtask BUT an existing open subtask of matching work_type already sits under the parent (Job 5.5 detected it). Executor reopens the existing subtask, posts new-work comment, reassigns to last non-PM dev. No new task created.
3. **`Hand off parent task`** — non-pod work (work_type ∈ AUDIT, QUOTE, SEO, DESIGN, CONTENT, BA). Executor reassigns the existing PM-owned parent directly to the functional pool leader; PM stays as internal follower. No subtask.
4. **`Flag`** — signal needs PM attention but is not delegate-able (PM owns next move: client email, scope decision, AM reply). No auto-execution. Flag rows carry summary + sources + suggested PM-next-step. PM resolves manually and marks `Skip. No Action Needed`.
5. **`Create parent task`** — Possible Orbit miss; an unactioned client email (delivery reply, delivery-token, or issue/feature mail) with NO corroborating Orbit task. Project resolved from the workload map or by lazy Orbit search; dedup-checked. On PM approval, Mode 2 creates the parent on the resolved project assigned to the PM, so the PM can spawn sub-tasks under it in later runs.

The "PM's own parent task on that project" semantic is the load-bearing constraint for paths 1, 2, and 3: the PM holds a parent task for each active engagement. Verbs 1+2 spawn / reuse sub-tasks under that parent; verb 3 reassigns the parent itself. If no PM-owned task exists on a project that needs new work, the row downgrades to a Flag row asking the PM to seed a parent task — see Job 5 "Picking the parent task" for the fallback (applies to verbs 1+2+3).

Every signal that cannot reduce to one of these five actions — AND that the PM has already handled overnight (see "PM-action detection" below) — is **dropped from the queue**. Dropped signals are recorded in the Run Log detail page under a `Filtered signals` section so the PM can audit what was suppressed and why (see Job 11).

### Drop list (do NOT emit a queue row for any of these)

The matcher checks the drop list BEFORE deciding Create-subtask vs Flag. If a signal matches a drop reason, no row is emitted; the signal goes to `filtered_signals`.

- Hours-overrun alerts (`no-reply@whitelabeliq.com` notifications). Already surfaced by Orbit hours flow per PM feedback.
- PM's own Orbit **overdue** tasks / passive status changes that the PM will see in Orbit's own due-date UI. **Due-today tasks are NOT dropped** — under the single-pane mandate (the PM must not have to open Orbit to learn what's due today) every due-today task surfaces via the collector's `pm_task_due_today` signal and is routed by the Job 5 due-today branch (actionable verb if another signal exists on the task, else a `Flag`). Only `due_date < today` (overdue) and bare status changes remain dropped here.
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

**Bypass for priority-lane and due-today signals.** Signals carrying `bypass_pm_action_filter: true` are NOT subject to this filter — they proceed directly to Job 5 regardless of whether the PM commented on the task overnight. Two signal classes set this flag: (a) the Priority Pass's `am_handed_to_pm_overnight_due_today` signals, and (b) the normal pass's `pm_task_due_today` signals. Rationale: for (a), an AM-handed task is pending delegation even if the PM acknowledged it ("got it, will look in the morning") — the delegation work itself hasn't happened. For (b), a standing due date is a fact about the task, not a signal the PM "replied to"; a prior comment does not retire today's deadline, so the task must still surface. Acknowledgement ≠ handled for either class.

This rule directly fixes the over-flagging observed in early runs. The PM action set (Sent emails, Orbit comments) IS the PM telling the system "I've already got this." The matcher must respect that signal — except where the priority-lane bypass overrides.

### Project-type lookup

`project_type` comes from the Orbit project metadata returned by `get_project_details` (already in the collector's relationship map). When `project_type` is missing or unresolved, log `project_type_unknown` in the filter trace and DROP the row (do not guess). With the bucket split retired, `project_type` is no longer load-bearing for action selection — the matcher uses it for project-shape context only (the brief mentions the project type so the assignee sees it).

## What the matcher produces

A list of items. Each item becomes one row in the Morning Queue database. For each item, the matcher sets:

- `summary` — one-line plain-English summary the PM reads (in normal professional English, not simplified). **Topic-style — names the deliverable / scope, NOT the PM action**. The action verb lives in `Recommended Action` column and the row's top callout. See Job 4 for composition rules and samples.
- `task_brief` — short paragraph (2–4 sentences) rendered below the Summary heading on the row detail page. Describes what the work is about + the new update / latest signal content that triggered this row. Pulled primarily from the most recent input signal (top of email thread, latest Orbit comment, new AM clarification) — not from the older parent-task description. See Job 7b for composition rules.
- `work_type` — classifier output from Job 5a; one of `HTML_CSS | PHP_BACKEND | QA | AUDIT | QUOTE | SEO | DESIGN | CONTENT | BA | OTHER`. Drives Job 5 action choice and Job 6 pod-boundary routing.
- `project` — render-ready string for the Notion `Project` column: `<Orbit project name> (#<project_number>)`, plus `— Maintenance` / `— Ad-hoc` suffix when the project type warrants. `Standalone` when no project resolves. **`project_number` is Orbit's user-visible project code (string, e.g. `"16915"`), NOT the internal `id` field — per SKILL.md non-negotiable rule #22.**
- `project_name`, `project_number`, and `orbit_project_id` — separate raw fields preserved alongside `project` for downstream consumers. `project_number` (string from `get_project_details.project_number`) is the user-visible identifier; `orbit_project_id` (int from `get_project_details.id`) is the internal PK used only inside URLs and the relationship map.
- `orbit_task_link` — render-ready URL for the Notion `Orbit Task Link` column. For `Create subtask` rows: the Orbit URL of the pinned `parent_task_id` (priority lane) OR the inferred PM-owned parent task. For `Flag` rows: the Orbit URL of the bound task if the flag is anchored to one; `—` (em-dash, written as plain text into the URL field, or left empty) when no Orbit task is bound (e.g., a Gmail-only flag for PM action). The sub-task that Mode 2 may create is NOT placed here — it goes into `outcome` only.
- `recommended_action` — short phrase, e.g., "Create task + handoff to Vijay"
- `recommended_assignee` — name + short reason
- `ai_notes` — anything unusual, uncertainty flags, split reasoning. Priority-lane rows always carry the `<AM> put this on your plate overnight, due today. Proposed delegate: <name> (<reason>).` line here.
- `source_signals` — the set of collector signals that contributed to this item
- `proposed_orbit_body` — 6-section task body (from `schemas/orbit-dq-standard.md`), in plain language (per `writers/plain-language.md`) — used by the Orbit Executor if approved
- `proposed_handoff` — plain-language message for the assignee, copied + sent by PM through whatever channel they use for that team member
- `proposed_email` — normal-English email draft if the action involves emailing an AM or client
- `latest_signal_anchor` — `{ source, id, timestamp_iso, author_name, excerpt, source_url }`. Required on every row. Picked by Job 7 as the newest across the collectors' per-task / per-thread `latest_signal` fields plus any attached Fathom `meeting_date`. Pass-through, not recomputed. Writer renders as the first line of the Task Brief block (`**Triggered by:** ...`). See SKILL.md rule #24.

## Jobs in order

### Job 1 — Group signals by project

**Entry-gate.** Job 1 reads `priority_signals[]` first (per Sort rule 0 — priority-lane rows lead the queue). Before any grouping work, verify `priority_signals[]` has been populated by Mode 1 sub-step 1d (the priority pass's local filter). If `priority_signals` is still `null` or `undefined` (rather than `[]` for "no priority signals this morning"), the matcher MUST wait for 1d to complete before proceeding. This is the explicit consumer-side encoding of the dependency between 1d and matcher — Mode 1's Step 1 launch graph deliberately does NOT serialize 1d ahead of 2a (Gmail collector); the dependency lives here, at the consumer.

In practice, the join point at Mode 1 sub-step 1f ensures all collector outputs (including 1d's `priority_signals[]`) are populated before Step 4 (matcher invocation) begins. The Job 1 entry-gate is the second layer of defense — if 1f's join semantics were ever weakened, this gate still catches the missing input.

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
- **Two Orbit signals carry the same `task_id`** — always the same item, merge unconditionally (no topic/window test needed). This is the rule the Job 5 due-today branch relies on: a `pm_task_due_today` signal and an `activity_log` / `new_comment` / `status_change` signal on the same task collapse into ONE row. The actionable signal supplies the verb; the due-today signal contributes only `row.due_today = true`. One task → one row.
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

### Job 4 — Generate the one-line summary (topic-style, deliverable-first)

Normal professional English. This is what the PM reads when scanning the queue. The summary is **topic-led, NOT action-led** — it names the deliverable / scope / subject of the work, not what the PM is being asked to do. The PM action verb lives in the `Recommended Action` column and the row's top Action Block callout, NOT in the Summary text.

This is a deliberate rollback from an earlier verb-first / action-led Summary rule. Reason: PMs scanning the queue want to know "what is this row about" before they care about what they're being asked to do. The verb is redundant in Summary because (a) the `Recommended Action` column already states it explicitly, (b) the top callout on the row detail page restates it, (c) when every row starts with the same handful of verbs the column visually flattens and stops aiding scan. Topic-led Summaries let the deliverable / client context land first.

#### Locked verb list — applies to `Recommended Action` column and Action Block callouts, NOT to Summary

There are exactly five locked verbs. Each row picks one in Job 5 + 5a + 5.5. The Summary text contains the deliverable / scope; the verb appears in `Recommended Action` (Notion column) and in the top callout on the row detail page.

| Verb | Used when |
|---|---|
| `Create subtask` | Net-new pod-resource work (work_type ∈ HTML_CSS, PHP_BACKEND, QA) landing under the PM-owned parent task. No existing open subtask of the same work_type exists under the parent. |
| `Reopen subtask` | Same pod-resource work_type, but an existing open subtask already sits under the parent (Job 5.5 detected it). Executor flips status, appends new-work comment, reassigns to last non-PM dev. |
| `Hand off parent task` | Work that does NOT touch HTML/PHP/QA pod resources (work_type ∈ QUOTE, SEO, AUDIT, DESIGN, CONTENT, BA). Executor reassigns the existing PM-owned parent directly to the pod leader; no subtask. |
| `Flag` | Signal needs PM attention but is not delegate-able (PM owns next move: client email, scope decision, AM reply). No auto-execution. |
| `Create parent task` | An unactioned client email with NO matching Orbit task — Possible Orbit miss (S1a delivery reply / S1b delivery-token / S2 issue-feature). On PM approval, Mode 2 creates a parent Orbit task on the resolved project assigned to the PM. See "Unactioned client signal → Create parent task" sub-section in Job 5. |

These five are the ONLY values that may appear in the `Recommended Action` column's verb position. If the matcher cannot frame the row with one of these five, the row was misclassified — re-run the Output gating filter and either drop or downgrade.

#### Summary patterns — topic-style

Pattern A — work-shaped row (`Create subtask` / `Reopen subtask` / `Hand off parent task` / `Create parent task`):
```
<Deliverable / scope phrase> — <client or project short name> [#<project_number>]
```

Pattern B — Flag row:
```
<Topic / client signal> — <one-clause why PM needs to look>
```

Examples (note: no verb at the start — verb lives in `Recommended Action` column):
- `Quote for Mega Menu restructure and four new pages — Process Barron change order #16915`
- `AI Visibility audit on approved competitor list — ECP audit #16842`
- `Contact Form brochure PDF replacement — Solstice WP #16720`
- `Plesk patch ETA confirmation with WP Maintenance — Brother Site Issues #16631`
- `Dev name nominations required for 27 May Joe Warner call — 2010 Solutions`
- `Missing FTP credentials blocking development — Conversant`
- `Contact form outage reported by client, time-critical — Agency X (Possible Orbit miss)`

Rules:
- **No locked verb prefix.** Do not start with `Create subtask`, `Reopen subtask`, `Hand off parent task`, `Flag`, or `Create parent task`. Those words belong in `Recommended Action`, not here.
- **Lead with the deliverable / scope.** Examples: `Quote for ...`, `AI Visibility audit on ...`, `Contact Form brochure PDF replacement`. The PM should be able to grasp what the work IS in the first 5–8 words.
- **Project / client trailer.** End with ` — <short name> #<project_number>` so the PM can match rows to projects without leaving the column. Use `project_number` (the user-visible string code, e.g. `#16915`), NEVER the internal `id` field (per SKILL.md non-negotiable rule #22). Skip the `#<project_number>` suffix for Flag rows that aren't bound to a specific Orbit project, or for `Create parent task` rows where the project lookup was unambiguous but no project number was retrieved before Mode 2.
- **Possible Orbit miss tag.** `Create parent task` rows append ` (Possible Orbit miss)` at the end of the Summary so the PM grasps the signal class at-a-glance.
- **Professional tone.** Full English phrasing — avoid casual contractions, sentence fragments where a noun phrase reads cleaner, slang, exclamation, or hedge words ("kinda", "stuff", "thing"). The Summary reads like a one-line entry in a professional task tracker, not a colleague's quick note.
- **No emojis. No decorative glyphs. No assignee name. No PM-action phrasing.** Assignee lives in `Recommended Assignee` column; PM action lives in `Recommended Action` column.
- **Max 120 chars total** (Notion title field stays scannable across the column).

### Job 4b — Context cross-link (Gmail → Orbit) + Fathom enrichment fetch

Two independent passes happen here. Both run before Job 5 row-type decisions so the row detail page renders the full backstory in one place. Priority-lane signals get both passes first; normal Orbit signals get them second.

#### Pass 1 — Gmail → Orbit cross-link

**Default-on enrichment, NOT conditional on "Orbit brief looks thin".** For every Orbit signal (priority-lane and normal-pass), the matcher MUST check whether a related Gmail thread exists and read what's there before Job 7 composes the proposed body. Emails routinely carry context that never made it into the formal Orbit task: AM clarifications, client constraints, scope tweaks, deadline nudges, prior decisions referenced ("as we agreed last week"), follow-ups on missing inputs. The Orbit task body cannot be assumed complete just because it exists. **The bias is over-include context, not under-include.**

For each Orbit signal, scan the Gmail signal list and link any Gmail signal whose `context_link` corroborates the Orbit event. A Gmail signal corroborates an Orbit signal when ANY of the following hold:

1. **Project match.** `gmail.context_link.project_id_candidates` contains the Orbit signal's `project_id`.
2. **Actor match.** `gmail.context_link.actor_emails` intersects with the Orbit signal's AM identity (for priority-lane: `signal.am_actor_email` + aliases) or with the parent task's follower / assignee emails.
3. **Topic match.** `gmail.context_link.topic_keywords` overlaps with the parent task title or task body, above a simple threshold (≥ 2 keyword matches, case-insensitive). No embeddings — substring or token-overlap is sufficient.
4. **Time proximity.** `gmail.context_link.timestamp` is within ±7 days of the Orbit signal's event timestamp (`event_timestamp` for priority-lane, `timestamp` for normal-pass). Widened from the original ±24h because email threads about a task routinely span days before the Orbit task surfaces overnight.

Match strength: a Gmail signal that hits 2+ rules is high-confidence corroboration. A signal that hits exactly 1 rule is weak corroboration — still attach, but mark it weak so the row detail Sources section can render it under a `Possible context` subheading. **Even weak matches are attached** — Job 7 decides whether to pull from them based on the thread contents, not on match strength alone.

For each Orbit signal, populate `context_signals[]` with the matched Gmail signals, each annotated with `match_strength: "strong" | "weak"` and `match_rules: [<which rules fired>]`.

**Read the thread, not just the metadata.** The Gmail collector (`collectors/gmail.md` § Full thread context) pulls every linked thread end-to-end via `gmail_read_thread`. Each attached Gmail signal therefore carries the full thread body (every message, every attachment, the PM's own prior replies). When Job 7 composes the proposed Orbit body, it MUST read through the full attached thread(s) — including long threads that span many days or many messages — and surface anything that adds DO scope, WHY motivation, CONTEXT history, DONE-WHEN criteria, or REFS the assignee would otherwise miss. Long threads are not a reason to skip — they are the reason to read carefully. See Job 7 for the composition rules.

**Additivity.** Linking a Gmail signal as context does NOT consume it. The same Gmail signal may also surface as its own row (under Job 5 — Create subtask or Flag) IF it contains a fresh ask (new requirement, new client question, new commitment). The matcher decides that independently per signal in Job 5.

#### Pass 2 — Fathom enrichment fetch (lazy, on-demand)

Independently of Pass 1, scan EVERY primary signal (Orbit-priority + Orbit-normal + Gmail) for meeting-reference trigger phrases per the trigger-phrase list documented in `collectors/fathom.md`. For each detected reference, call the Fathom enrichment service:

```
result = fetch_enrichment(reference, signal_context)
```

If `result` is non-null, attach it as `signal.enrichment.fathom = result` on the originating primary signal. If `result` is null (Fathom unavailable, no matching meeting, or reference unresolved), do NOT yet treat the null as terminal — first try the gmail-attachment fallback (next paragraph).

**Gmail-attachment fallback for Fathom misses.** When `fetch_enrichment()` returns null, call `gmail.find_transcript_in_email(reference, signal_context, gmail_signal_list)` (see `collectors/gmail.md` § Transcript fallback for the helper interface). The helper scans the already-collected Gmail signal list for transcript-shaped attachments from Fathom-sender / attendee-sender emails ±2 days around the meeting reference — no new Gmail MCP calls. If the helper returns a non-null `EnrichmentResult`, attach it as `signal.enrichment.fathom = result` exactly as if Fathom had served it, with one extra field `enrichment_source: "gmail_attachment_fallback"` so the writer can render the citation appropriately. Only if the gmail-attachment fallback also returns null does the primary signal flow through Job 5 without any enrichment. The Incidents log entry (per `collectors/fathom.md` error-handling table) fires once per Mode 1 run only when BOTH Fathom AND the gmail fallback failed for at least one signal.

Multiple primary signals may reference the same meeting; each gets its own `EnrichmentResult` attachment so the writer can cite the meeting under each row that referenced it. The enrichment service may cache duplicate lookups within a single Mode 1 run — that is an implementation detail and does not change the matcher contract.

**Critical:** Fathom enrichment never produces a `context_signal` and never originates a row. It only annotates an existing primary signal. If a meeting was referenced in a Gmail signal that is later filtered out by Job 5, the enrichment is discarded with it — there is no Fathom-only row.

**Writer impact.** `writers/notion.md` reads `context_signals[]` AND `enrichment.fathom` on each row. `context_signals[]` renders under the row's Sources H1 section per `schemas/row-detail-page.md`. `enrichment.fathom` renders under the Fathom H2 subsection of Sources (per `schemas/row-detail-page.md`), using the citation formats in `writers/source-citation.md`. Priority-lane rows see the AM-handed Orbit parent task at the top of Sources, the corroborating Gmail thread (if any) below, and the Fathom enrichment (if any) under the Fathom H2.

### Job 5 — Recommend the action (5 locked verbs, see Job 4 table)

The action set is closed: `Create subtask`, `Reopen subtask`, `Hand off parent task`, `Flag`, `Create parent task`. Picked in this order:

1. **Run Job 5a — work-type classifier.** Classify the signal into one of: `HTML_CSS | PHP_BACKEND | QA | AUDIT | QUOTE | SEO | DESIGN | CONTENT | BA | OTHER`. Job 5a output (`row.work_type`) drives both this Job 5 verb pick AND Job 6 pod-boundary routing.
2. **Priority-lane override (still applies).** If the signal carries `signal_type: am_handed_to_pm_overnight_due_today`, the parent is pinned to `signal.parent_task_id`. Action defaults to `Create subtask` (or `Reopen subtask` after Job 5.5 check). Never `Flag`.
2.5. **Due-today branch.** If the signal carries `signal_type: pm_task_due_today`, first merge by `task_id` against every other signal surfaced this run (Job 2 dedup, keyed on `task_id` for Orbit signals):
    - **Another signal exists on the same task** (new comment, status change, attached Gmail ask, or — handled above — an AM handoff). That signal's verb wins; the standalone due-today signal is consumed, NOT emitted as its own row. Set `row.due_today = true` so Job 10 prepends `Due today.` to AI Notes and Sort rule 0.5 lifts the row into the due-today band. One task → one row.
    - **due-today is the only signal on the task** → emit a `Flag` row. Anchor = the `orbit_due_today` `latest_signal` from the collector. `pm_next_step = "Due today — review and delegate, progress, or close."` The row carries **no proposed Orbit body, no recommended assignee (`— (PM action)`), and no handoff** — it stays a Flag; the skill does not invent work for a task with no new ask. **BUT this Flag is the one Flag class that fires a *light* deep-read** (see Job 7 § Light deep-read for due-today Flags): a single `get_task_details(task_id)` call — its bundled newest comments (off the flattened, newest-first tree) + `subtasks[]` are enough; NO `list_task_comments` fallback — so Job 7b can compose a "here's where this stands" task_brief and surface the open-subtask status — letting the PM decide inside Notion without opening Orbit (the single-pane mandate). If the cached/fetched `subtasks[]` shows an open subtask already under this task, the brief notes `subtask #<id> open under <assignee> — likely needs your review, not re-delegation.` and AI Notes carries `Due today — subtask #<id> already open under <assignee>; informational, no action needed unless it's stalled.` This Flag is exempt from the "PM will see it in Orbit's UI" drop (Output gating) — that exemption is the whole point of the change.
3. **Flag the PM-owned moves first.** If the next move is PM-owned (reply to an AM, decide a scope question, brief the team for a meeting), choose `Flag` regardless of work_type. Examples: "Ellen needs dev names for the 27 May Joe Warner call", "FTP credentials still missing — PM decides whether to chase client directly".
4. **Unactioned client signal check (Possible Orbit miss).** Before defaulting a Gmail-only signal to a workshop verb, run the "Unactioned client signal → Create parent task" detection below. If the entry gate + any one of the three trigger sub-classes (S1a/S1b/S2) fire, run project resolution + dedup: a resolved, non-duplicate project → `Create parent task`; project not found or a topic-matching open task already exists → `Flag` with the corresponding `pm_next_step`.
5. **Work-type → verb branch.** For signals that are dev-shaped work (not PM-owned, not Possible Orbit miss), the verb depends on `work_type`:

    | `work_type` | Default verb (pre Job 5.5) |
    |---|---|
    | `HTML_CSS`, `PHP_BACKEND`, `QA` | `Create subtask` (pod-resource path; Job 5.5 may flip to `Reopen subtask`) |
    | `AUDIT`, `QUOTE`, `SEO`, `DESIGN`, `CONTENT`, `BA` | `Hand off parent task` (non-pod path; no subtask) |
    | `OTHER` | `Create subtask` (fallback; Job 6 falls through to the existing 4-branch tree) |

6. **Edge case — `Create subtask` path requires PM-owned parent.** If `Create subtask` or `Reopen subtask` was chosen but the project has NO PM-owned parent task (PM-owned parent pool empty), downgrade to `Flag` with `pm_next_step: "Seed a parent task on <project> so future sub-tasks have a home."` Same edge handling as before. (Priority-lane signals always have a pinned parent, so they never hit this edge.)
7. **Edge case — `Hand off parent task` also requires the PM-owned parent.** If `Hand off parent task` was chosen but the PM-owned parent pool for the project is empty, downgrade to `Flag` with `pm_next_step: "Seed a parent task on <project> and hand it off manually to the <pool> lead — auto-handoff skipped because no parent exists."` Reason: handoff REASSIGNS an existing parent; it does not create one.
8. **Run Job 5.5 — existing-subtask check.** Only for rows that landed on `Create subtask` at step 5. If an open subtask of the same `work_type` already exists under the parent, flip the verb to `Reopen subtask` (see Job 5.5 for details). `Hand off parent task` rows skip Job 5.5 — they don't create or reuse subtasks.

If none of the five actions fits, the signal does not become a row. Apply the Output gating filter and route to the `Filtered signals` log (Job 11).

### Job 5a — Work-type classifier

Runs immediately after parent-task selection (when known) and before Job 5 verb-branch. Inputs: signal content (Gmail subject + body OR Orbit comment text OR new-task title), originating Orbit task title + description (from the Job 7 deep-read when available; otherwise from the workload snapshot), and any AI Notes from Job 4b cross-link.

Classifies to exactly one `work_type` token (greedy first-match in declaration order — earlier rules win when multiple fire):

| `work_type` | Token / keyword cues (case-insensitive substring match) |
|---|---|
| `AUDIT` | `audit`, `seo audit`, `ai audit`, `aeo audit`, `geo audit`, `ada audit`, `accessibility audit`, `visibility audit`, `performance audit` (audit beats SEO when both fire — Audit work routes to the same pool but is the more specific class) |
| `QUOTE` | `quote`, `quoting`, `estimate hours`, `quoted hours`, `cost estimation`, `change order`, `scope estimation`, `pricing` |
| `SEO` | `seo`, `keyword research`, `meta tags`, `meta descriptions`, `serp`, `backlinks`, `schema markup`, `sitemap` (when not audit-prefixed) |
| `DESIGN` | `figma`, `mockup`, `wireframe`, `design`, `visual`, `ui design`, `branding`, `logo`, `creative`, `style guide` |
| `CONTENT` | `copy`, `blog post`, `content writing`, `landing page copy`, `case study`, `whitepaper`, `email copy` |
| `BA` | `requirements gathering`, `scoping`, `business analysis`, `discovery doc`, `ba review`, `flow diagram` |
| `HTML_CSS` | `html`, `css`, `frontend`, `front-end`, `responsive`, `breakpoints`, `mega menu`, `hero image`, `navigation`, `landing page` (UI work without server logic), `markup`, `theme styling` |
| `PHP_BACKEND` | `php`, `backend`, `back-end`, `api`, `database`, `mysql`, `wordpress plugin`, `custom post type`, `acf field`, `server-side`, `data import`, `migration script`, `ssl`, `dns` (when paired with code work) |
| `QA` | `qa`, `testing`, `regression`, `bug`, `test case`, `verify`, `cross-browser test`, `device matrix`, `release checklist` |
| `OTHER` | none of the above fire |

Conflict resolution:

- Multiple cues from different work_types in the same signal — pick the work_type whose cue is closest to the verb / deliverable in the signal text (e.g. "audit the SEO landing page — also fix the broken hero image" → `AUDIT` wins because the verb "audit" attaches to the primary ask, even though "hero image" would have fired `HTML_CSS`).
- Mega-menu / page-structure work that touches BOTH front-end and back-end → default to `HTML_CSS` when the deliverable is the page structure / styling; `PHP_BACKEND` only when the deliverable is server-side logic (e.g. custom WP plugin, database hook). When genuinely ambiguous, surface `Uncertain:` in AI Notes and pick the more conservative classifier (`HTML_CSS` over `PHP_BACKEND` — frontend pod is the broader pool).
- The Mega Menu restructure + 4 new pages example from the user's screenshot classifies as `HTML_CSS` (page structure work), NOT `PHP_BACKEND` — the deliverable is markup + content + navigation, not server logic. The Quote on that same parent is `QUOTE` (separate row, separate verb path).

Output: `row.work_type` token. Always populated (never null — `OTHER` is the fallback).

### Job 5.5 — Existing-subtask check (idempotency for HTML_CSS / PHP_BACKEND / QA)

Runs only for rows that landed on `Create subtask` after Job 5 step 5. Skipped for `Hand off parent task`, `Flag`, `Create parent task`, and `Reopen subtask` rows (the last is the output of THIS job).

For each `Create subtask` survivor:

1. **Fetch parent's subtasks — issue a pre-batch here, before checking.** Before running the per-row check, issue `get_task_details(parent_task_id)` for **all `Create subtask` survivors as a parallel batch** (same 25-call-per-turn cap and chunking pattern as the Job 7 batch). This batch fires **now, in Job 5.5** — not in Job 7 — because `subtasks[]` must be available to decide the verb, and Job 7 runs only after Job 5 completes. Job 7's later pass over the same parent tasks will find these responses already in context and reuse them without re-issuing the calls. Each response's `subtasks[]` carries `id`, `title`, `assignee_id`, `assignee`, `status`, `is_completed`.
2. **Filter to open subtasks.** Drop entries where `is_completed == 1` OR `status ∈ { "Done", "Archive", "Closed" }`. Remaining set is candidate-for-reuse.
3. **Match by work_type.** For each candidate, re-run Job 5a on its `title` + (if available) description text to get the candidate's own work_type token. Keep candidates whose work_type matches `row.work_type`.
4. **Apply the match-strictness rule (work-type only).** Per locked decision, a work_type match is sufficient — no secondary keyword overlap required. If `row.work_type` is `OTHER`, no reuse is attempted (`OTHER` is too vague to dedupe).
5. **Branch on match count:**
    - **Zero matches:** stay at `Create subtask`. No change.
    - **Exactly one match:** flip to `Reopen subtask`. Set:
        - `row.existing_subtask_id` = matched subtask's `id`
        - `row.existing_subtask_title` = matched subtask's `title`
        - `row.last_dev_user_id` = last non-PM commenter on the matched subtask (see step 6).
        - `row.new_work_description` = matcher's plain-language summary of what the new signal adds (one short paragraph; this becomes the body of the comment Mode 2 will post).
    - **Multiple matches:** keep `Create subtask` and emit AI Note `Uncertain: parent #<parent_task_id> already has <N> open subtasks matching work_type <work_type> (#<id1>, #<id2>, ...). Please pick which to reopen or confirm a new subtask is intended.`
6. **Last-dev extraction.** Issue `get_task_details(existing_subtask_id)` — one call per matched subtask, fired here in Job 5.5. This is a **separate call** from Job 7's deep-read: Job 7 reads the parent task; this reads the matched subtask — different `task_id`. Use `get_task_details` not `list_task_comments` — the bundled comment tree is complete in a single call and avoids a redundant round-trip. Flatten and sort `comments[]` newest-first (§ `collectors/orbit.md` Comment ordering). Find the most recent comment whose `user_id != PM_user_id`. That commenter is `last_dev_user_id`. Fallback chain when no qualifying comment exists:
    - The subtask's current `assignee_id` if not PM.
    - The subtask's `created_by_id` if not PM.
    - If both fall back to PM (rare — pure PM-authored subtask history), emit AI Note `Uncertain: existing subtask #<id> has no non-PM activity history. Please pick the assignee.` and keep `Create subtask` (do not flip — there is no "last dev" to reassign to).
7. **Last-dev active check.** Look up `last_dev_user_id` in the `list_users` cache. If the user's `is_active == false` (or equivalent inactive flag), emit AI Note `Uncertain: last dev on existing subtask #<id> was <name> but they're no longer active in Orbit. Please pick a new assignee.` Keep verb at `Reopen subtask` but set `row.last_dev_user_id = null` — the executor will halt and surface the row for manual reassignment instead of auto-firing `update_task`.

Output deltas:

- `row.verb` may flip from `Create subtask` to `Reopen subtask`.
- New fields: `row.existing_subtask_id`, `row.existing_subtask_title`, `row.last_dev_user_id`, `row.new_work_description`.

#### Unactioned client signal → Create parent task (Possible Orbit miss path)

This path catches client emails that should have become an Orbit task but did not — whether the AM/PM never created the task, or the client delivered something we asked for and nobody acted. It feeds the `Create parent task` verb.

**Entry gate.** A Gmail signal qualifies when BOTH of these hold:

- **Unactioned.** After Job 4b Pass 1 the signal has an empty `context_signals[]` array (no Orbit signal linked as context) AND the PM-action filter (Job 5 PM-action detection) found no PM reply/comment on the thread or its project. The signal is genuinely sitting unhandled.
- **Sender is external.** The sender is NOT the running PM's canonical email or any alias (self-noise filter).

…AND **any one** of the three trigger sub-classes fires (read the collector's `awaiting_action_hint` block on the signal — `collectors/gmail.md`):

- **S1a — reply-to-our-ask + deliverable.** `awaiting_action_hint.is_reply_to_wliq_ask == true` (the client message is a reply on a thread where WLIQ previously asked for something — `thread_depth > 1` with a prior outbound WLIQ message in `pm_last_message_excerpt`/thread) AND `awaiting_action_hint.has_deliverable == true` (attachment present, or a link / "as requested / here is / attached" phrasing in the body).
- **S1b — delivery tokens.** `awaiting_action_hint.delivery_tokens` is non-empty: body OR subject contains delivery phrasing (`here is`, `here's`, `as requested`, `as discussed`, `attached`, `please find`, `you asked for`, `completed`, `done`, `sharing`, `for your review`).
- **S2 — issue / error / feature request (critical-language).** `awaiting_action_hint.issue_tokens` is non-empty: body OR subject contains at least one of `urgent`, `asap`, `today`, `eod`, `end of day`, `blocker`, `blocking`, `critical`, `escalation`, `please do`, `cannot wait`, `client is waiting`, `before tomorrow`, `bug`, `broken`, `not working`, `error`, `issue`, `can you add`, `feature request`, `please change` (case-insensitive substring match).

When the entry gate + at least one sub-class fire, run **project resolution** then **dedup**, and the outcome of THOSE — not the trigger — decides `Create parent task` vs `Flag`.

##### Project resolution — workload map first, then lazy Orbit search

1. **Map hit.** If `context_link.project_id_candidates` resolves to exactly one project in the relationship map, use it. (It need not have a PM-owned task — that requirement is dropped; the dedup step below handles "already covered".)
2. **Lazy Orbit search** — fires ONLY when the map yields zero candidates (low volume, mirrors the Job 4b Fathom lazy-fetch pattern):
   - **Resolve the client.** `list_clients(search_value=<sender company / name>)` — match by company name, or match the sender's email domain against the returned `website_link`. (`get_client_details(company_name=…)` is the fuzzy fallback; it returns the client's AM + `contact_people` emails + project *counts*, NOT the project list.)
   - **List the client's projects.** `list_projects(client_id=<matched client>)` — returns each project's `id`, `title`, `project_number`, `owner_id`, `account_manager_id`. (Or `list_projects(search_value=<project-name keywords from subject>)` when the subject names the project directly.)
   - **Disambiguate** by topic-keyword overlap between the email subject/body and each project `title`; pick the single best. If more than one survives with no clean winner → treat as **not found**.
   - **Found (any project owner).** Propose `Create parent task` on it regardless of whether the PM owns/AMs it. If the PM is not the project's owner or AM, append `ai_notes`: `Project resolved by search — you are not its owner/AM; confirm before approving.`
   - **Not found.** Emit a `Flag` row (PM-driven path below).
3. **`project_number`** for rendering comes from the search result (`list_projects`/`get_project_details`), cached per run. Never render the internal `id` (SKILL.md #22).

##### Dedup — "if not already" (topic-matching open task)

Before emitting `Create parent task` on a resolved `project_id`, run `get_project_task_list(project_id, search=<topic keywords>, is_completed="incomplete")`. Note `search` is a `task_title` substring filter, so ALSO pull the project's open top-level tasks (`parent_id == 0`) and compare their `task_title` to the email topic semantically — don't rely on the substring filter alone.

- **Topic-matching open task found** → do NOT create. Emit a `Flag` with `ai_notes` prefix `Possible duplicate:` and `pm_next_step: "Possible existing task — #<id> '<title>' on <project> may already cover this. Reopen/comment there, or reply 'create anyway' to add a new one."`
- **No match** → proceed to the `Create parent task` emission below.

##### Create parent task emission

Emit a `Create parent task` row with:

- `recommended_action`: `Create parent task on <project> assigned to you`
- `task_title`: derived from the email subject. 6–12 words, verb-led where the subject allows; strip greeting prefixes ("Re:", "FW:") and corporate noise. Example: subject `"Re: URGENT — broken contact form on Agency X homepage"` becomes `task_title: "Investigate broken contact form on Agency X homepage"`. For a delivery (S1a/S1b), name the follow-up action, not the delivery — e.g. `"Process client-delivered brand assets for Agency X homepage"`.
- `proposed_orbit_body`: full 6-section body per `schemas/orbit-dq-standard.md`, plain language per `writers/plain-language.md`. The body cites the Gmail thread as the originating signal.
- `assignee_id`: PM_user_id (the parent lands on the PM's plate so the PM can subsequently spawn sub-tasks under it the normal way).
- `parent_task_id`: `null` (this row CREATES the parent — no parent to nest under).
- `project_id`: the single resolved project (map or search).
- `ai_notes`: prefixed with `Possible Orbit miss:` — name the sub-class and source, e.g. `Possible Orbit miss: client delivery from jane@agencyx.com (reply to our request) with no corroborating Orbit task. Creating parent task on Agency X on your approval.` Append the not-owner note from project resolution if it applies.
- `pm_next_step`: omitted entirely (this is not a Flag row).

**Not-found / duplicate Flag rows are resolved by a PM Note in Mode 2** — `synthesis/note-interpreter.md` § "create it on <project> / create anyway" turns the PM's reply into a `create_parent_task` action.

**Why a parent and not a sub-task.** The PM-owned parent does not yet exist on the project — that's the whole point of "Possible Orbit miss". Creating a sub-task requires a parent; this signal class creates the parent itself. Future sub-tasks for the same engagement nest under this parent the normal way (via the standard `Create subtask` path in later morning runs).

**Why PM is the parent assignee.** The parent is the PM's coordination anchor on the project (per Output gating semantics). Even when the work itself is dev-shaped, the parent stays on PM; the sub-task that does the work goes to the dev later.

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
  - **`Create parent task` rows**: no Orbit task exists at row-create time (the parent will be created by Mode 2). Set `orbit_task_link = "—"`. After Mode 2 executes, the created parent's URL lands in `Outcome` only — `orbit_task_link` stays frozen at em-dash to indicate "this row created the task; see Outcome for the URL".
  - **Null `task_url` on a signal**: cross-check the Orbit collector's relationship map (every task in the universe is indexed by `task_id` with its URL). If the URL is still missing there, set `orbit_task_link = "—"` and append an AI Note: `Orbit task URL not surfaced by MCP on this signal — open via task ID #<id>.` Do not block the row.
- **`project`** — render-ready string for the Notion `Project` column. Format: `<project_title> (#<project_number>)` where `project_number` is the user-visible Orbit project code (string field from `get_project_details.project_number`, e.g. `"16915"`), NEVER the internal `id` field (per SKILL.md non-negotiable rule #22). For Maintenance / Ad-hoc projects, append `— Maintenance` / `— Ad-hoc` (read project_type from the Orbit relationship map). For rows with no resolved Orbit project (Gmail-only flag with no project mapping), emit `Standalone`. Preserve `project_name`, `project_number` (user-visible string), and `orbit_project_id` (internal int — used only inside URLs and the relationship map) separately for downstream consumers.

No Mode 2 step writes into `orbit_task_link` later — the column is frozen at row-create time and stays as the parent reference. Sub-task URLs (created by Mode 2) go into `Outcome` only, per `executors/orbit.md` Outcome format.

### Job 6 — Recommend the assignee

**Short-circuit for `Create parent task` rows.** Job 5 already set `assignee_id = PM_user_id` for these rows (the parent is the PM's coordination anchor). Skip pod-inference entirely. Render `recommended_assignee` as `You (PM)` with reason `Parent task assigned to you so you can spawn sub-tasks in later runs.` No availability check, no candidate pool, no Job-6 decision tree.

**Short-circuit for `Reopen subtask` rows.** Job 5.5 already resolved `row.last_dev_user_id`. Skip pod-inference. Render `recommended_assignee` as `<last_dev name> (<role>) — back on the existing subtask #<existing_subtask_id> they last worked.` If `last_dev_user_id` is null (Job 5.5 flagged inactive-dev or no-non-PM-history edge cases), render `— (please pick — see AI Notes)` and let the PM resolve.

**Pod leader routing for `Hand off parent task` rows.** Job 5a's `work_type` (`AUDIT | QUOTE | SEO | DESIGN | CONTENT | BA`) maps to a functional pool (table below). Read the pool from `pod-inference.md` output's `functional_pools[<pool>]` and pick the named lead per the pool's first row (or fall back to the highest-familiarity member when no explicit lead row is tagged). Render `recommended_assignee` as `<leader name> (<pool> lead) — handoff target for <work_type> work.` If the functional_pool is empty (matrix unavailable, or pool truly empty), drop `Uncertain:` AI Note and surface candidates from the broader functional_pools for PM pick.

**Named-lead exception (Marketing/SEO + Design) — Hand off rows only.** For `Hand off parent task` rows, two named leads override the generic pool-leader lookup above. This is a deliberate, explicit hardcoding (per PM directive — supersedes the no-named-person rule for these two leads only):

| `work_type` on the Hand off row | Recommended assignee (hardcoded) |
|---|---|
| `DESIGN` | **Jay Panchal** (Design lead) |
| `AUDIT`, `SEO`, `CONTENT` (Marketing/SEO matrix pool) | **Manan Rana** (Marketing/SEO lead) |

Render `recommended_assignee` as `Manan Rana (Marketing/SEO lead) — handoff target for <work_type> work.` or `Jay Panchal (Design lead) — handoff target for design work.` accordingly. `DESIGN` always resolves to Jay Panchal even though design sometimes sits inside the broader Marketing/SEO org — the design carve-out wins over the Marketing/SEO default. This exception is **independent of Pod Matrix state** — it fires even when the matrix is unavailable or the functional_pool came back empty, so these two handoffs never fall to `Uncertain:` for lack of a resolvable lead. `QUOTE` and `BA` are NOT covered by this exception — they continue to use the generic functional-pool-leader lookup (Quoting lead, BA lead). Only `Hand off parent task` rows are affected; `Create subtask` routing (HTML/PHP/QA pods) is untouched.

For all other rows (`Create subtask`, `Flag`), continue with the 4-branch decision tree below.

Call `synthesis/pod-inference.md` with the item's project ID. It returns the candidate pool — matrix members ∪ Orbit followers/recent-assignees — with role hints, familiarity scores, `has_history_on_project` booleans, matrix membership flags, plus the `floater_pool` and `functional_pools` for fallback paths.

Pick the single best assignee using a 4-branch decision tree. Familiarity wins by default; availability is checked only on the no-history fallback path (per `SKILL.md` non-negotiable rule #6).

#### Work-type → pod-boundary mapping (replaces the old loose role-fit heuristic)

The candidate filter for `Create subtask` rows is driven by `row.work_type` (Job 5a output), NOT by keyword scan over the task title. Strict per-pod boundaries:

| `row.work_type` | Eligible candidate roles | Notes |
|---|---|---|
| `HTML_CSS` | `FE` only (matrix `HTML` row + Floater HTML role) | Never WP/PHP. HTML/CSS work is FE pod's exclusive territory. |
| `PHP_BACKEND` | `WP` (matrix `WordPress / PHP`) + Orbit-history `BE` | Backend / PHP logic only. Audits are NOT routed here — they go to AUDIT pool via the `Hand off parent task` verb. |
| `QA` | `QA` matrix | |
| `OTHER` | falls back to keyword heuristic (legacy 4-branch) | Used only when Job 5a couldn't classify; treats every role as eligible. |

`AUDIT`, `QUOTE`, `SEO`, `DESIGN`, `CONTENT`, `BA` never reach this filter — Job 5 sent them down the `Hand off parent task` path which uses the functional-pool-leader routing above.

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

### Job 7 — Generate the proposed Orbit body (Create subtask + Create parent task paths)

The body content depends on the row's action.

#### Mandatory deep-read of the originating Orbit task (applies to every Create-subtask AND Create-parent-task row)

**Always read the Orbit task in full — not just the workload snapshot.** Before composing the 6-section body, the matcher MUST fully understand the originating Orbit task. The workload response from Step 1e returns a static snapshot (title, project, due_date, status, severity, parent_id) with **`description: null`** and `get_activity_log` returns deltas since `last_run_timestamp` — but **the full description, the complete comment history, and the subtask list are NOT in those responses**. Compose the brief blind to that prior history and the delivery team gets a brief that ignores 6 months of prior conversation on the task.

For every task that becomes a Create-subtask or Create-parent-task row, the matcher MUST call:

1. **`get_task_details(task_id)`** — the single **super-call**. One response returns the full task `description`, the **complete threaded comment tree** (`comments[]` with nested `replies[]`, all-time — NOT date-filtered to `last_run_timestamp`), `subtasks[]`, `followers[]`, and `task_attachments[]` (each with a pre-extracted `summary`). Default-on, NOT conditional on whether the workload returned a description (it never does). The bundled comments hold the load-bearing deep history (decisions, prior client feedback, AM clarifications, failed attempts, scope changes) that the Step 1e activity-log delta does not surface. **Read the comment tree by flattening `replies[]` recursively, then ordering newest-first** (see `collectors/orbit.md` § Comment ordering) — the newest activity may be a nested reply. **Strip HTML to plain text** before reasoning over the body and comments (they arrive as HTML).
2. **`list_task_comments(task_id)` — fallback only.** `get_task_details` already bundles the full comment tree, so do NOT call this by default (it would re-fetch the same comments, doubling comment tokens in context). Call it for a row only when step 1's `comments[]` looks paginated / capped on a very long history — then page back through the all-time history.

The deep-read fires during Job 7 composition (per-row, on the survivor set after Job 5 — typically 10-30 rows, not the full 30-80 workload size). It is mandatory for the row classes that produce an Orbit body. **Flag rows skip the deep-read — with ONE exception: `pm_task_due_today` bare Flags fire a *light* deep-read** (`get_task_details` only, no body). See § Light deep-read for due-today Flags below.

#### Light deep-read for due-today Flags

A bare `pm_task_due_today` Flag (a task due today with no other signal) does NOT compose a 6-section body, so it does not need the full all-time comment history. But under the single-pane mandate the PM should be able to decide what to do with it **without opening Orbit** — which requires knowing where the task currently stands. So this Flag class fires a trimmed **single-call** read:

1. **`get_task_details(task_id)`** — current body / status / severity / `subtasks[]` + the bundled comment tree. The same super-call as the full deep-read; for a light read just take the **newest** comments off the flattened-and-sorted tree (§ Comment ordering) — enough to answer "what's the latest on this task?" without paging the full history. **No `list_task_comments` fallback for the light read** (the bundled newest comments suffice; a bare due-today Flag never needs deep history).

Outputs feed exactly two things: (a) the Job 7b `task_brief` ("here's where this stands" — latest comment + open-subtask status), and (b) the open-subtask annotation in AI Notes. **No 6-section body, no assignee recommendation (Job 6 is skipped — `recommended_assignee = "— (PM action)"`), no handoff (Job 8 skipped).** If the call fails after retries, the row still ships as a Flag with the bare `orbit_due_today` anchor and AI Note `Due today — could not load task context this morning; open in Orbit to review.` — the deadline visibility is never lost.

**Issuance — parallel batch, not serial per-row.** After Job 5 filtering, collect the survivor set `S = { rows where action ∈ {Create subtask, Create parent task} } ∪ { pm_task_due_today bare Flags }`. Every row in S gets a **single** `get_task_details(task_id)` call (the full classes and the due-today light read use the same one call — they differ only in how much of the bundled comment tree they read, not in the call). Then fan out in a single LLM-turn batch:

- Issue `get_task_details(task_id)` for EVERY row in S as parallel tool calls in one turn (Claude Code supports multi-tool-use per turn).
- **Batch cap: 25 parallel tool calls per turn.** With one call per row, a batch covers up to 25 rows; if `|S| > 25`, chunk the survivor set (rows 1–25, 26–50, …). Chunk boundaries are arbitrary — order within S does not affect Job 7 composition since each row composes independently.
- Any `list_task_comments(task_id)` **fallback** (only for rows whose bundled `comments[]` looked truncated) issues in a follow-up batch after the `get_task_details` results land.
- Wait for all batches to land before starting Job 7 body composition. Composition is row-by-row but ALL row inputs are gathered up-front via parallel issuance.

**Wall-clock impact.** Serial issuance: |S| calls × MCP latency (~200ms each) = ~4s for S=20 rows. Parallel issuance: one batch of ≤25 calls = ~250ms + batch overhead. Halving the per-row call count (was two calls/row) also halves the deep-read tokens that land in context.

**Per-row failure semantics.** Each row's deep-read is independent; a failure on row X does not affect rows Y, Z, …

- **`get_task_details(X)` succeeds:** row X composes normally with full data (description + threaded comments + subtasks + attachments).
- **`get_task_details(X)` succeeds but its `comments[]` looked truncated, and the `list_task_comments(X)` fallback then fails after retries:** compose row X's body from `get_task_details` (description + the partial comment tree it did return) + the activity_log delta from Step 1e + Gmail/Fathom enrichment. Append `Uncertain: full comment history unavailable for this row — proposed body may miss prior decisions older than the returned comments.` to row X's AI Notes. Row still ships; PM sees the caveat.
- **`get_task_details(X)` fails after retries:** mark row X's Outcome with `FAILED — deep-read incomplete for this row; manual review needed before approval. Mode 2 will skip execution.` Row stays at Status `Recommended Action` and is excluded from Mode 2 execution dispatch even if PM marks Approved (Mode 2 reads the FAILED-deep-read flag and skips with a warning Outcome). The matcher does NOT attempt to compose a body from the workload snapshot alone (it has no description) — too much risk of confidently-wrong content.

Pull every fact from the full task + comment history that's relevant to the proposed sub-task — prior decisions, named POCs already involved, failed attempts to avoid, scope clarifications, dependencies, deadlines mentioned in earlier comments. Weave them into the 6-section body per the per-section mapping below.

#### Mandatory email-thread enrichment (applies to every Create-subtask AND Create-parent-task row)

**Always check email — not just when the Orbit brief looks thin.** In addition to the deep-read of the originating Orbit task above, before composing the 6-section body the matcher MUST inspect every Gmail signal attached to this Orbit signal via `context_signals[]` (Pass 1) and every Fathom enrichment attached via `signal.enrichment.fathom` (Pass 2). This is unconditional — it does not depend on whether the originating Orbit task body looks complete. Emails routinely carry nuance, prior decisions, scope tweaks, client constraints, and deadlines that never made it into the formal Orbit task; the proposed sub-task body is the place to surface them so the delivery team sees the whole picture in one place.

For each input source — (a) the originating Orbit task's full details + complete comment history (from the per-row deep-read above), (b) each attached `context_signal` (Gmail thread end-to-end), (c) each attached `enrichment.fathom` (meeting summary + action items + recording URL):

1. **Read every input in full, not just the metadata.** The Orbit deep-read returned the task body + all comments; the Gmail collector pulled every linked thread end-to-end (`collectors/gmail.md` § Full thread context); Fathom Pass 2 returned a scoped meeting summary. Long content (10+ messages in a thread, dozens of comments on a long-running task, multi-day discussions) is NOT a reason to skip — it is the reason to read carefully because the load-bearing context usually lives there.
2. **Extract every fact relevant to the sub-task** — proposed deliverable, named stakeholders, dates, blockers, prior decisions, file references, AM clarifications, client questions, sign-offs, prior failed attempts, scope changes.
3. **Weave the extracted facts into the 6-section body** per the mapping below. All three input sources (Orbit task + comments, Gmail threads, Fathom enrichment) feed the same body sections — the mapping is source-agnostic:
   - **DO** — if any input source defines the deliverable more precisely than the Orbit task title, use the more precise phrasing in DO and note the divergence in AI Notes.
   - **WHY** — pull motivation from AM/client wording across all sources. "Sarah said the board demo is Thursday, that's why this is today" reads better than a generic "client urgency".
   - **CONTEXT** — surface prior decisions, prior rounds, project phase, named POCs, dependencies. This is where long-thread context AND long-comment-history context land most often. Comments older than `last_run_timestamp` (not in the activity_log delta) often hold the load-bearing decisions.
   - **DONE WHEN** — if any input lists acceptance criteria ("must work on mobile", "include the legal disclaimer", "preview link to Jane"), they go here verbatim (in plain language).
   - **SELF-QA** — role-specific items per `schemas/orbit-dq-standard.md`, plus any explicit checks any input called out ("test on Safari iOS specifically").
   - **REFS** — cite EVERY context source: the Orbit parent task URL, the Gmail thread URL(s), the Fathom recording URL (if any), and any document URLs referenced anywhere (Drive / Docs / Sheets / SharePoint per `references/external-doc-access.md`).
4. **Conflict resolution.** When sources conflict (e.g., email says one deadline, Orbit task description says another), prefer the most recent authoritative source (latest AM message, latest client decision, latest task comment). Note the conflict in `ai_notes` with `Uncertain:` prefix if the resolution is unclear — let the PM disambiguate.
5. **Do not silently truncate.** If a thread or comment history is too long to summarize fully, capture the key facts in the body sections and add a one-line pointer in REFS: `Full thread (N messages) at <gmail thread URL>` or `Full comment history (M comments) on <orbit task URL>`.

This rule applies to **priority-lane rows especially**: an AM-handed parent task created overnight may have only a one-line title in Orbit ("Conversant Phase 2 Sprint 12 dev planning"), but the thread between the AM and PM about it almost always carries the actual scope, the named delegate the AM has in mind, the deadline reasoning, and the constraints. Without email enrichment the proposed sub-task body would be uselessly thin; with email enrichment it gives the delegate everything they need to start without asking.

#### Latest-signal anchor (applies to EVERY row, including Flag rows)

After all deep-reads land, Job 7 selects `row.latest_signal_anchor` as the newest signal across the row's input sources. The collectors already do the heavy lifting — each task carries `latest_signal` (`collectors/orbit.md`), each Gmail signal carries `latest_signal` (`collectors/gmail.md`), each attached Fathom enrichment carries `meeting_date`. Job 7 picks the newest of these by `timestamp_iso` and copies it through — no recomputation. Tiebreaker: prefer the source that carries the "what's new" content the matcher used when composing the row.

```
row.latest_signal_anchor = {
  source: "orbit_comment" | "orbit_task_body_update" | "orbit_status_change" | "orbit_due_date_change" | "orbit_due_today" | "gmail_message" | "fathom_meeting",
  id: <int or string>,
  timestamp_iso: <ISO 8601>,
  author_name: <string>,
  excerpt: <string ≤ 240 chars, plain text>,
  source_url: <string — link to the originating item>
}
```

If no input source carries a signal (rare — a row with no trigger), drop to `filtered_signals` with `filter_reason: no_trigger_signal_identifiable` rather than ship with a null anchor. The writer's render needs this field; a row without it cannot render its Triggered-by line. Applies to every verb — Flag, Reopen subtask, Hand off parent task, Create parent task — same contract, no exceptions.

#### Path-specific body rules

**Create subtask path** — pre-write the full 6-section task body per `schemas/orbit-dq-standard.md`, applying the mandatory enrichment above. Plain language (4th–5th grade English) per `writers/plain-language.md` since the delivery team reads it. Keep role-specific technical terms. Strip corporate English. The 6-section body lands in Orbit as the sub-task description when Mode 2 fires.

**Reopen subtask path** — does NOT pre-write a new 6-section body (the existing subtask already has one). Instead, compose `row.new_work_description` as a plain-language comment that will be posted on the existing subtask by Mode 2: 2–4 sentences naming what the new signal adds (new scope, new constraint, new client feedback, new deadline) plus the source citation (originating Gmail thread URL or Orbit comment URL). Header line `Reopened — new work from <signal date>:`. Body inherits the same plain-language enforcement as the team handoff draft.

**Hand off parent task path** — does NOT pre-write a new task body either; the existing parent already has one. Compose `row.new_work_description` as a brief paragraph the PM forwards to the pool leader explaining what's being handed off (the scope summary + the relevant client / AM context + the parent task URL). This text lives in the row's `Proposed Handoff` section. The executor's Orbit write is reassignment only — no new task body, no new comment by default (though the executor MAY post a brief "@<leader> handed to you per PM's morning queue" comment for audit trail — see `executors/orbit.md` Hand off path).

**Create parent task path** — pre-write the full 6-section task body per `schemas/orbit-dq-standard.md`. Same plain-language rules and same mandatory email-enrichment rules as Create subtask. The body cites the originating Gmail signal explicitly in the REFS section (sender, subject, thread URL) so the audit trail is clear when the PM (or someone reviewing later) opens the Orbit task. The originating Gmail thread for a Possible-Orbit-miss row IS the primary source — read its full depth.

**Flag path** — does NOT carry a `proposed_orbit_body`. No Orbit write happens for a Flag row. Job 7 body composition is skipped for Flag rows entirely (and Job 6 assignee + Job 8 handoff too). The row's detail page substitutes `Proposed Orbit Task Body` with `PM next step` (rendered from the `pm_next_step` clause set in Job 5). The row detail Sources section still renders all attached `context_signals[]` and `enrichment.fathom` so the PM can scan the email/meeting context while deciding their next move. **Exception — `pm_task_due_today` bare Flags:** Job 7's *light* deep-read still fires (details + newest comments) to feed the Job 7b `task_brief` and the open-subtask annotation; body composition, assignee, and handoff remain skipped. See § Light deep-read for due-today Flags.

### Job 7b — Compose the row's Task Brief

Every row in the queue — regardless of verb — carries a `task_brief` field that renders below the H1 Summary heading on the row detail page (above the H1 Sources heading, per `schemas/row-detail-page.md`). The brief is a 2–4 sentence narrative answering two questions: (a) what is this work about, (b) what's the new update / latest signal that triggered this row.

Composition rules:

- **Pull from the most recent input source first.** For Orbit-priority signals, that's the AM clarification / handoff context (from the parent task's `parent_task_body` and the most recent activity_log entry). For Gmail-driven signals, that's the most recent message in the thread. For `Reopen subtask` rows, that's the new signal that triggered the reopen (NOT the old subtask description — that lives in the existing Orbit body already).
- **Surface deltas, not all-time context.** The brief is for the PM to grasp what's NEW. The deeper history (prior comments, older email messages, full Fathom transcript) lives in the 6-section Orbit body (`Create subtask` / `Create parent task` paths) and the Sources block (every row). The brief stays tight.
- **Plain professional English.** The PM reads this — not the delivery team. No simplification pass needed.
- **Anchor on the deliverable when possible.** Lead with one sentence describing what the work IS (mirrors the Summary topic), then 1–3 sentences with the new signal's content.
- **Cite the trigger inline.** When a specific message / comment is the trigger, identify it briefly (e.g. "Pravin commented on the parent at 12:40 IST today, confirming the 4 new pages should mirror the existing parent/child pattern.").
- **For `Flag` rows:** the brief becomes a 2–3 sentence "here's what came in" summary of the signal the PM needs to react to.
- **For `pm_task_due_today` bare Flags:** the brief is a 2–3 sentence "here's where this stands" status, composed from the light deep-read (most-recent comment + open-subtask status from `subtasks[]`), NOT the bare due-date. Lead with what the task is, then the latest movement, then the open-subtask hint so the PM can triage in Notion. Example: `Homepage revisions round 2, due today. Latest: Vijay pushed the staging build 2 days ago, awaiting your QA sign-off. Subtask #112880 open under Vijay — likely needs your review, not re-delegation.` If the light read failed, fall back to `<task title>, due today. Could not load latest context this morning — open in Orbit to review.`
- **For `Hand off parent task` rows:** the brief explains why this work is being handed off to the pool leader rather than kept in-pod (e.g. "This is a quote / SEO audit / design ask — handed to the Quoting / Marketing-SEO / Design pool lead per the work-type routing rules.").
- **Length cap: 600 chars.** Notion truncates aggressively in row preview; the brief is a quick read on the detail page, not a placeholder for the Orbit body.

The writer (`writers/notion.md`) renders `task_brief` as a paragraph under the H1 `Task Brief` heading.

### Job 8 — Generate the proposed handoff

The handoff message is the team handoff draft the PM copies-and-sends after Mode 2 fires. It lands on the row's Notion detail page under the `Proposed Handoff` H1 section and is also appended to the row's `Outcome` block when Mode 2 executes. The PM delivers it through whatever channel they use for that team member (direct message, in-person, email, etc.). Generate it for the **Create subtask** path only:

- Tell the assignee a new sub-task landed under the PM's parent task, the work in one short paragraph, the parent context, and the sub-task URL.

Plain language per `writers/plain-language.md`. Reminder to log hours if Preferences has that always-include rule.

For priority-lane rows specifically, the handoff body should additionally note that the parent task was handed down by the AM: `<AM name> assigned this task to <PM name> overnight; passing the dev work to you to start today.` This sets the assignee's expectation that the timeline is same-day.

Flag rows do NOT generate a handoff (no dev work to delegate; PM owns the next move).

Create parent task rows do NOT generate a handoff either — the parent assignee is the PM, so there is no delegate to brief. The PM, once the parent exists, will spawn sub-tasks (and handoffs) in subsequent runs via the standard Create subtask path.

NO handoff is generated for AMs or clients — those are PM-handled outside the queue per the Output gating filter.

### Job 9 — Generate the proposed email

**Skipped.** Email drafting is outside the queue's scope under the 5-verb gating rule. Client and AM emails are PM-handled in Gmail directly. No row produces an email artifact.

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
