# Gmail Collector

> **This collector uses ONLY the Gmail MCP. Source allowlist — primary collection: Orbit, Gmail, Notion (Slack forbidden; Fathom forbidden as standalone source). Enrichment-on-demand: Fathom (lazy fetch via `collectors/fathom.md` when a primary signal references a meeting). Read-only references on demand: Google Drive/Docs/Sheets, SharePoint (see `references/external-doc-access.md`). Nothing else, ever.**

> **Preflight (`preflight.md`) must have run before this collector is invoked.**

## Purpose

Pull overnight email signals from the PM's WLIQ Gmail account that need their attention. Return a structured list of email items ready for `synthesis/matcher.md`.

**Dual role.** Gmail signals are NOT only standalone new-requirement signals. They also serve as **context enrichment** for Orbit signals — especially priority-lane signals from the Orbit collector's Priority Pass. A Gmail thread between the AM and the PM about the same project may not be a new actionable item on its own (no fresh ask, no new deliverable), but it IS the backstory the PM needs to brief the delegate. To support this, every Gmail signal carries cross-link metadata that `synthesis/matcher.md` Job 4b uses to attach it to corroborating Orbit signals. See `## Context-link metadata` below.

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
- **Monday override (IST):** if today is Monday, force `lookback = max(now − last_morning_run, 72 hours)`. Cron is weekday-only — Monday must always cover Fri/Sat/Sun even if a manual weekend run reset `last_morning_run`.

## What to pull

Run targeted searches — not a full inbox dump. All searches scoped to the WLIQ account only.

1. **Emails where the PM has not replied** — `gmail_search_messages` with query `is:unread OR (in:inbox -from:me)` limited to the lookback window. Filter further on the result set to identify threads where the PM's email isn't the last message.
2. **Starred or flagged emails** — `is:starred after:<lookback>`. PM marked these for follow-up.
3. **Emails with attachments from client/agency senders** — `has:attachment after:<lookback>` filtered to senders matching known Orbit clients.
4. **Threads with recent replies** — threads where the newest message is after `last_morning_run` AND the PM has previously participated. Use `gmail_read_thread` to verify.
5. **Unsent drafts** — `gmail_list_drafts`. Carry-forward candidates.
6. **Orbit notification mails (fixed-cost lane + general lane)** — `from:<Orbit
   notification-from address> after:<lookback>`. Always collected in the lookback window,
   then resolved via the two-tier project match in § Orbit-notification mails (fixed-cost
   lane + general lane): a tracked-project match tags `origin: fixed_cost_mail`, a
   general-project match (against the Orbit relationship map) tags
   `origin: orbit_notification_mail`, and no match on either tier skips the mail (not an
   error). `registry_snapshot` being absent/empty only narrows the tracked-tier match — it
   does not disable general-tier collection.

## What to skip

- **Orbit notification emails** — skipped for the workload lane (already covered by the
  Orbit collector's own activity-log pull), NOT skipped for the mail-parser lane: every
  Orbit notification mail (sender domain matching Orbit's notification-from address) is
  collected and resolved per § Orbit-notification mails (fixed-cost lane + general lane)
  below — tracked-project matches become `origin: fixed_cost_mail` (spec §11.2), non-tracked
  but resolvable matches become `origin: orbit_notification_mail`, and mails matching neither
  tier are skipped (not an error, not "excluded by design" — simply unresolvable).
- **Newsletters and marketing emails** — detect via `list-unsubscribe:` header, `noreply@` senders, well-known marketing domains.
- **Automated tool notifications** — GitHub, CI/CD alerts, third-party chat digests, calendar notifications.
- **Internal WLIQ blasts** that aren't addressed to the PM directly.
- **Messages from the PM's other authenticated accounts** (personal Gmail, etc.) — out of scope per the single-account rule.

## Sender + recipient classification

Use the Orbit relationship map (from the Orbit collector) to classify each sender AND every
recipient (To + CC) on the thread — same table, same identity sources, applied uniformly:

| Category | How to identify |
|---|---|
| Client / Agency contact | Address's email domain matches an Orbit client's domain, OR the address matches an Orbit client contact |
| Account Manager | Address matches Preferences' AM list (canonical email or alias) |
| Internal team | `@whitelabeliq.com` and not an AM and not leadership |
| Leadership | Brian Gerstner, Nishant Rana, or anyone flagged leadership in Preferences |
| Self | Matches the running PM's canonical email or any alias |
| Unknown | No match against any of the above — leave as `unknown`, never guess |
| Ambiguous | Matches more than one of the above (rare identity collision) — leave as `ambiguous`, never pick one arbitrarily |

**Recipients require no new MCP call.** `gmail_read_thread` (already called for § Full thread
context) returns each message's To/CC headers as part of the thread payload already in
context — this step only reads data already fetched. Deduplicate To+CC addresses across all
messages in the thread before classifying (classify each unique address once).

The `sender` field on a Gmail signal is filtered out entirely when the sender is `self` (self
noise, per § Aliases — both directions) — this is unchanged. `recipients` entries with
category `self` are kept (the PM being a To/CC recipient is meaningful — e.g. "AM cc'd the
PM on a client email").

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
    "category": "client" | "am" | "team" | "leadership" | "self" | "unknown" | "ambiguous"
  },
  "recipients": {
    "to": [{"name": <string>, "email": <string>, "category": "client" | "am" | "team" | "leadership" | "self" | "unknown" | "ambiguous"}],
    "cc": [{"name": <string>, "email": <string>, "category": "client" | "am" | "team" | "leadership" | "self" | "unknown" | "ambiguous"}]
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
  },
  "context_link": {
    "project_id_candidates": [<int>, ...],
    "actor_emails": [<string>, ...],
    "topic_keywords": [<string>, ...],
    "timestamp": <ISO datetime>
  },
  "awaiting_action_hint": {
    "is_reply_to_wliq_ask": <bool>,
    "has_deliverable": <bool>,
    "delivery_tokens": [<string>, ...],
    "issue_tokens": [<string>, ...]
  },
  "latest_signal": {
    "source": "gmail_message",
    "id": <string — message_id of the newest non-PM message in the thread>,
    "thread_id": <string>,
    "timestamp_iso": <ISO 8601>,
    "author_name": <string — sender display name>,
    "author_email": <string — sender canonical email>,
    "excerpt": <string ≤ 240 chars — first 240 chars of the message body, plain text>
  }
}
```

## `latest_signal` field — the deep-read anchor for Gmail

Every Gmail signal carries a structured `latest_signal` field naming the newest non-PM message in the thread. This mirrors the Orbit collector's per-task anchor (`collectors/orbit.md` § Per-task `latest_signal` field) and serves the same purpose: the matcher (Job 7) picks the newest across the row's input sources as `row.latest_signal_anchor`, the writer renders it. Per SKILL.md non-negotiable rule #24 — every row must carry an anchor or be dropped to `filtered_signals`.

Selection rule:

1. Walk the thread's messages in `gmail_read_thread` output, descending by message timestamp.
2. Pick the first message whose sender is NOT the PM's canonical email or any alias. The PM's own send IS NOT a new external signal — it's the PM talking. Skip past it.
3. If every message in the thread is from the PM (a pure outbox thread where no external party has yet replied), pick the newest message anyway, set `latest_signal.author_email` to the PM, and let the matcher decide whether to drop the row.

Shape:

```
latest_signal: {
  source: "gmail_message",
  id: <string — message_id>,
  thread_id: <string>,
  timestamp_iso: <ISO 8601>,
  author_name: <string>,
  author_email: <string>,
  excerpt: <string ≤ 240 chars, plain text — strip HTML, collapse whitespace, cap at 240>
}
```

The excerpt is taken from the message body's text content (HTML stripped). For HTML-only messages with no text part, render the HTML to plain text first.

The field is populated unconditionally on every Gmail signal — no opt-in, no skip. It is the contract the matcher and writer rely on for the anchor invariant.

## Context-link metadata

Every Gmail signal carries a `context_link` object so `synthesis/matcher.md` Job 4b can attach it to corroborating Orbit signals (especially priority-lane). Population rules:

- **`project_id_candidates`** — list of Orbit project IDs the thread plausibly refers to. Resolution order: (1) the project's full name appearing as a substring of the thread subject or body, (2) any sender or recipient email matching an Orbit `client_contacts` entry — emit that project's id, (3) attachment filename matching a substring of an Orbit project title. Multiple candidates are allowed; the matcher will narrow against Orbit signals. Empty list is allowed.
- **`actor_emails`** — every unique email address appearing in the thread's From, To, CC (excluding the PM's canonical + aliases). Used by Job 4b to detect AM threads relevant to the priority lane.
- **`topic_keywords`** — short list (≤ 8 entries) of significant nouns + project-specific terms extracted from the subject line + first message body. Use simple stopword filtering; no embeddings. Lowercase, deduplicated.
- **`timestamp`** — ISO datetime of the latest message in the thread. Used for Job 4b's ±24h time-proximity rule when matching against Orbit events.

The collector populates `context_link` on EVERY Gmail signal regardless of whether the matcher actually links it later. This is cheap — the data is already extracted during normal collection.

## Orbit-notification mails (fixed-cost lane + general lane)

Spec §11.2 (fixed-cost lane) + `docs/superpowers/specs/2026-07-08-provenance-classification-design.md` §4.2 (general lane). The main routine passes `registry_snapshot` into this
collector at dispatch (same object the Orbit collection sub-agent receives). Unlike the
original fixed-cost-only version, this section is now **always active** — it runs for every
Orbit notification mail in the lookback window regardless of `registry_snapshot` contents.

**Two-tier project resolution.** For each Orbit notification mail in the lookback window:

1. **Tracked-project match (fixed-cost lane, unchanged).** Match `#<project_number>` in
   subject/body first, `registry_snapshot.tracked_projects[]` title substring second.
   Match → tag `origin: fixed_cost_mail`. Everything about this tier — the Job 4c universe,
   the Pulse rollup, the shadow dedup — is **completely unchanged** by this generalization.
2. **General-project match (new lane).** No tracked-project match → try the same
   `#<project_number>` match, then project-title substring match, against the Orbit
   relationship map's project list (already an input to this collector, per § Sender +
   recipient classification). Match → tag `origin: orbit_notification_mail`.
3. **No match against either tier** → skip (NOT an error — another PM's project, or a
   project this PM has zero relationship to).
4. **Ambiguous match** (two titles match, no project number, in either tier) → skip + emit
   an incident naming the subject: `fc_mail_parse_failure` for a tracked-tier ambiguity
   (unchanged name), `orbit_mail_parse_failure` for a general-tier ambiguity (new name — kept
   distinct so the fixed-cost lane's existing incident stream is never touched). Never guess.

**Parse per mail** (subject + body HTML) — same table for both tiers:

| Field | Source |
|---|---|
| `project_id`, `project_number` | two-tier resolution above |
| `task_title` | task name in subject/body (Orbit templates embed it; null for project-level events) |
| `event_type` | one of `comment | status | assignment | due-date | attachment | task-created | task-completed` (map from the template's action phrase) |
| `actor` | the acting user named in the mail |
| `actor_category` | classify `actor` using the SAME method as `collectors/orbit.md` § Actor classification — match against Preferences' AM list → `am`; against the resolved project's team roster → `team`; against the project's `client_contacts[]` → `client`; no match → `unknown`; multiple matches → `ambiguous`. Orbit notification mails are system-sent (e.g. `notifications@orbit...`) — the actor is NEVER the mail's `sender.category`; it must come from parsing this field. |
| `excerpt` | first ~200 chars of the event body (comment text, status old→new, etc.), HTML-stripped |
| `timestamp` | the mail's Date header (IST) |

A mail that cannot be parsed into at least `{project, event_type, actor, timestamp}` → the
same incident naming as step 4 above (`fc_mail_parse_failure` tracked-tier /
`orbit_mail_parse_failure` general-tier) + skip that mail; the run continues.

**Signal shape — tracked tier (unchanged).** Each parsed tracked-project mail becomes one
signal tagged `origin: fixed_cost_mail`, mirroring an Orbit activity signal, with
`latest_signal` built from the mail itself (`{source: "orbit-mail", actor, actor_category,
timestamp, excerpt}`) — non-negotiable rule #24 holds. These signals carry the resolved
`project_id`/`project_number` directly, so they do NOT go through Job 4b Pass 1b (already
resolved) — they enter the matcher's Job 4c universe as first-class fixed-cost signals
(`synthesis/matcher.md` § Job 4c), exactly as before.

**Signal shape — general tier (new).** Each parsed non-tracked mail becomes one signal tagged
`origin: orbit_notification_mail`, same `latest_signal` shape as above. These signals do
**NOT** enter Job 4c (that universe is fixed-cost-tagged origins only) — they flow through
the matcher's normal Job 1 → Job 1.5 → Job 2 pipeline exactly like any other Orbit-derived
signal. They are **NOT** exempt from Job 2's dedup (unlike `fixed_cost_mail` — see
`synthesis/matcher.md` § Job 2 Fixed-cost mail carve-out, which names only `fixed_cost_mail`).

**Dedup responsibility** for the tracked tier is unchanged — sits with the matcher (shadow
mode runs both feeds): see `synthesis/matcher.md` § Job 4c shadow dedup. For the general
tier, ordinary Job 2 dedup rules apply, including the existing "same project AND same
topic/deliverable (keyword overlap)" fallback for cases where this mail-derived signal has no
`task_id` to match on. This collector does not dedup against Orbit itself, in either tier.

**Pulse rollup output — tracked tier only, unchanged.** The collector ALSO emits
`mail_activity_summary[]`: one entry per **tracked** project that had ≥ 1 parsed notification
mail this run — `{project_id, project_number, title, counts by event_type, newest_excerpt,
newest_timestamp}`. This is a rollup of mails already parsed above — zero extra Gmail calls.
Projects with no parsed mail this run are simply absent from the array (no zero-count
entries). `writers/notion.md` § Step 5.7 uses this to compose the Fixed-Cost Pulse in
mail-primary mode, when the unfiltered Orbit activity loop (and therefore
`collectors/orbit.md`'s `activity_summary[]`) did not run. General-tier
(`orbit_notification_mail`) mails are NOT rolled up here — no Pulse-equivalent exists for the
general lane; this is an intentional scope decision (spec §8), not an oversight.

## `awaiting_action_hint` — feeds the Possible-Orbit-miss trigger

Populated on `client_awaiting_response`, `attachment_received`, and `thread_updated` signals (the client-originated classes). It tells `synthesis/matcher.md` § "Unactioned client signal → Create parent task" which of the three trigger sub-classes apply, computed from data the collector already pulled — NO extra MCP calls. Other signal types (am_message, team_request, leadership_message, unsent_draft, starred_unaddressed) may set all fields false / empty.

- **`is_reply_to_wliq_ask`** — `true` when the latest message is a client reply on a thread where WLIQ asked for something: `thread_depth > 1` AND `pm_last_message_excerpt` (or any prior outbound WLIQ message in the thread) reads as a request/ask ("can you send", "please share", "could you provide", "let us know", "we need", "waiting on"). Plain-language judgment, no regex.
- **`has_deliverable`** — `true` when the latest client message carries something we can act on: `attachments` is non-empty, OR the body contains a shared link (Drive/Dropbox/WeTransfer/Figma URL), OR delivery phrasing ("as requested", "here is", "attached", "please find").
- **`delivery_tokens`** — the subset of delivery phrases actually found in subject/body (`here is`, `here's`, `as requested`, `as discussed`, `attached`, `please find`, `you asked for`, `completed`, `done`, `sharing`, `for your review`). Empty list when none.
- **`issue_tokens`** — the subset of issue / critical-language phrases found in subject/body (`urgent`, `asap`, `today`, `eod`, `end of day`, `blocker`, `blocking`, `critical`, `escalation`, `please do`, `cannot wait`, `client is waiting`, `before tomorrow`, `bug`, `broken`, `not working`, `error`, `issue`, `can you add`, `feature request`, `please change`). Empty list when none.

The matcher fires S1a when `is_reply_to_wliq_ask && has_deliverable`, S1b when `delivery_tokens` is non-empty, S2 when `issue_tokens` is non-empty. Any one is enough.

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

## Transcript fallback for Fathom enrichment

When the Fathom enrichment service (`collectors/fathom.md` `fetch_enrichment()`) returns `null` for a meeting reference — because Fathom MCP is down, auth expired, or no matching meeting was found — the matcher's Job 4b Pass 2 calls a fallback helper exposed by this collector:

```
find_transcript_in_email(reference, signal_context, gmail_signal_list) -> EnrichmentResult | null
```

The helper is conceptually part of this collector because it reuses the same Gmail signal data already pulled during the morning's primary collection pass — no additional Gmail MCP search calls are made (only attachment downloads on hit).

### Inputs

- **`reference`** — same shape as the `reference` input to `fetch_enrichment` in `collectors/fathom.md` (trigger_phrase, trigger_type, extracted_meeting_id, extracted_meeting_title, extracted_date_hint, extracted_attendee_name).
- **`signal_context`** — same shape as `fetch_enrichment`'s `signal_context` (primary_signal_id, primary_signal_source, primary_signal_timestamp, project_id_candidates, actor_emails).
- **`gmail_signal_list`** — the full list of Gmail signals already collected this morning (passed by reference; not re-fetched).

### Logic

1. **Candidate-message filter.** From `gmail_signal_list`, keep messages where ALL of:
   - `timestamp ∈ [primary_signal_timestamp − 2 days, primary_signal_timestamp + 1 day]`
   - Sender email matches one of: `*@fathom.video`, `noreply@fathom.video`, `support@fathom.video`, OR any address in `signal_context.actor_emails`
   - Message has at least one attachment OR the message body contains a `fathom.video/<id>` URL
2. **Attachment-shape filter.** For each candidate message, inspect attachment filenames. Accept any whose lowercased filename contains one of: `transcript`, `summary`, `recap`, `notes`, `meeting`, `call`. Accept extensions: `.txt`, `.md`, `.docx`, `.pdf`, `.vtt`, `.srt`.
3. **Download + extract.** When a matching attachment is found, download via the same path the collector uses for normal attachment handling (existing `download_url` flow). Extract text content. For `.vtt`/`.srt`, strip timecode lines but otherwise treat the remaining text as the transcript verbatim — no smart parsing (first version keeps it simple).
4. **Compose EnrichmentResult.** Return a partial `EnrichmentResult` (same shape as `collectors/fathom.md`'s output) with these field overrides:
   ```
   {
     "meeting_id": null,
     "meeting_title": <derived from attachment filename or, if missing, the email subject>,
     "meeting_date": <message timestamp; if attachment text contains a clearly-marked meeting date header, prefer that>,
     "meeting_duration_minutes": null,
     "attendees": [
       {
         "name": <derived>,
         "email": <from email recipients + sender>,
         "is_external": <true if domain != @whitelabeliq.com>,
         "is_pm": <true if matches PM canonical email or any alias>
       }
     ],
     "pm_attended": <true if PM in recipients/sender>,
     "summary_excerpt": <first 2-4 sentences of attachment text, scoped to reference per the same scoping rule as collectors/fathom.md>,
     "relevant_action_items": [],
     "recording_url": null,
     "citation": {
       "type": "gmail_attachment",
       "label": "Email attachment: <filename> from <sender>, <date>",
       "url": <gmail thread URL>
     },
     "match_confidence": "low" | "medium",
     "enrichment_source": "gmail_attachment_fallback"
   }
   ```
   `match_confidence` is `"medium"` when the attendee/email-domain overlap with `signal_context.actor_emails` is strong; `"low"` otherwise. The extra `enrichment_source` field lets the writer differentiate this from a true Fathom hit (which has `enrichment_source` absent OR `"fathom"`).
5. **Null return.** If no candidate message has a matching attachment, return `null`. The matcher then proceeds without enrichment for this signal.

### What this helper does NOT do

- Does NOT make new Gmail MCP search calls. It scans the already-collected `gmail_signal_list` only.
- Does NOT search Google Drive for transcripts. That would expand the external-doc-access scope from "fetch specific file" to "search files" — deliberately deferred per the user's "email attachments only" decision.
- Does NOT parse `.vtt`/`.srt` semantically beyond stripping timecode lines. First version takes the file's plaintext content verbatim.
- Does NOT extract structured action items from the transcript. `relevant_action_items` is returned empty; the `summary_excerpt` is the only narrative content.
- Does NOT cache results across signals — each call is independent. If two primary signals reference the same meeting, the helper runs twice (acceptable since attachment download is bounded by lookback-window signal count).

### Tool calls used by the helper

The helper relies on the existing Gmail MCP tools listed in the next section (`gmail_read_message` / `gmail_read_thread` for attachment access, no new calls). No new MCP tool is added to the allowlist.

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
