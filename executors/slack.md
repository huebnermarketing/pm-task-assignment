> **This executor uses ONLY the Slack MCP, and only for outbound-send. Source allowlist — primary collection: Orbit, Gmail, Notion. Enrichment-on-demand: Fathom (lazy fetch via `collectors/fathom.md`). Slack is NOT a collection source (the Slack collector was removed) — this executor's sole purpose is to send 2 specific kinds of message via Slack when the PM explicitly opted in. Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). No other MCP, ever.**

> **Preflight (`preflight.md`) must have run before this executor is invoked.**

# Slack Executor — outbound-send only

## Purpose

Slack is outbound-send only. The skill makes EXACTLY TWO kinds of Slack call, each gated by an explicit PM opt-in:

1. **Team handoff send** — Mode 2 sends a team handoff via Slack ONLY when the row's PM Note explicitly says `send` AND audience = `team`. Recipient is the row's resolved delegate (the matcher's `recommended_assignee`, or the PM's override via note).
2. **AM ping send** — Mode 2 sends an AM ping via Slack ONLY when the row's PM Note explicitly says `send` AND audience = `am`. Recipient is the row's AM (matched to a Preferences AM entry).

Without an explicit `send` note (or with an audience mismatch), the executor does NOT call Slack at all. The handoff/ping is rendered as a draft block by `writers/notion.md` and appended under the row's Outcome on today's dated page (see `Notion-appended drafts` below). The PM copies and sends it themselves through whatever channel they prefer.

The skill explicitly does NOT use Slack for: PM self-summary (now a Notion callout block at the top of today's dated page), escalation backup ping (now email-only via `executors/email.md`), connector-failure tier 1 (now email-sent), client-facing messages (always Notion draft, PM sends manually). Slack is not a collection source either — the `collectors/slack.md` file was deleted.

## Rule

**Team and AM handoffs default to Notion drafts.** The Slack send paths above are exceptions, only invoked when the PM has explicitly opted in via the PM Note. Drafts on Notion are the default and apply to every row that doesn't meet the explicit-send rule.

Client-facing communication is never auto-sent. Always drafted to Notion. The PM owns the send.

## Supported operations

### Send a message (only the 2 paths above)

Use `mcp__...slack.slack_send_message`. Allowed only for: team-handoff rows with explicit `send` + audience=team, and AM-ping rows with explicit `send` + audience=am.

Required parameters:
- `channel` — channel ID, user DM ID, or channel name (resolve to ID)
- `text` — the message body (Slack mrkdwn or plain text)

Optional:
- `thread_ts` — if replying within a thread

### Notion-appended drafts (default path for team / AM)

For every team or AM row that the executor does NOT send via the rules above, build the message body using the PM's `Handoff template` from Preferences and hand the result to `writers/notion.md`. The writer appends a `Handoff draft (copy + paste)` block under the row's Outcome on today's dated page, including:

- Audience tag (`Team`, `AM`, or `Client`)
- Recipient name + canonical email + optional Slack handle (so the PM can deliver through whatever channel they prefer)
- The message body, in plain language per `writers/plain-language.md`

No Slack API call happens for this path. The PM reads the draft on the dated page and copies it through whatever delivery channel they use.

## Message content rules

### Team member handoff messages (drafts on Notion by default)

How the PM's pod members get their morning's work — drafted to Notion under the row's Outcome, copied by the PM and delivered through whatever channel they use (in-person, chat, email). Real Slack send only on the team-handoff exception (explicit `send` note, audience = team).

- **Language:** plain-language 4th-5th grade English per `writers/plain-language.md`. Role-specific technical terms preserved.
- **Tone:** simple, direct, warm but not casual. Short sentences.
- **Template:** taken from the PM's `Handoff template` in Preferences. The default suggestion is documented there (and editable per PM):

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

### AM messages (drafts on Notion by default; Slack send only on explicit opt-in)

By default, the skill drafts AM messages to Notion. The Slack send path fires only when the PM left an explicit `send` note + audience = am on the row.

- **Language:** normal professional English.
- **Tone:** concise, confident.
- **Length:** 1-3 lines max.

Example body (default Notion draft; the PM copies and delivers however they choose):

```
Agency X homepage revisions picked up today — Vijay is on it. Target: preview ready before the Thursday board meeting. Will keep you posted.
```

When the PM opts into Slack send, the body is identical; the executor calls `slack_send_message` to the AM's Slack handle (resolved via Preferences `AM Slack handle` field) instead of leaving a draft on the dated page.

## Channel / DM resolution

When the PM note + audience match opts into a Slack send:

- **Team member (audience = team)** → DM the delegate directly. Resolve the Slack user ID via `slack_search_users` against the delegate's canonical email (resolved from Pod Matrix / Orbit relationship map). The dated page's Slack-handles reference block (rendered by `writers/notion.md` near the morning queue DB) is the secondary lookup if the Slack search returns no match.
- **AM (audience = am)** → DM the AM. Use the `AM Slack handle` from the Preferences AM entry. If the handle field is blank for that AM, fall back to a Notion draft for that row and log `am_handle_missing` to the Run Log; the PM will see the draft on the dated page and can copy + send manually.
- **Unknown recipient** → skip the send and fall back to a Notion draft. Log: `FAILED — couldn't find Slack handle for [name]; draft appended for you to copy.`

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
- `Sent handoff to [person] on Slack (@handle) per your 'send' note` — when the team-handoff send path ran
- `Sent AM ping to [AM name] on Slack (@handle) per your 'send' note` — when the AM-ping send path ran
- `Handoff draft appended to Notion page (audience: team — copy + paste to send)` — default path
- `Handoff draft appended to Notion page (audience: AM — copy + paste to send)` — default path

## Pod Daily Task block (end-of-Mode-2 digest)

Separate from the per-row drafts above. After all rows are processed, Mode 2 Step 9a builds a single **Pod Daily Task** copy-block at the bottom of the dated page — a concatenated list of every sub-task created that day, formatted to be pasted directly into whatever channel the PM uses for pod-wide daily task delivery (their choice). This is **not** a Slack-API send; the executor does not touch Slack for the digest. The build itself is handed to `writers/notion.md` — Flow — appending the Pod Daily Task block. Configuration lives in `schemas/preferences-page.md` — Pod Daily Task Block.

## AM Daily Ping block (end-of-Mode-2 digest)

A second end-of-Mode-2 copy-block sits directly below the Pod Daily Task block on the dated page. Mode 2 Step 9b builds **one 3-line ping per AM**, grouped by which projects each AM owns. This block is **never** Slack-API-sent — every line is a draft for the PM to copy and deliver through whatever channel they use for that AM (DM, email, in-person, etc.).

This complements (does not replace) the per-row AM drafts described above. Per-row drafts capture row-specific context under each row's Outcome on the dated page; the AM Daily Ping block is a single consolidated message per AM, ready to paste.

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
