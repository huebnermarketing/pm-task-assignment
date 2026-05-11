> **This executor uses ONLY the Slack MCP (and the Notion MCP indirectly via `writers/notion.md` for draft append). Source allowlist — primary collection: Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever.**

> **Preflight (`preflight.md`) must have run before this executor is invoked.**

# Slack Executor

## Purpose

Two real Slack-send paths the skill is allowed to take, plus a draft-into-Notion path that replaces all team / AM auto-sends:

1. **PM self-summary** — the Mode 2 completion recap sent to the PM themselves (operational, internal).
2. **Escalation message** from Mode 2 Step 3a (sent to the configured backup if their channel is Slack).
3. **Team handoff with explicit PM `send` instruction** — only when the row's PM Note explicitly says "send" AND audience = `team`. Never for AM, never for client.

For team and AM rows that don't meet path 3, the executor does NOT call Slack at all. Instead it produces a draft body and hands it to `writers/notion.md` to append under the row's Outcome on today's dated page (see `Notion-appended drafts` below). The PM copies and sends it themselves.

## Rule

**Team and AM Slack messages are never auto-sent unless path 3 above applies.** Drafts are appended to today's dated Notion page under the row's Outcome — the PM reads them there and copies into Slack on their own.

Client-facing Slack messages (in Slack Connect channels with external clients) are never auto-sent. Always drafted to Notion. The PM owns the send.

## Supported operations

### Send a message (only the 3 paths above)

Use `mcp__...slack.slack_send_message`. Allowed only for: PM self-summary, escalation backup ping, and team-handoff rows with explicit `send` PM note.

Required parameters:
- `channel` — channel ID, user DM ID, or channel name (resolve to ID)
- `text` — the message body (Slack mrkdwn or plain text)

Optional:
- `thread_ts` — if replying within a thread

### Notion-appended drafts (default path for team / AM)

For every team or AM row that the executor does NOT send via the rules above, build the message body using the PM's `Slack handoff template` from Preferences and hand the result to `writers/notion.md`. The writer appends a `Slack draft (copy to send)` block under the row's Outcome on today's dated page, including:

- Audience tag (`Team`, `AM`, or `Client`)
- Recipient name + Slack handle (so the PM can DM directly)
- The message body, in plain language per `writers/plain-language.md`

No Slack API call happens for this path. The PM reads the draft on the dated page and copies it into Slack themselves.

## Message content rules

### Team member handoff messages (drafts on Notion by default)

How the PM's pod members get their morning's work — drafted to Notion under the row's Outcome, copied by the PM into Slack manually. Real Slack send only on the path-3 exception (explicit `send` note, audience = team).

- **Language:** plain-language 4th-5th grade English per `writers/plain-language.md`. Role-specific technical terms preserved.
- **Tone:** simple, direct, warm but not casual. Short sentences.
- **Template:** taken from the PM's `Slack handoff template` in Preferences. The default suggestion is documented there (and editable per PM):

```
<project name / brief task title>

What to do: <one sentence, plain English>

Why: <one sentence — why this matters or who's waiting>

Context you need: <link to Orbit task, Figma, staging URL, file>

Please log your hours.
```

Example body the executor produces (for appending to the dated Notion page):

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

### AM messages (drafts on Notion only — never auto-sent)

The skill never pings AMs automatically. Every AM-bound message is built here and handed to `writers/notion.md` for append under the row's Outcome.

- **Language:** normal professional English.
- **Tone:** concise, confident.
- **Length:** 1-3 lines max.

Example body (the PM copies this from Notion into Slack on their own):

```
Agency X homepage revisions picked up today — Vijay is on it. Target: preview ready before the Thursday board meeting. Will keep you posted.
```

### PM self-summary (post-Mode 2)

- **Language:** normal English.
- **Structure:** bullet list of actions taken / skipped / failed. See the template in `modes/mode-2-execution.md` step 9.

### Escalation message (from Mode 2 Step 3a)

- Template lives in `modes/mode-2-execution.md` Step 3a.3.
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

## After sending or drafting

Return to Mode 2 with the outcome string for the row:
- `Slack sent to [person]` — when path 1, 2, or 3 ran
- `Slack draft appended to Notion page (audience: team — copy from today's queue page to send)`
- `Slack draft appended to Notion page (audience: AM — copy from today's queue page to send)`

## Pod Daily Task block (end-of-Mode-2 digest)

Separate from the per-row drafts above. After all rows are processed, Mode 2 Step 9a builds a single **Pod Daily Task** copy-block at the bottom of the dated page — a concatenated list of every task created or reassigned that day, formatted to be pasted directly into the pod's daily task Slack channel. This is **not** a Slack-API send; the executor does not touch Slack for the digest. The build itself is handed to `writers/notion.md` — Flow — appending the Pod Daily Task block. Configuration lives in `schemas/preferences-page.md` — Pod Daily Task Block.

## AM Daily Ping block (end-of-Mode-2 digest)

A second end-of-Mode-2 copy-block sits directly below the Pod Daily Task block on the dated page. Mode 2 Step 9b builds **one 3-line ping per AM**, grouped by which projects each AM owns. This block is **never** Slack-API-sent — every line is a draft for the PM to copy into a Slack DM to that AM themselves.

This complements (does not replace) the per-row AM drafts described above. Per-row drafts capture row-specific context under each row's Outcome on the dated page; the AM Daily Ping block is a single consolidated message per AM, ready to paste as a DM.

The per-AM 3-line body is drafted by `writers/plain-language.md` (called from `writers/notion.md` during the AM block build), not by this executor. This executor only documents the block's existence and the fact that no Slack API call ever fires for it. Configuration: `schemas/preferences-page.md` — AM Daily Ping Block. Build flow: `writers/notion.md` — Flow — appending the AM Daily Ping block.

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
