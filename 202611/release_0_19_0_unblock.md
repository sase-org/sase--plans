---
tier: epic
title:
  "Unblock and ship sase 0.19.0: three train defects fixed, a green rehearsal, PR #322
  merged, and the release proven on PyPI"
goal:
  "The release steward reports release PR #322 releasable instead of looping on metadata
  refresh. The release-please branch passes blocking lint. release.yml's publish and
  confirm jobs have uv. One rehearse-mode release.yml run on the refreshed release
  branch meets every acceptance criterion. #322 is merged through a human-approved gate,
  the steward's train publishes sase 0.19.0, and the published files are verified
  against the sealed manifest, attested, installable, and registered as SASE artifacts."
parent_bead: sase-1s7
phases:
  - id: train-fixes
    title:
      The steward, the release branch's generated docs, and the publish jobs no longer
      block the release
    depends_on: []
    size: medium
    description:
      "train-fixes: stop pin_changed comparing exact core pins, regenerate
      release_announce.html in publish.yml's sync-release-metadata job, and give
      release.yml's publish and confirm jobs a setup-uv step, with tests and docs."
  - id: rehearse
    title:
      An accepted release.yml rehearsal on the release branch refreshed onto the fixes
    depends_on:
      - train-fixes
    size: small
    description:
      "rehearse: confirm the fixes are on master with a green Master Gate, refresh #322
      with a bare publish.yml run, prove the steward no longer asks for a metadata
      refresh, dispatch at most two rehearse-mode release.yml runs, and accept one job
      by job."
  - id: release
    title: sase 0.19.0 is merged, published, verified on PyPI, and recorded as artifacts
    depends_on:
      - rehearse
    size: small
    description:
      "release: merge #322 through one human-approved gate running the steward's merge
      argv, watch the steward's train to PyPI, verify the published files, and register
      the evidence as SASE artifacts."
proposed_by: bbugyi200.athena.sase-1s7.land
proposed_with: claude/opus
create_time: 2026-11-11 03:22:27
status: wip
---

- **PROMPT:**
  [prompts/202611/release_0_19_0_unblock.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/release_0_19_0_unblock.md)
- **PARENT:**
  [202611/ci_green_v0_19_0_pypi_release.md](https://github.com/sase-org/sase--plans/blob/main/202611/ci_green_v0_19_0_pypi_release.md)

# Unblock and ship sase 0.19.0

This epic finishes the release half of parent epic **sase-1s7**. That epic's plan is
`plan:202611/ci_green_v0_19_0_pypi_release.md`; read it with
`sase bead read sase-1s7 -r "<why>"` and the PLAN path it prints.

sase-1s7 fixed master's deterministic reds, added GitHub 5xx retries to the steward, and
got sase-core's CI green. Master Gate is green at `b0f947b91d`. But its rehearsal
(`release.yml` run 42897628895) failed, and its release phase never started. Release PR
**#322** (`chore(master): release 0.19.0`) is open, and PyPI has no `sase` 0.19.0.

The sase-1s7 land agent found three deterministic defects. Each one blocks the release
by itself. All three live in this repo; none needs a sase-core change.

## Evidence

**Defect 1: the steward can never report `releasable`.**

- `tools/_release_watch_facts.py`, method `_release_pr_context`, sets
  `"pin_changed": source_pin != master_pin or exact_pin != master_core_pin`.
- `exact_pin` and `master_core_pin` are the `sase-core-rs==X.Y.Z` requirements in
  `pyproject.toml` at the PR head and at master.
- By design, only the `sync-release-metadata` job in `.github/workflows/publish.yml`
  moves that pin, and only on the release branch (`docs/contribute/rust-backend.md`,
  "Who owns the published version window"). So the PR pins `0.44.2` while master pins
  `0.43.2`.
- `pin_changed` is therefore always true. The Rust policy
  (`sase_core::release_steward::policy`) returns `waiting` / "release PR head or pin
  needs metadata refresh" and re-proposes `dispatch_publish_metadata` forever.
  `publish.yml` runs 42980061856 and 42981444513 both refreshed the PR and changed
  nothing.
- The Rust policy already enforces the right invariant later: the PR's exact pin must
  equal core-ensure's pick ("PR pin equals core-ensure pick").
- A read-only simulated tick with `pin_changed` limited to the source-pin comparison got
  past the Master Gate, PR-check, and core-ensure-pin clauses. It stopped only at the
  cadence window ("before 09:00 UTC"; merges are allowed from 09:00 UTC, one per UTC
  day).
- The bug is in the original reader commit `23cccbc969` and is not covered by any test.

**Defect 2: release-please leaves the release branch's lint red.**

- `tools/render_applicability` renders `docs/overrides/generated/release_announce.html`
  from `.release-please-manifest.json`. `just fmt-docs-check` (blocking `lint`, in both
  `ci.yml` and the train's `qualify-ci`) fails when the file is stale.
- release-please bumps the manifest to `0.19.0` on `release-please--branches--master`
  without regenerating the file. The rehearsal's `lint` failed with
  `release_announce.html is stale for manifest version 0.19.0`, which cascaded into
  `qualify` (`QUALIFY_RED(lint:harness)`).
- Master is consistent today: manifest `0.18.1`, file `v0.18.1`. Merging #322 unfixed
  would turn master's lint red as well.

**Defect 3: the train's `publish` and `confirm` jobs have no `uv`.**

- In `.github/workflows/release.yml`, `build`, `exercise-legs`, and `confirm-install`
  run `astral-sh/setup-uv@v4`.
- `publish` (the promotion predicate,
  `uv run --no-project --with packaging python tools/release_candidate evaluate`, plus
  its later `uv run` steps) and `confirm` (`uv run`, `uv tool install`) never install
  uv.
- The rehearsal's `publish` failed with `uv: command not found`. `confirm` runs only in
  release and resume mode, so a rehearsal can never surface its copy of the bug; the
  real train would have burned 0.19.0 there.

**State at planning (2026-11-11 ~08:00 UTC):**

- #322 head `76465de42e`, base `b0f947b91d`, MERGEABLE/CLEAN. `prepare`,
  `Conventional PR title`, and `release-core-floor-smoke` are green.
- PyPI: `sase` latest is 0.18.1. `sase-core-rs` 0.44.2 is published, and tag `v0.44.2`
  contains the source pin `5c0e49cf`.
- sase-core master CI 42673707335 is green.
- `RELEASE_STEWARD_MERGE_ENABLED` is unset, so the steward reports `releasable` with its
  merge suppressed.

**Out of scope here:** Telemetry `contention-test` timeouts (owned by
sase-1rq.2.1.4.1.5.3.1), Telemetry test-cost hard budgets (owned by sase-1rq.2.1), and
the advisory visual and coverage flakes (sase-1s1, sase-1s8, sase-1s9). Do not repair
them here.

## Carried-over reviewer decisions (final; do not re-ask)

The reviewer settled these for sase-1s7, and they bind this epic:

- **merge_prompt = gate.** Once #322 is releasable, one `/sase_gate` approval runs the
  steward's exact merge argv. A hand merge, or the owner enabling steward merges, also
  works.
- **rehearsal = release_branch.** One accepted rehearse-mode `release.yml` run on the
  refreshed release branch must come before the version is tagged.

## Rules for every phase

- **Read memory first** through `/sase_memory_read`:
  - always `lint_and_test.md` and `sase_beads.md`;
  - `tools.md` before changing `tools/`;
  - `docs_writing.md` before editing `docs/`;
  - `sase_artifacts.md` before creating artifacts.
- Read `docs/contribute/release-train.md` before touching the release path.
- **Re-inventory first.** Run `git fetch` and re-check on the current `origin/master`.
  Other epics land continuously.
- **Overlapping beads.** Never close, reopen, edit, reassign, relaunch, stop, or resume
  sase-1q2.9.x, sase-1r2.6, sase-1rq.\*, sase-1n6.11.4, or sase-1oe.4, or their agents
  and monitors. Never touch the sase-1r2 watchdog crontab entry. Notes are allowed where
  a phase says so.
- **Fix products, never tolerance.** Do not raise timeouts, budgets, or caps. Do not add
  retries around assertions. Do not skip, xfail, or delete assertions.
- **Verify.** Run `just fix`, then `sase tool run check`. Never run `just check-full`.
  The CI `lint` job enforces `toobig`, so never grow a file past its limit.
- **Hand waits to monitors.** Every CI, PyPI, or steward wait goes to `/sase_monitor`
  with an explicit deadline and a `--next` follow-up. Pure waits use `-w 0`. Use
  `gh run watch <id> -R sase-org/sase --interval 120 || true`, then judge with
  `gh run view <id> --json jobs`. Never end a turn waiting.
- **Outward actions allowed, and nothing else:**
  - `gh workflow run publish.yml -R sase-org/sase`, bare (the steward's own metadata
    refresh);
  - `gh workflow run release.yml -R sase-org/sase --ref release-please--branches--master -f mode=rehearse`,
    only in the rehearse phase;
  - `gh workflow run release-steward.yml -R sase-org/sase`, observe-only, with no `live`
    input;
  - `gh run rerun` only where a phase says so;
  - the release phase's single human-approved merge gate.
- **Forbidden:**
  - `release.yml` with `mode=release` or `mode=resume` (the steward does that);
  - `gh variable set`, or any environment or branch-protection edit;
  - sase-core's `release-plz.yml`;
  - merging any PR outside the approved gate.
- **No new beads.** Record follow-ups as `PROPOSED FOLLOW-UP: <summary — detail>` notes
  on your own phase bead.
- **Always close.**
  1. `sase bead epic-symbols <phase-id>` must list nothing.
  2. Close `-R done`, or `-R canceled` with the evidence when bounded work runs out, so
     dependent phases and the land agent always run.

## Phase train-fixes

**Repo:** sase. Read `tools.md` and `docs_writing.md`.

### 1. Steward `pin_changed`

In `tools/_release_watch_facts.py` (`_release_pr_context`), make `pin_changed` mean only
that the PR's source pin differs from master's: `source_pin != master_pin`.

- Drop the `exact_pin != master_core_pin` clause, and drop the `master_core_pin` read if
  nothing else uses it.
- Keep `exact_public_core_pin` in the facts. The Rust policy still checks it against
  core-ensure's pick.
- Do not change the Rust wire or policy.

Add facts-contract tests beside the existing cases
(`tests/_release_watch_facts_contract_cases.py` and the `test_release_watch_*` module
that collects them; reuse its fake GitHub transport):

- The PR pins `sase-core-rs==0.44.2`, master pins `==0.43.2`, both share one
  `sase-core-revision.txt` pin, and compare status is `behind`. Expect
  `pin_changed is False` and `stale_head is False`.
- The source pins differ. Expect `pin_changed is True`.
- Compare status is `ahead` or `diverged`. Expect `stale_head is True`, as today.

Then add one end-to-end decision test with the Rust binding through the existing steward
fixtures (`tests/_release_steward_fixtures.py`). A ready snapshot whose PR exact pin
equals core-ensure's pick but differs from master's exact pin reaches `releasable` with
the merge suppressed once the cadence allows it. It must not stop at "release PR head or
pin needs metadata refresh".

### 2. Release-branch generated docs

In `.github/workflows/publish.yml`, job `sync-release-metadata`, step "Reconcile release
metadata": after `uv lock`, run `python tools/render_applicability` (render mode, no
flags; stdlib only).

- Include `docs/overrides/generated/release_announce.html` in the `git diff --quiet`
  check and in the `git add` list next to `pyproject.toml` and `uv.lock`.
- Keep the existing commit message.

Before editing, check `tools/render_docs --format-only` for any other renderer whose
output depends on `.release-please-manifest.json` or the `pyproject.toml` version. To
confirm, render on a temporary checkout of `origin/release-please--branches--master`
outside the repository with `tools/render_docs --format-only --check`. Regenerate every
such output in the same step. Add the renderer only if it is stdlib-only; otherwise
install its deps the way the job installs Python.

Extend `tests/test_github_actions_publish.py`. The sync step must:

- run the announce render after `uv lock`;
- `git diff` and `git add` the partial alongside `pyproject.toml` and `uv.lock`.

Also add a unit test in `tests/test_render_applicability.py`. A manifest bumped to a new
version with the partial still at the old version must fail `check()`, and pass after
`render()`. This pins the failure mode the workflow step fixes.

### 3. uv in the train jobs

In `.github/workflows/release.yml`, add `- uses: astral-sh/setup-uv@v4` to `publish` and
to `confirm`. Mirror the step and options `confirm-install` and `build` already use
(cache options included). Place it after `actions/checkout@v4` and before the first step
that runs `uv`.

Then statically audit every job in `release.yml`, especially the release-only `stash`,
`confirm`, and `confirm-install`, which a rehearsal never runs. Check for any other
command that needs a tool the job never installs. Fix each gap in the same way.

Add a workflow test, following the existing `tests/_github_actions_*` helpers, that
fails if any `release.yml` job has a `run` step invoking `uv ` without an
`astral-sh/setup-uv` step earlier in that job.

### 4. Docs

In `docs/contribute/release-train.md`, add one sentence each:

- the steward asks for a metadata refresh only for a stale PR head or a changed source
  pin, while the PR's exact core pin is checked against core-ensure's pick;
- the release-metadata sync also regenerates the docs release line.

Keep both under the docs word caps.

### 5. Verify and land

Run `just fix` and `sase tool run check`.

**Close note.** Name the three fixes with file paths, the new tests, and the audit
result for the release-only jobs.

**Done when:**

- the new tests pass and the existing release_watch, publish, and render tests are
  green;
- `sase tool run check` passes.

## Phase rehearse

**Repo:** sase. This phase normally changes no files. Read `/sase_monitor` first.

### 1. Prerequisites

- train-fixes closed `done`, and its commit is on `origin/master`. Check with
  `git log origin/master --grep 'SASE_BEAD=\[<train-fixes phase id>\]'`, re-fetching for
  up to about 5 minutes.
- If train-fixes closed canceled, close
  `-R canceled -r "rehearsal not started: train-fixes canceled"`.
- The Master Gate push run at the newest `origin/master` SHA that contains that commit
  is green in every job. Wait for it with a zero-weight monitor (deadline about 2h).
- If Master Gate is red on a node unrelated to train-fixes, check
  `sase bead search '<node>'` and `sase agent list -j`. Rerun failed jobs once with
  `gh run rerun <id> --failed` for a flake, recording a `PROPOSED FOLLOW-UP:`. Otherwise
  close `-R canceled` with the evidence.

### 2. Refresh the release branch

- If #322 is already merged, close `-R done`, noting that the real train supersedes the
  rehearsal.
- Otherwise dispatch `gh workflow run publish.yml -R sase-org/sase`, bare. Wait with a
  zero-weight monitor (deadline about 1h) until that `publish.yml` run is **completed**.
  Waiting only for the branch to move is not enough: sase-1s7.4 attempt 1 raced the sync
  job.
- Then require all of:
  - the branch contains the train-fixes commit:
    `git merge-base --is-ancestor <sha> origin/release-please--branches--master`;
  - `git show origin/release-please--branches--master:docs/overrides/generated/release_announce.html`
    names `v0.19.0` (or the version the PR carries);
  - the branch's `pyproject.toml` pins the `sase-core-rs` version core-ensure picks
    (expected `0.44.2`).
- If the sync did not regenerate the partial, that is a train-fixes defect. Fix it on
  master, land, and refresh again.

### 3. Steward check

Run `.venv/bin/python tools/release_watch --once --json` (observe-only by default). Its
primary must not be `waiting` with reason "release PR head or pin needs metadata
refresh". Expect:

- `releasable` with the merge action suppressed; or
- `waiting` only on cadence ("before 09:00 UTC", or "daily merge already used"); or
- `waiting` on PR checks still running at the new head.

Any red primary state is recorded and fixed under the global rules, or the phase closes
canceled.

### 4. Rehearse

1. Run `gh run list -R sase-org/sase --workflow release.yml --limit 5`. If a run is
   queued or in progress, wait for it.
2. Dispatch
   `gh workflow run release.yml -R sase-org/sase --ref release-please--branches--master -f mode=rehearse`.
   Use no `core_version` override.
3. Record the run ID and branch head SHA.
4. Immediately run `tools/release_watch --once --json`. It must not report
   `ambiguous_trains`; if it does, `gh run cancel` the rehearsal and record why.
5. Hand `gh run watch` to a zero-weight monitor (deadline about 4h, `--next`). Never
   cancel because of a long queue.

### 5. Accept job by job

- `identity`, `build` (with containment), and `qualify-prepare` are green.
- `qualify` is green. That needs blocking `lint` and `test (3.12|3.13|3.14)`.
- An advisory red (`visual-test`, `perf-floors`, `tutorial-tests`,
  `ace-page-group-isolation`) becomes a `PROPOSED FOLLOW-UP:`, or a pointer note on
  sase-1s9 or sase-1s1 when it is their node. It never fails acceptance.
- All four `exercise-legs` and the merged `exercise` verdict are green, with 0.18.1 as
  the N-1 previous release.
- `publish` (verify-only in rehearse) is green, with an `upload` plan for every sealed
  file. Any refusal is red.
- `stash`, `confirm`, and `confirm-install` are skipped.

### 6. Bounded repair

- If `qualify` is red at attempt 1 only on blocking `test` failures, run
  `gh run rerun <id> --failed` once. Record the flaky nodes as `PROPOSED FOLLOW-UP:`.
- Any other red is a defect to fix on master before the tag exists. Fix it under the
  global rules, land, refresh the branch (step 2), and rehearse once more.
- Allow at most 2 rehearsal dispatches in this phase. If it is still red, close
  `-R canceled` with the run URL, failing job, and step.

### 7. Notes

Add the accepted run URL and the per-criterion verdict to sase-1q2.9.3 and sase-1oe.4.
Change no status.

**Done when** one rehearsal meets every acceptance criterion. Close with its URL, head
SHA, and the verdicts.

## Phase release

**Repo:** sase. Read `sase_artifacts.md`, `/sase_gate`, and `/sase_monitor` first.

### 1. Idempotent start

Take the target version from the open or just-merged release PR
`chore(master): release X.Y.Z` (expected 0.19.0, #322). If
`curl -fsS https://pypi.org/pypi/sase/<version>/json` already succeeds, go to step 6.

### 2. Prerequisites

- rehearse closed `done`. Otherwise close
  `-R canceled -r "release not attempted: rehearse not accepted"` with its evidence.
- Run `.venv/bin/python tools/release_watch --once --json`. Expect `releasable`.
- For `waiting` on cadence, PR checks, or a fresh metadata refresh, poll `--once` every
  5 minutes with a zero-weight monitor (deadline about 26h, which covers the 09:00 UTC
  window).
- If the steward asks for a metadata refresh again at an unchanged head, that is a
  train-fixes regression. Record it and close `-R canceled`.
- For a red state, record it and close `-R canceled`. Repairs belong to the earlier
  phases.

### 3. Merge (merge_prompt = gate)

1. Read the steward's merge action from `--once --json`: the PR number and head SHA.
2. Create one gate through `/sase_gate`, titled
   `Merge sase release PR #<n> (<version>)`.
3. Its approve action runs exactly
   `gh pr merge <n> -R sase-org/sase --merge --match-head-commit <head_sha>`.
4. Its body links:
   - the steward state;
   - the newest green Master Gate run;
   - the accepted rehearsal run;
   - sase-1s7.5's workflow table (Full CI blocking jobs green; Telemetry reds are
     off-release-path and owned elsewhere).
5. A second branch lets the owner say they merged by hand or set
   `RELEASE_STEWARD_MERGE_ENABLED=true` themselves.

Creating the gate ends the turn; continue in its follow-up. If `--match-head-commit`
refuses because release-please moved the head, re-read the action and raise the gate
once more. If the PR is already merged, skip the gate. Never merge outside the approved
gate, and never set the variable.

### 4. Watch the train

Hand
`.venv/bin/python tools/release_watch --until-released <version> --poll 5m --deadline 24h`
to a zero-weight monitor (timeout 25h, `--next`).

- The live steward cuts the tag and draft (via `publish.yml`), dispatches
  `release.yml mode=release` once, and handles its own reruns and resumes. This phase
  never dispatches `release.yml`.
- If the train ends `burned`, `abandoned`, `publish_conflict`, or `vetoed`, or stays red
  past the deadline: record the steward report, run URLs, and failing jobs as
  `PROPOSED FOLLOW-UP:`, then close `-R canceled`. Never start a second release cycle.

### 5. Confirm completion

The `release.yml` run at `v<version>` concluded success with `confirm` green, and
`tools/release_watch --once --json` reports `released`.

### 6. Verify PyPI

Every check must hold:

- `https://pypi.org/pypi/sase/<version>/json` lists exactly the sealed wheel and sdist,
  with upload times, sizes, and SHA-256 digests.
- Those digests equal the sealed candidate manifest on the GitHub release
  (`tools/release_candidate show`/`download`). The sets match exactly.
- Each file has PyPI provenance
  (`https://pypi.org/integrity/sase/<version>/<filename>/provenance`) naming
  `sase-org/sase` and `release.yml`.
- GitHub release `v<version>` is published (not a draft) and marked Latest.
- **Clean install.** In a fresh temp dir outside every repository, use isolated `HOME`,
  `SASE_HOME`, `XDG_*`, `UV_TOOL_DIR`, and `UV_TOOL_BIN_DIR`, set `UV_NO_CACHE=1`, and
  unset the caller's `SASE_*` variables. Then:
  1. `uv tool install --python 3.12 sase==<version>`;
  2. `sase version -j` reports `<version>` and the manifest's exact `sase-core-rs`
     (expected 0.44.2);
  3. `sase core health --json` passes.

### 7. Evidence artifacts

Write these files in a fresh temp dir outside every repository:

- `sase-<version>-pypi-release-evidence.md`, containing:
  - the version, PyPI URL, and release URL;
  - the merge commit, tag, and peeled SHA;
  - the `release.yml` run URL with each job's conclusion;
  - a table of each file's PyPI SHA-256 against the manifest's;
  - the provenance summary;
  - a clean-install transcript excerpt;
  - the steward's `released` verdict;
  - sase-1s7.5's workflow table;
  - the rehearsal URL.
- `pypi-sase-<version>.json`: the raw PyPI JSON.
- `install-check-sase-<version>.txt`: the raw clean-install transcript.

For each file:

1. `sase artifact create -p <file> -l "<label>" --bead <your phase id>`;
2. `sase artifact show <ref>`;
3. `sase artifact link add <ref> related bead:sase-1n6.11.4 "first release through the steward train"`;
4. `sase artifact link add <ref> related bead:sase-1r2 "sase 0.19.0 confirmed on PyPI"`;
5. `sase artifact link add <ref> related bead:sase-1s7 "release proof for epic sase-1s7"`.

Then note the artifact refs on sase-1n6.11.4, sase-1q2.9.4, and sase-1r2.6. Change no
status.

**Done when** every step 6 check holds and every artifact is registered. Close with the
artifact refs, the PyPI URL, and the `release.yml` run URL.

## Landing

This epic's land agent verifies the three fixes and the release evidence, and turns
`PROPOSED FOLLOW-UP:` notes into task beads through `/sase_new_task`. It does not close
sase-1s7: the `parent_bead` link hands that landing back to sase-1s7's land agent.

The final summary must list:

- the merged PR, tag, and `release.yml` run;
- the PyPI URL;
- every evidence artifact ref;
- the out-of-scope bot PRs (#300, #319, #320, #323, #326) and the external PR #301, with
  their CI state, none of them merged or closed.
