# Routine Entrypoints

This file holds the three exact prompts to paste into Claude Routines when deploying the PM Task Assignment skill for a new PM. Each routine is a cron-fired headless agent that loads the skill from a public GitHub repo OR from a local skill directory (whichever is available — see `ENVIRONMENTS.md`) and executes a single mode end-to-end. No mode performs any work outside its own scope.

Manual interactive runs (Claude Desktop) use the same skill files — see `invocation-commands.md` for the commands.

> Three routines, three modes. Routine 1 collects, Routine 2 executes, Routine 3 archives. Nothing else.

---

## Pre-deploy Checklist

Before creating any routine, the operator must have:

- [ ] **PM's Notion parent page ID** — the 32-character UUID (or the share-link short ID) of the PM's parent page that holds Preferences + dated queue pages + Run Log database.
- [ ] **PM's Preferences page URL** — the public Notion URL of the Preferences page on that parent.
- [ ] **MCP authentication confirmed** for the PM's Claude account — all 5 allowlisted MCPs connected and authenticated. The skill's source allowlist (per `SKILL.md` and `config.md`) is closed; no routine prompt may add to it or remove from it:
  - [ ] Orbit
  - [ ] Gmail
  - [ ] Slack
  - [ ] Fathom
  - [ ] Notion
- [ ] **Repo URL** of the public GitHub repo holding the skill — referred to below as `<REPO_URL>` (raw-content base, e.g. `https://raw.githubusercontent.com/<org>/<repo>/main`).
- [ ] **Routine TZ — must be IST.** The skill is timezone-locked to **Asia/Kolkata (Indian Standard Time, UTC+05:30)**. All cron expressions, all Preferences-time fields, all Run Log timestamps, and every "today / yesterday / previous-month" calculation in this skill are computed in IST — never UTC, never GMT, never the routine runner's local TZ. If the Routines runner does not natively interpret cron in IST, convert each cron line below from IST to the runner's TZ at routine-creation time so the wall-clock IST fire time stays the same. The skill itself does NOT auto-convert TZ at runtime; the routine runner is responsible for delivering an IST wall-clock fire.

If any item is missing, do not create routines. Stop and resolve first.

---

## Routine 1 — Mode 1 (Morning Collection)

Purpose: pull overnight signals from Orbit / Gmail / Slack / Fathom, write a dated queue page on the Notion parent, await PM approval (PM acts in their own time, not in this routine).

- **Cron (default):** `30 9 * * *` (09:30 IST daily)
- **Override:** if Preferences specifies a different morning run time, use that — still in IST.
- **Connectors required:** Orbit, Gmail, Slack, Fathom, Notion (all 5 of the closed allowlist).

### Prompt template

```
You are running the PM Task Assignment skill in Mode 1 (Morning Collection) as a headless scheduled routine. No human is in the loop — do not ask questions, do not wait for confirmation, decide and act per the skill files.

Strict-skill rule: this prompt does NOT override skill behavior. Where this prompt and any skill file disagree, the skill file wins. Allowlist, retry policy, plain-language rules, source-citation, parent-page hierarchy (Year → Month → Date), and the 6-section Orbit task body are all defined in the skill files — follow them as written.

Time zone: the skill is timezone-locked to Asia/Kolkata (Indian Standard Time, UTC+05:30). Every "today / yesterday / overnight window / Preferences run-time" calculation MUST be performed in IST. Do not use UTC, GMT, or the runner's local TZ for any date math, page title, log timestamp, or window boundary. Page titles use IST date in "DD Month YYYY" format (e.g., "30 April 2026"); Run Log "Started"/"Finished" timestamps are recorded in IST with the timezone offset suffix.

Load and follow these files in order. Prefer local files when present (Claude Code / SDK invocations); otherwise fetch from the public skill repo (Claude Routines fired by cron):
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/config.md
3. <REPO_URL>/preflight.md
4. <REPO_URL>/modes/mode-1-morning-collection.md

Per-PM injected values:
NOTION_PARENT_PAGE_URL=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Connectors available in this routine (all 5, closed allowlist): Orbit, Gmail, Slack, Fathom, Notion. Do NOT skip any collector. Do NOT use any MCP outside this list, even if it appears authenticated.

Execute Mode 1 end-to-end:
- Run preflight Steps 1–6 against the Notion parent and Preferences page (all in IST).
- Collect overnight signals from all 5 source connectors per Preferences. Window boundary in IST.
- Synthesize, group by project, recommend assignees by role-fit only (no availability check).
- Write today's dated queue page on the Notion parent per writers/notion.md, placed at Parent → <Year> → <Month> → <DD Month YYYY> per schemas/parent-page.md (the writer creates the Year and Month container sub-pages on demand if missing).
- Append a Run Log row + linked detail page via writers/run-log.md.

Apply the retry policy in connector-failure-notify.md to every MCP call: 4 attempts total (1 + 3 retries) with 2s/5s/15s incremental backoff. Retry only on transient errors (timeout, 5xx, 429, connection reset). Permanent errors (4xx auth, 404, validation) skip retry and fall through to the failure chain. Log every retry attempt to the Run Log detail page.

Do NOT run Mode 2. Do NOT run Monthly Archival. Do NOT execute any task assignments — Mode 1 only writes the queue; the PM approves later; Mode 2 executes.

Exit when the queue page is written and the Run Log row + detail page exist.
```

---

## Routine 2 — Mode 2 (Execution)

Purpose: read PM-approved rows from today's dated queue, execute the corresponding Orbit / Slack / Gmail actions, Slack the PM a summary.

- **Cron (default):** `45 10 * * *` (10:45 IST daily)
- **Override:** if Preferences specifies a different execution run time, use that — still in IST.
- **Connectors required:** Orbit, Gmail, Slack, Fathom, Notion (all 5 of the closed allowlist; Fathom included for late-arriving meeting context lookups).

### Prompt template

```
You are running the PM Task Assignment skill in Mode 2 (Execution) as a headless scheduled routine. No human is in the loop — do not ask questions, do not wait for confirmation, decide and act per the skill files.

Strict-skill rule: this prompt does NOT override skill behavior. Where this prompt and any skill file disagree, the skill file wins. Allowlist, retry policy, plain-language rules, source-citation, the 6-section Orbit task body, and approval semantics (row Status + page-level Ready toggle) are all defined in the skill files — follow them as written.

Time zone: the skill is timezone-locked to Asia/Kolkata (Indian Standard Time, UTC+05:30). "Today's queue page" is resolved by IST date. Run Log "Started"/"Finished" timestamps are in IST with offset suffix. Do not use UTC / GMT / runner-local TZ for any date math.

Load and follow these files in order. Prefer local files when present (Claude Code / SDK invocations); otherwise fetch from the public skill repo (Claude Routines fired by cron):
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/config.md
3. <REPO_URL>/preflight.md
4. <REPO_URL>/modes/mode-2-execution.md

Per-PM injected values:
NOTION_PARENT_PAGE_URL=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Connectors available in this routine (all 5, closed allowlist): Orbit, Gmail, Slack, Fathom, Notion. Do NOT use any MCP outside this list.

Execute Mode 2 end-to-end:
- Run preflight Steps 1–6 against the Notion parent and Preferences page.
- Resolve today's dated queue page via Parent → <Year> → <Month> → <DD Month YYYY> (IST). If the page-level Ready toggle is OFF, fire the inline escalation flow defined in modes/mode-2-execution.md Step 3a and exit.
- If Ready is ON: read every row's Status and PM Notes. Resolve note intent via synthesis/note-interpreter.md. Execute Approved rows and rows with PM notes via the executors (executors/orbit.md, executors/email.md, executors/slack.md).
- Write each row's Outcome + Status back via writers/notion.md. Apply writers/plain-language.md and writers/source-citation.md to all team-facing outputs.
- Slack the PM a completion summary.
- Append a Run Log row + linked detail page via writers/run-log.md.

Apply the retry policy in connector-failure-notify.md to every MCP call: 4 attempts total (1 + 3 retries) with 2s/5s/15s incremental backoff. Retry only on transient errors. Log every retry attempt to the Run Log detail page.

Do NOT run Mode 1. Do NOT run Monthly Archival. Do NOT collect new signals — Mode 2 only acts on rows already written by Mode 1.

Exit when execution finishes and the Run Log row + detail page exist.
```

---

## Routine 3 — Monthly Archival

Purpose: on the 1st of each month, verify the `Parent → Year → Month → Date` hierarchy on the Notion parent page is intact for the previous month, surface any drift in the Run Log, and reorder Year/Month/Date children as needed.

- **Cron (default):** `0 6 1 * *` (06:00 IST on the 1st of each month, in IST wall-clock)
- **TZ:** IST only. The "previous month" is computed by IST date — never UTC/GMT — so a fire on `1 May 2026 06:00 IST` archives April 2026 regardless of the runner's local TZ.
- **Connectors required:** Notion. (No collectors, no executors needed.)

### Prompt template

```
You are running the PM Task Assignment skill's Monthly Archival routine as a headless scheduled run. No human is in the loop — do not ask questions, do not wait for confirmation, decide and act per the skill files.

Strict-skill rule: this prompt does NOT override skill behavior. Hierarchy invariants (Parent → Year → Month → Date), drift handling (surface, don't auto-move), and ordering rules are defined in schemas/parent-page.md and modes/monthly-archival.md — follow them as written.

Time zone: the skill is timezone-locked to Asia/Kolkata (Indian Standard Time, UTC+05:30). Compute "previous month" in IST. Run Log timestamps in IST with offset suffix. Do not use UTC / GMT / runner-local TZ for the month-boundary calculation.

Load and follow these files in order. Prefer local files when present (Claude Code / SDK invocations); otherwise fetch from the public skill repo (Claude Routines fired by cron):
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/config.md
3. <REPO_URL>/preflight.md
4. <REPO_URL>/modes/monthly-archival.md

Per-PM injected values:
NOTION_PARENT_PAGE_URL=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Connectors used in this routine: Notion only. (Other allowlisted MCPs may be authenticated but are NOT called by archival.)

Execute Monthly Archival end-to-end:
- Run preflight against the parent page (Preferences verify-only).
- Verify the previous-month container exists at Parent → <Year> → <Month> (computed in IST, e.g., a 2026-05-01 fire targets 2026 → April).
- Sweep for misplaced dated pages from the previous month: at the parent level, under a Year (skipping Month), in a wrong Month, or under any legacy `[Month YYYY]` toggle block. For each, log a drift entry in the Run Log — do NOT auto-move.
- Reorder Year / Month / Date children to descending where drifted.
- Verify Run Log + Incidents + Preferences are the last three children of the parent, in that order.
- Append a Run Log row + linked detail page via writers/run-log.md, including every drift item surfaced.

Apply the retry policy in connector-failure-notify.md to every Notion MCP call: 4 attempts total (1 + 3 retries) with 2s/5s/15s incremental backoff. Retry only on transient errors. Log retry attempts.

Do NOT run Mode 1. Do NOT run Mode 2. Do NOT collect or execute anything. Do NOT auto-migrate legacy toggle blocks — surface them in the Run Log only.

Exit when archival verification and the Run Log row + detail page are complete.
```

---

## Validation Step

After all 3 routines are created in Claude Routines:

1. Open a regular (non-routine) Claude session with the same MCPs authenticated.
2. Run the manual `validate setup` command per `invocation-commands.md`, pointing at the same Preferences page URL.
3. Confirm preflight passes — parent page reachable, Preferences page parsed cleanly, all 5 MCPs respond (Orbit, Gmail, Slack, Fathom, Notion), Run Log + Incidents + Preferences present in the correct order at the bottom of the parent (or that preflight Step 6 created them on first run).
4. If `validate setup` reports errors, fix before relying on the routines.

Optional: trigger Routine 1 manually once via the Claude Routines UI ("run now") and inspect the resulting queue page + Run Log row to confirm shape.

---

## Editing the Schedule Later

A PM may want to change their morning or execution run time after deployment.

- **PM action:** edit the run-time field in their Preferences page on Notion.
- **Operator action (also required):** edit the cron line for the corresponding routine in the Claude Routines UI.

The skill, while running inside a routine, **cannot** modify the routine's cron schedule from the inside. Updating Preferences alone is not enough. Both edits are needed for the schedule to actually shift.

If the operator forgets to update the cron, Mode 1 / Mode 2 will continue firing at the old time; the skill will detect the mismatch on its next preflight (Preferences-time vs. actual fire-time delta) and emit a warning into the Run Log detail page, but it will still execute the run.
