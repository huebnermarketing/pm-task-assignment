# Schema — Row Detail Page

Each row in the Morning Queue database opens to its own page. This file defines what that page looks like inside.

## Non-negotiable rule from the design

**Primary content uses HEADINGS (always visible), not toggles.** The PM should be able to scroll the row page top-to-bottom and see everything important without clicking anything.

**ONE toggle appears at the very bottom** for the skill's working-memory reference context, clearly labeled so the PM knows they don't need to read it.

## Page structure (top to bottom)

### H1: Summary

Plain-language one-liner that matches the `Summary` column exactly. This is the PM's anchor — what they're looking at in one sentence.

Then a short paragraph with additional context if the summary alone isn't enough. Normal English.

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

#### H2: Slack — #agency-x (if applicable)

> **From:** Sarah Chen (AM)
> **Date:** 24 April 2026, 6:15 PM IST
> **Message:** "Hey Hiten — heads up, Jane from Agency X pinged me separately asking about the homepage. They're getting impatient. Can you prioritize this tomorrow morning?"
> [Jump to message](<url>)

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

### H1: Recommended Action

A short paragraph describing what the skill plans to do and why this specific choice.

> Create a new Orbit task under `Agency X — Homepage Redesign` for the revision work. Assign to **Vijay Patel (FE)**. Send Slack handoff to Vijay with full context and the 12 revisions from the PDF. Slack Caitlin (AM) to let her know the task is assigned and will begin today.
>
> **Why Vijay:** Primary FE on Agency X, logged 24 hrs on current homepage task, has 12 hours available this week.

### H1: Proposed Orbit Task Body

The full 6-section task body that will land in Orbit when Mode 2 executes. Per `schemas/orbit-dq-standard.md`. **In plain language** per `writers/plain-language.md` — the assignee reads this.

```
📌 DO: [what to do]
🎯 WHY: [why it matters]
🔗 CONTEXT: [project phase, what came before]
✅ DONE WHEN:
  • [criterion 1]
  • [criterion 2]
  • [criterion 3]
🔍 SELF-QA:
  • [role-specific check 1]
  • [role-specific check 2]
  • [always] Left a completion comment
📎 REFS: [link1] | [link2] | [link3]
```

### H1: Proposed Slack Handoff

The exact Slack message that will be sent to the assignee when Mode 2 executes. **In plain language.** Matches the template in `executors/slack.md`.

### H1: Proposed Email (if applicable)

The email draft that will land in the PM's Gmail drafts folder. **In normal professional English** (the recipient is typically a client or AM).

Full email: To, CC (if any), Subject, Body.

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

## Slack — #channel
<full citation>

# Recommended Action
<paragraph with reasoning>

# Proposed Orbit Task Body
📌 DO: ...
🎯 WHY: ...
🔗 CONTEXT: ...
✅ DONE WHEN: ...
🔍 SELF-QA: ...
📎 REFS: ...

# Proposed Slack Handoff
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
