# Schema — Preferences Page

The Preferences page sits at the bottom of the PM's Notion parent (the page identified in `config.md`). It's a regular Notion page (NOT a database). The skill reads it on every run. The PM edits it manually or via `PM Task Assignment, change my preference: ...`.

## Page identity

| Field | Value |
|---|---|
| Title | `Preferences` |
| Icon | ⚙️ |
| Parent | The PM's Notion parent (from `config.md`) |
| Position | Always the LAST child of the parent (enforced on every Mode 1 run and on monthly archival) |
| Type | Plain page (no database) |

## Content structure (top to bottom)

Headings and plain text. NO toggles — the PM needs to see everything without clicking.

### H1: Preferences — [PM Name]

Opening paragraph:

> "This page is read by the PM Task Assignment skill on every run. Edit any field and save — the skill picks up changes next time it runs. The skill does not self-edit this page except for: (1) appending to the Observed Patterns section over time, and (2) updating timestamps in the Internal State section at the end of each run."

### H2: Identity

| Field | Description |
|---|---|
| **Name** | The PM's full name |
| **Orbit user ID** | Integer (e.g., `935`) — looked up via `mcp__...orbit.list_users` during first-run |
| **Canonical email** | The PM's primary work email (e.g., `aditis@whitelabeliq.com`) |
| **Email aliases** | A bulleted list of all alias addresses that map to this PM. Empty if none. Example: `- aditi@whitelabeliq.com` |

The skill matches the authenticated Slack profile / Gmail account against the canonical email AND any aliases. Any one match = identity confirmed.

### H2: Run Schedule

- **Morning collection run (Mode 1):** [time] IST, daily
- **Execution run (Mode 2):** [time] IST, daily
- **Escalation check:** at end of execution run (fires automatically if Ready toggle is OFF)
- **Monthly archival:** 1st of each month at 6:00 AM IST

Plus a line about manual invocation:

> "You can also fire runs manually: `PM Task Assignment, run morning` / `PM Task Assignment, run execution now`. Both commands are lenient — variations like `PM Task Assignment run my morning` or `pm task assignment, execute` also work."

### H2: Escalation Backup

| Field | Description |
|---|---|
| **Name** | The backup person's full name |
| **Channel** | `Slack` or `email` |
| **Slack handle** | If channel = Slack, the @-handle |
| **Email** | If channel = email, the email address |
| **Notify time** | When to ping them (typically same as execution run time, or a configurable delay) |

### H2: Account Managers

One sub-section per AM. For each:

#### H3: [AM name]

- **Email:** [email]
- **Slack:** @[handle]
- **Preferred channel:** `Slack for quick handoffs, email for paper trail` / `Always Slack` / `Always email` / [custom]
- **Projects (known associations):** list of Orbit project names — auto-updated as new projects surface in matcher's grouping

### H2: Default Communication Preferences

A table:

| Audience | Default Channel | Tone | Default CCs |
|---|---|---|---|
| Clients (external) | Email | Professional, warm | CC relevant AM |
| Account Managers | Slack (or per-AM override above) | Concise, direct | — |
| Team members (India) | Slack | Plain English (4th-5th grade), role-specific terms preserved | — |
| Leadership (Brian, Nishant) | Email | Formal, structured | Each other |

Plus:

- **Email signature:** `[signature text]` — applied to all drafted emails

### H2: Always-Include Rules

A bulleted list of free-text rules the PM provided during setup. Examples:

- Always remind assignees to log their hours at the end of their Slack handoff
- For WordPress work, always include the client's WP admin URL in the Orbit task
- Never CC external clients on internal-facing threads
- When assigning QA, always link the original task being QA'd
- When reassigning due to capacity, briefly explain why in the handoff

Empty if the PM had no rules.

### H2: Exclude From Pod

A bulleted list of names or Orbit user IDs to exclude from pod inference. Default entries auto-populated at first run:

- Brian Gerstner
- Nishant Rana
- All AMs in the AM list above
- The PM themselves

Plus any always-exclude the PM adds (ex-employees, contractors phased out, etc.).

### H2: Observed Patterns (Adaptive Learning)

Populated over time by the skill. Each entry describes a pattern observed in the PM's actions. The PM can review, keep, or remove any entry.

Until the first pattern hits its threshold:

> "No observed patterns yet — collecting samples. Come back in a few weeks."

Once patterns emerge, format:

- **Pattern:** [Description — e.g., "Always emails the AM after receiving client feedback"]
- **Observed:** [N times since the skill started tracking]
- **Promoted to default:** [Yes / No]
- **Remove?** [Edit this page and delete the entry if you don't want it as default]

### H2: Tone Samples

For the plain-language writer's calibration. Either:

> **Status:** Skipped during first-run setup. Using generic professional tone.

OR, when samples have been provided:

> **Sample 1 (Slack to Vijay, 20 April 2026):**
> [verbatim sample text]
>
> **Sample 2 (Email to Caitlin, 22 April 2026):**
> [verbatim sample text]
>
> *(3-5 samples total)*

### H2: Internal State (Skill-managed — do not edit)

Updated by the skill on every run. PM can read but shouldn't edit.

- **Last morning run:** [ISO timestamp]
- **Last execution run:** [ISO timestamp]
- **Last archival:** [ISO timestamp]
- **Scheduled task IDs:**
  - Mode 1: `[ID]`
  - Mode 2: `[ID]`
  - Monthly archival: `[ID]`

### H2: Failures Log (Skill-managed)

Used by `connector-failure-notify.md` Tier 4. Empty by default. Each entry:

- **[Timestamp]** — [Connector] failed: [reason]. Status: `unacknowledged` / `acknowledged on [date]`.

The skill surfaces unacknowledged failures at the start of the PM's next manual invocation.

### H2: How to Update This Page

> "Edit any section directly — changes take effect on the next run.
>
> Or invoke the skill: `PM Task Assignment, change my preference: [your instruction]`. The skill updates only this page, won't fire the morning flow.
>
> Examples:
> - `PM Task Assignment, change my preference: run morning at 10:00 AM instead of 9:30`
> - `PM Task Assignment, change my preference: my new escalation backup is Hiten, ping him on Slack at 11:00 AM`
> - `PM Task Assignment, change my preference: add a rule to always link related Orbit tasks when reassigning`"

## How the skill reads Preferences

On every run:

1. Fetches the Preferences page via `notion-fetch`
2. Parses each section into structured data
3. Uses a flexible parser — tolerate minor formatting variations since the PM may edit freely

The parser looks for section headings (by text match) and extracts the content under each. If a section is missing, use defaults and log a warning (but don't fail).

## How the skill writes Preferences

Two cases:

1. **First-run setup** — creates the entire page from scratch using the user's answers.
2. **`change my preference` command** — targeted update to a specific section. Never rewrites the whole page.

For Observed Patterns, the skill appends new patterns over time. PM can review and remove.

For Internal State, the skill updates timestamp fields at the end of each run.

For Failures Log, the skill appends entries when connectors fail and marks them acknowledged after the next manual invocation.

## What this page does NOT include

- Per-project pod data. Pod is computed dynamically per-run via `synthesis/pod-inference.md` — no stored pod page.
- Task history. The Morning Queue pages serve that purpose.
- Conversation logs. Those live in Claude session history, not here.
- Credentials or tokens. MCP connections are handled by the PM's Claude account, not stored here.
- The Notion parent page ID. That's hardcoded in `config.md` — Preferences cannot override it.
