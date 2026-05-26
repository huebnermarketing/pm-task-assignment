# Reference — External Document Access (read-only)

Some signals collected from the primary sources (Gmail / Fathom / Orbit / Notion) reference documents stored elsewhere — typically a Google Drive file, a Google Doc, a Google Sheet, or a SharePoint document. The skill MAY fetch those documents read-only so the row's context (and any task body produced from it) is grounded in the actual content, not just the link.

This file defines when and how. It does NOT extend the primary collection scope — these MCPs are never used to scan or list, only to fetch a specific resource that an allowlisted primary signal already pointed at.

## When the skill MAY use these MCPs

All four conditions must hold:

1. The trigger comes from an allowlisted primary source (Gmail / Fathom / Orbit / Notion).
2. The trigger explicitly references the document — e.g., a link in an email body, an action item from a Fathom transcript citing a Drive file, an Orbit task description with a Drive URL, a Preferences page link.
3. The document content is needed to fill in row detail context, the proposed Orbit task body, or the proposed handoff draft. If the link is just incidental (e.g., a project-folder URL the PM never opens), do not fetch.
4. The MCP for that document type is authenticated and responsive. If it is not authenticated, log on the row's `AI Notes` (`Could not fetch [filename] — [MCP] not authenticated`) and continue.

## What is allowed

| MCP                           | Operations allowed                                                                                                              |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Google Drive (read-only)      | `list metadata` and `read content` of a SPECIFIC file referenced by an allowlisted primary signal. No drive-wide search or list. |
| Google Docs (read-only)       | `read content` of a specific Google Doc referenced by a primary signal.                                                         |
| Google Sheets (read-only)     | `read content` of a specific tab / range in a Google Sheet referenced by a primary signal.                                      |
| SharePoint (read-only)        | `read content` of a specific SharePoint document referenced by a primary signal. **Scope details TBD — confirm during retail review.** |

## What is forbidden

- No write of any kind — no create, no edit, no comment, no permission change.
- No drive-wide / site-wide listing or search. Only fetch the specific file the primary signal pointed at.
- No fetch of files NOT referenced by an allowlisted primary signal — no "I noticed this related doc, let me also pull it."
- No persistence of the document outside today's dated Notion page. The PM owns the data; the skill is just routing context.

## Where the content lands

When the skill reads an external document for a row's context, it cites the read in three places:

1. **Row detail page → `Sources` heading → `Document read` subsection** — per `schemas/row-detail-page.md`. Includes filename, source MCP, the date it was read by the skill, and a one-paragraph summary of what was extracted.
2. **Orbit task body → REFS section** — short citation link. If the document was read for context, the citation explicitly says so. Per `writers/source-citation.md`.
3. **Handoff draft → "Context you need" list** — filename + a pointer back to the originating signal (the email it was attached to, the Fathom call it was mentioned in, or the Orbit task that linked it).

## Auditability

Every external doc read produces a Run Log entry `external_doc_read` with: file ID, file name, source MCP, originating primary signal, row ID, and timestamp. This is how the post-mortem and the PM verify the skill stayed inside scope.

## Failure handling

| Failure                                              | Behavior                                                                                                                |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| MCP not authenticated                                | Log on row `AI Notes`. Skip the fetch. Continue building the row from primary signals.                                  |
| File not accessible to the PM                        | Log: `Document [filename] — access denied; file is not shared with you.`                                                |
| File too large or paginated                          | Read first page or first N tokens; log truncation in `AI Notes`; never fall back to summarization-by-other-tool.        |
| MCP returns a transient error                        | Apply the same retry-with-backoff policy as primary MCPs (see `connector-failure-notify.md`). Log every retry attempt.  |

## What this reference does NOT cover

- File modification of any kind (the skill is strictly read-only against these MCPs).
- General-purpose search across Drive / Docs / Sheets / SharePoint. Always navigate from a primary signal.
- Signals originating in these MCPs. They are reference-only — the skill does NOT collect signals from them.
- Attachments inline in Gmail or Orbit (those are handled by the Gmail / Orbit collectors directly via download-and-read, not via this reference).
