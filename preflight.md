# preflight.md — Required Pre-Run Sequence

> **MANDATORY: This sequence runs at the start of every skill invocation, scheduled or manual. SKILL.md and every mode/command file references this file as the first step. Do not skip steps. Do not call any tool outside of this sequence until preflight has completed successfully.**

## When this runs

Every entry into the skill:

- Manual command: `PM Task Assignment, run morning` / `run execution now` / `change my preference: ...` / `update tone samples`
- Scheduled run: Mode 1, Mode 2, monthly archival
- Any other invocation Claude perceives as related to PM Task Assignment

## The 6-step preflight sequence

### Step 1 — Read config.md in full and detect the execution surface

- Open `config.md` and read every section.
- Confirm `DEFAULT_NOTION_PARENT_PAGE_ID` is set to a real Notion page ID (32-char hex string with or without dashes).
- Confirm the source allowlist clause is loaded into context: primary collection — Orbit, Gmail, Fathom, Notion; Slack is outbound-send only (Mode 2 team-handoff + AM-ping with explicit PM `send` note — see `executors/slack.md`) and is never used for collection; read-only references on demand — Google Drive/Docs/Sheets, SharePoint per `references/external-doc-access.md`.
- Confirm the Preferences-must-be-respected clause is loaded.
- **Set the in-memory flag `is_interactive`.** `true` if the skill was triggered by a human-typed message in a Claude session (Claude Desktop manual command); `false` if the trigger is a routine / cron / SDK invocation. Mode files read this flag to decide whether to ask one clarifying question per ambiguous item or to fall straight to `Uncertain:`. See `ENVIRONMENTS.md`.

If `config.md` cannot be read or the page ID is missing/malformed, abort the run with a clear error message to the user: "config.md is missing or malformed. The skill cannot run. Please re-install or check the file."

### Step 2 — Confirm the Notion parent page exists

- Use `notion-fetch` to load the page identified by `DEFAULT_NOTION_PARENT_PAGE_ID`.
- If the page is unreachable (404, permission denied, or the Notion MCP is down), route to the **connector-failure-notify** flow (`connector-failure-notify.md`). Tier 3 of the fallback chain — write a bold-red note on whatever Notion page IS accessible — applies here only if a different Notion page can be written to. Otherwise, surface the error in the Claude conversation and ABORT the run.
- If the page exists, capture its title and basic metadata for later use.

### Step 3 — Look for the Preferences sub-page

- Search the parent's children for a sub-page titled exactly `Preferences`.
- **If Preferences does NOT exist:**
  - **ABORT the run.** Routines cannot run interactive setup. Email the PM (identity from routine config; sent, not drafted): subject `[PM Task Assignment] Preferences page missing — setup incomplete`, body `Preferences page is missing on the Notion parent — setup is incomplete. Duplicate the template per setup-template.md and create the Preferences sub-page before next routine fire.` Then exit. Do NOT route to first-run-setup.md from a routine.

- **If Preferences DOES exist:** continue to step 4.

### Step 4 — Parse Preferences and confirm required fields

Read the Preferences sub-page (per `schemas/preferences-page.md`). Confirm the required fields are populated:

- Identity (name, Orbit user ID, canonical email, email aliases if any)
- Run schedule (morning run time, execution run time)
- Escalation backup (name, email, time)
- Account managers (at least one, with canonical email and any aliases — the AM identity list is also consumed by the Mode 1 Step 3a Orbit-first priority pass)
- Communication defaults
- Internal state fields (last_morning_run, last_execution_run — may be null if first ever run)

If any **required** field is missing from a Preferences page that exists:

- Email the user themselves (sent, not drafted): "Your Preferences page is incomplete — [field] is missing. Please run `PM Task Assignment, change my preference: ...` to fix, or re-run first-run setup."
- ABORT the current run.

### Step 5 — Verify the 4 MCP connectors are responsive

For each of the 4 allowlisted MCPs, run a lightweight ping (a read-only call that should succeed quickly):

| MCP | Verification call |
|---|---|
| Orbit | `get_user_details` for the PM (identity check from Preferences) |
| Gmail | `gmail_get_profile` |
| Fathom | `list_meetings` with the smallest date range possible |
| Notion | `notion-fetch` of the parent page (already done in step 2 — reuse) |

> **Retry policy applies.** Each verification ping above uses the retry-with-backoff policy in `connector-failure-notify.md` (4 attempts total, 2s/5s/15s backoff, retry only on transient errors). A single transient blip during preflight does NOT abort the run — only exhaustion of retries counts as "MCP down" for the decision logic below.

For each MCP that fails:

- Route to `connector-failure-notify.md` with the MCP name and the error.
- Continue checking the rest — collect ALL failures, then notify.

After the verification:

- **If all 4 MCPs are responsive:** continue to the original mode/command logic. Preflight done.
- **If 1+ MCPs are down:** the failure-notify flow has been triggered. Decision per mode:
  - Mode 1 (collection): proceed with the available collectors. Note the missing source on the dated page summary. Don't abort.
  - Mode 2 (execution): if Notion or Orbit are down, ABORT — they're load-bearing. Other MCPs are non-blocking; proceed with what's available.
  - Other commands (preference edits, manual overrides): try to proceed with what's available; abort only if the specific MCP needed for THIS command is down. Escalation runs inside Mode 2 Step 3a — no separate Mode 3 entrypoint exists.

### Step 6 — Confirm parent-page operational sub-pages exist

Verify the persistent operational children of the Notion parent. Create any that are missing; create-or-confirm in this exact order so positions land correctly:

1. **`Run Log` sub-page** — sub-page containing the Run Log inline database per `schemas/run-log-database.md`. Holds one row per routine fire, with a linked detail page per row holding the decision trace. See `writers/run-log.md`.
2. **`Incidents` sub-page** — append-only inline database per `schemas/parent-page.md` § Incidents. Holds connector-failure rows from Tier 4 of `connector-failure-notify.md`.
3. **`Preferences` sub-page** — must be the very last child of the parent. If it's not last, move it to the end.

After creation/verification, the bottom three `child_page` blocks on the parent body (in order) must be: `Run Log`, `Incidents`, `Preferences`. Year heading-toggle blocks, when present, sit above all three.

**Hierarchy note (do NOT enforce in preflight):** preflight does NOT create or touch Year/Month heading-toggle blocks on the parent body, does NOT verify dated sub-page placement inside Month toggles, and does NOT auto-relocate flat dated pages found outside the toggle structure. That's `writers/notion.md`'s job at Mode 1 write time, plus drift surfacing during `modes/monthly-archival.md`. Preflight only confirms the three operational sub-pages (`Run Log`, `Incidents`, `Preferences`) exist and that their `child_page` blocks sit in the correct order at the bottom of the parent body — it does not touch dated content or toggle blocks.

## What preflight does NOT do

- Does NOT execute any morning collection logic, executor logic, or write any task data. That happens after preflight returns control.
- Does NOT modify Preferences.
- Does NOT modify config.
- Does NOT call any tool from outside the source allowlist (4 primary MCPs + 4 read-only reference MCPs per `references/external-doc-access.md` + the read-only Pod Matrix exception per `references/pod-matrix.md`).
- Does NOT fetch the Pod Matrix page. That is fetched by `modes/mode-1-morning-collection.md` Step 2.5 via `references/pod-matrix.md` (cached for the run). Mode 2 and Monthly Archival never fetch it.

## Identity check (also part of preflight, in step 5)

When verifying the Gmail connector, the skill confirms identity:

- Gmail authenticated account should match Preferences canonical email or one of the aliases.

If it does not match, ABORT the run. Email the PM (sent, not drafted): subject `[PM Task Assignment] Identity mismatch — routine cannot proceed`, body `Identity mismatch — Gmail connector authed as [X] but Preferences expects [Y]. Routine cannot proceed.` In routines the connectors are wired at create-time; mismatch means the routine was created on the wrong account. Do not proceed silently.

## Loading verification

If you (Claude) are reading this comment, you have loaded `preflight.md` correctly. After preflight completes, return control to the file that called it (SKILL.md or a mode/command file).
