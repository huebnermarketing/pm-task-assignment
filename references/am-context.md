# Reference — Account Manager (AM) role and sentence framing

> **One-line summary:** AMs are client-relay. Frame their asks as client asks routed through them — never as teammate work, never as ownership of delivery, never as resourcing authority. If you only read one sentence on this page, that is the one.

This page tells the skill (every collector, matcher, writer, executor) how to talk ABOUT an AM and how to talk TO an AM. AM identity (canonical email + aliases) lives in `schemas/preferences-page.md` § Account Managers. This page is about role context — what AMs do, what they don't do, how to phrase sentences so the framing is honest.

## What an AM is at WLIQ

- **Client-facing relationship owner.** The AM is the channel between the client and the PM. Every client ask reaches the PM through the AM (or directly cc'ing both).
- **Strict 1:1 mapping per PM.** Each PM has exactly one AM. The Preferences page carries the canonical AM identity for the running PM. The skill assumes single-AM throughout (collectors, matcher, writers). Multi-AM scenarios are intentionally out of scope — if a PM works with multiple AMs, Preferences picks the primary one.
- **No internal team.** AMs do not manage developers, designers, QA, or any pod. They have no headcount, no resourcing authority, no delivery responsibility. The PM owns delivery; the AM owns the relationship.
- **Sits upstream of the work, not inside it.** AM clarifications, sign-offs, and priority flips set the input boundary for the pod's work. They do not participate in execution.

## What AMs do

- Relay client asks (new requirements, scope changes, urgency flips, sign-offs, complaints).
- Clarify what the client meant when the PM is unsure.
- Confirm priorities when multiple asks compete for the same week.
- Hand items to the PM overnight when a same-day client deadline has been agreed (priority-lane signals — `collectors/orbit.md` § Priority Pass).
- Set client expectations on behalf of the PM (timing, scope, what's reasonable in a sprint).
- Carry urgency back to the client when the pod is blocked (missing inputs, escalations).

**Asks from AMs almost always carry client urgency.** When an AM says "small change", it is rarely small to the client — the client has framed it as small to the AM, and the AM is passing that framing on. Treat AM asks as the *visible surface of a client ask*, not as the AM's personal opinion.

## What AMs do NOT do

- Pick developers / designers / QA for tasks.
- Approve technical scope (architecture choices, library picks, refactor calls).
- Manage subtasks, assignments, or status flips inside Orbit.
- Estimate effort or hours.
- Run audits (SEO, AI, accessibility, performance) — those go to the Marketing / SEO pool.
- Write code, design, copy, or QA work.
- Hold pod-leader authority over any functional pool (FE, WP/PHP, QA, Design, Content, BA).
- Stand in for the PM on internal-team handoffs.

If a sentence implies the AM is doing any of the above, the framing is wrong — rewrite it.

## Framing examples — right vs. wrong

The load-bearing rule is in the table. Use it as the reference for Summary text, Task Brief content, Recommended Action reasoning, PM Next Step text, Outcome handoff bodies, and any other render site that mentions an AM by name.

| Wrong framing | Right framing | Why |
|---|---|---|
| "Ellen will handle the dev assignment" | "Ellen is waiting on dev nominations from you — she'll relay your pick to the client" | AM does not pick devs; PM nominates, AM relays |
| "Loop Ellen in on the QA pass" | "Confirm with Ellen what the client signed off on before QA closes" | AM is upstream confirmation, not downstream observer of internal work |
| "Ellen's team needs the deliverable by Thursday" | "Ellen needs the deliverable by Thursday — the client is waiting via Ellen" | No team behind an AM |
| "AM Ellen flagged a bug" | "Client (via AM Ellen) flagged a bug" | AM is the relay, not the originator — the bug came from the client |
| "Ellen will pick this up" | "Ellen is waiting on your decision so she can take it back to the client" | AM does not own internal pickup |
| "Ellen owns the timeline" | "Ellen owns the client conversation about the timeline; the timeline itself is yours to set" | Distinction matters when the timeline slips |
| "Hand this off to Ellen" | "Reply to Ellen with the answer she needs for the client" | Handoffs go to internal pod leaders, not AMs |
| "Ellen approved the scope" | "Ellen confirmed the client approved the scope" | AM does not approve scope — they relay client approval |
| "Ellen's update on the staging build" | "The client's update on the staging build, relayed by Ellen" | Update comes from the client; AM is the channel |

## When an AM IS the originator

The relay framing is the default — but a small set of asks genuinely originate with the AM, not the client. Examples:

- The AM noticed the PM is unreachable and is asking for a backup contact.
- The AM is requesting a status update for an internal sync (no client involved).
- The AM is heads-down on a new pitch and needs project history.

In those cases, framing as the AM's own ask is correct: "Ellen needs status for her internal sync — no client side, just a heads-up ask." Use judgement; the test is whether the ask exists because of a specific client action. If yes, frame as client-relayed. If no, frame as AM-originated.

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

## Priority-lane phrasing (already correct, do not break)

The existing phrasing for `signal_type: am_handed_to_pm_overnight_due_today` rows is consistent with this page and should not be rewritten by the AM-framing sweep:

```
<AM full name> put this on your plate overnight, due today.
Proposed delegate: <delegate name> (<reason>).
```

```
<AM name> assigned this task to <PM name> overnight; passing the dev work to you to start today.
```

These already frame the AM as relay (`put this on your plate` = handed it over from client, did not commit to do it themselves).

## What this page does NOT cover

- AM identity lookup (canonical email + aliases). See `schemas/preferences-page.md` § Account Managers and `synthesis/matcher.md` § Identity matching is alias-aware.
- AM Ping Drafts content / DSL. See `writers/notion.md` § AM Daily Ping block and `schemas/preferences-page.md` § AM Daily Ping Block.
- Tone level for AM-facing English (professional, not 4th-5th grade). See `writers/plain-language.md` § Where it applies (AM row + AM Ping Drafts row).
- Which AM signals trigger the priority lane. See `collectors/orbit.md` § Priority Pass.

## Cross-references

- [[feedback_assignee_role_boundaries]] — pool-vs-person routing; same role-boundaries principle, applied to developers + pod leaders.
- [[project_am_role_context]] — memory pointer back to this page (1:1 PM-AM decision, relay-not-team principle).
