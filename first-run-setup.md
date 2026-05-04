# First-Run Setup

> **Operator-facing per-PM setup.** Routines fire headless (no human in the loop), so per-PM Preferences must already exist on disk (in Notion) before the first routine fire. The operator duplicates a master `Preferences — Template` page into the PM's parent and fills it inline. For interactive (Claude Desktop) runs, the same Preferences page is read; if a required field is missing the skill reports it via the same preflight error path. See `ENVIRONMENTS.md` for the two execution surfaces.

## Why the change

Routines fire headless on cron — there is no human in chat at fire time, so anything the skill needs must already exist on disk (in Notion) before the first scheduled fire. Setup must be fully completed and the setup checklist fully ticked before the first routine runs, otherwise preflight aborts with a "setup incomplete" Run Log entry and the routine exits without writing anywhere else. This is operator-facing documentation; the PM never types answers into a chat — they edit a Notion page directly.

## The Preferences template page

The operator (you, deploying for a PM) duplicates a master `Preferences — Template` Notion page into the PM's parent workspace, renames it to `Preferences`, and fills the inline fields below. Each field is a Notion property, callout, or toggle block on the page; the schema is locked in `schemas/preferences-page.md` and must match exactly so preflight can parse it.

For each field below: a placeholder example is given, then "valid input" describes what preflight will accept, then "missing" describes what causes preflight to abort.

### Field 1 — Identity

Block type: four inline text properties at the top of the page.

- `Full name` — placeholder `[PM Full Name]`. Example: `Aditi Singh`. Valid: any non-empty string with at least a first and last name. Missing: blank, or only one word.
- `Orbit user ID` — placeholder `[PM Orbit ID]`. Example: `935`. Valid: integer between 1 and 99999. Missing: blank, non-numeric, or `0`.
- `Canonical email` — placeholder `pm@whitelabeliq.com`. Example: `aditis@whitelabeliq.com`. Valid: a `@whitelabeliq.com` address. Missing: blank, or any other domain.
- `Email aliases` — placeholder `[alias@whitelabeliq.com]` (one per line, optional). Example: `aditi@whitelabeliq.com`. Valid: zero or more `@whitelabeliq.com` addresses, one per line. Missing: never (this field is optional and can be empty).

### Field 2 — Morning run time (IST)

Block type: single text property labelled `Morning run time (IST)`.

- Placeholder: `09:30 IST`
- Valid: 24-hour `HH:MM IST` format, between `06:00 IST` and `12:00 IST`.
- Missing: blank, wrong format, missing `IST` suffix, or outside the allowed window.

### Field 3 — Execution run time (IST)

Block type: single text property labelled `Execution run time (IST)`.

- Placeholder: `10:45 IST`
- Valid: 24-hour `HH:MM IST` format, strictly **after** the morning run time, and before `14:00 IST`.
- Missing: blank, wrong format, equal to or earlier than morning run, or after `14:00 IST`.

### Field 4 — Escalation backup

Block type: a `Backup` toggle block containing four inline properties.

- `Backup name` — placeholder `[Backup Full Name]`. Example: `Hiten Upadhyay`. Valid: any non-empty name that is **not** the PM's own name from Field 1. Missing: blank, or matches the PM's own name (preflight rejects self-as-backup).
- `Backup channel` — placeholder `Slack`. Valid: one of `Slack`, `Email`. Missing: blank or any other value.
- `Backup handle/email` — placeholder `@[backup-handle]` (Slack) or `backup@whitelabeliq.com` (email). Example: `@hiten` (Slack) or `hitenu@whitelabeliq.com` (email). Valid: matches the channel above. Missing: blank, or doesn't match channel format.
- `Ping time (IST)` — placeholder `11:00 IST`. Valid: 24-hour `HH:MM IST`, **at or after** the execution run time. Missing: blank, wrong format, or earlier than execution time.

### Field 5 — Account managers

Block type: a Notion database labelled `Account Managers` embedded on the page. One row per AM. Columns:

- `AM name` — placeholder `[AM Full Name]`. Example: `Caitlin Sims`. Valid: non-empty.
- `AM email` — placeholder `am@whitelabeliq.com`. Example: `caitlin@whitelabeliq.com`. Valid: any email.
- `AM Slack handle` — placeholder `@[am-handle]`. Example: `@caitlin`. Valid: starts with `@`.
- `Channel preference` — placeholder `Slack for handoffs, email for paper trail`. Valid: free text describing routing rule.

Missing: zero rows. If the PM truly has no AMs, they enter a single row with `AM name = NONE` — preflight will accept this and the skill will skip AM-bound drafts until the row is replaced.

### Field 6 — Default email preferences

Block type: a `Email Defaults` toggle block with two inline properties.

- `Default CC rules` — placeholder `CC the relevant AM on every client email; CC [leadership] on every leadership email`. Example: `CC the relevant AM on every client email; CC Nishant on every leadership email`. Valid: free text, at least 5 characters. Missing: blank.
- `Email signature` — placeholder `Best,\n[PM First Name]`. Example: `Best,\nAditi`. Valid: free text, at least 3 characters. Missing: blank.

### Field 7 — Default Slack tone for team handoffs

Block type: a `Slack Tone` toggle block with one property.

- `Default Slack tone` — placeholder `[describe your team handoff tone]`. Example: `Direct and warm — short sentences, simple words, friendly. Always remind assignees to log hours.` Valid: free text, at least 20 characters. Missing: blank or under 20 chars.

### Field 8 — Always-include rules

Block type: a `Always-include rules` toggle block containing a bulleted list.

- Placeholder bullets:
  - `Always include the WP admin URL when assigning WordPress work`
  - `Never CC external clients on internal threads`
  - `When reassigning due to capacity, always explain why in the handoff`
- Valid: zero or more bullets. Missing: never (optional). If empty, preflight notes "no always-include rules" and proceeds.

### Field 9 — Tone calibration samples

Block type: a `Tone Samples` toggle block with 3-5 child callouts, each containing a verbatim Slack/email paste.

- Placeholder for each sample callout: a date label (example: `Sample 1 — Slack to [team member], 20 April 2026`) followed by the verbatim message body.
- Valid: 3-5 callouts, each with at least 30 characters of body text.
- Missing: this field is **optional**. Empty is allowed; preflight notes "tone samples skipped — using generic professional tone" and proceeds.

## Setup checklist callout

At the very TOP of the Preferences page, the template includes a single Notion callout block titled `Setup checklist — tick each box as you fill the field below`. Inside the callout is a to-do list with one checkbox per required field:

- [ ] Identity (name, Orbit ID, canonical email)
- [ ] Morning run time (IST)
- [ ] Execution run time (IST)
- [ ] Escalation backup (name, channel, handle, ping time)
- [ ] Account managers (at least one row, even if NONE)
- [ ] Default email preferences (CC rules + signature)
- [ ] Default Slack tone
- [ ] Always-include rules (or explicitly left empty)
- [ ] Tone samples (or explicitly left empty — optional)

**Preflight reads this checklist on every routine fire. Any unticked checkbox = preflight aborts with `setup incomplete: <field name>` in the Run Log, and the routine exits without firing the mode.** The PM ticks each box as they finish filling that field.

## Validation rules (preflight)

On every routine fire, preflight checks:

1. The Preferences page exists at the configured URL and is reachable via the Notion MCP.
2. Every checklist box is ticked. Any unticked box → abort.
3. All required fields above are populated according to their "Valid input" rule. Any missing required field → abort.
4. Time fields parse as `HH:MM IST` and obey the ordering rules (morning < execution ≤ ping).
5. Escalation backup name is not equal to the PM's own name.
6. Canonical email is on the `@whitelabeliq.com` domain.
7. Account Managers database has at least one row.
8. The setup checklist callout itself exists and is parseable (i.e. the operator did not accidentally delete it).

If all checks pass, preflight writes `setup OK` to the Run Log and hands off to the mode the routine invoked.

## Deployment steps for the operator

1. Open the master `Preferences — Template` page in the shared Notion workspace.
2. Duplicate it into the PM's parent page (use Notion `Duplicate to`).
3. Rename the duplicate to `Preferences` (drop the `— Template` suffix).
4. Fill all 9 fields with the PM's real values, using the placeholders above as a guide.
5. Tick every box in the setup checklist callout at the top of the page as each field is completed.
6. Capture two values:
   - The PM's parent page ID (the Notion page that contains `Preferences` as a sub-page).
   - The Preferences page URL (full Notion link).
7. Create 3 Claude Routines (see `ROUTINE-ENTRYPOINTS.md` for the exact prompts). Paste the parent page ID and Preferences URL into each routine prompt where placeholders are marked.
   - Mode 1 routine — fires daily at the PM's morning run time.
   - Mode 2 routine — fires daily at the PM's execution run time.
   - Monthly archival routine — fires on the 1st of each month at 06:00 IST.
8. Wait for the next morning's first scheduled fire. Verify a Run Log entry appears under the parent page with `setup OK` followed by Mode 1 output. If the entry says `setup incomplete: <field>`, fix that field, re-tick the checklist, and the next fire will pick up the corrected page.

## Edits after deploy

The PM edits Preferences directly in Notion. Any edit takes effect on the **next** routine fire — there is no `change my preference` chat command in v1. Time changes (morning run, execution run, ping time) require an additional step: the operator updates the cron expression on the corresponding Claude Routine to match. The skill itself does not reschedule routines; the routine's cron is the source of truth for fire time.

If the PM edits a required field to an invalid value, the next preflight aborts and writes the error to the Run Log. The PM (or operator) corrects the field, and the following fire proceeds.

## Tone samples

The PM pastes tone samples directly into the `Tone Samples` toggle block on the Preferences page — one callout per sample, with a date label. There is no `update tone samples` chat command. To add or replace samples, edit the toggle block in Notion; the next routine fire reads the updated samples.

## Reference

For the exact field-by-field block layout, property names, and Notion block types, see `schemas/preferences-page.md`. That file is the canonical schema; this file is the operator-facing how-to. If the two ever diverge, the schema wins and this file is updated to match.

---

> **Loading verification.** If you (Claude) are reading this file inside a routine, this is operator documentation, not a flow you execute. Continue with the calling mode/preflight logic.
