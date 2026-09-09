---
tier: epic
status: done
title:
  Finish the release gate — repair the chop's per-repo scoping, green both lanes, and
  ship v0.17.0
goal: "The three release repositories sase-org/sase, sase-org/sase-github, and
  sase-org/sase-telegram each merge their release PRs again under the merge strategy and
  gating workflows that repository actually has; Full CI and the Master Gate are both
  green on the tip; and sase v0.17.0 is tagged and published to PyPI.

  "
phases:
  - id: chopscope
    title: Scope ci_watch's release-gate variables per repository
    depends_on: []
    size: large
    description:
      "chopscope: make ci_watch's merge_method, gating_workflows, heavy_workflows, and
      heavy_max_age_hours resolvable per repository instead of chop-global, release
      bugyi-chops, roll the chezmoi config forward, and prove sase-telegram PR #21 and
      sase-github stop reporting gating_workflow_missing."
  - id: heavy
    title: Drive Full CI green
    depends_on: []
    size: medium
    description:
      "heavy: fix the three drifted ACE PNG goldens that reproduce deterministically on
      the tip, attribute the test (3.13) and coverage-contexts failures that pass
      serially, and confirm a completed Full CI run is green."
  - id: gatewall
    title: Bring the Master Gate to a durable green inside its 8-minute p50 budget
    depends_on: []
    size: medium
    description:
      "gatewall: attribute and fix the intermittent Master Gate shard failures, cut the
      gate's p50 wall from 10.3 to 8 minutes, and give tests/shard_timings.json a
      freshness path so the shard split cannot silently decay."
  - id: ship
    title: Ship v0.17.0 and re-measure every acceptance criterion
    depends_on:
      - heavy
      - gatewall
    size: medium
    description:
      "ship: watch ci_watch reach eligible on sase-org/sase, confirm the guarded merge
      of PR #284 succeeds with --merge --match-head-commit, confirm publish.yml tags and
      publishes v0.17.0 to PyPI, and re-measure all seven acceptance criteria against
      live data."
proposed_by: bbugyi200.athena.sase-um.land
parent_bead: sase-um
bead_id: sase-um.9
create_time: 2026-09-09 19:51:20
---

- **PROMPT:**
  [prompts/202608/release_gate_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/release_gate_completion.md)
- **PARENT:** [202608/release_gate_liveness.md](release_gate_liveness.md)
- **BEAD:**
  [sase-um.9](https://github.com/sase-org/sase--beads/blob/main/pages/sase-um/sase-um.9.md)

# Plan: Finish the release gate

## 1. Problem

Epic `sase-um` built and verified every piece of release-gate machinery its plan called
for: `master-gate.yml` (per-SHA, never cancelled), `full.yml` (scheduled heavy lane),
`publish.yml` throttled onto a schedule, `sase-core-revision.txt` plus
`tools/ratchet_core_revision`, and a `ci_watch` that decides release eligibility from a
`gating_workflows` allowlist and a heavy-lane freshness window. All of it works.

The epic still cannot close, for four reasons found while landing it. The first is a
regression the epic itself introduced and is the most urgent, because it is silently
shutting releases in two repositories that were fine before.

### 1.1 The chop's release-gate variables are chop-global, but it sweeps three repositories

`ci_watch` reads `merge_method`, `gating_workflows`, `heavy_workflows`, and
`heavy_max_age_hours` as flat scalars on `Config` (`src/bugyi_chops/ci_watch.py`, fields
at ~995-1047) and applies `config.merge_method` / `config.gating_workflows` to every
repository in `release_repositories` (~2017, ~2069-2093). Phase `config` of the parent
epic set them chop-wide:

```yaml
merge_method: merge
gating_workflows: ["Master Gate"]
heavy_workflows: ["Full CI"]
heavy_max_age_hours: 6
```

That is correct for `sase-org/sase` and wrong for the other two release repositories the
same chop sweeps. Measured 2026-08-28:

| Repository               | `allow_merge_commit` | `allow_squash_merge` | Workflows               |
| ------------------------ | -------------------- | -------------------- | ----------------------- |
| `sase-org/sase`          | true                 | false                | …, Master Gate, Full CI |
| `sase-org/sase-github`   | **false**            | **true**             | CI, PR Title, Publish   |
| `sase-org/sase-telegram` | **false**            | **true**             | CI, PR Title, Publish   |

So for both plugin repos:

- `_release_gate_reason` returns `gating_workflow_missing` on every tick, because
  `"Master Gate"` is not in `evidence.observed_workflows` and the predicate is
  `any(workflow not in observed ...)`. The gate can never open.
- `_evaluate_heavy_lane` returns `heavy_lane_not_green` for the same reason —
  `_newest_completed_run(runs, "Full CI")` is `None`.
- Even past those, `gh pr merge --merge` would fail against `allow_merge_commit: false`
  — the exact latent-merge-failure class the parent epic existed to remove, now mirrored
  onto the two repos that were previously merging fine.

This is live, not hypothetical: `sase-org/sase-telegram` PR #21
(`chore(master): release 0.4.10`) has been open since 2026-08-27T02:46Z, the day after
the rollout, and `ci_watch_state.json` records no merge for it. The parent plan's stated
safety property — "When it is empty (the default), behavior is exactly as today, so the
chop stays safe for the other repos it sweeps" — holds only while the vars are empty,
and the rollout filled them in for everyone at once.

### 1.2 Full CI is red, which holds the sase release shut

Newest completed `Full CI` run 33167273442 (2026-08-28T11:28-13:11Z) failed three jobs:

| Job                 | Failing node(s)                                                                                                                                                                                                                                                                                                                   |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `visual-test`       | `test_ace_png_snapshots_help_panel.py::test_help_panel_keymaps_png_snapshot`, `test_ace_png_snapshots_plan_toast.py::test_epic_plan_toast_png_snapshot`, `test_ace_png_snapshots_update_toast.py::test_startup_update_toast_png_snapshot`                                                                                         |
| `test (3.13)`       | `tests/test_ace_testing.py::test_ace_page_group_rejects_overlapping_checkouts`                                                                                                                                                                                                                                                    |
| `coverage-contexts` | `tests/ace/tui/widgets/test_tools_panel_event_loop.py::test_warm_cache_update_display_does_not_walk_artifacts_on_event_loop`, `tests/fakey/test_pipe_e2e.py::test_default_pipe_creates_family_member_with_fork_and_shared_workspace`, `tests/test_models_panel_layout.py::test_panel_preferred_width_fits_production_description` |

Because `ci_watch` now requires a green `Full CI` inside `heavy_max_age_hours: 6`, this
red lane is what keeps `sase-org/sase` PR #284 (`chore(master): release 0.17.0`, open
since 2026-08-07 and MERGEABLE/CLEAN) unmerged and `v0.17.0` unpublished — PyPI's latest
is still `0.16.0`.

### 1.3 The Master Gate is not durably green and misses its own p50 target

Over the last 50 gate runs on master: 39 success, 10 failure, 1 cancelled; median wall
10.32 min against the parent plan's ≤ 8 min acceptance criterion. Two of the failures
measured while landing were already fixed by later commits (`22f722168` retired the
stale `sase-ud(question_next_action)` Symvision epic-symbol that reddened `lint`), but
`test (5)` and `test (2)` still fail intermittently.

### 1.4 v0.17.0 has not shipped

Phase `verify` of the parent epic measured all seven acceptance criteria on 2026-08-28
and recorded 3 pass, 1 partial, 3 fail. The failures are criteria 1, 2, 5 and 7 — one
cancelled gate run, the 10.32-minute median, the unmerged release PR, and the
unpublished tag. Criterion 4 is partial: `ci_watch` now reports heavy-lane reasons
instead of `default_branch_not_green`, which proves the new gating logic fires, but the
decision has never reached `eligible`.

## 2. What is already done and must not be re-litigated

The parent epic's machinery is verified working end to end. Do not rebuild any of it:

- `.github/workflows/master-gate.yml` — per-SHA group `master-gate-${{ github.sha }}`
  with `cancel-in-progress: false`, `timeout-minutes` on every job, six shards fed by
  `tests/shard_timings.json`, core wheel from a revision-keyed cache.
- `.github/workflows/full.yml` — `schedule: '17 */2 * * *'` plus `workflow_dispatch`,
  calling `ci.yml` with `concurrency: full-ci`.
- `.github/workflows/ci.yml` — `pull_request` + `workflow_call` only, per-ref
  latest-wins.
- `.github/workflows/publish.yml` — release-please and `sync-release-metadata` on
  `schedule: '17 */3 * * *'` plus `workflow_dispatch`.
- `sase-core-revision.txt`, `tools/ratchet_core_revision`, `core-pin-ratchet.yml`, and
  `tools/check_sase_core_rs_bindings --remedy`.
- `tools/fetch_coverage_contexts` repointed at `full.yml`; both README badges present.
- `ci_watch`'s `gating_workflows` / heavy-freshness / `merge_method` implementation and
  its 104-test suite in `bbugyi200/bugyi-chops` 0.8.0.

The parent plan's rejected-alternatives list (section 8) and its deferred items
(section 9) also stand. The deferred items are now filed: `sase-v7` (two-speed CI
decision record), `sase-v8` (heavy-lane cost, R7), `sase-v9` (ci_watch watched-workflow
notification list).

## 3. Phases

### Scope ci_watch's release-gate variables per repository

All in `bbugyi200/bugyi-chops`, opened with `/sase_repo`
(`sase repo open gh:bbugyi200/bugyi-chops`).

**The shape.** Let each of `merge_method`, `gating_workflows`, `heavy_workflows`, and
`heavy_max_age_hours` accept either today's flat scalar/list (meaning "the default for
every release repository") or a mapping keyed by repository slug with an optional
default. The parent plan already anticipated exactly this — its phase `config` says "if
Phase `chop` chose per-repo mappings rather than flat lists, mirror that here" — so this
is completing a shape the plan left open, not reversing a decision. Pick one form and
document it in the README; a mapping with a reserved default key, for example:

```yaml
merge_method:
  default: squash
  sase-org/sase: merge
gating_workflows:
  sase-org/sase: ["Master Gate"]
heavy_workflows:
  sase-org/sase: ["Full CI"]
```

Resolution rules that must hold:

- A bare scalar or list keeps working and keeps meaning "every repository", so no other
  host's configuration breaks.
- A repository with no entry and no `default` resolves to today's built-in default:
  empty allowlists (actstat-only behavior) and `merge_method` — see the next bullet.
- **Reconsider the `merge_method` default.** It is `merge` today, and that default is
  what silently broke the two squash-only repositories. Either default it per repository
  from the GitHub API's `allow_merge_commit` / `allow_squash_merge` /
  `allow_rebase_merge` flags (one bounded `repos/{repo}` read, cached per tick), or keep
  an explicit default and add a **pre-merge preflight** that refuses to submit a merge
  whose method the repository does not allow, reporting a distinguishable reason such as
  `merge_method_not_allowed`. The preflight is the more valuable of the two because it
  fails loudly at plan time instead of burning a merge attempt, and it would have caught
  this regression on the first tick. Implementing both is fine.

**Tests.** `tests/test_ci_watch.py` is the contract. Add coverage for: a flat scalar
applying to every repo (today's behavior, unchanged); a mapping selecting different
methods and allowlists per repo in one tick; a repo absent from the mapping falling back
to `default` and then to the built-in; an unknown key or malformed mapping failing
closed through `Config.from_invocation` the way `merge_method: fast-forward` already
does; and the preflight refusing a disallowed method with its own reason string. Keep
every existing assertion green — the flat form is not being removed.

**Release and rollout.** bugyi-chops releases are tag-driven: bump `version` in
`pyproject.toml` (0.8.0 → 0.9.0), push the matching `v<version>` tag, let `publish.yml`
ship it. Then open the chezmoi repo with `/sase_repo` and update the `ci_watch` block in
`sase_athena.yml` (the chezmoi source path, not the materialized
`~/.config/sase/sase_athena.yml`) so that `sase-org/sase` keeps exactly today's
effective settings — `merge`, `["Master Gate"]`, `["Full CI"]`, `6` — and
`sase-org/sase-github` and `sase-org/sase-telegram` get `squash` with empty allowlists,
which is what they had before the parent epic's rollout. Install the released plugin,
apply the overlay, and run `sase chop doctor`.

**Acceptance.** A dry-run tick reports, for `sase-org/sase-telegram`, a reason that is
_not_ `gating_workflow_missing` and not `heavy_lane_not_green`; and `sase-org/sase`'s
reason is unchanged from before the rollout. If `sase-telegram` PR #21 is otherwise
ready, the live chop merges it and `ci_watch_state.json` records a
`squash_merge_submitted` outcome for that repo — its first release since 0.4.9 on
2026-08-22.

### Drive Full CI green

**The three visual goldens reproduce deterministically.** Verified 2026-08-28 on a clean
tree at master `52327ed78` after `just install`:

```bash
./.venv/bin/python -m pytest -q -p no:randomly -m visual \
  tests/ace/tui/visual/test_ace_png_snapshots_help_panel.py::test_help_panel_keymaps_png_snapshot \
  tests/ace/tui/visual/test_ace_png_snapshots_plan_toast.py::test_epic_plan_toast_png_snapshot \
  tests/ace/tui/visual/test_ace_png_snapshots_update_toast.py::test_startup_update_toast_png_snapshot
# 3 failed in 9.26s
```

They are three _different_ drifts, not one shared chrome shift — 55538, 3717 and 1707
changed pixels respectively — so judge each one on its own artifacts in
`.pytest_cache/sase-visual/`:

- `help_keymaps_changespecs_120x40`: the diff is the whole keymap list. Rows shift and
  new entries appear (`Toggle stitches collapsed/exp…`,
  `Toggle hooks collapsed/expanded`, `Toggle mentors collapsed/expa…`,
  `Expand all grouping banners`). Confirm against `src/sase/ace/tui/modals/help_modal/`
  that these entries were added deliberately before rebaselining.
- `startup_update_toast_120x40`: the diff is confined to the Artifacts sub-tab strip row
  — the agents-first reorder (`4dd299502`) that phase `visual` of the parent epic
  rebaselined 357 goldens for. This golden was simply missed by that sweep.
- `plan_toast_epic_120x40`: diagnose from its own artifacts; do not assume it shares a
  cause with either of the others.

Rebaseline only what you have traced to an intended change, per the standing lesson on
`sase-up`: a blanket `--sase-update-visual-snapshots` would bless a real regression
hiding among them. Note also that `sase-up` (the 360/840 visual-lane bead) already
carries a note recording that these three are being handled here.

**The four non-visual nodes pass serially on the tip.** Verified in the same clean
workspace:

```bash
./.venv/bin/python -m pytest -q -p no:randomly \
  tests/test_ace_testing.py::test_ace_page_group_rejects_overlapping_checkouts \
  tests/ace/tui/widgets/test_tools_panel_event_loop.py::test_warm_cache_update_display_does_not_walk_artifacts_on_event_loop \
  tests/test_models_panel_layout.py::test_panel_preferred_width_fits_production_description
# 3 passed in 4.61s
```

So each is either fixed by one of the 20 commits that landed after the failing run's SHA
`65e09744a`, or is lane-specific — `test (3.13)` is `just test-cost` and
`coverage-contexts` is the contexts lane, neither of which the fast gate runs. Attribute
rather than assume: reproduce each under the lane that failed it (`just test-cost` /
`just test-contexts`, or the exact node under the 3.13 interpreter) before deciding.
`tests/fakey/test_pipe_e2e.py::test_default_pipe_creates_family_member_with_fork_and_shared_workspace`
is already a known flake with a ready bead (`sase-r2`) and a
`tests/reproducible_flake_baseline.txt` entry — corroborate it there, do not re-file it.
Anything that fails and then passes on an unchanged tree gets a `flake` task bead
through `/sase_new_task`, never a mute.

**Exit condition.** A completed `Full CI` run on master is green. Because the lane runs
every two hours and takes ~1h43m, use `workflow_dispatch` to get a run rather than
waiting on the schedule, and drive it through `/sase_monitor` — it will outrun a single
agent turn.

### Bring the Master Gate to a durable green inside its 8-minute p50 budget

**Attribution first.** The gate is per-SHA, so every failure names exactly one commit.
Work newest-first so the tip converges rather than the history:

```bash
gh run list --workflow=master-gate.yml --branch=master --limit 50 \
  --json conclusion,createdAt,databaseId,headSha
gh run view <id> --log-failed
```

Recent failures cluster on `test (5)` and `test (2)`. One `lint` failure class is
already fixed — a stale `--epic-symbol 'sase-ud(question_next_action)'` entry, retired
by `22f722168` once `sase-ud` closed — which is worth remembering as a recurring shape:
a Justfile epic-symbol entry outlives its bead and reddens the gate for every commit
until someone removes it.

**Then the wall time.** Measured per-job on three green runs, the gate's critical path
is `core-wheel` (0.9-2.9 min on a cache hit) plus the slowest shard (6.1-13.4 min), and
the shards are already reasonably balanced (most legs 5-9 min). So the 10.32-minute
median is structural, not an imbalance. The levers, in the order worth trying:

1. Raise `SHARD_COUNT` from 6. The parent plan's guardrail is ≤ 60 job-minutes per
   commit; today's gate is roughly 8 shards' worth of budget away from that, so 8 shards
   is affordable. Re-derive the number from measured data rather than guessing.
2. Cut the per-shard setup. Each `test` leg pays `setup-sase` with
   `install-recipe: install-visual` even though the gate runs no visual tests; check
   whether the plain install recipe is enough and what it saves.
3. Only then consider restructuring `needs: core-wheel`.

**Then stop the shard table from decaying.** `tests/shard_timings.json` was written
once, by the gate's own landing commit `5d8872f4d`, from a host-local recording store
(`tools/refresh_shard_timings`). It retains the 800 slowest files individually and gives
every other file a 0.131 s default; 47 test files have been added since and nothing
re-generates it. The parent plan's preferred design was to have the heavy lane publish
its per-file duration table as a run artifact and have the gate consume the newest one;
either implement that, or add a scheduled/ratcheted refresh with a `--check` gate, but
do not leave the table with no freshness path at all.

**Exit condition.** `Master Gate` is green on the tip for a majority of samples taken 10
minutes apart over an hour, and the median wall over the trailing 50 runs is ≤ 8
minutes.

### Ship v0.17.0 and re-measure every acceptance criterion

With both lanes green, `ci_watch` should reach `eligible` for `sase-org/sase` within a
few five-minute ticks and merge PR #284 with `gh pr merge --merge --match-head-commit`.
Watch it rather than forcing it: the merge succeeding _is_ acceptance criterion 5, and a
hand-merge would destroy the evidence the parent epic exists to produce.

If the gate opens and the merge still fails, that failure takes priority over everything
else in this plan — it is the blocker the whole parent epic was built to remove — and it
almost certainly belongs to phase `chopscope`'s work.

After the merge, `publish.yml` runs on its 3-hourly schedule; use `workflow_dispatch`
with `publish_existing` if the schedule is the only thing in the way. Then re-measure
and record all seven of the parent plan's acceptance criteria against live data:

1. Zero `cancelled` in
   `gh run list --workflow=master-gate.yml --branch=master --limit 50`. Note that the
   one cancellation in the previous measurement (33127407974) had `test (1)` already
   failing while sibling shards were cancelled, under `fail-fast: false` — the signature
   of a deliberate `gh run cancel`, not a push supersession. Attribute it explicitly
   rather than counting it as a gate defect.
2. Median `Master Gate` wall ≤ 8 min over 50 consecutive runs.
3. ≥ 90% of master commits in a 24-hour window have a completed gate run.
4. `ci_watch` reports a reason other than `default_branch_not_green` at least once a day
   and reaches `eligible` at least once.
5. A `gh pr merge` against the repository's allowed strategy succeeds — proven by the
   real release merge.
6. PR CI queue wait ≤ 1 min median.
7. `v0.17.0` tagged and published to PyPI.

Record the numbers, not just pass/fail.

## 4. Risks and safeguards

| Risk                                                                | Safeguard                                                                                                                                      |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| The per-repo config change breaks the flat form other hosts rely on | The flat scalar/list stays supported and keeps its meaning; existing `tests/test_ci_watch.py` assertions stay green unchanged.                 |
| Rebaselining the three goldens blesses a real regression            | Three separate diffs, each traced to a named source change before acceptance; `sase-up` records the same lesson from a 360-golden sweep.       |
| Master keeps moving, so "green on the tip" never settles            | The gate is per-SHA and fast, so a green tip re-establishes within minutes; sample over an hour rather than demanding a single instant.        |
| Raising `SHARD_COUNT` blows the job-minute guardrail                | The parent plan's ≤ 60 job-min/commit ceiling is the bound; measure before and after.                                                          |
| Full CI's ~1h43m runtime makes iteration slow                       | `workflow_dispatch` instead of waiting for the two-hour schedule, driven through `/sase_monitor`; `sase-v8` tracks cutting the runtime itself. |
| The release merges but publish is delayed by the 3-hourly schedule  | `workflow_dispatch` with `publish_existing` covers a manual cut, exactly as the parent plan intended.                                          |

## 5. Out of scope

Filed as task beads by the `sase-um` land agent; do not fold them back in: `sase-v6`
(`rust-lsp-install` copies from the wrong cargo target and exits 0 after the copy
fails), `sase-v7` (the two-speed-CI decision record, which needs the user's explicit
approval before any memory write), `sase-v8` (R7 heavy-lane cost), and `sase-v9`
(`ci_watch` watched-workflow notifications). `sase-up` keeps whatever visual-lane debt
remains after the three goldens here are fixed.
