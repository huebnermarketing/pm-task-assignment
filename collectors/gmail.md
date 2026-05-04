# Gmail Collector

> **This collector uses ONLY the Gmail MCP. Source allowlist — primary collection: Orbit, Gmail, Slack, Fathom, Notion. Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). Nothing else, ever.**

> **Preflight (`preflight.md`) must have run before this collector is invoked.**

## Purpose

Pull overnight email signals from the PM's WLIQ Gmail account that need their attention. Return a structured list of email items ready for `synthesis/matcher.md`.

## Scope — single account only

**The collector reads ONLY the PM's `@whitelabeliq.com` Gmail account.** Even if the PM has multiple Google accounts authenticated in their Gmail MCP (e.g., personal Gmail, an AM's account they have delegated access to, an old account), this collector touches only the WLIQ work account.

How the skill identifies the right account:

1. Read the PM's canonical email and aliases from Preferences (Identity section)
2. Use the Gmail MCP's profile or account-selector to pin the search to the WLIQ account
3. If the Gmail MCP doesn't natively support per-account selection, filter results to messages where the To/From/CC contains either the canonical email or any alias

If neither approach works (rare), notify the PM via `connector-failure-notify.md` and skip Gmail for this run.

## Aliases — both directions

Email aliases listed in Preferences are treated as the same identity for both sender classification AND inbox scoping:

- A message addressed to an alias address (example: `aditi@whitelabeliq.com`) is considered to have arrived at the canonical address (example: `aditis@whitelabeliq.com`) too — both pull the same signal
- A message FROM any of the PM's alias addresses is considered FROM the PM (so it's filtered out as self-noise, not as inbound)

## Window

- **Default lookback:** 12–18 hours (overnight)
- **Extended lookback:** if Preferences' `last_morning_run` is older than 24 hours, lookback = (now − last_morning_run), capped at 7 days

## What to pull

Run targeted searches — not a full inbox dump. All searches scoped to the WLIQ account only.

1. **Emails where the PM has not replied** — `gmail_search_messages` with query `is:unread OR (in:inbox -from:me)` limited to the lookback window. Filter further on the result set to identify threads where the PM's email isn't the last message.
2. **Starred or flagged emails** — `is:starred after:<lookback>`. PM marked these for follow-up.
3. **Emails with attachments from client/agency senders** — `has:attachment after:<lookback>` filtered to senders matching known Orbit clients.
4. **Threads with recent replies** — threads where the newest message is after `last_morning_run` AND the PM has previously participated. Use `gmail_read_thread` to verify.
5. **Unsent drafts** — `gmail_list_drafts`. Carry-forward candidates.

## What to skip

- **Orbit notification emails** — already covered by the Orbit collector. Detect via sender domain matching Orbit's notification-from address.
- **Newsletters and marketing emails** — detect via `list-unsubscribe:` header, `noreply@` senders, well-known marketing domains.
- **Automated tool notifications** — GitHub, CI/CD alerts, Slack digests, calendar notifications.
- **Internal WLIQ blasts** that aren't addressed to the PM directly.
- **Messages from the PM's other authenticated accounts** (personal Gmail, etc.) — out of scope per the single-account rule.

## Sender classification

Use the Orbit relationship map (from the Orbit collector) to classify each sender:

| Category | How to identify |
|---|---|
| Client / Agency contact | Sender email domain matches an Orbit client's domain, OR sender matches an Orbit client contact |
| Account Manager | Sender matches Preferences' AM list |
| Internal team | `@whitelabeliq.com` and not an AM and not leadership |
| Leadership | Brian Gerstner, Nishant Rana, or anyone flagged leadership in Preferences |
| Self | Matches PM's canonical email or any alias — these are filtered out (not signal) |
| Unknown | Leave classification null; matcher will handle |

## Output shape — per email/thread

```
{
  "source": "gmail",
  "signal_type": "client_awaiting_response" | "starred_unaddressed" | "attachment_received" | "thread_updated" | "unsent_draft" | "am_message" | "team_request" | "leadership_message",
  "thread_id": <string>,
  "message_id": <string — latest message in thread>,
  "sender": {
    "name": <string>,
    "email": <string>,
    "domain": <string>,
    "category": "client" | "am" | "team" | "leadership" | "unknown"
  },
  "subject": <string>,
  "age_hours": <float — hours since the latest message>,
  "thread_depth": <int — number of messages in thread>,
  "thread_summary": <string — a brief summary of what the thread is about>,
  "pm_last_message_excerpt": <string or null — the PM's last message if they've participated>,
  "new_content": <string — the latest message body>,
  "attachments": [
    {
      "filename": <string>,
      "mime_type": <string>,
      "size_bytes": <int>,
      "download_url": <string>,
      "summary": <string or null>
    }
  ],
  "thread_url": <string — Gmail link>,
  "has_unsent_draft": <bool>,
  "draft_excerpt": <string or null>,
  "raw_source_data": <full thread object>,
  "citation": {
    "type": "gmail",
    "label": "Email from <sender.name>, <date>, subject: <subject>",
    "url": <thread_url>,
    "thread_id": <thread_id>
  }
}
```

## Full thread context

For every email flagged, pull the full thread via `gmail_read_thread`. The PM should never have to leave Notion to understand context.

## Attachment handling

- Record every attachment's filename and MIME type.
- Pull the `download_url` directly from Gmail attachment metadata.
- Don't rely on Gmail's auto-summary (usually unavailable).
- Matcher routes attachments through `writers/source-citation.md` for inclusion in row detail. If the assignee needs the file's actual content, the executor downloads via curl and Claude reads natively.

## Post-collection action — mark read / apply label

After a message has been successfully ingested into a Morning Queue row, the collector may mutate the source Gmail message based on the PM's `gmail_post_collection_action` Preference (default `none`):

| Setting                       | Action                                                                                       |
| ----------------------------- | -------------------------------------------------------------------------------------------- |
| `none` (default)              | Do nothing. Source message untouched.                                                        |
| `mark_read`                   | Remove the `UNREAD` label from the message.                                                  |
| `apply_label`                 | Apply the configured `Gmail label name` (default `pm-task-assignment/collected`). Create the label first if it does not yet exist.       |
| `mark_read_and_apply_label`   | Both of the above.                                                                           |

Run this step only after the row has been written to Notion successfully. If the row write fails, do not mutate the Gmail message — the source signal must remain visible to the PM until it is recorded.

If label creation or label application fails (Gmail MCP error), log the failure on the row's `AI Notes` and do not retry; the row already exists in Notion so the signal is captured. Mark-read failures follow the same rule.

## Tool calls (only Gmail MCP)

- `mcp__...gmail.gmail_get_profile` — confirm the PM's WLIQ email at preflight
- `mcp__...gmail.gmail_search_messages` — primary sweep
- `mcp__...gmail.gmail_read_thread` — full thread context
- `mcp__...gmail.gmail_read_message` — individual message if needed
- `mcp__...gmail.gmail_list_drafts` — for unsent drafts
- `mcp__...gmail.list_labels` — used during `apply_label` to detect whether the configured label already exists
- `mcp__...gmail.create_label` — used during `apply_label` to create the configured label on first use
- `mcp__...gmail.label_message` / `mcp__...gmail.label_thread` — used during `apply_label` to attach the label to the source message or thread
- `mcp__...gmail.unlabel_message` / `mcp__...gmail.unlabel_thread` — used during `mark_read` to remove the `UNREAD` label

No other MCP. Period.

## Error handling

| Failure | Behavior |
|---|---|
| Gmail MCP unavailable | Route to `connector-failure-notify.md`. Return empty signals list. Mode 1 logs the gap on the page summary. |
| Wrong account being read | Halt this collector. Notify PM via `connector-failure-notify.md`: "Gmail collector defaulted to the wrong account. Check that your WLIQ account is selected in your Gmail MCP." Skip Gmail for this run. |
| Specific thread fetch fails | Skip that thread. Continue with others. |
| Attachment metadata missing | Still include the signal. Note attachment can't be auto-included. |

## Privacy

- Read ONLY the PM's WLIQ inbox. Never any other mailbox.
- Don't persist email content outside Notion pages the PM owns.
- Aliases are treated as the same identity but DO NOT extend access — the skill still only reads the canonical account, just matches against alias addresses too.

## What this collector does NOT do

- Does not send, reply, forward, or draft emails.
- Does not mark emails as read or apply labels unless the PM enables it via `gmail_post_collection_action` in Preferences.
- Does not archive.
- Does not synthesize or group. That's the matcher.
- Does not filter by importance or urgency. All relevant signals are returned.
- Does not access any Gmail account other than the PM's WLIQ work account.
- Does not call any MCP other than Gmail.
