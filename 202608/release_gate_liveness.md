---
tier: epic
status: done
title:
  Release gate liveness — a fast per-SHA master gate, a scheduled heavy lane, and a
  ci_watch allowlist
goal: "sase-org/sase ships releases again without slowing agent velocity: every master
  commit gets its own uncancelled CI run, the release gate reads only the fast per-SHA
  gate, the exhaustive suite still runs on a cadence and still guards the release, and
  ci_watch merges the release PR with the merge strategy the repository actually allows.

  "
phases:
  - id: gate
    title: Fast per-SHA master gate
    depends_on: []
    size: large
    description:
      "gate: add master-gate.yml — a per-SHA, never-cancelled push lane that runs the
      whole fast suite in balanced shards plus the lint gate, gets the Rust core from a
      SHA-keyed wheel cache instead of a 9-minute source build, and adds its README
      badge."
  - id: chop
    title: ci_watch gating allowlist, heavy-lane freshness, and merge strategy
    depends_on: []
    size: large
    description:
      "chop: teach ci_watch a gating_workflows allowlist evaluated against HEAD-scoped
      runs, add a heavy-lane green-and-fresh release condition, switch the release merge
      to the repository's allowed merge-commit strategy, and cut a bugyi-chops release."
  - id: heavy
    title: Scheduled heavy lane off the push path
    depends_on:
      - gate
    size: medium
    description:
      "heavy: make ci.yml pull-request-and-workflow_call only, add full.yml as a
      scheduled caller carrying the exhaustive matrix, retire the flake-reproducer cron
      from the master push path, repoint the coverage-contexts consumer, and add the
      second README badge."
  - id: throttle
    title: Throttle release-please to a schedule
    depends_on: []
    size: medium
    description:
      "throttle: move publish.yml's release-please and release-metadata reconciliation
      from every master push onto a schedule plus workflow_dispatch while keeping
      publishing prompt, closing the empty-rollup and generator-busy gate conditions."
  - id: green
    title: Drive the master gate green
    depends_on:
      - gate
    size: large
    description:
      "green: use the new per-commit signal to attribute and fix every failure the gate
      reports until master is durably green, filing task beads for flakes rather than
      muting them."
  - id: corepin
    title: Pin the Rust core revision CI builds
    depends_on:
      - gate
      - heavy
    size: medium
    description:
      "corepin: record a sase-core revision in-repo, build CI's core wheel from that
      revision instead of unpinned HEAD, and add a ratchet path that proposes bumps so a
      sase-core push can no longer redden sase master on its own."
  - id: config
    title: Chop configuration rollout
    depends_on:
      - gate
      - chop
      - heavy
    size: small
    description:
      "config: update the ci_watch chop's chezmoi configuration with the new gating and
      heavy-lane variables, install the released plugin, and verify a dry-run tick
      reports the expected reason."
  - id: verify
    title: Measure the acceptance criteria and ship v0.17.0
    depends_on:
      - green
      - throttle
      - corepin
      - config
    size: small
    description:
      "verify: measure every acceptance criterion against live run data, confirm the
      release gate opened and the release published, and record what remains deferred."
proposed_by: bbugyi200.athena.0ek
bead_id: sase-um
create_time: 2026-09-09 19:51:21
---

- **PROMPT:**
  [prompts/202608/release_gate_liveness.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/release_gate_liveness.md)
- **BEAD:**
  [sase-um](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/README.md)

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                            | Why                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| derives-from | [research:202608/release_gate_liveness/release_gate_liveness.md][1] | Consolidated diagnosis of the 19-day release stall, its measurements, and the R1-R7 recommendation set this plan implements with two documented overrides. |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202608/release_gate_liveness/release_gate_liveness.md

<!-- sase:links:end -->

# Plan: Release gate liveness

## 1. Problem

`sase-org/sase` has not cut a release in ~19 days. Release PR #284
(`chore(master): release 0.17.0`) has been `MERGEABLE` / `CLEAN` the whole time. The
consolidated research report linked above establishes that this is not one bug but
three, each independently sufficient to stop a release:

1. **Liveness.** `ci_watch`'s first condition asks whether the _current tip of master_
   is both settled and green. Master CI takes ~107 minutes; commits land every ~11
   minutes. The `ci-refs/heads/master` concurrency group keeps one running plus one
   pending run, so each push cancels the pending one and only ~13% of commits ever get a
   completed run.
2. **Correctness.** Master CI has produced zero green runs in the measured window. The
   failures are real, repeatable assertions.
3. **A latent merge failure.** `ci_watch.merge()` calls `gh pr merge --squash`, but the
   repository has `allow_squash_merge: false`. The first time conditions 1 and 2 are
   ever satisfied, the merge still fails.

Cancellation is the symptom the owner noticed; it is not the disease. Deleting the
concurrency block or setting `queue: max` removes the cancellations and leaves the
release exactly as stalled, because the tip would sit at the back of a growing FIFO at a
10x arrival-to-service ratio. The fix is to apply this project's own
`decisions:two-speed-verification` to CI itself.

## 2. Owner decisions this plan encodes

The owner annotated the research report. These are settled inputs, not open questions:

- **Fix all three blockers.** Not just the liveness one.
- **Two distinct README badges**, one for the fast CI mode and one for the slow mode.
- **The gate's tests must still do a good job of finding issues pre-release.** This is
  the constraint that drove Section 4's override of the report's R1 test content.
- **Merge commits, not squash.** The repository allows merge commits only
  (`allow_merge_commit: true`, `allow_squash_merge: false`,
  `allow_rebase_merge: false`), so `ci_watch` changes to `--merge`. Note for the
  implementer: SASE stitches are how _agents_ land commits by direct push; they have no
  bearing on how `gh pr merge` merges the release PR. The two are unrelated mechanisms,
  and nothing about stitches needs to change.
- **`gating_workflows` allowlist in `ci_watch`, not a `ci-full` branch.** The heavy lane
  therefore stays on the default branch and the chop names what it gates on.
- **Last-known-good release train is deferred**, but likely soon. Section 9 records what
  this plan must not foreclose.

## 3. Measurements taken while planning

Reproduced against live data on 2026-08-26 from workspace HEAD `4bce1a4f6`. An
implementing agent should not need to re-derive these, but every command is given so
they can be re-run if the numbers look stale.

**Commit cadence** — last 400 master commits over 129.4 h: mean gap 19.5 min, median
10.8 min, ~74 commits/day.

```bash
git log --format=%ct -400 origin/master | tac | awk 'NR>1{print ($1-p)/60} {p=$1}'
```

**Settled-tip fraction against that real gap distribution.** This is the number that
decides whether `ci_watch`'s exact-HEAD predicate is ever satisfiable, computed as
`E[max(0, G - L)] / E[G]`:

| Gate wall time `L` | Tip settled (% wall-clock) |
| -----------------: | -------------------------: |
|              6 min |                      74.7% |
|          **8 min** |                  **68.4%** |
|             12 min |                      57.6% |
|             15 min |                      50.9% |
|             29 min |                      33.7% |
|    107 min (today) |                      13.4% |

Note that with **per-SHA** concurrency, run _attribution_ is 100% regardless of `L` —
every commit gets its own run to completion. `L` only decides how fast you learn and how
often the tip is settled. The report's "undisturbed push" column modelled the
shared-lane world and does not apply once the gate is per-SHA.

**Current master CI outcome distribution** (`ci.yml`, branch master, last 60): 44
cancelled, 14 failure, 0 success.

**Per-job cost of one master CI run** (the most recent complete master CI failure,
2026-08-26T19:53Z; `lint` failed early, so its 2 min is a lower bound):

| Job                              |                                  Wall |
| -------------------------------- | ------------------------------------: |
| `build-core`                     | 9 min (blocks every other source job) |
| `lint`                           |                                 2 min |
| `ace-page-group-isolation`       |                                 1 min |
| `perf-floors`                    |                                 5 min |
| `visual-test`                    |                                18 min |
| `test (3.14)` (`just test`)      |                                29 min |
| `coverage-contexts`              |                                38 min |
| `test (3.12)` (`just test-cov`)  |                                45 min |
| `test (3.13)` (`just test-cost`) |                                73 min |

**`release-core-floor-smoke` runs in 63–239 s**, including a full
`uv pip install -e '.[dev]'`, binding checks, six smoke scripts, and the `contract`
marker set. That is the proof that a PyPI-installed lane costs ~1–4 minutes, and it is
what makes a fast gate credible at all.

**docs-deploy is ~6 min and green; publish is ~3 min and green.** Neither is a
bottleneck, but both attach runs to the tip SHA, which matters in Section 5.

## 4. Two findings that override the research report

The report's R1 makes two specific prescriptions that live measurement refutes. Both
overrides are load-bearing; do not silently revert to the report's text.

### 4.1 The gate cannot install `sase-core-rs` from PyPI

R1 says: _"install `sase-core-rs` from PyPI at the declared floor; do not build the Rust
core from source in the gate."_ That is not currently possible. `sase` master requires
bindings that no published in-window release provides:

```
$ uv pip install "sase-core-rs>=0.31.12,<0.32.0"   # resolves to 0.31.14
$ python tools/check_sase_core_rs_bindings
sase_core_rs 0.31.14 is missing 2 of 396 required binding(s):
  bead_note_edit
  bead_note_remove
```

The declared window (`>=0.31.12,<0.32.0`) is also behind the published series entirely —
PyPI's latest `sase-core-rs` is `0.32.7`. A PyPI-floor gate would be red on arrival.

**Instead**: the gate keeps a source-built core wheel but stops paying 9 minutes for it,
by caching `dist/` under a key derived from the sase-core revision. See Phase `gate`.

### 4.2 The diff-scoped selector is not the gate's test content

R1 proposes `just test-scoped` as the gate's test leg, and the report's open decision #2
asks whether the selector resolves a sensible baseline for a pushed commit. It does
resolve one — but the selection escalates essentially always. Replaying the last 25
master commits through the selector with the existing measurement tool:

```bash
./.venv/bin/python tools/selection_backtest --limit 25 --json
```

**10 of 10 measurable commits escalated to the full suite**, under both the
`closure-only` and `closure+contexts` arms, via `src-data-asset` (4),
`serial-budget-exceeded` (5), `context-baseline-stale` (5), `selection-tooling` (1),
`packaging-config` (1), and `rename-or-delete` (1). Recall was a perfect 1.0 precisely
_because_ everything escalated. A gate whose fast path is "run everything anyway" is not
a fast gate.

Three further reasons the scoped lane is the wrong CI primitive:

- `just test-scoped` is serial by design and negotiates a host suite-gate lease. That
  machinery exists to keep parallel agents from starving one host; it is meaningless on
  an ephemeral runner and would make the gate _slower_, not faster.
- Its recall depends on a per-test coverage-contexts baseline and a host-local timing
  table, neither of which exists on a fresh runner.
- `tests/test_github_actions_ci.py::test_ci_never_runs_the_diff_scoped_test_lane`
  encodes the deliberate policy that "CI itself must never be the thing that skips
  tests." That policy is correct and this plan keeps it.

**Instead**: the gate runs the **entire fast suite** (`just test`), split across
balanced shards so wall time, not coverage, is what shrinks. This is also the direct
answer to the owner's "make sure these tests still do a pretty good job of fleshing out
issues pre-release" — the gate skips no test at all. What the gate omits is the _other_
lanes (the 3.13/3.14 matrix legs, `visual-test`, `coverage-contexts`, `perf-floors`, the
contention harness), and those move to the scheduled heavy lane, whose freshness the
release gate then requires (Phase `chop`).

## 5. Architecture

```
push to master ──▶ Master Gate (master-gate.yml)          per-SHA, never cancelled
                     ├── core-wheel   (cache hit ~30 s, miss ~9 min)
                     ├── lint         (~4 min)
                     └── test 1..N    (whole fast suite, balanced shards)
                   target p50 wall ≤ 8 min

every 2 h    ──▶ Full CI (full.yml → calls ci.yml)         exhaustive, master
                     build-core, test 3.12/3.13/3.14, visual-test,
                     coverage-contexts, perf-floors, ace-page-group-isolation,
                     contention harness

pull request ──▶ CI (ci.yml)                               unchanged, per-ref latest-wins

ci_watch tick (5 min)
   condition 1  ← Master Gate runs for the *exact* default-branch HEAD   (gating_workflows)
   condition 1b ← newest Full CI run on master is green and < N hours old (heavy freshness)
   conditions 2-8 unchanged
   merge         ← gh pr merge --merge --match-head-commit
```

Two properties make this work that are worth stating explicitly:

**The gate condition is evaluated HEAD-scoped, not through actstat's settled-commit
record.** `actstat` only emits a commit once _every_ workflow run attached to that SHA
has completed (`select_settled_commits` in `src/github.rs` requires
`runs.iter().all(run_is_completed)`). Because `docs-deploy`, `publish`, and the
scheduled heavy lane all attach runs to master SHAs, filtering actstat's `runs` array by
an allowlist would not help: the commit would not appear as settled in the first place.
So `gating_workflows` must be evaluated from `ci_watch`'s own HEAD-scoped query
(`GitHubReader.head_ci_evidence`, `repos/{repo}/actions/runs?head_sha=...`), which is
already implemented and bounded. The actstat sweep keeps driving the _notification_ path
unchanged.

**The heavy lane stays on the default branch,** per the owner's decision. It will
therefore attach a ~107-minute run to whatever tip SHA is current when the schedule
fires, and that SHA will not settle in actstat for the duration. That is accepted: it
delays incident notification for one SHA every two hours, and it does not touch the
release gate, which no longer reads actstat for condition 1.

## 6. Phases

### Fast per-SHA master gate

Add `.github/workflows/master-gate.yml`, `name: Master Gate`.

- `on: push: branches: [master]` plus `workflow_dispatch`.
- `concurrency: { group: master-gate-${{ github.sha }}, cancel-in-progress: false }`.
  Per-SHA means a later push can never cancel an earlier commit's gate, which is the
  entire point. Every job carries `timeout-minutes` (≤ 20) so a wedged job cannot hold
  an org runner slot for GitHub's 6-hour default.
- `permissions: contents: read`.

**`core-wheel` job.** Resolve the sase-core revision the gate will build
(`git ls-remote https://github.com/sase-org/sase-core HEAD` — one call, ~2 s; Phase
`corepin` later replaces this with an in-repo pin). Restore an `actions/cache` entry
keyed on that revision plus the maturin/toolchain identity, holding the `dist/`
directory `build-core` produces today (abi3 wheel, `sase-xprompt-lsp`,
`sase-core-sha.txt`). On a hit, skip the Rust toolchain, checkout, and maturin build
entirely and just re-upload the artifact. On a miss, build exactly as `ci.yml`'s
`build-core` does today and save the cache. sase-core moves a few times a day against
~74 sase commits/day, so the steady state is a hit.

**`lint` job.** `needs: core-wheel`, `./.github/actions/setup-sase` at 3.12, then the
same command set `ci.yml`'s `lint` job runs: sidecar bootstrap, `sase init memory` /
`sase skill init`, `just fmt-py-check`, `just fmt-md-check`, `just lint`,
`just validate`, `just validate-committed-plans`, `just build-check`. Today's `lint` job
reaches a conclusion in ~2 min, but it has been failing early, so treat that as a lower
bound and budget ~4 min including setup.

**`test` job, sharded.** `needs: core-wheel`,
`strategy: { fail-fast: false, matrix: { shard: [1, ..., N] } }`, Python 3.12 only, each
leg running its share of `just test` — the same fast suite, no coverage, no cost plugin,
no visual snapshots.

Implementation notes for the shard split, in preference order:

1. Have the heavy lane publish the per-file duration table
   (`tests/_test_selection_timings`) as a run artifact, and have the gate download the
   most recent one and bin test files into N shards by descending duration
   (longest-processing-time-first, which is within 4/3 of optimal). This reuses
   machinery that already exists rather than inventing a second timing source.
2. Fall back to a stable hash of the test file path modulo N when no table is available,
   so a first run or an expired artifact degrades to a rough split rather than failing.

Add the shard mode to `tools/run_pytest` rather than assembling pytest arguments in
YAML, so the split is testable in-repo and `SASE_JUST_INVOCATION_DIR` handling stays in
one place. Reject a shard index outside `1..N` loudly.

**Choosing N.** `just test` is 29 min wall on a 4-vCPU runner, so N=4 gives ~7.3 min of
test time plus ~2 min of setup, and N=6 gives ~4.8 min. Start at N=4, measure the p50
gate wall over at least 20 real runs, and raise N only if p50 exceeds 8 min. Budget
guardrail: keep the gate under **60 job-minutes per commit**. At N=4 that is ~37
job-min, or ~2,800 job-min/day against the account's 28,800 — about 10%, against the
~62% that verifying every commit at today's 234 job-min would cost.

**Tests and docs.** Extend `tests/test_github_actions_ci.py` with the gate's invariants:
per-SHA concurrency group with `cancel-in-progress: false`, a `timeout-minutes` on every
job, the shard matrix covering `1..N` exactly once, and — importantly — keep
`test_ci_never_runs_the_diff_scoped_test_lane` passing and extend it to cover
`master-gate.yml` too. The gate must never run `test-scoped`; Section 4.2 is the reason,
and the assertion is what keeps a future agent from "optimizing" the gate back into a
selection heuristic.

This phase is deliberately **additive**: `ci.yml` keeps its master push trigger until
Phase `heavy` removes it, so the gate removes no coverage while Phase `green` is using
it. The temporary overlap is nearly free, because master `ci.yml` runs keep being
cancelled by the existing concurrency group and a run cancelled while pending starts no
jobs. Do not assert `ci.yml`'s trigger set here; that assertion belongs to Phase
`heavy`.

**README.** Add the fast-mode badge:
`[![Master Gate](https://github.com/sase-org/sase/actions/workflows/master-gate.yml/badge.svg?branch=master)](...)`.
The second badge lands in Phase `heavy` when `full.yml` exists; the existing `CI` badge
stays and keeps meaning per-PR CI.

**Acceptance.** After ~24 h:
`gh run list --workflow=master-gate.yml --branch=master --limit 50` shows zero
`cancelled`; p50 wall ≤ 8 min; ≥ 90% of master commits in a 24-hour window have a
completed gate run; PR CI queue wait stays ≤ 1 min median (the canary that the gate has
not reintroduced contention).

### ci_watch gating allowlist, heavy-lane freshness, and merge strategy

All in `bbugyi200/bugyi-chops`, opened with `/sase_repo`
(`sase repo open gh:bbugyi200/bugyi-chops`). Three changes plus a release.

**(a) Merge strategy — the latent blocker.** `GitHubReader.merge()` hard-codes
`--squash`. Add a `merge_method` config var accepting `merge | squash | rebase`,
defaulting to `merge`, validated in `Config.from_invocation` like the other vars, and
pass the corresponding flag. **Keep `--match-head-commit`** — it is the race protection
that stops the chop merging a head it did not evaluate. Update the README, which
describes the chop as squash-merging in at least three places, and the chop's module
docstring.

**(b) `gating_workflows`.** A config var naming the workflows whose HEAD-scoped result
_is_ the default-branch gate for a release repository. When it is empty (the default),
behavior is exactly as today, so the chop stays safe for the other repos it sweeps. When
it is set for a repo, condition 1 of `plan_release_merge` is decided by a new evaluation
over `GitHubReader.head_ci_evidence(repo, head.sha)` restricted to runs whose workflow
name is in the allowlist:

- every allowlisted workflow has at least one completed run for HEAD and all of them are
  green (`GREEN_CONCLUSIONS`) → `GREEN`;
- any allowlisted run is red, or is `cancelled` with a red job → `RED`, with the same
  failing-job evidence shape the existing code builds;
- any allowlisted run is in flight, or an allowlisted workflow has no run for HEAD →
  `PENDING`, with a distinguishable reason such as `gating_workflow_missing` versus
  `gating_workflow_in_flight`.

Fail closed on anything unrecognized, matching the module's existing posture. The
existing actstat-driven `classify_repo` / `decide_repo` path continues to drive
notifications untouched — this change adds a gate evaluation, it does not replace the
sweep. Respect the existing per-tick query bounds (`MAX_HEAD_EVIDENCE_REPOS_PER_TICK`,
`MAX_HEAD_RUNS`, `MAX_HEAD_JOBS`); the allowlist path adds at most one runs query and a
bounded jobs query per release repo per tick.

**(c) Heavy-lane freshness.** The gate above deliberately does not look at the
exhaustive suite, so add the safeguard the research report's risk table calls for, as a
real condition rather than an advisory note. Two vars: `heavy_workflows` (names, default
empty) and `heavy_max_age_hours` (default e.g. 6). When set, a release may only merge
if, for each named workflow, the newest **completed** run on the default branch is green
and finished within the window. New failure reasons: `heavy_lane_not_green` and
`heavy_lane_stale`. Evaluate it from the existing `_workflow_runs(repo, branch)` query
so no new API surface is needed. This is what makes "the gate's tests still do a good
job pre-release" true end to end: the fast gate proves _this commit_ is sound, the
freshness condition proves the exhaustive suite was green on a recent master.

**Tests.** `tests/test_ci_watch.py` is the contract. Add coverage for: allowlist green /
red / in-flight / missing-run; allowlist empty preserving today's behavior exactly;
heavy-lane green, red, stale, and missing; each merge method emitting the right flag
with `--match-head-commit` retained; and config validation rejecting an unknown
`merge_method` or a non-list `gating_workflows`. Run the repo's own `just check`
equivalent before releasing.

**Release.** bugyi-chops releases are tag-driven: bump `version` in `pyproject.toml`,
push the matching `v<version>` tag, and let `publish.yml` ship it. Do not roll the
configuration out here — Phase `config` owns that, so the chop is on PyPI before any
host config references the new vars.

### Scheduled heavy lane off the push path

- `ci.yml`: remove `push: { branches: [master] }` and the `schedule: '17 6 * * *'` cron.
  Add `workflow_call`. Keep `pull_request` and every job body as-is; the `concurrency`
  block can simplify to per-ref latest-wins now that master is not a special case.
- New `.github/workflows/full.yml`, `name: Full CI`: `on: schedule` every 2 hours plus
  `workflow_dispatch`, one job that `uses: ./.github/workflows/ci.yml` with
  `secrets: inherit`, and `concurrency: { group: full-ci, cancel-in-progress: false }`
  so two scheduled runs never overlap. A called workflow evaluates the `github` context
  of the _caller_, so `ci.yml`'s existing `if:` conditions keep working:
  `github.event_name` reads `schedule`, `github.ref` reads `refs/heads/master`,
  `coverage-contexts` runs, `contention-test` runs (this is where the deliberate flake
  reproducer belongs — off the master push path, where it currently stamps `failure` on
  the tip the release gate reads once a day), and the release-please exclusions are
  inert.
- `tools/fetch_coverage_contexts` hard-codes `WORKFLOW = "ci.yml"`. Because a called
  workflow's jobs belong to the _caller's_ run, the `sase-coverage-contexts-<sha>`
  artifact will now attach to the `full.yml` run. Repoint the constant and re-verify the
  tool end to end against a real run before declaring the phase done — a silently broken
  contexts fetch degrades every agent's `just check`.
- Check the other `ci.yml` references surfaced by
  `grep -rn "ci\.yml" tools/ tests/ docs/`:
  `tests/test_last_workflow_set_status_diagnostics.py`,
  `tests/test_sdd_canonical_layout.py`, `tests/test_justfile_lint.py`,
  `tests/test_ci_bootstrap_sidecars_tool.py`, and `docs/rust_backend.md`. Update the
  ones that assert the master push path; leave the ones that assert PR behavior.
- README: add the slow-mode badge for `full.yml`, completing the owner's two-badge
  requirement. Add one sentence under the badges naming what each lane covers, so the
  distinction is legible to a reader who has not read this plan.
- `tests/test_github_actions_ci.py`: assert `ci.yml` no longer carries a master push
  trigger or a cron, that `full.yml` calls it, and that the heavy job set is present
  exactly once across the two files.

**Accept deliberately:** heavy-lane failures will not page through `ci_watch`'s
notification path while the tip they attach to is unsettled. Enable GitHub's native
workflow-failure notifications for the repository, and file a task bead (via
`/sase_new_task`) to extend `ci_watch` with a watched-workflow notification list.

### Throttle release-please to a schedule

`publish.yml` currently runs release-please, and its `sync-release-metadata` reconciler,
on every master push. At ~74 pushes/day that keeps the release branch head moving
constantly, which is what makes `ci_watch` condition 6 (`release_pr_empty_rollup`) fail
~11% of ticks and condition 8 (`release_generator_busy`) fail ~35–45% of ticks.

Move the release-please job and `sync-release-metadata` to `on: schedule` every 2–4
hours plus `workflow_dispatch`, and keep the _publish_ half responsive: either leave the
build/publish jobs on push (gated as now on `release_created`), or have the scheduled
release job dispatch the publish path directly. Preserve the existing three-attempt
retry-with-backoff structure and the `publish_existing` dispatch input.

Take care with the job graph: `build` currently keys off
`needs.release.outputs.release_created`, and `sync-release-metadata` is
`if: always() && github.event_name == 'push'`. Both conditions have to be re-derived for
the new trigger set, and
`tests/test_github_actions_ci.py::test_publish_depends_on_floor_exact_install_smoke` and
`::test_publish_sync_release_metadata_applies_ratchet_before_lock_refresh` must keep
passing. Accepted cost: a release may be cut up to one schedule interval late;
`workflow_dispatch` covers a manual cut.

### Drive the master gate green

The second blocker. Master CI has produced zero green runs; the most recent completed
failures span `visual-test` (present in 8/8 at the time of the report),
`test (3.12/3.13/3.14)`, `coverage-contexts`, `perf-floors`, and — in the two most
recent runs measured while planning — `lint`. The red set is moving, so do not work from
a static list.

Method, which is only available once Phase `gate` has landed:

1. Read the newest `Master Gate` failures. Each one names exactly one commit, because
   the gate is per-SHA. That is the attribution the old lane could not give.
2. Fix causes in the order the gate reports them, newest first, so the tip converges
   rather than the history.
3. Reproduce locally with `just check`, and gate landing on `just check-full` run
   through `/sase_monitor` with the `TESTING` / `TESTED` status pair — it routinely
   outruns a single agent turn.
4. For anything that fails and then passes on an unchanged tree, file a `flake` task
   bead through `/sase_new_task` rather than muting it, and record it in the existing
   reproducible-flake baseline machinery. `just selection-health --fail-on-new-flake`
   already tracks this.
5. Failures that belong to the exhaustive lane rather than the gate (`visual-test`
   especially) still block the release through the heavy-lane freshness condition from
   Phase `chop`, so they are in scope for this phase even though the gate does not run
   them.

Exit condition: `Master Gate` is green on the tip for a majority of samples taken 10
minutes apart over an hour, and the newest `Full CI` run is green.

### Pin the Rust core revision CI builds

`build-core` checks out `sase-org/sase-core` at unpinned HEAD. Two consequences: a push
to sase-core can redden sase master with no sase commit involved, and two runs of the
same sase SHA are not reproducible. Both directly attack a gate whose whole premise is
"this SHA is green."

- Record a sase-core revision in-repo (a small dedicated file is easiest to read from
  both workflows and `tools/`; do not overload `pyproject.toml`, which release-please
  rewrites).
- Point both the gate's `core-wheel` job and `ci.yml`'s `build-core` at that revision
  instead of HEAD, which also makes the gate's cache key deterministic rather than
  "whatever sase-core's HEAD was at that moment."
- Add a ratchet path that proposes bumps — a scheduled job or a chop that opens a bump
  when sase-core moves — and make the failure mode legible: when sase master needs a
  binding the pinned revision lacks, `tools/check_sase_core_rs_bindings` should say so
  and name the bump as the remedy. Coordinate with `tools/ratchet_core_window` and the
  existing `core_dependency_window_ratchet` research rather than building a second,
  divergent ratchet.
- Keep `tools/probe_core_floor`'s advisory role intact; this phase pins the _source_
  revision, it does not change the published dependency window.

### Chop configuration rollout

Open the chezmoi repo with `/sase_repo` and update the `ci_watch` chop block in
`sase_athena.yml` (source path under chezmoi, not the materialized
`~/.config/sase/sase_athena.yml`):

```yaml
vars:
  # ...existing vars unchanged...
  merge_method: merge
  gating_workflows: ["Master Gate"]
  heavy_workflows: ["Full CI"]
  heavy_max_age_hours: 6
```

Match the released chop's actual var names and shapes — if Phase `chop` chose per-repo
mappings rather than flat lists, mirror that here. Refresh the chop's prose
`description`, which currently says "guarded squash merge". Install the released plugin
version, then verify with a dry-run tick (`SASE_CHOP_DRY_RUN` left unset or `1`) that
the chop reports a sane reason — `eligible`, or a _specific_ condition such as
`heavy_lane_stale`, and never `default_branch_not_green` for a green tip.

### Measure the acceptance criteria and ship v0.17.0

Confirm each of these against live data and record the numbers:

1. `gh run list --workflow=master-gate.yml --branch=master --limit 50` shows **zero**
   `cancelled` conclusions.
2. Median `Master Gate` wall time ≤ 8 minutes over 50 consecutive runs.
3. ≥ 90% of master commits in a 24-hour window have a completed gate run.
4. `ci_watch` reports a reason other than `default_branch_not_green` at least once per
   day, and reaches `eligible` at least once.
5. A `gh pr merge` against the repository's allowed strategy succeeds — proven by the
   actual release merge, not a dry run, once conditions 1–8 pass.
6. PR CI queue wait stays ≤ 1 minute median after the gate ships.
7. `v0.17.0` is tagged and published to PyPI.

If the gate opens but the merge fails, the failure is in Phase `chop`'s work and takes
priority over everything else — that is the blocker the whole plan exists to remove.

## 7. Risks and safeguards

| Risk                                                            | Safeguard                                                                                                                                                                                            |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The fast gate misses a regression the heavy lane would catch    | The gate runs the _entire_ fast suite, not a selection; the heavy lane runs every 2 h; the release gate additionally requires a recent green `Full CI`; `just check-full` still gates epic landings. |
| A flaky test reds the gate and re-closes the release            | The gate is per-SHA and fast, so the next commit re-establishes a green tip within minutes. Track flakes as `flake` task beads and via `just selection-health --fail-on-new-flake`; do not mute.     |
| Sharding is unbalanced and one leg blows the budget             | Bin by measured durations with a hash fallback; enforce the ≤ 60 job-min/commit guardrail and the ≤ 8 min p50 acceptance criterion; raise N rather than dropping tests.                              |
| The gate reintroduces org-wide runner contention                | ~37 job-min/commit against 28,800/day is ~10%. Watch PR CI queue wait as the canary — it must stay under 1 min median.                                                                               |
| sase-core moves and every gate run pays a 9-minute cache miss   | Misses cluster and then stop; Phase `corepin` removes the dependency on sase-core's HEAD entirely.                                                                                                   |
| Heavy-lane failures go unnoticed once off the notification path | Native GitHub workflow-failure notifications, plus a task bead to give `ci_watch` a watched-workflow list.                                                                                           |
| Heavy-lane freshness window is too tight and blocks releases    | It is a config var; widen `heavy_max_age_hours` rather than removing the condition. A persistently red heavy lane _should_ block a release.                                                          |
| Throttled release-please delays a release                       | Up to one schedule interval, against a 19-day stall. `workflow_dispatch` covers manual cuts.                                                                                                         |
| Merge commits change master's history shape                     | Accepted and chosen by the owner. Master is currently 100% single-parent; a merge commit per release is the only change. `commit_change_set` uses first-parent (`sha^`) and is unaffected.           |

## 8. Explicitly rejected

Recorded so a later agent does not relitigate them: `queue: max` on the master group
(removes cancellations, leaves the gate shut); per-SHA concurrency on the _full_
`ci.yml` (~62% of budget and the tip still settles ~14% of the time); deleting the
master concurrency block (reproduces the starvation that stalled the sase-core v0.6.0
publish); `cancel-in-progress: true` on master (already tried; no master run reaches a
terminal conclusion); a GitHub merge queue (requires PR-ifying ~74 direct-push
commits/day and its throughput ceiling is governed by the same arrival/service ratio);
bigger or self-hosted runners (buys capacity, not latency); verifying every Nth commit;
waiting for a quiet period; slowing the agents down; and dropping the default-branch
guard entirely.

## 9. Deferred

- **Last-known-good release train** — changing `ci_watch` to cut the release from the
  newest green commit rather than the tip. The owner has deferred this but expects to
  build it soon. Nothing in this plan should foreclose it: keep the gate's per-SHA
  result queryable by SHA, and keep `gating_workflows` a property of the _evaluation_
  rather than something baked into the HEAD predicate, so the same allowlist can later
  answer "which SHA was last green?" instead of only "is HEAD green?".
- **R7, the cost of `test (3.13)` and `coverage-contexts`** — `just test-cost` runs 73
  min against `just test`'s 29 min for the same suite, and `test-contexts` is 38 min.
  That is heavy-lane budget, not gate budget, so it does not block this plan. File it as
  a task bead.
- **A decision record for CI's two-speed split.** This plan changes a real policy: CI is
  no longer the thing that runs everything on every push. That is worth an immutable
  decision record alongside `decisions:two-speed-verification`. SASE memory must not be
  edited without the user's explicit approval, so propose it as a `memory` task bead
  through `/sase_new_task` rather than writing it.
