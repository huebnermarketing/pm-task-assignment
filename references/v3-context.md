# Reference — V3 (PM Automations V3)

V3 is the long-term architecture / reference Notion workspace for WLIQ's PM automations program. It is not where this skill writes. It is not where this skill collects. It exists for design context and historical record.

Public link: [PM Automations V3](https://www.notion.so/3433846840c88082b79bf6af6c89a39f)

## What lives in V3

- The original architecture diagrams for WLIQ's PM automations vision.
- The locked-in design decisions that shaped this skill (data quality standards, plain-language rules, role-fit recommendation logic).
- The Orbit Data Quality Standard that this skill's `schemas/orbit-dq-standard.md` is derived from.
- Long-form rationale documents that the skill files reference for "why" but never modify.

## What V3 is NOT

- Not a place this skill writes. Not even auto-comments. Not even auto-citations.
- Not a primary collection source. The skill's collectors target Orbit / Gmail / Fathom / Notion (the PM's own parent page), not V3.
- Not editable from any skill code path. The "V3 stays sealed" rule is non-negotiable across every mode, executor, and writer.

## How files in this skill relate to V3

| File                                     | V3 link                                                                                        |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `schemas/orbit-dq-standard.md`           | Carries the 6-section task body forward from V3's Orbit Data Quality Standard.                 |
| `writers/plain-language.md`              | The plain-language rule was locked in V3 design discussions and is enforced in this skill.     |
| `synthesis/pod-inference.md`             | The role-fit-only assignment policy comes from V3.                                             |
| `modes/mode-1-morning-collection.md`     | Mode 1 lookback / silent-exit behavior is V3-locked.                                           |

## Operational rule for the skill

If a piece of information is sourced from V3, the skill cites it textually (e.g., "carried from WLIQ's V3 reference") but never opens or modifies the V3 page itself. If a future workflow seems to demand writing to V3, treat that as a design-question for the V3 maintainers — not as a runtime decision the skill should make.
