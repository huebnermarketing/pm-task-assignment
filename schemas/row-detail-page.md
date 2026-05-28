# Schema — Row Detail Page

Each row in the Morning Queue database opens to its own page. This file defines what that page looks like inside.

## Non-negotiable rule from the design

**Primary content uses HEADINGS (always visible), not toggles.** The PM should be able to scroll the row page top-to-bottom and see everything important without clicking anything.

**The top of the page is action-first, then brief-first, then sources.** When the PM opens a row, the FIRST thing they see is what AI proposes to do (top callout: task title + assignee for Create subtask / Reopen subtask, pod leader for Hand off parent task, PM-next-step for Flag, proposed parent title for Create parent task). Directly below the H1 Summary heading sits the H1 **Task Brief** — a 2–4 sentence description of what the work is about + the new update / latest signal text that triggered this row. H1 **Sources** sits BELOW the brief, not directly under Summary. The PM should not have to scroll past Sources to find the actionable proposal OR the new context.

**ONE toggle appears at the very bottom** for the skill's working-memory reference context, clearly labeled so the PM knows they don't need to read it.

## Page structure (top to bottom)

### Top callout — Action Block (ALWAYS first, NO heading above it)

A single Notion `callout` block at the very top of the page. Plain text only — no emojis, no decorative glyphs. The first line names the action verb in bold; subsequent lines carry the structured fields. Content depends on the row's verb (one of the five locked verbs from `synthesis/matcher.md` Job 4):

**Create subtask:**
```
Create subtask: <proposed task_title>
Parent: <project short name> #<parent_task_id> (link)
Assignee: <full name> (<role>)
Why this assignee: <one-line reason from Job 6>
```

**Reopen subtask:**
```
Reopen subtask: <existing subtask title> (#<existing_subtask_id>)
Parent: <project short name> #<parent_task_id> (link)
Reassign to: <last dev full name> (<role>) — last non-PM activity on this subtask
What's new: <one-line summary of new_work_description>
```

**Hand off parent task:**
```
Hand off parent task: <parent task title> (#<parent_task_id>)
Reassign to: <pod leader full name> (<pool> lead)
work_type: <work_type token> — non-pod work, no subtask created
PM stays as: follower (audit only)
```

**Flag:**
```
Flag for PM: <one-line topic>
PM next step: <pm_next_step clause from Job 5>
No Orbit auto-execute on this row.
```

**Create parent task:**
```
Create parent task: <proposed parent title>
Project: <project short name> (#<project_number>)
Assignee: You (PM)
Note: Possible Orbit miss — no corroborating Orbit task found
```

Notion's `callout` block type carries its own optional icon — leave the icon slot empty or use Notion's default. The verb word at the start of the first line is the sole indicator of row type; PMs scan the bold text, not an icon.

### H1: Summary

The topic-style one-liner that matches the `Summary` column exactly. Below the action callout so the PM has already seen the actionable proposal.

### H1: Task Brief

A 2–4 sentence paragraph describing what the work is about + the new update / latest signal that triggered this row. Composed by matcher Job 7b. Pulls primarily from the most recent input source (top of email thread, latest Orbit comment, AM clarification) — not from the older parent-task description. The full Orbit task body / older context still lives in the 6-section Orbit body (further down on this page for Create subtask / Create parent task rows) and the Sources block (every row).

Length cap 600 chars. For `Flag` rows this becomes a 2–3 sentence "here's what came in" summary. For `Hand off parent task` rows the brief notes why this work is being handed off (work_type → non-pod routing). For `Reopen subtask` rows the brief surfaces what's NEW relative to the existing subtask, not a recap of the existing scope.

### H1: Sources

Every source that contributed to this item, each as an H2 subsection. Cited per `writers/source-citation.md`.

#### H2: Email — Gmail (if applicable)

Full citation. Example:

> **From:** Jane Miller <jane.m@agencyx.com>
> **Date:** 25 April 2026, 4:32 AM IST
> **Thread:** Homepage redesign — revision feedback
> **Subject:** Homepage redesign — revision feedback
> **Excerpt:** "Hi Hiten, attached is our feedback on the homepage mockups. We'd love to get the revisions started ASAP..."
> **Attachments:** `homepage_revision_feedback.pdf` (auto-extracted: 12 revisions — hero, navigation, footer, pricing)
> [Open in Gmail](<url>)

#### H2: Orbit (if applicable)

> **Project:** Agency X — Homepage Redesign ([link](<url>))
> **Related task:** Homepage redesign v2 ([link](<url>))
> **Current status:** In Progress
> **Current assignee:** Vijay Patel
> **Overdue:** 1 day

#### H2: Fathom enrichment (if applicable)

Appears only when matcher Job 4b Pass 2 fetched Fathom enrichment for this row (i.e. a primary signal referenced a meeting via a trigger phrase). Renders the `enrichment.fathom` payload from the signal — never appears for rows without a triggering reference.

> **Meeting:** [Meeting title]
> **Date:** 24 April 2026, 3:00 PM IST
> **Duration:** 38 minutes
> **Attendees:** [list]
> **Triggered by:** [the trigger phrase from the primary signal — e.g. "per our call"]
> **Relevant extract:** [scoped 2-4 sentence summary excerpt]
> **Relevant action items:** [only items overlapping the primary signal's project / actors]
> **Match confidence:** high / medium / low
> [Watch recording](<url>)

#### H2: Document read (if applicable)

Only appears when the skill opened and read a document for context.

> **Document:** `homepage_revision_feedback.pdf`
> **Attached to:** Orbit task #105892 / Gmail thread with Jane Miller
> **Content read by the skill on:** 25 April 2026
> **Summary of what was read:** [what the skill pulled from the document]
> [Download](<url>)

### H1: Recommended Action (context expansion)

A short paragraph expanding on the action callout at the top of the page. Adds the WHY-this-choice reasoning so the PM understands the matcher's logic without re-deriving it. For `Create subtask` / `Reopen subtask` rows, this is where the assignee-pick reasoning (Job 6 branch trace, or Job 5.5 last-dev extraction for Reopen) lives in narrative form. For `Hand off parent task` rows this explains the work_type → pool routing. For `Flag` rows this names what is known about the signal and what the PM specifically needs to weigh.

> Sub-task under `ECP AI Visibility Audit` parent task #110464 (currently assigned to you). Title: `Run AI Visibility audit on approved competitor list`. Assign to **Hitesh Asnani**.
>
> **Why Hitesh:** Already on the project (in followers); Ellen named him alongside you in the approval comment. SEO/AI audit specialist (Marketing/SEO Matrix). Branch (a) history-wins overrides the cross-matrix preference because prior involvement + AM mention is the decisive signal.

### H1: Proposed Orbit Task Body (Create subtask / Create parent task rows ONLY)

The full 6-section body that lands in Orbit when Mode 2 executes. Format per `schemas/orbit-dq-standard.md`, plain language per `writers/plain-language.md`. Skipped for `Reopen subtask` (existing subtask already has a body), `Hand off parent task` (existing parent already has a body), and `Flag` (no Orbit write) rows.

### H1: Proposed New-Work Comment (Reopen subtask rows ONLY)

The plain-language comment Mode 2 will post on the existing subtask. Header line `Reopened — new work from <signal_date IST>:` plus 2–4 sentences describing what the new signal adds. Source citation for the originating Gmail thread / Orbit comment included.

### H1: PM Next Step (Flag rows ONLY)

A short paragraph expanding the `pm_next_step` clause. What the PM should do, why, and any helper context (e.g., "Reply to Ellen on the Joe Warner thread; suggested devs given Joe framed the call as technical: <names from Orbit followers on #16320>"). Normal English.

### H1: Proposed Handoff (Create subtask / Reopen subtask / Hand off parent task rows)

The handoff body Mode 2 will draft and append to the row's Outcome on today's dated Notion page. Plain language, audience = team dev (for Create subtask / Reopen subtask) or pool leader (for Hand off parent task). The PM copies the rendered text and delivers it through whatever channel they use for that recipient. `Flag` and `Create parent task` rows skip this section (no delegate to brief).

### H1: AI Notes (if any)

Only appears if there's something notable. Otherwise omit this section entirely.

Includes:
- Any `Uncertain:` flags
- Why a PM note was interpreted in a particular way
- When the source signal was split into multiple items
- When a collector failed for this item
- When the recommended assignee is unconventional

### Bottom toggle: Reference Context (for the skill — not for review)

A single toggle block at the very bottom. Clearly labeled so the PM knows it's not for them:

> **Reference Context for the Skill — not for your review, just the skill's memory.**

Inside the toggle:

- Raw signals from every collector (full content, not excerpts)
- The pod inference reasoning (who was considered, why this person was picked)
- Cross-source match reasoning
- Any debug info or metadata
- Links to prior instances of this type of item (for adaptive learning lookup)

The toggle stays closed by default. The PM never needs to open it. The skill reads from this toggle during Mode 2 when interpreting a PM note — the full context lives here.

## What the page does NOT include

- **No toggles for primary content** — headings only. PM scrolls and sees everything.
- **No hidden important content.** If it matters for the PM's decision, it's at the top, not behind a click.
- **No marketing or boilerplate language.** Every section is factual and actionable.
- **No redundant info from the database columns.** Status, Recommended Action, and Recommended Assignee are already visible in the row; the page doesn't repeat them — it expands on them.
- **No auto-suggested next steps beyond the Recommended Action section.** The page is about THIS item, not future items.

## Rendering notes

- Use Notion heading blocks (not just bold text) for H1 / H2 structure so the Notion outline makes sense
- Keep images / screenshots off unless specifically relevant — text is more scannable
- Use Notion callout blocks for the very top-of-page warning IF the item is time-critical (e.g., overdue, blocker)
- Links open in new tabs by default in Notion

## Example (structure only, not content)

```
[Top callout — Action Block — plain-text verb label per row type, no emoji]

# Summary
<topic-style one-line summary>

# Task Brief
<2-4 sentence paragraph: what the work is about + the new update / latest signal>

# Sources

## Email — Gmail
<full citation>

## Fathom enrichment — call title (only if matcher fetched it)
<scoped citation with trigger phrase + relevant extract>

# Recommended Action
<paragraph with reasoning — assignee-pick logic for Create/Reopen, work_type → pool routing for Hand off, signal-weighing for Flag>

# Proposed Orbit Task Body  (Create subtask / Create parent task only)
**DO:** ...
**WHY:** ...
**CONTEXT:** ...
**DONE WHEN:** ...
**SELF-QA:** ...
**REFS:** ...

# Proposed New-Work Comment  (Reopen subtask only)
Reopened — new work from <date>:
<2-4 sentences describing what's new>

# PM Next Step  (Flag only)
<paragraph expanding pm_next_step clause>

# Proposed Handoff  (Create subtask / Reopen subtask / Hand off parent task)
<plain-English handoff text for the recipient>

# AI Notes
<only if needed>

> ▸ Reference Context for the Skill — not for your review
  <hidden details, skill's working memory>
```
