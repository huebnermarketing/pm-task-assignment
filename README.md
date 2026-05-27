# PM Task Assignment — Installation & Distribution

A Claude skill for WLIQ Project Managers. Gives every PM their pre-office morning hour back.

## What's in this package

A folder of Markdown files that together form the skill. Claude reads `SKILL.md` first when the skill is triggered, which immediately routes to `config.md` (mandatory first load) and `preflight.md` (mandatory pre-run sequence). Supporting files load on demand. No build step, no compilation. Drop the folder in, Claude runs it.

```
PM Automation MVP 1/
├── SKILL.md                         ← main entry — points to config.md first
├── README.md                        ← this file
├── DEPLOYMENT.md                    ← how to deploy to a new PM (edit config, ship, install)
├── config.md                        ← MANDATORY first load — Notion parent + operational rules
├── preflight.md                     ← MANDATORY pre-run sequence (6 steps)
├── connector-failure-notify.md      ← 3-tier failure fallback chain
├── first-run-setup.md               ← 9-field conversational/template setup
├── invocation-commands.md           ← command syntax + lenient routing
├── modes/                           ← Mode 1 (with Step 3a Orbit-first priority pass), Mode 2, monthly archival
├── collectors/                      ← Orbit, Gmail (primary collection); Fathom (enrichment-on-demand, lazy)
├── synthesis/                       ← matcher (Job 4b Pass 1 Gmail cross-link + Pass 2 Fathom enrichment fetch), pod inference, note interpreter
├── executors/                       ← Orbit, Email (sent-email exceptions), Slack (outbound-send only)
├── writers/                         ← Notion writer, plain-language splitter, source citation
├── schemas/                         ← parent page, Preferences, Morning Queue DB, row detail, Orbit DQ
└── references/                      ← Pod Matrix, due-date categories, status values, external-doc access
```

**Important deployment files:**

- `config.md` — contains the hardcoded Notion parent page ID. Edit this file when deploying to a new PM. See `DEPLOYMENT.md` for the procedure.
- `preflight.md` — the skill runs this 6-step sequence on every invocation before any other logic. Don't skip when reviewing.
- `connector-failure-notify.md` — when an MCP fails, the skill notifies the PM through 3 tiers: self-email (sent, not drafted) → Notion red callout → Incidents sub-page (with repeat-failure email escalation to backup).

## Prerequisites

Each PM needs:

1. A Claude account with Cowork enabled.
2. MCP connections to: Orbit, Gmail, Notion (3 primary collection MCPs, all authenticated) + Fathom (enrichment-on-demand — matcher Job 4b Pass 2 lazy-fetches meeting context when a primary signal references a call; failure is non-blocking). Plus Slack (by Salesforce) for outbound-send only — Mode 2 sends team handoffs + AM pings to Slack ONLY when the PM leaves an explicit `send` note on the row + audience matches `team` or `am`. Optional read-only references: Google Drive/Docs/Sheets, SharePoint (used on demand when primary signals link to those documents — see `references/external-doc-access.md`).
3. A Notion workspace and a parent page where the skill will write morning queues.
4. Three Claude Routines configured per `ROUTINE-ENTRYPOINTS.md` (Mode 1, Mode 2, Monthly Archival). Each routine fires on its own cron — there is no in-skill scheduled-tasks dependency.

## Installation

1. Unzip or copy the `PM Automation MVP 1` folder into your Claude skills directory. On Windows that's typically under `%APPDATA%\Claude\skills\` or wherever Cowork lists your skill folder.
2. Restart Claude / Cowork so the skill is registered.
3. In Claude, type: `PM Task Assignment, run morning`. This triggers the first-run setup (since no Preferences page exists yet).
4. Answer the 10 setup questions. The skill writes them to a new `Preferences` sub-page under the Notion parent you specify.
5. The skill also registers two scheduled tasks on your configured times: Mode 1 (morning collection) and Mode 2 (execution).

From day 2 onward the skill runs automatically at your configured times. You only type a command when you want to manually fire a run, change a preference, or update tone samples.

## Distribution

- **To share with another PM:** Copy the `PM Automation MVP 1` folder. Send via a chat app, Orbit attachment, or email. The receiving PM follows the same installation steps above. Their Preferences will be their own — the skill's behavior adapts per user.
- **No per-PM code changes needed.** The skill is generic. First-run setup collects everything specific to each PM.

## First installer

Aditi Singh (subject to change based on availability). Aditi will install this package, run first-run setup, and then run the skill in parallel with her manual morning process for 4–6 weeks. Accuracy and behavior are compared daily. Once validated, distribute to the rest of the PM team.

## What happens on first run

The skill detects no `Preferences` page exists and asks 10 questions conversationally:

1. Your name + Orbit user ID (or email so the skill can look up Orbit ID)
2. Which Notion page should be the parent for your morning queues
3. Morning collection run time (default 9:30 AM IST)
4. Execution run time (default 10:45 AM IST)
5. Escalation backup — name + channel + time to notify them
6. Account manager(s) — names, canonical emails (load-bearing for the Mode 1 Step 3a priority pass), email aliases, optional Slack handles for the AM-ping send path
7. Default email preferences — CCs, signature, tone
8. Default handoff tone for team handoffs
9. Always-include rules — e.g., "always remind assignees to log hours"
10. Tone calibration samples — optional, skippable

All answers are written to the `Preferences` sub-page at the bottom of the Notion parent. That page is PM-editable. The skill reads it on every run.

## What happens every morning (after setup)

- **9:30 AM IST** (or your configured time): Mode 1 fires. Skill runs Step 3a (Orbit-first priority pass — AM-handed parent tasks due today) then Step 3b (parallel Orbit-normal + Gmail). Synthesizes. During synthesis the matcher lazily fetches Fathom enrichment for any signal that references a meeting (no eager Fathom collection). Writes today's dated sub-page at the top of your Notion parent with an inline Morning Queue database — priority-lane rows sort at the top. No prompts. No input required.
- **You at \~10:00 AM**: Open Notion. Read the queue. For each row, either flip Status to `Approved`, drop a note in PM Notes (including the optional `send` note + audience match if you want Slack auto-send), or leave it at `Recommended Action` or `Skip. No Action Needed`. Flip the `Ready for Execution` toggle at the top when done.
- **10:45 AM IST** (or your configured time): Mode 2 fires. Reads the Ready toggle. If ON, executes every `Approved` row and every row with a note. Writes Done outcomes. Writes a completion-summary callout block at the top of today's dated page (no DM — you're already in Notion).
- **If Ready toggle is OFF at 10:45**: Skill emails your configured escalation backup with a link to the page.

## Non-negotiable rules (enforced by this skill)

- Plain-language English (4th–5th grade) ONLY for handoff drafts to the India delivery team and Orbit task bodies. Role-specific technical terms preserved. PMs and AMs get normal professional English.
- Nothing client-facing sends without your explicit approval.
- Every sourced document is cited by filename in the output.
- V3 Notion pages are read-only reference. This skill does not edit them. See `references/v3-context.md`.
- No availability checking. Skill recommends the most suitable person by role; you override via notes if they're unavailable.
- Mode 1 never asks questions. Unclear items become their own rows with `Uncertain:` notes.

## Troubleshooting

**"The skill didn't fire this morning."**
Check the Mode 1 Claude Routine — it may be paused or its cron may have drifted. Type `PM Task Assignment, run morning` to fire Mode 1 manually for today. Re-enable or fix the routine for tomorrow.

**"A collector failed."**
If a primary MCP source is unavailable during Mode 1 (Orbit, Gmail, or Notion down/auth expired), the skill continues with what it has and writes a note on today's page: "Gmail unavailable — manual check needed for the last <lookback_window>." Fix the auth and the next run will pick up everything that was missed (dynamic lookback based on last-run timestamp). If Fathom is down, the skill logs it to the Incidents page only (non-blocking — primary signals still flow; meeting-enrichment context is just skipped for that run).

**"Mode 2 didn't execute even though I flipped Ready."**
Check that Ready was ON before the scheduled execution time. If not, type `PM Task Assignment, run execution now` to fire Mode 2 manually.

**"I want the skill to stop running while I'm on planned leave."**
Pause the Mode 1 / Mode 2 Claude Routines from the Routines UI for the duration of your leave; re-enable them on return. The skill itself does not pause from inside a routine.

## Support

Project owner: Ishant Kulshreshtha (ishantk@whitelabeliq.com)
Design source of truth: [PM Automation MVP 1](https://www.notion.so/3493846840c8805b9424f63dcdc9b9ce)
Long-term architecture reference (read-only): [PM Automations V3](https://www.notion.so/3433846840c88082b79bf6af6c89a39f) — see `references/v3-context.md` for what lives there and how this skill relates to it.
