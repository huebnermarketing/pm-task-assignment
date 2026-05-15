# Plain-Language Writer

## Purpose

Ensures every string that reaches the India delivery team is written in simple, accessible English — the 4th-to-5th grade general English level. Role-specific technical terms are preserved untouched. Everything else gets simplified.

This rule applies ONLY to content read by the India delivery team. PMs, AMs, and leadership get normal professional English.

## The rule in one sentence

> Simple general English for the delivery team. Role-specific technical jargon stays. Corporate English goes.

## Where it applies

| Content | Language rule |
|---|---|
| Slack handoff messages to team members | Plain-language (4th-5th grade) |
| Orbit task body (all 6 sections) | Plain-language (4th-5th grade) |
| Orbit task comments the team will read | Plain-language (4th-5th grade) |
| Slack handoff titles / subjects | Plain-language |
| Email TO the delivery team (rare) | Plain-language |
| **Emails to clients or AMs** | **Normal professional English** |
| **Slack to AMs or leadership** | **Normal professional English** |
| **Summary column in Morning Queue (PM reads)** | **Normal professional English** |
| **Proposed Email section of row detail (PM reviews)** | **Normal professional English** |
| **Internal documentation** | **Normal English** |

## What counts as 4th-5th grade general English

- **Short sentences.** Usually 10-15 words. Definitely no complex clauses or nested conditionals.
- **Simple verbs.** "Make", "do", "send", "start", "finish", "check", "fix" — not "execute", "facilitate", "orchestrate", "implement" unless the role-specific version demands it.
- **Common words.** "Help" not "assist". "Use" not "utilize". "Start" not "commence". "Change" not "modify".
- **Active voice.** "The client approved it" not "It was approved by the client".
- **Direct instructions.** "Do X. Then Y." Not "Once you have completed X, proceed to Y."
- **One idea per sentence.** Break up compound thoughts.

## What counts as a role-specific technical term (keep as-is)

Domain vocabulary the assignee uses every day. Don't simplify or replace:

| Role | Technical terms that stay |
|---|---|
| FE / WordPress dev | plugin, theme, child theme, hook, filter, WP admin, custom post type, shortcode, WooCommerce, REST API, Gutenberg, Elementor, Beaver Builder, responsive, breakpoint |
| BE dev | API endpoint, database, migration, schema, PHP version, cron, webhook, middleware, authentication, SSL, cache |
| Designer | Figma, mockup, wireframe, prototype, component, design system, hi-fi, lo-fi, artboard, asset |
| QA | regression, smoke test, test case, browser matrix, staging, production, bug, console error |
| Content | SEO, meta, H1, alt text, internal link, anchor text, schema markup |
| Any dev | staging URL, preview link, commit, branch, deploy, build, rollback |

## Corporate English to strip

Replace these patterns:

| Corporate English | Plain language |
|---|---|
| "Please find attached" | "Here is" / "I'm sharing" |
| "In order to" | "To" |
| "At this point in time" | "Now" |
| "Commence working on" | "Start" |
| "Per our conversation" | "Like we talked about" |
| "Utilize" | "Use" |
| "Approximately" | "About" |
| "Subsequent to" | "After" |
| "Prior to" | "Before" |
| "In the event that" | "If" |
| "Expeditiously" | "Quickly" / "Soon" |
| "Please ensure that" | "Please make sure" |
| "Going forward" | "From now on" |
| "Kindly" (at the start) | (drop it or say "Please") |

## Sentence structure patterns

### Good (plain-language)

> "Start the revisions today. The client needs them by Thursday. Everything is in the Orbit task."

> "Fix the hero image. Make it match the new brand guide. Test on mobile when done."

> "Ravi is taking this over. He worked on Phase 1 and knows the theme. Please log your hours when you finish."

### Not good (too corporate)

> "Kindly commence the revision work at your earliest convenience. The client has indicated that they require these deliverables prior to their board meeting on Thursday."

> "In order to facilitate the updates, please ensure that you have reviewed the attached feedback documentation thoroughly prior to initiating the work."

## Handling tone samples (per-PM calibration)

When the PM has provided tone samples in Preferences, use them to match voice. The samples are observations, not constraints — they tell the writer how this PM tends to speak to their team. If the sample uses shorter greetings, shorter greetings. If the sample includes emojis, use them sparingly. Preserve the PM's natural voice.

Never copy a sample verbatim. Use it for style guidance only.

When no samples are available, default to a generic professional-warm tone at the required grade level.

## How to apply the rule

Whenever any module calls this writer:

1. Receive the target content + the target audience (one of: team, client, AM, leadership, PM).
2. If audience ≠ team, pass content through UNCHANGED.
3. If audience = team, run simplification:
   - Shorten sentences
   - Replace corporate vocabulary
   - Preserve role-specific terms (look up role if known)
   - Keep all factual content — never lose meaning for simplicity's sake
4. Return the simplified version.

## Role-specific term detection

When simplifying, identify terms that should be preserved:

- Look at the assignee's Orbit role (FE, BE, etc.) if known
- Look at the task domain (WordPress work, design work, SEO work)
- Cross-reference the term against the table above
- When in doubt, preserve — it's safer to keep a term than to simplify something the assignee expects to see

## Special cases

### Short notes / titles

For titles and one-liners, prioritize clarity over grade-level strictness. A Slack handoff title like "Fix hero image bug" is fine — it's already plain. Don't over-simplify short strings.

### Numbers and dates

Use specific numbers (`3`, `12`), specific dates (`30 April 2026`), specific times (`2:00 PM`). Never "approximately" or "around" unless precise data isn't available.

### Greetings and closings

- Greetings: `Hey [name]` or `Hi [name]` — warm, short.
- Closings: Drop formal closings for internal team Slack. An occasional `Thanks` is fine. No `Best regards` or `Sincerely`.

## What this writer does NOT do

- Does not simplify terms the role requires.
- Does not apply to PM-facing or AM-facing content.
- Does not translate to other languages — only simplifies English.
- Does not check spelling or grammar (assume input is already clean).
- Does not remove critical context. If simplification would lose meaning, keep the longer version.
