---
tier: tale
title: Unblock release metadata reconciliation
goal:
  The Publish workflow reconciles release-please metadata without failing on benign
  transitive lockfile drift, allowing ci_watch to merge release PRs only after its
  normal green-branch guard passes.
size: medium
proposed_by: bbugyi200.athena.0a6--1
create_time: 2026-08-21 21:14:01
status: wip
---

# Plan: Unblock release metadata reconciliation for release-please PRs

## Problem

`ci_watch` did not submit `sase-org/sase#284` during the monitored rollout window. This
was the correct fail-closed behavior, not a `ci_watch` launch or merge bug.

Evidence to preserve:

- `ci_watch` live run `20260821T205019_517899` merged `sase-org/sase-telegram#19`, but
  recorded `sase-org/sase#284` as `release_reason=default_branch_not_green`.
- Later live runs `20260821T210027_615835` and `20260821T210537_321114` kept
  `sase-org/sase#284` at `release_reason=default_branch_not_green`.
- GitHub `Publish` run `32525172639` on
  `master@f929b5e2c803431399cf0372ab8d50ced906ef6c` failed in job
  `sync-release-metadata`, step `Reconcile release metadata`.
- The failing log line was
  `[ratchet_core_window] uv.lock changed unrelated package asttokens`, followed by exit
  code 3.
- `ci_watch` had `proposed_launches: []`, no `LaunchApproval` notification, and no
  launch interaction request in this window. Do not reintroduce repair-agent behavior.

The required fix is to make the `Publish` workflow's release metadata reconciliation
safe and deterministic when the `sase-core-rs` ratchet lock refresh also produces benign
resolver-owned drift such as the observed `asttokens` lockfile movement. Do not loosen
`ci_watch`'s default-branch-green release guard.

## Implementation

1. Reproduce the failure path locally from the current `tools/ratchet_core_window`
   contract before changing it. Use a fixture or the release branch state to show that a
   valid `sase-core-rs` ratchet can fail only because `uv.lock` also changes an
   unrelated transitive package.

2. Update `tools/ratchet_core_window` so the default ratchet path remains strict, but
   the release metadata workflow can explicitly opt into a bounded reconciliation mode.
   That mode should:
   - still refuse to lower the `sase-core-rs` floor;
   - still validate the expected `pyproject.toml` and `sase-core-rs` lock metadata;
   - still fail on direct dependency changes, package-set changes, malformed lockfiles,
     or resolver output that cannot be attributed to a normal lock refresh;
   - allow unrelated transitive package metadata/version/hash movement only when the
     caller has opted in;
   - print a concise summary of any allowed unrelated packages so CI logs explain why
     the workflow continued.

3. Wire `.github/workflows/publish.yml` to use that explicit reconciliation mode in the
   `sync-release-metadata` job before the final `uv lock` and branch push. Keep the
   workflow's existing behavior of committing only `pyproject.toml` and `uv.lock` to
   `release-please--branches--master`.

4. Update tests:
   - add `tests/test_ratchet_core_window_tool.py` coverage proving default mode still
     rejects unrelated package drift and restores files;
   - add coverage proving the explicit reconciliation mode accepts a representative
     unrelated transitive package drift while still rejecting direct dependency and
     package-set changes;
   - update `tests/test_github_actions_ci.py` so the publish workflow test asserts the
     explicit mode is used by `sync-release-metadata`;
   - keep existing PyPI failure, downgrade refusal, and report-only behavior intact.

5. Run `just install` first if the workspace environment is stale, then run targeted
   tests for the ratchet tool and publish workflow. Finish with `just check`.

## Rollout Verification

After the code lands on `master`, verify the next `Publish` run for `sase-org/sase`
succeeds through `sync-release-metadata` and updates the release-please branch when
needed. Then let `ci_watch` submit `sase-org/sase#284` only after its normal guards see:

- the default branch green;
- the release generator idle;
- the PR non-draft, mergeable, clean, and fully settled;
- the final head reread matching the merge command's `--match-head-commit`.

Report the successful `Publish` run ID, the `ci_watch` run ID that submits
`sase-org/sase#284`, the PR URL, and the release notification id. If another guard
blocks the release, record that exact guard instead of manually merging the PR.
