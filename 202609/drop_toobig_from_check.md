---
tier: tale
title: Drop toobig from just check while CI keeps enforcing it
goal:
  just check and just check-full no longer run the toobig line-count gate, while CI
  still runs it via just lint and fails when it fails.
size: small
proposed_by: bbugyi200.athena.0rp
create_time: 2026-09-24 18:33:47
status: wip
---

# Drop `toobig` from `just check` (keep it enforced in CI)

## Goal

Stop running the `toobig` line-count gate (`just _lint-toobig`) as a stage of the agent
verification recipe `just check`. The gate must stay enforced in CI: CI still runs it
and fails when it fails.

**Why:** agents can rarely act on a `toobig` failure. A file that grows past the
threshold is usually not the current agent's to split, and the `toobig_split` routine
(the `toobig-*` agent clan) already handles those splits. Today every unrelated agent's
`just check` goes red on the same oversized file, and the only fixes open to that agent
are out of scope.

## How CI keeps enforcing it (no workflow change needed)

CI never calls `just check`. It runs the gate through `just lint`:

- `.github/workflows/ci.yml` job `lint` runs `just lint`.
- `.github/workflows/master-gate.yml` job `lint` runs the same steps
  (`tests/test_github_actions_ci_master_gate.py` enforces the match).
- The `lint` recipe in `Justfile` ends with `@just _lint-toobig`.

So removing the stage from `check` leaves CI coverage unchanged. **Do not** edit `lint`,
`_lint-toobig`, the public `toobig` recipe, or either workflow file.

## Design decision: remove it from `check-full` too

`check` and `check-full` are required to share one identical list of non-test gates
(`test_check_and_check_full_share_an_identical_gate_list` in
`tests/test_justfile_lint.py`). The project also has this rule: "a `just check` pass
with a `just check-full` failure is a test-infrastructure bug" (`docs/development.md`,
`sase/memory/lint_and_test.md`). If `toobig` stays in `check-full` only, both of those
stop being true for this one gate. Agents also run `check-full` mainly to repair
unrelated CI failures, and a `toobig` stage there would block that repair in exactly the
same way.

So remove the `lint (toobig)` stage from **both** recipes. `just lint`, `just toobig`,
and CI remain the places where the gate runs.

## Changes

### 1. `Justfile`

- In the `check:` recipe, delete this line:
  `@tools/run_silent "lint (toobig)"      just _lint-toobig`
- In the `check-full:` recipe, delete the same line.
- In the comment block above `check:` (the one that begins "Agent default: whole-repo
  lint gates plus a diff-scoped test lane..."), add a short paragraph explaining the
  omission. Match the comment style around it. Suggested wording:

  ```
  # The `toobig` line-count gate is deliberately not a `check`/`check-full`
  # stage: an agent can rarely act on it (the toobig_split routine owns those
  # splits). It still runs in `just lint`, which CI's lint and master-gate jobs
  # run, and on demand via `just toobig`.
  ```

- Keep the header comment on `lint` ("... + symvision + toobig + keep-sorted"). It stays
  accurate.

### 2. `tests/test_justfile_lint.py`

- Remove `'tools/run_silent "lint (toobig)"      just _lint-toobig',` from
  `_CHECK_GATE_LINES`.
- Replace `test_check_mirrors_lint_toobig_stage` with a test that asserts the opposite
  for both recipes. For example, `test_check_and_check_full_skip_toobig_stage`: for each
  of `check` and `check-full`, `_dry_run(recipe)` must contain neither `_lint-toobig`
  nor `lint (toobig)`.
- Keep `test_lint_includes_toobig_stage` and
  `test_public_toobig_target_uses_private_lint_stage` unchanged. They now act as the
  guard that CI's `just lint` still enforces the gate. Give
  `test_lint_includes_toobig_stage` a one-line docstring saying this is how CI enforces
  `toobig` now that `check` skips it.
- Update the docstring of `test_check_and_check_full_share_an_identical_gate_list` only
  if it counts the gates ("a tenth lint or validation gate"). Adjust or drop the number
  so it isn't stale.

### 3. `tests/test_github_actions_ci_workflow.py`

- `test_lint_job_uses_single_lint_command` already asserts that the CI `lint` job runs
  `just lint`. Leave it as is. Optionally add a comment tying it to the `toobig`
  enforcement. No behavior change.

### 4. `docs/development.md`

- In "### Diff-scoped checks (`just check`)", the first paragraph says "every whole-repo
  lint gate runs unchanged". Change it to say every whole-repo lint gate except `toobig`
  runs, and add one sentence on why: agents rarely can act on it, and the toobig_split
  routine owns those splits. Also say it still runs in `just lint`, and therefore in CI.
- In the same section, the paragraph that describes `just check-full` as "every lint
  gate" needs the same exception (it also skips `toobig`).
- The command table line `just lint # Run ruff, mypy, ..., toobig, and keep-sorted`
  stays as is.

### 5. Leave unchanged (checked, still correct)

- `src/sase/xprompts/split_file.md` explicitly runs `just _lint-toobig` before
  `sase tool run check`. The split agent _can_ act on the gate, and
  `tests/test_xprompt_inline_code.py` pins that text. Keep it.
- `CHANGELOG.md` is generated by release-please. Do not edit it.
- `.github/workflows/*.yml`: no change (see above).

### 6. Memory: do NOT edit

`sase/memory/lint_and_test.md` says `just check` runs "every whole-repo lint gate".
After this change that is slightly stale. The user did not authorize a memory edit, so
**do not edit any file under `sase/memory/`**. Instead, use `/sase_new_task` to file a
`memory` task bead (size `xsmall`) naming `sase/memory/lint_and_test.md` and the
proposed wording: `just check` runs every whole-repo lint gate except `toobig` (CI
enforces it via `just lint`; the toobig_split routine owns splits).

## Verification

1. `just --dry-run check` and `just --dry-run check-full` contain no `_lint-toobig`;
   `just --dry-run lint` still does.
2. `.venv/bin/pytest tests/test_justfile_lint.py tests/test_github_actions_ci_workflow.py tests/test_github_actions_ci_master_gate.py tests/test_xprompt_inline_code.py`
   passes.
3. Run `just fix`, then `sase tool run check` (the `Justfile` is in the selector's
   broadening set, so expect the scoped lane to escalate to the full test lane; run it
   through `/sase_monitor` if it will not fit in the turn). It must pass, and its stage
   list must no longer show `lint (toobig)`.
