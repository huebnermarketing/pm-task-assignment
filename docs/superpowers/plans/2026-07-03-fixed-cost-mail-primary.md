# Fixed-Cost Mail-Primary Activity Lane (v4) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the fixed-cost lane's 20–30 daily unfiltered `get_activity_log` calls with parsed Orbit notification emails, behind an audited shadow → mail-primary state machine.

**Architecture:** This repo is a prompt-programmed Claude skill — the markdown files ARE the executable program, so every task is a precise markdown edit and "tests" are grep/consistency assertions. Spec: `docs/superpowers/specs/2026-07-02-fixed-cost-project-tracking-design.md` **§11** (v4). Discovery (§4.2), the daily roster loop, FC-3, and FC-4 are UNCHANGED; only the fixed-cost event *source* changes.

**Tech Stack:** Markdown prompt files; Orbit MCP; Gmail MCP; Notion writer.

## Global Constraints

- **DO NOT COMMIT. DO NOT `git add`.** Krishna reviews and commits manually. The working tree already holds his uncommitted v3 review set — leave everything uncommitted.
- Locked names — copy verbatim, never paraphrase:
  - Registry header fields: `Activity source: <shadow | mail-primary>` and `Clean audits: <n>`
  - Signal origin tag: `origin: fixed_cost_mail`
  - Incident keys: `fc_mail_parse_failure`, `mail_coverage_gap`
  - Audit step name: `FC-0 — Friday coverage audit` (runs BEFORE FC-1, dual-feed runs only: shadow Fridays + mail-primary first-Friday re-audit)
  - Run-log stat keys: `activity_source`, `clean_audits`, `mail_signals`, `coverage_gaps`
- Cutover threshold: **2** consecutive clean Friday audits. Re-audit: **first Friday of each month** (IST calendar). Any gap → revert to `shadow` + `Clean audits: 0`.
- Rule 24 (latest_signal anchor) and rule 20 (actionable-only rows) hold for mail-derived signals.
- Rule 22 rendering (`<Title> (#<project_number>)`) holds everywhere; never internal `id`.
- The Gmail collector remains Gmail-MCP-only (its Orbit data arrives via `registry_snapshot` and the relationship map passed in — zero Orbit/Notion calls from inside it).
- Existing v3 behavior for FC-3, FC-4, roster loop, discovery + guard: byte-identical.

## File Structure

| File | Responsibility in this plan |
|---|---|
| `schemas/fixed-cost-registry.md` | Header callout gains 2 machine fields + semantics (T1) |
| `collectors/gmail.md` | Exclusion lift + scope gate + parse spec + signal shape (T2) |
| `collectors/orbit.md` | Activity loop conditional on `activity_source`; shadow audit export (T3) |
| `synthesis/matcher.md` | `origin: fixed_cost_mail` in Job 4c universe + shadow dedup (T4) |
| `synthesis/fixed-cost-state.md` | NEW FC-0 audit step; source-agnostic FC-1/FC-2; fc_output additions (T5) |
| `writers/notion.md` | Registry refresh maintains the 2 fields; flip/revert logic (T6) |
| `writers/run-log.md`, `schemas/run-log-detail-page.md` | 4 new stat keys + stats line (T7) |
| `modes/mode-1-morning-collection.md`, `SKILL.md`, `ROUTINE-ENTRYPOINTS.md` | Orchestration: snapshot to Gmail collector, Gmail-failure fallback, baked prompts (T8) |
| (whole tree) | Final consistency sweep (T9) |

---

### Task 1: Registry schema — Activity source + Clean audits header fields

**Files:**
- Modify: `schemas/fixed-cost-registry.md` (§1 Header callout, ~lines 17–40)

**Interfaces:**
- Produces: header lines `Activity source: <shadow | mail-primary>` and `Clean audits: <n>` that T3 (collector reads via `registry_snapshot`), T5 (FC-0 mutates via `fc_state_patch`), and T6 (writer renders) all reference by these exact names.

- [ ] **Step 1: Extend the header callout block.** In §1, replace the 4-line callout code block with a 6-line one:

```
Fixed-cost tracking: <N> active projects
Last refresh: <YYYY-MM-DD HH:MM IST> · Discovery: <filtered | fallback-sweep>
Last successful discovery: <YYYY-MM-DD HH:MM IST>
Activity source: <shadow | mail-primary> · Clean audits: <n>
Managed by PM Task Assignment — edit only the Manual pin rows and the Ask column.
```

(That is 5 lines of content; update the prose "Four lines:" to "Five lines:".)

- [ ] **Step 2: Add semantics prose** immediately after the existing `Last successful discovery` explanation paragraph:

```markdown
`Activity source` selects the fixed-cost lane's event feed (spec §11.3). `shadow` (the
initial value, and the value assumed when the line is missing on a pre-v4 page): BOTH the
unfiltered per-project `get_activity_log` loop AND Orbit-notification-mail parsing run;
Orbit is authoritative. `mail-primary`: the unfiltered `get_activity_log` loop is OFF; the
mail-derived signals are the fixed-cost event feed (roster loop and workload lane
unaffected). `Clean audits` counts consecutive gap-free `FC-0 — Friday coverage audit`
passes; it reaches **2** → the writer flips `Activity source: mail-primary` at registry
refresh. Any `mail_coverage_gap` → reset to 0 (and revert to `shadow` if in mail-primary).
Both fields are machine-maintained by the writer (`writers/notion.md` § Flow — refreshing
the Fixed-Cost Registry); the PM never edits them.
```

- [ ] **Step 3: Verify.**

Run: `grep -n "Activity source\|Clean audits" schemas/fixed-cost-registry.md`
Expected: callout line + semantics paragraph, both using the locked spellings.

---

### Task 2: Gmail collector — Orbit-notification parsing for tracked fixed-cost projects

**Files:**
- Modify: `collectors/gmail.md` (§ What to skip ~line 50; § What to collect; new section)

**Interfaces:**
- Consumes: `registry_snapshot` (passed in at dispatch — T8 wires this; shape per `collectors/orbit.md` ~line 59: `tracked_projects[]` with `project_id`, `project_number`, `title`).
- Produces: signals tagged `origin: fixed_cost_mail` with shape `{project_id, project_number, task_title, event_type, actor, excerpt, timestamp, latest_signal}` — consumed by T4 (matcher Job 4c universe) and T5 (FC-0/FC-1/FC-2).

- [ ] **Step 1: Amend the exclusion bullet** (§ What to skip, "Orbit notification emails"). Replace the bullet with:

```markdown
- **Orbit notification emails** — skipped for the workload lane (already covered by the
  Orbit collector), EXCEPT mails that resolve to a project in the run's
  `registry_snapshot.tracked_projects[]` (fixed-cost lane, spec §11.2) — those are collected
  and parsed per § Orbit-notification mails (fixed-cost lane) below. Detect via sender
  domain matching Orbit's notification-from address; all non-tracked Orbit notification
  mails stay excluded exactly as before.
```

- [ ] **Step 2: Add collection query.** In § What to collect, add item 6:

```markdown
6. **Orbit notification mails for tracked fixed-cost projects** — `from:<Orbit
   notification-from address> after:<lookback>`. Collected ONLY when the mail resolves to a
   `registry_snapshot.tracked_projects[]` entry (see § Orbit-notification mails
   (fixed-cost lane)). Skipped entirely when `registry_snapshot` is absent/empty.
```

- [ ] **Step 3: Add the new section** `## Orbit-notification mails (fixed-cost lane)` after § Context-link metadata:

```markdown
## Orbit-notification mails (fixed-cost lane)

Spec §11.2. The main routine passes `registry_snapshot` into this collector at dispatch
(same object the Orbit collection sub-agent receives). This section activates ONLY when
`registry_snapshot.tracked_projects[]` is non-empty.

**Scope gate.** For each Orbit notification mail in the lookback window, resolve the
project: match `#<project_number>` in subject/body first, tracked-project title substring
second. Resolves to a tracked project → collect. No match → skip (NOT an error — it is
another PM's project or an untracked type; the workload-lane exclusion still applies).
Ambiguous match (two tracked titles match, no number) → skip + emit
`fc_mail_parse_failure` incident naming the subject. Never guess.

**Parse per mail** (subject + body HTML):

| Field | Source |
|---|---|
| `project_id`, `project_number` | scope-gate resolution against `registry_snapshot` |
| `task_title` | task name in subject/body (Orbit templates embed it; null for project-level events) |
| `event_type` | one of `comment | status | assignment | due-date | attachment | task-created | task-completed` (map from the template's action phrase) |
| `actor` | the acting user named in the mail |
| `excerpt` | first ~200 chars of the event body (comment text, status old→new, etc.), HTML-stripped |
| `timestamp` | the mail's Date header (IST) |

A mail that cannot be parsed into at least `{project, event_type, actor, timestamp}` →
`fc_mail_parse_failure` incident (subject included) + skip that mail; the run continues.

**Signal shape.** Each parsed mail becomes one signal tagged `origin: fixed_cost_mail`,
mirroring an Orbit activity signal, with `latest_signal` built from the mail itself
(`{source: "orbit-mail", actor, timestamp, excerpt}`) — non-negotiable rule #24 holds.
These signals carry the resolved `project_id`/`project_number` directly, so they do NOT go
through Job 4b Pass 1b (already resolved) — they enter the matcher's Job 4c universe as
first-class fixed-cost signals (`synthesis/matcher.md` § Job 4c).

**Dedup responsibility** sits with the matcher (shadow mode runs both feeds): see
`synthesis/matcher.md` § Job 4c shadow dedup. This collector does not dedup against Orbit.
```

- [ ] **Step 4: Verify.**

Run: `grep -n "fixed_cost_mail\|fc_mail_parse_failure\|registry_snapshot" collectors/gmail.md`
Expected: exclusion bullet, collect item 6, and the new section all present; banner at line 3 untouched (Gmail-MCP-only rule intact — no Orbit/Notion calls added).

---

### Task 3: Orbit collector — activity loop conditional on `activity_source`

**Files:**
- Modify: `collectors/orbit.md` (§ Fixed-cost extension: `registry_snapshot` input shape ~line 59; merged activity loop ~lines 104–126; sub-agent return list ~line 32)

**Interfaces:**
- Consumes: `registry_snapshot.activity_source` and `.clean_audits` (new fields, read from the T1 header by the main routine — T8 wires the read).
- Produces: in shadow mode, the existing fixed-cost activity signals PLUS (unchanged) roster; in mail-primary mode, roster only. Return object gains `fixed_cost_activity_loop_ran: true | false` (T8's Mode 1 assertions and T5's FC-0 read it).

- [ ] **Step 1: Extend the `registry_snapshot` input shape** (~line 59) to include the two new fields:

```
`registry_snapshot` invocation input: {tracked_projects[] (incl. manual pins),
last_successful_discovery, activity_source (shadow | mail-primary; missing → shadow),
clean_audits (int; missing → 0), client_ask_ledger[], fc_state}
```

(Adjust to the file's existing formatting of that block; add the two fields + defaults.)

- [ ] **Step 2: Make the fixed-cost half of the merged loop conditional.** In the merged activity loop (the bullet "Fixed-cost projects (whether or not also in workload): `get_activity_log(...)` — no `user_id` param"), prepend:

```markdown
- **Activity-source gate (spec §11.3).** The unfiltered fixed-cost `get_activity_log`
  calls below run ONLY when `registry_snapshot.activity_source == "shadow"`, OR when it is
  `"mail-primary"` AND today (IST) is the first Friday of the month (monthly re-audit), OR
  when the main routine dispatched this sub-agent with `gmail_collector_failed: true`
  (one-run fallback — Gmail down means no mail feed this run; never run blind). In
  mail-primary mode outside those cases, SKIP the unfiltered fixed-cost `get_activity_log`
  calls entirely — the mail-derived signals (`origin: fixed_cost_mail`, from the Gmail
  collector) are the fixed-cost event feed. The `get_project_task_list` roster call per
  tracked project and the workload-project `get_activity_log` calls (`user_id=PM`) run
  UNCHANGED in every mode. Overlap rule in mail-primary mode: a fixed-cost project also in
  the workload gets its normal `user_id=PM` call only (the all-actor view arrives by mail).
```

- [ ] **Step 3: Add the return flag.** In the sub-agent return list (~line 32 area), add:

```markdown
- `fixed_cost_activity_loop_ran` (boolean) — true when the unfiltered fixed-cost
  `get_activity_log` loop executed this run (shadow mode, monthly re-audit, or
  Gmail-failure fallback); the FC-0 audit and the Mode 1 narrative read this instead of
  re-deriving the calendar/mode logic.
```

- [ ] **Step 4: Verify.**

Run: `grep -n "activity_source\|fixed_cost_activity_loop_ran\|first Friday" collectors/orbit.md`
Expected: input-shape fields, the gate bullet, the return flag. Roster bullet and discovery section show no diff (`git diff collectors/orbit.md` — changes confined to the three spots above).

---

### Task 4: Matcher — mail signals in Job 4c universe + shadow dedup

**Files:**
- Modify: `synthesis/matcher.md` (§ Job 4c Input universe ~lines 245–258; § Pass 1b note ~line 210)

**Interfaces:**
- Consumes: `origin: fixed_cost_mail` signals (T2 shape).
- Produces: deduplicated Job 4c input universe consumed by T5's FC-0..FC-4.

- [ ] **Step 1: Widen the Job 4c input universe.** In § Job 4c "Input universe (widened — fixes Gmail blindness)", extend the universe sentence to add a fourth member:

```markdown
PLUS any signal tagged `origin: fixed_cost_mail` (Orbit-notification mails parsed by the
Gmail collector for tracked projects, spec §11.2 — these arrive pre-resolved with
`project_id`/`project_number` and do NOT pass through Pass 1b)
```

- [ ] **Step 2: Add the shadow dedup rule** immediately after the universe definition:

```markdown
**Shadow dedup (spec §11.3).** When the run carries BOTH feeds — `activity_source: shadow`
or the monthly re-audit, where the collector returns `fixed_cost_activity_loop_ran: true`
alongside `origin: fixed_cost_mail` signals — every `fixed_cost_mail`
signal that matches an `origin: fixed_cost_registry` signal on (project_id, task, event
class, timestamp within the same lookback window) is a duplicate: **the Orbit copy wins**,
the mail copy is dropped from row consideration (it still counts in FC-0's mail event set
and in the `mail_signals` stat). (A Gmail-failure fallback run carries no mail feed at all —
Gmail didn't run — so this dedup is simply vacuous there; only the Orbit copy exists.) Mail
signals with no Orbit counterpart survive — in shadow
mode that asymmetry is exactly what FC-0 audits (an ORBIT event with no MAIL counterpart is
a coverage gap; the reverse is fine, mail can be faster than the lookback bound).
```

- [ ] **Step 3: Verify.**

Run: `grep -n "fixed_cost_mail" synthesis/matcher.md`
Expected: universe extension + dedup rule in Job 4c; Pass 1b section unchanged (it keeps handling *client/AM* Gmail signals; notification mails bypass it).

---

### Task 5: State tracker — FC-0 Friday coverage audit + source-agnostic wording

**Files:**
- Modify: `synthesis/fixed-cost-state.md` (new section before `## FC-1` ~line 32; FC-1/FC-2 input wording; `## Output & downstream` ~line 152)

**Interfaces:**
- Consumes: both signal sets from Job 4c input (T4), `registry_snapshot.activity_source`/`.clean_audits`, collector flag `fixed_cost_activity_loop_ran` (T3).
- Produces: `fc_output.fc_state_patch` gains `{activity_source_next, clean_audits_next, coverage_gaps[]}` — T6's writer applies them to the registry header.

- [ ] **Step 1: Insert the FC-0 section** immediately before `## FC-1 — Resolution scan`:

```markdown
## FC-0 — Friday coverage audit (dual-feed runs only)

Spec §11.3. Runs BEFORE FC-1, only when ALL of: today (IST) is a Friday; the collector
returned `fixed_cost_activity_loop_ran: true`; the run collected mail-derived signals
(`origin: fixed_cost_mail` universe member) — i.e. both feeds are present side by side.
This covers every shadow-mode Friday AND the mail-primary monthly re-audit (first Friday —
the only mail-primary day the loop runs). On non-Friday shadow runs, FC-0 is a no-op (both
feeds still run; only the compare is weekly). A Gmail-failure fallback run has no mail
feed, so FC-0 skips.

**Compare.** Per tracked project, over this run's lookback window, build two event sets
from the Job 4c input universe: ORBIT = `origin: fixed_cost_registry` events; MAIL =
`origin: fixed_cost_mail` events (including mail signals the shadow dedup dropped as
duplicates — dedup removes them from row consideration, not from this audit). For every
ORBIT event of an actionable class (comment, status change, assignment, due-date move,
task completion, task creation, attachment) with NO MAIL counterpart (same project, same task, same event
class, timestamp within the window): record a coverage gap `{project_number, task_title,
event_type, orbit_timestamp}` and emit a `mail_coverage_gap` incident line naming all
four. MAIL events with no ORBIT counterpart are NOT gaps (mail can precede the
lookback-bounded log view).

**Verdict → state.**
- Zero gaps → `clean_audits_next = registry_snapshot.clean_audits + 1`. If that reaches 2
  and `activity_source == shadow` → `activity_source_next = "mail-primary"` (cutover; the
  writer logs the flip to the Run Log).
- Any gap → `clean_audits_next = 0`; if `activity_source == mail-primary` (monthly
  re-audit found drift) → `activity_source_next = "shadow"` (auto-revert; incident already
  emitted per gap).
- FC-0 did not run (not Friday / single feed) → both fields pass through unchanged.

Results travel in `fc_output.fc_state_patch` (§ Output & downstream) — FC-0 performs NO
Notion writes itself, same as FC-1..FC-4.
```

- [ ] **Step 2: Make FC-1/FC-2 input wording source-agnostic.** In FC-1 and FC-2, wherever the input events are described as coming from the activity sweep/log, widen to: "the Job 4c input universe (Orbit activity signals and/or `origin: fixed_cost_mail` signals — whichever feeds this run carries per `activity_source`)". Keep every gate, ordering rule, and format untouched.

- [ ] **Step 3: Extend `fc_output`.** In `## Output & downstream`, extend the `fc_state_patch` description with the three new keys:

```markdown
`fc_state_patch` additionally carries `activity_source_next` (`"shadow"` |
`"mail-primary"` | absent = unchanged), `clean_audits_next` (int | absent = unchanged),
and `coverage_gaps[]` (the FC-0 gap records, for the run-log `coverage_gaps` stat) —
applied by the writer to the registry header (`schemas/fixed-cost-registry.md` §1) at the
end-of-Mode-1 registry refresh.
```

- [ ] **Step 4: Verify.**

Run: `grep -n "FC-0\|activity_source_next\|clean_audits_next\|coverage_gap" synthesis/fixed-cost-state.md`
Expected: FC-0 section before FC-1; fc_output extension; FC-3/FC-4 sections show zero diff.

---

### Task 6: Notion writer — registry refresh maintains the two fields

**Files:**
- Modify: `writers/notion.md` (§ Flow — refreshing the Fixed-Cost Registry, ~line 254)

**Interfaces:**
- Consumes: `fc_output.fc_state_patch.{activity_source_next, clean_audits_next}` (T5).
- Produces: rendered header line `Activity source: … · Clean audits: …` (T1 format); Run Log line on flip/revert.

- [ ] **Step 1: Add header-maintenance bullets** to the registry-refresh flow, alongside the existing `Last refresh`/`Last successful discovery` bullets:

```markdown
- **Activity source / Clean audits (spec §11.3).** Render the header line `Activity
  source: <value> · Clean audits: <n>` from `fc_output.fc_state_patch`: apply
  `activity_source_next` / `clean_audits_next` when present, otherwise carry the current
  values forward. When the applied patch CHANGES `Activity source` (shadow → mail-primary
  cutover, or mail-primary → shadow revert), also write one Run Log line naming the
  direction and the trigger (`clean_audits reached 2` | `mail_coverage_gap on re-audit`).
  A pre-v4 page missing the line entirely → write it as `Activity source: shadow · Clean
  audits: 0` this refresh (schema default).
```

- [ ] **Step 2: Verify.**

Run: `grep -n "Activity source" writers/notion.md`
Expected: the new bullet inside § Flow — refreshing the Fixed-Cost Registry; rendering matches T1's callout line exactly (` · ` separator).

---

### Task 7: Run-log — four new fixed-cost stat keys

**Files:**
- Modify: `writers/run-log.md` (`fixed_cost_stats` schema ~line 30; stats line ~line 89)
- Modify: `schemas/run-log-detail-page.md` (Fixed-cost lane stats section)

**Interfaces:**
- Consumes: `fc_output.fc_state_patch.coverage_gaps[]` (T5), mail signal count (T2/T4), `fixed_cost_activity_loop_ran` (T3).

- [ ] **Step 1: Extend the `fixed_cost_stats` schema** in `writers/run-log.md` with four keys (inside the existing object):

```markdown
activity_source: "shadow" | "mail-primary",     // value AFTER this run's patch
clean_audits: <int>,                            // value AFTER this run's patch
mail_signals: <int>,                            // origin: fixed_cost_mail signals collected
coverage_gaps: <int>,                           // FC-0 gaps this run (0 when FC-0 didn't run)
```

- [ ] **Step 2: Extend the one-line summary** (item 3, ~line 89). Append to the existing format string:

```
, source: <shadow|mail-primary>, audits: <clean_audits>, mail signals: <M>, gaps: <G>
```

Add after the existing decision-trace mapping sentence: `Each FC-0 coverage gap maps into
its own decision-trace line, formatted [subject: task title (#project_number)] → [action:
"coverage gap"] → [reason: <event_type> seen in activity log, no mail counterpart].`

- [ ] **Step 3: Mirror in `schemas/run-log-detail-page.md`** — extend the Fixed-cost lane stats section's example line with the same four appended fields.

- [ ] **Step 4: Verify.**

Run: `grep -n "mail_signals\|coverage_gaps\|activity_source" writers/run-log.md schemas/run-log-detail-page.md`
Expected: schema keys + summary line + detail-page mirror, spellings matching the Global Constraints exactly.

---

### Task 8: Orchestration — Mode 1, SKILL.md, ROUTINE-ENTRYPOINTS

**Files:**
- Modify: `modes/mode-1-morning-collection.md` (`registry_snapshot` pass-through ¶ ~line 79; step 2a; post-collection narrative; stats line)
- Modify: `SKILL.md` (data-source sentence ~line 86 area; fixed-cost lane summary ~line 70)
- Modify: `ROUTINE-ENTRYPOINTS.md` (Routine 1 baked prompt fixed-cost sentences)

**Interfaces:**
- Consumes: everything above.
- Produces: dispatch wiring (snapshot → Gmail collector; `gmail_collector_failed` → supplemental Orbit fallback).

- [ ] **Step 1: Widen the `registry_snapshot` pass-through** (mode-1 ~line 79). Extend the paragraph: the snapshot read now also parses `Activity source` and `Clean audits` from the header (missing → `shadow` / `0`), and the same `registry_snapshot` object is passed into BOTH the 1e Orbit collection sub-agent AND the 2a Gmail collector (which uses only `tracked_projects[]` for its scope gate — it stays Gmail-MCP-only).

- [ ] **Step 2: Add the Gmail-failure fallback** to the post-collection narrative:

```markdown
**Gmail-collector failure in mail-primary mode (spec §11.3).** If `Activity source:
mail-primary` and the 2a Gmail collector hard-fails (after its existing retries), the
fixed-cost lane must not run blind: dispatch ONE supplemental Orbit sub-agent scoped to
the fixed-cost activity loop only (unfiltered `get_activity_log` per tracked project,
shadow-mode shape; no discovery, no workload, no roster — those already ran in 1e), log
the incident, and continue. The mode is NOT flipped by a one-off failure. In shadow mode a
Gmail failure needs no fallback (the Orbit loop already ran); FC-0 simply skips (single
feed).
```

- [ ] **Step 3: Update the 1e/2a narrative + stats.** In the fixed-cost discovery/activity sentences (~line 82), note the activity-source gate ("the unfiltered per-project `get_activity_log` half runs per the `activity_source` gate — `collectors/orbit.md` § Fixed-cost extension") and append the four new stat keys to the mode's `fixed_cost_stats` mention.

- [ ] **Step 4: SKILL.md.** (a) In the fixed-cost lane summary (~line 70), append: "Event feed per the registry's `Activity source` state machine — Orbit activity log in `shadow`, parsed Orbit-notification mails in `mail-primary`, audited weekly via FC-0 (spec §11)." (b) In the mandatory-sequence paragraph (~line 86), amend the final fixed-cost sentence: the unfiltered `get_activity_log` per tracked project is **conditional on `activity_source`** (shadow / monthly re-audit / Gmail-failure fallback only); roster stays unconditional. (c) Data-sources sentence (~line 12 block): Gmail's role line gains "+ Orbit-notification mails for tracked fixed-cost projects (spec §11.2)".

- [ ] **Step 5: ROUTINE-ENTRYPOINTS.md.** In the Routine 1 baked prompt's fixed-cost sentences, mirror the conditional: replace the unconditional "one unfiltered activity call per tracked fixed-cost project" phrasing with "fixed-cost event feed per the registry's `Activity source` (shadow: unfiltered activity calls; mail-primary: parsed Orbit-notification mails), plus the roster call per tracked project". Keep the Validation Step registry mention unchanged.

- [ ] **Step 6: Verify.**

Run: `grep -n "activity_source\|Activity source\|gmail_collector_failed" modes/mode-1-morning-collection.md SKILL.md ROUTINE-ENTRYPOINTS.md`
Expected: all three files reference the gate; baked prompt no longer asserts the unconditional unfiltered loop.

---

### Task 9: Whole-tree consistency sweep

**Files:** read-only over the repo; fix any stragglers found.

- [ ] **Step 1: Locked-name sweep.**

Run: `grep -rn "mail-primary\|fixed_cost_mail\|Clean audits\|FC-0\|mail_coverage_gap\|fc_mail_parse_failure" --include="*.md" . | grep -v ".superpowers\|docs/superpowers"`
Expected: hits ONLY in the 10 files from §11.8 (T1–T8 targets). Any variant spelling (`mail primary`, `fixed-cost-mail`, `clean_audit `) = bug; fix to the locked form.

- [ ] **Step 2: Stale-assertion sweep.** Confirm no file still asserts the unfiltered fixed-cost activity loop unconditionally:

Run: `grep -rn "unfiltered" --include="*.md" collectors modes writers synthesis schemas SKILL.md ROUTINE-ENTRYPOINTS.md README.md ENVIRONMENTS.md | grep -i "fixed"`
Expected: every hit is either inside the shadow-mode branch text or explicitly conditioned on the activity-source gate.

- [ ] **Step 3: Cross-file contract check.** Verify by reading (no edits unless mismatched): T2's signal shape fields == T4's dedup keys == T5's FC-0 compare keys; T5's `fc_state_patch` keys == T6's writer bullet == T7's stat keys; T3's `fixed_cost_activity_loop_ran` spelling identical in orbit.md, fixed-cost-state.md, mode-1.

- [ ] **Step 4: Spec sync.** Re-read spec §11 against the implemented text; any drift found in Steps 1–3 that required deviating from §11 → update the spec section to match reality (spec follows code, per this repo's convention).

- [ ] **Step 5: Report.** Summarize files touched + diffstat (`git diff --stat`). DO NOT commit.
