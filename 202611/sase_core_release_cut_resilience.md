---
tier: tale
title: Make sase-core's daily release cut survive macOS runner starvation
goal:
  The daily Release-plz cut merges an already-green release PR instead of failing on a
  fresh, starved macOS CI run.
size: medium
decisions:
  dispatch_cut:
    ask:
      Should the agent propose one live sase-core release cut after the fix lands,
      instead of awaiting the daily cut?
    choices:
      daily_cut:
        The next scheduled cut proves the fix; no off-schedule release or extra PyPI
        quota
      dispatch:
        Agent files a /sase_gate for one live cut; v0.44.0 ships ~1 day sooner for ~75
        MB of quota
    default: daily_cut
    why:
      Matches the approved core_cut=wait precedent on sase-1pi; the daily cut verifies
      it for free
    answer: dispatch
  changelog_date:
    ask:
      Should a heal run still refresh a current release PR once per UTC day so its
      changelog date tracks the cut day?
    choices:
      keep:
        Never refresh a current PR; the date marks the last master change; no extra CI
        runs
      daily:
        First heal run of each UTC day refreshes a current PR (one extra CI run a day);
        cuts never do
    default: keep
    why:
      The date names the newest change the release holds, and each skipped refresh saves
      a macOS run
    answer: keep
proposed_by: bbugyi200.athena.18a
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 15:20:39
status: wip
---

# Make sase-core's daily release cut survive macOS runner starvation

All work happens in the linked **sase-core** repo. Open it with
`sase repo open sase-core -r "<why>"`, read its `AGENTS.md`, and work only in the
printed path. Every file path below is relative to that checkout. No Rust code, no
`sase` (Python) code and no memory notes change.

## Diagnosis (evidence gathered with `actstat` and `gh`)

`actstat -n 8 --repo sase-org/sase-core` shows two kinds of red:

1. **CI `cargo test --workspace` on macOS** at `65925ca8`, `6cd64a9c`, `f1bbb4f3` (runs
   41561831046, 41565437194, 41578286141). These are the two
   `sudo_runner::tests::etxtbsy::*` tests, already fixed by `3c2a9dac` ("gate kernel
   ETXTBSY replay tests to Linux"). Master CI has been green on both OSes since. **No
   work needed.**
2. **Release-plz → "Merge release PR"** (the daily cut) at `3c2a9dac` (run 41664032466)
   and `55532e2d` (run 41860672878). Cuts have actually failed every day since Nov 1.
   The newest tag is still `v0.43.3` (Oct 31), release PR #343
   (`chore: release v0.44.0`) is still open, and phase `sase-1r2.5` is stuck at
   `CORE_LAG` waiting for a core cut.

| Cut (run)           | Typed result / detail                                                      | Cause                                                                                                                                                                                                                                                                           |
| ------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Nov 1 (41316391437) | `CORE_CUT_RED` ubuntu, "checks remained red after one rerun"               | Real rustfmt failure on master `764317b9`, fixed by `dc119c0b`. A correct red                                                                                                                                                                                                   |
| Nov 2 (41490979725) | `CORE_CUT_RED` macOS, "rerun attempt did not complete before the deadline" | Real ETXTBSY failure (fixed by `3c2a9dac`), and then the rerun starved                                                                                                                                                                                                          |
| Nov 3 (41664032466) | `CORE_CUT_RED` macOS, "cancelled checks are not rerun"                     | The cut's fresh release-PR CI (41664195679) had its macOS job starved. The merge script refused it at once, and the starvation retry's rerun starved too                                                                                                                        |
| Nov 4 (41860672878) | `CORE_CUT_RED` macOS, "check timeout"                                      | The cut's fresh release-PR CI (41860817486) had its macOS job starved, cancelled at 15:59:12. The check suite stayed `in_progress` until 16:17:29, so the 40-minute budget ran out. The starvation retry (41865745918) logged "no red jobs in this attempt" for this failed run |

**Root cause (ours).** Every Release-plz run, the daily cut included, runs
`release-plz release-pr`. Upstream `update_pr`
(`crates/release_plz_core/src/command/release_pr/mod.rs` in release-plz) **always
force-pushes** a new commit, even when master has not moved. Consecutive heads
`6534bf66` → `0505b0b9` and `c123f419` → `3fc3dfa0` have identical trees (`c9b79bc782`,
`634bc06840`) and differ only in timestamp. Each force-push discards the release PR's
green CI and starts a new ~33-minute CI run with a macOS leg. So the cut can never merge
an already-green PR: it always waits for a brand-new CI run inside a 40-minute budget,
and any macOS delay sinks it. Without that re-push, both recent cuts would have merged:

- Nov 3: the pre-cut head `2512dc2f` had its starved CI rerun by the starvation
  workflow, and it went green at 15:53.
- Nov 4: head `6534bf66` had been green since 12:16.

**Contributing (ours):**

- The merge script refuses runner-starved checks ("cancelled checks are not rerun").
- It races with `runner-starvation-retry.yml`, which reruns the same run.
- Its 40-minute budget (45-minute job) cannot fit one starvation (always exactly 15
  minutes) plus a macOS rerun (~30–40 minutes).
- The starvation retry trusts one job listing that GitHub can serve stale.

**Not ours (out of scope):** GitHub's hosted macOS acquisition failures (5 of the last
~100 macOS jobs, all on release-PR runs between 14:58 and 18:20 UTC, each with
`runner_id: 0`, cancelled exactly 15 minutes after being queued), and the `41 7 * * *`
cron firing 7–10 hours late.

## Changes

### 1. Skip no-op release PR refreshes (the root-cause fix)

- New stdlib helper `.github/scripts/release_pr_current.py`. It decides whether the open
  release PR already reflects the commit this run checked out. Reuse
  `release_plz_merge.GitHubCLI.find_release_pr()` so the PR guard (base, author, branch
  shape, title, body footer) stays defined in one place; do not copy `PR_FILTER`.
- The PR is **current** only when all of these hold:
  - a guarded release PR exists;
  - `gh api repos/$GH_REPO/pulls/<n>/commits` returns exactly one commit;
  - that commit has exactly one parent, equal to `TARGET_SHA` (`${{ github.sha }}`, the
    commit `actions/checkout` used);
  - the PR head already has registered checks (`GitHubCLI.checks(n)` is non-empty), so a
    head whose CI never registered still gets refreshed.

  > [!decision] changelog_date = daily Add one more condition, used only when the env
  > var `REFRESH_STALE_DAY=true`: the head commit's committer date is on today's UTC
  > day. Set `REFRESH_STALE_DAY: ${{ github.event.schedule == '23 */6 * * *' }}` on the
  > step, so only heal runs refresh a stale-dated PR and cuts, pushes and dispatches
  > never do. Pin this wiring in the helper's workflow test.

- Write `current=true|false` and a one-line `reason=` to `$GITHUB_OUTPUT`, the step
  summary and stdout. **Fail open:** catch every exception, report `current=false` with
  the reason, and always exit 0, so any doubt falls back to today's behaviour (run
  release-plz).
- In `.github/workflows/release-plz.yml`, job `release-plz-pr`:
  - Right after checkout, add a step `id: current` with `continue-on-error: true`, env
    `GH_TOKEN: ${{ secrets.SASE_RELEASE_TOKEN || secrets.GITHUB_TOKEN }}`,
    `GH_REPO: ${{ github.repository }}` and `TARGET_SHA: ${{ github.sha }}`, running
    `python3 .github/scripts/release_pr_current.py`.
  - Gate the toolchain, rust-cache and `Run release-plz` steps on
    `steps.current.outputs.current != 'true'`. Use `!=` so that a missing output also
    runs release-plz.
  - Leave the job-level `if:` unchanged. `test_release_cadence.py` pins it, and the PR
    still picks up every master commit, because a push always moves `github.sha`.
- Effect: a push still refreshes the PR, while heal runs and the cut on an unchanged
  master no longer force-push. The cut then merges the PR's existing green head, or
  waits on or reruns that head's existing CI. This also removes four CI runs a day (each
  with a macOS leg) that heal runs triggered for nothing.

### 2. Merge script: rerun starved checks and coordinate with other rerunners

In `.github/scripts/release_plz_merge.py`:

- Add `job_info(job_id)` to `MergeGitHub` and `GitHubCLI`, using
  `gh api repos/$GH_REPO/actions/jobs/<id>`. Add `job_id_from_link()`, which parses
  `/job/(\d+)` from the check link.
- A `cancel`-bucket check whose job is starved becomes rerunnable through the existing
  `run_context` path. Starved means the same predicate as the starvation workflow:
  import `rerun_starved_jobs.is_starved`, do not copy it. Any other cancelled check, or
  a run whose own conclusion is `cancelled`, is still refused with "cancelled checks are
  not rerun". The existing `test_cancelled_run_is_never_rerun` must keep passing.
- **In-flight rerun by someone else.** The red-check classification happens after every
  check reported complete. If a run's status there is not `completed`, another actor
  (normally `runner-starvation-retry.yml`) already reran it. Record it as a requested
  rerun, with its baseline attempt equal to the current attempt − 1, and wait for it. Do
  not refuse it.
- **Rerun request fails.** If `rerun_failed` raises, re-read `run_info`. If the attempt
  advanced past the recorded one, or the status is not `completed`, treat it as an
  in-flight rerun and wait. Otherwise refuse as today.
- Keep "one rerun round per cut". Keep every `MergeResult` outcome and the
  `CORE_CUT_RED check=… run=… attempt=…` typed line byte-compatible: sase's
  `tools/_release_watch_health.py` (`_TYPED_CUT_RE`) and `sase_core::release_steward`
  parse it.

### 3. Give the cut a budget that fits one starvation plus a macOS rerun

- `CHECK_TIMEOUT_SECONDS` 2400 → 6600 (110 minutes). The comment should state the
  arithmetic: an in-flight CI run (~35 minutes) + one starvation (15 minutes) + one
  macOS rerun (~35 minutes) + GitHub check-suite lag (observed 18 minutes).
- `release-plz-merge` `timeout-minutes` 45 → 120. Update the job comment ("waits on
  checks for up to 45 minutes").

### 4. Starvation retry: re-read stale job lists and tolerate a concurrent rerun

In `.github/scripts/rerun_starved_jobs.py`:

- When `RUN_CONCLUSION` is `failure` but the listing shows **no red job**, GitHub served
  an inconsistent view (Nov 4, both 41865745918 and 41867495937). Re-list up to
  `RELIST_ATTEMPTS = 5` times, `RELIST_INTERVAL_SECONDS = 60` apart, with an injectable
  sleep, then decide again. If there is still no red job, print
  `::warning::run concluded failure but no red job was visible after N reads` and
  exit 0. The 10-minute job timeout covers this.
- Add `run_info(run_id)` (`gh run view <id> --json attempt,status`). If
  `gh run rerun --failed` fails but the run's attempt has advanced past `RUN_ATTEMPT`,
  or its status is not `completed`, print a `::notice::` that it was already rerun and
  exit 0. Otherwise exit 1 as today.
- Keep `MAX_ATTEMPTS = 2` and the "every red job starved" rule unchanged.

### 5. Comments and docs

- `release-plz.yml` header: explain why a current release PR is not refreshed, citing
  the tree-identical force-push evidence and the upstream `update_pr` behaviour. Keep
  the daily-cadence text that `DecisionRecordTests` pins.
- `runner-starvation-retry.yml` header: mention the relist and concurrent-rerun
  tolerance.
- `docs/pypi-retention.md`:
  - "Release cadence": change the "Updates release PR" column to "Refreshes release PR
    (only when master moved)" and add a short paragraph on why.
  - "Runner starvation": the merge step now reruns starved checks or waits on another
    actor's rerun, and the retry re-reads stale job lists.
  - "Troubleshooting": one line per `CORE_CUT_RED` detail ("check timeout", "cancelled
    checks are not rerun", "rerun attempt did not complete before the deadline", "checks
    remained red after one rerun") with what each means now.
- Never edit `version` fields or `CHANGELOG.md` (release-plz owns them).

## Tests (stdlib `unittest` under `.github/scripts`, run by `./scripts/check.sh script-test`)

- **New `test_release_pr_current.py`:**
  - Fake-GitHub unit tests: current, wrong parent, two commits, no release PR, no
    registered checks, a gh error fails open to `current=false`, and the
    `$GITHUB_OUTPUT` lines.
  - Workflow-text wiring tests in the style of `test_release_cadence.py`: the
    `id: current` step runs the helper with `continue-on-error: true`, and the
    release-plz step's `if:` is `steps.current.outputs.current != 'true'`.
  - A test that the helper gets the PR from `release_plz_merge` and never defines its
    own filter.
  - Under `changelog_date = daily` only: stale-day and heal-only wiring tests.
- **`test_release_plz_merge.py`:**
  - `FakeGitHub` gains `job_info`, whose default is a non-starved job, so existing tests
    keep their meaning.
  - New tests: a starved cancelled check is rerun and then merged; a rerun already in
    flight is waited on with no `rerun_failed` call and then merged; a failing
    `rerun_failed` with an advanced attempt is waited on, not refused; a failing
    `rerun_failed` with no advance is still `CORE_CUT_RED`.
  - A wiring test: the merge job's `timeout-minutes × 60` exceeds
    `CHECK_TIMEOUT_SECONDS`, and `CHECK_TIMEOUT_SECONDS ≥ 6000`.
- **`test_rerun_starved_jobs.py`:**
  - A relist that later shows a starved job reruns it.
  - A relist that never shows a red job warns, exits 0 and does not rerun.
  - A rerun error after another actor's rerun exits 0.
  - Adapt `test_rerun_failure_exits_1_with_error` so that its fake reports no advanced
    attempt.

## Verification

1. Iterate with `./scripts/check.sh script-test` from the sase-core checkout.
2. Run
   `actionlint .github/workflows/release-plz.yml .github/workflows/runner-starvation-retry.yml`.
3. Pass the full gate with `sase tool run check` in sase-core, not bare `just check`,
   with a tool timeout of at least 10 minutes. Any failure outside `.github/` that
   persists on rerun is pre-existing: check `sase bead list -T flake` and never weaken
   an assertion.
4. Suggested Conventional Commit subject (completion is host-owned):
   `fix(release): stop no-op release PR refreshes and rerun starved checks in the daily cut`.

## After landing

> [!decision] dispatch_cut = daily_cut Do nothing more. The next scheduled `41 7 * * *`
> cut is the live proof: it should merge #343's existing green head without a new CI
> run. `sase-1r2.5` already watches for that cut. In the final response, say how to
> confirm it: `gh run list -R sase-org/sase-core -w Release-plz` → "Merge release PR"
> succeeded, and a `v0.44.x` tag appears.

> [!decision] dispatch_cut = dispatch After the change is committed, file a `/sase_gate`
> that proposes
> `gh workflow run release-plz.yml --repo sase-org/sase-core -f dry_run=false`. The gate
> must say it runs only after the fix commit is on `origin/master` and that commit's CI
> is green on both OSes. Never dispatch directly.

## Out of scope (considered and rejected)

- **A CI `concurrency` group that cancels superseded release-PR runs.** When the
  starvation retry reruns a superseded run, that rerun would join the group and cancel
  the live head's run. Change 1 already removes most of the overlap.
- **Letting heal runs merge when a cut is owed.** That changes the cadence contract and
  the steward's "only merge-capable runs are cuts" classification from `sase-1pi.2`. It
  is a separate decision.
- **The hosted macOS capacity and the late cron.** Both are GitHub-side.
