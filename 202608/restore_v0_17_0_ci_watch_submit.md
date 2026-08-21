---
tier: tale
title:
  Restore green default-branch Publish and core-floor smoke so ci_watch can submit PR
  284
goal:
  Master Publish sync-release-metadata and release PR 284 release-core-floor-smoke are
  both green after ratcheting sase-core-rs to 0.29.9, without anyone merging PR 284 by
  hand, so the configured ci_watch chop can submit the v0.17.0 release.
size: medium
proposed_by: bbugyi200.athena.sase-ry.2--2
bead: sase-ry.2
create_time: 2026-08-21 22:54:14
status: wip
---

- **BEAD:**
  [sase-ry.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ry/sase-ry.2.md)

# Restore green Publish and core-floor smoke for v0.17.0

Work this as a child of `plan:202608/release_v0_17_0.md` (phase
`await_ci_watch_submission` / bead `sase-ry.2`). SASE writes the canonical PARENT link
on archival; do not hand-author PARENT bullets.

## Outcome and boundaries

Unblock `ci_watch` from submitting `sase-org/sase#284` (`chore(master): release 0.17.0`)
by restoring the two gates it currently fails closed on:

1. Default branch `master` is red, so the chop records `#284` as
   `base branch not green`.
2. The release PR itself is `UNSTABLE` because `release-core-floor-smoke` failed.

Do **not** invoke `gh pr merge` or otherwise bypass `ci_watch`. Do **not** reopen or
compete with `plan:202608/simplify_ci_watch.md` — that tale already landed: AXE
restarted at 2026-08-21T20:50Z, `inhibit_if.agent_runners.max=0` is gone, and `ci_watch`
has been running the chop every 5 minutes (`reason=no_actions` then a live skip of
`#284` as `base branch not green`). Waiting another chop interval will not merge a red
default branch or a red release PR.

Stay in the sase Python repo unless a reproduced ratchet failure proves the lock refresh
needs a sibling checkout. Published `sase-core-rs==0.29.9` already exports the two
missing bindings; do not wait on `sase-org/sase-core#156` (`chore: release v0.29.10`,
currently UNSTABLE) and do not change sase-core for this tale.

After the gates are green, this phase still owns the wait for `ci_watch` to submit.
Start a new `/sase_monitor` with `WAITING FOR SUBMIT` / `SUBMITTED`; never merge.

## Evidence at planning time

Recorded 2026-08-21T22:33Z–22:35Z by `sase-ry.2`.

- PR 284 remains OPEN, MERGEABLE, UNSTABLE at head
  `e4f917901734b2d79cb53d6d0c8fd2b6a8f539af`. Required rollup: Conventional PR title
  SUCCESS; source lanes SKIPPED; `release-core-floor-smoke` FAILURE in run `32532918668`
  / job `96928367962` at 2026-08-21T22:26:12Z.
- Smoke log:
  `sase_core_rs 0.29.6 is missing 2 of 368 required binding(s): authenticate_finalizer_plan, validate_finalizer_plan`.
  Those `require_rust_binding` sites landed on master in
  `6639a28016163be274ace52c293bd7aeebfb8470`
  (`feat(finalizers): enforce bounded attempts and immutable evidence`) via
  `src/sase/core/finalizer_facade.py`.
- Local install against CPython 3.12: `sase-core-rs==0.29.6` has neither binding;
  `sase-core-rs==0.29.9` (current PyPI latest) has both. `pyproject.toml` still declares
  `sase-core-rs>=0.29.6,<0.30.0`.
- Default-branch evidence `ci_watch` is using: `sase-org/sase` red at
  `master@6639a2801616`, `Publish › sync-release-metadata — failure` (run `32532695440`
  / job `96928366087`). The failing step is
  `python tools/ratchet_core_window --allow-transitive-lock-refresh`, exit 3:
  `uv.lock package asttokens changed fields outside version, sdist, or wheels`.
- Live `ci_watch.report.json` (mtime 2026-08-21T22:33:12Z):
  `release_candidates=1 merged=0`, `#284` decision `base branch not green`. AXE HEALTHY;
  lumberjack `ci_watch` PID 487913, interval 5m, cycles=21, errors=0.
  `sase-org/sase-telegram#19` is MERGED; `sase-org/sase-github` has no open release PR.
- This is the same class of defect the floor smoke exists to catch (shipping Python that
  calls a binding the published floor lacks). The Publish job is already supposed to
  ratchet the floor to the newest complete `sase-core-rs` release onto
  `release-please--branches--master`; it cannot, because the transitive lock validator
  rejects the asttokens field diff.

## Diagnose the asttokens field diff first

Work from current `origin/master` plus the live release-please branch, not an older
workspace tree.

1. Check out `release-please--branches--master` (read-only first) and run
   `python tools/ratchet_core_window --allow-transitive-lock-refresh` the same way
   `.github/workflows/publish.yml` `sync-release-metadata` does.
2. Capture the before/after `[[package]]` stanza for `asttokens`.
   `_validate_transitive_lock_refresh` in `tools/ratchet_core_window` only treats
   `version`, `sdist`, and `wheels` as refreshable; every other top-level key must be
   identical. Dump the key-set and value diff (likely a new key such as `dependencies` /
   `optional-dependencies`, or a `source` change). Do not guess — pin the failing keys
   in the test fixture.
3. Confirm a successful ratchet would write `sase-core-rs>=0.29.9,<0.30.0` and refresh
   the `sase-core-rs` lock artifacts. If PyPI has moved past 0.29.9 by implementation
   time, ratchet to the newest complete release that still exports
   `authenticate_finalizer_plan` and `validate_finalizer_plan`.

## Fix the ratchet so Publish can apply 0.29.9

Keep the fail-closed posture: unrelated **direct** dependency edits stay refused;
default mode without `--allow-transitive-lock-refresh` still refuses any transitive
package change.

Extend `--allow-transitive-lock-refresh` just enough for the observed asttokens field
diff:

- Prefer treating additional **benign lock-format keys** as refreshable (same family as
  `version` / `sdist` / `wheels`) when they are lock metadata uv rewrites while bumping
  `sase-core-rs`, not a change to a direct `project.dependencies` package.
- Do not silently accept arbitrary package-set or package-order changes; keep those as
  `EXIT_COULD_NOT_DETERMINE`.
- Do not weaken core-package validation (`sase-core-rs` may still change only
  version/sdist/wheels and must land on the target version).
- Do not “fix” this by editing generated release changelog/version files, and do not
  lower or remove the `release-core-floor-smoke` gate.

Update `tests/test_ratchet_core_window_tool.py`. That file already uses an `asttokens`
fixture and covers:

- default mode rejects transitive refresh
  (`uv.lock changed unrelated package asttokens`);
- `--allow-transitive-lock-refresh` allows version/sdist/wheels refresh
  (`asttokens 3.0.0 -> 3.0.1`).

Add a regression that applies the **same extra field change seen in the live failure**
and expects exit 2 (ratchet applied) with `--allow-transitive-lock-refresh`, and still
expects exit 3 without the flag. Keep `tests/test_github_actions_ci.py` asserting
Publish still invokes `python tools/ratchet_core_window --allow-transitive-lock-refresh`
(not `--report-only`, and not a merge).

If reproducing the live `uv lock` also requires a committed `uv.lock` refresh of
asttokens on master so the next Publish run is a no-op on that package, include that
lock refresh in the same change and explain why it is necessary rather than only a
validator widening.

## Verify the floor smoke against 0.29.9

After the ratchet can apply:

- `pyproject.toml` must require a floor that exposes both bindings (0.29.9 at planning
  time).
- In a 3.12 venv, `uv pip install sase-core-rs==<floor>` then
  `tools/check_sase_core_rs_bindings` and `tools/validate_sase_core_rs` must pass on the
  tree that contains `src/sase/core/finalizer_facade.py`.
- Run `just install` then `just check` on the sase change. Use monitored
  `just check-full` only if scoped selection escalates or the change hits the broadening
  set.

Land through the normal SASE patch/commit workflow. The landing push to master should
retrigger Publish `sync-release-metadata`, which should now succeed, commit
`chore: sync release metadata` onto `release-please--branches--master` if needed, and
leave `#284` with the ratcheted floor.

## After green, wait — do not merge

Re-query:

- `gh pr checks 284` — `release-core-floor-smoke` SUCCESS (or a newer green run on a
  newer head); source lanes may still SKIP.
- `gh run list --branch master` for Publish on the landing SHA — `sync-release-metadata`
  SUCCESS so `ci_watch` no longer classifies the default branch as red.
- `sase axe status` and `~/.sase/axe/lumberjacks/ci_watch/ci_watch.report.json` — `#284`
  decision is no longer `base branch not green`.

Then start `/sase_monitor` with `WAITING FOR SUBMIT` / `SUBMITTED` until GitHub reports
PR 284 `MERGED`. The `--next` prompt must keep the predecessor-plan rule, forbid
`gh pr merge`, and tell the successor to distinguish an ordinary 5-minute chop /
`max_merges_per_tick=1` delay from a new fault.

`sase-org/sase-core#156` may still be open and red; at planning time `ci_watch` listed
`#284` as the only pending sase release candidate. If a later tick skips sase because
merge_order prefers a now-green core/github/telegram release PR, that is an ordinary
one-tick delay, not a bypass.

## Notes and non-goals

- Append a concise evidence note to `sase-m4` when the Publish and smoke conclusions
  change (this is GitHub Actions stabilization evidence for the v0.17.0 release
  attempt). Do not close `sase-m4`.
- Record progress on `sase-ry.2` only. Do not close `sase-ry`, `sase-ry.3`, or any
  ancestor.
- Do not create task beads; use `PROPOSED FOLLOW-UP:` on `sase-ry.2` for anything
  outside this tale (for example sase-core `#156` cargo-test red, or sase-telegram
  default-branch `CI › check (3.12)` red).
- `sase-ru.6--mon` still WAITING RELEASE for the same v0.17.0; that is no longer an
  inhibit (the guard is gone) and is not this tale's job to stop.
