# Schema — Orbit Data Quality Standard

The six-section template every Orbit task body the skill creates MUST follow. Carried verbatim from WLIQ's V3 reference:
[📛 Orbit Data Quality Standard](https://www.notion.so/3423846840c881b09483f2cebe1b5ec6)

## The template

```
📌 DO: [What specifically needs to be done. 1–2 sentences max.]

🎯 WHY: [Why this matters. 1 sentence.]

🔗 CONTEXT: [Project phase. What came before. What depends on this.]

✅ DONE WHEN:
  • [Criterion 1]
  • [Criterion 2]
  • [Criterion 3]

🔍 SELF-QA:
  • [Work-type specific check 1]
  • [Work-type specific check 2]
  • Compared against original requirement
  • Left a completion comment

📎 REFS: [Links separated by | ]
```

Every task the skill creates follows this structure, in this order, with these exact emoji labels.

## Language rule

**Plain language** (4th–5th grade English) per `writers/plain-language.md`. Delivery team reads this. Role-specific technical terms preserved.

## Section-by-section guidance

### 📌 DO

What exactly needs to happen. One to two sentences, specific.

Good:
> "Swap the hero image on the Agency X homepage with the new client creative. Optimize for web (under 200KB), responsive across breakpoints."

Not good:
> "Please work on the hero image updates as per the attached feedback document."

### 🎯 WHY

Why this matters. One sentence. Gives the assignee context to make judgment calls.

Good:
> "Client is presenting to their board next Thursday. They need the updates today."

Not good:
> "Per the client's request, we need to ensure timely delivery of the assets."

### 🔗 CONTEXT

Project phase and dependencies. Two to four sentences. What came before, what depends on this.

Good:
> "This is Agency X's second revision round on the homepage. First round was approved on April 12. Jane Miller is the primary client POC. The site runs on WordPress with a custom child theme."

Not good:
> "Related context is available in prior communications and project documentation."

### ✅ DONE WHEN

Bulleted list. Three to six criteria. Specific, verifiable, unambiguous.

Good:
- All 12 revisions applied to the mockup
- Preview link shared with Jane via Slack
- No console errors on mobile or desktop
- Mannan signs off on QA

Not good:
- Work complete
- Client satisfied
- Quality assured

### 🔍 SELF-QA

Role-specific self-check items. Plus always: "Compared against original requirement" and "Left a completion comment".

Role-specific self-QA examples:

| Role | Self-QA items |
|---|---|
| FE | Tested on Chrome, Safari, Firefox, Edge; Responsive on desktop + tablet + mobile; No console errors; Performance check (page load under 2s) |
| BE | API tested; Error handling verified; DB migrations clean; Documented |
| WP | Plugin conflicts checked; Theme updates tested on staging first; WP admin URL still accessible; All dynamic pages rendering |
| QA | Test cases covered; Regression passed; Browser matrix complete; Device matrix complete |
| Design | Brand consistent; Responsive preview; Correct file formats exported; Assets named per project convention |
| Content | Grammar and spelling checked; SEO meta filled; Tone matches client voice; Internal links verified |

### 📎 REFS

Pipe-separated list of links. Every link is clickable Markdown.

Good:
> [Figma file](<url>) | [Staging URL](<url>) | [Previous homepage task (Round 1)](<url>) | [Client feedback PDF: homepage_revision_feedback.pdf](<url>)

Not good:
> Files and references available in project folder.

## Hard rules

- Every section is present. If a section legitimately has nothing to say, still include the heading with a placeholder like `[None — reference the task title for context]` rather than omitting.
- Keep sections tight. If DO or WHY run longer than two sentences, split the task — it's probably too big.
- Keep formatting exact. The emoji labels and section order are recognizable at-a-glance and become muscle memory for the team.
- REFS always includes at least one actual link. If there are no relevant links, include a link to the originating signal (the email thread, the Slack message, the Fathom recording).

## Example — full Orbit task body

```
📌 DO: Apply the 12 revisions from the client feedback PDF. Hero image swap, navigation restructure, two new footer callouts, and pricing table restructure.

🎯 WHY: Client is presenting to their board next Thursday. Sarah (AM) said they are getting impatient.

🔗 CONTEXT: This is Agency X's second revision round on the homepage. Round 1 was approved on April 12. Jane Miller is the client POC. The site is WordPress with a custom child theme.

✅ DONE WHEN:
  • All 12 revisions applied to the mockup
  • Preview link shared with Jane via Slack
  • No console errors on mobile or desktop
  • Mannan signs off on QA

🔍 SELF-QA:
  • Tested on Chrome, Safari, Firefox, Edge
  • Responsive on desktop + tablet + mobile
  • No console errors
  • Performance check passes
  • Compared against original requirement
  • Left a completion comment

📎 REFS: [Figma file](<url>) | [Staging URL](<url>) | [Previous homepage task (Round 1)](<url>) | [Client feedback PDF: homepage_revision_feedback.pdf](<url>) | [WP admin URL](<url>)
```

## What this template does NOT include

- Estimated hours (set as an Orbit task field, not in the body)
- Due date (set as an Orbit task field)
- Assignee (set as an Orbit task field)
- Comments or updates (those go in Orbit comments, not the body)
- AI Notes or uncertainty flags (those go in the skill's row detail page only)
