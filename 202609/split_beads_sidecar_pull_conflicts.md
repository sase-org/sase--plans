---
tier: tale
title: Let sidecar pulls resolve bead conflicts in the split-beads clone
goal:
  A strict sidecar materialization (for example, a TUI-approved epic launch in a leased
  workspace) semantically resolves bead conflicts in the split-beads sidecar clone
  instead of misclassifying them as non-bead conflicts and failing, and its errors no
  longer blame Git credentials for integration conflicts.
size: small
proposed_by: bbugyi200.athena.0pd
create_time: 2026-09-22 12:30:18
status: wip
---

# Plan: Let sidecar pulls resolve bead conflicts in the split-beads clone

## Background: what failed

On 2026-09-22 the user approved the epic plan `202609/self_healing_workspace_prep.md`
from the TUI, and the epic launch failed with this error:

```
Epic launch failed: could not resolve the SDD and bead stores: could not materialize
beads sidecar repository sase-org/sase--beads ... at <ws>/sase/repos/beads: git rebase
failed: ... could not apply 95916d6ef... chore(beads): update sase-16d ...;
non-bead conflicts remain: events/streams/sase-16d.jsonl, issues.jsonl. Verify that the
repository exists and your Git credentials can read it.
```

Reconstructed sequence (from the notification, `~/.sase/bead_push_logs/`, and
`~/.sase/logs/tui_git_ops.jsonl`):

1. An earlier agent in the same numbered workspace wrote a local bead commit
   (`update sase-16d`) in the workspace's split-beads sidecar clone
   (`sase/repos/beads`). Its detached bead sync worker rebased that commit and resolved
   the conflict semantically, but it never pushed. The log has no `completed` or
   `failed` record and no traceback. The agent then finished, and the workspace went
   back to the pool with the commit still unpublished.
2. Another clone then pushed an `update artifact links` commit that also touched
   `events/streams/sase-16d.jsonl` and `issues.jsonl`.
3. TUI approval calls `start_epic_launch_monitor`. It leases a pooled workspace through
   `acquire_operational_lease`, and it got the same workspace.
   `prepare_from_primary_remote` resets only the main checkout, not the sidecars.
4. `sase bead work` → `_resolve_plan_file_context` → `ensure_beads_sidecar_clone` →
   `ensure_sidecar_sdd_clone(strict=True)` → `pull_sdd_clone` →
   `integrate_machine_managed_sdd_repository`. The rebase conflicted on the two bead
   files. The semantic resolver classified both as **non-bead** and aborted. Because the
   call was strict, the epic launch failed.
5. Seconds later, a bead sync worker in the same clone hit the identical conflict,
   resolved it semantically, and pushed. So the conflict was always resolvable. Only the
   materialization path could not see that it was a bead conflict.

## Root cause

`src/sase/sdd/_store_integration.py::pull_sdd_clone` hardcodes the bead-store location
as a `beads/` subdirectory of the clone:

- `integrate_machine_managed_sdd_repository(workspace_sdd, beads_dir=(workspace_sdd / "beads"), ...)`
- `_has_unpushed_bead_commits` →
  `unpushed_bead_commit_count(workspace_sdd, workspace_sdd / "beads")`

That is correct for the combined plans sidecar (`sase/repos/plans/beads/`). It is wrong
for the split-beads sidecar (`sase/repos/beads/`), where the bead store is the clone
root (`config.json`, `events/`, `issues.jsonl` at top level). In that layout:

- `resolve_beads_dir(root, root / "beads")` (`src/sase/bead/conflict_resolver_paths.py`)
  returns `None`, because `beads/` is a canonical relpath but is not a directory.
  `_resolve_semantic_conflicts` then gets `bead_prefix=None`, so every conflicted path
  is "unclaimed" and the result is `non-bead conflicts remain: ...`.
  `_repair_or_abort_rebase` then aborts with `ABORTED_UNSUPPORTED_CONFLICTS`, and
  `pull_sdd_clone(strict=True)` raises `SddMaterializationError`. Reproduced:
  `resolve_beads_dir(clone, clone / "beads")` returns `None`, while
  `resolve_beads_dir(clone)` and `resolve_beads_dir(clone, clone)` both return the clone
  root.
- `_has_unpushed_bead_commits` counts `@{upstream}..HEAD -- beads/`, which is always 0
  in a split-beads clone. The guard that keeps unpublished bead history out of the
  failed-integration cooldown therefore never fires for that clone. After one failure, a
  later non-fresh pull within the cooldown is silently suppressed. `pull_sdd_clone` then
  returns `False` without raising, even in strict mode, and the clone is left
  unintegrated.
- The bead sync worker is not affected. `push_bead_work_launch_async` passes a correct
  beads dir via `semantic_beads_dir_for_sync`.

Existing tests cover `_pull_sdd_clone` only with the combined layout
(`tests/sdd_store/test_sidecar_clone_pull_refresh.py` builds `plans/beads/...`). No test
exercises a split-beads clone.

The other hardcoded `/ "beads"` integration callers are correct for their layouts, so
leave them alone:

- `_store_adoption.py::_push_sidecar_store` bootstraps a combined `beads/` store.
- `_store_materialization.py` refreshes a separate-repo `.sase/sdd/beads`.

## Changes

### 1. Resolve the clone's real bead store in `pull_sdd_clone`

In `src/sase/sdd/_store_integration.py`:

- Compute the bead store once per call from the clone's actual layout. Reuse the
  existing auto-detection in
  `sase.bead.conflict_resolver_paths.resolve_beads_dir(clone_root)` (called with no
  explicit `beads_dir`). It already returns:
  - `<clone>/sdd/beads` or `<clone>/beads` for nested stores
  - the clone root when `config.json` is at the top level
  - `None` when there is no store or the layout is ambiguous

  If you would rather have a tiny named helper (for example,
  `sidecar_clone_bead_store_dir(clone_root) -> Path | None`), build it on
  `resolve_beads_dir` or on `sase.sdd._bead_state.has_bead_state`. Check the clone root
  first, then `beads/`. Do not add a third probe.

- Pass that value as `beads_dir=` to `integrate_machine_managed_sdd_repository`. `None`
  is acceptable. It means "no bead store here" for plain sidecars such as research, and
  the resolver chain auto-detects.
- Use the same value in `_has_unpushed_bead_commits`:
  - If there is no bead store, return `False`: nothing bead-shaped can be parked.
  - Keep the existing fail-open `True` on exceptions.
  - With the clone root as the store, `unpushed_bead_commit_count(root, root)` must
    count commits that touch any path. `relative_pathspec`
    (`src/sase/bead/_sync_git.py`) yields `.`, so the pathspec becomes `./`, which git
    accepts. Lock that in with the cooldown test below.
- Resolve with the clone's own layout on every call. Do not cache per kind. A plans
  clone can be schema-2 (combined) or schema-3 (split, no beads in plans).

### 2. Don't blame credentials for an integration conflict

In `src/sase/sdd/_store_workspace.py::ensure_beads_sidecar_clone`, the wrapper always
appends "Verify that the repository exists and your Git credentials can read it." That
misled this diagnosis: the failure was a rebase conflict, not access.

- Append that hint only for clone/fetch/access failures. Do not append it when the clone
  exists and its integration (pull/rebase) failed.
- Suggested mechanism:
  - Add a narrow `SddMaterializationError` subclass (for example, `SddIntegrationError`)
    in `src/sase/sdd/_store_types.py`.
  - Raise it from the strict branch at the end of `pull_sdd_clone`.
  - Omit the hint for it, and also for `SddRepositoryHealthError`, in the wrapper.
- `ensure_sidecar_sdd_clone` re-raises `SddMaterializationError` subclasses unchanged.
  Confirm the subclass survives that path.
- Keep the rest of the message intact.

### 3. Tests (real git; never touch the real `~/.sase`)

Add them to `tests/sdd_store/test_sidecar_clone_pull_refresh.py`, or to a sibling file
if that one would grow past the repo's size limits. Reuse the helpers in
`tests/sdd_store/_helpers.py`. For real bead stores, reuse
`tests/test_bead/sync_conflict_regression_helpers.py::_seed_claim_soak_remote(..., beads_dirname=BEADS_DIRNAME_ROOT)`
or `BeadProject.init(seed, beads_dirname=".")`.

1. **Incident regression (split-beads layout).**
   - Setup:
     - Seed a bare remote whose bead store is at the repo root.
     - Clone it twice: a "workspace sidecar" clone and a "peer" clone.
     - In the workspace clone, commit a local change to bead X, for example an update or
       note through `BeadProject`, so both its stream and `issues.jsonl` change. Do not
       push it.
     - In the peer clone, commit and push a different change to the same bead X.
   - Assert:
     - `ensure_sidecar_sdd_clone(workspace_clone, remote_url, strict=True, fresh=True)`
       succeeds.
     - HEAD contains upstream and has the local commit rebased on top.
     - X's stream holds both events.
     - No rebase is in progress.
   - This test must fail on current code with `non-bead conflicts remain`.
2. **Combined layout unchanged.**
   - The same conflict shape with the store under `beads/` in a plans-style clone still
     integrates.
   - The existing `test_failed_integration_cooldown_does_not_park_unpushed_bead_commits`
     keeps passing.
3. **Cooldown guard, split layout.** Mirror
   `test_failed_integration_cooldown_does_not_park_unpushed_bead_commits` with the store
   at the clone root. An unpushed root-layout bead commit must bypass the
   failed-integration cooldown, so `integrate_machine_managed_sdd_repository` is called.
4. **Non-bead sidecar.**
   - A research-style clone with no bead store passes `beads_dir=None`.
   - `_has_unpushed_bead_commits` is `False` there.
   - A genuine non-bead conflict still aborts with `non-bead conflicts remain`.
5. **Error text.** A strict pull that fails integration surfaces through
   `ensure_beads_sidecar_clone` without the credentials hint. A clone failure (bad
   remote URL) still includes it.

### 4. Verification

- Run `just check` via `sase tool run check`, per the `lint_and_test` reference memory.
  Do not run `just check-full`.
- Also run
  `pytest tests/sdd_store tests/test_bead -k "pull or sidecar or cooldown or conflict"`
  explicitly.

## Relationship to the sase-16e epic (Self-healing agent workspace preparation)

sase-16e does **not** fix this failure, and this plan does not overlap its phases:

- Its rescue-store, checkout-heal, and reclone phases harden `prepare_workspace` for
  **agent launches** (opt-in for the launch and retry callers only). The epic launch
  never goes through `prepare_workspace`. It uses `acquire_operational_lease`.
- Its core-merge and bead-sync phases fix the Rust event-stream merge order and
  sync-worker rollback. Here the conflict was rejected before any merge ran, because the
  paths were misclassified.
- Its visibility phase concerns the TUI agent-row error.

Stay out of the files that sase-16e phases are rewriting: `runner_workspace_prepare.py`,
`runner_workspace_beads.py`, `sync_worker.py`, `_stream_integrity*.py`, and the rescue
store. If a rebase conflict arises anyway, resolve it by keeping both sides.

## Non-goals / follow-ups

- **Unmergeable leftover sidecar state in a leased workspace.** Example: a stream wedged
  by the old non-append-only merge. This will still fail an epic launch after this fix.
  The right fix is to opt `acquire_operational_lease` into sase-16e's rescue/heal ladder
  once it lands. File that as a follow-up task bead through `/sase_new_task` and relate
  it to sase-16e. Do not implement it here.
- **The sync worker that stopped after integrating and before pushing (step 1).** It
  left no traceback, so it was probably killed or hung. The available logs are not
  enough to plan a fix. If the implementer finds a concrete cause while testing, file it
  through `/sase_new_task` rather than widening this plan.
- No feature flag. This is a bug fix to existing behavior.
