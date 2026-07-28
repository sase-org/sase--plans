---
tier: epic
title: Repair the three failing CI jobs on master
goal: 'CI on sase-org/sase master goes green: `lint` resolves epic-symbol beads instead
  of failing on every `--epic-symbol` entry, `published-core-minimum-smoke` finds
  every required `sase_core_rs` binding in the published minimum, and `phase7-perf-floor`
  stops tripping on documented notification-store runner variance.

  '
phases:
- id: beadstore
  title: Resolve the bead store from a plain checkout
  depends_on: []
  size: medium
  description: 'beadstore: teach bead-store location resolution to honor a checkout-local
    `.sase/sdd-store.json` sidecar record when no SASE workspace context resolves,
    so `sase bead show` works in CI and every `--epic-symbol` entry stops reporting
    a missing bead.

    '
- id: corefloor
  title: Publish sase-core 0.11.3 and raise the declared minimum
  depends_on: []
  size: small
  description: 'corefloor: land the open sase-core release so the five `plan_reference_*`
    bindings reach PyPI, then raise sase''s declared `sase-core-rs` floor and its
    tracking test to the published version.

    '
- id: perffloor
  title: Give the mark_all_read anchor a documented variance factor
  depends_on: []
  size: small
  description: 'perffloor: add a per-anchor `rust_slowdown_factor` override with a
    written rationale for the one write-heavy notification anchor whose ceiling has
    no slack, matching the mechanism already used for its two sibling anchors.

    '
- id: verify
  title: Confirm master is green and triage residual test flakes
  depends_on:
  - beadstore
  - corefloor
  - perffloor
  size: small
  description: 'verify: confirm a full CI run passes with all three fixes present,
    and determine whether the two `test (3.14)` failures seen once on master are flakes
    or real defects.

    '
create_time: 2026-07-27 12:01:22
status: wip
bead_id: sase-a0
---

- **PROMPT:** [202607/prompts/fix_ci_failures.md](prompts/fix_ci_failures.md)
- **AGENTS:**
  - [bbugyi200.athena.ma](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.ma/README.md)
  - [bbugyi200.athena.sase-a0.5.2--1](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-a0.5.2.md#member-1)

# Plan: Repair the three failing CI jobs on master

CI on `sase-org/sase` master is red. `actstat` and the job logs show three independent failures, plus one unconfirmed
test failure. Each has a distinct root cause; none of them share code, so the three fix phases are independent and can
run in parallel.

A note on why so many jobs show as cancelled in `actstat` output: `.github/workflows/ci.yml` sets
`concurrency: cancel-in-progress: true` per ref. Push volume on master is high enough that most runs are superseded
before they finish, so a job showing `⊘ cancelled` is usually not a signal about that job's health. Only jobs with an
explicit `failure` conclusion are diagnosed here.

## Evidence baseline

Job conclusions across the five most recent settled CI runs on master:

| Run         | lint      | published-core-minimum-smoke | phase7-perf-floor | test (3.14) |
| ----------- | --------- | ---------------------------- | ----------------- | ----------- |
| 30279783020 | failure   | failure                      | —                 | cancelled   |
| 30279340202 | cancelled | failure                      | failure           | cancelled   |
| 30278943499 | cancelled | failure                      | success           | cancelled   |
| 30278336216 | failure   | failure                      | success           | cancelled   |
| 30274179282 | failure   | failure                      | success           | failure     |

`lint` and `published-core-minimum-smoke` fail every time they are allowed to finish. `phase7-perf-floor` failed once
out of five. `test (3.14)` failed once.

## beadstore: Resolve the bead store from a plain checkout

### Symptom

The `lint` job fails at step 17 (`just lint`), in the `_lint-symvision` recipe:

```
Error: --epic-symbol 'sase-9z(ParsedPlanReference)': bead 'sase-9z' not found. Remove this --epic-symbol entry.
Error: --epic-symbol 'sase-9z(canonicalize_plan_reference)': bead 'sase-9z' not found. Remove this --epic-symbol entry.
Error: --epic-symbol 'sase-9z(render_plan_reference)': bead 'sase-9z' not found. Remove this --epic-symbol entry.
Error: --epic-symbol 'sase-9z(resolve_plan_reference_resolution)': bead 'sase-9z' not found. Remove this --epic-symbol entry.
error: Recipe `_lint-symvision` failed on line 243 with exit code 1
```

Bead `sase-9z` is not missing. It exists and is open (`status: claimed`, tier `epic`) in the `sase-org/sase--plans`
sidecar, committed at 12:40 UTC on 2026-07-27 — nearly three hours before the 15:28 UTC run that reported it missing.
`sase bead show sase-9z` succeeds locally.

### Root cause

`symvision` validates each `--epic-symbol <bead>(<symbol>)` entry by shelling out to `$BD_COMMAND show <bead_id>` and
treating any non-zero exit as "bead not found" (`symvision/cli.py`, the `--epic-symbol` validation loop). The `Justfile`
`_lint-symvision` recipe sets `BD_COMMAND=tools/sase_bead`, and `tools/sase_bead` runs `sase bead show`. Because
`symvision` invokes the command with `capture_output=True`, the real error message is swallowed and never reaches the CI
log — which is why the log jumps straight from the command line to symvision's own generic diagnostic.

The real error is that `sase bead show` cannot find the bead store at all. In `src/sase/bead/cli_common.py`,
`resolve_beads_location()` only reaches the sidecar-store branch after `_resolve_workspace_context()` returns a context.
That helper needs one of:

1. a `.sase/checkout.json` workspace marker (`find_marker_from_cwd`),
2. a `~/.sase/projects/*/registry.json` scan hit (`scan_projects_for_cwd`), or
3. a resolvable primary workspace (`resolve_primary_workspace`).

The CI checkout at `/home/runner/work/sase/sase` has none of these: it is a plain `actions/checkout` clone on a runner
with a fresh `HOME`. So `_resolve_workspace_context()` returns `None` and resolution falls through to
`_resolve_legacy_beads_location()`, which only walks parents looking for in-tree `sdd/beads` or `.sase/sdd/beads`. This
repo has neither — beads live in the plans sidecar. `resolve_beads_location()` therefore returns `None`,
`sase bead show` exits 1, and every `--epic-symbol` entry is reported as a missing bead.

The `.sase/sdd-store.json` record that the `lint` job writes (the "Record split SDD sidecar store" step) is never
consulted, because `read_sdd_store_record` is only reached from inside the workspace-context branch, via
`resolve_sdd_store` →`_resolve_sdd_storage`, which resolves the record path through `get_primary_workspace_dir`. Writing
that file has no effect on a checkout that is not a recognized workspace.

### Reproduction

The failure reproduces outside CI by recreating the CI layout — a directory that is not a SASE workspace, containing
`.sase/sdd-store.json` with `storage: sidecar_repos` and a real plans sidecar clone at `sase/repos/plans`, run with an
empty `HOME`:

```
Error: issue not found: sase-9z
EXIT=1
```

The phase agent should reproduce this first and keep it as the acceptance check.

### Why this has never worked

This is not a new regression. `--epic-symbol` has never worked in CI. The same `bead '<id>' not found` failure appears
for unrelated beads on unrelated days — for example bead `sase-99` in run 30160372882 on 2026-07-25. The `lint` job only
ever passes when the `Justfile` happens to carry zero `--epic-symbol` entries, which is why the breakage looks
intermittent when read from run history alone. Any epic that whitelists a symbol turns master red until the entries are
removed.

### Fix

Add a resolution tier to `resolve_beads_location()` in `src/sase/bead/cli_common.py`, between the workspace-context
branch and the `_resolve_legacy_beads_location()` fallback:

- Walk from the resolved cwd up through its parents (mirroring the existing parent walk in
  `_resolve_legacy_beads_location`) looking for a checkout root that carries `.sase/sdd-store.json`.
- Read it with the existing `read_sdd_store_record` helper from `src/sase/sdd/_store_records.py` rather than parsing
  JSON by hand, so schema handling stays in one place.
- When the record's `is_sidecar_storage` is true, resolve the plans root with the existing
  `sidecar_repo_clone_dir(<checkout_root>, "plans")` from `src/sase/_linked_repo_paths.py` (`SIDECAR_REPO_CLONES_SUBDIR`
  is `("sase", "repos")`, so this yields `<checkout_root>/sase/repos/plans` — exactly where CI checks the sidecar out).
  The beads directory is `<plans_root>/beads`, matching `SddStore.kind_root("beads")` for sidecar storage.
- Honor `require_existing` the same way the workspace branch does: return `None` when the beads directory is absent, so
  callers that probe cheaply keep their current behavior.
- Return a `_BeadsLocation` with `beads_dirname=BEADS_DIRNAME_NON_VC` and `storage=SDD_STORAGE_SIDECAR_REPOS`,
  consistent with the workspace-context sidecar branch.

Constraints for the implementing agent:

- **Reads only.** `resolve_beads_location()` also feeds write and auto-commit paths. This fallback must not make a plain
  checkout a legitimate write target. Verify that `assert_bead_store_write_sandboxed` still refuses fast-path mutations
  from a non-workspace checkout, and that `auto_commit_bead_store` is not newly reachable there. If honoring that
  requires threading a read-only flag through, do it explicitly rather than relying on callers.
- **No Rust change needed.** Per the repo's core-backend boundary rule, check the boundary before writing code — but the
  whole SDD store discovery chain (`sase.sdd.store`, `sase.bead.cli_common`, `sase._linked_repo_paths`) already lives in
  Python in this repo, and the fix is local path/config resolution, not domain behavior. No `sase_core_rs` wire or
  binding change is expected. Do not add a new binding for this.
- **Do not weaken the epic-symbol gate.** Removing the `--epic-symbol` entries from the `Justfile` would turn `lint`
  green today, but bead `sase-9z` is still open with phase `sase-9z.4` in progress, so the four whitelisted symbols are
  legitimately awaiting their consumer. Removing them would force deleting live symbols. Fix the resolution, not the
  whitelist.
- **Do not patch the installed `symvision` package.** Per the symvision guidance, fix sase's code; symvision's behavior
  here is correct given a failing `bd` command.

### Verification

- A regression test that constructs a temp checkout (no workspace marker, no project registry) with
  `.sase/sdd-store.json` declaring sidecar storage and a beads directory under `sase/repos/plans/beads`, then asserts
  the bead store resolves and a bead read succeeds. Include the negative case: no record file, or a record that does not
  declare sidecar storage, still falls through to the legacy resolver.
- A test asserting a non-workspace checkout is still refused as a write target.
- Re-run the manual CI-layout reproduction and confirm it now exits 0.
- `just install` then `just check` (ephemeral workspaces may have drifted dependencies).
- Confirm `just _lint-symvision` still passes locally with the four `sase-9z` entries in place.

## corefloor: Publish sase-core 0.11.3 and raise the declared minimum

### Symptom

`published-core-minimum-smoke` fails at step 7 on every run:

```
sase_core_rs 0.11.2 is missing 5 of 193 required binding(s):
  plan_reference_canonicalize
  plan_reference_parse
  plan_reference_render
  plan_reference_resolution_wire_schema_version
  plan_reference_resolve
```

### Root cause

The gate is correct and is catching a real defect. `pyproject.toml` line 46 declares `"sase-core-rs>=0.11.2,<0.12.0"`.
The job installs the exact declared minimum (0.11.2, which is also the newest version on PyPI) and checks that every
name passed to `require_rust_binding` in `src/sase` exists in it.

The five `plan_reference_*` bindings are defined in the sibling core repo at `crates/sase_core_py/src/lib.rs`, but only
on commits that landed _after_ the v0.11.2 release commit:
`1136e72 feat(plan): add durable plan reference contract (sase-9z.1)`,
`96f8bc4 fix(bead): canonicalize created plan design refs (sase-9z.3)`, and
`b5c37cc feat(beads): validate doctor plan references (sase-9z.5)`. Local development never notices, because
`just install` builds the core from the sibling checkout. Every _published_ sase install would hit `AttributeError` on
those paths — which is precisely the failure class this gate was added to prevent (see the module docstring in
`tools/check_sase_core_rs_bindings`, which cites the 0.11.0/0.6.0 `aggregate_clan_runtime` incident).

### Fix

This phase spans two repositories. Use the `/sase_repo` skill to open `sase-core` and work only from the path it prints.

1. In `sase-core`, release-plz has an open release PR: **#35, `chore: release v0.11.3`** (branch
   `release-plz-2026-07-27T13-26-47Z`, touching `Cargo.toml`, `Cargo.lock`, and the two `CHANGELOG.md` files). Confirm
   it is current with master — it was opened at 13:26 UTC and `b5c37cc` may have landed afterward. If the PR predates
   any of the three binding commits, let release-plz refresh it rather than hand-editing the version. Note that
   `release-plz.toml` sets `features_always_increment_minor = false`, which is why two `feat` commits still produce a
   **patch** bump to 0.11.3 — inside sase's existing `<0.12.0` range. Do not assume 0.12.0.
2. Merge the release PR and let `.github/workflows/release-plz.yml` tag `v0.11.3` and publish the `sase-core-rs` wheels
   to PyPI. The workflow already guards on whether the version is absent from PyPI, and exposes manual `publish_pypi` /
   `expected_version` inputs for recovery if the automatic publish does not fire.
3. Wait for `sase-core-rs==0.11.3` to be installable from PyPI. Confirm before proceeding: install it into a scratch
   venv and run `tools/check_sase_core_rs_bindings` against it — the same check CI runs. It must report zero missing
   bindings.
4. Only then, in the sase repo, raise the floor in two places that must stay in sync:
   - `pyproject.toml` line 46 → `"sase-core-rs>=0.11.3,<0.12.0"`
   - `tests/test_sase_core_rs_telemetry_smoke_tool.py` line 33, which asserts
     `declared_minimum_version(...) == "0.11.2"` → `"0.11.3"` Refresh `uv.lock` accordingly. Commit
     `fa07151cf test: track sase-core-rs 0.11.2 minimum (sase-9x)` is the precedent for the shape of this change.

### Blocking risk

Step 4 cannot be verified until step 2's publish completes, and the publish is not under this repo's control. If the
core release stalls, land the other two phases anyway and report the core release as the remaining blocker rather than
bumping the floor to an unpublished version — that would turn the gate's failure into a silent install failure for
users.

### Verification

- `tools/check_sase_core_rs_bindings` reports zero missing bindings against a venv holding the exact new declared
  minimum.
- `just install` then `just check`.

## perffloor: Give the mark_all_read anchor a documented variance factor

### Symptom

`phase7-perf-floor` failed in run 30279340202:

```
[FAIL] notification_store.synthetic_5k.notification_store_5k_mark_all_read:
       rust=194856.24us python=n/a ceiling=100359.00us confirmation=157548.52us
       FAIL: absolute floor confirmed: initial rust median 194856.24us and confirmation 157548.52us exceed ceiling 100359.00us
```

### Root cause

Runner IO variance, not a code regression. The evidence is that the whole write-heavy family was elevated together in
that one run, while the read-only anchor was not:

| Anchor                             | run 30279340202 (failing) | run 30278943499 | run 30278336216 |
| ---------------------------------- | ------------------------- | --------------- | --------------- |
| `..._load_snapshot` (read-only)    | 9,567us                   | 14,026us        | 14,438us        |
| `..._mark_dismissed_burst`         | 153,211us                 | 48,800us        | 48,838us        |
| `..._mark_all_read`                | **194,856us**             | 50,219us        | 44,870us        |
| `notification_modal_dismiss_burst` | 222,726us                 | 158,815us       | 163,119us       |

`mark_dismissed_burst` was 3.1x its usual median in the same job, and `mark_all_read` 4.1x, while `load_snapshot` was
actually _faster_ than usual. A code regression in read-state mutation would not speed up snapshot loading, and would
reproduce across runs — it does not: `mark_all_read` passed at 44.9/48.3/50.2ms in the three neighboring runs.

`mark_all_read` is the only member of that family that trips, because it is the only one with a tight ceiling. Its
ceiling is 100,359us — the Phase 7B baseline of 71,685us times the global 1.40x factor — about 2.1x its typical current
median. Its siblings carry enormous slack: `mark_dismissed_burst`'s ceiling is 6,811,056us (≈140x its typical median)
and `modal_dismiss_burst`'s is 5,098,492us. So a runner-wide IO spike that lifts the entire family 3-4x only ever
surfaces on this one anchor.

The existing bounded-confirmation mechanism (`_apply_notification_confirmation` in
`tests/perf/phase7/regression_floor.py`) did not filter this, because the slow window outlasted the job: the
confirmation re-run also came in at 157,548us.

### Fix

Use the mechanism the repo already established for exactly this class of anchor. In
`tests/perf/baselines/phase7_regression_floor.json`, add an entry under `tolerance.per_anchor_rust_slowdown_factors`
keyed by the full anchor id `notification_store.synthetic_5k.notification_store_5k_mark_all_read`, with a
`rust_slowdown_factor` and a `comment` recording the reasoning.

This is the same shape as the two existing overrides in that file:
`notification_store.synthetic_5k.notification_modal_dismiss_burst` at 1.6x and
`scan_agent_artifacts.synthetic_6p_200pp.scan_facade` at 1.8x, each with a written rationale. `_load_baseline` in
`tests/perf/phase7/regression_floor.py` validates override keys against known anchor ids and errors on an unknown id, so
the key must match exactly.

Suggested factor: **3.0x**, giving a ceiling of ~215,055us. Rationale to record in the comment: the observed spike
reached 194,856us with a 157,548us confirmation while sibling write-heavy anchors were simultaneously elevated 3-4x; the
anchor's typical CI median is ~47,500us, so a 3.0x factor still leaves roughly a 4.5x detection margin for a genuine
regression, and this remains by a wide margin the tightest gate in the notification-store family. The phase agent should
confirm the arithmetic against the baseline's `phase7b_rust_median_s` of 0.071685 and adjust if it prefers a tighter
number that still clears the observed spike — but it must clear 194,856us, or the same run would fail again.

Note the tolerance block's own `comment` states that per-anchor overrides "are reserved for explicitly documented
high-variance anchors and do not weaken the stable anchors" — this anchor now qualifies on the evidence above, and the
global 1.40x gate is untouched.

### Verification

- `just check`.
- Run the Phase 7E floor check locally and confirm the anchor reports the new per-anchor factor in its note line, the
  way the two existing overrides do.
- Do not hand-edit `sdd/plans/202604/perf_artifacts/rust_backend_phase7_floor_check.json`; it is the generated report.

## verify: Confirm master is green and triage residual test flakes

Once the three fix phases have landed, confirm a full CI run passes end to end. Because `cancel-in-progress` supersedes
most runs, check a run that actually settles rather than assuming from a cancelled one.

There is one unresolved item this phase must also close out. In run 30274179282, `test (3.14)` failed with two failures:

```
FAILED tests/ace/tui/test_statistics_pane_loading.py::test_auto_refresh_soak_keeps_event_loop_and_message_pump_responsive - AssertionError: assert 3 >= 4
FAILED tests/test_fork_workflow.py::test_inline_deferred_fork_survives_workspace_removal_and_late_preprocessing - AssertionError: assert False
==== 2 failed, 22261 passed, 16 skipped, 89 warnings in 2302.41s (0:38:22) =====
```

These were not diagnosed as part of the three fixes, and there is no clean second data point because every later `test`
job was cancelled before finishing. Assessment so far:

- The first is a soak test asserting a refresh count (`assert 3 >= 4`) under a 38-minute loaded parallel run — the shape
  of a timing flake, and its most recent code change (`937016404 test: split statistics pane coverage by behavior`) was
  a test reorganization.
- The second is less clearly a flake. It is worth noting that `05b45da01 fix(xprompt): preserve fork query spacing` was
  already an ancestor of the commit that failed, so that fix is not the remedy.

Determine whether each is a flake or a real defect: re-run the 3.14 matrix leg on current master, and if a failure
reproduces, fix it or file it as a tracked bead with the evidence. Do not paper over a reproducible failure by
re-running until green.
