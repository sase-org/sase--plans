---
tier: tale
title: Repair master CI so ci_watch can merge the v0.17.0 release PR
goal:
  "The CI workflow is green on a master push, unblocking the ci_watch chop so it merges
  pull request 284 (chore(master): release 0.17.0)."
size: medium
proposed_by: bbugyi200.athena.0c3
---

# Plan: Repair master CI so ci_watch can merge the v0.17.0 release PR

## Context

`actstat --repo sase-org/sase` reports master red on every run that reaches a terminal
conclusion. On the newest master run (`30e73cfb`, CI run `32669819975`) `build-core`,
`lint`, `perf-floors` and `ace-page-group-isolation` all pass; the five failing jobs are
`test (3.12)`, `test (3.13)`, `test (3.14)`, `coverage-contexts` and `visual-test`. No
master CI run has succeeded since at least 2026-08-21.

Pull request 284 (`release-please--branches--master`, "chore(master): release 0.17.0")
has its own CI jobs skipped by design — `build-core` carries
`if: ... head.ref != 'release-please--branches--master'` and the source lanes inherit
that exclusion — so the release PR is gated on master CI, not on its own checks. Making
master green is the whole job.

There are exactly three independent root causes. Each one is fully diagnosed below;
implement all three directly.

## Root cause A: the `/sase_hg_commit` removal left its consumers behind

`6e99c7fc6 chore: Remove /sase_hg_commit` (2026-08-23) deleted
`src/sase/xprompts/skills/sase_hg_commit.md` and nothing else. Four tests still reach
for it, and they fail on every test leg plus `coverage-contexts`:

- `tests/main/test_init_skills_source_content.py::test_commit_skill_sources_do_not_reference_legacy_bead_flag[sase_hg_commit]`
- `tests/main/test_init_skills_source_content.py::test_commit_skill_sources_reject_legacy_sase_commit_cli[sase_hg_commit]`
- `tests/main/test_init_skills_source_content.py::test_hg_commit_skill_invokes_stitch_create`
- `tests/main/test_init_skills_sources.py::test_docs_xprompt_bundled_skills_table_matches_packaged_sources`

The first three raise `FileNotFoundError` on
`get_sase_package_skills_dir() / "sase_hg_commit.md"`. The fourth compares the
`### Bundled Skills` table in `docs/xprompt.md` against `*.md` under the packaged skills
directory; the table still carries a `sase_hg_commit` row that no longer has a source.

### Fix A

1. In `tests/main/test_init_skills_source_content.py`, narrow both
   `@pytest.mark.parametrize("skill_name", ["sase_git_commit", "sase_hg_commit"])`
   decorators (currently at lines 62 and 108) to `["sase_git_commit"]`. Keep the
   parametrize decorator rather than inlining it, so a future second commit skill is one
   list entry away.
2. Delete `test_hg_commit_skill_invokes_stitch_create` in the same file (currently lines
   116-124). Its subject no longer ships.
3. Delete the `| sase_hg_commit | ... |` row from the `### Bundled Skills` table in
   `docs/xprompt.md` (currently line 1250). That row is the longest in the table, so
   Prettier will reflow every column: run `just fmt` and confirm `just fmt-md-check`
   passes, because the `lint` job runs it.
4. Update the two stale prose references in `docs/commit_workflows.md` (lines 26 and 61)
   so they stop advertising a skill that no longer exists.
5. Leave `docs/blog/posts/commit-workflows-plugins.md` alone. A dated blog post is a
   historical record, not live documentation.
6. `tests/test_agy_integration_polish.py` asserts that the `agy` provider does **not**
   materialize `sase_hg_commit`; that assertion still holds and must stay. Its docstring
   and the one in `tests/test_gemini_active_surface_guard.py` now describe a skill that
   no longer exists — reword them, but do not weaken either guard.

### Explicitly out of scope for A

`sase/memory/generated_skills.md` still documents `/sase_hg_commit` in its "Commit
Skills per Runtime" section (lines 50-63). No test gates that file, so it is not needed
for green CI, and memory files may only be edited with explicit user permission in the
conversation that asks for it. Do **not** edit it. Instead file a `memory` task bead
through `/sase_new_task` with `path=sase/memory/generated_skills.md` and a proposed
change that drops the `/sase_hg_commit` row and its two follow-up paragraphs.

## Root cause B: a Python 3.14-only argparse probe runs on 3.12 and 3.13

`tests/main/test_parser_narrowing.py::test_construction_still_rejects_badly_formed_help`
fails with `Failed: DID NOT RAISE ValueError` on `test (3.12)`, `test (3.13)` and
`coverage-contexts`, and passes on `test (3.14)`.

The test asserts that `parser.add_argument("--x", help="%(unknown)s")` raises
`ValueError("badly formed help string")`. That validation lives in
`argparse._ActionsContainer._check_help`, which CPython added in 3.14. Verified
directly:

- `cpython-3.12.13` and `cpython-3.13.13`:
  `hasattr(argparse.ArgumentParser, "_check_help")` is `False`, and
  `add_argument("--x", help="%(unknown)s")` returns normally.
- `cpython-3.14.3` (local) and `cpython-3.14.7` (the CI leg): `_check_help` exists and
  raises through `_get_validation_formatter`.

The test entered the tree in `afe374f93` (2026-08-23) alongside
`_SaseArgumentParser._get_validation_formatter`, so it has never been green on 3.12 or
3.13. The guard it provides is real — that override exists to make construction-time
help validation cheap, and it must not silently disable the check — but the guard is
only meaningful where the stdlib validates at all.

### Fix B

Skip the test where the stdlib hook is absent, keyed on the hook rather than on a
version tuple, and say why in a comment:

```python
@pytest.mark.skipif(
    not hasattr(argparse.ArgumentParser, "_check_help"),
    reason="argparse validates help strings only on Python 3.14+",
)
def test_construction_still_rejects_badly_formed_help() -> None:
    ...
```

Probing for `_check_help` rather than `sys.version_info >= (3, 14)` keeps the test
correct if the validation is ever backported or removed. Leave the body unchanged so the
3.14 leg still proves that `_SaseArgumentParser._get_validation_formatter` does not
swallow the error.

## Root cause C: 363 ACE PNG goldens are stale

`visual-test` reports `363 failed, 425 passed, 1 skipped`. This is stale goldens, not
renderer drift, and the local run on this Linux x86_64 host reproduces CI byte for byte:
`changespec_initial_120x40` reports `Changed pixels: 6556/1520532` in both places, and
the local failure set is the same 337 test functions CI reports plus one flake covered
below.

The dominant cause is a single commit. `72f93fb1f`
(`fix: stabilize visual closeout and verification gates`, 2026-08-21 12:38) added
`tabs.styles.width = "1fr"` in `src/sase/ace/tui/widgets/artifacts/view.py` after
constructing the `PanelTabStrip`. `PanelTabStrip.__init__` sets
`self.styles.width = "100%"` (`src/sase/ace/tui/widgets/panel_tab_strip.py:69`), and an
inline style beats the `#artifacts-subtabs { width: 1fr; }` rule in `styles.tcss`.
Before that commit the strip claimed the full 120 cells starting at column 8, overflowed
`#artifacts-header`, and clipped `ArtifactsSplitBadge` off the right edge entirely.
After it, the header lays out as intended — `#artifacts-split-spacer` (8) + strip
(`1fr`) + `#artifacts-split-badge` (8) — so the `{██░░}` badge paints and the tab strip
re-centers.

Measured on `changespec_initial_120x40`: the tab ink is the same width in both images
(763 px expected vs 764 px actual), and only its center moves, from cell 67.4 to cell
59.5 — exactly the 8-cell shift the restored badge column implies. The new rendering is
the intended design (the spacer exists solely to balance the badge), so the goldens are
wrong and must be regenerated.

`72f93fb1f` refreshed only 35 goldens, which is why roughly 330 stayed stale. Smaller
independent drifts piled up on top while the lane stayed red — for example
`agents_clan_tree_expanded_120x40` (723 px), `footer_leader_overflow_120x40` (7166 px)
and `agents_runner_slot_waits_120x40` (16 px). They are all covered by the same
regeneration.

### Fix C

1. Regenerate serially, to keep contention out of the frames that get accepted:

   ```bash
   SASE_PYTEST_WORKERS=1 just update-visual-snapshots
   ```

2. Re-run the lane clean and require zero failures:

   ```bash
   SASE_PYTEST_WORKERS=1 just test-visual
   ```

3. Review the regenerated corpus as test data. Spot-check that the Artifacts header now
   shows the brace badge at the right edge — that is the intended change — and skim
   `git diff --stat` for goldens whose size moved far more than their neighbours.

### Two known readiness flakes — do not paper over them

- **Local-only.**
  `tests/ace/tui/visual/test_ace_png_snapshots_axe_layout.py::test_axe_constrained_width_no_wrap_png_snapshot`
  failed on this host and passed in CI. The local frame
  (`axe_constrained_width_no_wrap_60x30`, 23145 px / 3.9% changed) was captured before
  the AXE detail pane painted: it is missing the
  `Status: running / Interval / Cycles / Errors / Chops run` block and shows a scrollbar
  instead. **Do not commit that frame.** After regenerating, open
  `tests/ace/tui/visual/snapshots/png/axe_constrained_width_no_wrap_60x30.png` and
  confirm those lines are present; if they are not, restore that golden from `git` and
  regenerate it alone with
  `SASE_PYTEST_WORKERS=1 just test-visual -- --sase-update-visual-snapshots tests/ace/tui/visual/test_ace_png_snapshots_axe_layout.py`.
- **CI-only.**
  `tests/ace/tui/visual/test_ace_png_snapshots_models_panel_providers.py::test_models_panel_provider_disabled_png_snapshot`
  failed in CI with
  `Timed out after 15.00s waiting for Launch Control provider disable title line`, while
  locally it failed only as an ordinary PNG mismatch. That is a readiness race under the
  CI lane's worker pool, not a product defect. Do not change the test to chase it; file
  a `flake` task bead through `/sase_new_task` with that node ID and this evidence.

## Verification

Run in this order from the workspace checkout:

1. `just install` — workspaces are ephemeral and this one already needed a
   `sase_core_rs` rebuild.
2. `SASE_PYTEST_WORKERS=1 just test-visual` — expect `789 passed`.
3. `just check-full` through `/sase_monitor`, never inline:

   ```bash
   sase monitor start --command 'just check-full' \
     --start-status TESTING --stop-status TESTED \
     --next 'Fix anything just check-full reported, then commit the master CI repair.'
   ```

   `check-full` is required rather than `just check`: this change touches the visual
   lane, which the diff-scoped selector excludes unconditionally.

4. Commit through `/sase_git_commit`. Consider two stitches — one for the source and
   docs fixes, one for the golden corpus — so the ~360-file binary refresh does not bury
   the code review.
5. After the push, re-run `actstat --repo sase-org/sase` and confirm CI is green;
   `ci_watch` then merges pull request 284 on its next sweep.

## Follow-ups to file, not to fix here

- A `memory` task bead for `sase/memory/generated_skills.md` (root cause A).
- A `flake` task bead for the `models_panel_provider_disabled` CI timeout (root cause
  C).
- A `memory` task bead for `sase/memory/build_and_run.md` line 64, which claims "CI
  allows a small ratio-only renderer drift tolerance". `.github/workflows/ci.yml` sets
  no `SASE_VISUAL_PNG_*` variable, the failure output reports
  `tolerance_source: default` with `allowed: 0 pixels`, and `docs/development.md` states
  PNG comparison is byte-exact "in every visual-bearing CI lane". The memory note is
  wrong and misleads anyone debugging this lane.

Every one of these must go through `/sase_new_task` so duplicates and related active
epics are checked first.
