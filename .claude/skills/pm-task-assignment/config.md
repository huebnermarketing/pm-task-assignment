# config.md — PM Task Assignment

> **MANDATORY: This file is the FIRST thing the skill reads on every invocation. Do not call any tool, do not load any other skill file, do not act on any user message until this file has been read in full and the preflight sequence has run. SKILL.md, every mode file, and every command file all reference this requirement.**

This file holds two things together:

1. **Per-installation defaults** — values you (the deployer) edit before installing on a new PM's machine
2. **Operational rules** — non-negotiable rules about how those values are used

Both are equally important. Read both before proceeding.

---

## Notion parent page

```
DEFAULT_NOTION_PARENT_PAGE_ID = 34c3846840c8808ea886e7e6df15e2af
DEFAULT_NOTION_PARENT_URL    = https://www.notion.so/Test-Run-Ishant-PM-MVP-1-34c3846840c8808ea886e7e6df15e2af
```

### How the skill uses this

- This is the **only** Notion page the skill reads from or writes to. Never any other Notion page.
- Today's dated sub-page is created at the top of this parent on every Mode 1 run.
- Older dated pages live below today's, in reverse chronological order, until they're archived into named-month toggles.
- Month toggles (`April 2026`, `March 2026`, ...) sit below the current month's dated pages.
- The `Preferences` sub-page is **always** the last child of this parent. Its position is enforced on every Mode 1 run and on monthly archival.
- The structure of the parent and its children is defined and enforced per `schemas/parent-page.md`.

### To install for a different PM

1. Replace `DEFAULT_NOTION_PARENT_PAGE_ID` and `DEFAULT_NOTION_PARENT_URL` above with the new PM's Notion page values.
2. (Optionally) Update `DEFAULT_PM_NAME` and `DEFAULT_PM_EMAIL` below for clarity. These are optional — the skill infers identity from Preferences after first-run setup.
3. Ship the entire `PM Automation MVP 1` folder to that PM. They install it on their Claude/Cowork.

See `DEPLOYMENT.md` for the full deployment procedure.

---

## Preferences sub-page

The `Preferences` sub-page sits at the bottom of the Notion parent page above. It is the runtime source of truth for everything specific to the PM running the skill — their identity, run times, escalation backup, AMs, communication defaults, always-include rules, observed patterns, and tone samples.

### Rules — non-negotiable

- The skill **must** read Preferences on every invocation, before any collector, executor, or mode logic runs.
- The skill **must not** bypass Preferences settings under any circumstance — including experimental runs, forced runs, sandbox runs, or runs where the user invokes a different scope.
- The skill **must not** override Preferences from elsewhere (with one exception: `DEFAULT_NOTION_PARENT_PAGE_ID` is set in this config and is not editable from Preferences).
- If `Preferences` does not exist on the Notion parent, the skill **must** route to `first-run-setup.md` immediately and refuse all other actions until setup completes.
- The skill **must not** edit Preferences except through the explicit `PM Task Assignment, change my preference: ...` command and through the observed-patterns logging defined in `synthesis/note-interpreter.md` and `schemas/preferences-page.md`.

The structure of the Preferences page is defined in `schemas/preferences-page.md`.

---

## Source allowlist

> **This skill uses only these MCPs:** Orbit, Gmail, Slack, Fathom, Notion, scheduled-tasks.
>
> **Any other MCP is forbidden** — including any that may seem relevant to a specific signal, including any added to the user's Cowork after installation, including any that the running user explicitly asks the skill to use. The allowlist is closed.
>
> **This rule applies even under experimental scope, forced runs, sandbox runs, or any kind of override.** If a signal seems relevant from a forbidden source, ignore it.

This rule is repeated in SKILL.md and at the top of every collector / executor file. The repetition is deliberate — the skill should never expand its source list.

---

## Identity

The running user's identity is determined from Preferences (after first-run setup completes).

- **Identity check on every invocation:** the skill matches the authenticated Slack profile and Gmail account against the canonical email AND any aliases listed in Preferences. If any one matches, identity is confirmed.
- The canonical email and aliases are stored in Preferences, populated during first-run setup (`first-run-setup.md`).
- Many WLIQ team members have email aliases (e.g., `ishant@whitelabeliq.com` is an alias for `ishantk@whitelabeliq.com`). The skill treats canonical and aliases as the same identity for sender classification, recipient matching, and self-notification.

---

## Failure fallback chain

If any MCP connector fails or returns auth errors, the skill notifies the running PM about the failure. Fallback chain in order:

1. **Slack DM to the PM themselves.** Primary channel.
2. **Email from the PM's Gmail to themselves (sent, not drafted).** Used if Slack is the failed connector or otherwise unreachable.
3. **Bold-red callout at the top of today's dated Notion page.** Used if both Slack and Gmail are unreachable.
4. **Surface at the start of the PM's next manual invocation.** The very first thing the skill says when next invoked is: "Heads up — last [run/escalation] failed because [reason]. Fix this before proceeding: [remediation step]."

Defined in `connector-failure-notify.md`.

The skill **must** trigger this fallback chain whenever Orbit, Gmail, Slack, Fathom, Notion, or scheduled-tasks return errors. The skill **must not** silently fail or proceed without notifying the PM.

---

## Run schedule defaults

These are skill-wide defaults. Each PM's actual schedule lives in their Preferences (set during first-run setup).

```
DEFAULT_MORNING_RUN_TIME    = 09:30 IST   (Mode 1 — collection)
DEFAULT_EXECUTION_RUN_TIME  = 10:45 IST   (Mode 2 — execution)
DEFAULT_ESCALATION_TIME     = 10:45 IST   (collapsed into end of Mode 2)
DEFAULT_MONTHLY_ARCHIVAL    = 06:00 IST   (1st of each month)
```

If a PM doesn't override these in first-run setup, these are used.

---

## Optional defaults (for clarity during installation)

These are not strictly required — the skill infers identity from Preferences. But pre-filling them makes deployment cleaner.

```
DEFAULT_PM_NAME  = (set per installation)
DEFAULT_PM_EMAIL = (set per installation)
```

---

## What this file does NOT contain

- Per-PM communication preferences (those live in Preferences)
- Per-PM AM list (Preferences)
- Per-PM escalation backup (Preferences)
- Tone samples (Preferences)
- Observed patterns (Preferences)
- Credentials, tokens, or secrets (handled by the user's MCP authentication, never in this skill)

This file is the deployment-time configuration only. Everything personal lives in Preferences.

---

## Loading verification

If you (Claude) are reading this comment, you have loaded `config.md` correctly. Continue to `preflight.md` for the next steps. Do not skip preflight. Do not act on the user's message until preflight has completed.
