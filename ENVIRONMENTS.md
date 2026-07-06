# ENVIRONMENTS — Where this skill runs

This skill is designed to behave correctly under two execution surfaces. The mode files and writers do not assume one or the other; they branch on `is_interactive` (set during preflight Step 1) and adapt their question-asking and confirmation behavior accordingly.

## Surface 1 — Routine / Code (headless)

The default mode of operation. Used by:

- **Claude Routines** firing on cron with a baked prompt that loads files from the public skill repo (or local disk). See `ROUTINE-ENTRYPOINTS.md` for the three baked prompts.
- **Programmatic invocations** via the Claude Code SDK / CLI / API, where the skill is loaded as a directory and a prompt is fed in headlessly.
- **Cloud agents** running unattended.

Behavior:

- Never asks the PM a question. There is no human in the loop.
- Ambiguous items become their own row with `Uncertain:` notes in `AI Notes`. The PM resolves them later when they review.
- Logs every decision to the Run Log so the trace is auditable.
- Errors notify the PM via the 4-tier fallback chain in `connector-failure-notify.md`, never via mid-run prompts.

## Surface 2 — Claude Desktop (interactive)

Used when a PM (or an operator) types a manual command — e.g., `PM Task Assignment, run morning` — in a regular Claude Desktop session.

Behavior:

- May ask **one** clarifying question per ambiguous item before falling back to `Uncertain:`. This is the only way the interactive surface behaves differently from headless. Everything else (preflight, writes, run-log, plain-language enforcement) is identical.
- Surfaces preflight progress to the user as it runs (so they see what was checked and what failed).
- May offer the user a chance to fix Preferences mid-flow if a required field is missing — but only with the PM's explicit consent. Never auto-edits.
- All manual commands documented in `invocation-commands.md` are available.

## Detecting which surface is active

Preflight Step 1 sets the in-memory flag:

- `is_interactive = true` if the skill was triggered by a human-typed message in a Claude session.
- `is_interactive = false` if the skill was triggered by a routine / cron / SDK call without a live human.

Mode files read this flag once and behave accordingly. Once set during a single run, the value does not change.

## Loading skill files

Two paths, in priority order:

1. **Local files** — if `/skills/pm-task-assignment/` (or wherever the skill is mounted in the runtime) contains the skill folder, load files from disk. This is how Claude Desktop and Claude Code invocations typically work.
2. **Repo fetch** — fall back to fetching `<REPO_URL>/<filename>` from the public skill repo. This is how routines work when the routine prompt embeds a `<REPO_URL>` placeholder.

If neither path resolves a file, abort the run with a clear error: `Skill file [name] not found locally or on repo. Cannot proceed.`

## What is NOT environment-dependent

These behaviors are identical across both surfaces:

- The 6-step preflight sequence in `preflight.md`
- The source allowlist (5 primary MCPs + 4 read-only references per `references/external-doc-access.md`)
- The plain-language rules in `writers/plain-language.md`
- The 6-section Orbit task body in `schemas/orbit-dq-standard.md`
- Approval semantics: row Status + page-level `Ready for Execution` toggle in Mode 2
- The Notion parent-page hierarchy `Parent → Year → Month → Date`
- Run Log + Incidents writes
- Fixed-Cost Registry writes
- Failure-notify fallback chain

Only the question-asking behavior differs. Everything else is one consistent skill, regardless of how it was launched.
