---
tier: tale
title: Fix sase-core CI by pinning astral-sh/setup-uv to an existing release tag
goal:
  sase-core's CI `release workflow scripts` job and the weekly PyPI headroom detector
  resolve astral-sh/setup-uv again, and an offline script test keeps a floating setup-uv
  ref from coming back.
size: small
proposed_by: bbugyi200.athena.127
proposed_with: claude/opus
create_time: 2026-10-19 16:48:13
status: wip
---

# Plan: Fix sase-core CI by pinning `astral-sh/setup-uv` to a release tag that exists

## Problem

Every sase-core `CI` run on `master` has failed since `9876b3ae`
(`feat(releases): protect PyPI consumer windows`). The same failure repeats on
`a908ceb3` and `f0f2275f`. `actstat` and `gh run view --log-failed` show one failing
job, `release workflow scripts` (job id `release-scripts` in
`.github/workflows/ci.yml`). It dies in step 1, "Set up job":

```
##[error]Unable to resolve action `astral-sh/setup-uv@v10`, unable to find version `v10`
```

The other `CI` jobs pass in the same runs: `cargo fmt + clippy + test` on ubuntu and
macOS, and `maturin build + import smoke`. Release-plz runs also pass. Because the job
fails before any step runs, the release-helper script tests have not run in CI since
`9876b3ae`.

## Root cause

Since v8, `astral-sh/setup-uv` publishes only exact `vMAJOR.MINOR.PATCH` tags. It no
longer publishes floating `vN` or `vN.M` tags. `v7` is the last floating major that
resolves. `v8`, `v9`, `v10` and `v10.3` all return 404. The current release is `v10.3.0`
(commit `1c37ad07a6a961277cf70c0d37d6f313000f5884`). So `@v10` can never resolve.

Two commits from the closed epic `sase-1my` added the bad ref:

- `9876b3ae` switched the `release-scripts` job from bare `python3` to
  `astral-sh/setup-uv@v10`, because `scripts/check.sh script-test` now runs
  `uv run --no-project --with packaging --with requests python -m unittest discover ...`.
  This is the failure we see now.
- `a908ceb3` added the new weekly `.github/workflows/pypi-headroom.yml` detector with
  the same `astral-sh/setup-uv@v10`. It has not run yet, because its cron is Monday
  08:17 UTC. Its next scheduled run would fail at "Set up job" the same way. "The red
  run is the alert" for that detector, so the run would look like a PyPI runway alert
  while measuring nothing.

`actionlint` passes both files, so nothing local catches a ref that does not exist.

Nothing in the code under test is broken. The release-helper tests already pass offline
with the exact command CI runs, `./scripts/check.sh script-test` (94 tests, OK). They
also pass when forced onto the runner's Python 3.12
(`uv run --no-project --python 3.12 ...`). Once the action resolves, the job should go
green.

## Changes (all in the linked `sase-core` repo)

Open the repo with `sase repo open sase-core -r "<reason>"` and work only in the path it
prints. Read its `AGENTS.md` first.

1. **Add the regression guard first:** create
   `.github/scripts/test_workflow_action_pins.py`. Model it on its siblings
   (`test_pypi_headroom.py`, `test_release_cadence.py`): a shebang, a module docstring,
   `from __future__ import annotations`, stdlib-only `unittest`,
   `REPO_ROOT = Path(__file__).resolve().parents[2]`, and no YAML parser.
   - Scan every `.github/workflows/*.yml` and `*.yaml` file for
     `uses: astral-sh/setup-uv@<ref>`. Use a regex that tolerates the `- uses:` form and
     a trailing `# comment`.
   - Assert that the scan finds at least one reference, so a renamed path or a broken
     regex cannot make the guard pass vacuously.
   - Assert that each `<ref>` matches either an exact release tag `^v\d+\.\d+\.\d+$` or
     a full commit SHA `^[0-9a-f]{40}$`. Use `self.subTest(...)` per hit, and make the
     failure message name the `file:line` and the bad ref.
   - The docstring explains the rule in a few lines. setup-uv stopped publishing
     floating `vN` and `vN.M` tags after v7. A floating ref such as `@v10` fails the job
     at "Set up job" with "unable to find version", before any step runs. That is the
     failure that kept `release workflow scripts` red from `9876b3ae` on. Scope the rule
     to setup-uv on purpose. The other actions this repo uses (`actions/checkout@v4`,
     `actions/setup-python@v5`, `Swatinem/rust-cache@v2`, and so on) still publish
     floating majors.
   - Run `./scripts/check.sh script-test` before touching the workflows and confirm the
     new test fails on both `ci.yml` and `pypi-headroom.yml`. This proves the guard
     catches the real bug.

2. **`.github/workflows/ci.yml`** (`release-scripts` job): change
   `- uses: astral-sh/setup-uv@v10` to `- uses: astral-sh/setup-uv@v10.3.0`. Add a
   one-line comment above it: setup-uv publishes no floating major tags after v7, so pin
   an exact release (see `.github/scripts/test_workflow_action_pins.py`).

3. **`.github/workflows/pypi-headroom.yml`** (`headroom` job): make the same change to
   `astral-sh/setup-uv@v10.3.0`. A comment is optional. Keep every string that
   `test_pypi_headroom.py` asserts on: `astral-sh/setup-uv`, "never deletes", "the red
   run is the alert", `ci_watch`, the cron line, and the rest.

Pin the exact tag, not a SHA. Every other action in this repo is pinned by tag, and
`v10.3.0` is an immutable release tag, so the exact tag keeps workflows readable and
consistent. Do not touch `scripts/check.sh`, because its `uv run` command is correct. Do
not edit any `version` field or `CHANGELOG.md`. Do not touch the sase repo's own
workflows. They use `astral-sh/setup-uv@v4`, which still resolves.

## Verification

- `gh api repos/astral-sh/setup-uv/git/ref/tags/v10.3.0 --jq .object.sha` prints
  `1c37ad07a6a961277cf70c0d37d6f313000f5884`. This confirms the pinned ref exists.
- `actionlint` (from the sase-core root) reports nothing.
- `./scripts/check.sh script-test` passes, and the new guard is in the count (95+
  tests).
- `sase tool run check` in the sase-core root passes. This is the guarded `just check`
  gate and runs the same steps as CI. Give it an explicit tool timeout of at least 10
  minutes.
- Do not `workflow_dispatch` `pypi-headroom.yml` to test it. It can open or close a
  mirrored issue on `sase-org/sase`, which is outward-facing. Its next Monday cron run
  exercises the fix.

After the host lands the commit, the first `CI` run on the new `master` head should show
`release workflow scripts` green, and `actstat` should show sase-core `✔`. Check with
`actstat --color never -n 2` or
`gh run list --repo sase-org/sase-core --workflow CI --limit 1`.

Suggested commit subject: `fix(ci): pin astral-sh/setup-uv to an existing release tag`.
