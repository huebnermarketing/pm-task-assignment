---
name: pm-task-assignment
description: Automates a WLIQ Project Manager's pre-office morning hour. Collects overnight signals from Orbit, Gmail, Slack, and Fathom; writes a plain-language morning queue to a Notion page; executes internal task assignments and reassignments in Orbit (plus Slack drafts appended to Notion and Gmail drafts) after the PM approves. Runs as 3 separate Claude Routines (Mode 1 morning collection, Mode 2 execution, monthly archival) on cron; same skill also runs interactively in Claude Desktop or programmatically via Claude Code / SDK (see `ENVIRONMENTS.md`). Manual out-of-routine commands also accepted — e.g. "PM Task Assignment, run morning", "PM Task Assignment, run execution now", "PM Task Assignment, validate setup" — documented in `invocation-commands.md`. Lenient about phrasing — variations like "skill PM task assignment, run my morning" or "PM task assignment run morning" all route correctly.
---

# PM Task Assignment

> **MANDATORY FIRST STEP on every invocation, scheduled or manual: load `config.md`, then run the preflight sequence in `preflight.md`. Do not proceed to mode logic, collectors, executors, or any tool call until preflight has completed successfully. This rule is non-negotiable and applies to every code path that reaches this skill.**

## Source allowlist — read this before touching any tool

This skill uses these MCPs as **primary sources**: **Orbit, Gmail, Slack, Fathom, Notion**.

It uses these MCPs as **on-demand read-only references** when an allowed primary signal links to a file there: **Google Drive, Google Docs, Google Sheets, SharePoint**. Read-only. Never write. Trigger condition: a primary signal (Gmail / Slack / Fathom / Orbit / Notion) contains a link or attachment pointing at the document; the skill MAY fetch the doc's content for context. See `references/external-doc-access.md`.

Any other MCP is forbidden — including any added to the user's Cowork after installation, including any the running user explicitly asks the skill to use.

This rule applies even under experimental scope, forced runs, sandbox runs, or any kind of override. If a signal seems relevant from a forbidden source, ignore it.

The clause is repeated in `config.md`, in `preflight.md`, and at the top of every collector / executor file. The repetition is deliberate.

## What this skill does

Eliminates the one hour WLIQ Project Managers currently spend from home every morning before the India office opens at 11:00 AM. The PM currently scans Gmail, Slack, Fathom, and Orbit for overnight signals, figures out which project each signal belongs to, decides who should pick up each piece of work, writes Orbit tasks with enough context for the assignee to start, then walks desk to desk in the office to brief each team member.

This skill replaces all of that except the PM's approval moment. Each PM runs the skill on their own Claude account. Their MCP connections (Orbit, Gmail, Slack, Fathom, Notion) provide the data. The skill writes the morning's work to a Notion page identified in `config.md`. The PM reviews, approves, and the skill executes.

**Execution model.** This skill runs under two surfaces: (1) headless — Claude Routines on cron, or programmatic Claude Code / SDK invocations — never asks questions; (2) interactive — a PM types a manual command in Claude Desktop — may ask one clarifying question per ambiguous item before falling back to `Uncertain:`. Both surfaces share the same preflight, allowlist, writes, plain-language rules, and run-log behavior. The only difference is question-asking. See `ENVIRONMENTS.md` for the surface-detection contract and what is shared. See `ROUTINE-ENTRYPOINTS.md` for the three baked routine prompts; `invocation-commands.md` for the interactive commands.

## The two modes

**Mode 1 — Scheduled morning run.** Fires automatically at the time configured in Preferences (default 9:30 AM IST). After preflight, pulls overnight signals. Synthesizes them into items. Writes today's dated sub-page at the top of the PM's Notion parent (the one identified in `config.md`). No PM input required during this run.

**Mode 2 — Scheduled execution run.** Fires automatically at the configured execution time (default 10:45 AM IST). After preflight, reads the page-level `Ready for Execution` toggle at the top of today's dated page.

- If **ON**, executes every row the PM marked `Approved` and every row with a PM note. Writes `Done` outcomes back. Slacks the PM a completion summary.
- If **OFF**, runs the inline escalation flow (Mode 2 Step 3a) — Slacks (or emails) the backup person configured in Preferences and exits. Escalation is a fallback branch of Mode 2, not a separate mode.

## Invocation commands (manual overrides)

**These commands are for out-of-routine manual use only.** Routines do not parse user messages — they fire by cron with a baked prompt that loads this skill from the repo. See `ROUTINE-ENTRYPOINTS.md` for the 3 routine prompts. Manual commands below are used only when an operator runs the skill in a regular Claude session for catch-up or preference edits.

The PM types one of these in Claude when they want to override the schedule or edit configuration. The skill is **lenient** about phrasing — case-insensitive, comma-optional, "skill" prefix optional, "my" or "the" qualifiers fine. Any reasonable variation routes correctly.

- `PM Task Assignment, run morning` — manually fire Mode 1 (scheduled run missed, or on-demand morning read)
- `PM Task Assignment, run execution now` — manually fire Mode 2 (PM flipped Ready after the scheduled time passed)
- `PM Task Assignment, change my preference: [instruction]` — edit the Preferences page only; does not fire the morning flow
- `PM Task Assignment, update tone samples` — add or replace tone calibration samples for the plain-language writer

Variants Claude must accept and route correctly:

| Variant | Routes to |
|---|---|
| `PM Task Assignment, run morning` | Mode 1 |
| `pm task assignment, run morning` | Mode 1 |
| `Skill PM Task Assignment, run morning` | Mode 1 |
| `PM Task Assignment run morning` (no comma) | Mode 1 |
| `PM Task Assignment, run my morning` | Mode 1 |
| `PM Task Assignment, run the morning` | Mode 1 |
| `PM Task Assignment, run execution` | Mode 2 |
| `PM Task Assignment, run execution now` | Mode 2 |
| `PM Task Assignment, execute` | Mode 2 |
| `PM Task Assignment, change my preference: ...` | Preference edit |
| `pm task assignment change preference: ...` | Preference edit |

Internal logic for routing: any message that contains the skill name + an action verb (run, execute, change, update) + a context word (morning, execution, preference, tone) routes correctly. See `invocation-commands.md` for full routing rules.

## First run

The skill detects first run during preflight (step 3) by checking whether a `Preferences` sub-page exists under the Notion parent identified in `config.md`. If no Preferences page exists, the skill runs the first-run setup flow conversationally (see `first-run-setup.md`) and asks 9 setup questions.

Once Preferences is populated, every subsequent run reads it silently and proceeds.

## Architecture overview — what calls what

```
EVERY ENTRY POINT (scheduled or manual):
  → Load config.md
  → Run preflight.md (5 steps)
  → If preflight succeeds → continue to mode/command logic
  → If preflight fails → connector-failure-notify.md or first-run-setup.md or abort

Mode 1 (scheduled collection):
  → preflight
  → Read Preferences (already loaded by preflight)
  → Call collectors in parallel:
    ├─ collectors/orbit.md
    ├─ collectors/gmail.md
    ├─ collectors/slack.md
    └─ collectors/fathom.md
  → synthesis/matcher.md — group signals by project
  → synthesis/pod-inference.md — compute pod per project
  → recommendation logic (inside matcher) — pick assignee by ROLE FIT only (no availability check)
  → writers/notion.md — write today's dated sub-page with inline database + Ready toggle at top
    → Enforces parent-page structure per schemas/parent-page.md
    → Enforces Morning Queue schema per schemas/morning-queue-database.md
    → Enforces row detail layout per schemas/row-detail-page.md
  → writers/run-log.md — append run entry to Run Log database

Mode 2 (scheduled execution):
  → preflight
  → Read today's dated page — check Ready toggle
  → If OFF: run inline escalation flow (Step 3a) → Slack/email backup → stop
  → If ON:
    → For each row: read Status + PM Notes
    → synthesis/note-interpreter.md — resolve PM note intent
    → Route to executors:
      ├─ executors/orbit.md (tasks, projects, comments, due dates)
      ├─ executors/email.md (drafts only, plus the 3 documented send-not-draft exceptions)
      └─ executors/slack.md (team handoffs, AM pings)
    → writers/notion.md — write Done state + Outcome per row
    → writers/plain-language.md — enforce language rule on outputs
    → writers/source-citation.md — cite every source in Orbit task bodies and elsewhere
    → writers/run-log.md — append run entry
  → Slack PM themselves the completion summary

Monthly archival (scheduled on 1st of month at 6:00 AM IST):
  → preflight
  → modes/monthly-archival.md — move previous month's dated pages into a named toggle
  → Enforces parent-page structure per schemas/parent-page.md
  → writers/run-log.md — append run entry

First run (no Preferences page exists):
  → first-run-setup.md — ask 9 questions, write Preferences page, register scheduled tasks

Connector failure at any step:
  → connector-failure-notify.md — Slack PM → email PM (sent, not drafted) → Notion red callout → next-run warning
```

## Data sources (all via MCP, scoped per the allowlist)

| Source | Scope | Tool family |
|---|---|---|
| **Orbit** | PM's projects (owner + follower + active in last 6 months), activity since last run, attachments | `mcp__...orbit.*` |
| **Gmail** | PM's `@whitelabeliq.com` inbox ONLY (single account, even if other accounts are authenticated). Aliases respected. Overnight window. Filtered for client/AM/team/leadership. Skips Orbit notification emails. | `mcp__...gmail.*` |
| **Slack** | Client/AM/project direct asks, project-specific channels, PM DMs. Intentionally minimal — not a full Slack scan. | `mcp__...slack.*` |
| **Fathom** | Calls PM attended (and missed) in last 24h + action items + recording links | `mcp__...fathom.*` |
| **Notion** | Read Preferences. Write dated sub-pages, inline database, row detail pages. On 1st of month: move previous month into named toggle. ONLY the Notion parent page identified in `config.md`. | `mcp__...notion.*` |

Explicitly forbidden: Pipedrive, Apollo, Common Room, Hex, Calendar, Keka, Atlassian, Linear, Intercom, Figma, Klaviyo, Ahrefs, Canva, ClickUp, Monday, Fireflies, Pendo, Amplitude, Quickbooks, or any other connector. Google Drive / Docs / Sheets and SharePoint are allowed read-only on-demand only as defined in `references/external-doc-access.md` — never as primary collection sources.

## Non-negotiable rules

1. **Preflight runs first.** Every invocation, every code path. No exceptions.
2. **V3 stays sealed.** Do not edit any page under the V3 Notion space. See `references/v3-context.md` for what V3 is and why.
3. **Plain language only for India delivery team outputs.** Slack handoffs to team members and Orbit task bodies use 4th–5th grade general English with role-specific technical terms preserved. PMs, AMs, and leadership get normal professional English. See `writers/plain-language.md`.
4. **Source citation on everything sourced from a document.** When the skill reads a PDF, image, PPT, or doc for context, it cites the filename in the output. See `writers/source-citation.md`.
5. **Nothing client-facing or team-facing is auto-sent.** Emails are drafts (with three documented exceptions in `executors/email.md`). Slack to team or AM is drafted into today's dated Notion page under the row's Outcome — the PM copies and sends from there. The only Slack sends Mode 2 may make are: the PM self-summary, the escalation backup ping (Step 3a), and a team handoff with explicit PM `send` note + audience = team.
6. **Availability is not checked.** Skill recommends the most suitable person by role. PM overrides via note if unavailable.
7. **Mode 1 never asks the PM questions.** If unsure about an item, list it as its own row with an `Uncertain:` note in AI Notes. Never block waiting for input.
8. **PM notes are interpreted as natural language.** See `synthesis/note-interpreter.md`. Short notes like `assign to Vijay`, `save as draft`, `mark as high priority` must resolve correctly.
9. **Row-level approval is explicit per row.** No bulk-approve. No status sweep. Each row either gets flipped to `Approved`, gets a note, gets marked `Skip. No Action Needed`, or stays at `Recommended Action` (= no action).
10. **Page-level approval is the `Ready for Execution` toggle at the TOP of each dated page.** Mode 2 only executes if the toggle is ON.
11. **All Orbit task bodies follow the 6-section standard.** Sections in order: DO · WHY · CONTEXT · DONE WHEN · SELF-QA · REFS. Header style defaults to bold professional headers; PM may opt into `emoji` style via Preferences `orbit_task_header_style`. See `schemas/orbit-dq-standard.md`.
12. **Source allowlist is closed.** See top of file.
13. **Notion parent is hardcoded in `config.md`.** No discovery, no asking the PM where to write. The page is fixed per installation.
14. **Connector failures notify the PM via the 4-tier fallback chain** in `connector-failure-notify.md`. Never silently fail.
15. **Email aliases are respected.** Identity matching across all collectors and executors uses the canonical email plus any aliases stored in Preferences.
16. **Every routine fire writes a run-log entry** to the Run Log database on the Notion parent. Brief reason traces (one line per decision: subject → action → reason). See `writers/run-log.md` and `schemas/run-log-database.md`.
17. **All MCP calls retry with backoff before failing.** Every MCP tool call (any of the 5 allowlisted MCPs) wraps in the retry policy defined in `connector-failure-notify.md`: 4 attempts total (1 + 3 retries), 2s/5s/15s incremental backoff, retry only on transient errors (timeout, 5xx, 429, connection reset). Permanent errors (4xx auth, 404, validation) skip retry. After exhaustion, the failure chain fires. The run-log records all retry attempts.

## File map

Everything below this file provides the detailed behavior. Load the specific file when that behavior is needed.

- `ENVIRONMENTS.md` — the two execution surfaces (headless routine/code vs interactive Claude Desktop) and how the skill adapts.
- `config.md` — hardcoded Notion parent + operational rules. Loaded FIRST every invocation.
- `preflight.md` — the 6-step preflight sequence. Runs after config, before any mode logic.
- `connector-failure-notify.md` — 4-tier failure fallback chain.
- `first-run-setup.md` — operator-facing per-PM Preferences-page setup. Use this before the first routine fire (or for any new PM rollout).
- `invocation-commands.md` — exact interactive command syntax (`PM Task Assignment, run morning` / `run execution now` / `validate setup`) and lenient routing rules. Used by Claude Desktop sessions; routines never reach this routing.
- `setup-template.md` — operator-facing one-time per-PM Preferences template. Reference companion to `first-run-setup.md`.
- `ROUTINE-ENTRYPOINTS.md` — the 3 baked routine prompts (Mode 1 morning collection, Mode 2 execution, monthly archival). Each routine fires its prompt by cron and loads this skill from the public GitHub repo.
- `schemas/run-log-database.md` — Run Log database schema (one row per routine fire) on the Notion parent.
- `schemas/run-log-detail-page.md` — per-row detail page layout with the decision trace (subject → action → reason lines).
- `writers/run-log.md` — appends a Run Log entry (and detail page) on every routine fire.
- `DEPLOYMENT.md` — how to deploy this skill to a new PM (edit config, ship, install).
- `modes/mode-1-morning-collection.md` — Mode 1 end-to-end orchestration.
- `modes/mode-2-execution.md` — Mode 2 end-to-end orchestration.
- `modes/monthly-archival.md` — 1st-of-month archival at 6:00 AM IST.
- `collectors/orbit.md`, `gmail.md`, `slack.md`, `fathom.md` — per-source data collection.
- `synthesis/matcher.md` — signal grouping and summary generation, alias-aware.
- `synthesis/pod-inference.md` — how to compute pod per project from Orbit.
- `synthesis/note-interpreter.md` — natural-language PM note resolution.
- `executors/orbit.md`, `email.md`, `slack.md` — per-action-type write operations.
- `writers/notion.md` — Notion page and database creation. Enforces parent-page structure.
- `writers/plain-language.md` — language-splitter rules.
- `writers/source-citation.md` — citation formats.
- `schemas/parent-page.md` — parent Notion page structure (header callout + Year > Month > Date sub-page hierarchy + Run Log + Incidents + Preferences last).
- `schemas/preferences-page.md` — Preferences layout.
- `schemas/morning-queue-database.md` — inline database schema (9 columns, 4 Status options, 4 Source Systems options).
- `schemas/row-detail-page.md` — row detail page layout (headings + reference toggle at bottom).
- `schemas/orbit-dq-standard.md` — 6-section Orbit task body template.
- `references/due-date-categories.md` — PM note → category ID mapping.
- `references/status-values.md` — Status enum semantics.

## Build status

**Version:** 1.1.0 (post-test-run rewrite)
**Built for:** White Label IQ Project Managers
**First installer:** Aditi Singh (subject to change based on availability)
**Distribution:** Editable per-PM via `config.md`. Ship the folder, the receiving PM swaps the Notion page ID, installs on their own Claude account.
**Parallel run expected:** 4–6 weeks alongside the PM's manual morning hour, with accuracy compared before full rollout.

## Source of truth for design decisions

All design decisions that shape this skill are locked in the MVP project workspace in Notion:
[PM Automation MVP 1](https://www.notion.so/3493846840c8805b9424f63dcdc9b9ce)

Never re-invent rules. If a behavior is underspecified in these files, consult the Notion workspace decisions and open items lists.
