---
tier: tale
goal: The ACE SASE PLAN section reports plan, tale, or epic according to how the user
  approved the plan, without changing plan-path selection or epic roadmap behavior.
create_time: 2026-07-16 07:23:09
status: wip
prompt: 202607/prompts/ace_plan_approval_tiers.md
---

# Plan: Correct ACE SASE PLAN approval tiers

## Context

ACE currently models the SASE PLAN section's effective tier as `plan`, `epic`, or `none`. Its derivation collapses tale
and commit-only approvals into `plan`, treats an explicitly uncommitted approval as `none`, and maps an authored tale to
`plan`. That loses the distinction already preserved by the plan-approval protocol: a user can approve without an SDD
commit, approve and commit as a tale, or approve and commit/launch as an epic.

This is an ACE presentation-model correction, not an expansion of bead-tier storage. Plan beads continue to use their
separate `plan`/`epic` execution classification, and the existing canonical-path and phase-roadmap rules remain intact.

## Approval-tier contract

Make explicit approval metadata authoritative over commit success and authored frontmatter:

| Evidence                                                                  | Displayed tier |
| ------------------------------------------------------------------------- | -------------- |
| `plan_action=approve`, or legacy `plan_committed=false` without an action | `plan`         |
| `plan_action=tale` or legacy commit-only `plan_action=commit`             | `tale`         |
| `plan_action=epic`                                                        | `epic`         |

An approval action continues to describe the user's chosen tier even if the corresponding commit or epic launch later
fails; `plan_committed` still selects the canonical archived versus SDD path, but does not downgrade the displayed
choice. When explicit action metadata is absent, retain compatibility by using a valid authored `tier: tale` or
`tier: epic`; a legacy committed plan with no readable authored tier falls back to `tale`, while an unresolved tier
remains unavailable.

## Implementation

1. Update the associated-plan presentation model in `src/sase/ace/tui/models/agent_associated_plan.py` so its tier type
   contains exactly `plan`, `tale`, and `epic`, and encode the precedence above in the effective-tier derivation. Keep
   commit-state calculation, canonical path selection, cache keys, deferred enrichment, role detection, and phase
   availability unchanged so the work adds no event-loop I/O or new refresh path.
2. Extend `src/sase/ace/tui/widgets/prompt_panel/_agent_plan_section.py` with a distinct tale presentation, reusing
   ACE's established plan/tale/epic visual language (cyan, gold, and amethyst), and remove the obsolete `none` value
   treatment. Preserve `unavailable` as the rendering for a genuinely unresolved optional tier.
3. Update the associated-plan model tests to cover the full precedence and compatibility matrix: no-commit approval,
   tale approval, commit-only approval, epic approval, failed/uncommitted persistence that must not override an explicit
   choice, authored-tier fallback for pending/legacy records, committed legacy fallback, and missing/damaged metadata.
   Keep assertions that path selection and epic phase summaries are independent of the displayed tier.
4. Update section-rendering tests to expect all three supported values and their distinct styles. Adjust the existing
   tale-backed ACE visual scenario and its PNG golden so it visibly guards `Tier: tale`; retain epic roadmap coverage
   and compact tale layout coverage.
5. Revise both SASE PLAN descriptions in `docs/ace.md` to document `plan`, `tale`, and `epic`, the approval-action
   meanings, the compatibility fallback, and the separation between displayed approval tier and committed/uncommitted
   path selection.

## Validation

- Run focused associated-plan model and section-rendering tests while iterating.
- Run the dedicated ACE visual snapshot test and inspect/accept the intentional tale-tier golden change.
- Run `just install` as required for an ephemeral workspace, followed by `just check` for the repository-wide lint,
  type, test, documentation, and snapshot gates.

## Non-goals and risks

- Do not add `tale` to `BeadTier`, bead database constraints, bead CLI choices, JSONL migration, or `sase bead work`;
  those tiers answer a different execution question.
- Do not change plan approval actions, persisted metadata, SDD schemas, or plan validation. The existing `plan_action`
  and `plan_committed` signals already contain the needed information.
- Preserve legacy and damaged-record behavior deliberately: explicit actions win, authored frontmatter is only a
  fallback, and truly unknown values render as unavailable rather than being mislabeled.
