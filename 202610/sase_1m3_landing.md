---
tier: tale
title: Finish the sase stats project landing
size: medium
goal:
  Keep the Statistics → Projects report painted across refresh ticks, make the `sase
  stats project` help tests pass on every supported Python, capture the live Projects
  screenshot the acceptance phase could not, and close epic sase-1m3.
bead: sase-1m3
proposed_by: bbugyi200.apollo.sase-1m3.land
proposed_with: claude/opus
create_time: 2026-10-19 16:30:14
status: wip
---

- **PARENT:**
  [202610/sase_stats_project.md](https://github.com/sase-org/sase--plans/blob/main/202610/sase_stats_project.md)
- **BEAD:**
  [sase-1m3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1m3/README.md)

# Finish the `sase stats project` landing (epic `sase-1m3`)

This tale is the remaining epic-caused work found by the `sase-1m3` land audit.
Implement it directly, then perform the complete closeout in §5 **in the same turn**.
This tale has no land agent of its own and nothing resumes the landing afterwards. Do
not wait for this work's commit SHA, push, or CI: host finalization commits after your
turn ends.

## Context already settled by the land agent

Read `sase-1m3` and its notes with `sase bead read sase-1m3 -r "<why>"` (the last note
is the land agent's follow-up triage), and read sections 6.5, 19, and 20 of
`plan:202610/sase_stats_project.md` with `sase artifact read`. Read reference memory
`tui.md`, `tui_perf.md`, `tui_screenshot.md`, `lint_and_test.md`, and `cli_rules.md`
with `/sase_memory_read` before changing code.

All twelve phases are closed. The land audit read their notes, the epic's commits
(`7c6b32457b`, `7545e85abf`, `c68a30365f`, `2cae86a1aa`, `5fe54b5974`, `dd963c0fa6`,
`9c4c262559`, `a53f10f6a6`, `0f2052be6a`, `4e5b7d2eb6`) and the sources, and found:

- **Integration is clean.** All 57 recent agent commits with an agent footer carry
  `SASE_MODEL` (only six early ones from stale installs lack it). 32 of 35 plans added
  since provenance landed carry `proposed_with`. No commit since the epic started
  duplicates or conflicts with the stats code. `sase bead epic-symbols sase-1m3` is
  empty. `tests/main/test_parser_stats.py` and `tests/stats` pass on Python 3.12 (175
  passed).
- **Every `PROPOSED FOLLOW-UP` is already triaged**, and the outcomes are recorded in
  the last `sase-1m3` note. New tasks: `sase-1n1` (warm `project_stats` latency: a 5 ms
  git child-exit poll plus about 60 git spawns per project), `sase-1n2` (Code
  cold/changed-tip budgets), `sase-1n3` (the deferred decision record), `sase-1n4` (a
  guide candidate), and `sase-1n5` (the other seven `sase stats` views). Corroborations:
  a +1 on `sase-1ii`, and notes on `sase-1hy` and `sase-1l2`. The rest are declined or
  point to existing beads. Do not file them again, and do not try to fix the performance
  tasks here.
- **Remaining epic-caused defects**, fixed below:
  1. Epic note #4 (the user's report of a "spinning circle every few seconds") is
     **confirmed**. Commit `0f2052be6a` changed `StatisticsPane._start_load` to call
     `_paint_loading()` whenever `_last_result` no longer passes
     `_result_matches_selection`. For a preset range, `_start_load` re-resolves
     `self._range` with `end_ts = ceil(now)`, so on **every** 30-second refresh tick the
     last result stops matching, and the whole body is replaced by the "Loading
     statistics" panel for the ~2.5 s warm query. (Before `0f2052be6a`, a refresh
     painted loading only when `_last_result is None`.) The core fingerprint also hashes
     the request, including the range, so a preset tick never takes the `same_primary`
     path. `_on_primary_success` then replaces the report with a Code-less primary,
     which resets `_code_complete`, and the card shows `measuring code…` again until the
     ~1.6 s Code worker lands. A pilot probe (open Statistics, select Projects, settle,
     sleep 1.2 s, call `_start_load()`) recorded `_paint_loading` on the tick. The same
     moving range makes `_paint_error` drop the last good report on a failed refresh
     instead of showing `stale · <time>`, which breaks §6.5.
  2. Epic note #5: the 3.13 and 3.14 legs of the CI matrix fail
     `tests/main/test_parser_stats.py::test_project_help_golden` and
     `::test_project_help_includes_definitions_and_examples`. Both hard-code Python
     3.12's argparse option layout
     (`-f {json,markdown,rich}, --format {json,markdown,rich}`), while 3.13+ renders
     `-f, --format {json,markdown,rich}`. sase supports `requires-python >= 3.12`, and
     `.github/workflows/ci.yml` tests 3.12, 3.13, and 3.14.
  3. Acceptance item §20 "Capture a live `sase screenshot` of Statistics → Projects and
     inspect the PNG" never succeeded (`sase-1m3.12` note #4).

## 1. Keep the painted report across revalidating refreshes

Work in `src/sase/ace/tui/modals/statistics_pane.py`, plus
`statistics_pane_rendering.py` and `statistics_pane_project_merge.py` where needed. Keep
every timer, worker-state, and pump callback thin (`tui_perf.md` rules 1, 2, 4, and 5).
Add no new refresh path.

- **Selection identity.** Add a helper that returns the user's selection without the
  resolved bounds: view, `_preset_key`, `_custom_range_value` (used only when there is
  no preset), project filter, macro focus, and the perf grouping when the view is
  `perf`. Record the identity each load was started with. Record the identity of the
  report that is currently painted when a result is stored in `_last_result`.
- **`_start_load`.** Paint loading only when nothing is painted for the current
  identity: a first load, or a user change of view, range, project, macro, or perf
  grouping. A revalidation for the same identity keeps the painted report and shows only
  the existing `refreshing…` status.
- **`_paint_error`.** Keep the last good report with `stale · <time>` when the painted
  identity still matches. This applies even when the resolved preset bounds have moved.
- **Code carry-over on Projects revalidation.** When a primary result with a new
  fingerprint arrives for the painted identity and the painted report already includes
  Code:
  - Keep the fresh un-merged primary as its own record.
  - When `merge_project_code_stats(new_primary, painted_report)` succeeds, paint that
    merged result immediately. It has fresh Output/Runs and the previous Code values,
    and no `measuring code…`. Then start the Code worker.
  - When it returns `None`, because a repo tip moved, keep the painted report until the
    fresh Code result lands, and then paint the fresh primary merged with that Code
    once.
  - Always merge the fresh Code result onto the stored un-merged primary, never onto the
    carried view, so stale Code issues and coverage cannot persist.
  - If the Code worker fails, paint the stored primary with the existing Code error
    status.
  - `_code_complete` must be `True` again after the fresh merge, so the
    `_wait_statistics_settled` test helper still settles.
- **Unchanged behavior.** First paint still never waits on Code (§6.5). User-initiated
  selection changes still show loading and then progressive paint. Results for a stale
  generation or selection are still discarded. An equal fingerprint still neither
  rebuilds nor repaints.
- **Tests** (extend `tests/ace/tui/test_statistics_pane_loading.py` and/or
  `tests/ace/tui/test_statistics_pane_projects.py`; reuse `_patch_center`,
  `_open_statistics`, and `_wait_statistics_settled`). Drive the clock by monkeypatching
  the pane module's `resolve_preset`, or the loader-facing clock seam, so a tick
  re-resolves different bounds without sleeping. Make the fake loader's fingerprint vary
  with the range so the production path is exercised. Assert:
  - a Projects revalidation tick never calls `_paint_loading` and never paints a report
    without Code once Code was complete; after settle, the painted report carries the
    new range and fresh Code;
  - the same no-loading guarantee holds for one non-Projects view (for example
    `overview`);
  - a user range change (`t`) still paints loading;
  - a failed refresh after the bounds moved keeps the last report and shows
    `stale · <time>`;
  - when the refs conflict, the previous report stays painted until fresh Code lands,
    and then exactly one merged paint follows.

  Keep the existing loading, projects, filter, and soak tests green.

- Rendered output for loaded states does not change, so no PNG golden should change. If
  you do touch rendering, run the targeted Statistics visual check through
  `/sase_monitor` as `tui_screenshot.md` describes.

## 2. Make the `sase stats project` help portable across Python versions

- Hoist the identical `_DoctorHelpFormatter` from `src/sase/main/parser_doctor.py` into
  one shared helper module (for example `src/sase/main/_help_format.py`, with a public
  class such as `StableInvocationHelpFormatter(argparse.RawDescriptionHelpFormatter)`).
  It renders every option as `-s, --long METAVAR`, the Python 3.13+ layout, on every
  interpreter. Use it for the doctor parser and for both parsers in
  `src/sase/main/parser_stats.py` (`stats` and `stats project`). Satisfy Symvision
  (`symvision.md`) without new exemptions.
- Regenerate `tests/stats/golden/cli/project_help.txt` with
  `SASE_UPDATE_PROJECT_RENDER_GOLDENS=1` (the `_UPDATE` switch in
  `tests/main/test_parser_stats.py`). Update
  `test_project_help_includes_definitions_and_examples` to the new spellings
  (`-f, --format {json,markdown,rich}`, `-g, --group-by {model,patch,project,repo}`,
  `-p, --project PROJECT`, `-r, --range RANGE`).
- Pin the help-colour environment in both help tests so Python 3.14's coloured help can
  never engage: set `NO_COLOR=1` and remove `FORCE_COLOR` and `PYTHON_COLORS` with
  `monkeypatch`.
- Show the formatter is interpreter-independent. Load the new module by file path in a
  standalone script that builds a parser with the same option shapes, and run it under
  this checkout's Python 3.12 and under `uv run --no-project --python 3.14 python -I
  <script>` (and 3.13 if available). The printed help must be byte-identical. If `uv`
  cannot provide the interpreter offline, record that and rely on the formatter's
  deterministic `_format_action_invocation`.
- The generated CLI reference (`tools/render_cli`) is already interpreter-independent.
  Run `just docs-gen-check` (part of `just check`) and regenerate with `just docs-gen`
  only if it reports drift. Follow `cli_rules.md` for help text.
- Add a short note to `sase-14p`: the new shared formatter is a ready fix pattern for
  its `test_notify_rules` help assertion. Do not fix `sase-14p` here.

## 3. Verify

- Run `just fix`, then the focused suites: the Statistics pane tests
  (`tests/ace/tui/test_statistics_*.py`), `tests/stats`,
  `tests/main/test_parser_stats.py`, and the doctor parser tests.
- Run `sase tool run check` in this sase checkout (`just install-venv` first if the venv
  is stale). The land audit reproduced these failures on clean HEAD `4e5b7d2eb6`, and
  they are not this tale's:
  - `test_changed_reply_invalidates_hint_document_and_mappings` (`sase-1l3`/`sase-1hy`);
  - `test_hint_mode_keeps_collapsed_header_unchanged` (note on `sase-1hy`);
  - `test_bottom_pinned_body_stays_pinned_across_row_count_change` (`sase-1ki`);
  - `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`
    (the manifest lacks `test_github_actions_install_smoke.py` and
    `test_github_actions_testpypi_drill.py` from epic `sase-1my`, whose landing owns the
    refresh).

  Treat any other failure as yours until triage shows KNOWN/FLAKY or a clean-HEAD
  reproduction. Do not run `just check-full`.

## 4. Live screenshot of Statistics → Projects

Follow `tui_screenshot.md`. Run the capture with this checkout's
`.venv/bin/sase screenshot` so the TUI runs the fixed code. Use `--keep` and drive the
printed `sase_tmux_target` with `tmux send-keys`:

1. `#` opens the Admin Center.
2. `6` selects the Statistics tab (tab order: config, logs, machines, procs, projects,
   statistics, ...).
3. `]` moves from Overview to Projects. Confirm the order in the Statistics help (`?`)
   if it differs.

Wait with `-w` on the Projects heading text, not on a fixed sleep. Capture once after
the report and Code have painted. Capture again after at least one 30-second refresh
tick (or after pressing `r`, which takes the same `_start_load` path). Inspect both
PNGs: the second must still show the report, with at most `refreshing…`/`updated` in the
status, and never the "Loading statistics" panel or `measuring code…`. Record on
`sase-1m3` the PNG paths, what you saw, and the observed time from selecting Projects to
the first painted report. That time is the §20 first-paint figure; its slowness belongs
to `sase-1n1`. If the live TUI genuinely cannot reach the tab, record the exact blocker
on `sase-1m3` and keep going. The pilot tests in §1 are the regression proof; do not let
this step block the closeout.

## 5. Close epic `sase-1m3` (final step, same turn)

1. Run `sase bead epic-symbols sase-1m3`. For every entry, resolve the symbol: wire it
   up, privatize it, add a justified non-test pragma, or delete it, following the
   Symvision epic-whitelist policy (`symvision.md`). Re-key a Justfile line only to a
   specifically identified still-open later bead that needs it. The audit found no
   entries; check again on the final tree.
2. Close the epic normally with `sase bead close sase-1m3 --note "<verification>"`. In
   the note, record:
   - the land audit's verification above: all twelve phases and both nested epics
     reviewed, the integration findings, and the follow-up triage (cite the triage note
     and tasks `sase-1n1` through `sase-1n5`);
   - this tale's fixes: the refresh flicker root cause and fix with its tests, the
     portable help formatter, the screenshot outcome, and the `sase tool run check`
     result and its triage;
   - the symbol cleanup result.

   If the close is refused for leftover `--epic-symbol` entries, finish that cleanup and
   close again. If it is refused for unfinished descendants, finish them or record the
   outcome deliberately. Never use `--force` merely to make the command succeed.

3. From this sase checkout run `just symvision` and confirm the whitelist is clean.
4. Open the plans repository through `/sase_repo` (`sase repo open plans -r "<why>"`)
   and set `status: done` in the frontmatter of `202610/sase_stats_project.md`, the PLAN
   path printed by `sase bead read sase-1m3`.
5. `sase-1m3` has no `parent_bead`, so there is no ancestor to close.

Submit `/sase_final` last, covering every repository you changed: this sase checkout and
the plans repository.
