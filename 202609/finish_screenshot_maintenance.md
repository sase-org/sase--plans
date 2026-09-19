---
tier: epic
title: Finish screenshot-maintenance landing integration
goal: "The canonical finalizer guidance commits every dirty screenshot golden and labels
  unrelated golden updates, while the full ACE and pager visual corpus passes against
  the post-epic metadata-only Agents detail default with every intentional golden change
  reviewed.

  "
parent_bead: sase-12z
phases:
  - id: finalizer-guidance
    title: Require screenshot goldens in finalizer commits
    depends_on: []
    size: small
    description:
      "finalizer-guidance: teach the canonical finalizer skill to retain every dirty
      screenshot golden and add the required trailer for unrelated updates, with source
      and rendered-skill regressions."
  - id: visual-integration
    title: Integrate the metadata-only Agents default with the visual corpus
    depends_on:
      - finalizer-guidance
    size: medium
    description:
      "visual-integration: repair stale secondary-panel test flows, review and accept
      only the intentional metadata-only golden changes, and prove the complete corpus
      is clean."
proposed_by: bbugyi200.athena.sase-12z.land--1
create_time: 2026-09-18 20:31:28
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_screenshot_maintenance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_screenshot_maintenance.md)

# Plan: Finish screenshot-maintenance landing integration

## Landing audit baseline

The four original `sase-12z` phases and their commits are complete. The candidate
capture protocol, safe maintenance runner, manifest-backed review reports, Just/CI
wiring, documentation, and the prior full golden refresh are present on `master`.
`sase bead epic-symbols sase-12z` currently reports no entries.

Two phase-proposed follow-ups are already resolved and are not part of this plan:

- `sase-12z.1` reported completion-spec snapshot ordering drift. The later portable
  completion-loader work updated the checked-in spec, and
  `tests/completion/test_snapshot.py` now passes 4/4. Do not create a task for it.
- `sase-12z.4` reported sidecar clone-staging failures. The later unborn-empty-remote
  fix addresses that path, and all `tests/sdd_store/test_sidecar_*.py` tests plus
  `test_sidecar_materialization_uses_remote_not_divergent_primary` now pass 59/59. Do
  not create a task for it.

One commit landed after the original phase-4 refresh:
`5d158ad65ba99293e4cad22d013abbc0e064ad76` changed the Agents detail view's initial
layout from secondary-larger to metadata-only. The subsequent full
`just fix-tui-screenshots --check` run reached 962 passing visual tests and one skip,
but six parametrized nodes failed before comparison because five old test flows still
assume that File or LLM Calls is visible initially:

- external-repository File panel;
- linked-repository File panel;
- linked-repository commit-message/File panel;
- expanded and full LLM Calls detail levels; and
- metadata slow-tool fold levels.

The failed run correctly left goldens untouched and marked its inventory incomplete.
Exact comparison of its 700 retained partial captures found 116 updated ACE candidates,
554 unchanged ACE candidates, and 30 unchanged pager candidates. Every update belongs to
an Agents surface. Representative review confirms the intended header/layout change from
`view: file` to `view: none`; the complete successful run remains authoritative because
failed nodes could not contribute all captures or prove orphanhood.

## Phase `finalizer-guidance`: Require screenshot goldens in finalizer commits

Update the canonical generated-skill source `src/sase/xprompts/skills/sase_final.md`; do
not edit installed provider copies. Add a clear finalizer rule that:

1. every dirty ACE or pager screenshot golden is included in the repository's commit,
   even when the visible change appears unrelated to the agent's authored source work;
2. unrelated golden changes are inspected rather than silently discarded or deferred;
3. when such changes are genuinely unrelated, the Conventional Commit message ends with
   the exact trailer `UNRELATED_SCREENSHOT_UPDATES=<reason>` using a concrete
   explanation in place of `<reason>`; and
4. no trailer is required when the screenshot changes are part of the work described by
   the commit.

Extend the source-template regressions so both the canonical template and the packaged
or rendered finalizer skill are pinned to the all-goldens rule and exact trailer syntax.
Use the existing finalizer-skill tests in `tests/test_xprompt_skill_sources.py` and
`tests/main/test_init_skills_source_content.py` rather than creating a parallel test
harness. Preview generation with `sase skill init --diff`; do not deploy generated
skills from this unlanded revision.

Validation:

- Run the focused finalizer-skill source tests.
- Run `just fix` (or at minimum `just fmt`) and `just check` after the tracked changes.

Acceptance:

- The canonical source unambiguously requires all dirty screenshot goldens in the
  commit.
- Direct-source and packaged/rendered-skill tests fail if either the requirement or the
  exact `UNRELATED_SCREENSHOT_UPDATES=<reason>` trailer is removed.
- No generated skill is deployed from the working branch.

## Phase `visual-integration`: Integrate the metadata-only Agents default with the visual corpus

Keep the new application default; repair test setup rather than reverting production
behavior or weakening semantic assertions. Reuse or extend
`tests/ace/tui/visual/_ace_agents_png_snapshot_helpers.py` so tests state the view they
need through the public Agent-view picker and wait for the relevant content before
asserting or capturing.

- The external-repository, linked-repository, and linked commit-message File-panel tests
  must explicitly select a visible File secondary-larger layout before checking File
  content. Reuse the existing File-layout helper where possible.
- The LLM Calls helper must select LLM Calls, wait for its rows/content, then explicitly
  choose a visible secondary layout before detail-level assertions and snapshots. The
  populated, expanded, and full snapshots must continue covering the LLM Calls panel,
  not silently turn into metadata-only snapshots.
- The slow-tool test is metadata coverage. Replace its stale picker/`0` sequence with an
  explicit metadata-only selection that dismisses the picker before section navigation;
  retain the section, footer, fold-level, and convergence checks.
- Sweep the visual tests for the same initial-secondary-layout assumption and update any
  additional flow exposed by targeted or complete execution. Do not broadly rewrite
  tests whose intended snapshot is the new default metadata-only view.

First run a targeted check-only screenshot capture for the affected test files. It must
execute the semantic assertions successfully without writing goldens. Then run
`just fix`, `just check`, and the integrated `just check-full` through `/sase_monitor`
with a follow-up that inspects the new screenshot manifest, report, and Git diff before
finalization. Review every creation/removal and every update group, expanding unexpected
groups. The 116 retained partial ACE updates are the expected starting signal: they
represent the deliberate metadata-only default and may be accepted when the successful
report confirms that cause. Pager changes, unexpected non-Agents changes, new/stale
goldens, or secondary-panel snapshots that lost their intended content require a code or
test fix rather than blind acceptance.

After the reviewed update succeeds, run a complete `just fix-tui-screenshots --check`
through `/sase_monitor`. Require a complete inventory, zero drift, an unchanged golden
tree, and no test failures. Record the final test counts, change counts, report paths,
and review conclusion for the child epic's land agent.

Acceptance:

- All previously failing semantic visual nodes pass while continuing to capture their
  named File, LLM Calls, or slow-tool content.
- The full update report contains only intentional post-epic integration changes; all
  golden modifications are included in the phase commit.
- `just check` passes, the monitored `just check-full` passes, and the final monitored
  full check reports no drift.

## Scope boundary

This child epic contains only the remaining implementation and integration work. The
parent land agent remains responsible for the normal `sase-12z` close note, final
epic-symbol recheck, post-close Symvision check, and marking the original linked plan
done after this child epic lands.
