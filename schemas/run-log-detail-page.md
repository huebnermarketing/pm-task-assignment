# Schema: Run Log Detail Page

## Purpose

A human-readable trace of what a single routine run **saw**, **decided**, **did**, and **skipped** — with a short reason on every decision. One detail page per Run Log row. The PM (or operator on triage) reads this when something looks off in the database row; otherwise it sits silently as audit history.

**Brief, not verbose.** One line per decision. No paragraphs, no full signal text, no tool-call dumps. Reasons are 5–15 words and concrete.

---

## Page Title

Matches the `Run ID` from the corresponding Run Log database row exactly. Example: `2026-04-29-mode1-0930`.

The page lives as a sub-page of the parent (same parent as the `Run Log` sub-page) or as a child of the `Run Log` sub-page — implementation chooses one and is consistent. Default: child of the `Run Log` sub-page so all detail pages are co-located. The database row's `Detail` column holds the URL.

---

## Page Layout (sections in order)

### 1. Header callout

Single callout block, one line:

> Mode N · `HH:MM`–`HH:MM` IST · Status · `<S>` signals · `<W>` items · `<E>` errors

Example:

> Mode 1 · 09:30–09:33 IST · OK · 18 signals · 6 items · 0 errors

### 2. Sources section

Heading: `Sources`. One bullet per primary collector + one bullet for Fathom enrichment.

```
- Orbit: 12 signals, OK
- Gmail: 4 signals, OK
- Fathom (enrichment): 2 fetched / 3 references / 1 unresolved, OK
```

Primary collector line format: `[name]: [count] signals, [status]`. Fathom line format: `Fathom (enrichment): [N fetched] / [M references detected] / [K unresolved], [status]`. The "references detected" count comes from matcher Job 4b Pass 2 trigger-phrase scanning; "fetched" is the subset where the enrichment service returned non-null; "unresolved" is the remainder (no matching meeting, or Fathom failure).

If a primary collector failed, the status is `FAILED — <one-line reason>`. If Fathom failed, the status is `FAILED (non-blocking) — <one-line reason>`.

### 3. Decisions section

Heading: `Decisions`. One bullet per item written to the queue. Format:

```
Item N — [subject] → [action] → [reason]
```

`[subject]` is the short signal subject (project + assignee or task name). `[action]` is the queue-row action label (`assign-task`, `nudge-PM`, `escalate`, etc.). `[reason]` is 5–15 words, concrete. Show the connection to evidence.

### 4. Skipped section

Heading: `Skipped`. One bullet per signal the skill chose not to write a queue row for. Format:

```
[source]: [subject] → skipped → [reason]
```

### 5. Uncertain section

Heading: `Uncertain`. Items where the skill could not confidently route. The signal still becomes a queue row, but the row is flagged `needs-PM-decision`. List the candidate options the skill considered. Format:

```
Item N — [subject] → uncertain → candidates: [opt-A], [opt-B] — [reason]
```

### 6. Connector failures section

Heading: `Connector failures`. One bullet per tier reached in `connector-failure-notify.md`. Format:

```
[connector]: [step] → tier-N → [error one-liner]
```

Omit the section if no failures occurred (do not write an empty heading).

### 7. Mode 2 execution section (Mode 2 only)

Heading: `Execution outcomes`. One bullet per approved row Mode 2 attempted. Format:

```
Row N: [task] → executed | failed | skipped → [PM-note interpretation if any]
```

The `[PM-note interpretation]` segment is only present when the PM left a note on the row that the skill had to interpret (e.g. "wait until Wed", "ask Priya first"). Interpretation is one short clause — what the skill understood it to mean.

Omit the section for Mode 1 and Monthly Archival.

### 8. Footer

Two links, one bullet each:

- `↩ Run Log database` — link to the `Run Log` sub-page.
- `↩ Today's queue page` (Mode 1 / Mode 2) **or** `↩ Verified month container` (Monthly Archival) — link to the page the run wrote to or verified.

---

## Style Rules

- **One line per decision.** No paragraphs. No multi-sentence reasons.
- **Reasons are 5–15 words.** Concrete, evidence-anchored. Good: `role-fit WP-dev, only WP-dev in pod`. Bad: `best fit`, `seemed appropriate`, `matched preferences`.
- **No full signal text.** No email bodies, no Fathom transcripts.
- **No tool-call dumps.** No JSON payloads, no MCP request/response.
- **Privacy:** subjects may mention project names + internal assignee names (already on the parent page). Do not include client PII (client emails, phone numbers, contract details) beyond what is already publicly on the parent.
- If the writer receives a multi-paragraph reason from a calling mode, **truncate to 15 words and append `…`** (the writer enforces this — see `writers/run-log.md`).

---

## Example detail page (Mode 1)

```
2026-04-29-mode1-0930

> Mode 1 · 09:30–09:33 IST · OK · 18 signals · 6 items · 0 errors

Sources
- Orbit: 12 signals, OK
- Gmail: 4 signals, OK
- Fathom (enrichment): 0 fetched / 0 references / 0 unresolved, OK

Decisions
- Item 1 — Acme WP redesign, due Fri → assign-task → role-fit WP-dev, only WP-dev free in pod
- Item 2 — Beta SaaS bug overdue 2d → nudge-PM → owner Priya, severity high, no comment 2d
- Item 3 — Gamma renewal email from client → draft-reply → renewal Q asked, template-match exists
- Item 4 — Delta hosting alert from Orbit comment → ack-and-route → ops-pod owns, standard escalation

Skipped
- Gmail: newsletter from Atlassian → skipped → marketing, no project tag, sender on ignore-list
- Orbit: closed task comment "thanks!" → skipped → courtesy comment, no action implied

Uncertain
- Item 5 — Epsilon scope-creep ask → uncertain → candidates: assign-to-Raj, escalate-to-PM — ambiguous between dev work and contract change

↩ Run Log database
↩ Today's queue page
```

---

## Loading verification footer

> If you are reading this file as part of skill execution, you have correctly loaded `schemas/run-log-detail-page.md`. The structure above is the source of truth for every Run Log Detail page. The writer at `writers/run-log.md` produces pages matching this layout; do not deviate.
