> **This service uses ONLY the Fathom MCP. Source allowlist — primary collection: Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand: Fathom (this file). Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever — including any that may seem relevant to a specific signal.**

> **Preflight (`preflight.md`) must have run before this service is invoked. Do not call any tool until preflight has completed.**

# Fathom Enrichment Service

## Purpose

Fetch meeting context **on-demand** when a primary signal (mail / Orbit) references a meeting. Fathom is **enrichment-only**: it never originates a row in the Morning Queue and is never invoked unless `synthesis/matcher.md` Job 4b requests enrichment for a specific primary signal.

This is a fundamental shift from earlier versions of this skill. Standalone Fathom signals — action items, missed-meeting alerts, decisions extracted from PM-attended meetings, internal commitments — are **not** surfaced unless a corresponding mail or Orbit signal exists. The matcher does the detection; this service responds.

## When this service is invoked

`synthesis/matcher.md` Job 4b calls this service once per primary signal that contains a meeting reference. Matcher detects references via the trigger phrases below; this service does not scan signals itself.

### Trigger phrases (matcher-side detection — documented here for cross-reference)

A primary signal contains a meeting reference when ANY of the following appear in its body / subject / Orbit comment text:

- Direct call-back: `"per our call"`, `"as discussed"`, `"in the meeting"`, `"on the call"`, `"during our sync"`, `"following our conversation"`, `"per the discussion"`, `"recap from <something>"`
- Meeting-title cite: a known Fathom meeting title appears verbatim
- Attendee cite: `"<attendee name> said"`, `"<attendee name> mentioned"`, `"<attendee name> agreed"`, `"<attendee name> committed to"`
- Recording link: a `fathom.video/<id>` URL appears in the signal
- Date+meeting reference: `"yesterday's call"`, `"Tuesday's meeting"`, `"Friday's sync"`, `"this morning's standup"`

Matcher passes the detected reference plus the originating signal's context (project_id, actor_emails, timestamp) to this service.

## Interface

```
fetch_enrichment(reference, signal_context) -> EnrichmentResult | null
```

**Input — `reference`:**
```
{
  "trigger_phrase": <string — the matched text from the primary signal>,
  "trigger_type": "direct_callback" | "meeting_title" | "attendee_cite" | "recording_link" | "date_reference",
  "extracted_meeting_id": <string or null — populated only if trigger_type = "recording_link">,
  "extracted_meeting_title": <string or null — populated if trigger_type = "meeting_title">,
  "extracted_date_hint": <ISO date or null — populated if trigger_type = "date_reference">,
  "extracted_attendee_name": <string or null — populated if trigger_type = "attendee_cite">
}
```

**Input — `signal_context`:**
```
{
  "primary_signal_id": <string>,
  "primary_signal_source": "gmail" | "orbit",
  "primary_signal_timestamp": <ISO datetime>,
  "project_id_candidates": [<int>, ...],
  "actor_emails": [<string>, ...]
}
```

**Output — `EnrichmentResult` (or `null` if no matching meeting found):**
```
{
  "meeting_id": <string>,
  "meeting_title": <string>,
  "meeting_date": <ISO datetime>,
  "meeting_duration_minutes": <int>,
  "attendees": [
    {
      "name": <string>,
      "email": <string>,
      "is_external": <bool>,
      "is_pm": <bool>
    }
  ],
  "pm_attended": <bool>,
  "summary_excerpt": <string — relevant 2-4 sentence slice of the auto-generated summary, scoped to the primary signal's project/actors>,
  "relevant_action_items": [
    {
      "description": <string>,
      "assignee_name": <string or null>,
      "timestamp_in_recording": <string or null>
    }
  ],
  "recording_url": <string — always present>,
  "citation": {
    "type": "fathom",
    "label": "Fathom meeting: <meeting_title>, <date>",
    "url": <recording_url>,
    "meeting_id": <meeting_id>,
    "timestamp_reference": <string or null — if a specific moment is being cited>
  },
  "match_confidence": "high" | "medium" | "low"
}
```

`relevant_action_items` is **scoped** — only action items whose text overlaps with the primary signal's project name, actor names, or topic keywords. The full action-item list is not returned; that would defeat the enrichment-only contract.

## Lookup strategy

The service picks the cheapest tool call that satisfies the reference:

| `trigger_type` | Strategy | Tool call |
|---|---|---|
| `recording_link` | Direct fetch by meeting ID | `fathom.get_summary(meeting_id)` |
| `meeting_title` | Search by exact title within `signal_context.primary_signal_timestamp ± 14 days` | `fathom.search_meetings(title=..., date_range=...)` |
| `attendee_cite` | Search by attendee name + date window | `fathom.search_meetings(attendee=..., date_range=...)` |
| `direct_callback` | Search by `signal_context.actor_emails` as attendees + date window `[primary_signal_timestamp − 14 days, primary_signal_timestamp]` | `fathom.search_meetings(attendee_emails=..., date_range=...)` |
| `date_reference` | Resolve the date hint to an actual date (e.g. "yesterday" → primary_signal_timestamp − 1 day), then search by `actor_emails` on that date | `fathom.search_meetings(attendee_emails=..., date=...)` |

If multiple meetings match, pick the one with highest overlap of `attendees ∩ signal_context.actor_emails`, ties broken by chronological proximity to `primary_signal_timestamp`. Return `match_confidence: "high"` if only one candidate, `"medium"` if disambiguation by overlap, `"low"` if disambiguation by timing only.

Only call `fathom.get_transcript` if the auto-generated summary clearly fails to cover the reference (rare — auto-summaries are usually sufficient for enrichment context).

## Summary excerpt scoping

Do not return the full Fathom summary. Extract a 2-4 sentence slice that mentions the primary signal's project, the actors involved, or the topic of the trigger phrase. If no such slice can be extracted with reasonable confidence, return the first 2-3 sentences of the summary and set `match_confidence: "low"`.

This keeps Notion rows readable. A row enriched with 8 paragraphs of meeting summary defeats the plain-language rule.

## Tool calls

- `mcp__...fathom.search_meetings` — primary lookup tool. Search by title, attendee, attendee emails, date range.
- `mcp__...fathom.get_summary` — fetch the auto-generated summary for a specific meeting.
- `mcp__...fathom.get_transcript` — full transcript. **On-demand only** when the summary doesn't cover the reference.

Tools NOT used by this enrichment-only model:
- `fathom.list_meetings` — would imply eager collection. Forbidden.
- `fathom.list_team_members` — attendee identity comes from the meeting object itself.

## Error handling

| Failure | Behavior |
|---|---|
| Fathom MCP unavailable / auth expired | Return `null`. The matcher's Job 4b Pass 2 will attempt a gmail-attachment fallback (`collectors/gmail.md` § Transcript fallback) before treating the null as terminal. The Incidents log entry is written once per Mode 1 run **only if both Fathom AND the gmail-attachment fallback failed for at least one signal**, with format: `Fathom enrichment unavailable AND no email-attachment fallback found for N primary signals.` Do NOT trigger the connector-failure fallback chain (Fathom is non-blocking — see `connector-failure-notify.md`). |
| Reference resolves to zero meetings | Return `null`. Matcher proceeds without enrichment for this signal. |
| Reference resolves to a meeting but `get_summary` fails | Return `EnrichmentResult` with `summary_excerpt = null` and `match_confidence: "low"`. Still include recording_url so the PM can watch. |
| Recording URL is missing from the meeting object | Return the result with `recording_url = null`. Matcher writer should flag in AI Notes: "Fathom recording URL unavailable." |

## What this service does NOT do

- Does not scan or list meetings without a matcher-supplied reference.
- Does not originate Morning Queue rows.
- Does not detect missed meetings (no Calendar dependency, no eager meeting list).
- Does not modify Fathom data.
- Does not download recordings.
- Does not create meetings.
- Does not transcribe (relies on Fathom's existing summaries and transcripts).
- Does not deduplicate against Calendar or Orbit.
- Does not return the full meeting summary or full action-item list — only the slice relevant to the primary signal.
