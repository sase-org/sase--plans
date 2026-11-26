---
tier: tale
title: Unblock sase-core's release cut by sizing its CI wait
goal:
  The daily sase-core release cut merges the release PR once its CI is green, instead of
  timing out on the slow macOS leg.
size: small
decisions:
  dispatch_cut:
    ask:
      After the fix lands, should the implementer propose an immediate release cut that
      ships sase-core-rs 0.45.0?
    choices:
      gate:
        Propose the dry_run=false dispatch via /sase_gate; you confirm before anything
        publishes
      defer:
        Dispatch nothing; the next daily cut or the steward's dispatch proves the fix
        within a day
    default: defer
    why:
      a cut happens within a day anyway; a manual cut spends ~75 MB of PyPI quota on
      your timing
    answer: defer
proposed_by: bbugyi200.athena.1gt
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 14:42:44
status: wip
---

# Fix sase-core's failing release cut: wait one full CI pass, plus one rerun pass

All file paths below are in the linked **sase-core** repo. Open it with
`sase repo open sase-core -r "<why>"` and work only in the path it prints. Read its
`AGENTS.md` first.

## Diagnosis

`actstat --repo sase-org/sase-core` and the run logs show this state:

**Currently red: Release-plz › Merge release PR.** Every merge-capable cut since Nov 19
has failed in that job after about 41 minutes. The cuts are the delayed `41 7 * * *`
schedule runs plus the steward's `workflow_dispatch`. Failing runs: 44569701433,
44772370047, 44919916609, 45039426088, 45249112381, 45466156083, 45672227748,
45871345431, 45876879662. Each one emits
`CORE_CUT_RED check=cargo_fmt_+_clippy_+_test_(macos-latest) … attempt=1`. The detail is
either `check timeout` or `rerun attempt did not complete before the deadline`.

- No sase-core release has shipped since v0.44.2 (Nov 9).
- Release PR #346 (`chore: release v0.45.0`) has been open since Nov 12.
- sase's release steward reports the failed cuts as `core_cut_red` and `core_lag`, so
  the sase release train is blocked too.

**Not a current problem: old CI reds on master.** Commits d0366dc … 71944fb (Nov 21–25)
failed the macOS `cargo test` step. The same 8 tests failed every time
(`agent_archive::store::tests::*named_bundle*` and
`agent_scan::index::tests::lookup::pointer_lookup_resolves_aliases_and_symlinks`). These
were deterministic macOS path bugs, fixed by fdc57d0. de47605's clippy failure was fixed
by 575d430. Master CI is green at fdc57d0 and at 76638b57 (HEAD). No action is needed.

**Root cause.** Three things combine:

1. **The merge budget is fixed and stale.** `.github/scripts/release_plz_merge.py` gives
   the release PR's checks one shared deadline, `CHECK_TIMEOUT_SECONDS = 2400` (40 min),
   and the job has `timeout-minutes: 45`. That deadline covers both the first CI pass
   and the single rerun. It was set on Oct 19 (3f0920f8). At that time the macOS leg ran
   at p50 27 / p95 36 min.
2. **The macOS `rust-checks` leg has grown.** Successful-run durations over the last 400
   CI runs:

   | Period   | p50 | p95 | max |
   | -------- | --- | --- | --- |
   | October  | 27  | 36  | 41  |
   | Nov 1–15 | 33  | 48  | 78  |
   | Nov 16+  | 51  | 61  | 61  |

   (minutes)

   Linux stays at about 7 minutes. The rust-cache restore is a full hit, so the time is
   spent compiling the workspace crates. `sase_core` check takes 9m10s on macOS versus
   59s on Linux, and the test-profile build takes 34m versus 4m. Queue time is only
   seconds to 5 minutes.

3. **Every cut restarts the release PR's CI from zero.** Each Release-plz run's
   `release-plz-pr` job force-pushes a new commit with the same tree. On Nov 26,
   0c6bee4c, f4ba72db and 29e001e7 all have tree 9d126fece2 and parent 76638b57. So a
   cut needs one entire macOS pass to finish inside 40 minutes, but the median pass now
   takes about 51. Example: f4ba72db's CI went green at 18:47, 14 minutes after its cut
   had already timed out at 18:33.

## Changes

### 1. Bound every check the cut waits on

Add `timeout-minutes` to every job in the workflows that run on `pull_request`:

- `.github/workflows/ci.yml`:
  - `rust-checks`: **120**. The max observed is 78, and this leaves headroom for the
    upward trend. Add a comment saying this cap sizes `release_plz_merge.py`'s per-pass
    budget, so the two must be raised together.
  - `release-scripts`: **15**.
  - `wheel-smoke`: **60**. Observed runs take 11–17 minutes with a warm cache; this
    leaves room for a cold release build.
- `.github/workflows/pr-title.yml`:
  - `conventional-title`: **5**.
  - `cargo-version-guard`: **10**.

### 2. Give each pass its own budget (`.github/scripts/release_plz_merge.py`)

- Set `CHECK_TIMEOUT_SECONDS = 130 * 60`. That is one full pass of the slowest PR check
  (120 min) plus 10 minutes for queueing and check registration. Say so in a comment.
- Give the rerun phase a fresh deadline instead of reusing the first pass's
  `check_deadline`. Right after the reruns are requested, compute
  `rerun_deadline = monotonic() + check_timeout`. Pass it to `wait_for_reruns` and to
  the second `wait_for_checks`. A slow first pass then cannot starve the rerun (the Nov
  19 and Nov 21 failures were starved this way). The worst case becomes two passes.
- Set `POLL_INTERVAL_SECONDS = 30`. A wait of up to 4.5 hours then makes about 500 `gh`
  calls on the shared release token instead of about 1,600.
- Keep these unchanged:
  - Outcome names, `typed_line` formats and the `GITHUB_OUTPUT` keys. sase's
    `tools/_release_watch_health.py` parses `CORE_CUT_RED check=<c> run=<u> attempt=<n>`
    and `CHECKS_NEVER_REGISTERED …` verbatim.
  - The merge safety guards pinned by `MergeSafetyGuardTests`.
  - The one-rerun policy.

### 3. Merge job timeout (`.github/workflows/release-plz.yml`)

- Set the `release-plz-merge` job to `timeout-minutes: 270`. That is at least 2 × 130
  plus 5 minutes of overhead, and stays under GitHub's 360-minute job limit.
- Fix the stale comment "(which waits on checks for up to 45 minutes)".
- Add one sentence to the job's comment block: every run re-pushes the release PR, so a
  cut waits through a full CI pass from zero, plus at most one rerun pass.

### 4. Tests (stdlib `unittest`, run by `./scripts/check.sh script-test`)

`.github/scripts/test_release_cadence.py`: add a `MergeBudgetTests` class that uses the
same text-parsing style as the existing helpers. Import `release_plz_merge` the way
`test_release_plz_merge.py` does.

- `test_every_pull_request_check_job_declares_a_timeout`: for every workflow file whose
  `on:` block contains `pull_request:` (today `ci.yml` and `pr-title.yml`), every job
  under `jobs:` has `    timeout-minutes: <int>`.
- `test_one_pass_outlasts_the_slowest_pull_request_check`:
  `CHECK_TIMEOUT_SECONDS >= (max PR-job timeout + 10) * 60`.
- `test_merge_job_outlasts_a_pass_and_one_rerun_pass`: the `release-plz-merge` job's
  `timeout-minutes * 60 >= 2 * CHECK_TIMEOUT_SECONDS + 300`.

`.github/scripts/test_release_plz_merge.py`: add three tests that use `FakeClock`.
Remember that `FakeGitHub` returns its check rounds by call index.

- `test_check_timeout_is_typed_core_cut_red_without_rerun`: checks stay pending for
  longer than the budget. Expect `CORE_CUT_RED` with detail `check timeout`, no reruns
  and no merge.
- `test_rerun_gets_a_fresh_budget_after_a_slow_first_pass`. Setup: with `timeout=10` and
  `poll_interval=1`, the first pass stays pending until about t=9 and then fails. The
  rerun attempt completes about 5 seconds later, which is past the old shared deadline
  but inside a fresh budget. Expect `MERGED` and `reruns == ["123"]`. Check that this
  test fails against the old shared-deadline code, where the result is `CORE_CUT_RED`
  with `rerun attempt did not complete before the deadline`.
- `test_rerun_that_outlasts_its_own_budget_is_typed_core_cut_red`: the rerun attempt
  never completes. Expect `CORE_CUT_RED` with that same detail, and no merge.

### 5. Docs (`docs/pypi-retention.md`, "Release cadence")

After "That updates the release PR, waits for its checks, and squash-merges it.", add a
short paragraph covering these points:

- The cut's own release-PR push restarts CI, so the merge job waits one full pass. A
  pass is 130 minutes: the slowest PR check's 120-minute cap plus slack.
- It then reruns failed jobs once and waits one more pass. The job's 270-minute timeout
  covers both.
- `test_release_cadence.py` ties these numbers to the workflow timeouts, so raise them
  together.
- A check that is slow but green now merges instead of reporting `CORE_CUT_RED`.

### 6. Follow-up task bead

Use `/sase_new_task` to file one task bead (default size `large`) about the macOS leg's
slowdown. Include the evidence above:

- The leg is 6–10x slower than Linux despite full cache hits.
- It is growing: p50 went from 27 to 51 minutes in about five weeks.

Hypothesis to measure: memory pressure on the 3-vCPU / 7 GB arm64 runner when
`--all-targets` compiles `sase_core`'s lib and lib-test concurrently. Possible levers:
`CARGO_BUILD_JOBS`, debuginfo level, and where clippy runs. Keep the macOS signal; the
ci.yml cost note already says not to drop it. This plan does not tune the leg.

## Verification

1. `./scripts/check.sh script-test`: all script tests pass, including the new ones.
2. `actionlint .github/workflows/ci.yml .github/workflows/pr-title.yml .github/workflows/release-plz.yml`:
   no findings beyond what the same command reports on unchanged master.
3. `sase tool run check` in sase-core: green.
4. Commit subject (Conventional Commits):
   `ci(release): wait a full CI pass and a fresh rerun pass before cutting`.

> [!decision] dispatch_cut = gate After the work is verified and declared, end the turn
> with a `/sase_gate` that proposes
> `gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false`. Explain
> in the gate that this merges release PR #346 and publishes sase-core-rs 0.45.0 once
> the PR's CI is green, which can take up to about 4.5 hours in the worst case.

> [!decision] dispatch_cut = defer Dispatch nothing. In the final report, give the
> dispatch command above as the way to cut early, and note that the next daily cut is
> the proof.

## Non-goals and rejected alternatives

- **Gating the cut on Linux checks only.** Rejected: it drops the macOS signal from the
  release commit, which changes versions and `Cargo.lock`.
- **Only raising the shared 40-minute deadline.** Rejected: a slow first pass can still
  starve the rerun.
- **Speeding up macOS first.** Rejected as the primary fix: the cause is not proven yet,
  and a faster leg would not remove the mismatch between a fixed budget and a CI that
  keeps growing. It is the follow-up bead instead.
- **Also out of scope:** cancelling superseded PR CI runs, changing release-plz's
  re-push behaviour, and any change to sase or its steward.
- No recorded decision governs sase-core's release cut. `decisions:ci-two-speed-split`
  governs sase's own CI; it is the pattern ci.yml's cost note points to. This plan keeps
  the macOS signal, which is consistent with it.
