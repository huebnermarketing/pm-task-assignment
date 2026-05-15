> **This executor uses ONLY the relevant MCP from the 6-MCP allowlist. The allowlist is closed: Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever.**

> **Preflight (`preflight.md`) must have run before this executor is invoked.**

# Slack Executor

## Purpose

Send Slack messages on the PM's behalf. Used for three main flows:

1. **Team-member handoffs** — giving an assignee their context after Mode 2 approves the item
2. **AM pings** — notifying the AM that an action has been taken (e.g., "Vijay has this now")
3. **PM self-summary** — the Mode 2 completion recap sent to the PM themselves

Plus the escalation message in Mode 3 (if backup channel = Slack).

## Rule

**Internal team messages are sent on PM approval.** The approval gesture is flipping a row to `Approved` or providing a PM note that doesn't say "save as draft" or "don't send".

Client-facing Slack messages (in Slack Connect channels with external clients) are RARE and require explicit PM approval via an `Approved` row. Even then, the skill should prefer to suggest PM writes it themselves rather than auto-sending.

## Supported operations

### Send a message

Use `mcp__...slack.slack_send_message`.

Required parameters:
- `channel` — channel ID, user DM ID, or channel name (resolve to ID)
- `text` — the message body (Slack mrkdwn or plain text)

Optional:
- `thread_ts` — if replying within a thread

### Send a message draft (save for review)

Use `mcp__...slack.slack_send_message_draft` if available, OR save the message to a designated Slack Canvas the PM can review later.

For MVP, use drafts only when PM explicitly says "save as draft" in their note.

## Message content rules

### Team member handoff messages

The big one. This is how the PM's pod members get their morning's work.

- **Language:** plain-language 4th-5th grade English per `writers/plain-language.md`. Role-specific technical terms preserved.
- **Tone:** simple, direct, warm but not casual. Short sentences.
- **Structure (suggested template):**

```
<project name / brief task title>

What to do: <one sentence, plain English>

Why: <one sentence — why this matters or who's waiting>

Context you need: <link to Orbit task, Figma, staging URL, file>

Please log your hours.
```

Example:

```
Agency X — Homepage Revisions

What to do: Apply the 12 revisions from the client feedback PDF. Hero image, navigation, two footer sections, pricing table.

Why: Client is presenting to their board next Thursday. They need the updates started today.

Context you need:
  • Orbit task: [link]
  • Feedback PDF: [attached to task]
  • Staging: [URL]
  • Figma: [URL]

Please log your hours.
```

### AM pings

- **Language:** normal professional English.
- **Tone:** concise, confident.
- **Length:** 1-3 lines max.

Example:

```
Agency X homepage revisions picked up today — Vijay is on it. Target: preview ready before the Thursday board meeting. Will keep you posted.
```

### PM self-summary (post-Mode 2)

- **Language:** normal English.
- **Structure:** bullet list of actions taken / skipped / failed. See the template in `modes/mode-2-execution.md` step 9.

### Escalation message (from Mode 3)

- Template from `modes/mode-3-escalation.md`.
- Sent to the backup person.

## Channel / DM resolution

When the PM note or matched context specifies a recipient:

- **Individual person** → DM them directly. Resolve the Slack user ID via `slack_search_users` with their name.
- **AM** → DM. Use the handle from Preferences.
- **Multiple people** → send one DM to each OR one group DM (pick group DM for same-team context).
- **Project channel** → find by name. Channels usually match the Orbit client or project name with slight variations. Use `slack_search_channels`.
- **Unknown recipient** → skip and log: `FAILED — couldn't find Slack handle for [name]. Check their Preferences or send manually.`

## Slack formatting

Use Slack mrkdwn:
- `*bold*` (single asterisks, not double)
- `_italic_`
- Bullet lists with `•` or `-`
- Links: `<URL|display text>`
- Code blocks for task IDs or URLs: backticks

Keep messages scannable. No walls of text.

## Source citation

If the Slack handoff references a document (PDF, image, PPT) that was read to produce it, include the filename in the context section:

```
Context you need:
  • Orbit task: [link]
  • Feedback PDF: `homepage_revision_feedback.pdf` — attached to the Orbit task
  • Staging: [URL]
```

Per `writers/source-citation.md`, documents that were read MUST be cited.

## After sending

Return to Mode 2 with the outcome string for the row:
- `Slack sent to [person]`
- `Slack sent to [person] + [person] (AM notified)`
- `Slack draft saved (not sent per your note)`

## Error handling

| Failure | Behavior |
|---|---|
| Slack MCP unavailable | Log in Outcome: `FAILED — Slack unavailable.` Skip this op. |
| Recipient handle invalid | Log: `FAILED — couldn't find [name]'s Slack. Check Preferences or send manually.` |
| Channel not accessible to the PM | Log and skip. |
| Message too long | Truncate with a "full version in Orbit task" pointer; prefer shorter handoffs anyway. |

## What this executor does NOT do

- Does not send messages impersonating someone else.
- Does not post to channels the PM isn't a member of.
- Does not auto-send in Slack Connect channels to external parties unless explicitly approved.
- Does not react with emojis on behalf of the PM.
- Does not scrape reactions or read receipts.
- Does not create channels.
- Does not modify messages after sending.
