# Schema — Row Detail Page

Each row in the Morning Queue database opens to its own page. This file defines what that page looks like inside.

## Non-negotiable rule from the design

**Primary content uses HEADINGS (always visible), not toggles.** The PM should be able to scroll the row page top-to-bottom and see everything important without clicking anything.

**The top of the page is action-first, not narrative-first.** When the PM opens a row, the FIRST thing they see is what AI proposes to do (task title + assignee for Create subtask, or the PM-next-step for Flag). Sources and context move below. The PM should not have to scroll to find the actionable proposal.

**ONE toggle appears at the very bottom** for the skill's working-memory reference context, clearly labeled so the PM knows they don't need to read it.

## Page structure (top to bottom)

### Top callout — Action Block (ALWAYS first, NO heading above it)

A single Notion `callout` block at the very top of the page. Content depends on the row's action:

**Create subtask:**
```
🎯 Create subtask: <proposed task_title>
Parent: <project short name> #<parent_task_id> (link)
Assignee: <full name> (<role>)
Why this assignee: <one-line reason from Job 6>
```

**Flag:**
```
🚩 Flag for PM: <one-line topic>
PM next step: <pm_next_step clause from Job 5>
No Orbit auto-execute on this row.
```

The 🎯 / 🚩 glyphs are part of the structural emoji allowlist for row callouts (see `writers/plain-language.md` emoji policy). They mark row type at a glance when the PM opens the page.

### H1: Summary

The narrative one-liner that matches the `Summary` column exactly. Below the action callout so the PM has already seen the actionable proposal.

Then a short paragraph with additional context if the summary alone isn't enough. Normal English.

### H1: Proposed Orbit Task Body (Create subtask rows ONLY)

The full 6-section body that lands in Orbit when Mode 2 executes. Moved UP — sits before Sources so the PM sees the brief before context citations.

Format per `schemas/orbit-dq-standard.md`, plain language per `writers/plain-language.md`.

For Flag rows, this section is replaced by:

### H1: PM Next Step (Flag rows ONLY)

A short paragraph expanding the `pm_next_step` clause. What the PM should do, why, and any helper context (e.g., "Reply to Ellen on the Joe Warner thread; suggested devs given Joe framed the call as technical: <names from Orbit followers on #6994>"). Normal English.

### H1: Proposed Handoff (Create subtask rows ONLY)

The handoff body Mode 2 will draft and append to the row's Outcome on today's dated Notion page. Plain language, audience = team dev. The PM copies the rendered text and delivers it through whatever channel they use for that team member (in-person walk-through, email, chat app, etc.). Flag rows skip this section.

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

#### H2: Fathom (if applicable)

> **Meeting:** [Meeting title]
> **Date:** 24 April 2026, 3:00 PM IST
> **Duration:** 38 minutes
> **Attendees:** [list]
> **Relevant extract:** [action item or quote]
> [Watch recording](<url>)

#### H2: Document read (if applicable)

Only appears when the skill opened and read a document for context.

> **Document:** `homepage_revision_feedback.pdf`
> **Attached to:** Orbit task #105892 / Gmail thread with Jane Miller
> **Content read by the skill on:** 25 April 2026
> **Summary of what was read:** [what the skill pulled from the document]
> [Download](<url>)

### H1: Recommended Action (context expansion)

A short paragraph expanding on the action callout at the top of the page. Adds the WHY-this-choice reasoning so the PM understands the matcher's logic without re-deriving it. For Create subtask rows, this is where the assignee-pick reasoning (Job 6 branch trace) lives in narrative form.

> Sub-task under `ECP AI Visibility Audit` parent task #110464 (currently assigned to you). Title: `Run AI Visibility audit on approved competitor list`. Assign to **Hitesh Asnani**.
>
> **Why Hitesh:** Already on the project (in followers); Ellen named him alongside you in the approval comment. SEO/AI audit specialist (Marketing/SEO Matrix). Branch (a) history-wins overrides the cross-matrix preference because prior involvement + AM mention is the decisive signal.

(For Flag rows, this section names what is known about the signal and what the PM specifically needs to weigh.)

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
# Summary
<one-line summary>
<optional 1-2 sentence context>

# Sources

## Email — Gmail
<full citation>

## Fathom — call title
<full citation>

# Recommended Action
<paragraph with reasoning>

# Proposed Orbit Task Body
**DO:** ...
**WHY:** ...
**CONTEXT:** ...
**DONE WHEN:** ...
**SELF-QA:** ...
**REFS:** ...

# Proposed Handoff
<plain-English handoff text>

# Proposed Email
To: ...
Subject: ...
Body: ...

# AI Notes
<only if needed>

> ▸ Reference Context for the Skill — not for your review
  <hidden details, skill's working memory>
```
