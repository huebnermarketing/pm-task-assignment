# Schema — Orbit Data Quality Standard

The six-section template every Orbit task body the skill creates MUST follow. Carried from WLIQ's V3 reference (see `references/v3-context.md` for what V3 is):
[Orbit Data Quality Standard](https://www.notion.so/3423846840c881b09483f2cebe1b5ec6)

## Header style — configurable per PM

Two header styles supported. Default is `professional`. The PM may override via the `orbit_task_header_style` field in their Preferences page (see `schemas/preferences-page.md`).

| Style value     | Section headers used                                                                            |
| --------------- | ----------------------------------------------------------------------------------------------- |
| `professional` (default) | `**DO:**`, `**WHY:**`, `**CONTEXT:**`, `**DONE WHEN:**`, `**SELF-QA:**`, `**REFS:**`   |
| `emoji`         | `📌 DO`, `🎯 WHY`, `🔗 CONTEXT`, `✅ DONE WHEN`, `🔍 SELF-QA`, `📎 REFS`                          |

Section content, ordering, and rules below are identical across styles. Only the header glyph changes.

## The template (default — `professional` style)

```
**DO:** [What specifically needs to be done. 1–2 sentences max.]

**WHY:** [Why this matters. 1 sentence.]

**CONTEXT:** [Project phase. What came before. What depends on this.]

**DONE WHEN:**
  • [Criterion 1]
  • [Criterion 2]
  • [Criterion 3]

**SELF-QA:**
  • [Work-type specific check 1]
  • [Work-type specific check 2]
  • Compared against original requirement
  • Left a completion comment

**REFS:** [Links separated by | ]
```

## The template (`emoji` style — opt-in)

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

Every task the skill creates follows the section order and content rules below, using whichever header style Preferences specifies (default `professional`).

## Language rule

**Plain language** (4th–5th grade English) per `writers/plain-language.md`. Delivery team reads this. Role-specific technical terms preserved.

## Source rule — body is composed from cross-linked context, not just the Orbit task snapshot

The 6-section body is NOT a copy of the originating Orbit task description, nor a copy of the workload snapshot. The matcher composes it by reading FOUR input sources in full, default-on, before composing any text:

1. **The originating Orbit task in full** — mandatory per-row deep-read fires during matcher Job 7 (see `collectors/orbit.md` § Per-row deep-read). Two MCP calls per row:
   - `get_task_details(task_id)` — full task body / description / all fields. NOT just the workload snapshot.
   - `list_task_comments(task_id)` — **complete all-time comment history**, NOT date-filtered to `last_run_timestamp`. The collection-phase activity_log returns only the deltas since last_run; older comments (prior client feedback, AM clarifications, decisions, failed attempts, scope changes from earlier weeks/months) live here and routinely hold the load-bearing context for the brief.
2. **Every cross-linked Gmail thread** attached as `signal.context_signals[]` via matcher Job 4b Pass 1. Read the FULL thread depth — every message, every reply, every attachment — not just the latest message. Long threads are common when an engagement spans days; the load-bearing context (prior decisions, named POCs, scope clarifications, deadline reasoning) often lives mid-thread.
3. **Any Fathom enrichment** attached as `signal.enrichment.fathom` via matcher Job 4b Pass 2 (including the gmail-attachment fallback variant when Fathom MCP was unavailable).
4. **Any external document referenced** by any of the above and fetched via `references/external-doc-access.md` (Drive / Docs / Sheets / SharePoint, read-only).

Per-section mapping rules — all FOUR input sources feed the same sections; the mapping is source-agnostic (full detail in `synthesis/matcher.md` Job 7 § Mandatory email-thread enrichment + § Mandatory deep-read of the originating Orbit task):

- **DO** — if any input source defines the deliverable more precisely than the Orbit task title, use the more precise phrasing.
- **WHY** — pull motivation from AM/client wording across all sources ("board demo Thursday", "client is impatient") rather than generic "client urgency".
- **CONTEXT** — surface prior rounds, project phase, named POCs, dependencies. Long-thread email context AND long comment history on the Orbit task itself both land here most often. Comments older than `last_run_timestamp` (not in the activity_log delta) routinely carry load-bearing prior decisions.
- **DONE WHEN** — acceptance criteria called out anywhere ("must work on Safari iOS", "preview link to Jane", "include legal disclaimer") go here verbatim in plain language.
- **SELF-QA** — role-specific items plus any explicit checks any source called out.
- **REFS** — cite EVERY context source: Orbit parent task URL, every Gmail thread URL, Fathom recording URL (if any), any document URLs referenced anywhere.

**This is unconditional.** Do NOT skip the Orbit deep-read or the email/Fathom enrichment because the workload snapshot or the task title looks "complete enough". The runtime cannot judge completeness without reading; reading first is the only way to know. The bias is over-include, not under-include.

**Conflict resolution.** When sources conflict (email says one deadline, Orbit task description says another, an older comment said a third), prefer the most recent authoritative source (latest AM message, latest client decision, latest task comment). When the resolution is unclear, surface the conflict in the row's AI Notes with `Uncertain:` prefix and let the PM disambiguate.

## Section-by-section guidance

Section headers below use the `professional` style. The `emoji` equivalents are documented in the header-style table above.

### DO

What exactly needs to happen. One to two sentences, specific.

Good:
> "Swap the hero image on the Agency X homepage with the new client creative. Optimize for web (under 200KB), responsive across breakpoints."

Not good:
> "Please work on the hero image updates as per the attached feedback document."

### WHY

Why this matters. One sentence. Gives the assignee context to make judgment calls.

Good:
> "Client is presenting to their board next Thursday. They need the updates today."

Not good:
> "Per the client's request, we need to ensure timely delivery of the assets."

### CONTEXT

Project phase and dependencies. Two to four sentences. What came before, what depends on this.

Good:
> "This is Agency X's second revision round on the homepage. First round was approved on April 12. Jane Miller is the primary client POC. The site runs on WordPress with a custom child theme."

Not good:
> "Related context is available in prior communications and project documentation."

### DONE WHEN

Bulleted list. Three to six criteria. Specific, verifiable, unambiguous.

Good:
- All 12 revisions applied to the mockup
- Preview link shared with Jane (email or chat — PM picks the channel)
- No console errors on mobile or desktop
- Mannan signs off on QA

Not good:
- Work complete
- Client satisfied
- Quality assured

### SELF-QA

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

### REFS

Pipe-separated list of links. Every link is clickable Markdown.

Good:
> [Figma file](<url>) | [Staging URL](<url>) | [Previous homepage task (Round 1)](<url>) | [Client feedback PDF: homepage_revision_feedback.pdf](<url>)

Not good:
> Files and references available in project folder.

## Hard rules

- Every section is present. If a section legitimately has nothing to say, still include the heading with a placeholder like `[None — reference the task title for context]` rather than omitting.
- Keep sections tight. If DO or WHY run longer than two sentences, split the task — it's probably too big.
- Keep formatting consistent within a single PM's tasks. The header style is set per-PM in Preferences (`orbit_task_header_style`); the executor reads it on every run and applies it uniformly. Section order is fixed regardless of style.
- REFS always includes at least one actual link. If there are no relevant links, include a link to the originating signal (the email thread, the Orbit parent task, the Fathom recording).

## Example — full Orbit task body (`professional` style, default)

```
**DO:** Apply the 12 revisions from the client feedback PDF. Hero image swap, navigation restructure, two new footer callouts, and pricing table restructure.

**WHY:** Client is presenting to their board next Thursday. Sarah (AM) said they are getting impatient.

**CONTEXT:** This is Agency X's second revision round on the homepage. Round 1 was approved on April 12. Jane Miller is the client POC. The site is WordPress with a custom child theme.

**DONE WHEN:**
  • All 12 revisions applied to the mockup
  • Preview link shared with Jane (email or chat — PM picks the channel)
  • No console errors on mobile or desktop
  • Mannan signs off on QA

**SELF-QA:**
  • Tested on Chrome, Safari, Firefox, Edge
  • Responsive on desktop + tablet + mobile
  • No console errors
  • Performance check passes
  • Compared against original requirement
  • Left a completion comment

**REFS:** [Figma file](<url>) | [Staging URL](<url>) | [Previous homepage task (Round 1)](<url>) | [Client feedback PDF: homepage_revision_feedback.pdf](<url>) | [WP admin URL](<url>)
```

The same body in `emoji` style swaps `**DO:**` → `📌 DO:`, `**WHY:**` → `🎯 WHY:`, etc.

## What this template does NOT include

- Estimated hours (set as an Orbit task field, not in the body)
- Due date (set as an Orbit task field)
- Assignee (set as an Orbit task field)
- Comments or updates (those go in Orbit comments, not the body)
- AI Notes or uncertainty flags (those go in the skill's row detail page only)
