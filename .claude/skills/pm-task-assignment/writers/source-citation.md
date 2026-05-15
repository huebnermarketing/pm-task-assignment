# Source Citation

## Purpose

Every output the skill produces that's derived from a source must cite the source. This applies to Morning Queue row detail pages, Orbit task bodies, Slack handoffs that reference attached documents, and emails that quote prior conversation.

## The rule

> If content came from somewhere, cite that somewhere by its filename or URL.

Non-negotiable. Applies to:

- Email threads (cite sender, date, thread, subject)
- Slack messages (cite channel, sender, timestamp, link)
- Orbit tasks / comments / attachments (cite task link, exact quote for comments, filename for attachments)
- Fathom meetings (cite meeting title, date, recording URL, timestamp for specific moments)
- **Documents the skill reads** (PDFs, images, PPTs, Word docs, etc. — cite the filename and where it was attached)

## Why this matters

- PM can verify anything back to the source
- Assignee can check the original document if context is ambiguous
- Creates a paper trail inside the task
- Prevents "telephone" distortion where meaning drifts through retellings

## Citation formats

### Email

```
Email — Gmail
From: Jane Miller <jane.m@agencyx.com>
Date: 25 April 2026, 4:32 AM IST
Thread: Homepage redesign — revision feedback
Subject: Homepage redesign — revision feedback
Excerpt: "<relevant quote from the email, under 200 characters>"
[Open in Gmail](<gmail_url>)
```

### Slack message

```
Slack — #agency-x
From: Sarah Chen (AM)
Date: 24 April 2026, 6:15 PM IST
Message: "<exact message text>"
[Jump to message](<slack_permalink>)
```

### Orbit task or comment

For a task:
```
Orbit task #105892 — [task title]
URL: https://app.whitelabeliq.com/93640173/project/.../105892
```

For a comment — paste the exact quote:
```
Orbit comment by [Author] on [Date]:
"<exact comment text>"
[Open in Orbit](<task_url>)
```

### Fathom meeting

```
Fathom meeting — [Meeting title]
Date: [DD Month YYYY]
Duration: [N] minutes
Attendees: [list]
[Watch recording](<recording_url>)
```

For a specific moment:
```
Fathom moment — [Meeting title], [timestamp in recording]:
"<what was said or the action item>"
[Jump to moment](<recording_url>?t=<timestamp>)
```

### Attached document the skill read

This is the rule the PM explicitly called out. When the skill reads a document:

```
Document — <filename>
Attached to: Orbit task #<task_id> / Gmail thread [<subject>] / [other source]
Content read by the skill on <date>
[Download](<download_url>)  <!-- pre-signed S3 URL or similar -->
```

If the skill summarized or used specific content from inside the document, say so:

```
From `homepage_revision_feedback.pdf` (attached to Orbit task #105892):
  • 12 revisions identified, covering hero image, navigation, two footer sections, and pricing table
  • Summary extracted by the skill on 25 April 2026
[Download feedback PDF](<url>)
```

## Where citations appear

### In the Morning Queue row detail page

Under the `Sources` heading — see `schemas/row-detail-page.md`. Every source that contributed to the item gets a subsection with a full citation.

### In the Orbit task body

In the `📎 REFS` section of the 6-section body — see `schemas/orbit-dq-standard.md`. Pipe-separated list of short citation links:

```
📎 REFS: [Figma](<url>) | [Staging](<url>) | [Client feedback PDF](<url>) | [Previous task](<url>)
```

For document sources, the filename appears explicitly:

```
📎 REFS: [Figma](<url>) | [Client feedback PDF: homepage_revision_feedback.pdf](<url>)
```

### In Slack handoffs

Citations are more casual but still present. List under "Context you need":

```
Context you need:
  • Orbit task: [link]
  • Feedback PDF: `homepage_revision_feedback.pdf` attached to the task
  • Staging: [URL]
```

### In emails

Inline, in the body, where the reference is used:

> "Per the revision feedback attached (`homepage_revision_feedback.pdf`), we'll be starting with the hero image and navigation changes..."

## What to cite vs. not cite

### Cite always
- Anything quoted or paraphrased
- Any factual claim the PM could want to verify
- Any document the skill opened and read

### Don't cite
- General context or common knowledge
- Things the skill computed itself (e.g., pod inference reasoning, availability scores) — those go in `AI Notes` or the bottom Reference Context toggle, not in citations
- Preferences page content (the PM knows what's in their own preferences)

## Writing style

- Citations are factual and precise. No embellishment.
- Use the exact filename as it appears in the source, quoted if it contains spaces.
- Dates in the locked format: `25 April 2026`
- Times in IST: `4:32 AM IST`
- Links use Markdown link syntax for clickable display

## When a source can't be cited precisely

If a download URL is expired or unavailable, cite what you can:

```
Document — homepage_revision_feedback.pdf
Attached to: Orbit task #105892 (link temporarily unavailable)
Content read by the skill on 25 April 2026
```

If a Slack or Gmail message can't be pulled precisely, cite approximately:

```
Slack — #agency-x (approximate)
Recent message from Sarah Chen (AM) regarding Agency X impatience.
[Link unavailable]
```

Transparent > silent. The PM needs to know when a citation is imperfect.

## What this module does NOT do

- Does not fabricate citations. If a source can't be traced, say so.
- Does not bury citations in tiny text. They're visible and scannable.
- Does not cite the Preferences page as a source for general context.
- Does not cite the skill's own computations — those are reasoning, not sources.
