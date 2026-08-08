---
tier: tale
title: Repair the post-start XPrompt write-target conflict and land sase-h8.10.5
goal:
  Restore a coherent XPrompt write-target API on the integrated base, verify the
  combined tree, close sase-h8.10.5 normally, pass post-close Symvision, and mark its
  linked epic plan done.
proposed_by: bbugyi200.athena.sase-h8.10.5.land
bead: sase-h8.10.5
create_time: 2026-08-08 18:09:02
status: wip
---

- **PARENT:**
  [202608/h8_10_remaining_landing.md](https://github.com/sase-org/sase--plans/blob/main/202608/h8_10_remaining_landing.md)
- **BEAD:**
  [sase-h8.10.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-h8/sase-h8.10.5.md)

# Repair the post-start XPrompt write-target conflict and land `sase-h8.10.5`

## Goal

Land epic `sase-h8.10.5` only after integrating the two commits that reached
`origin/master` during its final verification. Restore a coherent XPrompt write-target
API, revalidate the combined tree, close the epic normally, run the required post-close
Symvision cleanup, and finally mark the epic's linked plan done.

## Verified baseline

- Epic `sase-h8.10.5` is still `in_progress`; all three children (`sase-h8.10.5.1`
  through `.3`) are closed with resolution `done`. The epic itself has no notes. Every
  child note and full bead history was reviewed.
- The epic's main-repository commits are:
  - `47cad6a02` (`sase-h8.10.5.2`): plan-link concurrency tests use
    `tests._load_tolerant.LOAD_TOLERANT_TIMEOUT` while the 0.2-second negative ordering
    assertion remains intentionally separate; the current XPrompt task-worker wording is
    asserted.
  - `38fd25afd` (`sase-h8.10.5.1`): the load-sensitive timed contract oracle and its
    normalization machinery were removed; the current manifest has an exact 35-entry
    budget with a diagnostic naming the measured serial cost and curation plan.
  - `607b72bb0` (`sase-h8.10.5.3`): the flake baseline cutoff moved past fixed
    historical XPrompt failures and contains no stale node entries.
- The plans-sidecar commit `aeecfc99` removed the dangling parent-plan link from
  `202608/flake_class_residue.md`; the file retains `parent_bead: sase-h8` and
  `status: wip`. The epic's own linked plan is
  `plans:202608/h8_10_remaining_landing.md`, also currently `wip`.
- Fresh verification at `607b72bb0` passed:
  - 78 focused watchdog, residue, contract-manifest, plan-link concurrency, XPrompt,
    metadata-search, and wait-checker tests;
  - `tools/check_test_wait_helpers`;
  - six contention repeats of `tests/test_contract_manifest.py`, with zero failed nodes;
  - `just check-full`, including the post-run flake-baseline gate;
  - `just test-visual`: 563 passed, 1 skipped.
- Follow-up disposition is already durable:
  - The fixed historical XPrompt baseline proposal is resolved in-epic by `607b72bb0`.
  - Child `sase-h8.10.5.3`'s five new one-of-three full-contention nodes were routed as
    one semantic duplicate to in-progress umbrella task `sase-ct` by a
    `sase-h8.10.5.land` `+1`, and a matching `DISCOVERED ISSUE` note was added to active
    epic `sase-h8`; no duplicate task was created. The already-recorded
    `test_large_backlog_builds_one_inventory_and_publishes_each_hood_once` recurrence
    supplied no new evidence and was not counted again.
  - The bead-page publication import proposal did not recur: the installed runtime now
    imports `ARTIFACT_REF_PATH_FILTER_WIRE_SCHEMA_VERSION`, and a read-only
    `sase bead pages refresh --bead sase-h8.10.5.3 --json` completed with no errors.
  - A separate setup-cache defect discovered during verification is ready task
    `sase-hu`, with evidence `file:explicit:714a5e3b00cf5e01c18f9fec`. It is not caused
    by this epic and does not block its feature: cached core-binding failures survive a
    successful editable Rust rebuild because the environment fingerprint omits the
    extension target behind `sase_core_rs.pth`.
- The final base refresh advanced the checkout to `be6277b67` through two non-epic
  commits:
  - `8f8c39829` lazily loads memory XPrompts and renamed public `XPromptWriteTarget` to
    private `_XPromptWriteTarget`.
  - `be6277b67` renders artifact refs through ref XPrompts and depends on that rename.
    The ref-rendering and lazy-memory changes do not duplicate the epic's contract
    budget or wait convention, and the wait-idiom checker remains green.
- The rename did not integrate with XPrompt write-target consumers that landed around
  it. `rg` shows runtime imports/construction of `XPromptWriteTarget` in
  `_prompt_bar_save_xprompt.py`, `_prompt_bar_save_xprompt_git.py`, and
  `xprompt_browser_actions.py`, plus direct tests in
  `tests/xprompt/test_write_targets.py`. The symbol no longer exists or appears in
  `__all__`. A focused 60-test command on `be6277b67` therefore produced 60 setup
  errors, all rooted at `_prompt_bar_save_xprompt_git.py` importing the missing name.
  This is remaining post-start integration work and must be repaired before closure.

## Phase 1: Restore one coherent write-target API

1. Reconfirm `HEAD` against `origin/master` and inspect any additional commits that
   landed after `be6277b67` before editing. Integrate new overlap rather than
   overwriting it.
2. Repair the cross-commit contract in `src/sase/xprompt/write_targets.py` and every
   caller. The current evidence favors restoring `XPromptWriteTarget` as the public
   dataclass and restoring its `__all__` entry: multiple production modules and tests
   construct or annotate it across module boundaries, so a private name would either be
   misused across modules or require a new factory/value protocol. If current source has
   since established a cleaner public replacement, migrate all consumers to that
   replacement instead. Do not leave compatibility aliases or Symvision suppressions
   merely to hide a split API.
3. Search the complete tree for both `XPromptWriteTarget` and `_XPromptWriteTarget`.
   Every remaining reference must agree with the selected public/private boundary, and
   importing `sase.ace.tui` in a fresh interpreter must succeed.
4. Preserve both post-start features: lazy memory-note discovery and artifact-ref
   renderer routing. Do not revert either commit wholesale.

## Phase 2: Revalidate the integrated combined tree

1. Run `just install` first.
2. Run focused coverage for the broken API and the two post-start commits:

   ```bash
   .venv/bin/pytest -q \
     tests/xprompt/test_write_targets.py \
     tests/ace/tui/actions/test_prompt_save_xprompt_git.py \
     tests/ace/tui/modals/test_xprompt_select_modal.py \
     tests/main/test_project_handler_list_show.py \
     tests/test_artifact_ref_preprocessing.py \
     tests/test_xprompt_ref_sources.py \
     tests/test_bead_xprompt_tags.py
   .venv/bin/python tools/check_test_wait_helpers
   ```

3. Re-run
   `SASE_CONTENTION_REPEAT=6 just test-contention -- tests/test_contract_manifest.py`
   only if any intervening commit changes the manifest, contract-selection tooling, or
   wait/selection behavior. Otherwise retain the fresh zero-failure six-repeat evidence
   above and run `tests/test_contract_manifest.py` once in the whole-tree lane.
4. Run `just check-full` on the repaired current head. The previous fast-forward broke
   repository-wide imports, so a diff-scoped pass alone is insufficient. Require all
   lint/SASE/committed-plan gates, the full pytest lane, and the flake-baseline gate to
   pass.
5. Run `just test-visual` on the repaired current head. Do not update snapshots unless a
   deliberate rendering change is identified; the API repair itself should not change
   pixels.
6. Re-fetch `origin/master` after verification and audit any new commits since the
   repaired head. If another integration issue appears, keep it in this plan rather than
   closing over a stale base.

## Phase 3: Close and finish the epic plan

This is the final phase. Perform it only after phases 1-2 are green.

1. Re-run `sase bead show sase-h8.10.5` and each of its three children. Confirm all
   descendants remain closed and reread every note. Build a close note that records: the
   three epic commits and plans-sidecar repair; current source verification; all
   post-start commits through the final base refresh; the write-target integration fix;
   focused, contention, full, visual, wait-checker, and flake-gate evidence; the
   `sase-ct`/`sase-h8` duplicate outcome; the non-recurring page-publication proposal;
   and ready task `sase-hu`.
2. Close normally, without force:

   ```bash
   sase bead close sase-h8.10.5 --note "<comprehensive verification and integration summary>"
   ```

   If descendant validation rejects the close, investigate and finish or deliberately
   reopen the named work. Never force merely to make the command succeed.

3. After the close succeeds, run `just symvision`. If it reports an expired
   `sase-h8.10.5` whitelist entry or unused code, read `symvision.md` through
   `/sase_memory_read`, remove/fix exactly what it reports, and rerun Symvision. Run
   `just check` for any resulting main-repository edit (use `just check-full` if the
   broadening set is touched).
4. Use the already-opened plans sidecar and change only the frontmatter of
   `202608/h8_10_remaining_landing.md` from `status: wip` to `status: done`. This edit
   is last, after the epic is closed and post-close Symvision is green. Validate the
   plan and inspect both repository diffs/statuses.

## Acceptance criteria

- `sase.ace.tui` and every XPrompt write-target consumer import successfully on the
  integrated current base; there is one coherent write-target type boundary.
- Lazy memory XPrompt loading and ref-XPrompt artifact rendering remain intact.
- Focused integration tests, the wait checker, `just check-full`, the flake baseline,
  and `just test-visual` are green; the deterministic contract budget retains its fresh
  six-repeat zero-failure evidence unless a new relevant change requires rerunning it.
- Every proposed follow-up has the recorded outcome above, with no duplicate task.
- `sase-h8.10.5` closes with resolution `done` and no force, post-close Symvision is
  green, and `plans:202608/h8_10_remaining_landing.md` has `status: done`.
