---
tier: epic
title: Green Master Gate and Full CI with host-independent TUI goldens
goal: 'Master Gate stays green on master, and a fresh Full CI on the landed tip passes
  every job, including visual-test on GitHub''s UTC runners, without raised timeouts,
  skips, or goldens copied from CI.

  '
decisions:
  tui_screenshot_rule:
    ask: Add a host-independent golden rule (clock, zone, hostname, HOME, loaded state)
      to tui_screenshot.md?
    memory:
    - tui_screenshot.md
    default: false
    answer: true
phases:
- id: picker-race
  title: Save-location picker mount race and host-isolated mini-macro location goldens
  depends_on: []
  size: small
  description: 'picker-race: render choices delivered before mount in SaveLocationPickerModal,
    add a regression test, isolate the mini-macro location PNG tests from the host
    config, and wait for loaded state before capture.'
- id: clock-seams
  title: Pin every clock, zone, and hostname the visual fixtures observe
  depends_on: []
  size: medium
  description: 'clock-seams: make the FINAL and epic-follow fixture epochs zone-explicit,
    route every pager age through the pinned time-band epoch, pin the memory pane
    clock, pin the retry harness hostname, and refresh only the goldens those fixes
    move.'
- id: deck-races
  title: Agents-deck capture races and the sase-1ii deck-state timeouts
  depends_on: []
  size: medium
  description: 'deck-races: fix the Tool Runs deck fast-path detail race and its fixture,
    wait on semantic state for the gate footer and session runtime captures, and root-cause
    the spread_deck_sticky and left_right_search_committed timeouts tracked by sase-1ii.'
- id: rebaseline
  title: Full visual-lane rebaseline for the intended Models and dictionary-card changes
  depends_on:
  - picker-race
  - clock-seams
  - deck-races
  size: medium
  description: 'rebaseline: on the integrated tip, regenerate the remaining legitimately
    stale goldens (Models summary line, dictionary card), inspect every change, and
    prove a clean full check under TZ=UTC.'
- id: ci-proof
  title: Prove Master Gate and a fresh Full CI green on one SHA
  depends_on:
  - rebaseline
  size: small
  description: 'ci-proof: wait for Master Gate on the landed tip, dispatch and monitor
    Full CI, triage any red against existing flake owners, and record same-SHA run
    evidence.'
proposed_by: bbugyi200.athena.0zp
decided_by: reviewer
decided_via: tui
create_time: 2026-10-11 13:15:11
status: wip
bead_id: sase-1jz
---

- **PROMPT:** [prompts/202610/ci_green_visual_lane.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/ci_green_visual_lane.md)
- **BEAD:** [sase-1jz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1jz/README.md)

# Green Master Gate and Full CI with host-independent TUI goldens

CI has two red signals with different causes. The per-push **Master Gate** is now only
intermittently red, on one product race. The scheduled **Full CI** has not passed since
at least 2026-10-09, and its remaining deterministic blocker is the `visual-test` job.
That job fails because many committed PNG goldens only reproduce on the host that
generated them. Each test must be made host-independent before its golden is
regenerated. Otherwise the next regeneration on a different host flips the goldens again
(commit `4bc1db294c`, authored on a UTC host, and `c04b04d9e6`, on an EDT host, already
flip-flopped the FINAL and retry goldens).

This is an epic because the fixes touch independent subsystems (a modal, the pager, the
FINAL/retry fixtures, the Agents deck), and each of those phases can refresh its own
disjoint goldens in parallel. The full-lane rebaseline needs every fix integrated first,
and the CI proof needs landed commits plus a multi-hour external wait.

## Evidence (observed 2026-10-11, master `725f7ea04e` and newer)

**Master Gate**

- Run 38155623673 on `725f7ea04e` passed every job. That commit moved
  `sase-core-revision.txt` to `21ef6d13bb8f`, which contains sase-core `96f7796d` (plan
  `project` frontmatter). That cleared the two `tests/test_plan_validate.py` reds that
  failed every gate since `e926d4100b`.
- `c10f074f59` had already cleared the deterministic completion-snapshot,
  marker-path-audit, incident-replay and flag-candidate reds. Every failure that
  repeated across the last 30 failed gates passes on `725f7ea04e`. That includes the
  contract-manifest and usage-limit reds, which earlier commits fixed.
- Run 38156804000 on `7ad439fd42` then failed only on
  `tests/ace/tui/actions/test_prompt_snippet_location_flow.py::test_shift_tab_round_trip_preserves_trigger`
  (`assert 'has ⇥ todo' in 'Finding destinations…'`). The same race explains
  `tests/ace/tui/actions/test_prompt_mini_macro_location_flow_existing.py::test_existing_typeahead_seeds_finder_query`
  and the Full CI snippet-finder timeout below.
- Other single hits in those 30 runs:
  - `test_link_follow_seam.py::test_beads_sync_reports_visible_and_missing` (twice)
  - the `test_sudo_acceptance_credentials.py` canary tests (owner `sase-1gh`)
  - `test_prompt_next_word_inline_tail.py` (owner `sase-1ib`)
  - `test_prompt_next_word_midword.py`
  - `test_agent_terminate_processes.py`
  - `fakey/test_provider_drain_e2e.py`
  - `test_prompt_bar_hot_spare.py`

  They are out of scope here (see ci-proof).

**Full CI `visual-test`**

- Scheduled runs 38141210293 (`cca1773157`) and 38116223239 ended with status `failed`,
  not `drift`, so no comparison ran.
  - In both,
    `test_ace_png_snapshots_snippet_name.py::test_snippet_location_flow_finder_png_snapshot`
    timed out waiting for `2 snippets`, with the identical last-frame digest
    `34428cb8c1af`.
  - In 38116223239, `test_agents_deck_blocks_spread_deck_sticky_png_snapshot` also timed
    out.
- PR #319's run 38143553606 (`e926d4100b` plus the current pin) completed the
  comparison: `created=3 updated=70 unchanged=965`.
  - Its `visual-failure-report.html` artifact embeds expected and candidate PNGs for all
    73 records.
  - The diffs below were classified by cropping the differing regions.
- A targeted local `just test-visual` on `725f7ea04e` (EDT host, sase-core-rs 0.38.0)
  over the drifting files:
  - `spread_deck_sticky` timed out again.
  - With it deselected: `created=3 updated=36 unchanged=49`.
- Comparing CI and local separates **host-dependent** drift (CI only) from **universal**
  drift (both).

| Class                           | Goldens                                                                                                                                                                            | CI                                                                                                 | Local                                                        | Cause                                                                                                                                                                                                                                   |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Zone at import                  | `agents_final_*` (11), `agents_waiting_epic_follow_*` (3)                                                                                                                          | drift (`07:30`→`03:30`, `14:32`→`10:32`)                                                           | clean                                                        | Naive `datetime(...).timestamp()` built at import in the host zone (`_ace_agents_final_png_snapshot_shared.py` `FIXED_NOW_TS`/`T0_TS`, `test_ace_png_snapshots_agents_waiting.py` `_FOLLOW_SINCE`), rendered in the pinned Eastern zone |
| Hostname                        | `agents_retry_e2e_{countdown,completed_chain,running_fallback}`                                                                                                                    | drift (artifact suffix `3b253ae7aa66`→`f9f3bc999f01`)                                              | clean                                                        | `continuation_capture` `owner()` embeds `socket.gethostname()` in the node record whose sha256 names the displayed artifact file                                                                                                        |
| Wall clock                      | 18 `timeband_*`, `memory_pane_changes_lens_review_{dark,light}`                                                                                                                    | drift (`16d`→`18d`)                                                                                | drift                                                        | `src/sase/pager/_screen_chrome.py` `_history_chrome_state` ages with `time.time()` and ignores `_time_band_now_epoch`; `src/sase/ace/tui/modals/memory_pane_changes_card.py` reads `time.time()`                                        |
| Host config and loading capture | `mini_macro_location_flow_{picker,finder}`                                                                                                                                         | drift                                                                                              | clean                                                        | Picker golden captured `Finding destinations…`. Both tests read the real user config (golden lists `/home/bryan/.config/sase/sase.yml:actstat`, 107 definitions; CI `/home/runner/...`, 79)                                             |
| Capture race                    | `tool_runs_card_session`, `tool_runs_split`, `agents_session_panel_turns_gate_90x40`, `agents_running_session_runtime_collapsed`                                                   | CI: `run detail unavailable · summary retained`, missing `(/) blocks`/`cards`/`compact Runs` hints | local: only the Models line; runtime shows `main 0`→`main 1` | Tool Runs deck fast path paints without cached block details and strands the worker; footer and turn count captured before settle                                                                                                       |
| Timeout                         | `agents_deck_blocks_spread_deck_sticky`, `agents_decks_left_right_search_committed`                                                                                                | timeout or drift                                                                                   | timeout                                                      | `sase-1ii` (ready, unowned)                                                                                                                                                                                                             |
| Intended, never rebaselined     | ~35 Agents-tab session and detail goldens (`agents_deck_blocks_*`, `agents_deck_view_*`, sessions, jump panel, fleet, monitor lanes, `agents_retry_e2e_plan_session_countdown`, …) | drift                                                                                              | drift                                                        | New `Models: <model>` summary line from `d9ee437e7e` (feat: show session provider and model summary)                                                                                                                                    |
| Intended, never committed       | `word_definition_modal_120x40` updated; `word_definition_modal_long_70x24`, `_wordnet_120x40`, `_wordnet_light_120x40` created                                                     | drift                                                                                              | drift                                                        | Dictionary card redesign from `f1f37fcdfa` shipped without goldens                                                                                                                                                                      |

Not in scope:

- **Release PR CI.** The release-please PR #299 `release-core-floor-smoke` fails because
  its published window `sase-core-rs>=0.35.0,<0.36.0` lacks current bindings.
  sase-core-rs 0.38.0 is on PyPI. Moving that window belongs to the release chain
  (`sase-1i5.9.1.3`, `tools/ratchet_core_window`), not to master CI.
- **PR #319.** It is redundant now that master carries the same pin.

## Rules for every phase

- Read `lint_and_test.md`, `tui.md`, `tui_screenshot.md`, and (for any wait, refresh, or
  worker change) `tui_perf.md` through `/sase_memory_read`. Read `sase_beads.md` before
  touching beads.
- Re-inventory on the current master tip before editing. The evidence above is a
  starting point, not a whitelist. In-flight epics (for example `sase-1jm.3.1.7` and
  `sase-1jm.4.1.5`) may commit Agents-tab goldens. Before regenerating, rebase onto
  current master and rerun your check if an owned golden moved upstream.
- **Fix waits and fixtures, never tolerance.** Do not raise timeouts, add retries around
  assertions, skip, xfail, loosen pixel comparison, or delete assertions. Wait on the
  semantic state the capture is meant to show (loaded rows, a footer hint, a settled
  count), not on visual idle or a title that renders before loading.
- **No golden from an unfixed test.** Regenerate a golden only after its test is
  host-independent. Never copy CI candidate PNGs into the tree.
- **Owned goldens only.** Each repair phase regenerates goldens only with targeted
  `just fix-tui-screenshots -- <owned test files>`. It inspects every created and
  updated image in the report. Accept only:
  1. The phase's intended change.
  2. The `Models: <model>` summary line from `d9ee437e7e`.
  3. The dictionary card from `f1f37fcdfa`.

  Anything else is a new determinism bug to fix, or to record as `PROPOSED FOLLOW-UP:`.

- **Host-independence proof.** For each changed visual test file, run the targeted
  `just fix-tui-screenshots --check -- <files>` twice:
  - once normally;
  - once with `TZ=UTC` exported for the whole command (CI's collection-time zone).

  Both must report clean with no failed tests.

- Run long visual commands through `/sase_monitor` with `TESTING`/`TESTED` and a precise
  `--next`. Run `just fix` and `sase tool run check` for every changed checkout. Do not
  run `just check-full`; Full CI is the exhaustive signal.
- Shared domain behavior stays in sase-core. None of these fixes needs a Rust change. If
  one turns out to, follow `rust_core_backend_boundary` and move
  `sase-core-revision.txt` past the landed core commit.
- Phase workers create no beads. They record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead and close only that phase with `sase bead close <id> --note '<evidence>'`,
  after `sase bead epic-symbols <own-phase-id>` is clean. Land through `/sase_final`.

## Phase picker-race

Fixes the current Master Gate red and the Full CI snippet-finder timeout.

1. In `src/sase/ace/tui/modals/save_location_picker_modal.py`:
   - The bug: `set_choices` and `set_load_error` set state and then call `_rebuild()`.
     `_rebuild()` returns early while `not self.is_mounted`, and `_finish_mount` only
     resets the highlight and footer. Data that arrives between compose and mount leaves
     the list at `Finding destinations…` forever, even though `loaded` is true.
   - The fix: make the mount path render whatever state was delivered, for example
     `_finish_mount` calling `_rebuild()` (keep `can_focus = False`). It must be
     idempotent and must not re-dismiss or replay a buffered pending pick that
     `set_choices` already resolved.
2. Add a non-visual regression test that delivers choices, and separately a load error,
   before the modal mounts. Assert that the option list shows the choices or the error,
   never the loading row. Rerun the two flaky action tests above 20 times serially, then
   their files under `-n 8` three times. Record pass counts.
3. Make the mini-macro location PNG tests (`test_ace_png_snapshots_mini_macro.py`,
   picker and finder) host-independent:
   - **Isolate from real files.** The flow in
     `src/sase/ace/tui/actions/agent_workflow/_prompt_bar_mini_macro_location.py`
     (`_load_and_show`) reads real user and home config. Stub those loaders, or point
     them at a deterministic fixture tree, the way the snippet location PNG tests stub
     `_load_snippet_*`.
   - **Wait for loaded content.** The picker must wait for loaded content (a destination
     row such as `Project macros`), not just the `where should it live?` title.
   - **Refresh and inspect both goldens.** The picker must show the loaded list. The
     finder must show only fixture definitions. Neither may show any `/home/<user>`
     path.
4. Prove `snippet_location_flow_*` and `snippet_name_collision` stay unchanged and pass
   their targeted check under both TZ settings.

Owned goldens: `mini_macro_location_flow_*`, `snippet_location_flow_*` (expected
unchanged).

## Phase clock-seams

1. **Zone at import.**
   - Build `FIXED_NOW`/`FIXED_NOW_TS`/`T0_TS` in
     `tests/ace/tui/visual/_ace_agents_final_png_snapshot_shared.py`, and
     `_FOLLOW_SINCE` in `tests/ace/tui/visual/test_ace_png_snapshots_agents_waiting.py`,
     with an explicit `ZoneInfo("America/New_York")`. That zone matches
     `_pin_configured_timezone` in `tests/_conftest_runtime.py`; follow the pattern in
     `tests/ace/tui/test_agent_wait_epic_follow_tui.py`.
   - Keep `FIXED_NOW` and `FIXED_NOW_TS` describing the same instant.
   - Grep the visual fixtures for other naive `datetime(...)` → `.timestamp()` values
     that feed rendering, and fix them the same way.
   - Expected: the current `agents_final_*` and `agents_waiting_epic_follow_*` goldens
     pass unchanged under both TZ settings.
2. **Pager wall clock.**
   - Add one screen helper that returns `_time_band_now_epoch` when set and
     `int(time.time())` otherwise. Use it in `src/sase/pager/_screen_chrome.py`
     (`_history_chrome_state` age), the loading path and the main path of
     `src/sase/pager/_screen_time_band.py`, and `src/sase/pager/_screen_timeline.py`
     (`build_picker_rows`).
   - Declare the attribute on the screen so the test no longer needs
     `# type: ignore[attr-defined]`. Production never sets it, so runtime behavior is
     unchanged.
   - Add a unit test that the subject-chip age uses the pinned epoch.
   - Regenerate the 18 `timeband_*` goldens that carry an age (past, cause, diff-range,
     many-versions × dark/light × 120x40/60x30, plus `narrow_trail` dark and light). In
     each, the age token must be the only change.
3. **Memory pane clock.**
   - Pin "now" for `tests/ace/tui/visual/test_ace_png_snapshots_memory_pane_review.py`;
     the card ages with `time.time()` in `memory_pane_changes_card.py`. Use the
     monkeypatch style the FINAL tests use, or a seam if one already exists nearby.
   - Correct the test docstring that claims no wall-clock reads.
   - Prefer an epoch that keeps the committed `16d ago` golden byte-identical. Otherwise
     regenerate both images.
4. **Retry hostname.**
   - Pin `socket.gethostname` to a fixed name in `FakeyRetryHarness.__init__`
     (`tests/fakey/harness.py`). Do not change the continuation wire's owner semantics
     in production.
   - Regenerate the `agents_retry_e2e_*` goldens. The countdown, completed-chain and
     running-fallback images should change only in the artifact suffix.
     `agents_retry_e2e_plan_session_countdown` may also gain the Models line.
5. Run the host-independence proof over every visual file this phase touched.

Owned goldens: `timeband_*`, `memory_pane_changes_lens_review_*`, `agents_retry_e2e_*`,
`agents_final_*` and `agents_waiting_epic_follow_*` (the last two expected unchanged).

## Phase deck-races

1. **Tool Runs fast path.**
   - The bug: `ToolRunsDeckView.update_display`
     (`src/sase/ace/tui/widgets/decks/tool_runs/widget.py`) paints from a cached node
     summary. If any block's detail is missing from the detail cache, that block renders
     `run detail unavailable · summary retained` and no worker starts. Meanwhile, an
     earlier worker is dropped as stale when the agent-detail generation bumps.
   - The fix: when cached block details are incomplete, the fast path must still start
     or keep a load, so loaded detail replaces the placeholder. A generation bump must
     not strand the subject's only load.
   - Add a non-visual regression test: summary cached, details missing, generation
     bumped mid-load. The final render must show the argv line and the loaded detail.
2. **Tool Runs fixture.**
   - In `tests/ace/tui/visual/test_ace_png_snapshots_tool_runs_session.py`, seed after
     `patch_startup_loaders`, which now resets the ToolRun snapshot, as the header and
     rows tests already do. Fix the cards test if it shares the ordering.
   - In `tests/ace/tui/visual/_ace_tool_runs_png_snapshot_shared.py`, make the shared
     stub loader store detail through the same detail-cache path production uses.
   - Wait for `no log recorded` and the `$ just check` argv line in the rail. The
     `s1000000` sentinel renders from the brief alone, so it is not a load signal.
3. **Footer and turn-count settle.**
   - The Runs load handler refreshes panel chrome but not the footer, so the
     `H compact Runs`, `cards` and `blocks` hints go stale. Confirm this against the
     footer convention in `src/sase/ace/CLAUDE.md`; if it is a product bug, refresh the
     footer when the Runs document is stored.
   - Make `test_ace_png_snapshots_agents_agent_session_panel_gate.py` wait for the MAIN
     deck's blocks to be navigable (`(/) blocks` in the footer) before capturing.
   - Make
     `test_ace_png_snapshots_agents_agent_sessions.py::test_running_agent_session_current_runtime_png_snapshots`
     wait for the settled main-turn count (`main 0` vs `main 1` flips locally).
4. **sase-1ii.**
   - Root-cause why
     `test_ace_png_snapshots_agents_deck_blocks.py::test_agents_deck_blocks_spread_deck_sticky_png_snapshot`
     never reaches "Small session renders a spread deck", and why
     `test_ace_png_snapshots_agents_decks.py::test_agents_decks_left_right_search_committed_png_snapshot`
     never satisfies its predicate. The first fails locally on `725f7ea04e` and in Full
     CI 38116223239, but passed in 38143553606.
   - Fix the layout selection or the wait, not the timeout.
   - Record in the phase close note that `sase-1ii` is resolved, so the land agent can
     close it with that evidence.
5. **Stress and refresh.**
   - Run each repaired visual node five times targeted, and each repaired file once
     under `-n 8`.
   - Then refresh owned goldens and run the host-independence proof.

Owned goldens: every image produced by `test_ace_png_snapshots_tool_runs_session.py`,
`test_ace_png_snapshots_agents_deck_blocks.py`,
`test_ace_png_snapshots_agents_decks.py`,
`test_ace_png_snapshots_agents_agent_session_panel_gate.py`, and
`test_running_agent_session_current_runtime_png_snapshots`. Changes beyond each fix are
limited to the Models line.

## Phase rebaseline

1. Start from current master with all three repair phases landed. Confirm that each
   phase's owned targeted check is clean on that tip.
2. Run the full `just fix-tui-screenshots` update through `/sase_monitor`
   (`TESTING`/`TESTED`). The run must end with status `applied` or `clean`, never
   `partial`, and with zero failed tests. If it is `partial`, read the WARNING block and
   the manifest's `skipped` list, fix the cause, and rerun.
3. Inspect every creation and every update group, expanding any group with unexpected
   differences. Expected:
   - the `Models: <model>` line from `d9ee437e7e` on the remaining Agents-tab session
     and detail goldens;
   - the `f1f37fcdfa` dictionary card: `word_definition_modal_120x40` updated, plus the
     three created `word_definition_modal_*` images. First confirm that the word-lookup
     tests read only `tests/fixtures/dict` and make no network calls.

   Anything else, especially a time token, hostname or path, `/home/<user>`, or a
   loading placeholder, is a missed determinism bug. Fix the test or seam rather than
   accepting the image.

4. Run the full `just fix-tui-screenshots --check` with `TZ=UTC` exported through
   `/sase_monitor`. It must report status clean, `created=0 updated=0 stale=0`, and no
   failed tests.
5. Run `sase tool run check`.

> [!decision] tui_screenshot_rule
>
> When accepted, add two or three sentences to `tui_screenshot.md` › Golden Maintenance,
> then run `sase memory init`. The rule: a golden must not depend on the wall clock, the
> host zone at import time, `socket.gethostname()`, the real HOME or user config, or a
> loading frame. Prove it with a `TZ=UTC` targeted check before committing images. CI
> renders on UTC runners with per-job hostnames and `HOME=/home/runner`.

> [!decision] tui_screenshot_rule = no
>
> Leave memory untouched. If no human reviewed this plan, record the proposed rule as a
> `PROPOSED FOLLOW-UP:` note on this phase bead.

## Phase ci-proof

1. Confirm every phase commit is on `origin/master` with `git fetch` and `git log`.
   Watch Master Gate on that tip, or on a newer one, through `/sase_monitor` until it
   completes green.
2. Dispatch Full CI on master (`gh workflow run "Full CI" --ref master`) once Master
   Gate is green, and monitor it to completion. Acceptance: Master Gate and Full CI are
   both green on the same SHA, across every job:
   - `build-core`
   - `lint`
   - `test` (3.12 coverage leg, 3.13, 3.14)
   - `visual-test`
   - `perf-floors`
   - `ace-page-group-isolation`
3. **If a job is red:**
   - **Test node matching a known flake owner** (`sase-1gh`, `sase-1ib`, the
     `test_link_follow_seam` beads-sync race in `sase-1i5.9.1.2.1.7` note 8, or another
     single-hit node listed above): rerun only the failed jobs once with
     `gh run rerun <id> --failed`, and record both attempts.
   - **New flaky node:** record it as `PROPOSED FOLLOW-UP:`.
   - **Deterministic `visual-test` drift or timeout:** this is a determinism case the
     earlier phases missed. Fix it minimally here under the same rules, then dispatch a
     fresh Full CI.
   - **Never** relax a gate or edit CI schedules, shards, or workflows to get green.
4. Record the SHA, both run URLs, and any rerun as a note on this phase. Also add that
   evidence as a note on the stale same-goal phase `sase-1i5.9.1.2.1.7`; do not close
   it.
