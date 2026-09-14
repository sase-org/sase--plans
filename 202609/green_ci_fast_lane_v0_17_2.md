---
tier: epic
title: Green CI, a fed selector, and the v0.17.2 release
goal: "Master Gate and Full CI pass on the master tip, the diff-scoped test lane stops
  escalating on stale coverage baselines so `just check` gets fast again, and sase
  v0.17.2 reaches PyPI through the ci_watch-owned release path.

  "
phases:
  - id: red-lane-fixes
    title: Fix the deterministic red-lane test failures at master tip
    depends_on: []
    size: medium
    description:
      "red-lane-fixes: repair the stale Justfile-lint assertion, the
      require_rust_binding literal-scan violations in managed_tmp_reaper, and any Master
      Gate / Full CI test failure still deterministic at the worker's HEAD, coordinating
      with the active epics that own those domains."
  - id: hermetic-git-identity
    title: Test-owned git identity for every test-created repository
    depends_on: []
    size: medium
    description:
      "hermetic-git-identity: export a fixture-owned git config and author/committer
      identity for the whole suite plus a redirected-HOME guard test, so the CI 'Author
      identity unknown' class can never silently return."
  - id: visual-goldens
    title: Rebaseline the drifted ACE PNG goldens
    depends_on:
      - red-lane-fixes
    size: medium
    description:
      "visual-goldens: re-run the visual suite, audit each mismatching golden against
      the landed UI commits that changed it, and accept only intentional pixel changes."
  - id: budget-recalibration
    title: Re-derive the scoped lane's serial-budget crossover from current evidence
    depends_on: []
    size: small
    description:
      "budget-recalibration: measure the governed full lane's current wall clock at the
      worker widths agents actually get, and update or deliberately confirm the 232 s
      crossover constant with recorded provenance."
  - id: green-ci-and-baseline
    title: Observe green CI and feed the selector a fresh baseline
    depends_on:
      - red-lane-fixes
      - hermetic-git-identity
      - visual-goldens
      - budget-recalibration
    size: medium
    description:
      "green-ci-and-baseline: watch Master Gate go green on the tip, dispatch and watch
      a fully green Full CI whose coverage-contexts job uploads a .coverage baseline,
      then install that baseline locally and verify the scoped lane consults it instead
      of depth-boosting."
  - id: release-v0-17-2
    title: Ship sase v0.17.2 to PyPI through ci_watch
    depends_on:
      - green-ci-and-baseline
    size: small
    description:
      "release-v0-17-2: verify ci_watch submits release PR #299 once its gating
      conditions clear (never hand-merge), watch the Publish workflow create tag v0.17.2
      and upload to PyPI, and prove the published package installs."
proposed_by: bbugyi200.athena.0kh
create_time: 2026-09-14 09:06:43
status: wip
---

- **PROMPT:**
  [prompts/202609/green_ci_fast_lane_v0_17_2.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/green_ci_fast_lane_v0_17_2.md)

# Plan: Green CI, a fed selector, and the v0.17.2 release

## Why this shape

This epic implements steps 1 and 2 of the research decision report — read it first:

```sh
sase artifact read research:202609/test_feedback_speedup_decision.md "Governing evidence for the red-lane repair and selector-feed program"
```

Its central finding, re-verified on 2026-09-14: the slow agent feedback loop is a
plumbing failure, not an architecture failure. Red CI means the `coverage-contexts` job
never publishes a per-test coverage baseline; every scoped `just check` then fires
`context-baseline-stale` plus `no-baseline-depth-boost`, selections inflate, the 232 s
serial budget trips, and the "fast" lane escalates into the full lane. On this host's
durable health store (`tools/selection_health --json`, 2026-09-14): 2,347 scoped runs,
66% escalation rate, all 839 baseline-consulting runs stale. Fixing CI and feeding the
selector is the speedup; verification architecture does not change. The leak-detector
optimization program (the report's step 3) is deliberately deferred — see "Deferred work
for the land agent".

Fresh evidence pinned for workers (the tree moves hourly; always re-verify at your own
HEAD before touching anything):

- Master Gate: every completed run on 2026-09-14 failed (e.g. runs 34840740636,
  34843463275, 34843655120). The failure set is shrinking commit-by-commit as active
  epics land fixes; at `f86056c7fd` only the four nodes named in `red-lane-fixes` still
  failed locally.
- Full CI: seven consecutive scheduled runs red, ~1h42m each (newest sampled 34808346506
  at `2863ed2f19`): failed jobs `lint`, `test (3.12)`, `test (3.14)`, `visual-test`,
  `perf-floors`, `contention-test`, `coverage-contexts` ("No files were found with the
  provided path: .coverage"). The `lint` and `perf-floors` failures shared one root
  cause — a `sase.pager` ↔ `sase.ace.tui.actions` circular import — which is already
  fixed at `f86056c7fd` (both import orders verified); treat it as fixed unless it
  reproduces.
- Release: release-please PR #299 ("chore(master): release 0.17.2") is open and
  MERGEABLE; sase-core-rs 0.34.26 (the pyproject floor) is already published on PyPI;
  `sase axe chop run ci_watch -n` currently reports `gating workflow running` for
  sase-org/sase. The host chop ci_watch owns the release merge.

## Coordination rules (every phase)

Roughly 50 epics are in progress on this project and several own domains this epic
touches. Before fixing any failing test, check whether an active epic owns it: weighted
queue capacity and fleet load (`sase-zp`, `sase-zm`, `sase-10h`, `sase-10j`), remote
fleet rows (`sase-xe.16.11.7.*`), disk retention / managed-tmp (`sase-zw`), and the
long-running red-lane collector epic `sase-th` (all phases closed; its notes hold prior
root-cause records — `sase bead show sase-th`). Consume an owner's landed fix rather
than re-fixing; when a narrow repair is needed for green, make it minimal and record it
on the owning bead with `sase bead note`. Do not create task beads — record
`PROPOSED FOLLOW-UP:` notes on your own phase bead instead.

Distinguish deterministic failures from flakes by rerunning on an unchanged tree before
fixing. Verify with `just check` after changes; the land agent runs `just check-full`
through a monitor before landing the combined tree.

## red-lane-fixes

Reproduce the current red set at your HEAD first; fix only what still fails. Confirmed
deterministic at `f86056c7fd` (all reproduced serially in a fresh workspace):

1. `tests/test_justfile_lint.py::test_rust_dev_install_disables_cargo_incremental_cache`
   — stale exact-string assertions. Commit `eea8af0421` (Cargo build-dir isolation)
   merged the incremental-cache removals into combined commands, e.g.
   `rm -rf "$py_target_dir/build/$profile/incremental" "$py_target_dir/$profile/incremental"`,
   so `output.count('rm -rf "$py_target_dir/$profile/incremental"')` finds 0. The recipe
   behavior is intact (`CARGO_INCREMENTAL=0` appears twice; both incremental directories
   are still removed). Update the assertions to the current recipe text while still
   pinning both the env var and the removal of both per-target `incremental` paths, for
   both `$py_target_dir` and `$lsp_target_dir`.
2. `tests/test_check_sase_core_rs_bindings_tool.py` (all three tests) — root cause in
   `src/sase/core/managed_tmp_reaper.py` (commits `70b018b91a`, `6a60ee5fb7`): it calls
   `require_rust_binding(_REAP_BINDING)` and
   `require_rust_binding(_REAP_WIRE_SCHEMA_VERSION_BINDING)` with module-level
   constants, which the static bindings scan rejects ("use a string literal or a
   one-level forwarder called with string literals"). Follow the scanner's contract
   (literal arguments or a conforming forwarder); do not weaken the scanner. Then re-run
   the whole file: the `artifact_context_query` and dev-extension-exposure tests may
   share this root cause or need the linked sase-core checkout at the pinned revision in
   `sase-core-revision.txt` (`just install` builds it).
3. The Full CI-only failure classes from run 34808346506 — usage-config and
   usage-transport, gate decision/conformance/notification-gate, bead contention
   regressions (lock-timeout waits), commit-workflow bead lifecycle e2e, pager refresh
   provider, monitor delivery — all passed as whole files at `f86056c7fd` during
   planning (88 passed, 3 skipped, serial, Python 3.14), and several failing node names
   no longer even exist. Treat them as fixed by the intervening commits; re-verify
   cheaply, and if one fails in CI but not locally, suspect load-dependent flakiness
   (the contention tests wait on lock deadlines) before code: record a
   `PROPOSED FOLLOW-UP:` note with the evidence (task type `flake` exists for the land
   agent to file) rather than loosening timeouts.
4. `full / contention-test` also died once from a runner shutdown signal
   (infrastructure); do not chase that as a test bug unless it repeats.

Exit: the nodes above pass serially and `just check` is green on your diff.

## hermetic-git-identity

On 2026-09-13, the dominant Master Gate failure class was `Author identity unknown` from
git commits inside `tests/sdd/test_artifact_link_*` (production code committing in
temporary repositories inherits the runner's absent git identity; developer machines
mask this via `~/.gitconfig`). No repo-side fix ever landed — the class is latent, not
gone. Make the suite hermetic:

1. A session-scoped autouse fixture (root `tests/` conftest) that exports
   `GIT_CONFIG_GLOBAL` and `GIT_CONFIG_SYSTEM` pointing at a fixture-owned config file
   declaring a deterministic test identity (e.g. "SASE Test
   <sase-test@example.invalid>"), plus `GIT_AUTHOR_NAME/EMAIL` and
   `GIT_COMMITTER_NAME/EMAIL` defaults. Tests that deliberately exercise
   missing-identity or custom-config behavior must be able to override or opt out; audit
   existing per-repo `git config user.*` calls stay valid on top.
2. A guard test that runs one representative repo-creating test in a subprocess with
   `HOME` redirected to an empty directory and no inherited git env, proving the suite
   no longer depends on host git config. This is the regression tripwire: it must fail
   if the fixture is removed.
3. Prove it: the full `tests/sdd/` directory passes with `HOME` redirected to an empty
   temp dir. (During planning, `tests/sdd/` was verified green under a redirected HOME
   at `f86056c7fd` with per-test identity still present, so the fixture's job is to keep
   it that way for every future test, not to fix a currently-red node.)

Exit: redirected-HOME run green; guard test in place; `just check` green.

## visual-goldens

Full CI run 34808346506's `visual-test` job shows ~25 PNG mismatches, several with large
changed-pixel ratios (`agents_list_120x40` 7.47%, `agents_output_variables_*` 6.25%,
`agents_capacity_budget_accent` 4.27%) — consistent with landed agents-list /
queue-badge / monitor-state UI work, not renderer drift (CI already tolerates small
ratio-only drift; these exceed it). After `red-lane-fixes` lands:

1. `just test-visual`; for each failure inspect `.pytest_cache/sase-visual/`
   actual/expected/diff artifacts and attribute the change to the UI commit that caused
   it (`git log` on the owning widget/panel sources). `sase bead show sase-th` phase 4
   is the precedent for this audit.
2. Accept intentional changes with `--sase-update-visual-snapshots`; anything you cannot
   attribute to a deliberate UI change is a bug to fix, not a golden to refresh. Confirm
   changed pixels stay confined to the surfaces the causing commits touched.
3. Re-run the suite to prove exact-equality green locally.

Exit: `just test-visual` green; every refreshed golden attributed in the stitch
description.

## budget-recalibration

The scoped lane escalates when a selection's estimated serial cost exceeds
`DEFAULT_MAX_SERIAL_SECONDS`, which aliases `FULL_LANE_WALL_SECONDS = 232.0`
(`tests/_test_selection_health.py`) — measured at 28 workers on 2026-08-06. The research
report flags it as "both the escalation trigger and a poor counterfactual" at the widths
agents actually get today. This phase is evidence-gated, not a foregone change:

1. From the durable health store (`tools/selection_health --json` and its records),
   derive the governed full lane's current wall clock at the worker widths recent
   full-lane runs actually used, on this host class.
2. If the measured crossover materially differs from 232 s, update the constant (or
   split the escalation budget from the health-report crossover metric if their meanings
   have diverged) with provenance recorded in the comment, and update any tests pinning
   the value. If it does not differ materially, record the measurement and change
   nothing.
3. Do not loosen escalation blindly: `no-baseline-depth-boost`-inflated history
   over-states serial cost, so prefer measurements from runs with a fresh baseline where
   available, and leave recall guards (change-set rules, ratio rule) untouched.

Exit: a recorded measurement, and either a provenance-carrying constant update with
green `just check`, or a written justification for no change.

## green-ci-and-baseline

Runs after the four prior phases' commits are on master. All long waits go through your
`/sase_monitor` skill — never inline.

1. Watch Master Gate on the newest master SHA until green. If a new deterministic
   failure appeared from commits that landed meanwhile, make the narrow repair
   (coordination rules apply) and watch the next gate.
2. Dispatch Full CI (`gh workflow run full.yml`) on master and watch it (~100 min) until
   every job is green, including `coverage-contexts` actually uploading a `.coverage`
   artifact (the job previously exited 1 with "No files were found").
3. Feed the selector: `just refresh-contexts-baseline`, then run a scoped `just check`
   on a small diff and verify from its manifest/explanations that the context baseline
   was consulted fresh — `context-baseline-stale` and `no-baseline-depth-boost` no
   longer fire.
4. Record before/after evidence from `just selection-health` (escalation rate, context
   staleness, median scoped duration) in your phase-bead notes so the trend is on the
   record. If escalations stay dominated by budget or staleness rules afterward, record
   a `PROPOSED FOLLOW-UP:` with the residual rule histogram.

Exit: Master Gate green on tip; a fully green Full CI run; fresh baseline installed and
consulted on this host.

## release-v0-17-2

The host chop ci_watch owns the release: it merges release-please PRs itself when its
per-repo conditions clear (merge-method merge for sase-org/sase). Epic sase-um's notes
are the operating precedent (`sase bead show sase-um` and its phase pages): **never
hand-merge the release PR.**

1. Preconditions from `green-ci-and-baseline` hold. Inspect
   `sase axe chop run ci_watch -n` (add `-V` for detail): PR #299 should progress from
   `heavy_lane_not_green` / `gating workflow running` to eligible. If ci_watch reports
   `check_error`, diagnose per sase-um.9.4's precedent (host-side chop environment); if
   the repair is outside this repo, record it and raise it to the owner rather than
   working around the chop.
2. Wait (monitor a sleep, or re-inspect on its ~15-minute cadence) for live ci_watch to
   merge #299. Then confirm the Publish workflow's release generation: the 3-hourly cron
   (`17 */3 * * *`) runs release-please; `release_created=true` yields tag `v0.17.2`, a
   GitHub release, build, two install smokes, and the PyPI upload. Per sase-um.9.5.4
   precedent it is acceptable to dispatch
   `gh workflow run publish.yml -f publish_existing=false` instead of waiting for the
   cron. Watch the run to completion through a monitor.
3. Verify the artifact the user asked for: poll `https://pypi.org/pypi/sase/json`
   through a monitor until `info.version == "0.17.2"`, then smoke it in a throwaway
   venv:
   `uv venv /tmp/v0172 && uv pip install --python /tmp/v0172/bin/python sase==0.17.2`
   and check `sase version` and `sase core health --json` (mirroring the publish
   workflow's install smoke).
4. If the upload half-fails, publish.yml's `skip-existing: true` makes a re-dispatch
   with `publish_existing=true` safe; use it only on a partial-upload failure.

Exit: PR #299 merged by ci_watch, tag `v0.17.2` and GitHub release exist, PyPI serves
sase 0.17.2, and the smoke install passes.

## Deferred work for the land agent

File these as task beads when closing the epic (phase workers must not):

1. The leak-detector cost-reduction program — step 3 of the research report
   (`research:202609/test_feedback_speedup_decision.md` §6): suite-scale four-way timing
   experiment through the monitor flow, then a Rust-core fingerprint walk and/or
   dirty-module tracking behind the sase-core boundary, validated against the existing
   detector as an oracle; per-test blocking semantics non-negotiable. Measured prize:
   ~5x multiplier on the cost lane's test phase. Coordinate with epic `sase-j7`
   (process-global state leaks), which owns the adjacent flake class.
2. Harness reductions and executor benchmarks (the report's options D and E) as small
   evidence-gated batches under the prior report's acceptance criteria.
3. Any `PROPOSED FOLLOW-UP:` notes the phases above recorded.
