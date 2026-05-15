> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

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

Call `synthesis/pod-inference.md` with the item's project ID. It returns the candidate pool for that project with role hints and task history.

From the candidates, pick the single best one by ROLE FIT ONLY. No availability check.

Role-fit heuristics:
- If the task is FE / WordPress-themed → pick someone with FE or WordPress history on the project
- If it's BE / API / database → pick someone with BE history
- If it's QA → pick the QA person
- If it's design → pick the designer
- If it's project coordination / AM-liaison → suggest the PM themselves
- If no clear role fit, leave `recommended_assignee = null` and flag AI Notes: `Uncertain: I can't tell who's the best person for this. Please drop a note.`

Write the assignee as `Name (Role) — short reason`. Example: `Vijay Patel (FE) — primary FE on Agency X, 24 hrs on current homepage task`.

If the PM knows better and wants to reassign outside the pod, their note will override (PM note always wins).

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

- Does not check availability or capacity.
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
