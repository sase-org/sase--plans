---
tier: tale
title: Render ARTIFACTS directly below PLAN
goal: 'The Agents prompt and metadata views present the ARTIFACTS lane immediately
  after PLAN in SASE CONTEXT, with navigation hints, documentation, and visual coverage
  matching the new narrative order.

  '
create_time: 2026-07-17 09:02:15
status: done
prompt: 202607/prompts/artifacts_below_plan.md
---

# Plan: Render ARTIFACTS Directly Below PLAN

## Context

The prompt-panel context renderer currently declares `PLAN`, `MEMORY`, `SKILLS`, `WORKSPACES`, and `ARTIFACTS` as one
ordered set of optional lanes. It renders only non-empty lanes, and it invokes their renderers in that same order. As a
result, the order controls both what users see and the numeric file-hint allocation performed while each lane is built.
The ARTIFACTS lane is currently last, even though its commits, deltas, and explicit output files are the most useful
companion to the plan's statement of intent.

This is presentation-only Textual/Python behavior. It does not require a Rust core or wire-format change, and it should
not change artifact discovery, enrichment, persistence, or the internal `Commits` / `Deltas` / `Artifacts` field order.

## Implementation

1. Change the declarative SASE CONTEXT lane order in `src/sase/ace/tui/widgets/prompt_panel/_agent_context.py` to
   `PLAN`, `ARTIFACTS`, `MEMORY`, `SKILLS`, `WORKSPACES`. Continue to build each lane atomically, omit empty lanes,
   preserve the single `sase-context` navigation marker, and calculate the returned PLAN text range exactly as today.

   Because optional lanes are filtered from the declared sequence, ARTIFACTS will be directly below PLAN whenever both
   exist and will be the leading present lane when PLAN is absent. Rendering ARTIFACTS earlier must also leave hint
   assignment coupled to display order: plan paths first, then artifact commits/deltas/files, followed by audited memory
   and any later hint-bearing lanes.

2. Strengthen the renderer and prompt-header contracts in `tests/ace/tui/widgets/test_agent_context.py` and
   `tests/ace/tui/widgets/test_agent_display_plan_section.py`. Encode the new full order explicitly rather than relying
   only on the production order constant, cover every optional-lane presence combination, and assert that ARTIFACTS
   follows PLAN before MEMORY/SKILLS/WORKSPACES in the maximal header flow. Retain coverage for ARTIFACTS' internal
   field order and for PLAN remaining the first lane when present.

3. Update the user-facing descriptions in `docs/ace.md` and `docs/agent_images.md` so they describe ARTIFACTS as the
   plan-adjacent output lane rather than the bottom-ranked lane. Document the complete new lane sequence and keep the
   separate promise that `Commits`, `Deltas`, and `Artifacts` remain ordered within ARTIFACTS.

4. Regenerate only the ACE PNG goldens whose rendered lane positions actually change. The all-lanes metadata zoom
   fixture `agents_context_zoom_modal_120x40.png` is the primary expected update; inspect snapshot failure artifacts and
   the resulting image to confirm that ARTIFACTS is immediately below PLAN, content is neither clipped nor lost, and the
   remaining lanes retain their relative order. Do not accept unrelated renderer drift or rewrite unaffected goldens.

## Validation

After running `just install` as required for an ephemeral workspace:

- Run the focused SASE CONTEXT and prompt-header widget tests, including the exhaustive optional-lane combinations and
  the artifact-metadata integration coverage.
- Run the affected context-zoom visual test once to obtain and inspect the intentional diff, update its golden with
  `--sase-update-visual-snapshots`, then rerun without the update flag to prove exact equality.
- Run `just check` for formatting, lint, SASE validation, the complete test suite, and all committed PNG snapshots.

## Risks and Guardrails

- Reordering render invocation also reorders numeric path hints. Tests should verify that hint mappings stay aligned
  with the paths as displayed, avoiding a visual/navigation mismatch.
- ARTIFACTS can be substantially taller than an audited event lane. The visual review must confirm the zoom panel still
  scrolls normally and that moving the lane earlier does not hide or truncate later context.
- Keep enrichment and rendering costs unchanged: this work only changes the traversal order and must not add synchronous
  I/O, extra refreshes, or a second header build on the Textual event loop.
