---
tier: tale
title: Reserve epic phase planning handoffs for large phases
goal: 'Epic phase workers sized small or medium implement their assigned bead directly,
  while only large phase workers receive the #plan xprompt, with launch previews,
  authoring guidance, model-role descriptions, documentation, and regression coverage
  all reflecting the same policy.

  '
create_time: 2026-07-23 12:43:49
status: wip
---

- **PROMPT:** [202607/prompts/large_only_epic_phase_planning.md](prompts/large_only_epic_phase_planning.md)

# Plan: Reserve epic phase planning handoffs for large phases

## Context and behavioral contract

`sase bead work` currently appends `#plan` to both medium- and large-sized phase worker prompts. The plan-file dry-run
preview advertises that same behavior, and the plan authoring guidance and model-role descriptions teach users that both
sizes plan before implementation. Change the contract so:

- Missing/legacy and explicit `small` phases continue to implement directly and use `@small_phase_worker` when no
  explicit model is stored.
- `medium` phases implement directly without `#plan` and continue to use `@medium_phase_worker` when no explicit model
  is stored.
- Only `large` phases append `#plan`, after the phase-work xprompt reference, and continue to use `@large_phase_worker`
  when no explicit model is stored.
- An explicit per-phase model continues to override the size-selected model alias without changing whether the phase
  receives `#plan`.
- `%auto`, clan and bead identity, agent and bead dependency waits, ChangeSpec wrapping, land-agent selection, and all
  other epic launch behavior remain unchanged.

This is prompt-rendering policy and its presentation/documentation, so it stays in the existing Python launch-rendering
layer; the Rust epic DAG/wire payload already supplies the normalized phase size needed by that layer and requires no
schema or binding change.

## Centralize and apply the planning decision

In `src/sase/bead/work.py`, add a small authoritative phase-size policy helper that uses the existing legacy-size
normalization and returns true only for `PhaseSize.LARGE`. Use it in `render_multi_prompt` when deciding whether to
append `#plan`. Keeping model selection in `phase_model_directive_value` separate makes it explicit that medium model
routing remains intact even though the planning handoff is removed.

In `src/sase/bead/cli_work_from_plan_render.py`, use the same helper to add the ` · #plan` preview suffix only for large
phases. Do not duplicate a raw string set check in the preview, so future policy changes cannot make preview output
disagree with the emitted multi-prompt.

## Align user-facing semantics

Update `src/sase/main/plan_explain.py` so epic plan authoring guidance defines `small` as focused direct implementation,
`medium` as substantial work that can still be implemented directly from its phase description, and `large` as work that
needs a separate planning handoff and may itself justify an epic plan. Say unambiguously that small and medium phase
agents implement directly and only large phase agents create a plan before implementation.

Update `src/sase/llm_provider/model_alias_policy.py` so the `medium_phase_worker` description says it implements
directly; keep the large role description plan-first. Update its assertion in
`tests/llm_provider/test_config_aliases.py`. Mirror the corrected medium-role wording in the Models-panel visual fixture
and refresh only the affected intentional PNG golden(s), so the UI regression example does not preserve the old
contract.

Revise the phase-routing explanations in `docs/sdd.md`, `docs/beads.md`, and `docs/llms.md`. Preserve the documented
alias fallbacks, explicit-model precedence, legacy size behavior, and large-epic lander threshold while changing every
statement that says medium phases receive `#plan` or merit their own plan file.

## Regression coverage

Update `tests/test_bead/test_work_rendering_models.py` so both implicit and explicit model cases assert this exact
matrix: missing/small no `#plan`, medium no `#plan`, and large has `#plan` as the final line after the work-phase
reference. Retain the model assertions to prove removing the planning marker does not collapse medium onto the small
model lane.

Update `tests/test_bead/test_cli_work_epic_dry_run.py` to assert that a stored medium phase keeps its explicit model but
does not contain `#plan`, while the large phase still ends with the work reference followed by `#plan`.

Update `tests/test_bead/test_cli_work_from_plan_preview.py` to expect the medium preview without the planning suffix and
the large preview with it. Keep the legacy size-less preview coverage proving that it remains small/direct.

## Verification

Run the focused phase-rendering, epic dry-run, plan-file preview, and model-alias tests first. If the Models-panel
fixture changes a visual, run the dedicated Models-panel visual snapshot test, inspect the generated
actual/expected/diff artifacts, and accept only the intentional text-only golden update with
`--sase-update-visual-snapshots`; then rerun that visual test normally.

Because implementation changes files in this repository, run `just install` before the final `just check`, as required
by the repository instructions. Finish by searching the source and documentation for stale claims that medium phase
workers receive `#plan` or plan before implementation, and re-run any focused test touched by the final cleanup.
