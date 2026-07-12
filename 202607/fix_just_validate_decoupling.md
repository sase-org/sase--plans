---
create_time: 2026-07-12 09:11:15
status: done
prompt: 202607/prompts/fix_just_validate_decoupling.md
tier: tale
---
# Fix `fix_just` Lumberjack Churn Without Dropping `sase validate` From CI

## Problem

The `fix_just` lumberjack chop keeps spawning fixer agents (branches `sase_fix_just_linters_4` through `_14`, plus
recurring `fix_hook` agents) because `just lint` and ChangeSpec hooks keep failing for reasons that are not fixable from
this repository's source tree. The latest fixer attempt (commit `09d61f57f`, branch `sase_fix_just_linters_14`)
decoupled `sase validate` from `just lint` — directionally right, but it silently removed SASE validation from CI,
because the CI lint job runs `just lint` (`.github/workflows/ci.yml`, "Lint" step). The agent's claim that "`just check`
retains the gate" is true but irrelevant to CI: CI never runs `just check`.

There are three independent root causes feeding the churn, confirmed by evidence:

1. **SDD companion data is broken** (this alone keeps master CI red — the `lint` job is the only failing job on master
   right now, and it fails exactly in the `sase validate` → `sdd validate` stage with 3 errors):
   - `plans/202607/prompt_stash_pin_fixes.md` (on `sase-org/sase--sdd` master) has
     `prompt: .sase/sdd/prompts/202607/prompt_stash_pin_fixes.md` — the old, un-nested path. The snapshot actually lives
     at `plans/202607/prompts/prompt_stash_pin_fixes.md` (which correctly links back to the plan). This produces both a
     `link-missing-target` error and a `reverse-link` error.
   - `plans/202607/migrate_actstat_sdd_prompts.md` has
     `prompt: .sase/sdd/plans/202607/prompts/migrate_actstat_sdd_prompts.md`, but that snapshot file was never committed
     to the companion (`link-missing-target`).
   - Both plans were created 2026-07-11 during the prompt-snapshot nesting migration; this is a half-completed data
     migration, not a validator bug.

2. **Home initialization drift**: `sase init --check` fails outside CI (CI regenerates home state via
   `sase init memory --no-commit` before linting). Locally it wants to refresh 8 memory files / provider shims for the
   relocated linked-repository clones (from the "relocate linked repository clones" change) and to connect/import the
   SDD companion. This drift is environment state — a repo fixer agent cannot (and must not) fix it: memory-file edits
   require explicit user permission.

3. **Hook environments fail in `_setup` itself**: the `just lint` / `just test` ChangeSpec HOOKS die with
   `[validate_sase_core_rs_version] missing TOML file: ../sase-core/Cargo.toml` before any linter runs. The `_setup`
   recipe guards the local Rust-core build on `[ -d "{{ sase_core_dir }}" ]` only, so a stale or empty `../sase-core`
   directory (leftover from the clone relocation) hard-fails setup. This is what spawns the recurring `fix_hook` agents,
   and it would continue even after the lint/validate split.

Structurally, coupling environment/data validation (`sase validate` = `sase init --check` + `sase sdd validate`) into
`just lint` means the chop's `fix_linters` agent gets launched for failures that have no repo-source fix. That is the
recurrence engine.

## Goals

- `sase validate` keeps gating CI (explicit requirement from Bryan — do not drop it).
- `just lint` becomes deterministic on repository source, so the `fix_just` chop only spawns fixer agents for real
  source-lint failures.
- Master CI returns to green (requires the companion data repair).
- ChangeSpec hooks stop failing in `_setup` in environments without a valid `sase-core` checkout.
- Surface (but do not silently perform) the home-drift cleanup that needs Bryan's approval.

## Non-Goals

- No changes to the `fix_just` chop configuration (chezmoi) or `xprompts/fix_just.yml` — once `just lint` is
  source-only, the chop's behavior is correct as-is.
- No removal of the SASE validation stage from `just check` (agents' full gate keeps it).
- No new CLI commands or flags.

## Design

### 1. Keep the lint/validate split (already on branch `sase_fix_just_linters_14`)

Commit `09d61f57f` already removes the SASE validation stage from `just lint`, keeps it in `just check`, updates
README/docs, and adds a regression test. Build on that branch rather than reverting it.

### 2. Restore SASE validation in CI as an explicit step

In `.github/workflows/ci.yml`, add a step to the `lint` job immediately after "Lint":

```yaml
- name: SASE validation
  run: just validate
```

Net CI behavior is unchanged from before the split (still gates on `sase validate`), with a bonus: failure attribution
in the GitHub UI now distinguishes source-lint failures from validation failures. The existing job setup (SDD companion
checkout, `sase init memory --no-commit`, `sase skill init --force`) stays — it exists to make validation hermetic in
CI.

### 3. Add the missing positive tests

`tests/test_justfile_lint.py` currently only asserts the negative (`lint` lacks validation). Add:

- `test_check_retains_sase_validation_stage` — dry-run `check`, assert `tools/run_silent "SASE validation"` (guards the
  invariant the CL claims but never tests).
- A test asserting the CI lint job still runs `just validate` (read `.github/workflows/ci.yml` and assert a step runs
  it), so a future agent cannot silently drop validation from CI again — exactly the bug being fixed here.

### 4. Harden `_setup` against stale/partial `sase-core` checkouts

Change the `_setup` guard in the `Justfile` from `[ -d "{{ sase_core_dir }}" ]` to requiring the checkout to actually be
usable (`[ -f "{{ sase_core_dir }}/Cargo.toml" ]`). When the directory exists but has no `Cargo.toml`, print a clear
notice (e.g. "sase-core checkout at ... has no Cargo.toml; treating as absent and using the published sase-core-rs
wheel") and skip the local build path — the subsequent `uv pip install --no-sources` already installs the pinned
`sase-core-rs` distribution. This kills the `fix_hook` churn class at the root while still hard-failing on genuine
version mismatches when a real checkout is present.

### 5. Repair the SDD companion data (makes master CI green) — needs Bryan's sign-off

Against a clean, up-to-date checkout of the `sase--sdd` companion (the working repo's `.sase/sdd` after syncing to
`origin/master`, or a fresh clone):

1. Run `sase sdd repair-links` (dry) to see what it can infer, then `sase sdd repair-links -w`. Expected: it fixes
   `plans/202607/prompt_stash_pin_fixes.md`'s `prompt:` link to the nested path.
2. For the missing `migrate_actstat_sdd_prompts` snapshot: recover the launch prompt from the agent chat transcript and
   create `plans/202607/prompts/migrate_actstat_sdd_prompts.md` with the proper `plan:` backlink; if recovery is not
   practical, drop the dangling `prompt:` line from the plan instead.
3. Verify `sase sdd validate` reports 0 errors, then commit and push to the companion's master (direct commits to master
   match the companion's existing auto-sync convention).

This is a data repo owned by Bryan; the push happens only because this plan was approved.

### 6. Environment cleanup (manual, for Bryan — listed for visibility, not for the agent)

- Run `sase init memory` in the home context to clear the 8-file provider-shim drift for the relocated linked-repo
  clones. (Memory files — requires your approval/action; the implementing agent must NOT do this unilaterally.)
- Optionally delete the stale `../sase-core` directory in the hook-runner environment; step 4 makes this unnecessary for
  correctness, but the leftover directory is dead weight.

## Verification

- `just check` passes in the implementation workspace (source stages; validation stage expected to pass once steps 5–6
  land, otherwise its failure must be limited to the known drift and called out in the reply).
- `just lint` dry-run tests pass (`tests/test_justfile_lint.py`).
- CI on the PR branch: `lint` job green including the new "SASE validation" step.
- After step 5 lands: next master push CI run is fully green.
- The next `fix_just` chop run launches no `fix_linters`/`fix_tests` fixer (observable via the chop's decide_fixers
  output: `reason=all_just_checks_passed`).
