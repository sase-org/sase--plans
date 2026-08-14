---
tier: tale
title: A lane's dirty-path baseline belongs to the lane, not to each continuation run
goal:
  A run that continues an existing lane inherits that lane's dirty-path baseline instead
  of capturing a fresh one, so a monitor follow-up is never told its own lane's
  uncommitted work is pre-existing dirt it must not commit. With that fixed, epic
  sase-lb.1 is closed, its expired symbol whitelist is cleaned up, and its plan file is
  marked done.
size: medium
proposed_by: bbugyi200.athena.sase-lb.1.land
bead: sase-lb.1
create_time: 2026-08-14 13:08:29
status: done
---

- **PROMPT:**
  [prompts/202608/lane_baseline_inheritance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/lane_baseline_inheritance.md)
- **PARENT:**
  [202608/workspace_claim_invariant.md](https://github.com/sase-org/sase--plans/blob/main/202608/workspace_claim_invariant.md)
- **BEAD:**
  [sase-lb.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-lb/sase-lb.1.md)

# Plan: A lane's dirty-path baseline belongs to the lane, not to each continuation run

## Problem

Epic `sase-lb.1` phase `sase-lb.1.6` taught the commit finalizer to ignore paths that
were already dirty when a run started, so an agent is no longer told it must commit
another agent's in-flight edits. The capture is unconditional: every runner start
snapshots the workspace's dirty paths into that run's own artifacts directory.

That is wrong for a run that _continues_ an existing lane. A monitor follow-up is a
brand-new agent run in the same workspace, launched after the starter was SIGTERMed
mid-turn with the `monitored` outcome — so the starter's own uncommitted work is still
on disk and its commit finalizer never ran. The follow-up captures a fresh baseline that
includes that work, and the finalizer then excludes it from the must-commit set and
reports it under:

```
Pre-existing changes detected before this run started (not from this run — do not commit these):
```

The follow-up is explicitly launched to finish the lane's work, so nothing else will
ever commit it. This breaks the repo's own documented workflow: run `just check-full`
through `/sase_monitor`, hand it a `--next` action, and let the follow-up agent act on
the result. Before phase `sase-lb.1.6` the follow-up was told to commit that work; now
it is told not to.

The same reasoning applies to any family-attach child: it runs in the parent's workspace
and continues the family's work, so the family's earlier uncommitted work is the child's
responsibility, not foreign dirt.

## Working tree state you are inheriting

The epic's land agent left one uncommitted change in the workspace:
`tests/llm_provider/test_commit_finalizer_baseline.py`. It fixes
`test_pre_existing_sibling_file_is_excluded_and_reported_separately`, whose mock agent
deleted its own uncommitted file — which bead `sase-ly`'s discarded-dirty-work guard
(landed mid-epic) correctly rejects. The mock now commits its own file, and the test
additionally asserts the pre-existing file is neither committed nor discarded.
`just check` passes with it.

Commit that change along with this plan's work. Do not treat it as pre-existing dirt —
it is this lane's work, which is precisely the failure mode this plan exists to fix.

## Root cause

`src/sase/axe/run_agent_runner_bootstrap.py::_capture_commit_finalizer_baseline` calls
`capture_dirty_baseline(resolve_finalizer_project_dir(), artifacts_dir)` on every runner
start, gated only by `SASE_DISABLE_COMMIT_STOP_HOOK`. It has no notion of a run that
inherits an existing lane, so each continuation re-baselines against a workspace that
already contains its own lane's work.

The correct unit is the lane: the baseline should be captured once, by the lane's first
agent, and inherited by every continuation. Truly foreign dirt — whatever was in the
workspace before the lane started — stays excluded, which preserves everything phase
`sase-lb.1.6` was built to do.

## Fix

Inherit the parent's baseline instead of capturing a fresh one when this run is a
family-attach continuation.

- A family-attach child already receives its parent's identity in the environment:
  `load_family_attach_plan_from_env()` in `src/sase/agent/_family_attach_launch.py`
  reads the `SASE_AGENT_FAMILY_ATTACH` payload (`_types.FAMILY_ATTACH_ENV`), whose
  `parent_artifacts_dir` field points at the parent run's artifacts directory. That
  environment variable is set by the launcher, so it is readable at bootstrap time —
  `_capture_commit_finalizer_baseline` runs before `extract_directives_and_write_meta`,
  but the env payload does not depend on it.
- In `_capture_commit_finalizer_baseline`, before capturing: if a family-attach plan is
  present and `<parent_artifacts_dir>/commit_finalizer_baseline.json` exists, copy that
  file into this run's artifacts directory and return without capturing.
- Fall back to a fresh capture whenever there is no plan, no parent baseline file, or
  the copy fails. Baseline handling is best-effort by design — never fail a run over it,
  matching `capture_dirty_baseline`'s existing contract.
- Keep the `SASE_DISABLE_COMMIT_STOP_HOOK` early return exactly as it is.
- `BASELINE_FILENAME` is already exported from
  `src/sase/llm_provider/commit_finalizer_baseline.py`; reuse it rather than hardcoding
  the filename.

Inheriting a stale baseline is safe: `split_pre_existing_changed_files` requires the
recorded status _and_ content hash to still match, so a path the parent has since
committed simply no longer appears, and a path the child edits again keeps a different
fingerprint and stays in the must-commit set.

## Tests

Add to `tests/test_run_agent_runner_bootstrap_baseline.py` (which already covers the
capture-after-entering-the-workspace ordering):

- A family-attach run whose parent artifacts directory holds a baseline inherits that
  file byte-for-byte and does not re-capture, so work the parent left uncommitted stays
  in the must-commit set.
- A family-attach run whose parent has no baseline file falls back to a fresh capture.
- A run with no `SASE_AGENT_FAMILY_ATTACH` in the environment captures fresh, exactly as
  today.
- A malformed `SASE_AGENT_FAMILY_ATTACH` payload does not fail the run; it falls back to
  a fresh capture. (`load_family_attach_plan_from_env` raises `FamilyAttachError` on a
  bad payload, so this path must catch it.)
- An unreadable parent baseline file falls back to a fresh capture.

Add to `tests/llm_provider/test_commit_finalizer_baseline.py` an end-to-end assertion of
the behavior that motivated this plan: given a baseline captured _before_ a starter's
work existed, a continuation run's finalizer still lists that work in the must-commit
set rather than under the pre-existing header.

## Verification

- `just check` after the fix.
- `just check-full` through `/sase_monitor` before landing: this touches the runner
  bootstrap and the launch path, both in the broadening set, and it lands epic
  `sase-lb.1`'s combined tree. Hand the monitor a `--next` action so the follow-up agent
  performs the landing step below.

## Landing epic sase-lb.1

This plan is the last outstanding work for epic `sase-lb.1`. After the fix is verified,
finish the landing:

1. Close the epic:

   ```
   sase bead close sase-lb.1 --note "<what was verified>"
   ```

   The note must record: all seven phases verified against their notes and against the
   source; the integration with bead `sase-ly`'s discarded-dirty-work guard (which
   landed mid-epic and conflicted with phase `sase-lb.1.6`'s test, since fixed); and
   this baseline-inheritance regression and its fix. It must also record the disposition
   of the single `PROPOSED FOLLOW-UP` collected from the child beads — the
   `test_pre_existing_sibling_file_is_excluded_and_reported_separately` failure recorded
   on `sase-lb.1.7`, which was not an unrelated follow-up but epic-caused fallout from
   that `sase-ly` integration, and was fixed rather than filed as a task.

2. Run `just symvision` and remove the stale epic-symbol whitelist entries and unused
   code it reports — `sase-lb.1`'s whitelist entries expire when the epic closes.
3. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202608/workspace_claim_invariant.md`.

If the close is rejected, the named phases were never completed: finish or reopen them
rather than forcing. Never force merely to make the command succeed.
