---
tier: tale
title: Close out sase-10w.5 with green CI and a consulted coverage baseline
goal: Master Gate and Full CI are green on a tip containing the core-pin fix, a fresh
  coverage-contexts baseline is installed and proven consulted by the scoped lane,
  evidence is on the bead, and sase-10w.5 is closed.
size: medium
proposed_by: bbugyi200.athena.sase-10w.5.f0
status: done
---

# Plan: Close out sase-10w.5 — green CI on the fixed tip, then a fresh, consulted coverage baseline

## Context

Bead `sase-10w.5` is the `green-ci-and-baseline` phase of epic `sase-10w` (plan
`plan:202609/green_ci_fast_lane_v0_17_2.md`). Its exit criteria:

1. Master Gate green on the master tip.
2. A fully green Full CI run whose `coverage-contexts` job uploads a
   `sase-coverage-contexts-<sha>` artifact containing `.coverage`.
3. `just refresh-contexts-baseline` installs that baseline on this host, and a scoped
   run on a small diff consults it fresh: `context-baseline-stale`,
   `context-baseline-missing` and `no-baseline-depth-boost` do not fire.
4. Before/after `just selection-health` evidence on the phase bead, plus a
   `PROPOSED FOLLOW-UP:` if escalations stay dominated by budget or staleness rules.

State as of planning (2026-09-14, ~15:55 UTC). Re-verify all of it, because master moves
hourly:

- The previous turn on this bead found Master Gate run 34861266462 (on `cc91c0aa43`)
  red. The CI-built core wheel lacked seven bindings that Python requires. The fix
  `9ff2662c0c fix(ci): bump pinned sase-core revision` (sets `sase-core-revision.txt` to
  `afe7b70dbede…`) **has landed on `origin/master`**. Master Gate run **34863923659** on
  `9ff2662c0c` was in progress (`core-wheel` job).
- The newest Full CI run (34846053851, scheduled, `f86056c7fd`, before phases 1–4
  landed) failed `lint`, `test (3.12)`, `test (3.14)`, `visual-test`, `contention-test`,
  `coverage-contexts`. No Full CI has run on a SHA that contains `9ff2662c0c`.
- Full CI (`.github/workflows/full.yml`) calls `ci.yml` and runs on cron `17 */2 * * *`
  plus `workflow_dispatch`. Its concurrency group is `full-ci` with
  `cancel-in-progress: false`. GitHub keeps only one _pending_ run per group, so a newer
  queued run replaces an older pending one. `contention-test` only runs on `schedule`,
  and `release-core-floor-smoke` only runs on the release-please PR, so neither gates a
  dispatched master run. Recent runs took about 1h40m.
- `coverage-contexts` runs `just test-contexts` and then uploads `.coverage` under
  `if: always()` with `if-no-files-found: warn`. "Artifact uploaded" and "job green" are
  separate checks; verify both.
- `tools/fetch_coverage_contexts` (`just refresh-contexts-baseline`) walks recent
  `full.yml` master runs newest first. It downloads the first artifact whose SHA is an
  ancestor of `HEAD` into `${SASE_HOME:-~/.sase}/test-selection/contexts/`, which is
  shared by every workspace on this host. It stops early with `already cached` if that
  SHA is already cached. It does not check run conclusion, so confirm that the SHA it
  prints is the green run's SHA.
- Freshness (`tests/_test_selection_contexts.py`): a baseline is stale when it is more
  than `DEFAULT_MAX_DISTANCE = 50` commits behind `HEAD`. `no-baseline-depth-boost`
  (`tests/_test_selection.py`) fires whenever contexts are not usable. The scoped lane's
  change set is the merge-base diff against `origin/master` plus the working tree. Its
  manifest is written to `.pytest_cache/sase-selection/manifest.json` with keys
  `rules_fired`, `escalated`, `outcome`, and
  `contexts.{baseline, consulted, stale, distance, selected_count, matched_files}`.
- Full-suite rules decided before contexts are consulted: `base-unresolved`,
  `root-conftest`, `packaging-config`, `justfile`, `src-data-asset`,
  `selection-tooling`, `core-identity-changed`. The proof diff must avoid all of them.
  That is why the previous turn's core-pin diff could not serve as proof (manifest
  showed `contexts.consulted: false`).
- Before-snapshot already recorded in bead note #1: 2,343 scoped runs, 1,551 escalated
  (66.2%), median 121.8 s, p90 648.8 s; `context-baseline-stale` and
  `no-baseline-depth-boost` 832 each, `serial-budget-exceeded` 436. Note #2 already
  records the separate `sase-core-rs` PyPI floor gap. That gap belongs to `sase-10w.6` /
  the land agent, not this phase; do not act on it here.
- The earlier epic note about symvision flags on `continuation_protected_dirs` /
  `iter_ace_run_month_dirs` is already resolved at HEAD (the helper is now private
  `_iter_ace_run_month_dirs`).

## Operating rules

- **Every long wait goes through `/sase_monitor`**, never inline polling. That means CI
  watches, sleeps, and any long `just` command. Each `sase monitor start` ends the turn.
  Put the exact continuation in `--next`: the step number below, the run ID, and what to
  check. The follow-up agent forks this conversation. Use a quiet polling loop rather
  than `gh run watch`, which floods retained output. For example:
  `while [ "$(gh run view <ID> --json status -q .status)" != completed ]; do sleep 120; done; gh run view <ID> --json conclusion,jobs -q '.conclusion, (.jobs[] | "\(.name)\t\(.conclusion)")'`
- Coordination rules from the epic plan apply. Distinguish flakes from deterministic
  failures by rerunning on an unchanged tree first (`gh run rerun <ID> --failed`, once).
  Consume fixes that active owning epics have already landed instead of re-fixing.
  Record flakes as `PROPOSED FOLLOW-UP:` notes (the land agent files `flake` beads).
- **Do not create beads**, and do not close `sase-10w`, `sase-10w.6`, or any ancestor.
  Record discovered work with `sase bead note sase-10w.5 'PROPOSED FOLLOW-UP: …'`.
- Never hand-merge or touch release PR #299 or `publish.yml`. That is `sase-10w.6`.
- If a deterministic CI failure needs a code repair: make the narrowest fix, verify it
  with `just check` (monitor it if it runs long), and land it through the normal
  `/sase_final` commit declaration with the bead kept `in_progress`. Add a bead note
  listing which of the steps below remain. Do not close the bead in a turn that lands a
  repair: the new tip needs its own Master Gate and Full CI.
- Leave no repo changes behind except a deliberate, verified repair. The proof diff in
  step 4 is temporary and must be reverted.

## Steps

### 1. Master Gate green on the tip

1. `git fetch origin master`. Find the newest Master Gate run for `origin/master`'s SHA:
   `gh run list --workflow master-gate.yml --branch master --limit 5 --json databaseId,headSha,status,conclusion`.
   Confirm the SHA contains `9ff2662c0c`
   (`git merge-base --is-ancestor 9ff2662c0c <sha>`).
2. If it is still running, start a monitor with the quiet polling loop (timeout about
   45m, labels `WATCHING GATE` / `WATCHED GATE`). In `--next`, ask the follow-up to
   resume at step 1.3 with that run ID.
3. On completion:
   - **Green:** go to step 2.
   - **Red:** pull the failed job logs
     (`gh api repos/sase-org/sase/actions/jobs/<job_id>/logs`, tail around the error).
     Check specifically whether `lint / Check pinned core bindings` still reports
     missing symbols; it should not after the pin bump. Treat infrastructure failures
     like the earlier `just` download 504 as infra: `gh run rerun <ID> --failed` and
     watch again. Handle deterministic failures under the repair rule above.

### 2. A fully green Full CI run with an uploaded contexts artifact

1. Before dispatching, list Full CI runs:
   `gh run list --workflow full.yml --limit 5 --json databaseId,headSha,status,conclusion,event,createdAt`.
   If a run is `queued` / `in_progress` on a SHA that contains `9ff2662c0c` and whose
   Master Gate is green, watch that run instead of dispatching a duplicate. A scheduled
   `:17` run counts. Otherwise dispatch with `gh workflow run full.yml --ref master`,
   then find the new run ID from
   `gh run list --workflow full.yml --event workflow_dispatch --limit 1`. Record the run
   ID and head SHA.
2. Start a monitor with the quiet polling loop (timeout about 4h to cover queueing
   behind an in-flight run, labels `WATCHING FULL CI` / `WATCHED FULL CI`). In `--next`,
   ask the follow-up to resume at step 2.3 with that run ID and SHA. If the run is
   cancelled because a newer run replaced it while pending, switch to the newer run on a
   qualifying SHA and keep watching.
3. On completion, require **every** job to be `success` or an expected `skipped`
   (`docs-build`, `release-core-floor-smoke`, and `contention-test` on non-schedule
   events). Then verify the artifact:
   `gh api repos/sase-org/sase/actions/runs/<ID>/artifacts --jq '.artifacts[] | "\(.name)\t\(.size_in_bytes)\t\(.expired)"'`
   must list `sase-coverage-contexts-<sha>`, unexpired, with a plausible size (tens of
   MB). Also confirm the `coverage-contexts` job log does not contain
   `No files were found with the provided path: .coverage`.
4. If a job fails, triage it the same way as step 1.3. Suspected flakes (for example
   lock-deadline contention or load-dependent timing): `gh run rerun <ID> --failed` once
   and watch again. A rerun that goes green satisfies the criterion; record the flake as
   a follow-up note with the job, test node, and both attempts. Handle deterministic
   failures under the repair rule, then repeat steps 1–2 on the new tip.

### 3. Install the fresh baseline on this host

1. Make sure the workspace `HEAD` contains the green Full CI SHA
   (`git merge-base --is-ancestor <sha> HEAD`) and is within 50 commits of it
   (`git rev-list --count <sha>..HEAD`). If the workspace is behind, sync it to
   `origin/master` first.
2. Capture a "before-proof" health snapshot:
   `just selection-health --json > /tmp/sase_10w5_health_before.json`. Keep the headline
   numbers: scoped runs, escalation rate, median / p90 scoped duration, and the
   `rule_histogram` counts for `context-baseline-stale`, `context-baseline-missing`,
   `no-baseline-depth-boost`, and `serial-budget-exceeded`.
3. Run `just refresh-contexts-baseline`. It must print
   `cached coverage-contexts baseline <sha12>` (or `already cached`) for **the green
   run's SHA**. If it picked a different, older SHA, re-run with `--force` after
   confirming why. For example, the green run's artifact may not have been listed within
   `--limit`; widen `--limit` in that case.

### 4. Prove the scoped lane consults it fresh

1. Make a temporary comment-only edit to one leaf, non-tooling Python module under
   `src/sase/` that has a small importer closure, so the serial budget is unlikely to
   trip. Do **not** touch `Justfile`, `pyproject.toml`/`uv.lock`, root conftest modules,
   `tests/_test_selection*` / `tools/run_pytest` (selection tooling),
   `sase-core-revision.txt`, or src data assets.
2. Run `just test-scoped`, monitoring it if the selection is large. Then inspect
   `.pytest_cache/sase-selection/manifest.json` and `tools/print_scoped_summary`. Pass
   criteria:
   - `contexts.consulted == true`, `contexts.stale == false`,
     `contexts.baseline == <green sha>`, `contexts.distance <= 50`
   - `rules_fired` contains none of `context-baseline-stale`,
     `context-baseline-missing`, `no-baseline-depth-boost`
   - Record `escalated`, `outcome`, `selected_count`, `contexts.selected_count`, and
     `duration`. Escalation by `serial-budget-exceeded` / `selection-ratio-exceeded`
     does not fail this proof, because contexts are consulted before those rules. It
     does feed step 5's follow-up decision.
3. Revert the temporary edit (`git checkout -- <file>`) and confirm `git status` is
   clean.

### 5. Record evidence and close

1. Capture the after snapshot: `just selection-health --json`. Add one concise evidence
   note: `sase bead note sase-10w.5 'EVIDENCE: …'`. Include:
   - the green Master Gate run ID and SHA
   - the green Full CI run ID, SHA, and artifact name and size, plus any rerun history
   - the refresh output
   - the proof run's manifest facts from step 4.2
   - before/after headline numbers from step 3.2 and this snapshot
2. If recent scoped runs remain dominated by `serial-budget-exceeded` or staleness rules
   (for example the proof run itself escalated on budget), add
   `sase bead note sase-10w.5 'PROPOSED FOLLOW-UP: … — <residual rule_histogram>'`.
3. Run `sase bead epic-symbols sase-10w.5`. If any entries appear, resolve each symbol
   or re-key its Justfile line to a still-open bead (the parent epic or `sase-10w.6`),
   and run `just check` on that change.
4. Only when steps 1–4 all passed on unchanged evidence:
   `sase bead close sase-10w.5 --note "<Master Gate run/SHA green; Full CI run/SHA green with sase-coverage-contexts artifact; baseline <sha12> installed; scoped proof consulted fresh baseline, no stale/missing/depth-boost rules>"`.
   Do not close `sase-10w` or `sase-10w.6`.

## Verification

- `gh run view <gate-id>` and `gh run view <full-id>` both conclude `success`, and the
  Full CI artifact listing shows `sase-coverage-contexts-<sha>`.
- `ls ${SASE_HOME:-~/.sase}/test-selection/contexts/` contains `<sha>.sqlite` (and its
  breadth sidecar).
- The manifest from step 4 meets the pass criteria, and the tree is clean afterwards.
- `just check` if and only if a repair or epic-symbol change was made.
- `sase bead show sase-10w.5` shows the evidence note and `CLOSED`. The parent epic and
  `sase-10w.6` are untouched.
