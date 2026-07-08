# Signal Provenance Classification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tag every Gmail and Orbit signal with an explicit provenance (`am_relay` / `am_direct` / `client_direct` / `client_orbit_comment` / `am_orbit_comment` / `null`) and a freshness flag (`row.is_fresh`), so Summary/AI-Notes phrasing never conflates AM-relay with direct-client contact and never claims "overnight" for a stale signal — the exact compound bug found auditing the 2026-07-07 Mode 1 run (Naz Lax #8081). Also closes the follower-only visibility gap by generalizing the fixed-cost-only Orbit-notification-mail parser to all projects.

**Architecture:** This is a **prompt-programmed skill** — implementation = editing markdown behavior files that Claude executes at runtime. There is no compilable code and no test runner; each task's "test" is a structural verification (grep for required anchors, cross-file name consistency). Spec: `docs/superpowers/specs/2026-07-08-provenance-classification-design.md` (read the referenced §§ per task).

**Tech Stack:** Markdown behavior files; Gmail MCP (`gmail_read_thread` — already fetched, no new calls); Orbit MCP (`get_activity_log`, `get_task_details` — already fetched, no new calls).

## Global Constraints

Copied from the spec + SKILL.md non-negotiables. Every task inherits these:

- **DO NOT COMMIT.** No `git add`, no `git commit`, anywhere. Krishna reviews and commits manually. A task ends with edited files in the working tree, verified.
- **Five locked verbs only** (SKILL.md rule 18). This design changes wording only — no task may add, remove, or reroute a verb, except the one explicitly-scoped new Job 5 branch in Task 5 (follower-only signals → `Flag`, itself one of the five existing verbs).
- **Rule 20:** empty queue is valid. No task here invents rows.
- **Rule 22:** user-facing project references use `project_number` as `#<project_number>`, never internal `id`/`url_slug`. Unaffected by this plan but must not regress.
- **Rule 23:** AM = client relay — this plan makes that rule machine-checkable via `row.provenance` (Task 6/7).
- **Rule 24:** every emitted row carries `latest_signal_anchor`; this plan adds `row.provenance` and `row.is_fresh` as pass-through computations alongside it — never a replacement.
- **Never guess a role.** Ambiguous or unresolved actor/sender classification → `unknown`/`ambiguous`, never a picked category. Matches house style already used for `project_type_unknown`, `activity_log_unmatched_entry`.
- **No fixed-hour recency constant.** The freshness guard always reuses the run's own already-computed dynamic lookback window (`collectors/gmail.md` § Window: 12–18h normal, up to 7 days extended, 72h forced Monday) — never a hardcoded cutoff (would break Monday/extended-lookback runs).
- **Fixed-cost lane untouched.** `origin: fixed_cost_mail` signals, the Client-Ask Ledger, `fc_state`, and Job 4c keep their exact existing behavior. This plan only removes the mail parser's scope *gate* for non-fixed-cost projects and adds a new, separate `origin: orbit_notification_mail` tag for the newly-unlocked case.
- **Display never changes.** The commenter/sender's name (`actor_name`/`sender.name`) always renders, unaffected by any of this. `actor_category`/`provenance` are internal fields that only select phrasing.
- Repo root for all paths: `/home/Krishna/Projects/PM Task Automation/pm-task-assignment/`.
- Match each file's existing voice: imperative second-person behavior instructions, `##`/`###` sections, tables for enums, explicit "does NOT do" lists.

---

### Task 1: Orbit collector — `actor_category` classification

**Files:**
- Modify: `collectors/orbit.md`

**Interfaces:**
- Consumes: `AM_user_ids` (already resolved by this same collector, § AM identity resolution), the workload/pod-matrix team roster (already resolved via `list_users`), and `client_contacts[]` per project (already in the relationship map, populated via `list_clients`/`get_client_details` in the mandatory sequence step 7).
- Produces: `actor_category: "am" | "team" | "client" | "unknown" | "ambiguous"` on every comment entry, every `activity_log_entry` signal, and the per-task `latest_signal` object. Consumed by Task 4 (`synthesis/matcher.md` Job 1.5).

- [ ] **Step 1: Read** `collectors/orbit.md` §§ "Priority Pass (Mode 1 sub-step 1d) → AM identity resolution", "Comment ordering — flatten the thread tree, then sort descending (newest first)", "Per-task `latest_signal` field — the deep-read anchor", "Output shape — per signal" in full.

- [ ] **Step 2: Insert new section** immediately after `## Comment ordering — flatten the thread tree, then sort descending (newest first)` and before `## Per-task \`latest_signal\` field — the deep-read anchor`. New section content:

````markdown
## Actor classification — `actor_category` on every activity/comment entry

Every comment and activity-log entry gets a role tag so downstream framing
(`synthesis/matcher.md` Job 1.5, `references/am-context.md`) never conflates who actually
posted it with a generic team comment. Runs once, after the MANDATORY tool call sequence
completes (`AM_user_ids` resolved in step 3, `client_contacts` resolved in step 7) and before
the sub-agent returns its output — same timing as the flatten-then-sort comment step above,
so both post-processing passes happen together, no new MCP calls.

Classification order (first match wins — never guess past a miss):

1. **`actor_id` ∈ `AM_user_ids`** (resolved once per run, § AM identity resolution) → `am`.
2. **`actor_id` matches the task's current assignee, any name in `references/pod-matrix.md`'s
   resolved roster, or the project's `recent_task_assignees[]`** → `team`.
3. **`actor_id`/`actor_name` matches an entry in the project's `client_contacts[]`**
   (relationship map, populated from `list_clients`/`get_client_details` in mandatory
   sequence step 7) by email first, name substring second → `client`.
4. **No match against any of the above** → `unknown`. Do not guess.
5. **Matches more than one of the above** (a rare identity collision — e.g. an id appearing
   in both `AM_user_ids` and the team roster) → `ambiguous`. Never pick one arbitrarily.

Add `actor_category: "am" | "team" | "client" | "unknown" | "ambiguous"` to:
- Every entry in the per-task `comments[]` list (§ Comment ordering) — computed from that
  entry's `user_id`.
- Every `activity_log_entry` signal (§ Output shape — per signal) — computed from that
  signal's `actor_id`.
- The per-task `latest_signal` object (§ Per-task `latest_signal` field) — same value as
  whichever comment/entry became that task's anchor.
````

- [ ] **Step 3: Update the per-task `latest_signal` shape.** Find this block under `## Per-task \`latest_signal\` field — the deep-read anchor`:

```
latest_signal: {
  source: "orbit_comment" | "orbit_task_body_update" | "orbit_status_change" | "orbit_due_date_change" | "orbit_due_today",
  id: <int — comment id, activity_log entry id, or task_id depending on source>,
  timestamp_iso: <ISO 8601>,
  author_id: <int>,
  author_name: <string>,
  excerpt: <string, ≤ 240 chars, plain text>
}
```

Replace with:

```
latest_signal: {
  source: "orbit_comment" | "orbit_task_body_update" | "orbit_status_change" | "orbit_due_date_change" | "orbit_due_today",
  id: <int — comment id, activity_log entry id, or task_id depending on source>,
  timestamp_iso: <ISO 8601>,
  author_id: <int>,
  author_name: <string>,
  actor_category: "am" | "team" | "client" | "unknown" | "ambiguous",
  excerpt: <string, ≤ 240 chars, plain text>
}
```

- [ ] **Step 4: Update the per-signal output shape.** Find this block under `## Output shape — per signal`:

```
{
  "source": "orbit",
  "signal_type": "activity_log_entry" | "overdue_task" | "new_task" | "status_change" | "new_comment" | "new_attachment" | "pm_task_due_today",
  "bypass_pm_action_filter": <bool — true only on pm_task_due_today signals (and priority-pass signals); absent/false otherwise>,
  "origin": <"fixed_cost_registry" — present only on signals sourced from a fixed-cost project's unfiltered activity log (§ Fixed-cost extension); absent on every ordinary user-scoped signal>,
  "project_id": <int>,
  "project_title": <string>,
  "project_url": <string>,
  "client_name": <string>,
  "sub_client_name": <string or null>,
  "account_manager": <string>,
  "project_owner": <string>,
  "task_id": <int or null>,
  "task_title": <string or null>,
  "task_url": <string or null>,
  "actor_name": <string>,
  "actor_id": <int>,
  "timestamp": <ISO datetime>,
```

Replace the `"actor_id": <int>,` line with:

```
  "actor_id": <int>,
  "actor_category": "am" | "team" | "client" | "unknown" | "ambiguous",
```

- [ ] **Step 5: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "## Actor classification" collectors/orbit.md                          # section exists
grep -c "actor_category" collectors/orbit.md                                    # >= 4 (new section + 3 shape sites)
grep -n "\"am\" | \"team\" | \"client\" | \"unknown\" | \"ambiguous\"" collectors/orbit.md | wc -l   # >= 1
```

- [ ] **Step 6: DO NOT COMMIT.**

---

### Task 2: Gmail collector — recipient classification

**Files:**
- Modify: `collectors/gmail.md`

**Interfaces:**
- Consumes: the Orbit relationship map (already an input to this collector, per § Sender classification), the full thread payload already pulled via `gmail_read_thread` (already includes To/CC headers — no new MCP call).
- Produces: `recipients: {to: [...], cc: [...]}` on every Gmail signal, each entry classified with the same category enum `sender.category` already uses. Consumed by Task 4 (`synthesis/matcher.md` Job 1.5).

- [ ] **Step 1: Read** `collectors/gmail.md` §§ "Sender classification", "Output shape — per email/thread" in full.

- [ ] **Step 2: Rename and extend the classification section.** Find:

```
## Sender classification

Use the Orbit relationship map (from the Orbit collector) to classify each sender:

| Category | How to identify |
|---|---|
| Client / Agency contact | Sender email domain matches an Orbit client's domain, OR sender matches an Orbit client contact |
| Account Manager | Sender matches Preferences' AM list |
| Internal team | `@whitelabeliq.com` and not an AM and not leadership |
| Leadership | Brian Gerstner, Nishant Rana, or anyone flagged leadership in Preferences |
| Self | Matches PM's canonical email or any alias — these are filtered out (not signal) |
| Unknown | Leave classification null; matcher will handle |
```

Replace with:

```
## Sender + recipient classification

Use the Orbit relationship map (from the Orbit collector) to classify each sender AND every
recipient (To + CC) on the thread — same table, same identity sources, applied uniformly:

| Category | How to identify |
|---|---|
| Client / Agency contact | Address's email domain matches an Orbit client's domain, OR the address matches an Orbit client contact |
| Account Manager | Address matches Preferences' AM list (canonical email or alias) |
| Internal team | `@whitelabeliq.com` and not an AM and not leadership |
| Leadership | Brian Gerstner, Nishant Rana, or anyone flagged leadership in Preferences |
| Self | Matches the running PM's canonical email or any alias |
| Unknown | No match against any of the above — leave as `unknown`, never guess |
| Ambiguous | Matches more than one of the above (rare identity collision) — leave as `ambiguous`, never pick one arbitrarily |

**Recipients require no new MCP call.** `gmail_read_thread` (already called for § Full thread
context) returns each message's To/CC headers as part of the thread payload already in
context — this step only reads data already fetched. Deduplicate To+CC addresses across all
messages in the thread before classifying (classify each unique address once).

The `sender` field on a Gmail signal is filtered out entirely when the sender is `self` (self
noise, per § Aliases — both directions) — this is unchanged. `recipients` entries with
category `self` are kept (the PM being a To/CC recipient is meaningful — e.g. "AM cc'd the
PM on a client email").
```

- [ ] **Step 3: Add `recipients` to the output shape.** Find this block under `## Output shape — per email/thread`:

```
{
  "source": "gmail",
  "signal_type": "client_awaiting_response" | "starred_unaddressed" | "attachment_received" | "thread_updated" | "unsent_draft" | "am_message" | "team_request" | "leadership_message",
  "thread_id": <string>,
  "message_id": <string — latest message in thread>,
  "sender": {
    "name": <string>,
    "email": <string>,
    "domain": <string>,
    "category": "client" | "am" | "team" | "leadership" | "unknown"
  },
  "subject": <string>,
```

Replace with:

```
{
  "source": "gmail",
  "signal_type": "client_awaiting_response" | "starred_unaddressed" | "attachment_received" | "thread_updated" | "unsent_draft" | "am_message" | "team_request" | "leadership_message",
  "thread_id": <string>,
  "message_id": <string — latest message in thread>,
  "sender": {
    "name": <string>,
    "email": <string>,
    "domain": <string>,
    "category": "client" | "am" | "team" | "leadership" | "self" | "unknown" | "ambiguous"
  },
  "recipients": {
    "to": [{"name": <string>, "email": <string>, "category": "client" | "am" | "team" | "leadership" | "self" | "unknown" | "ambiguous"}],
    "cc": [{"name": <string>, "email": <string>, "category": "client" | "am" | "team" | "leadership" | "self" | "unknown" | "ambiguous"}]
  },
  "subject": <string>,
```

- [ ] **Step 4: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "## Sender + recipient classification" collectors/gmail.md      # renamed section exists
grep -n "\"recipients\":" collectors/gmail.md                          # output shape updated
grep -c "ambiguous" collectors/gmail.md                                 # >= 2 (table + shape)
grep -n "no new MCP call" collectors/gmail.md                          # cost note present
```

- [ ] **Step 5: DO NOT COMMIT.**

---

### Task 3: Gmail collector — generalize the Orbit-notification-mail parser

**Files:**
- Modify: `collectors/gmail.md`

**Interfaces:**
- Consumes: Task 2's sender/recipient classification method (reused for the mail-parsed `actor` field), the Orbit relationship map (already an input, now also used to resolve non-tracked projects).
- Produces: `origin: orbit_notification_mail` signals (new, for non-fixed-cost projects) alongside the existing unchanged `origin: fixed_cost_mail` signals. Consumed by Task 4/5 (`synthesis/matcher.md` Job 1.5 provenance + Job 5 follower-only branch).

- [ ] **Step 1: Read** `collectors/gmail.md` § "Orbit-notification mails (fixed-cost lane)" in full.

- [ ] **Step 2: Replace the section.** Find the section starting `## Orbit-notification mails (fixed-cost lane)` and ending right before `## \`awaiting_action_hint\` — feeds the Possible-Orbit-miss trigger` (i.e. the whole section including its "Scope gate", "Parse per mail" table, signal-shape paragraph, dedup paragraph, and Pulse-rollup paragraph). Replace the ENTIRE section with:

````markdown
## Orbit-notification mails (fixed-cost lane + general lane)

Spec §11.2 (fixed-cost lane) + `docs/superpowers/specs/2026-07-08-provenance-classification-design.md` §4.2 (general lane). The main routine passes `registry_snapshot` into this
collector at dispatch (same object the Orbit collection sub-agent receives). Unlike the
original fixed-cost-only version, this section is now **always active** — it runs for every
Orbit notification mail in the lookback window regardless of `registry_snapshot` contents.

**Two-tier project resolution.** For each Orbit notification mail in the lookback window:

1. **Tracked-project match (fixed-cost lane, unchanged).** Match `#<project_number>` in
   subject/body first, `registry_snapshot.tracked_projects[]` title substring second.
   Match → tag `origin: fixed_cost_mail`. Everything about this tier — the Job 4c universe,
   the Pulse rollup, the shadow dedup — is **completely unchanged** by this generalization.
2. **General-project match (new lane).** No tracked-project match → try the same
   `#<project_number>` match, then project-title substring match, against the Orbit
   relationship map's project list (already an input to this collector, per § Sender +
   recipient classification). Match → tag `origin: orbit_notification_mail`.
3. **No match against either tier** → skip (NOT an error — another PM's project, or a
   project this PM has zero relationship to).
4. **Ambiguous match** (two titles match, no project number, in either tier) → skip + emit
   an incident naming the subject: `fc_mail_parse_failure` for a tracked-tier ambiguity
   (unchanged name), `orbit_mail_parse_failure` for a general-tier ambiguity (new name — kept
   distinct so the fixed-cost lane's existing incident stream is never touched). Never guess.

**Parse per mail** (subject + body HTML) — same table for both tiers:

| Field | Source |
|---|---|
| `project_id`, `project_number` | two-tier resolution above |
| `task_title` | task name in subject/body (Orbit templates embed it; null for project-level events) |
| `event_type` | one of `comment | status | assignment | due-date | attachment | task-created | task-completed` (map from the template's action phrase) |
| `actor` | the acting user named in the mail |
| `actor_category` | classify `actor` using the SAME method as § Sender + recipient classification (Task 2) — match against Preferences' AM list → `am`; against the resolved project's team roster → `team`; against the project's `client_contacts[]` → `client`; no match → `unknown`; multiple matches → `ambiguous`. Orbit notification mails are system-sent (e.g. `notifications@orbit...`) — the actor is NEVER the mail's `sender.category`; it must come from parsing this field. |
| `excerpt` | first ~200 chars of the event body (comment text, status old→new, etc.), HTML-stripped |
| `timestamp` | the mail's Date header (IST) |

A mail that cannot be parsed into at least `{project, event_type, actor, timestamp}` → the
same incident naming as step 4 above (`fc_mail_parse_failure` tracked-tier /
`orbit_mail_parse_failure` general-tier) + skip that mail; the run continues.

**Signal shape — tracked tier (unchanged).** Each parsed tracked-project mail becomes one
signal tagged `origin: fixed_cost_mail`, mirroring an Orbit activity signal, with
`latest_signal` built from the mail itself (`{source: "orbit-mail", actor, actor_category,
timestamp, excerpt}`) — non-negotiable rule #24 holds. These signals carry the resolved
`project_id`/`project_number` directly, so they do NOT go through Job 4b Pass 1b (already
resolved) — they enter the matcher's Job 4c universe as first-class fixed-cost signals
(`synthesis/matcher.md` § Job 4c), exactly as before.

**Signal shape — general tier (new).** Each parsed non-tracked mail becomes one signal tagged
`origin: orbit_notification_mail`, same `latest_signal` shape as above. These signals do
**NOT** enter Job 4c (that universe is fixed-cost-tagged origins only) — they flow through
the matcher's normal Job 1 → Job 1.5 → Job 2 pipeline exactly like any other Orbit-derived
signal. They are **NOT** exempt from Job 2's dedup (unlike `fixed_cost_mail` — see
`synthesis/matcher.md` § Job 2 Fixed-cost mail carve-out, which names only `fixed_cost_mail`).

**Dedup responsibility** for the tracked tier is unchanged — sits with the matcher (shadow
mode runs both feeds): see `synthesis/matcher.md` § Job 4c shadow dedup. For the general
tier, ordinary Job 2 dedup rules apply, including the existing "same project AND same
topic/deliverable (keyword overlap)" fallback for cases where this mail-derived signal has no
`task_id` to match on. This collector does not dedup against Orbit itself, in either tier.

**Pulse rollup output — tracked tier only, unchanged.** The collector ALSO emits
`mail_activity_summary[]`: one entry per **tracked** project that had ≥ 1 parsed notification
mail this run — `{project_id, project_number, title, counts by event_type, newest_excerpt,
newest_timestamp}`. General-tier (`orbit_notification_mail`) mails are NOT rolled up here —
no Pulse-equivalent exists for the general lane; this is an intentional scope decision (spec
§8), not an oversight.
````

- [ ] **Step 3: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "origin: orbit_notification_mail" collectors/gmail.md          # new tag present
grep -n "origin: fixed_cost_mail" collectors/gmail.md                  # unchanged tag still present
grep -n "orbit_mail_parse_failure" collectors/gmail.md                 # new incident name present
grep -n "fc_mail_parse_failure" collectors/gmail.md                    # old incident name still present (tracked tier)
grep -n "actor_category" collectors/gmail.md                           # parse table updated
grep -n "system-sent" collectors/gmail.md                              # sender-vs-actor nuance documented
```

- [ ] **Step 4: DO NOT COMMIT.**

---

### Task 4: Matcher — Job 1.5 provenance composition + Job 7 freshness guard

**Files:**
- Modify: `synthesis/matcher.md`

**Interfaces:**
- Consumes: `sender.category`/`recipients.{to,cc}[].category` (Task 2), `actor_category` (Task 1, Task 3), `origin: orbit_notification_mail`/`origin: fixed_cost_mail` (Task 3).
- Produces: `signal.provenance` (one of `am_relay | am_direct | client_direct | client_orbit_comment | am_orbit_comment | null`), `signal.ambiguous_actor_name` (string or absent), `row.provenance`, `row.is_fresh` (boolean). Consumed by Task 5 (Job 4/5/10 phrasing + routing) and Task 6 (`references/am-context.md`).

- [ ] **Step 1: Read** `synthesis/matcher.md` §§ "Job 1 — Group signals by project", "Job 2 — Deduplicate across sources", "Job 7 — Generate the proposed Orbit body" § "Latest-signal anchor (applies to EVERY row, including Flag rows)" in full.

- [ ] **Step 2: Insert new Job 1.5.** Find this exact text (the end of Job 1 followed by the Job 2 heading):

```
For priority-lane Orbit signals (`signal_type: am_handed_to_pm_overnight_due_today`): `project_id` and `parent_task_id` are already on the signal — no inference needed. Skip the rest of Job 1 for these and pass straight to Job 4b.

### Job 2 — Deduplicate across sources
```

Replace with:

````markdown
For priority-lane Orbit signals (`signal_type: am_handed_to_pm_overnight_due_today`): `project_id` and `parent_task_id` are already on the signal — no inference needed. Skip the rest of Job 1 for these and pass straight to Job 4b.

### Job 1.5 — Provenance classification

Runs immediately after Job 1 (grouping), before Job 2 (dedup), Job 4 (Summary), Job 4b
(context-link), and Job 10 (AI Notes) — all four need `provenance` to render correctly. Pure
lookup over categories the collectors already computed (`collectors/gmail.md` § Sender +
recipient classification, `collectors/orbit.md` § Actor classification) — no new identity
resolution happens here.

| Signal source | Condition | `signal.provenance` |
|---|---|---|
| Gmail | `sender.category == am` AND any of `recipients.to`/`recipients.cc` is `client` | `am_relay` |
| Gmail | `sender.category == am` AND no `client` in recipients | `am_direct` |
| Gmail | `sender.category == client` | `client_direct` |
| Gmail | `sender.category` is `team` / `leadership` / `self` / `unknown` | `null` |
| Orbit (comment/activity entries, incl. `origin: orbit_notification_mail` / `origin: fixed_cost_mail`) | `actor_category == client` | `client_orbit_comment` |
| Orbit (as above) | `actor_category == am` | `am_orbit_comment` |
| Orbit (as above) | `actor_category` is `team` / `unknown` | `null` |

**Ambiguous actor/sender.** When the collector-supplied category is `ambiguous` (matched more
than one role — see `collectors/orbit.md` § Actor classification, `collectors/gmail.md` §
Sender + recipient classification), set `signal.provenance = null` and
`signal.ambiguous_actor_name = <the actor/sender's name>`. Job 10 reads this field to emit
`Uncertain: actor <name> matched more than one role; provenance not set.` Never guess a role.

`provenance: null` is the common case — most signals are ordinary internal/team activity and
render exactly as they do today. The five non-null tags (`am_relay`, `am_direct`,
`client_direct`, `client_orbit_comment`, `am_orbit_comment`) only matter when set.

**Row-level provenance is finalized in Job 7**, once `row.latest_signal_anchor` is chosen —
see Job 7 § Latest-signal anchor below.

### Job 2 — Deduplicate across sources
````

- [ ] **Step 3: Add row-level provenance + freshness right after the anchor.** Find this exact text under `#### Latest-signal anchor (applies to EVERY row, including Flag rows)`:

````
```
row.latest_signal_anchor = {
  source: "orbit_comment" | "orbit_task_body_update" | "orbit_status_change" | "orbit_due_date_change" | "orbit_due_today" | "orbit-mail" | "gmail_message" | "fathom_meeting",
  id: <int or string>,
  timestamp_iso: <ISO 8601>,
  author_name: <string>,
  excerpt: <string ≤ 240 chars, plain text>,
  source_url: <string — link to the originating item>
}
```

If no input source carries a signal (rare — a row with no trigger), drop to `filtered_signals` with `filter_reason: no_trigger_signal_identifiable` rather than ship with a null anchor. The writer's render needs this field; a row without it cannot render its Triggered-by line. Applies to every verb — Flag, Reopen subtask, Hand off parent task, Create parent task — same contract, no exceptions.
````

Replace with the same text, plus this new content appended immediately after it (before `#### Path-specific body rules`):

````
```
row.latest_signal_anchor = {
  source: "orbit_comment" | "orbit_task_body_update" | "orbit_status_change" | "orbit_due_date_change" | "orbit_due_today" | "orbit-mail" | "gmail_message" | "fathom_meeting",
  id: <int or string>,
  timestamp_iso: <ISO 8601>,
  author_name: <string>,
  excerpt: <string ≤ 240 chars, plain text>,
  source_url: <string — link to the originating item>
}
```

If no input source carries a signal (rare — a row with no trigger), drop to `filtered_signals` with `filter_reason: no_trigger_signal_identifiable` rather than ship with a null anchor. The writer's render needs this field; a row without it cannot render its Triggered-by line. Applies to every verb — Flag, Reopen subtask, Hand off parent task, Create parent task — same contract, no exceptions.

**Row-level provenance and freshness (computed immediately after the anchor above, same
Job 7 step, no new MCP calls).**

- `row.provenance` = the `signal.provenance` value (Job 1.5) of whichever signal became
  `row.latest_signal_anchor` above — not a blend of every signal in the row, not a vote.
  `null` when the anchor signal's own provenance is `null` (ordinary internal/team signal).
- `row.is_fresh` = `true` only when `row.latest_signal_anchor.timestamp_iso` falls within
  this run's own lookback window (`[last_run_timestamp, now]` — the same dynamic window
  `collectors/gmail.md` § Window already computes per run: 12–18h normal, up to 7 days
  extended, 72h forced on Monday). `false` otherwise. **Never a fixed hour cutoff** — a
  hardcoded window would incorrectly reject valid same-run corroboration on Monday/extended
  runs.
- Both fields are pass-through computations over data already in context.

**Job 4b Pass 1 (context-link) is unaffected.** Its actor-match/topic-match corroboration
rules still reach further back than the lookback window on purpose — attaching genuinely old
backstory (`context_signals[]`) to a fresh row is intentional (Job 4b's own stated rationale:
"email threads about a task routinely span days before the Orbit task surfaces overnight").
`row.is_fresh` gates only the anchor/framing layer above — an old signal can still be
attached as context; it can never itself claim to be the fresh, triggering event unless it
actually is.
````

- [ ] **Step 4: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -c "^### Job 2 — Deduplicate across sources" synthesis/matcher.md   # exactly 1
grep -n "### Job 1.5 — Provenance classification" synthesis/matcher.md  # section exists, once
grep -n "row.provenance" synthesis/matcher.md                           # >= 2 (definition + usage)
grep -n "row.is_fresh" synthesis/matcher.md                             # >= 1
grep -n "ambiguous_actor_name" synthesis/matcher.md                     # field defined
```

- [ ] **Step 5: DO NOT COMMIT.**

---

### Task 5: Matcher — wire provenance/freshness into Job 2, Job 4, Job 5, Job 10

**Files:**
- Modify: `synthesis/matcher.md`

**Interfaces:**
- Consumes: `signal.provenance`, `signal.ambiguous_actor_name`, `row.provenance`, `row.is_fresh` (Task 4), `origin: orbit_notification_mail` (Task 3).
- Produces: phrasing/routing behavior only — no new fields.

- [ ] **Step 1: Read** `synthesis/matcher.md` §§ "Job 2 — Deduplicate across sources" § "Fixed-cost mail carve-out", "Job 4 — Generate the one-line summary" § "Summary patterns — topic-style", "Job 5 — Recommend the action", "Job 10 — Write AI Notes" in full.

- [ ] **Step 2: Clarify the Job 2 carve-out is unaffected.** Find this paragraph under `### Job 2 — Deduplicate across sources`:

```
**Fixed-cost mail carve-out.** Signals with `origin: fixed_cost_mail` are EXEMPT from Job 2 dedup — never merge them here, even when they match an Orbit signal on project/topic/actors/window. Their dedup is owned exclusively by Job 4c's shadow dedup (Orbit wins; the mail copy must stay countable for FC-0's audit and the `mail_signals` stat — a Job 2 merge would destroy that distinct mail-side event).
```

Append immediately after it (same paragraph block, new sentence):

```
`origin: orbit_notification_mail` signals (the generalized, non-fixed-cost mail lane —
`collectors/gmail.md` § Orbit-notification mails) are explicitly **NOT** part of this
carve-out — they dedup normally under this Job's existing rules, including the "same project
AND same topic/deliverable (keyword overlap)" fallback above for the common case where a
mail-derived signal has no `task_id` to match a same-task Orbit signal on.
```

- [ ] **Step 3: Add provenance-aware phrasing to Job 4.** Find the end of `#### Summary patterns — topic-style` (the bulleted "Rules" list ending with `**Max 120 chars total** (Notion title field stays scannable across the column).`) and insert immediately after it:

````markdown
#### Provenance-aware phrasing (reads `row.provenance` + `row.is_fresh`)

When `row.provenance` is non-null, the Summary/Task Brief/AI-Notes text for this row MUST
follow the matching phrasing rule in `references/am-context.md` § Provenance-tag framing —
never infer framing from `sender.category`/`actor_category` directly. Time-relative words
("overnight", "just now", "today", "handed to you overnight") may appear ONLY when
`row.is_fresh == true`; when `false`, use a plain date reference instead (e.g. "Client asked
about this on Jun 30" rather than "Client asked overnight"). `provenance: null` rows are
unaffected — render exactly as before this design.
````

- [ ] **Step 4: Add the follower-only Job 5 branch.** Find this numbered pair under `### Job 5 — Recommend the action`:

```
2.6. **Fixed-cost state candidates (from Job 4c).** Row candidates carrying `fc_row_type: follow_up_reminder | stale_work_ping` arrive pre-classified as `Flag` — Job 4c (`synthesis/fixed-cost-state.md`) already decided they're actionable-for-PM before handing them to Job 5. No delegation target at emission (Recommended Assignee `—`); a PM note can promote a stale-work ping to a dev exactly like a due-today Flag (see synthesis/fixed-cost-state.md § Output & downstream).
3. **Flag the PM-owned moves first.**
```

Replace with:

```
2.6. **Fixed-cost state candidates (from Job 4c).** Row candidates carrying `fc_row_type: follow_up_reminder | stale_work_ping` arrive pre-classified as `Flag` — Job 4c (`synthesis/fixed-cost-state.md`) already decided they're actionable-for-PM before handing them to Job 5. No delegation target at emission (Recommended Assignee `—`); a PM note can promote a stale-work ping to a dev exactly like a due-today Flag (see synthesis/fixed-cost-state.md § Output & downstream).
2.7. **Follower-only Orbit-mail branch.** If the signal carries `origin: orbit_notification_mail` (the generalized mail parser, `collectors/gmail.md` § Orbit-notification mails) AND the PM is NOT the assignee of the resolved task (or the mail is project-level with no `task_id`) — the PM is a follower here, not an owner — emit a `Flag` immediately. Never attempt `Create subtask` / `Reopen subtask` / `Hand off parent task` for this branch: there is no PM-owned parent to act under by definition, and a passive "someone commented on a task you follow" signal is never dev-shaped work regardless of `work_type`. `pm_next_step: "Review comment on a task you follow — no delegation assumed."` `recommended_assignee = "— (PM action)"`. Skip Job 6, Job 7's full deep-read, and Job 8 exactly like any other Flag row (this branch does not qualify for the `pm_task_due_today` light-deep-read exception). Does not apply to `origin: fixed_cost_mail` signals — those keep their existing Job 4c routing unchanged.
3. **Flag the PM-owned moves first.**
```

- [ ] **Step 5: Add the ambiguous-actor bullet to Job 10.** Find this block under `### Job 10 — Write AI Notes`:

```
Include only things worth the PM knowing:
- `Uncertain:` flags (always)
- Why a PM note was honored differently than the recommendation
- When a single signal was split into multiple items
- When the recommended assignee isn't obvious
- When a collector failed and this item has partial context
```

Replace with:

```
Include only things worth the PM knowing:
- `Uncertain:` flags (always)
- Why a PM note was honored differently than the recommendation
- When a single signal was split into multiple items
- When the recommended assignee isn't obvious
- When a collector failed and this item has partial context
- When `signal.ambiguous_actor_name` is set (Job 1.5) — emit `Uncertain: actor <name> matched more than one role; provenance not set.`
```

- [ ] **Step 6: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "NOT.* part of this" synthesis/matcher.md                      # Job 2 clarification present
grep -n "#### Provenance-aware phrasing" synthesis/matcher.md          # Job 4 subsection present
grep -n "2.7\. \*\*Follower-only Orbit-mail branch" synthesis/matcher.md  # Job 5 branch present
grep -n "ambiguous_actor_name" synthesis/matcher.md | wc -l            # >= 2 (Job 1.5 def + Job 10 usage)
```

- [ ] **Step 7: DO NOT COMMIT.**

---

### Task 6: `references/am-context.md` — Provenance-tag framing

**Files:**
- Modify: `references/am-context.md`

**Interfaces:**
- Consumes: `row.provenance` values (Task 4), the five-tag enum (`am_relay | am_direct | client_direct | client_orbit_comment | am_orbit_comment`).
- Produces: the phrasing table Task 5's Job 4 subsection points to.

- [ ] **Step 1: Read** `references/am-context.md` in full (it's short — one pass).

- [ ] **Step 2: Insert new section** immediately after `## When an AM IS the originator` and before `## Priority-lane phrasing (already correct, do not break)`. Insert:

````markdown
## Provenance-tag framing (machine-checkable)

`synthesis/matcher.md` Job 1.5 computes `row.provenance` — one of five tags — on every row
that involves an AM or a client. This turns the judgment calls above into an explicit,
checkable field instead of an inference from `sender.category`/`actor_category` alone.
Required phrasing per tag:

| `row.provenance` | Required phrasing | Forbidden phrasing |
|---|---|---|
| `am_relay` | The existing relay framing in the table above (e.g. "Client (via AM Ellen) flagged a bug") — now confirmed by the tag, not inferred from "sender happens to be AM." | Anything implying the AM originated the ask themselves. |
| `am_direct` | Per § "When an AM IS the originator" above — frame as the AM's own ask, no client involved. | Client-relay language when no client is on the thread. |
| `client_direct` | "Client (`<name>`) emailed you directly — no AM involved." | "AM put this on your plate" / "AM handoff" / any AM-relay language — there is no AM on this thread. |
| `client_orbit_comment` | "Client (`<name>`) commented directly on Orbit task #`<id>`." | Rendering the client's comment as generic team activity. |
| `am_orbit_comment` | Same relay-vs-originated judgment as `am_relay`/`am_direct` above, anchored on an Orbit comment instead of an email. | Treating an AM's own Orbit comment as if it were a routine dev/QA comment. |

**Freshness is separate from provenance.** Time-relative words ("overnight", "just now",
"today") require `row.is_fresh == true` (`synthesis/matcher.md` Job 7 § Latest-signal anchor)
regardless of which provenance tag applies — a correctly-tagged `client_direct` row from a
week-old thread still must not say "overnight." Use a plain date instead.

**Display is unaffected.** The name shown to the PM (`sender.name` / `actor_name`) is
unchanged by any of this — it always renders. `provenance` is an internal field that only
selects which phrasing template applies; when unresolved or ambiguous (`provenance: null`),
the row simply falls back to today's default (unmarked) phrasing.
````

- [ ] **Step 3: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "## Provenance-tag framing" references/am-context.md    # section exists
grep -c "client_direct\|client_orbit_comment\|am_orbit_comment\|am_relay\|am_direct" references/am-context.md   # >= 5
grep -n "Display is unaffected" references/am-context.md        # display-invariant note present
```

- [ ] **Step 4: DO NOT COMMIT.**

---

### Task 7: SKILL.md — rule amendment, new rule, file map

**Files:**
- Modify: `SKILL.md`

**Interfaces:**
- Consumes: everything produced by Tasks 1–6 (this task only documents/cross-references, no new behavior).

- [ ] **Step 1: Read** `SKILL.md` §§ "Non-negotiable rules" (specifically rule 23), "File map" in full.

- [ ] **Step 2: Amend rule 23.** Find:

```
22. **Project identifiers user-facing → use Orbit's `project_number`, never `id` or `url_slug`.** Orbit's `get_project_details` returns three project identifiers: `id` (internal numeric PK, e.g. `8598`), `url_slug` (URL-encoded id, e.g. `"51083298598"`), and `project_number` (the user-visible code shown in the Orbit UI, e.g. `"16915"`). Every user-facing rendering of a project — Notion `Project` column, Summary text, Sources block citations, Slack drafts, handoff drafts — uses `project_number` rendered as `#<project_number>` (with the leading hash). Internal `id` and `url_slug` are NEVER user-facing; they only appear inside URLs (where Orbit needs them) and inside the relationship map (where downstream tooling reads them). Format: `<Project Name> (#<project_number>)` for the Project column; `Process Barron Change Order #16915` for inline Summary mentions.

23. **AM-framing rule — AMs are client-relay, never teammates.** Every text the skill renders that mentions an AM (Summary, Task Brief, Recommended Action, Outcome handoff bodies, PM Next Step, AI Notes, AM Ping Drafts) frames the AM as the client-relay channel — never as an internal teammate, never as the developer-picker, never as the scope-approver, never as the delivery-owner. See `references/am-context.md` for the canonical role description and the right-vs-wrong framing table. The 1 PM ↔ 1 AM cardinality is locked: Preferences carries the single AM. Existing priority-lane phrasings (`<AM> put this on your plate overnight, due today`) already comply and stay verbatim.
```

Replace with:

```
22. **Project identifiers user-facing → use Orbit's `project_number`, never `id` or `url_slug`.** Orbit's `get_project_details` returns three project identifiers: `id` (internal numeric PK, e.g. `8598`), `url_slug` (URL-encoded id, e.g. `"51083298598"`), and `project_number` (the user-visible code shown in the Orbit UI, e.g. `"16915"`). Every user-facing rendering of a project — Notion `Project` column, Summary text, Sources block citations, Slack drafts, handoff drafts — uses `project_number` rendered as `#<project_number>` (with the leading hash). Internal `id` and `url_slug` are NEVER user-facing; they only appear inside URLs (where Orbit needs them) and inside the relationship map (where downstream tooling reads them). Format: `<Project Name> (#<project_number>)` for the Project column; `Process Barron Change Order #16915` for inline Summary mentions.

23. **AM-framing rule — AMs are client-relay, never teammates.** Every text the skill renders that mentions an AM (Summary, Task Brief, Recommended Action, Outcome handoff bodies, PM Next Step, AI Notes, AM Ping Drafts) frames the AM as the client-relay channel — never as an internal teammate, never as the developer-picker, never as the scope-approver, never as the delivery-owner. See `references/am-context.md` for the canonical role description and the right-vs-wrong framing table. The 1 PM ↔ 1 AM cardinality is locked: Preferences carries the single AM. Existing priority-lane phrasings (`<AM> put this on your plate overnight, due today`) already comply and stay verbatim. This framing is machine-checkable: `synthesis/matcher.md` Job 1.5 computes an explicit `row.provenance` tag (`am_relay | am_direct | client_direct | client_orbit_comment | am_orbit_comment`) so framing is never inferred from `sender.category`/`actor_category` alone — see `references/am-context.md` § Provenance-tag framing.

27. **Freshness claims require an in-window anchor.** Time-relative language ("overnight", "just now", "today", "handed to you overnight") may appear in Summary/Task Brief/AI Notes ONLY when the row's `latest_signal_anchor` timestamp falls within the run's own lookback window (`row.is_fresh == true` — `synthesis/matcher.md` Job 7 § Latest-signal anchor). A stale signal that becomes a row's only anchor (no genuinely fresh event exists) still ships — correct `provenance`, correct verb — just without freshness wording. Prevents a signal's age from being silently overstated. See `docs/superpowers/specs/2026-07-08-provenance-classification-design.md` for the anomaly this closes.
```

- [ ] **Step 3: Update File map entries.** Find:

```
- `collectors/orbit.md`, `gmail.md` — primary data collection (called by Mode 1 Step 3a/3b). `collectors/orbit.md` documents both the normal pass and the Mode 1 Step 3a priority pass (AM-handed-to-PM detection).
```

Replace with:

```
- `collectors/orbit.md`, `gmail.md` — primary data collection (called by Mode 1 Step 3a/3b). `collectors/orbit.md` documents both the normal pass and the Mode 1 Step 3a priority pass (AM-handed-to-PM detection). Both collectors also tag `actor_category`/recipient categories (`am`/`team`/`client`/`unknown`/`ambiguous`) consumed by the matcher's provenance classifier.
```

Find:

```
- `synthesis/matcher.md` — signal grouping, Job 4b Pass 1 (Gmail → Orbit context cross-link) + Pass 2 (Fathom enrichment fetch), and summary generation, alias-aware.
```

Replace with:

```
- `synthesis/matcher.md` — signal grouping, Job 1.5 (provenance classification) + Job 7's freshness guard (`row.is_fresh`), Job 4b Pass 1 (Gmail → Orbit context cross-link) + Pass 2 (Fathom enrichment fetch), and summary generation, alias-aware.
```

Find:

```
- `references/am-context.md` — Account Manager role description + sentence framing (right-vs-wrong examples). Read once per session so AM mentions in Summary / Task Brief / Recommended Action / handoff bodies frame the AM as client-relay, not as an internal teammate.
```

Replace with:

```
- `references/am-context.md` — Account Manager role description + sentence framing (right-vs-wrong examples). Read once per session so AM mentions in Summary / Task Brief / Recommended Action / handoff bodies frame the AM as client-relay, not as an internal teammate. § Provenance-tag framing maps the matcher's `row.provenance` tag to required/forbidden phrasing.
```

- [ ] **Step 4: Verify:**

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
grep -n "^27\. \*\*Freshness claims" SKILL.md                          # new rule present
grep -n "row.provenance" SKILL.md                                      # rule 23 amendment present
grep -n "Job 1.5 (provenance classification)" SKILL.md                 # file map updated
grep -n "Provenance-tag framing" SKILL.md                              # file map updated
```

- [ ] **Step 5: DO NOT COMMIT.**

---

### Task 8: Cross-file consistency verification (no edits expected)

**Files:**
- Read-only across all files touched above: `collectors/orbit.md`, `collectors/gmail.md`, `synthesis/matcher.md`, `references/am-context.md`, `SKILL.md`.

- [ ] **Step 1: Name consistency** — every grep must return a hit in EVERY listed file:

```bash
cd "/home/Krishna/Projects/PM Task Automation/pm-task-assignment"
for t in "actor_category" ; do echo "== $t"; grep -l "$t" collectors/orbit.md collectors/gmail.md synthesis/matcher.md; done
for t in "row.provenance" ; do echo "== $t"; grep -l "$t" synthesis/matcher.md references/am-context.md SKILL.md; done
for t in "row.is_fresh" ; do echo "== $t"; grep -l "$t" synthesis/matcher.md references/am-context.md SKILL.md; done
grep -l "orbit_notification_mail" collectors/gmail.md synthesis/matcher.md
grep -l "am_relay\|am_direct\|client_direct\|client_orbit_comment\|am_orbit_comment" synthesis/matcher.md references/am-context.md
grep -l "ambiguous" collectors/orbit.md collectors/gmail.md synthesis/matcher.md
```

- [ ] **Step 2: Ordering invariants** — confirm by reading: `synthesis/matcher.md` Job 1.5 sits between Job 1 and Job 2 exactly once (no duplicate `### Job 2` heading survived the Task 4 Step 2 insertion); Job 7's freshness-guard paragraph sits between the `latest_signal_anchor` block and `#### Path-specific body rules`; `collectors/orbit.md`'s new Actor classification section sits between Comment ordering and Per-task `latest_signal` field.

- [ ] **Step 3: Spec-coverage sweep** — open `docs/superpowers/specs/2026-07-08-provenance-classification-design.md` and confirm each numbered section has an implementing task:
  - §3 (five provenance classes) → Task 4 (Job 1.5 table), Task 6 (framing table)
  - §4.1 (Gmail recipients) → Task 2
  - §4.2 (Orbit actor classification + mail-parser generalization) → Task 1, Task 3
  - §5 (Job 1.5 composition) → Task 4
  - §5.1 (recency guard) → Task 4 (Job 7), Task 5 (Job 4 phrasing gate)
  - §6 (framing rule updates) → Task 5 (Job 4 subsection), Task 6 (am-context.md), Task 7 (rule 23 amendment)
  - §7 (error handling) — actor unresolved/ambiguous → Task 1/2/3 (collector classification) + Task 4/5 (Job 1.5 + Job 10 Uncertain note); mail-parser failure → Task 3; dedup risk → Task 5 (Job 2 clarification); follower-only routing → Task 5 (Job 5 branch 2.7); volume risk → intentionally no task (flagged-not-solved, per spec)
  - §8 (out-of-scope) — verified as NOT implemented anywhere above: no `cc_am_aware` field, no separate Sources-section marker, no throttling logic, fixed-cost lane untouched.

- [ ] **Step 4: Report** — list every modified file with a one-line summary for Krishna's review. **DO NOT COMMIT — final state is an uncommitted working tree.**

---

## Self-review notes (done at plan time)

- **Spec coverage:** every numbered spec section maps to a task per Task 8 Step 3 above; no gaps found.
- **Type/name consistency:** `actor_category`, `signal.provenance`, `row.provenance`, `row.is_fresh`, `signal.ambiguous_actor_name`, `origin: orbit_notification_mail` — same names used identically across Tasks 1–7; Task 8 enforces with grep.
- **No placeholders:** every insert step carries full markdown content or an exact, bounded find/replace anchored to text quoted verbatim from the current files (read during plan authoring on 2026-07-08).
- **Task sizing:** 8 tasks, one clear deliverable each; Tasks 2/3 share a file but are independently reviewable (recipient classification vs. mail-parser generalization); Task 4/5 split the same way (produce vs. consume the new fields in `synthesis/matcher.md`).
