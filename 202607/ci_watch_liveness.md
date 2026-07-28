---
tier: epic
title: Make the ci_watch chop actually observe high-cadence repositories
goal: Rework ci_watch so a continuously-busy repository like sase-org/sase can be
  classified red or green instead of being permanently masked as run_in_flight, stop
  counting CI-less repositories as errors, retain per-tick decision evidence, and
  redeploy with a soak that proves the fix half fires.
phases:
- id: red-liveness
  title: Replace the blanket in-flight veto with job-level terminal-failure evidence
  depends_on: []
  size: medium
  description: 'red-liveness: classify on terminal job evidence with a supersession
    check, a sha-independent debounce persisted in the chop state dir, and a benign
    no_ci state, so busy repositories become observable without weakening fail-closed
    behavior.'
- id: ledger-and-status
  title: Retain per-tick decision evidence and report no_op honestly
  depends_on:
  - red-liveness
  size: small
  description: 'ledger-and-status: give the decision ledger a per-run filename so
    ticks stop overwriting each other, and stop counting skipped release plans as
    actionable work.'
- id: rollout
  title: Redeploy the chop, retune the athena lane, and soak the fix half
  depends_on:
  - ledger-and-status
  size: small
  description: 'rollout: bump and publish bugyi-chops, refresh the athena install,
    adjust the chezmoi lane vars, and verify against live ticks that sase is observable
    and errors drop to zero.'
create_time: 2026-07-27 14:00:30
status: wip
bead_id: sase-a4
---

- **PROMPT:** [202607/prompts/ci_watch_liveness.md](prompts/ci_watch_liveness.md)

# Plan: make ci_watch observable on high-cadence repositories

## Context

The `ci_watch` chop shipped under epic `sase-a2` (plan `plans:202607/ci_watch_chop.md`) and is deployed and ticking
cleanly on the athena `ci_watch` lumberjack lane at a 300s interval. A review of live axe logs confirmed that the
deployment, the `source`/`dry_run` context threading from phase `sase-a2.1`, the dual actstat/allowlist scoping, and the
merge half's guard chain all work as designed. It also found that **the fix half is structurally unable to ever fire for
`sase-org/sase`**, that one repository is counted as a permanent error, and that per-tick evidence is destroyed.

This plan fixes those defects. It does not revisit anything the review found working.

### Verified evidence

All figures below were measured against the live system on 2026-07-27 and should be treated as the baseline the rollout
phase re-measures.

**Defect 1 — the in-flight veto permanently masks red (critical).** Every logged tick classified both `sase-org/sase`
and `sase-org/sase-core` as `pending` with reason `run_in_flight`. Feeding a live actstat sweep through the chop's own
pure functions shows the red predicate is correct and the veto is what discards it:

```
sase-org/sase        classify=pending  active=True  settled=f15c05d  actionably_red(settled)=True
sase-org/sase-core   classify=pending  active=True  settled=e97d150  actionably_red(settled)=True
sase-org/sase-github classify=green    active=False settled=dd5deed  actionably_red(settled)=False
sase-org/sase-telegram classify=green  active=False settled=bfb2a79  actionably_red(settled)=False
```

`sase-org/sase` at `f15c05d` had two jobs at terminal `failure` (`published-core-minimum-smoke` and `bead-backend`) on a
`cancelled` run — precisely the "cancelled with a failing job is red" case the chop was written for — and it was
silently skipped.

The masking is structural, not transient. Measured over `sase-org/sase` master:

- at least one run in flight **100.0% of a 4-hour window, with zero settled gaps**;
- push batches every **~4.2 min** (median) against a CI workflow that takes **~7.8 min** (median, up to 51 min);
- **96 of the last 100** master CI runs ended `cancelled`, because cancel-in-progress concurrency kills each run when
  the next push lands.

So `sase-org/sase` can never reach `RED` (no fix agent will ever be proposed) and can never reach `GREEN` (its release
PR would never merge even after `merge_enabled` is flipped). `sase-org/sase-core` is only 18% busy and is not
structurally blocked, but is masked by the same rule whenever it is active.

Root cause is two places in `src/bugyi_chops/ci_watch.py`: `classify_repo` returns `PENDING` on
`observation.active is not None`, and the revalidation loop in `build_ci_watch_result` demotes any `RED`/`GREEN` repo to
`PENDING` with reason `newer_head_or_run_in_flight` when `github.has_in_flight_runs(...)` is true. The implementation
faithfully matches the `sase-a2` policy table; the **policy** is what is wrong for a repository with this cadence.

**Defect 2 — a CI-less repository is a permanent phantom error (moderate).** `sase-org/sase-nvim` has zero GitHub
Actions workflows (`gh api repos/sase-org/sase-nvim/actions/workflows` reports `total_count` 0; there is no `.github`
directory), so actstat emits no row for it at all. `ActstatClient.sweep` synthesizes `error="missing_observation"` for
any allowlisted repo with no row, and `classify_repo` maps that to `RepoState.ERROR`. Every tick therefore reports
`errors=1` forever, so a genuine new `repo_error` is invisible in the summary line — it just keeps reading `errors=1`.
The `sase-a2` policy table only ever assigns `errors` to real `repo_error` rows.

**Defect 3 — the decision ledger is destroyed every tick (moderate).** `_write_ledger` writes to a fixed filename,
`ci_watch_decisions.json`, in the run directory's **parent** (`.../chops/ci_watch/runs/`), which is shared by every run.
Seven completed runs produced exactly one ledger file, holding only the newest tick's decisions. Post-hoc auditing of
what the chop decided and why is impossible beyond the latest tick, which materially hampered this review.

**Defect 4 — every tick reports `status: ok` (minor).** `actionable` is true when `fix_proposed`, `merged`, **or**
`release_plans` is non-empty. During the intended `merge_enabled: false` soak, two release PRs are permanently
eligible-but-not-merged, so `release_plans` is always non-empty and the chop never reports `no_op`. It reads as though
it acted on every tick when it did nothing.

### Non-negotiable invariants

These come from `sase-a2` and remain binding. No phase may relax them.

- **The merge predicate is the only gate before a PyPI/crates publish.** No `sase-org` repository has branch protection.
  Every merge guard in `plan_release_merge` stays; redundancy is the point. Agents never enter the merge path.
- The zero-agents gate stays **in-script** (not `inhibit_if`, not `wait_on`), keeps failing closed as
  `agents_check_failed`, and keeps gating only the fix half.
- `max_fix_proposals_per_tick` (1) and the stable `agent_name: ci_fix.<slug>` launch-time collision idempotence stay
  exactly as they are.
- Unknown or unparseable input still fails closed; process-level actstat failures still fail the whole tick.
- No tokens in vars, evidence, prompts, or logs; argv lists only, never `shell=True`.
- Nothing runtime-specific in proposals or prompts.

## Phase red-liveness

Work in the `bugyi-chops` repository: open it with `sase repo open gh:bbugyi200/bugyi-chops` and use the printed path.
All code lives in `src/bugyi_chops/ci_watch.py` with tests in `tests/test_ci_watch.py`. Follow house style: reuse
`_common.py` (`run_chop`, `result_with_summary`, `safe_fragment`, `context_vars`), strict mypy, ruff, and the 90%
coverage gate via the `just` targets (`just install`, then `just check`).

The goal is to separate **"the failure evidence is stale or superseded"** from **"something is running"**. Only the
former should suppress a fix. Do not simply delete the veto.

### 1. Benign `no_ci` classification

Distinguish "actstat reported nothing because the repository has no Actions at all" from "actstat should have reported
something".

- When an allowlisted repo has no actstat row, probe it once per tick with a new bounded `GitHubReader` method backed by
  `gh api repos/<repo>/actions/workflows`. A `total_count` of 0 yields a new benign outcome (reason `no_ci`) counted in
  a new `no_ci` counter, **not** `errors`, and excluded from fix and merge consideration.
- Any other result, or a failed/unparseable probe, keeps today's fail-closed behavior: `RepoState.ERROR` with reason
  `missing_observation`.
- Keep real `repo_error` rows counted in `errors` exactly as today.

### 2. Terminal-evidence classification

Replace the `observation.active is not None` veto and the `newer_head_or_run_in_flight` demotion with the following.
Implement the decision as pure functions so every row is directly testable.

Resolve `head = github.default_branch_head(repo)` and let `settled` be the actstat `commit` row.

- **`actionably_red(settled)` is true and `settled` is the current HEAD** → red pinned at `head.sha`. Sibling workflows
  still running on that same HEAD (e.g. `Publish`, `Deploy Docs`) do **not** suppress it: the failing jobs already
  reached a terminal `failure` conclusion and no in-flight sibling can retract that.
- **`actionably_red(settled)` is true and HEAD is newer** → consult HEAD before deciding, via a new bounded
  `GitHubReader` method over `gh api repos/<repo>/actions/runs` filtered by `head_sha`:
  - HEAD has its own terminal red conclusion → red pinned at `head.sha` (prefer the fresher evidence).
  - HEAD's runs are all completed and green → `GREEN`; the older red was fixed.
  - HEAD still has runs in flight → red pinned at the settled sha, flagged `head_unsettled`. **This is the case that
    rescues `sase-org/sase`**, where HEAD is essentially never settled.
- **`settled` is green and is the current HEAD** → `GREEN`.
- **`settled` is green and HEAD is newer** → `PENDING`, reason `newer_head_unsettled`. The merge half stays conservative
  here deliberately.
- **Anything else** → `PENDING`, reason `superseded_or_unsettled` (unchanged).

Cap the number of extra `gh` calls per tick and per repo, and fail closed to `ERROR` for that repo (not the whole tick)
if a supersession query fails or returns an unexpected shape.

### 3. Supersession check

Before a red candidate becomes actionable, confirm the specific failing job identities have not already succeeded on a
newer commit. Compare the failing job-name set from the red evidence against HEAD's terminal job conclusions; drop any
job that has since concluded `success` on HEAD, and if the set empties, classify `GREEN` with reason
`superseded_by_newer_success`. `cancelled` jobs are never treated as either failure or success.

### 4. Debounce, keyed on job identity rather than sha

Persist a small streak file under `invocation.context.state_dir` (the lumberjack state directory), written atomically
with the same `mkstemp` + `os.replace` + `fsync` discipline `_write_ledger` already uses.

- The fingerprint **must be sha-independent** — use the sorted set of failing job names. This is essential: on
  `sase-org/sase` the HEAD sha changes roughly every four minutes, so any sha-keyed streak would reset forever and the
  debounce would never mature.
- Increment the streak when the same fingerprint is red on consecutive ticks. Reset it when the repo is observed green,
  when the fingerprint changes, or when a proposal is emitted for it.
- Require `streak >= red_debounce_ticks` (new var, default 2) before proposing, so the chop does not chase a failure
  that an already-in-flight push is about to fix. Record `red_debounce` as the ledger reason while a candidate is
  maturing, and count it under `fix_suppressed`.
- Treat a missing, unreadable, or malformed streak file as "no history" and rebuild it, so corrupted state can never
  crash a tick or block the chop permanently.

### 5. Dedupe key

Change the proposal `dedupe_key` from `ci_fix:<repo>:<head_sha>` to a sha-independent key derived from the repo and the
failing job fingerprint. With HEAD moving every few minutes, the sha-keyed form would launch a fresh fixer for every new
HEAD on the same persistent failure.

Keep `agent_name: ci_fix.<slug>` unchanged so a launch-time name collision with a still-live fixer resolves as an
idempotent recorded skip. Keep pinning the concrete run URL and sha inside the prompt as evidence — only the dedupe key
becomes sha-independent. Document in the chop description body that a fixer landing either changes the failing job set
or turns the repo green, and that either outcome resets the streak and releases the key.

### 6. Green-for-merge narrowing

`plan_release_merge` and its guard chain are unchanged. The only change is that the `GREEN` state feeding guard (4) is
now produced by the rules above, so a repo whose unrelated long-running workflow (`Publish`, `Deploy Docs`) is in flight
is no longer blocked from being considered green. `generator_busy` still independently blocks merges while the release
generator itself is running, and the PR is still re-read and head-pinned immediately before merging.

### 7. Counters, ledger, and prompt

Add `no_ci` and a debounce-suppression counter to `_new_counters`. Record the new reasons in the ledger
(`head_unsettled`, `red_debounce`, `superseded_by_newer_success`, `newer_head_unsettled`, `no_ci`), plus the streak
value and the failing job fingerprint for each red candidate. Extend the fix prompt to state when the pinned failure is
on a settled commit older than an unsettled HEAD, so the agent re-verifies against current state.

### 8. Tests

Extend the existing fake-adapter suite in `tests/test_ci_watch.py`. Cover at minimum:

- red on the current HEAD is proposed while an unrelated workflow is still in flight on that same HEAD;
- red on a settled commit is proposed when HEAD is newer and still unsettled (the `sase` case), pinned at the settled
  sha and flagged `head_unsettled`;
- red is suppressed when HEAD has since concluded green for the same job set (`superseded_by_newer_success`);
- red on HEAD itself takes precedence over older settled red evidence;
- debounce: first observation suppresses with `red_debounce`, second consecutive observation of the same job fingerprint
  proposes; a changed fingerprint or an intervening green resets the streak; a missing or corrupt streak file is
  tolerated;
- the dedupe key is stable across two ticks whose HEAD shas differ but whose failing job set is identical;
- `no_ci` classification for a repo with zero workflows versus `missing_observation` error for a repo whose workflow
  probe is non-zero or fails;
- the supersession/HEAD queries failing degrade that repo to `ERROR` without failing the whole tick;
- every existing merge guard still individually disqualifies, and `merge_enabled: false`, missing `SASE_CHOP_DRY_RUN`
  mirror, and dry-run each still render the plan with zero mutations;
- the zero-agents gate, `agents_check_failed`, and the per-tick fix cap behave exactly as before;
- no token-like strings appear in evidence or prompts.

## Phase ledger-and-status

Same repository and tooling as the previous phase.

1. **Retain per-tick evidence.** `_write_ledger` currently writes the fixed name `ci_watch_decisions.json` into the run
   directory's shared parent, so each tick overwrites the last. Derive a per-run filename from the result file's stem
   (for example `<run-stem>.decisions.json`) so every tick keeps its own ledger, and return that name as the evidence
   reference. Keep the atomic `mkstemp` + `os.replace` + `fsync` write. Confirm the evidence reference the chop returns
   still resolves correctly for the axe runner.
2. **Report `no_op` honestly.** `actionable` must no longer count release plans that were computed but deliberately not
   merged. Base it on work actually performed — a fix proposed or a merge executed — so a merges-disabled soak tick with
   nothing to do reports `no_op` with a truthful reason instead of `ok`.
3. Add tests for both: two consecutive ticks leave two distinct ledger files, and a tick with eligible-but-skipped
   release plans and no proposals reports `no_op`.

## Phase rollout

Depends on both prior phases being merged.

1. **Publish.** Bump the `bugyi-chops` version in `pyproject.toml` per repo convention and merge. Confirm the
   `bugyi_chop_ci_watch` entry point in `[project.scripts]` is unchanged.
2. **Refresh the athena install.** Run `uv tool upgrade sase` — the `sase` uv tool venv installs `bugyi-chops` from its
   GitHub repo — then verify the `bugyi_chop_ci_watch` entry point exists in that venv's bin directory. If the upgrade
   reports anything unexpected beyond refreshed packages, stop and report rather than forcing.
3. **Retune the lane.** In the `chezmoi` repository (open it with `sase repo open chezmoi`), edit the `ci_watch` lane in
   `home/dot_config/sase/sase_athena.yml`:
   - add the new `red_debounce_ticks` var (default 2) if it should be tunable from config;
   - drop `sase-nvim` from `merge_order`, since it has no `release_repositories` entry and can never merge — keep it in
     `vars.repos` so it is still swept and now reports the benign `no_ci` state;
   - keep `merge_enabled: false`; this phase does not flip it;
   - update the lane and chop `description` bodies to describe the new terminal-evidence policy and the debounce,
     following the axe description grammar (summary line, blank line, body).

   After committing, run `chezmoi update -a --force` as that repository's instructions require.

4. **Smoke test.** Run `sase axe chop run ci_watch --dry-run -V` and confirm populated sweep counters and zero
   mutations.
5. **Verify against live ticks.** Watch `~/.sase/axe/logs/lumberjack-ci_watch.log` and the per-run ledgers across
   several ticks and confirm all of:
   - `errors=0` (`sase-org/sase-nvim` now reports `no_ci`);
   - `sase-org/sase` is no longer permanently `pending` — it reaches `red` or `green`;
   - each tick writes its own decision ledger file;
   - a tick with nothing to do reports `no_op`;
   - at most one fix proposal per tick, and none while any agent is live.
6. **Soak and report.** Leave `merge_enabled: false` for a few days. Confirm the debounce prevents fixer churn as HEAD
   moves, that the red predicate never fires on superseded `cancelled` runs, and that the zero-agents gate defers fixers
   whenever agents are live. Report the observed proposal rate before recommending that `merge_enabled` be flipped to
   `true` with `max_merges_per_tick: 1`.

## Risks and boundaries

- **Merging safety is unchanged and must stay that way.** This plan only changes how a repository becomes `GREEN`; every
  merge guard, the PR re-read, `--match-head-commit` pinning, `generator_busy`, and the per-tick cap remain. Do not
  "simplify away" any redundant guard.
- **Fixer churn is the main new risk.** Making a permanently-busy repository observable means `sase-org/sase` can now
  propose fixers where it previously proposed none. The sha-independent debounce, the sha-independent dedupe key, the
  one-proposal-per-tick cap, and the zero-agents gate are four independent brakes; the rollout phase must confirm the
  observed rate before merging is enabled.
- **Extra `gh` calls.** The supersession and HEAD queries add API traffic per tick. Bound them, restrict them to repos
  that actually need adjudication, and keep the 120s `chop_timeout` in mind.
- **Complementary repository-config concern, out of scope here.** `sase-org/sase` master CI is `cancelled` 96 times out
  of 100 because cancel-in-progress concurrency applies to master pushes. That means master CI results are almost never
  authoritative for anyone, not just this chop. Worth evaluating separately whether cancel-in-progress should apply to
  master at all; this chop cannot fix it and should not try.
- The seconds-wide race between the in-script idle check and the runner launching a proposal remains accepted and
  documented, unchanged from `sase-a2`.
- Keep the zero-agents gate in-script. `inhibit_if` would suppress observation and merging too, and `wait_on` cannot
  express "wait for system idle".
