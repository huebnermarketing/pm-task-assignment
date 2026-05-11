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
| **Orbit user ID** | Integer — looked up via `mcp__...orbit.list_users` during first-run. Example: `935`. |
| **Canonical email** | The PM's primary work email on `@whitelabeliq.com`. Example: `aditis@whitelabeliq.com`. |
| **Email aliases** | A bulleted list of all alias addresses that map to this PM. Empty if none. Example: `- aditi@whitelabeliq.com`. |

The skill matches the authenticated Slack profile / Gmail account against the canonical email AND any aliases. Any one match = identity confirmed.

### H2: Run Schedule

- **Morning collection run (Mode 1):** [time] IST, daily
- **Execution run (Mode 2):** [time] IST, daily — if the Ready toggle is OFF at this time, Mode 2 runs its inline escalation flow (Step 3a) and exits without executing any rows.
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
- **Orbit task body header style:** `professional` (default) | `emoji` — header glyphs used in the 6-section Orbit task body. See `schemas/orbit-dq-standard.md` for the full mapping. Section order and content are unchanged across styles; only the header glyph changes.
- **Gmail post-collection action:** `none` (default) | `mark_read` | `apply_label` | `mark_read_and_apply_label` — what the Gmail collector does to a source message after it lands in the morning queue. See `collectors/gmail.md`.
- **Gmail label name:** `pm-task-assignment/collected` (default) — only used when the post-collection action includes `apply_label`. The label is created in the PM's Gmail on first use if it does not exist.
- **Slack handoff template:** the per-PM template body for team handoff drafts. The format is whatever the PM writes here; the executor does not impose its own structure beyond plain-language enforcement and source-citation rules. Default suggestion (the PM may edit freely):

  ```
  <project name / brief task title>

  What to do: <one sentence, plain English>

  Why: <one sentence — why this matters or who's waiting>

  Context you need: <link to Orbit task, Figma, staging URL, file>

  Please log your hours.
  ```

- **AM communication:** team and AM Slack messages are **never auto-sent** by the skill. Drafts are appended to the row's Outcome on today's dated Notion page after Mode 2 runs; the PM copies and sends from there. The only Slack sends Mode 2 may perform are: (a) the PM self-summary DM, (b) the escalation message to the backup, and (c) a team handoff explicitly approved with a PM note that says "send" AND audience = team.

### H2: Pod Daily Task Block

Controls the daily copy-block appended at the bottom of each dated page after Mode 2 — the one the PM copies into the pod's daily task Slack channel. See `writers/notion.md` — Flow — appending the Pod Daily Task block.

The block is split into two parts:

- **Anchor (skill-managed, not customizable):** the first two blocks the writer emits — a Divider and a Heading-2 with the exact text `Pod Daily Task — copy into Slack`. The anchor lets the writer find the existing block on a same-day rerun to refresh it. Renaming or removing the anchor breaks rerun idempotency.
- **Body (PM-customizable):** every block below the anchor. Defined by the `pod_block_layout` DSL field below. The DSL lets the PM choose block types, order, nesting, and which template fields each block uses.

The skill reads every field on every Mode 2 run. Token substitution is literal — anything that isn't a recognized `{token}` is kept verbatim, including punctuation, whitespace, and emoji. Unrecognized tokens are left as-is in the output (so a typo in the template surfaces visibly in Notion). If a field is missing or empty on the Preferences page, the default below is used.

#### Behavior switches

- **Pod daily task enabled:** `true` (default) | `false`. When `false`, Mode 2 skips the digest build entirely. The PM self-summary's closing line is omitted.
- **Pod daily task Slack channel:** free-text reference, e.g., `#pod-daily-tasks`. The skill does NOT post here — this is a reminder for the PM about where to paste. Empty by default.
- **Include reassignments:** `true` (default) | `false`. When `false`, only newly-created Orbit tasks appear in the digest; reassignments are excluded.

#### Templates

- **Caption template** — rendered as the paragraph between the H2 and the date toggle. No tokens. Default:

  ```
  Copy everything inside the date toggle below and paste into the pod's daily task Slack channel. Each line is one task you assigned today.
  ```

- **Date label template** — rendered as the title of the Heading-1 toggle wrapping the task list. Tokens: `{date_dd}`, `{date_mm}`, `{date_yyyy}`, `{date_month_name}`, `{date_iso}` (`YYYY-MM-DD`). Default:

  ```
  {date_dd}/{date_mm}/{date_yyyy}
  ```

  Example renderings on 11 May 2026 — default → `11/05/2026`; `{date_dd} {date_month_name} {date_yyyy}` → `11 May 2026`.

- **Task heading template** — rendered as the H2 block per qualifying row. Tokens: `{client}`, `{project}`, `{task_title}`, `{task_id}`. Default:

  ```
  {client}{project}{task_title}{task_id}
  ```

  The no-space default intentionally mirrors Orbit's URL-preview concat in Slack — when the PM pastes the task URL on the next line, Slack renders the same string in its preview. If the PM prefers a readable form, try `{client} — {task_title} (#{task_id})`.

- **Task line template** — rendered as the to-do block body directly under each task heading. Tokens: `{orbit_task_url}`, `{assignee_first_name}`, `{assignee_full_name}`, `{task_id}`, `{task_title}`. Default:

  ```
  {orbit_task_url} - {assignee_first_name}
  ```

  Literal hyphen, single spaces. The full Orbit URL must be present in the rendered output — Slack renders it as a clickable link on paste.

#### Token reference

| Token | Source | Available in |
|---|---|---|
| `{date_dd}` | Today's date, zero-padded day | Date label |
| `{date_mm}` | Today's date, zero-padded month | Date label |
| `{date_yyyy}` | Today's date, 4-digit year | Date label |
| `{date_month_name}` | Today's date, English month name (`May`) | Date label |
| `{date_iso}` | Today's date in `YYYY-MM-DD` | Date label |
| `{client}` | Row's Orbit client name | Task heading |
| `{project}` | Row's Orbit project name | Task heading |
| `{task_title}` | Row's Orbit task title | Task heading, Task line |
| `{task_id}` | Trailing numeric segment of the Orbit task URL | Task heading, Task line |
| `{orbit_task_url}` | Full URL to the Orbit task | Task line |
| `{assignee_first_name}` | Resolved assignee's first name | Task line |
| `{assignee_full_name}` | Resolved assignee's full name | Task line |

#### Block layout DSL (advanced)

`pod_block_layout` controls which Notion blocks the writer emits BELOW the anchor and how they nest. Store it on the Preferences page inside a code block (any language hint — the writer reads the raw text).

**Syntax rules**

- One block per line.
- Format: `<block_type>` for blocks with no content (e.g., `divider`), or `<block_type>: <content>` for blocks with text content.
- Indentation = nesting. Use two spaces per nesting level. A line indented under a `toggle_h1` becomes its child; same for `toggle_h2`, `toggle_h3`, `toggle`, `to_do`, `bulleted_list_item`, `numbered_list_item`. Headings and paragraphs cannot have children in Notion — indenting under them is rejected at parse time and logged.
- `for_each_task:` introduces a loop. Lines indented under it are repeated once per qualifying row, in matcher order. Exit the loop by dedenting back to its level. The loop may appear at any nesting depth.
- Inline content supports two kinds of references:
  - **Tokens** — `{client}`, `{date_dd}`, `{orbit_task_url}`, etc. Available tokens depend on context: outside `for_each_task`, only date tokens are available; inside, both date tokens and task tokens are available. See the Token reference above.
  - **Template field references** — `{{caption_template}}`, `{{date_label_template}}`, `{{task_heading_template}}`, `{{task_line_template}}`. The writer expands these to the rendered value of the named template field (token substitution is applied to the template field's content first, in the right context).
- Lines starting with `#` are comments and ignored.
- Blank lines are ignored.

**Supported block types**

| Block type | Content? | Nestable? | Notes |
|---|---|---|---|
| `divider` | No | No | Horizontal rule |
| `paragraph` | Yes | No | Plain text block |
| `heading_1`, `heading_2`, `heading_3` | Yes | No | Non-toggle heading |
| `toggle_h1`, `toggle_h2`, `toggle_h3` | Yes (the title) | Yes | Heading rendered as a toggle |
| `toggle` | Yes (the title) | Yes | Regular toggle |
| `callout` | Yes | Yes | Prefix with an emoji + space to set the icon, e.g. `callout: 📋 Reminder text`. Default icon if none given: 💡 |
| `quote` | Yes | No | Indented quote block |
| `bulleted_list_item` | Yes | Yes | |
| `numbered_list_item` | Yes | Yes | |
| `to_do` | Yes | Yes | Unchecked by default. Prefix the content with `[x] ` for default-checked (rare for this use case). |

**Reserved directive**

| Directive | Behavior |
|---|---|
| `for_each_task:` | Block-loop. Children are repeated per qualifying row. |

**Default layout** (used if the field is missing, empty, or the code block isn't found):

```
paragraph: {{caption_template}}
toggle_h1: {{date_label_template}}
  for_each_task:
    heading_2: {{task_heading_template}}
    to_do: {{task_line_template}}
```

This renders to the screenshot reference: caption paragraph → date toggle wrapping the task list → per task an H2 heading + a to-do block.

**Example overrides**

*PM wants a callout warning above the task list and bullet items instead of to-dos:*

```
callout: 📋 Daily pod task list — paste this into #pod-daily-tasks
toggle_h1: {{date_label_template}}
  for_each_task:
    heading_3: {{task_heading_template}}
    bulleted_list_item: {{task_line_template}}
```

*PM wants no toggle at all and a flat per-task list with custom punctuation:*

```
paragraph: {{caption_template}}
heading_2: Tasks for {date_dd} {date_month_name}
for_each_task:
  to_do: {client} — {task_title} → {orbit_task_url} ({assignee_first_name})
```

*PM wants the task heading and to-do nested under a per-task toggle (so each task collapses individually):*

```
paragraph: {{caption_template}}
toggle_h1: {{date_label_template}}
  for_each_task:
    toggle_h3: {{task_heading_template}}
      to_do: {{task_line_template}}
      paragraph: Assignee: {assignee_full_name}
```

**Validation behavior**

- Unknown block types → that line is skipped; the writer logs `pod_layout_unknown_block: <type>` and continues with remaining lines.
- Indentation under a non-nestable block (e.g., a child indented under `heading_2`) → the child is rendered as a sibling at the parent's level; the writer logs `pod_layout_illegal_nesting: <parent_type>` and continues.
- Missing `for_each_task:` entirely → the digest renders with no per-task content; the writer logs `pod_layout_no_task_loop`. The default closing line in the PM self-summary changes to: `Pod Daily Task block rendered but has no task loop — see preferences.`
- Layout parse error (malformed indentation, unterminated content) → fall back to the documented default layout for this run; log `pod_layout_parse_failed: <reason>`. Do not abort Mode 2.

### H2: AM Daily Ping Block

Controls a second copy-block appended directly below the Pod Daily Task block on each dated page after Mode 2. One short ping per AM, drafted by the skill but **never auto-sent** — the PM DMs each AM manually.

Like Pod Daily Task, this block has a fixed anchor and a customizable body. The body is rendered from `am_block_layout` using the same DSL.

- **Anchor (skill-managed, not customizable):** Divider + Heading-2 with the exact text `AM Ping Drafts — copy into Slack`. Anchor matters for same-day rerun detection.
- **Body:** PM-defined layout, default = one Heading-3 per AM + one paragraph holding the 3-line ping body.

The per-AM body is **drafted by the skill, not a pure template** — it summarizes the day's qualifying rows for that AM's projects through `writers/plain-language.md`, using PM tone samples. The PM controls the body shape via `am_ping_body_guidance` (free-text instructions the skill follows when drafting).

#### Behavior switches

- **AM ping enabled:** `true` (default) | `false`. When `false`, Mode 2 skips this block entirely.
- **Include reassignments:** `true` (default) | `false`. When `false`, only newly-created Orbit tasks count toward an AM's daily ping eligibility.
- **Quiet AMs:** bulleted list of AM names to skip from the digest even when they have qualifying work today. Empty by default. Useful for AMs who explicitly opted out of daily pings.

#### Templates

- **AM ping caption template** — paragraph rendered above the per-AM list. No tokens. Default:

  ```
  One short ping per AM, wrapping today's work on their projects. DM each AM at their handle below. The skill never auto-sends these — copy and send yourself.
  ```

- **AM heading template** — rendered as the per-AM heading block. Tokens: `{am_name}`, `{am_first_name}`, `{am_last_name}`, `{am_slack_handle}`, `{am_email}`. Default:

  ```
  {am_name} (@{am_slack_handle})
  ```

- **AM ping body guidance** — free-text instructions the skill follows when drafting the per-AM ping body. NOT a template (no token substitution applied to the guidance itself); it's prompt-style direction handed to `writers/plain-language.md` along with the row context for the AM's qualifying rows. Default:

  ```
  Draft a 3-line ping for the AM. The 3 lines must be:
  Line 1: which tasks were picked up today for the AM's projects, and who on the team is on each one. One sentence.
  Line 2: the target deliverable, deadline, or expected outcome — only if the row context makes this concrete. If unclear, replace with one line of what the AM should expect next.
  Line 3: a brief follow-up commitment (e.g., "Will keep you posted." or "Ping me if priorities shift.").
  Tone: normal professional English, concise, confident. Do not greet the AM by name (the heading already shows it). Do not include URLs or Orbit task IDs — those belong in the row detail pages, not the ping.
  ```

  The PM may edit this guidance freely; the skill regenerates the per-AM body on every Mode 2 run using whatever guidance is currently on the Preferences page.

#### Token reference (in addition to date tokens)

| Token | Source | Available in |
|---|---|---|
| `{am_name}` | AM's full name from the Account Managers section | Caption, AM heading |
| `{am_first_name}` | AM's first name | AM heading |
| `{am_last_name}` | AM's last name | AM heading |
| `{am_slack_handle}` | AM's Slack handle from Preferences | AM heading |
| `{am_email}` | AM's email | AM heading |
| `{am_tasks_count}` | Number of qualifying tasks for this AM today | AM heading |
| `{am_projects_csv}` | Comma-separated list of project names this AM owns that had work today | AM heading |

Task tokens like `{client}` or `{task_id}` are NOT available inside the AM context — pings are AM-level, not task-level. Use these tokens only inside a `for_each_task:` loop in the Pod Daily Task layout.

#### Block layout DSL

`am_block_layout` follows the exact same DSL grammar as `pod_block_layout` (see Pod Daily Task Block → Block layout DSL). The only new directive is `for_each_am:` which replaces `for_each_task:` — it iterates over AMs that have at least one qualifying row today.

**Supported block types:** identical to Pod Daily Task. **Reserved directive:** `for_each_am:` (the only loop available in this layout — `for_each_task:` is rejected at parse time with `am_layout_unknown_directive: for_each_task`).

**Default layout:**

```
paragraph: {{am_ping_caption_template}}
for_each_am:
  heading_3: {{am_heading_template}}
  paragraph: {{am_ping_body}}
```

`{{am_ping_body}}` is the synthesized 3-line draft for the current AM in the loop. It is the only template-field reference that resolves to LLM-drafted content rather than substituted text.

**Example overrides**

*PM wants each AM wrapped in a per-AM toggle so the page collapses cleanly:*

```
paragraph: {{am_ping_caption_template}}
for_each_am:
  toggle_h3: {{am_heading_template}}
    paragraph: {{am_ping_body}}
```

*PM wants a callout above with explicit reminder, plus the AM's projects listed under the heading:*

```
callout: 📨 Copy each ping below into a DM to the AM. Do not auto-send.
for_each_am:
  heading_3: {am_name} — covers {am_projects_csv}
  paragraph: {{am_ping_body}}
  paragraph: DM handle: @{am_slack_handle}
```

**Validation behavior**

Same as Pod Daily Task layout — unknown block types skip with log, illegal nesting renders as sibling with log, parse failure falls back to default layout, missing `for_each_am:` renders with no per-AM content and logs `am_layout_no_am_loop`.

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
