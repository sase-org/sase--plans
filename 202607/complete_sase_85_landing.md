---
tier: tale
title: Complete and land rich epic clan summaries
goal: 'Epic sase-85 has launch-level stale-store recovery coverage against the current
  bead-work flow, its rich persisted output is exercised through the clan panel, and
  the epic is closed with post-close Symvision cleanup and plan status finalized.

  '
create_time: 2026-07-20 12:23:05
status: wip
prompt: 202607/prompts/complete_sase_85_landing.md
---

# Plan: Complete and land rich epic clan summaries

## Context

The landing audit confirmed that the implementation commits for `sase-85.1` and `sase-85.2` provide the intended
behavior: a missing epic lookup retries after a synchronous, non-pushing bead-store integration; final fallback paths
emit diagnostics; the launcher allows 20 seconds; and the renderer supplies bounded Rich markup with the full goal,
launch-time phase progress and statuses, size chips, descriptions, child epics, and the plan reference. Focused
renderer, sync, persistence, launch, and PNG snapshot tests pass.

The audit also found one unfulfilled phase note. `sase-85.3` requires the end-to-end launch exercises to include
recovery from a stale sidecar clone. Its commit added only a launch/persistence test against an already-current store
and a panel test built from handwritten markup. The unit-level retry test in
`tests/test_bead/test_clan_summary_epic_script.py` proves the helper calls its refresh hook, but it does not prove that
the current `sase bead work` launch path persists a rich summary when the claimed workspace starts stale. The panel
exercise also does not render the markup produced by that launch.

Commits after the first `sase-85` change were reviewed for integration. In particular, approved epic plan launches now
use store-scoped serialization and a blocking terminal sync, ACE now enriches agents with phase-bead and authored-plan
context, and test execution now uses host-global worker budgeting. The summary feature remains compatible with those
changes; the missing coverage should exercise the current launch flow rather than bypassing it.

## Phase 1: Complete launch-level recovery and rendering coverage

Strengthen `tests/test_bead/test_cli_work_epic_launch.py` and, only where shared setup belongs elsewhere, its existing
bead/sync test helpers.

1. Add an exercise whose launch workspace begins with a valid but stale remote-backed plans/bead clone that cannot
   resolve the epic. Make the remote/current store contain the epic and all of its phases, then run the same directive
   extraction and summary persistence boundary used by `sase bead work`. Assert that the built-in epic summary performs
   the blocking refresh, retries successfully, and persists the rich header and every phase title instead of the bare
   `[bold]EPIC <id>[/]` fallback. Prefer a real local Git remote and two clones so the test covers store resolution and
   integration; if the harness makes the subprocess boundary impractical, preserve the launch/persistence boundary and
   explicitly assert the retry hook is exercised before accepting a narrower simulation.
2. Make the clan-panel exercise render the actual persisted markup produced by the launch scenario, not a separately
   handwritten approximation. Keep focused assertions for the epic title, every phase title, rich layout markers, and
   absence of the exact fallback.
3. Remove redundant assertions in the existing phase-title loop and keep fixtures scoped so the file remains reliable
   under the repository's normal parallel test runner.
4. Run the focused epic-summary, remote-sync, persistence, launch, and clan-panel visual suites. Run `just install`
   before repository checks, then run `just check`.

Do not change the summary product behavior unless the realistic stale-clone exercise reveals an actual defect in the
current refresh or launch integration. If it does, repair the smallest owning layer and cover the regression through the
same launch-level test.

## Phase 2: Final landing

This is the final phase and must run only after Phase 1 is complete and green.

1. Re-run `sase bead show sase-85` and its three child beads to confirm all phase notes, including stale-clone recovery,
   are now represented by passing source-level tests.
2. Close the original epic with `sase bead close sase-85`.
3. After closure, follow the repository's Symvision-memory instructions and run `just symvision`. Remove stale `sase-85`
   epic-symbol whitelist entries and any unused code it reports, then rerun Symvision until clean. If this changes
   repository files, rerun `just check` after `just install` as required.
4. Open the plans sidecar through `sase repo open plans`, edit `202607/epic_clan_summary_rich.md`, and change its
   frontmatter from `status: wip` to `status: done`. Preserve all other plan content.
5. Verify `sase bead show sase-85` reports the epic closed, the plan frontmatter reports `done`, both repositories have
   only the intentional landing changes, and the final validation results are recorded for handoff.

## Risks and boundaries

- A mocked refresh that creates a bead in the same store can pass while the real stale clone remains broken; retain the
  remote-backed distinction through as much of the launch path as the test harness supports.
- Summary scripts are subprocesses, so in-process monkeypatches do not automatically cross the execution boundary.
  Arrange filesystem/Git state or use an explicit seam at the launch resolver rather than relying on an ineffective
  patch.
- Closing the bead expires Symvision epic exemptions immediately. Do not mark the plan done or stop after closure until
  the post-close Symvision cleanup and required checks are complete.
