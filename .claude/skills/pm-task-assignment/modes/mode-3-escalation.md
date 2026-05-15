> **MANDATORY: `preflight.md` must run before any logic in this file. Do not call any tool, do not act on user input, until preflight has completed successfully. This includes scheduled-task triggers — preflight runs even when invoked by the scheduler.**

> **Source allowlist (closed):** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. No other MCP, ever — including any that may seem relevant to a specific signal. The allowlist is enforced even under experimental scope or forced runs.

# Mode 3 — Escalation Run

## When this runs

Collapsed into the end of Mode 2. Fires if and only if the page-level `Ready for Execution` toggle at the top of today's dated page is OFF at the scheduled Mode 2 time.

Can also fire independently if the PM has configured a different escalation time than execution time (rare). In that case, a separate scheduled task runs just this file.

## Purpose

If the PM hasn't marked today's page ready by the configured time, something is wrong — they may be sick, in a meeting, traveling, or forgot. Notify the escalation backup so the team isn't stuck.

## End-to-end flow

### Step 1 — Read Preferences

Pull:
- PM identity (for the backup message)
- Escalation backup: name, channel (Slack or email), time
- Today's dated page URL (recorded by Mode 1 at the end of its run)

### Step 2 — Read the Ready for Execution toggle

Fetch today's dated page. Check the Ready toggle.

- **If ON:** do nothing. Exit. (Mode 2 will handle it.)
- **If OFF:** proceed to escalation.

### Step 3 — Compose the escalation message

Template for Slack:

```
[PM name] hasn't marked today's morning queue ready for execution by [escalation time] IST.

This means one of two things:
  • They're unavailable today (sick, emergency, meeting).
  • They forgot to flip the Ready toggle.

Can you cover? Review the queue and either:
  • Flip the Ready toggle yourself if everything looks good — the skill will execute at the next hourly check.
  • Act on the items manually if you'd rather handle them yourself.

Morning queue: [link to today's dated page]
Items count: [N]
```

Template for email (if backup is configured for email instead of Slack):

```
Subject: Morning queue not marked ready — can you cover for [PM name]?

Hi [Backup name],

[PM name] hasn't marked today's morning queue ready for execution by [escalation time] IST. They may be unavailable or forgot to flip the toggle.

Would you mind covering? Open the queue and either:
  - Flip the "Ready for Execution" toggle at the top of the dated page if the recommendations look right — the skill will execute at the next hourly check.
  - Act on any items manually if you'd rather handle them yourself.

Queue: [link to today's dated page]
Items count: [N]

Thanks,
PM Task Assignment skill
(Automated message on behalf of [PM name])
```

### Step 4 — Send the message

Route based on Preferences:
- If channel = Slack: use `executors/slack.md` to send to the backup's Slack handle.
- If channel = email: use `executors/email.md` to send directly (this is one of the rare cases where an email IS sent, not drafted — it's internal/operational, not client-facing).

### Step 5 — Log the escalation

Add a note to today's dated page, under the summary line at the top:
> "Escalation fired at [time] IST. [Backup name] notified via [channel]."

This way the PM sees it when they eventually open the page.

### Step 6 — Notify the PM themselves

Also send a Slack DM to the PM:
> "I escalated today's morning queue to [backup name] at [time] IST because the Ready for Execution toggle was still off. If this was unintentional, flip the toggle now and type `PM Task Assignment, run execution now` to proceed."

### Step 7 — Stop

Do not execute any rows. Escalation is a handoff, not an action. The backup (or the PM themselves, once they notice) has to flip Ready and trigger Mode 2.

## Hourly re-check (light)

For 3 hours after escalation, every hour: re-check the Ready toggle silently. If someone flips it ON, fire Mode 2 automatically. This handles the case where the PM or backup fixes the toggle without manually running the command.

After 3 hours, stop re-checking. Everything else that day is manual via the command.

## Edge cases

- **No escalation backup configured.** In Preferences, `escalation_backup` is empty or unset. In that case, do not escalate. Instead, Slack the PM: "I noticed the Ready toggle isn't on and there's no escalation backup configured. Today's queue was not executed. Add a backup via `PM Task Assignment, change my preference: add [name] as my escalation backup`." Exit.
- **Backup's Slack handle isn't valid.** Fall back to email if email is also configured. If neither works, Slack the PM that escalation failed and why.
- **PM is the backup.** If somehow the PM configured themselves as their own backup, detect this and refuse — Slack: "Your escalation backup is set to yourself. That won't work. Set someone else via `PM Task Assignment, change my preference: change my escalation backup to [name]`."

## What Mode 3 does NOT do

- Does not execute any rows.
- Does not edit Preferences.
- Does not retry after 3 hours (manual only from there).
- Does not send the escalation message more than once per day.
