# First-Run Setup

> **MANDATORY: Preflight (`preflight.md`) must have run before this file is loaded. This file is only triggered when preflight Step 3 detects that no `Preferences` sub-page exists on the Notion parent.**

## When this runs

The skill detects first run during preflight by checking whether a `Preferences` sub-page exists on the Notion parent identified in `config.md`. If it doesn't exist, this flow fires.

The skill never asks for the Notion parent — it's hardcoded in `config.md` and was already verified in preflight Step 2.

This flow only runs once per installation. Once `Preferences` exists, it's read silently on every subsequent run.

## Style of asking — options-based

Every question is asked in Claude's standard options-based style: a short prompt, then 3–5 numbered or bulleted options, with the last option being `Something else (please type)` for free-form input.

Some questions don't have natural options (always-include rules, tone samples) — those use free-form text from the start with a clear example.

Ask questions one at a time. Wait for the answer. Acknowledge briefly. Move on.

## The 9 setup questions

### Q1 — Identity

> "Let's get you set up. What's your name and your WLIQ email?"
>
> *Options:*
> - Type your name and email (e.g., `Aditi Singh, aditis@whitelabeliq.com`)

After they answer, use `mcp__...orbit.list_users` with `search_value` = the email to look up the Orbit user ID. Confirm:

> "Found you in Orbit — Aditi Singh, user ID 935. Right person?"
> - Yes, that's me
> - No, different account (please specify)

### Q2 — Email aliases

> "Some WLIQ team members have email aliases — for example, you might receive mail at `aditi@whitelabeliq.com` even though your actual account is `aditis@whitelabeliq.com`. (This happens because some first-name-only addresses were already taken so accounts use first-name + last-initial.) Do you have any aliases I should know about?"
>
> *Options:*
> - No, only my main work email
> - Yes — let me list them

If `Yes`, ask:

> "Paste your aliases, one per line."

Store both the canonical email (from Q1) and any aliases in Preferences. The skill will treat all of them as the same identity.

### Q3 — Morning collection run time

> "What time should I fire your morning collection run each day?"
>
> *Options:*
> - 9:00 AM IST
> - 9:30 AM IST (recommended)
> - 10:00 AM IST
> - Something else (type a time, e.g., `8:45 AM IST`)

Validate the time is in IST and a real 24-hour value. Confirm back.

### Q4 — Execution run time

> "What time should I fire the execution run each day? This is when I act on the items you've approved. Default is 10:45 AM IST — gives you about 75 minutes after the morning run to review and flip approvals."
>
> *Options:*
> - 10:30 AM IST
> - 10:45 AM IST (recommended)
> - 11:00 AM IST (just before office opens)
> - Something else (type a time)

Validate: execution time must be AFTER morning time. Reject and re-ask if not.

### Q5 — Escalation backup

> "If you can't review your morning queue one day — sick, emergency, pulled into something — who should I notify so they can cover?"
>
> *Options (auto-populated from Orbit's PM list at WLIQ):*
> - Hiten Upadhyay
> - Dhruvi Chandarana
> - Aditi Singh
> - [Other WLIQ team members]
> - Something else (type a name)

Then:

> "And how should I reach them? At what time should I ping them if your Ready toggle is still off?"
>
> *Channel options:*
> - Slack (recommended for fast response)
> - Email
> - Something else
>
> *Time options:*
> - Same as your execution time
> - 15 minutes after your execution time
> - 30 minutes after your execution time
> - Something else (type a time)

### Q6 — Account managers

> "Who are your account managers? I'll add them one at a time."
>
> *Options (auto-populated from WLIQ AM list in Orbit):*
> - Caitlin Sims
> - Sarah Chen
> - Jessica Provencher
> - Ellen Thomas
> - [Other AMs]
> - Something else (type a name)

For each AM the PM picks:

> "For Caitlin Sims — what's her email and Slack handle? And do you prefer Slack or email when contacting her?"
>
> *Channel preference:*
> - Slack for quick handoffs, email for paper trail (recommended)
> - Always Slack
> - Always email
> - Something else

Loop until the PM says they're done adding AMs.

### Q7 — Default email preferences

> "When I draft emails on your behalf, any default CCs you always want me to add?"
>
> *Options:*
> - No default CCs
> - CC the relevant AM on every client email
> - CC Nishant on every leadership email
> - Both (CC AM on client emails, CC Nishant on leadership emails)
> - Something else (free text rules)

Then:

> "Do you have a signature you want appended to every email?"
>
> *Options:*
> - `Best, [your first name]` (default)
> - `Thanks, [your first name]`
> - Something else (paste your signature)

### Q8 — Default Slack preferences for team handoffs

> "When I send Slack messages to your team members in India, what tone matches how you communicate with them?"
>
> *Options:*
> - Direct and warm — short sentences, simple words, friendly
> - Brief and to-the-point — minimal context, just the ask
> - Detailed — full context every time
> - Something else (describe)

> "Do you want me to always remind assignees to log their hours at the end of the handoff?"
>
> *Options:*
> - Yes, always
> - No
> - Only when the task has estimated hours

### Q9 — Always-include rules

This question is free-form by design. Walk the PM through it with examples.

> "Last functional question — anything you want me to ALWAYS include or ALWAYS avoid? Examples:
> - 'Always include the WP admin URL when assigning WordPress work'
> - 'Never CC external clients on internal threads'
> - 'When reassigning due to capacity, always explain why'
> - 'When assigning QA, always link the original task being QA'd'
>
> Type your rules, one per line. Skip if you have none."

Store as a free-text bulleted list in Preferences.

### Q10 (optional) — Tone calibration samples

> "Optional — can you paste 3 to 5 real Slack messages or emails you've recently sent to your team? I'll use them to match your tone."
>
> *Options:*
> - Yes, let me paste samples
> - Skip — use a generic professional tone

If skipped, note in Preferences: `Tone samples: SKIPPED — using generic tone. Run "PM Task Assignment, update tone samples" anytime to add.`

If they provide samples, store each verbatim with a date and short description (e.g., `Sample 1 — Slack to Vijay, 20 April 2026`).

## After all 10 (or 9 if Q10 skipped) answers

### Step 1 — Summarize

Show the PM a clean summary of everything they entered. Ask:

> "Look right? Should I save this as your Preferences page?"
>
> *Options:*
> - Yes, save it
> - Let me edit something first

If they want to edit, loop back to the relevant question.

### Step 2 — Write the Preferences page

On confirmation, create the `Preferences` sub-page on the Notion parent (per `schemas/preferences-page.md`). Position it at the bottom of the parent.

### Step 3 — Register scheduled tasks

Via `mcp__...scheduled-tasks.create_scheduled_task`:

1. **Mode 1** — daily at Q3 time, IST. Invokes the skill to run morning collection.
2. **Mode 2** — daily at Q4 time, IST. Invokes the skill to run execution.
3. **Monthly archival** — 1st of each month at 6:00 AM IST. Invokes the skill to archive previous month's pages.

Store the scheduled-task IDs in the `Internal State` section of Preferences.

### Step 4 — Confirmation message

> "You're all set up. Here's what happens next:
> - Tomorrow at [Q3 time] IST, I'll fire the morning collection automatically. You'll see today's queue when you open Notion in the morning.
> - At [Q4 time] IST, I'll execute everything you've approved.
> - If you don't flip the Ready for Execution toggle by then, I'll Slack [escalation backup] to cover.
>
> Your Preferences are at [Notion link] — edit any time and I'll pick up the changes on the next run.
>
> You can also manually fire a run with `PM Task Assignment, run morning` or change a preference with `PM Task Assignment, change my preference: [your instruction]`.
>
> See you tomorrow."

## Edge cases

- **Orbit identity lookup fails (Q1).** Ask for the Orbit user ID directly. If they don't know it, walk them through finding it in Orbit. Worst case, list candidate users and let them pick.
- **PM declines to specify aliases (Q2).** Just store the canonical email. The skill will handle alias matching only against the canonical.
- **Time zone is not IST (Q3, Q4).** Accept the time in their local zone, convert to IST internally, store both. Note in Preferences: `Original time zone: [zone].`
- **Escalation backup is themselves (Q5).** Detect and refuse: "Your backup is set to yourself, which won't work. Pick someone else." Re-ask.
- **No AMs to list (Q6).** Allow `None`. The skill won't draft AM-bound messages until the PM adds AMs later via `change my preference`.
- **scheduled-tasks MCP unavailable.** Note in Preferences that scheduling failed; PM will need to fire runs manually until the connector is restored. Slack the PM via `connector-failure-notify.md`.

## Language

This entire setup conversation is in normal professional English. The plain-language rule (4th–5th grade) only applies to content the skill produces FOR the India delivery team later.

## After first run

Every subsequent invocation skips this file entirely. Read Preferences and proceed.
