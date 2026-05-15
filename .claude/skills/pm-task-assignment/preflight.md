# preflight.md — Required Pre-Run Sequence

> **MANDATORY: This sequence runs at the start of every skill invocation, scheduled or manual. SKILL.md and every mode/command file references this file as the first step. Do not skip steps. Do not call any tool outside of this sequence until preflight has completed successfully.**

## When this runs

Every entry into the skill:

- Manual command: `PM Task Assignment, run morning` / `run execution now` / `change my preference: ...` / `update tone samples`
- Scheduled run: Mode 1, Mode 2, monthly archival
- Any other invocation Claude perceives as related to PM Task Assignment

## The 5-step preflight sequence

### Step 1 — Read config.md in full

- Open `config.md` and read every section.
- Confirm `DEFAULT_NOTION_PARENT_PAGE_ID` is set to a real Notion page ID (32-char hex string with or without dashes).
- Confirm the source allowlist clause is loaded into context: only Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks. Nothing else.
- Confirm the Preferences-must-be-respected clause is loaded.

If `config.md` cannot be read or the page ID is missing/malformed, abort the run with a clear error message to the user: "config.md is missing or malformed. The skill cannot run. Please re-install or check the file."

### Step 2 — Confirm the Notion parent page exists

- Use `notion-fetch` to load the page identified by `DEFAULT_NOTION_PARENT_PAGE_ID`.
- If the page is unreachable (404, permission denied, or the Notion MCP is down), route to the **connector-failure-notify** flow (`connector-failure-notify.md`). Tier 3 of the fallback chain — write a bold-red note on whatever Notion page IS accessible — applies here only if a different Notion page can be written to. Otherwise, surface the error in the Claude conversation and ABORT the run.
- If the page exists, capture its title and basic metadata for later use.

### Step 3 — Look for the Preferences sub-page

- Search the parent's children for a sub-page titled exactly `Preferences`.
- **If Preferences does NOT exist:**
  - This is a first-run scenario.
  - If the invoking command is `PM Task Assignment, run morning` or any other run-style command, **DO NOT proceed to mode logic.** Instead, route to `first-run-setup.md` and run the setup flow conversationally.
  - If the invoking command is `change my preference: ...` or `update tone samples`, also route to `first-run-setup.md` first — the PM must complete setup before they can edit preferences.
  - Setup creates the Preferences sub-page. Once setup is complete, return to the original command (or, if it was a scheduled run, simply log that setup was completed and exit — don't auto-fire the morning run on the same invocation).

- **If Preferences DOES exist:** continue to step 4.

### Step 4 — Parse Preferences and confirm required fields

Read the Preferences sub-page (per `schemas/preferences-page.md`). Confirm the required fields are populated:

- Identity (name, Orbit user ID, canonical email, email aliases if any)
- Run schedule (morning run time, execution run time)
- Escalation backup (name, channel, time)
- Account managers (at least one, with email or Slack handle)
- Communication defaults
- Internal state fields (last_morning_run, last_execution_run — may be null if first ever run)

If any **required** field is missing from a Preferences page that exists:

- Slack the user themselves: "Your Preferences page is incomplete — [field] is missing. Please run `PM Task Assignment, change my preference: ...` to fix, or re-run first-run setup."
- ABORT the current run.

### Step 5 — Verify the 6 MCP connectors are responsive

For each of the 6 allowlisted MCPs, run a lightweight ping (a read-only call that should succeed quickly):

| MCP | Verification call |
|---|---|
| Orbit | `get_user_details` for the PM (identity check from Preferences) |
| Gmail | `gmail_get_profile` |
| Slack | `slack_read_user_profile` |
| Fathom | `list_meetings` with the smallest date range possible |
| Notion | `notion-fetch` of the parent page (already done in step 2 — reuse) |
| scheduled-tasks | `list_scheduled_tasks` |

For each MCP that fails:

- Route to `connector-failure-notify.md` with the MCP name and the error.
- Continue checking the rest — collect ALL failures, then notify.

After the verification:

- **If all 6 MCPs are responsive:** continue to the original mode/command logic. Preflight done.
- **If 1+ MCPs are down:** the failure-notify flow has been triggered. Decision per mode:
  - Mode 1 (collection): proceed with the available collectors. Note the missing source on the dated page summary. Don't abort.
  - Mode 2 (execution): if Notion or Orbit are down, ABORT — they're load-bearing. Other MCPs are non-blocking; proceed with what's available.
  - Mode 3 (escalation) and other commands: try to proceed with what's available; abort only if the specific MCP needed for THIS command is down.

## What preflight does NOT do

- Does NOT execute any morning collection logic, executor logic, or write any task data. That happens after preflight returns control.
- Does NOT modify Preferences.
- Does NOT modify config.
- Does NOT call any tool from outside the 6-MCP allowlist.

## Identity check (also part of preflight, in step 5)

When verifying the Slack and Gmail connectors, the skill confirms identity:

- Slack profile email or username should match Preferences canonical email or one of the aliases.
- Gmail authenticated account should match Preferences canonical email or one of the aliases.

If neither matches, surface a warning to the user: "I notice the Slack/Gmail account I'm authenticated to doesn't match the identity stored in Preferences. Are you running this on the right machine?" Do not abort — the PM may have legitimate reasons (testing, multi-device). Just flag.

## Loading verification

If you (Claude) are reading this comment, you have loaded `preflight.md` correctly. After preflight completes, return control to the file that called it (SKILL.md or a mode/command file).
