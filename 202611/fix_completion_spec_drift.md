---
tier: tale
title: Fix Master Gate completion spec snapshot drift
goal:
  Master Gate is green again and CLI flag changes can no longer silently drift the
  completion spec snapshot.
size: small
decisions:
  autosync:
    ask:
      Should `just fix` regenerate the completion snapshot and `just lint` check it to
      stop recurring drift?
    choices:
      fix_and_lint:
        Also sync it in `just fix` and check it in `just lint`, like the Footprint
        catalog (~1s per run)
      snapshot_only:
        Only regenerate `cli_spec.json`; drift stays guarded solely by the sharded test
        suite
    default: fix_and_lint
    why:
      "`just fix` already regenerates the argparse-derived CLI docs; only this snapshot
      is skipped"
    answer: fix_and_lint
proposed_by: bbugyi200.athena.1g8.w0.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 20:39:55
status: wip
---

# Fix Master Gate: CLI completion spec snapshot drift

## Problem

`actstat` shows `sase-org/sase` Master Gate run #3785 (commit `1a468da521`,
"feat(memory): add audited roster web read with bare-web guard") failing in shard
`test (6)` at "Run sharded fast suite". The other seven shards, lint, docs, and
core-wheel passed. The two failing tests are:

- `tests/completion/test_snapshot.py::test_checked_in_snapshot_has_no_drift`:
  `AssertionError: sase CLI completion spec is out of sync with the argparse tree`
- `tests/completion/test_snapshot.py::test_current_structural_view_matches_checked_in_snapshot`

Both reproduce locally at the current `master` tip (`8e6818d0a6`) with
`.venv/bin/python -m pytest tests/completion/test_snapshot.py -q`: 2 failed, 2 passed.
The two later commits (`47cec6ea49`, `8e6818d0a6`) still carry the drift, so their
Master Gate runs will fail the same way until this lands.

## Root cause

Commit `1a468da521` added a new `-a/--all-strands` flag to both `sase memory read` and
`sase memory show` in `src/sase/main/parser_memory.py`, and reworded their help text. It
did not regenerate the checked-in structural completion snapshot,
`tests/completion/snapshots/cli_spec.json`. A structural diff of the snapshot against
`current_structural_view()` shows exactly two drifted nodes:

- `root/memory/read`: new option `all_strands`, and `description_digest`
  `4034fd54741931a0` changed to `cbf3c50f1b9e8cdd`
- `root/memory/show`: new option `all_strands`, and `description_digest`
  `4f38a03aeb95eb1a` changed to `f70e6c5aa135f741`

Nothing else differs.

The process gap: the same commit correctly updated `docs/reference/cli/memory.md`, which
is also generated from the argparse tree, because `just fix` (the commit-dispatch fix
hook) runs `docs-gen`. The completion snapshot is regenerated only by the manual
`just sync-completion-spec` recipe. Neither `just fix` nor `just lint` runs it, so the
only guard is a unit test in the sharded suite. The diff-scoped `just check` lane only
sometimes runs that test. This has happened before: `2e04694766` "sync completion
snapshot", plus the `caa7917ac` / `6f5df19d6` resyncs noted in
`tests/reproducible_flake_baseline.txt`. The Footprint catalog already uses the fix
pattern this plan copies: `sync-footprint-catalog` runs in `just fix`, and
`sync-footprint-catalog-check` runs in `just lint`.

Governing decisions: `decisions:check-full-is-explicit` (verify with `just check`, not
`just check-full`) and `decisions:ci-two-speed-split` (the Master Gate's per-SHA lint
and eight shards are the signal this repairs).

## Changes

### 1. Regenerate the snapshot (both choices)

- Run `just sync-completion-spec`, which rewrites
  `tests/completion/snapshots/cli_spec.json` from the live argparse tree.
- Check that `git diff --stat` touches only that file and that the JSON diff covers only
  the `memory read` / `memory show` nodes listed above. If other nodes change, stop and
  find out why before committing; that would mean a second, unexplained CLI change.

> [!decision] autosync = fix_and_lint
>
> Do steps 2 and 3 below as well.

> [!decision] autosync = snapshot_only
>
> Skip steps 2 and 3. Step 1 plus verification is the whole change.

### 2. Justfile wiring (`autosync = fix_and_lint`)

Follow the existing Footprint recipes (`sync-footprint-catalog` /
`sync-footprint-catalog-check`):

- Change the `sync-completion-spec` recipe's dependency from `_setup` to
  `_setup-app-import`. `just fix` deliberately avoids `_setup` because it may fetch
  sase-core and rebuild `sase_core_rs`. `_setup-app-import` probes the venv and falls
  back to `_setup` only when it is not provisioned. Building the argparse tree needs
  only an importable `sase`, as `docs-gen` does.
- Add a sibling recipe next to it:

  ```just
  # Check the completion spec snapshot matches the argparse tree (CI mode, no writes).
  sync-completion-spec-check: _setup-app-import
      {{ venv_bin }}/python tools/sync_completion_spec --check
  ```

  `tools/sync_completion_spec --check` already exists. It exits 1 with the drift
  message, which names `just sync-completion-spec`.

- Append `sync-completion-spec` to the `fix:` dependency list, after
  `sync-footprint-catalog`.
- In the `lint:` recipe, add a stage right after the Footprint stage:

  ```just
      @printf "\n---------- Checking completion spec snapshot is current... ----------\n"
      @just sync-completion-spec-check
  ```

  Also add "completion spec" to the `lint` recipe's descriptive comment.

- Update the `_setup-app-import` comment, which currently says "`just fix` runs docs-gen
  and sync-footprint-catalog on every commit", so it also names `sync-completion-spec`.
- Do not add the stage to `check` or `check-full`. That matches the Footprint check,
  which is also lint-only. `just fix` now regenerates the snapshot before commit
  dispatch, and CI's `just lint` is the backstop.

### 3. Justfile contract tests (`autosync = fix_and_lint`)

- `tests/test_justfile_lint_setup.py`: add `"sync-completion-spec"` and
  `"sync-completion-spec-check"` to `_SASE_BACKED_FIX_RECIPES`. This makes
  `test_sase_backed_fix_recipes_probe_then_delegate_to_setup` enforce the
  `_setup-app-import` probe for both recipes.
- `tests/test_justfile_lint_lint.py::test_fix_routes_generated_docs_to_their_declared_venvs`:
  the spy `.venv/bin/python` exits 91 (`forbidden-app-python`) for any unlisted script.
  Add an allow line for `tools/sync_completion_spec`, matching the
  `tools/sync_footprint_catalog` line. Then assert
  `"sase-python tools/sync_completion_spec --write" in recorded`.
- `tests/test_justfile_lint_lint.py`: add `test_lint_includes_completion_spec_stage`,
  modeled on `test_lint_includes_toobig_stage`. It asserts that the `dry_run("lint")`
  output contains `just sync-completion-spec-check`.

No source-code, docs, or memory changes are needed. `cli_rules.md` does not mention the
snapshot, and the contributor docs summarize `just fix` only loosely and do not list the
Footprint sync either.

## Verification

1. `.venv/bin/python tools/sync_completion_spec --check` exits 0.
2. `.venv/bin/python -m pytest tests/completion -q` passes, including both
   `test_snapshot.py` drift tests.
3. With `fix_and_lint`:
   `.venv/bin/python -m pytest tests/test_justfile_lint.py tests/test_justfile_lint_check.py tests/test_justfile_lint_lint.py tests/test_justfile_lint_setup.py -q`
   passes. Also, `just sync-completion-spec-check` exits 0, and running `just fix` on
   the clean tree leaves `cli_spec.json` unchanged.
4. Run `just fix`, then `just check`, as the `lint_and_test.md` memory note prescribes.
   `just check` is a guarded recipe, so run it through `sase tool run`. Do not run
   `just check-full`.
5. After landing, use `actstat` / `gh run list --repo sase-org/sase` to confirm the new
   tip's Master Gate goes green. The in-flight runs for `47cec6ea49` and `8e6818d0a6`
   predate the fix and are expected to fail `test (*)` on these same two tests. If a
   Master Gate shard fails for any other reason, diagnose it. Fix it in this change when
   it is small and clearly caused by those two commits. Otherwise file it with
   `/sase_new_task`.

## Out of scope

- Re-architecting the diff-scoped test selector. Its stale coverage baseline (about 5.5k
  commits behind HEAD) is a separate concern.
- Unrelated `actstat` failures in other repos (dotfiles, bob-cli, sase-core,
  sase-github, bobs-org.github.io). The request covers only the sase repo.
