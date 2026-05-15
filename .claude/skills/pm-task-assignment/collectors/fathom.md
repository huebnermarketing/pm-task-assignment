> **This collector uses ONLY the relevant MCP from the 6-MCP allowlist. The allowlist is closed: Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal.**

> **Preflight (`preflight.md`) must have run before this collector is invoked. Do not call any tool until preflight has completed.**

# Fathom Collector

## Purpose

Pull action items, commitments, decisions, and missed-meeting signals from the PM's Fathom meetings in the lookback window. Always include the Fathom recording link so the PM can watch if needed.

## Scope

The PM's Fathom account. Meetings within the lookback window.

## Three meeting categories

### 1. Internal meetings

Team meetings, leadership syncs, internal discussions. Extract:
- **Things asked of the PM** — tasks, updates, deliverables someone requested
- **Things the PM committed to** — promises the PM made on the call
- **Things expected of the PM** — implicit assignments discussed

### 2. External / project meetings

Meetings with clients, agencies, partners. Extract:
- **Client requests** — anything the client asked for
- **Client feedback** — what the client said during review/walkthrough
- **Commitments made to the client** — deliverables promised on the call
- **Decisions made** — scope, direction, timeline agreements
- **Escalations or concerns raised** — issues the client flagged

### 3. Missed meetings

Meetings the PM was invited to but didn't attend (detected by: PM has the event on their calendar, the meeting has a Fathom recording from another attendee, but the PM isn't in the Fathom participant list).

For missed meetings, the content is:
- **A brief summary** — what was discussed, key decisions, action items
- **Anything affecting the PM** — commitments made on their behalf, requests directed at them
- **The recording link** — so they can watch if they need to

## Every meeting includes the Fathom recording link

Non-negotiable. Every signal surfaced from a meeting has the recording URL. PMs don't need to watch every recording, but the link must be present.

## Output shape — per meeting

```
{
  "source": "fathom",
  "signal_type": "action_item_for_pm" | "action_item_for_team_member" | "client_request" | "client_commitment_made" | "decision_made" | "internal_expectation" | "escalation_raised" | "missed_meeting",
  "meeting_id": <string>,
  "meeting_title": <string>,
  "meeting_date": <ISO datetime>,
  "meeting_duration_minutes": <int>,
  "meeting_type": "internal" | "external_client" | "external_partner" | "missed",
  "attendees": [
    {
      "name": <string>,
      "email": <string>,
      "is_external": <bool>,
      "is_pm": <bool>
    }
  ],
  "pm_attended": <bool>,
  "fathom_summary": <string — the full auto-generated summary>,
  "action_items_extracted": [
    {
      "description": <string>,
      "assignee_name": <string or null>,
      "timestamp_in_recording": <string or null>,
      "is_for_pm": <bool>
    }
  ],
  "decisions_made": [<string>, ...],
  "commitments_to_client": [<string>, ...],
  "recording_url": <string — always present>,
  "full_transcript_available": <bool>,
  "raw_source_data": <full meeting object>,
  "citation": {
    "type": "fathom",
    "label": "Fathom meeting: <meeting_title>, <date>",
    "url": <recording_url>,
    "meeting_id": <meeting_id>,
    "timestamp_reference": <string or null — if citing a specific moment in the recording>
  }
}
```

## One meeting may produce multiple signals

A single 30-minute internal meeting that discussed Agency X, Agency Y, and Agency Z should split into three items — one per project. The matcher handles the split downstream, but the collector can emit multiple signals per meeting if the auto-extracted action items clearly split by topic.

## Tool calls

- `mcp__...fathom.list_meetings` — list the PM's meetings in the lookback window.
- `mcp__...fathom.search_meetings` — alternative search by keyword or attendee.
- `mcp__...fathom.get_summary` — the auto-generated meeting summary.
- `mcp__...fathom.get_transcript` — full transcript (on-demand only, when the summary is insufficient for a specific item).
- `mcp__...fathom.list_team_members` — to resolve attendee identities.

## Missed meeting detection

Use the Calendar collector's output (once built) to compare against Fathom participant lists. In the MVP, Calendar is out of scope — so detect missed meetings by cross-referencing Fathom recordings the PM has access to against the PM's own attendance (via Fathom's participant list).

This is a partial mechanism. If needed, the PM can manually flag missed meetings by editing the row.

## Attendee classification

Use the Orbit relationship map to classify each attendee:

- External client or agency contact → client
- AM from Preferences → am
- WLIQ team member → team
- Brian / Nishant → leadership
- Unknown → unknown (matcher handles)

## Error handling

| Failure | Behavior |
|---|---|
| Fathom MCP unavailable | Return empty signals list with error note. |
| Individual meeting fetch fails | Skip it. Continue with others. |
| Recording URL is missing | Still emit the signal but without the URL. Flag in AI Notes: "Fathom recording URL was unavailable." |

## What this collector does NOT do

- Does not modify Fathom data.
- Does not download recordings.
- Does not create meetings.
- Does not transcribe anything itself (relies on Fathom's existing summaries and transcripts).
- Does not deduplicate against Calendar or Orbit.
