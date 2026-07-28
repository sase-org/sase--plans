---
tier: tale
title: Land sase-ak after post-start integration
goal:
  The tribe-wait epic is fully integrated with later queued-runner work, reverified, closed without force, clean under
  Symvision, and marked done in its linked plan.
bead: sase-ak
create_time: 2026-07-28 18:23:11
status: wip
---

- **PROMPT:** [202607/prompts/land_sase_ak.md](prompts/land_sase_ak.md)
- **PARENT:**
  [202607/tribe_wait_reference_validation_and_display.md](https://github.com/sase-org/sase--plans/blob/main/202607/tribe_wait_reference_validation_and_display.md)
- **BEAD:** [sase-ak](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ak/README.md)

# Land `sase-ak` after integrating post-epic changes

## Objective

Finish the `sase-ak` land-agent audit by repairing the integration regression introduced after the tribe-wait display
API landed, removing an explicitly temporary parallel-phase compatibility bridge, re-verifying the epic against its plan
and current source, and only then closing the epic and completing its post-close Symvision and plan-file cleanup.

## Evidence already established

- `sase bead show sase-ak` reports four phase children, all closed with `resolution: done`: `sase-ak.1`, `sase-ak.2`,
  `sase-ak.3`, and `sase-ak.4`.
- The linked epic plan is `202607/tribe_wait_reference_validation_and_display.md` in the `plans` sidecar and currently
  has `status: wip`.
- The four epic commits are:
  - `d67de4caf` — reserved tribe target guards (`sase-ak.1`)
  - `21e75272f` — shared snapshot-driven tribe binding (`sase-ak.2`)
  - `ed04c42f2` — ACE tribe wait rendering (`sase-ak.3`)
  - `641229f89` — reserved wait unresolvable rendering (`sase-ak.4`)
- Non-epic commits after the first epic commit are `2c77fbecd`, `ee2bb5eee`, `5568411c9`, and `d8c2f5019`. The first
  three are isolated from the feature. `d8c2f5019` changes queued runner-slot rendering in the same detail-header
  surface as `ed04c42f2`.
- Current `just lint` fails mypy at `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py` because
  the queued branch calls `build_wait_lanes()` without the tribe-display phase's required `tribe_wait_bindings` keyword.
- The same mismatch is a runtime regression: these two existing tests fail with
  `TypeError: build_wait_lanes() missing 1 required keyword-only argument: 'tribe_wait_bindings'`:
  - `TestRunnerSlotWaitRendering::test_explicit_drain_barrier_is_queued_and_unambiguous`
  - `TestRunnerSlotWaitRendering::test_explicit_priority_renders_in_detail_wait_line`
- `src/sase/core/wait_dependency_resolution/_tribe_binding.py` still contains `_is_reserved_tribe_name()`, documented as
  a bridge only “until both changes are landed.” Both phases are now landed, so its `getattr`/literal fallback
  duplicates the canonical predicate in `sase.core.agent_tribe`.

## Phase 1: Integrate the current tree

1. In the queued branch of `src/sase/ace/tui/widgets/prompt_panel/_agent_display_header_metadata.py`, pass
   `tribe_wait_bindings=tribe_wait_bindings` to `build_wait_lanes()`. Preserve the runner-only lane filter and the
   queued-status behavior from `d8c2f5019`.
2. In `src/sase/core/wait_dependency_resolution/_tribe_binding.py`, call the canonical
   `agent_tribe.is_reserved_tribe_name()` directly. Remove the temporary `_is_reserved_tribe_name()` bridge, its
   fallback literal, and its stale parallel-phase comment.
3. Do not broaden the work into unrelated runner-slot, wait, or tribe behavior. Existing tests already cover the
   regression and the reserved binding.

## Phase 2: Re-verify the epic

1. Run the focused tests that cover:
   - reserved `%wait` and `#fork` rejection and reserved tribe assignment;
   - tribe binding ordering, cutoff/self exclusion, clan aggregation, pending/bound/reserved classification, and index
     delegation;
   - ordinary/tribe wait satisfaction and missing-target classification;
   - `[tribes]` pending, bound, and unresolvable detail rendering;
   - queued explicit threshold/priority rendering;
   - row marker and render-cache invalidation.
2. Run `just test-visual` because phases 3 and 4 changed ACE PNG snapshots. Do not update goldens unless an intentional
   visual change is present and its diff artifacts are inspected.
3. Run `just check`. `just install` has already been run in this workspace, but rerun it first if the environment has
   changed or the follow-up runs in a different workspace.
4. Re-read the four bead descriptions/notes and the epic plan requirements against the current source and confirm every
   item remains implemented. Recheck `git log` after `d67de4caf` for any newer non-epic commit that landed after this
   plan was written, and integrate any newly relevant overlap before proceeding.

## Phase 3: Close and clean up the epic

This is the final phase and must run only after phases 1–2 are green.

1. Close the epic without force:

   ```bash
   sase bead close sase-ak --note "<concise audit summary covering the four phase implementations, focused/full/visual verification, and the post-start commit integration>"
   ```

   If the close is rejected, resolve or deliberately disposition the named unfinished phases; never force merely to make
   the command succeed.

2. After the close, run `just symvision`. Follow its diagnostics exactly: remove stale `sase-ak` epic-symbol entries if
   any are reported, and delete or privatize code that is genuinely unused now that the epic is closed. Do not whitelist
   symbols only used by tests.
3. If Symvision cleanup changes the primary repository, rerun the exact Symvision command and then `just check`.
4. Open the `plans` sidecar through `sase repo open plans -r "<reason>"`, then change only the linked epic plan
   frontmatter from `status: wip` to `status: done`. This plan-file update is the final mutation.
5. Report the closed bead resolution, verification results, integration fixes, Symvision result/cleanup, and the
   completed linked plan status.
