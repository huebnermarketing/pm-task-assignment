> **This executor uses ONLY the relevant MCP from the 6-MCP allowlist. The allowlist is closed: Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever.**

> **Preflight (`preflight.md`) must have run before this executor is invoked.**

# Email Executor

## Purpose

Draft and optionally send emails on the PM's behalf via Gmail MCP. Default behavior is DRAFT ONLY — never auto-send client-facing email. Internal operational emails are the rare exceptions where the skill sends directly.

## The rule

**Nothing client-facing goes out without PM approval.** The PM's approval is the explicit gesture of either flipping a row to `Approved` OR dropping a PM note that includes "send" language (e.g., "send this to Jane"). Without one of those, it's a draft.

## The three send-not-draft exceptions

These are the only cases where this executor sends an email rather than saving it as a draft. Every other email is a draft.

1. **Escalation email to the configured backup person** (from `modes/mode-3-escalation.md`, when the backup's preferred channel in Preferences is `email`). Internal, operational. Sent.
2. **Self-notification on connector failure** (from `connector-failure-notify.md` Tier 2, when Slack is unreachable). The PM's Gmail sends an email TO themselves (canonical email or any alias) so they see the connector failure in their inbox. Sent.
3. **PM note explicitly says "send"** (e.g., a PM Notes value like `send this to Caitlin now`). The matcher's note interpreter recognizes the explicit send instruction and the executor sends rather than drafts.

Anything outside these three is always a draft. No fourth exception. Adding a new send case requires updating this file AND `executors/email.md`-aware modes/notes.

## Supported operations

### Create a draft in the PM's Gmail

Use `mcp__...gmail.gmail_create_draft`.

Required parameters:
- `to` — recipient email
- `subject` — email subject
- `body` — email body (plain text or HTML)

Optional:
- `cc` — from PM note or Preferences default
- `bcc` — rare, only from explicit PM note
- `thread_id` — if replying to an existing thread (prefer this for replies)

Draft is saved in the PM's Gmail. They open, review, send from Gmail.

### Send an email

Only two cases where the skill sends directly:

1. **Escalation email to the PM's configured backup** (from `modes/mode-3-escalation.md`). Internal, operational.
2. **When the PM's note explicitly says "send"** — e.g., "send this to Caitlin", "send the email".

For all other cases (drafts are the default), leave it as a draft.

If sending is needed, use `mcp__...gmail.gmail_send_message` (or similar tool depending on the Gmail MCP's available operations).

## Email content rules

### To clients and agencies

- **Language:** normal professional English (client-side recipients are typically US-based and English-native).
- **Tone:** professional, warm, clear. Match the tone samples in Preferences if available.
- **Structure:**
  - Greeting with recipient name
  - Context (1-2 sentences)
  - Specific ask, info, or decision
  - Closing with signature (from Preferences)
- **CCs:** default AM CC if the Preferences rule says "always CC AM on client emails." Otherwise, CC only per PM note or explicit always-include rule.
- **Never auto-send.**

### To Account Managers

- **Language:** normal professional English.
- **Tone:** concise, direct — AMs appreciate brevity.
- **Structure:** short context + what you need from them or what you're telling them.
- **Default to Slack** unless Preferences says email is preferred for that specific AM.
- Drafts only unless PM says send.

### Internal to PM's team (team member handoffs that go via email)

- **Language:** plain-language 4th-5th grade English per `writers/plain-language.md`, because the delivery team reads this.
- **Tone:** simple, direct, actionable.
- Rare — team handoffs usually go via Slack, not email. Only email when the PM specifically requests it.

### Operational / system (escalations, confirmations)

- **Language:** normal English.
- **Tone:** functional, clear.
- May send directly (escalation is the main case).

## Subject line conventions

- Prefix with project or client: `Agency X — Homepage Revisions`
- For replies, keep the original subject with `Re:`
- Keep under 60 characters for scannability

## Source citation in emails

Per `writers/source-citation.md`, if the email body references content from an attachment or document, cite it inline:

> "Per the revision feedback attached (`homepage_revision_feedback.pdf`), the hero image should be swapped and..."

Not all emails need citations — only those that reference sourced content beyond the conversation at hand.

## Signature

From Preferences' `signature` field. Default: `Best, [PM first name]`.

## CC and BCC logic

- **Default CC:** per Preferences' `default_ccs` rule (e.g., "always CC Nishant on leadership emails")
- **Per-row CC:** per PM note (e.g., `add Nishant as CC`)
- **BCC:** only when PM note explicitly says "BCC [person]" — rare

## Edge cases

### Client's email domain doesn't match the Orbit client

- Still send to the email address the PM note or source signal provided
- Log in AI Notes: `Email sent to [address] — domain didn't match Orbit client record, so I couldn't verify. Please double-check.`

### Reply threading

- Always try to reply within the original thread if the email is a response to a prior client/AM message
- Fetch the `thread_id` from the Gmail source signal
- If the source isn't Gmail (e.g., the trigger was a Fathom meeting), start a new thread

### HTML vs plain text

- Use HTML for formatting when it helps readability (bullet lists, bold for headings)
- Keep it simple — no heavy styling, no embedded images unless from an attached file

### Attachments

- If the email should include an attachment (rare), the skill can:
  - Include a link to the attachment in the body text
  - For the MVP: do NOT auto-attach files via Gmail MCP (may not be supported cleanly); instead, say "Please find the latest version at [link to Orbit task or Drive]"
  - PM can manually attach when they review the draft

## After drafting or sending

Return to Mode 2:
- If drafted: Outcome = `Draft saved to your Gmail. Review and send when ready.`
- If sent: Outcome = `Email sent to [recipient]. Thread: [link if available].`

## Error handling

| Failure | Behavior |
|---|---|
| Gmail MCP unavailable | Log in Outcome: `FAILED — Gmail unavailable.` Skip this operation. Continue with other ops for the row. |
| Invalid recipient address | Log and skip. Do not attempt to guess. |
| Gmail send fails | Fall back to draft. Log: `Send failed — saved as draft. [error message]. Please review.` |

## What this executor does NOT do

- Does not send client-facing emails without explicit PM approval.
- Does not send emails from addresses other than the PM's own.
- Does not modify existing sent emails.
- Does not set up filters, labels, or rules.
- Does not schedule emails for future send.
- Does not track email opens or replies automatically (next run's Gmail collector picks up replies).
