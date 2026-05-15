> **This collector uses ONLY the relevant MCP from the 6-MCP allowlist. The allowlist is closed: Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal.**

> **Preflight (`preflight.md`) must have run before this collector is invoked. Do not call any tool until preflight has completed.**

# Slack Collector

## Purpose

Pull only the minimal set of Slack signals that need the PM's attention. Slack is intentionally scoped narrow — this collector does not try to replicate the Slack experience. PMs already read their Slack.

## What we DO collect

1. **Direct messages to the PM** from clients, agencies, AMs, leadership, or team members that are direct asks or flags.
2. **@-mentions of the PM** in project-specific channels or Slack Connect channels with clients.
3. **Client/Agency messages in Slack Connect channels** — any new messages in channels where the PM is a member and the external client has posted.
4. **AM messages tagged to projects** — messages from AMs asking about a specific project the PM owns.
5. **Explicit blockers, approval requests, or escalation language** in project channels the PM is a member of. Heuristics: "blocked", "waiting on you", "need your sign-off", "heads up", "issue with", "urgent", "can you approve".

## What we DO NOT collect

- General channel chatter not related to the PM's projects.
- Bot messages and app notifications.
- Tool integration messages (Orbit, CI/CD, deploys, etc.) — they're noise here.
- Social / watercooler channels.
- Messages in channels the PM is not a member of.
- Messages the PM has already responded to.

## Channel scoping

PMs may be in dozens of channels. Scope this way:

1. **DMs and group DMs** — always scanned.
2. **Slack Connect channels** — channels with external participants (clients). Always scanned if the PM is a member.
3. **Project-specific channels** — channels whose name matches a project the PM follows in Orbit. Best-effort match via channel name containing client name or project name.
4. **AM's own DM threads with the PM** — scanned.
5. **Everything else** — skipped by default. A PM can add explicit channels to their watch list via `PM Task Assignment, change my preference: watch channel #xyz` (Preferences honors an explicit watch list).

## Output shape — per conversation/thread

```
{
  "source": "slack",
  "signal_type": "direct_question_to_pm" | "blocker_flagged" | "approval_request" | "client_message" | "am_coordination" | "escalation",
  "channel": {
    "id": <string>,
    "name": <string>,
    "type": "dm" | "group_dm" | "public_channel" | "private_channel" | "slack_connect"
  },
  "thread_ts": <string or null — if it's a thread>,
  "message_ts": <string>,
  "sender": {
    "slack_id": <string>,
    "name": <string>,
    "is_external": <bool>,
    "category": "client" | "am" | "team" | "leadership" | "external_other" | "unknown"
  },
  "age_hours": <float>,
  "content_summary": <string — short summary of the thread/message>,
  "new_content": <string — exact text of the latest relevant message>,
  "pm_has_replied": <bool>,
  "related_project_guess": <string or null — best-effort project match>,
  "message_url": <string — Slack permalink>,
  "raw_source_data": <full thread>,
  "citation": {
    "type": "slack",
    "label": "Slack message from <sender.name> in <channel.name>, <date>",
    "url": <message_url>,
    "channel_name": <channel.name>,
    "thread_ts": <thread_ts or null>
  }
}
```

## Full thread context

For any flagged thread, pull the full thread via `slack_read_thread`. The PM should not need to jump to Slack to understand what's going on. The matcher and presenter rely on full context.

## Tool calls

- `mcp__...slack.slack_read_user_profile` — confirm the PM's Slack identity once at first run.
- `mcp__...slack.slack_search_public_and_private` — sweep across channels the PM has access to.
- `mcp__...slack.slack_search_users` — resolve sender identities.
- `mcp__...slack.slack_search_channels` — find project-specific channels.
- `mcp__...slack.slack_read_channel` — recent messages from a specific channel.
- `mcp__...slack.slack_read_thread` — full thread context.

## Sender classification

Same four-way classification as Gmail:

- `client` — external Slack user, domain on the external-participants list
- `am` — matches PM's AM list in Preferences
- `team` — WLIQ workspace member who's not an AM or leadership
- `leadership` — Brian, Nishant, or anyone flagged in Preferences
- `unknown` — leave null; matcher handles

## Error handling

| Failure | Behavior |
|---|---|
| Slack MCP unavailable | Return empty signals list with error note. Mode 1 logs on page summary. |
| Individual channel fails | Skip it, continue. |
| PM isn't a member of a channel we tried to read | Skip silently; not a failure. |

## What this collector does NOT do

- Does not send, reply, or react.
- Does not create channels or canvases.
- Does not scrape private DMs between third parties.
- Does not read the full workspace — only channels the PM has access to.
- Does not deduplicate against email or Orbit signals. That's the matcher.
