---
tier: epic
title: Repair failing sase GitHub Actions (Master Gate, Full CI, Publish)
goal: 'Master Gate, the scheduled Full CI lanes, and the scheduled Publish workflow
  stop failing for reasons this repo owns. The persistent failures get root-cause
  fixes and the recurring flakes get race-free fixes. The only remaining release-PR
  blocker is the upstream sase-core 0.36.6 publish.

  '
phases:
- id: release-metadata
  title: Unblock Publish release-metadata sync
  depends_on: []
  size: small
  description: 'release-metadata: make tools/ratchet_core_window ignore the uv lockfile
    `revision` header that newer uv bumps on every rewrite, add guardrail tests, and
    commit a refreshed uv.lock so `uv lock --check` passes on master again.

    '
- id: gate-deterministic
  title: Fix Master Gate's persistent textual-ansi failure and the FrontmatterPanel
    teardown race
  depends_on: []
  size: small
  description: 'gate-deterministic: decouple the terminal-native syntax-palette test
    from Textual''s builtin theme catalog, which renamed textual-ansi in 8.2. Make
    FrontmatterPanel.on_mount tolerate a Mount dispatched while the panel is being
    pruned, with a regression test.

    '
- id: gate-flakes
  title: Remove recurring Master Gate test races
  depends_on: []
  size: medium
  description: 'gate-flakes: fix five recurring order and timing races. They are launch-context
    rebroadcast identity, the AcePage pump-task drain plus onboarding refresh rescheduling,
    the non-leader kill wait, the shared aggregate-runtime cache, and the panel-shell
    semicolon press racing the grammar load.

    '
- id: full-ci
  title: Fix scheduled Full CI perf-floors, visual-test, and timing flakes
  depends_on: []
  size: small
  description: 'full-ci: stop the hermetic tool-runs smoke from requiring the live-only
    DoD-17 to pass, pin the output-variables PNG snapshot to paged decks and regenerate
    its stale golden, and make the startup-clock and proc-query budget assertions
    immune to runner load.'
proposed_by: bbugyi200.athena.0ww
create_time: 2026-10-05 12:16:18
status: wip
bead_id: sase-1gt
---

- **PROMPT:** [prompts/202610/fix_sase_ci_failures.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/fix_sase_ci_failures.md)
- **BEAD:** [sase-1gt](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1gt/README.md)

# Plan: Repair failing sase GitHub Actions

## Context and diagnosis

`actstat` only shows the newest settled commit per repo, so it looked mostly green.
Direct `gh run list` history shows:

- **Master Gate** (per-SHA push gate) has not passed since 2026-10-01 23:52Z.
- Every scheduled **Full CI** run fails.
- Every scheduled **Publish** run has failed since 2026-10-02 00:47Z.
- The release-please PR ("chore(master): release 0.18.0") fails
  `release-core-floor-smoke`.

Older failures are already fixed on master and are absent from the latest four Master
Gate runs: lint "Check pinned core bindings", the mypy errors, the
`JinjaScopeRequestWire` ValueErrors, the `SASE_LAUNCH_SWARM_XPROMPTS` ImportErrors, and
the prompt-catalog deterministic failure. A survey of the last 29 Master Gate runs gives
the remaining failures below.

| Failure                                                                                                                                                   | Frequency                                       | Kind                  |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | --------------------- |
| `tests/pager/test_syntax_theme.py::test_terminal_native_colors_degrade_to_neutral_without_invalid_rich` → `KeyError: 'textual-ansi'`                      | 28/29 Master Gate runs; every Full CI test lane | deterministic         |
| `NoMatches: No nodes match '#frontmatter-raw'` (or `#frontmatter-feedback`) on `FrontmatterPanel` from `on_mount`, in random short AceApp tests           | 8/29, about one shard per recent run            | teardown race         |
| `tests/ace/tui/test_launch_context_source.py::test_every_tick_rebroadcasts_to_mounted_views`                                                              | 4/29                                            | race                  |
| `tests/ace/tui/test_runtime_tick_caches.py::test_aggregate_wires_cached_across_ticks`                                                                     | 3/29                                            | test-order dependence |
| `tests/test_ace_testing.py::test_ace_page_fast_startup_is_structurally_quiet`                                                                             | 2/29                                            | race                  |
| `tests/test_agent_terminate_processes.py::test_immediate_stage_signals_a_non_leader_and_its_tree`                                                         | 1/29                                            | race                  |
| `tests/ace/tui/command_line/test_panel_shell_pilot.py::test_empty_panel_semicolon_hops_to_palette_and_back` (TimeoutError)                                | 1/29                                            | load-sensitive wait   |
| Full CI `perf-floors`: `tests/test_sase_tool_runs_smoke.py::test_tool_runs_harness_passes_every_hermetic_case`                                            | every Full CI run since 10-02                   | deterministic         |
| Full CI `visual-test`: `test_agent_output_variables_multi_agent_png_snapshot`                                                                             | 4/14, golden also stale                         | race + stale golden   |
| Full CI 3.14: `test_gc_telemetry.py::test_exec_anchor_rejects_garbage_and_future`, `test_proc_query.py::test_whole_corpus_evaluation_stays_within_budget` | 2/7, 1                                          | timing flakes         |
| Publish `sync-release-metadata`: `[ratchet_core_window] uv.lock top-level metadata changed unexpectedly` (exit 3)                                         | every run since 10-02                           | deterministic         |

**Out of scope (upstream).** The release PR's `release-core-floor-smoke` installs the
published floor `sase-core-rs` 0.35.0, which lacks about 90 required bindings. Even
PyPI's newest release, 0.36.5, still lacks about 9. Those bindings are:
`argument_list_continuation_edit`, `check_input_value`,
`macro_argument_choice_candidates`, `macro_completion_spacer_to_parentheses_edit`,
`macro_input_type_catalog`, `macro_input_type_label`, `resolve_input_type`,
`validate_enum_choices`, and `classify_model_value`. All of them ship in the pending
sase-core release PR (sase-org/sase-core#321, v0.36.6), whose CI is still being
stabilized by other work.

Once `release-metadata` lands and 0.36.6 is on PyPI, the next scheduled Publish run
ratchets the release branch floor to `>=0.36.6,<0.37.0` and the floor smoke should pass.
No phase here edits sase-core.

Also out of scope:

- Two rarer flakes not seen in the latest four runs and not root-caused:
  `test_link_follow_planner::test_epic_context_selects_and_expands` and
  `test_vim_normal_key_containment`.
- The `HeaderTitle`-on-`UsageHeader` NoMatches that appears only inside the
  vim-containment teardown.

Phase workers record these as `PROPOSED FOLLOW-UP:` notes instead of fixing them.

General rules for every phase:

- Each phase touches a disjoint file set, and all phases can run in parallel.
- Run `just install` first if the workspace venv is stale. A stale venv shows up as an
  ImportError on `sase_core_rs` from a partially initialized module.
- Verify with `sase tool run check`. Never run `just check-full`, which nobody
  requested.
- Read the `lint_and_test` memory note before finishing.

## Phase `release-metadata`: Unblock Publish release-metadata sync

**Root cause A: the ratchet tool rejects uv's lock revision bump.**
`tools/ratchet_core_window` (`_validate_uv_lock_diff`, around lines 775-778) requires
every top-level `uv.lock` key except `package` to be identical before and after its
`uv lock` refresh.

- `.github/workflows/publish.yml` uses `astral-sh/setup-uv@v4` with no version pin, so
  it takes the latest uv.
- From uv 0.12.22, any `uv lock` that rewrites the file changes the header from
  `revision = 3` to `revision = 5`, and a ratchet always rewrites it.
- Last good run: 36906695754, with uv 0.12.21. First bad run: 36947770276, with uv
  0.12.22. CI now runs uv 0.12.23.
- uv 0.12.21 reads revision-5 locks but writes them back as revision 3, so the header
  can move either way. It is a lockfile-format detail, not dependency metadata.

**Root cause B: master's `uv.lock` is stale.** Root cause A currently hides this one.

- Commit `fe53ae4fc4` (2026-10-03) added `mkdocs-redirects>=1.2,<2` to the docs extras
  in `pyproject.toml` without relocking.
- `uv lock --check` fails on master today.
- With A fixed, the ratchet would next fail with "uv.lock package order or package set
  changed unexpectedly".

**Changes:**

1. In `_validate_uv_lock_diff`, exclude `revision` from the top-level comparison
   alongside `package`, for example with an `ignored = {"package", "revision"}` set.
   `version`, `requires-python`, `resolution-markers`, `[manifest]` and `[options]` stay
   strict. Add a short comment explaining why.
2. Add guardrail tests in `tests/test_ratchet_core_window_tool_guardrails.py`, reusing
   `tests/_ratchet_core_window_tool_helpers.py`:
   - a fake lock refresh that only changes `revision = 3` to `revision = 5` is accepted;
   - one that changes `requires-python` (or another top-level key) is still refused with
     the existing error.
3. Run `uv lock` on master and commit the refreshed `uv.lock`. Confirm the diff is only
   the `mkdocs-redirects` addition, plus a header revision change if your uv writes one.
   Then confirm `uv lock --check` passes.

**Verify:**

- Run the ratchet test modules (`tests/test_ratchet_core_window_tool_*.py`,
  `tests/test_ratchet_core_window_source_normalization.py`).
- Recommended: run an end-to-end scratch reproduction under `/tmp`, never in the
  checkout.
  1. `git worktree add` `origin/release-please--branches--master`.
  2. Apply the tool change and the lock refresh there.
  3. With uv ≥ 0.12.22 (for example `uvx uv@0.12.23`) and
     `UV_DEFAULT_INDEX=https://pypi.org/simple/`, run
     `python tools/ratchet_core_window --allow-transitive-lock-refresh`.
  4. Expect exit 2 ("applied") followed by a clean `uv lock --check`. Remove the
     worktree afterwards.
- Finish with `sase tool run check`.

**Non-goals:**

- Do not pin uv in the workflows.
- Do not add a `uv lock --check` CI gate. The release-please version bump would trip it
  before sync-release-metadata runs.
- Record both as `PROPOSED FOLLOW-UP:` ideas.

## Phase `gate-deterministic`: Fix Master Gate's persistent textual-ansi failure and the FrontmatterPanel teardown race

### textual-ansi KeyError

**Root cause:** the fast and full lanes install with
`uv pip install --no-sources … -e ".[dev]"`, which ignores `uv.lock`.

- That resolves the floating `textual[syntax]>=0.45.0` to **textual 8.2.8**. The local
  lock and the `visual` extra pin 8.0.1.
- Textual 8.2 replaced the `textual-ansi` builtin with `ansi-dark` and `ansi-light`,
  which have the same `ansi_*` color values.
- `tests/pager/test_syntax_theme.py:84` indexes `BUILTIN_THEMES["textual-ansi"]` and has
  failed in CI since it was added on 2026-10-02.
- The product code is fine. Under 8.2.8, `syntax_palette_from_theme` on `ansi-dark` and
  `ansi-light` already satisfies every invariant the test checks (backgrounds `#000000`
  and `#FFFFFF`, no terminal colors, contrast ≥ `MIN_SYNTAX_CONTRAST`).

**Changes:**

1. Make the test independent of Textual's theme catalog. Build an explicit
   terminal-native `textual.theme.Theme` in the test:
   - `ansi_default` foreground, background, surface, panel and boost;
   - `ansi_*` primary, secondary, accent, warning, error and success;
   - one variant with `dark=False` and one with `dark=True`;
   - no `ansi=` keyword, which 8.0.1 does not accept.

   Assert the existing invariants for both variants. Also parametrize over every
   installed `BUILTIN_THEMES` entry whose background `is_terminal_color(...)`, so
   `textual-ansi` on 8.0.1 and `ansi-dark`/`ansi-light` on 8.2.x stay covered. Assert
   that the combined case list is non-empty, so the test can never pass vacuously.

2. Refresh the two code comments that name `textual-ansi` as the example terminal-native
   theme: `src/sase/pager/syntax_theme.py` near line 468 and
   `src/sase/pager/history/styles.py` near line 227. Mention the 8.2 names. Comments
   only; no behavior change.

### FrontmatterPanel `#frontmatter-raw` / `#frontmatter-feedback` NoMatches

**Root cause:** Textual's `Widget.mount()` returns an empty `AwaitMount` when the parent
is `_closing` or `_pruning`.

- This happens when the app shuts down at the end of `run_test`, or when
  `AcePageGroup`'s reset calls `app.recompose()` while `PromptInputBar` (see
  `src/sase/ace/tui/widgets/_prompt_input_bar_lifecycle.py:67`) is still composing.
- In that case `FrontmatterPanel` never mounts its composed children, but Textual still
  dispatches `Mount`.
- `FrontmatterPanel.on_mount` (`src/sase/ace/tui/widgets/frontmatter_panel.py:204-208`)
  then calls `query_one("#frontmatter-raw")` and `query_one("#frontmatter-feedback")`,
  and `_refresh()` queries `#frontmatter-rows`.
- The resulting NoMatches surfaces as an unhandled TUI exception that fails whichever
  short test was running.

**Changes:**

1. In `on_mount`, resolve the children defensively. Catch `textual.css.query.NoMatches`
   (the file already imports `NoScreen` from `textual.dom`, so follow that style) and
   return early without calling `_refresh()`, leaving `_raw_editor` and
   `_feedback_widget` as `None`.
2. Add a one-line comment saying why: Textual skips child mounts for a pruning parent
   but still dispatches Mount.
3. Do not add sleeps or wider waits. Every other method that touches these children
   already handles `None` or runs only on a live panel. Confirm this for `on_resize` and
   the host APIs called after mount.
4. Add a regression test next to `tests/ace/tui/widgets/test_frontmatter_panel.py` that
   drives `on_mount` on a panel whose composed children are absent and asserts it does
   not raise. For example, construct a `FrontmatterPanel` and mount it inside a minimal
   Textual `App.run_test()` while patching `compose` to yield nothing, or call
   `on_mount` with the children removed.

**Verify:** run `tests/pager/test_syntax_theme.py` and the frontmatter panel tests, then
`sase tool run check`.

Optional cross-check of the theme test under 8.2.8: use a throwaway venv in `/tmp`, for
example `uv venv /tmp/tx && uv pip install --python /tmp/tx/bin/python textual==8.2.8 …`
with `PYTHONPATH=src`. Do not mutate the workspace venv.

## Phase `gate-flakes`: Remove recurring Master Gate test races

Fix each race by waiting on the condition that guarantees the asserted state. Never add
fixed sleeps. Prefer product fixes where the race is real.

1. **`test_every_tick_rebroadcasts_to_mounted_views`**
   (`tests/ace/tui/test_launch_context_source.py` around lines 653-669).
   - The test asserts `all(state is source.state for state in calls)` after
     `source.refresh(); await page.pause()`.
   - `LaunchContextSource` resolve workers (`launch_context_source.py`, `_tick` around
     lines 260-297) can still be running after AcePage's startup wait. The process-wide
     0.5s launch-default peek cache and config-token revalidator are shared across
     tests, so the tick can see a changed token and start a new resolve. Its
     `_apply_*_result` then broadcasts a new state object.
   - The product behavior is correct. Make the test prove only what it claims: a tick
     rebroadcasts the current state even when nothing changed.
   - Capture `expected = source.state` before `refresh()`, confirm `_tick` broadcasts
     synchronously (read the code), and assert `calls` is non-empty and
     `calls[0] is expected`.
   - If the broadcast is not synchronous, instead wait until no resolve is in flight
     before the refresh, using a real readiness condition.
2. **`test_ace_page_fast_startup_is_structurally_quiet`** (`tests/test_ace_testing.py`).
   - `_drain_pump_free_tasks` in `src/sase/ace/testing/ace_page.py` (around lines
     142-155) gathers the registered tasks. On Python 3.12, `asyncio.gather` over tasks
     that are already done returns without yielding, so the registry-discard done
     callback (`src/sase/ace/tui/util/pump_tasks.py` around line 98) is still queued
     when the test asserts. That leaves a
     `<Task cancelled name='sase-agents-onboarding-plugins'>` behind.
   - Also, `_run_agents_onboarding_plugins_refresh`
     (`src/sase/ace/tui/actions/agents/_display_detail_onboarding.py` around lines
     157-176) reschedules itself from its `finally` block when a refresh is pending,
     even after cancellation. The launch-targets refresh has the same pattern; check it
     too.
   - Fix the drain: after each gather, discard done tasks from every registry. Loop,
     with a bounded number of passes, cancelling any newly registered tasks until all
     registries are empty.
   - Fix the refreshers: do not reschedule when the task was cancelled. For example,
     catch `asyncio.CancelledError`, clear the pending flag, and re-raise. Use the same
     idiom for every refresher that has this pattern.
   - Add or extend a unit test for the drain that reproduces the already-done-task case.
3. **`test_immediate_stage_signals_a_non_leader_and_its_tree`**
   (`tests/test_agent_terminate_processes.py` around lines 272-283).
   - With `wait=False`, `_signal_non_leader` signals the runner and then its descendants
     asynchronously. The test waits only for the runner pid, then immediately asserts
     the same-group child is gone.
   - Fix: `wait_until_gone(proc.pid, pids["same_group"])`, using
     `tests/_agent_process_tree_helpers.py:141`, before the assertions.
4. **`test_aggregate_wires_cached_across_ticks`**
   (`tests/ace/tui/test_runtime_tick_caches.py` around line 81).
   - The test clears only `_aggregate_wires_cache`. `_aggregate_result_cache` in
     `_agent_time_aggregate.py` (around lines 167-189) is keyed on content, and earlier
     tests in the same xdist worker can fill it with identical agents.
     `_aggregate_runtime` then returns early (around lines 286-296) and never fills the
     wires cache.
   - Fix: clear both caches. Prefer an autouse fixture in that module, or reuse an
     existing cache-reset helper if one exists.
5. **`test_empty_panel_semicolon_hops_to_palette_and_back`**
   (`tests/ace/tui/command_line/test_panel_shell_pilot.py`).
   - The 1.0s budget expires inside Textual's `wait_for_idle(0)`. Opening the panel
     starts the command-line grammar load (a spec subprocess plus JSON parses, about
     0.8s on CI) plus history and context threads, so a correct `press` can exceed 1s on
     a loaded runner.
   - Fix: before pressing, wait for `is_command_line_grammar_pending(app)` to become
     false (grep for it). Raise the per-step budget to comfortably exceed Textual's idle
     caps, for example 5s.
   - Apply the same fix to the sibling tests in that module that use the same 1.0s
     pattern.

**Verify:**

- Run each touched test module several times, for example with `-p no:randomly` and with
  `--count`-style repetition via a shell loop. Where cheap, also run under `-n auto`, so
  order effects show up.
- Then run `sase tool run check`.
- Record the rarer unresolved flakes listed in Context as `PROPOSED FOLLOW-UP:` notes.

## Phase `full-ci`: Fix scheduled Full CI perf-floors, visual-test, and timing flakes

1. **perf-floors, deterministic:**
   `tests/test_sase_tool_runs_smoke.py::test_tool_runs_harness_passes_every_hermetic_case`.
   - Every hermetic case passes; the failure is the DoD assertion at line 154. It
     requires DoD-17 to be `"pass"`.
   - The live case `dod-17-live-escalate-join` was added in f3899b4171, is listed in
     `LIVE_CASES`, and returns `not_run(..., dod=["DoD-17"])` in hermetic mode
     (`tools/_smoke_tool_runs_cases_detach.py` around lines 295-299). `dod_summary` in
     `tools/smoke_sase_tool_runs` lets `not-run` beat `pass`, so DoD-17 is always
     `not-run` in hermetic runs.
   - Fix it like the existing DoD-8 precedent: drop `17` from the all-pass tuple and
     assert `dod["DoD-17"] == "not-run"`. The per-case hermetic assertion above it still
     requires every hermetic `dod-17-*` case to pass.
   - Verify by running that single slow test directly, for example with
     `pytest -m slow tests/test_sase_tool_runs_smoke.py::test_tool_runs_harness_passes_every_hermetic_case`
     or the `just test-slow` recipe with that node id. It takes several minutes, so use
     `/sase_monitor` if it would exceed your synchronous limit.
2. **visual-test:**
   `tests/ace/tui/visual/test_ace_png_snapshots_agents.py::test_agent_output_variables_multi_agent_png_snapshot`.
   - This multi-card fixture sits near the deck `spread_max_screens` threshold
     (`src/sase/ace/tui/widgets/decks/panel_spread.py`). It sometimes settles as spread,
     scrolls the Context card's OUTPUT VARIABLES off-screen, and fails the
     `assert_page_svg_contains` checks.
   - `pin_decks_paged()` in `tests/ace/tui/visual/_ace_agents_png_snapshot_helpers.py`
     exists for exactly this, and the sibling session-panel snapshot tests call it.
   - Separately, the golden
     `tests/ace/tui/visual/snapshots/png/agents_output_variables_multi_agent_120x40.png`
     is stale. It predates the 09-27 deck chrome (`C·A` view badge, `final 0` strip
     entry, `P view` footer); passing runs reported a 0.678% pixel diff.
   - Fix: import `pin_decks_paged` and call it before `patch_startup_loaders(...)`. Then
     regenerate only this golden with a targeted `just fix-tui-screenshots -- <node id>`
     (use `/sase_monitor` if needed).
   - Inspect the run's report under `.pytest_cache/sase-visual/` and the golden diff
     before finishing. Expect paged Context with the newer chrome and the four
     output-variable strings visible. Confirm the run status is not `partial` for this
     golden.
3. **3.14 flake:**
   `tests/ace/tui/util/test_gc_telemetry.py::test_exec_anchor_rejects_garbage_and_future`
   (around line 435).
   - The fallback in `src/sase/ace/tui/util/startup_clock.py` (around lines 51-69)
     computes `now - (boottime_now - start_boot_s)`, reading `now` before the env parse,
     the `sysconf` call and the `/proc/self/stat` read. Two calls therefore differ by
     I/O time.
   - The default relative `pytest.approx` is only about 60-80µs at CI uptimes; observed
     gaps were 66-79µs.
   - Fix the test: compare with an absolute tolerance, for example
     `pytest.approx(fallback, abs=0.05)`, because `/proc` start time has 10ms
     resolution.
   - Also read `now` immediately next to `boottime_now` in `startup_clock.py` so the
     fallback is as tight as possible.
4. **3.14 flake:**
   `tests/ace/tui/test_proc_query.py::test_whole_corpus_evaluation_stays_within_budget`
   (around lines 415-433).
   - It makes a single cold `perf_counter` measurement against a 1.0s budget. On a
     loaded runner it measured 2.41s.
   - Fix: take the best of three runs, each with a fresh `ProcQueryFilter()`, using
     `time.process_time()`, and keep the 1.0s budget.

**Verify:** run the touched non-visual test modules, the targeted slow test and the
targeted visual capture, then `sase tool run check`.

## Done criteria

- Each phase's targeted tests pass locally and `sase tool run check` is green.
- After landing, the next Master Gate run on master has no failures from the families
  above.
- Full CI's perf-floors and visual-test no longer fail on the diagnosed tests.
- Publish's `sync-release-metadata` either applies the ratchet (exit 2) or reports
  already in sync. Its floor smoke may stay red only until sase-core 0.36.6 is
  published.
