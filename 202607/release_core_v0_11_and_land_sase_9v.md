---
tier: tale
title: Release sase-core v0.11.0, bump the sase pin, and land epic sase-9v
goal: 'The published sase-core-rs release and the sase dependency window both contain
  the sase-9v.9 bead-mutation atomicity fix, so published/locked installs get the
  same concurrency guarantees as dev installs, and epic sase-9v is closed with its
  plan file marked done.

  '
bead: sase-9v
parent: sase/repos/plans/202607/bead_review_hardening.md
create_time: 2026-07-26 13:37:34
status: wip
---

- **PROMPT:** [202607/prompts/release_core_v0_11_and_land_sase_9v.md](prompts/release_core_v0_11_and_land_sase_9v.md)

# Plan: Release sase-core v0.11.0, bump the sase pin, and land epic sase-9v

## Context

Epic sase-9v ("Harden the bead subsystem against the verified gaps from the post-sase-9r/9s review", plan
`202607/bead_review_hardening.md`) is code-complete. All 11 phase beads are closed, and land-agent verification
confirmed every phase bullet of the epic plan against sase HEAD `4f65c6bf5` and sase-core HEAD `5df18bb` — including
phase 9 (`core_mutation_atomicity`): all eleven bead-store mutations now run under `with_bead_mutation_lock`, JSONL
writes use per-process unique temp paths, the work planner treats dangling out-of-epic blockers as satisfied with a
deterministic warning, and `OperationOutcomeWire` is removed.

What remains is delivery of the core fix to non-dev installs:

- sase-core master commit `5df18bb` (`fix(beads)!: make store mutations atomic (sase-9v.9)`) is **unreleased**. The
  latest PyPI `sase-core-rs` is 0.10.0, and `git tag --contains 5df18bb` is empty (v0.10.0 predates the commit).
- The commit carries a `BREAKING CHANGE` footer (`OperationOutcomeWire` is no longer exported), so the next release is
  **v0.11.0**. sase pins `sase-core-rs>=0.10.0,<0.11.0` in pyproject.toml, and `uv.lock` resolves registry 0.10.0 — so
  published-wheel and lock-driven environments will keep running a core **without** the atomicity hardening this epic
  exists to deliver.
- Unlike the sase-9t release cycle, nothing is functionally broken on 0.10.0: sase consumes no new core API from
  sase-9v.9 (removing `OperationOutcomeWire` was possible precisely because nothing consumed it), and the focused bead
  suites pass either way. This is a behavior-only delivery gap, not a compatibility break — but it means the epic's
  stated goal ("sase-core bead mutations are atomic against each other") is only true in dev installs today.
- Dev workspaces already run the fix because `just install` builds `sase_core_rs` editable from the local sase-core
  checkout.

Release automation facts (same machinery as the sase-9t v0.10.0 cycle):

- sase-core releases are fully automated by release-plz (`.github/workflows/release-plz.yml`). Merging conventional
  commits to master maintains an open release PR; merging that PR tags `v<version>`, creates the GitHub release, builds
  the wheel matrix, and publishes to PyPI. Publishing is self-healing: every push and a 6-hourly scheduled run re-check
  "tagged workspace version missing from PyPI" and finish an interrupted publish, so retries are safe. Do not hand-edit
  versions in sase-core (`pr-title.yml` blocks manual version edits).
- Release PR **sase-org/sase-core#32** ("chore: release v0.11.0", branch `release-plz-2026-07-26T16-22-03Z`, refreshed
  2026-07-26 16:22Z — after `5df18bb` landed at 16:08Z) is open, and its changelog correctly lists
  `*(beads)* [breaking] make store mutations atomic (sase-9v.9)` under `sase_core` 0.11.0. It needs no refresh unless
  sase-core master gains new commits before the merge.
- The most recent release-plz runs are green (the one earlier failure was GitHub API rate limiting, since recovered).

Repo access rule: open sase-core with the `/sase_repo` skill and use only the path it prints. GitHub PR/issue/workflow
state may be inspected with `gh`.

**Non-goal:** the deferred follow-up recorded in sase-9v.1's bead notes — closing a single store-write-lock span across
`commit_epic_graph_checkpoint`'s epic-creation mutations and checkpoint commit in `src/sase/bead/cli_work_from_plan.py`
— is separate design work (the naive span would hold the lock across plan-file commits, interactive confirmation
prompts, agent launch, and network publication). It stays out of this plan; plan it separately if wanted.

## Step 1 — Merge the sase-core v0.11.0 release PR and verify the publish

1. Open sase-core via `/sase_repo`. Confirm `origin/master` still contains `5df18bb`, that no tag contains it yet
   (`git tag --contains 5df18bb` empty), and that PR #32 still targets version 0.11.0 with the sase-9v.9 changelog line.
   If master gained commits after the PR's last refresh, let release-plz refresh the PR (rerun its latest workflow or
   wait for the on-push/scheduled run) before merging so the changelog stays complete.
2. Merge PR #32 (normal merge flow for release PRs; its title already satisfies the conventional-commit check).
3. Watch the release-plz workflow run triggered by the merge push: it must tag `v0.11.0`, create the GitHub release,
   build the wheel matrix, and publish to PyPI. If the run dies partway, rerun it or wait for the 6-hourly self-heal run
   — the publish gate is idempotent.
4. Verify the publish: `curl -s https://pypi.org/pypi/sase-core-rs/json` reports `info.version == "0.11.0"` with a wheel
   set matching 0.10.0's platforms (manylinux x86_64 + aarch64, macOS universal2, Windows amd64, plus sdist).

## Step 2 — Bump the sase dependency window

In the sase repo (this repo):

1. pyproject.toml: `sase-core-rs>=0.10.0,<0.11.0` → `sase-core-rs>=0.11.0,<0.12.0`.
2. Re-lock so `uv.lock` resolves registry sase-core-rs 0.11.0 (e.g. `uv lock`; then `just install` to rebuild the
   workspace environment).
3. tests/test_sase_core_rs_telemetry_smoke_tool.py: update the declared-minimum assertion in
   `test_declared_minimum_tracks_pyproject_dependency` from `"0.10.0"` to `"0.11.0"` (it tracks the pyproject inclusive
   floor).
4. Run `just check`; all stages must pass. Commit the bump as `build(deps): require sase-core-rs 0.11 (sase-9v)`
   (mirroring the sase-9t bump commit's shape: pyproject.toml + uv.lock + smoke-test assertion).

## Step 3 — Land epic sase-9v (final step)

1. Close the epic: `sase bead close sase-9v`.
2. Run `just symvision`. Epic-symbol whitelist entries for sase-9v expire at close — remove the stale whitelist entries
   and any unused code it reports, and commit those cleanups if any result.
3. Mark the epic plan file done: in the plans store (resolve with `sase repo open sase--plans` /
   `sase repo path plans`), edit `202607/bead_review_hardening.md` frontmatter `status: wip` → `status: done`, and
   commit it through the plans-store flow.
4. Sanity check: `sase bead show sase-9v` reports CLOSED with all 11 phases closed, and `sase bead list` shows no stray
   open sase-9v children.

## Verification

- PyPI `sase-core-rs` latest version is 0.11.0 with the full wheel matrix.
- sase `uv.lock` pins registry sase-core-rs 0.11.0; `just check` is green on the bump commit.
- `just symvision` is green after the sase-9v whitelist cleanup.
- `sase bead show sase-9v` is CLOSED and the epic plan file frontmatter reads `status: done`.
