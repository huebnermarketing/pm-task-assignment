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

### Job 4 — Generate the one-line summary

Normal professional English. This is what the PM reads when scanning the queue.

Pattern: `[Client or project] — [what happened / what's needed]. [Optional second clause with context].`

Examples:
- `Agency X — homepage revision feedback from Jane, 12 revisions, client presenting to board next Thursday.`
- `DigitalFirst — 3 action items from yesterday's Q3 call need Orbit tasks.`
- `Priya Sharma is out sick today; 3 tasks due this week need reassignment.`
- `CloudBase — Ravi finished the API integration. Ready for QA.`
- `BrightPath — revised proposal due Friday, client flagged they need numbers before committing.`

Rules:
- Professional English for the PM. No plain-English simplification here (the PM is an adult US-trained English speaker).
- Plain-language simplification kicks in later, only for delivery-team-facing outputs per `writers/plain-language.md`.
- Keep to one line when possible; max two.
- No emojis in the summary (keeps the queue scannable).
- Include the client or project name at the start for at-a-glance grouping.

### Job 5 — Recommend the action

For each item, pick the most likely action from the PM Action set:

- **Create Task** — new work from an email, Slack, or Fathom action item that doesn't exist in Orbit yet.
- **Reassign** — existing task needs to be moved to a different assignee.
- **Create Project + Task** — new project from an agency/AM intake, then first task.
- **Status Update** — task is stuck, needs status change (e.g., move to In Review).
- **Draft Email** — response to a client/AM needed, best composed in Gmail first.
- **Slack Handoff** — team member needs context for work already in Orbit.
- **Add Orbit Comment** — record decision or progress note without creating new work.
- **Approve + Notify** — approve something (asset, scope, direction) and notify downstream.
- **Defer or Flag** — nothing clearly actionable; PM decides next step.

One item typically has one primary action. Secondary actions (notify AM, CC Nishant, etc.) go in the recommended_action phrase as "+ Slack Caitlin" or similar.

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

### Job 7 — Generate the proposed Orbit task body

For items that will result in a new Orbit task, pre-write the 6-section task body per `schemas/orbit-dq-standard.md`.

Write this version in plain language (4th–5th grade English) per `writers/plain-language.md` — because the assignee (delivery team) will read it.

Keep role-specific technical terms. Strip corporate English.

### Job 8 — Generate the proposed Slack handoff

For items that will result in a Slack message to a team member, pre-write the handoff in plain language.

Include:
- Clear first-line summary of what needs doing
- Why it matters (1 sentence)
- Where to start (file, Orbit task link, etc.)
- Any specific prep (watch Fathom recording, read attached PDF)
- Reminder to log hours (if Preferences has that always-include rule)

### Job 9 — Generate the proposed email

Only when the action involves emailing a client or AM. Use normal professional English (the recipient is US-based; plain-English rule does not apply).

Draft includes:
- Appropriate subject line
- Greeting
- Context + specific ask or info
- Closing + signature from Preferences

### Job 10 — Write AI Notes

Include only things worth the PM knowing:
- `Uncertain:` flags (always)
- Why a PM note was honored differently than the recommendation
- When a single signal was split into multiple items
- When the recommended assignee isn't obvious
- When a collector failed and this item has partial context

Leave empty if nothing notable.

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
