---
tier: tale
title: Turn sase master CI green with one stop-the-line repair commit
goal:
  Every deterministic failure currently red on the Master Gate and Full CI (lint,
  collection, test, visual-test, and perf-floors jobs) is fixed in one reviewed commit,
  with no long-term process or workflow changes.
size: medium
proposed_by: bbugyi200.athena.0uw
create_time: 2026-10-01 12:51:17
status: wip
---

<!-- sase:links:start -->

## Links

| Relation     | Artifact                                                                                                | Why                                                                                |
| ------------ | ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| derives-from | [research:202610/master_ci_red_why_and_how_to_stay_green/master_ci_red_why_and_how_to_stay_green.md][1] | Section 4 stop-the-line pass and the failure inventory this repair plan implements |

[1]:
  https://github.com/sase-org/sase--research/blob/main/202610/master_ci_red_why_and_how_to_stay_green/master_ci_red_why_and_how_to_stay_green.md

<!-- sase:links:end -->

# Turn sase `master` CI green: one stop-the-line repair commit

## Goal

`master` has been red on every Master Gate run since 2026-09-18, and Full CI has been
red for even longer. The research report
`research:202610/master_ci_red_why_and_how_to_stay_green/master_ci_red_why_and_how_to_stay_green.md`
covers the background. Its §4 says to fix every current deterministic break in one
reviewed commit by a single build-cop agent. This plan is that commit.

**In scope:** fixes for the failures that are red right now.

**Out of scope:** the report's longer-term recommendations:

- the green-or-revert landing contract
- the landing freeze/hold
- `.python-version` parity
- symvision and selector unmasking
- CI workflow changes

Do not touch `.github/workflows/`, the finalizer, or `ci_watch`.

Every item below was reproduced at master `c5ecf13a48`, where the latest Master Gate and
Full CI runs failed. The root causes were traced to their culprit commits. Nearly every
fix is a stale test pin, which means the culprit commit changed behavior on purpose and
missed a test. The exceptions are one TUI source fix (item 12) and four PNG goldens. No
change crosses into `sase-core`.

## Step 0: environment and re-inventory (do this first)

1. **Refresh the dev env.** Run `sase tool run install`. Workspace venvs have been seen
   with a stale `sase_core_rs` (0.35.0) that lacks bindings master needs, such as
   `build_agent_tab_catalog`, `tool_run_record_demand`, `scan_note_attachment_refs` and
   `memory_history_wire_schema_version`. That produces dozens of local-only failures,
   which you must not "fix". After install, check that
   `.venv/bin/python -c "import sase_core_rs, importlib.metadata as m; print(m.version('sase_core_rs'))"`
   reports 0.36.x or newer.
2. **Re-inventory against the newest completed Master Gate.** `master` moves about 90
   commits a day.
   - List the newest runs with
     `gh run list --workflow "Master Gate" --branch master --limit 5`.
   - For the newest completed run, show its failures with
     `gh run view <id> --log-failed | grep -E "(FAILED|ERROR) tests|  [A-Za-z_]+ in src/"`.
   - **Already fixed upstream:** if an item below is already fixed on `master`, skip it.
     Phase sase-1dr.12 is actively verifying the sase-1dr epic, so items 5–8 may land
     first.
   - **New deterministic failure:** if the inventory shows a new deterministic failure
     with an obvious stale-pin cause, fix it the same way.
   - **Anything bigger:** report it in your final message instead of widening scope.

## Fixes

### A. Master Gate `lint` job

1. **symvision: three unused public symbols** (culprit `018061f6f2`, sase-1cx.3; bead
   sase-1dn). Tests never keep a public symbol alive.
   - `src/sase/tool/owner.py`: delete the dead `owner_ref` function and its `__all__`
     entry. The watchdog it was meant for calls `observe_owner_fact` instead.
   - `src/sase/tool/handoff_launch.py`: rename `HandoffSubmitResult` to
     `_HandoffSubmitResult` everywhere in the file and drop it from `__all__`.
   - `src/sase/tool/starter.py`: rename `StarterResolution` to `_StarterResolution`
     everywhere in the file and drop it from `__all__`.
   - `tests/tool/test_inline_escalation.py`: update its `HandoffSubmitResult` imports
     and uses to `_HandoffSubmitResult`. Tests may import private names.
   - Afterwards, symvision must print "All public/private classes/functions are used
     properly!" No other `just lint` gate is red at `c5ecf13a48`.

### B. Collection error (every test shard, perf-floors, visual)

2. **Broken facade import in `tests/ace/tui/widgets/test_agent_header_panel.py`**
   (culprit `c6b802a647`; bead sase-1dh).
   - **Imports:** the facade imports the deleted `test_hint_document_forces_expansion`
     from `test_agent_header_panel_basic`. Replace that import with the two tests that
     replaced it: `test_hint_mode_keeps_collapsed_header_unchanged` and
     `test_hint_mode_numbers_expanded_header_first`.
   - **`__all__`:** make the same swap there, keeping the list alphabetical.
   - **What it fixes:** the `ImportError`, plus
     `tests/test_contract_manifest.py::test_contract_manifest_matches_marker_selection`
     (its `pytest -m contract --collect-only` exits 2).
   - **Manifest:** `tests/contract_manifest.txt` already matches the post-fix selection
     (73 entries), so do not regenerate it. If it does drift, the command is
     `just refresh-contract-manifest`.

### C. Launch-seam fakes (bead sase-1cm, reopened once for the same class of break)

3. **Fakes that reject `history_text`.** Commit `ce0f61846c` made
   `src/sase/main/query_handler/_launch.py` pass `history_text=` to
   `launch_agents_from_cwd`. Make the fakes robust to future kwargs; do not add one more
   named kwarg.
   - **`tests/test_partial_launch_cleanup.py`:** change `fail_launch` to
     `def fail_launch(query: str, **kwargs: object)`, with `del query, kwargs`.
   - **`tests/test_force_reuse_launch_seam_rejection.py`:** two tests use
     `assert_called_once_with(...)`. Replace it with targeted checks in the style of the
     sibling `tests/test_force_reuse_launch_seam_consume.py`:
     `mock_launch.assert_called_once()`, then `args, kwargs = mock_launch.call_args`,
     `assert args == (prompt,)`, `assert kwargs["origin"] == "typed"`, and
     `assert kwargs.get("history_text") is None`.

### D. sase-1dr (memory history) leftovers. All intended changes; fix the pins

4. **`memory/history:at` has no completion caption** (culprit `ebf070e16a`).
   - In `src/sase/completion/kinds.py`, add `"at",` to the `"text"` tuple of
     `_VALUE_HINT_TABLE` in alphabetical position, between `"assignments"` and
     `"attestation"`. `--at` takes an ordinal, `~N`, a SHA prefix, `now` or a date;
     sibling options `since`/`until`/`before`/`after` are also `"text"`. It is the only
     `at` dest in the CLI.
   - Then run `just sync-completion-spec`, which runs
     `tools/sync_completion_spec --write`. Expect a one-line diff in
     `tests/completion/snapshots/cli_spec.json`: `"value_hint": null` becomes `"text"`.
   - Fixes `tests/completion/test_kind_coverage.py` and keeps
     `tests/completion/test_snapshot.py` green.
5. **Memory help subcommand list.** In
   `tests/main/test_parser_command_help.py::test_memory_help_marks_primary_command_and_init_alias`,
   expect `{agent-docs,history,init,list,log,read,show,web}`.
6. **Memory log JSON fields.** In
   `tests/main/test_memory_log.py::test_memory_log_json_id_outputs_raw_event`, add the
   new `MemoryReadEvent` fields from `297faf1d38`: `"blob_oid": None` and
   `"included_blob_oids": []`. Leave `"schema_version": 1` unchanged.
7. **Fourth artifact-index write.** In
   `tests/test_axe_run_agent_runner_started_at.py::TestRunStartedAtRecording::test_home_mode_running_marker_cleanup_updates_artifact_index`,
   expect `setup_index_update.call_count == 4`. The fourth update is the launch-evidence
   `write_agent_meta` in `run_agent_runner_launch.py`, added by `297faf1d38`. Add a
   short comment naming the four sources:
   - artifacts setup
   - bootstrap meta
   - home running marker
   - launch-evidence meta (sase-1dr.1)

### E. sase-1ck (bead attachments) leftovers (bead sase-1de, plus one with no bead)

8. **Unreviewed artifact-directory operation.**
   - In `tests/test_agent_artifact_directory_operation_audit.py`, add a `DirOpReview`
     exemption for `"src/sase/bead/attachments/lifecycle.py:quarantine_local_object"`
     right after the `store.py:remove` entry. Rationale: it moves a corrupt
     content-addressed attachment object into `SASE_HOME/attachments/quarantine` and
     removes only its derived view directory under `SASE_HOME/attachments/views`. It
     never touches an agent artifact directory.
   - The coverage-declaration companion test must stay green.
9. **Sidecar description wording.** In
   `tests/test_config_schema_repositories.py::test_config_schema_documents_intrinsic_agents_sidecar_contract`,
   assert `"~/.sase/projects/<project_key>/repos/<role>"`.
   - `e300173faf` deliberately generalized the hand-maintained description in
     `src/sase/config/sase.schema.json`, so do not edit the schema.
   - Keep the `"hidden agents"` assertion.
10. **Help-rendering assertion that only passes on 3.13+** (culprit `56d5cd277e`).
    Master Gate runs Python 3.12. In
    `tests/test_bead/test_show_images.py::test_parser_help_covers_images_and_open`,
    replace `assert "-i, --images" in show_help` with the existing portable helper:
    `assert_metavar_option_documented(show_help, "-i", "--images", "{auto,cells,kitty,never}")`.
    Import it from `tests.main.parser_help_helpers`. No other `"-X, --long"` help
    assertion in the repo is version-sensitive.

### F. TUI import budget (bead sase-13p)

11. **Module-count budget.**
    `tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
    is a deterministic module-count budget, not a timing test. CI measures 3489 on 3.12
    and 3491 on 3.14, against the cap of 3485.
    - **Why it grew:** pager memory-history mixins (read view, word diff, time band,
      timeline picker) are imported eagerly through `PagerScreen`, and toobig splits
      (finalizers, monitor, agents display) add more modules. Neither can be trimmed
      cheaply.
    - **Fix:** follow the precedent of the earlier bumps (3290 → 3400 → 3450 → 3485).
      Raise `_MAX_MODULE_COUNT` to 3530.
    - **Comment:** extend the comment block with one sentence naming those sources and
      the ~3490 CI measurement.
    - If the re-inventory shows CI already above ~3510 because of newer splits, choose
      the cap so CI keeps about 30 modules of headroom, and say so in the comment.

### G. Full CI `visual-test` job (58 failed + 1 error)

12. **Facades collected twice (~52 failures).** These four visual facade modules
    re-export their split tests without the repo-standard `__test__ = False`, so each
    test is collected twice and the capture session raises
    `VisualCaptureError: duplicate canonical golden path`:
    - `tests/ace/tui/visual/test_ace_png_snapshots_tool_runs.py`
    - `tests/ace/tui/visual/test_ace_png_snapshots_command_line.py`
    - `tests/ace/tui/visual/test_ace_png_snapshots_agents_final.py`
    - `tests/ace/tui/visual/test_ace_png_snapshots_agents_sase_context.py`

    In each file, add `__test__ = False` right after `pytestmark = pytest.mark.visual`.
    This matches the 31 non-visual facades. Nothing imports from these facades, and
    their split modules already capture every golden pixel-exact, so no golden changes.

13. **Agent tab strip empty states (3 timeouts): a real source bug** (culprit
    `5b68fd9729`). The affected tests are `test_agents_tab_strip_empty_png_snapshot`,
    `test_agents_tab_strip_query_hides_png_snapshot` and
    `test_agents_tab_strip_feed_unavailable_png_snapshot`.
    - **Cause:** in `src/sase/ace/tui/widgets/decks/panel_navigation.py`, the end of
      `show_main_document` only calls `self.set_deck(DeckId.MAIN)` when
      `is_new_subject`. The empty document and the "No agents on …" cause document both
      have `subject=None`, so the empty → cards transition never calls `set_deck`, which
      is the only call that re-shows the main scroll container. The panel renders blank
      chrome.
    - **Fix:** change the condition to
      `if is_new_subject or bool(previous_document.cards) != bool(document.cards):`.
      This keeps the perf skip for same-subject streaming where cards exist on both
      sides. Check that `previous_document` cannot be `None` on that path; it comes from
      `self._main_document`. If it can, guard it.
    - **Golden:** `empty` and `query_hides` then pass pixel-exact.
      `agents_tab_strip_feed_unavailable_120x40` needs a refresh, because its local tab
      label changed from `local` to `⌂ athena` (from `6db24e776c`). Before accepting,
      confirm that the label comes from a pinned fixture hostname and not the real host.
      If it comes from the real host, pin it in the fixture instead.
14. **`test_ace_png_snapshots_axe_runs.py::test_axe_chop_run_info_panel_png_snapshot`**
    (stale test). `880f1f863e` intentionally sorts routines alphabetically, so the items
    are now
    `[bgcmd#1, checks, checks/smoke, hooks, hooks/fast_lint, hooks/slow_typecheck]`.
    - Press `j` five times, not three.
    - Assert `current_idx == 5`.
    - Fix the item-order comment and both assertion messages; the second message wrongly
      says "idx 2".
    - Keep the `("hooks", "slow_typecheck")` assertion.
    - Refresh golden `axe_chop_run_info_panel_120x40`.
15. **Narrow top-bar usage tests** (bead sase-18n):
    `test_top_bar_usage_attention_narrow_png_snapshot` and
    `test_top_bar_usage_badges_crowded_narrow_png_snapshot` time out in
    `wait_for_startup`.
    - **Cause:** `f4d70c452b` added the `inbox:` group label, which is dropped at narrow
      widths (`⚑1 ✉18`). `tests/ace/tui/visual/_ace_png_snapshot_startup.py` compares
      the full `inbox: ⚑1 ✉18` text exactly.
    - **Fix:** add a `_badge_body(text)` helper that strips
      `f"{NotificationIndicator.GROUP_LABEL}: "`. Compare
      `_badge_body(_indicator_plain(page) or "") == _badge_body(expected_badge)`.
    - **Goldens:** refresh `top_bar_usage_attention_80x24` and
      `top_bar_usage_badges_crowded_60x24` (new `·` separators). In the 80x24 frame, the
      active "Artifacts" tab label disappears because the bar is crowded. Accept it as
      current behavior and file a follow-up (see Beads).

### H. Full CI `perf-floors` job

16. **`tests/test_sase_tool_runs_smoke.py::test_tool_runs_harness_passes_every_hermetic_case`**
    has two failing harness cases:
    - **`dod-1-catalog-sase`** (deterministic since `be6daf95d1` added
      `sase tool stats`). In `tools/_smoke_tool_runs_cases_basic.py`, expect
      `{failures,list,receipt,receipts,run,runs,show,stats,stop,wait}`.
    - **`dod-6-typical`** (intermittent; it failed in the latest run). `tool list`'s
      `last` orders by whole-second `created_ts` and breaks ties by random `run_id`, so
      the `--extra` run can lose to a same-second cohort run.
      - In `tools/_smoke_tool_runs_cases_evidence.py` `typical_cohorts`, add
        `time.sleep(1.1)` right after `baseline = quick()`.
      - Add a two-line comment explaining the tie.
      - Add `import time`.

### I. Load-flake hardening (the cause is confirmed and the fix is small)

17. **`tests/ace/tui/modals/test_node_finder_modal.py::test_invalid_keys_flash_without_dismiss_or_app_leak`**
    (`assert 'no hint' in ''`). The 1.2 s `_FLASH_S` timer clears the flash under load.
    Add a `monkeypatch` parameter and call
    `monkeypatch.setattr("sase.ace.tui.modals.node_finder_modal._FLASH_S", 60.0)` before
    `run_test`. Apply the same patch to `test_quotation_mark_back_and_flash`, which has
    the same exposure.
18. **`tests/ace/tui/test_screenshot_export.py::test_signal_schedule_spawns_export_task`.**
    Its frame convergence runs out of the 3.0 s settle budget under load. Follow the
    file's existing pattern:
    `monkeypatch.setattr(screenshot_export_module, "_SCREENSHOT_SETTLE_TIMEOUT_SECONDS", 10.0)`,
    and use `timeout=15.0` on the test's `page.wait_for`.

**Leave alone (known load flakes with beads; do not change):**

- `test_deck_block_spread_pilot` (sase-1bl, sase-1br)
- `test_ace_page_group_reports_reset_hook_leaks` (sase-13c)
- `test_plugins_browser_pane_cached_open.py::test_mutation_completion_after_unmount_invalidates_memo`
- `tests/ace/tui/actions/test_prompt_snippet_location_flow.py::test_shift_tab_round_trip_preserves_trigger`
  (caught mid-load: `'Finding destinations…'`)
- `tests/test_sudo_acceptance_credentials.py::test_sudo_local_flow_never_persists_canary_credentials`
  ("credential length leaked")

All of them pass 3/3 locally, and the last three appeared on one SHA only. If the
re-inventory shows any of the unbeaded ones failing again, file or +1 a `flake` bead
through `/sase_new_task` instead of changing them here.

## Verification

Run `just fmt` (or `just fix`) first. Then:

1. **Collection:** `.venv/bin/python -m pytest --collect-only -q` reports zero
   collection errors. `.venv/bin/python -m pytest -m contract --collect-only -q`
   exits 0.
2. **Targeted tests** with `.venv/bin/python -m pytest -q`, all passing:
   - every test file edited above
   - `tests/completion/`
   - `tests/ace/tui/widgets/test_agent_header_panel_basic.py`
   - `tests/test_force_reuse_launch_seam_consume.py`
   - `tests/ace/tui/widgets/decks/`
   - `tests/ace/tui/test_agent_tab_strip*.py`, since item 13 touches deck navigation
3. **perf-floors:** run
   `just test-slow -- tests/test_sase_tool_runs_smoke.py -k hermetic`. It takes about
   2–3 min; if it exceeds your synchronous limit, use `/sase_monitor`.
4. **Visual** (through `/sase_monitor` with `TESTING` / `TESTED` if it is long):
   - **Refresh the four goldens** with the targeted update form,
     `just fix-tui-screenshots -- <node ids for items 13–15>`. Inspect the run report
     (`.pytest_cache/sase-visual/latest-report.json`) and every golden diff. Generation
     is not approval.
   - **Check the touched suites** with
     `just test-visual -- tests/ace/tui/visual/test_ace_png_snapshots_{tool_runs,command_line,agents_final,agents_sase_context}*.py tests/ace/tui/visual/test_ace_png_snapshots_agent_tab_strip.py tests/ace/tui/visual/test_ace_png_snapshots_axe_runs.py tests/ace/tui/visual/test_ace_png_snapshots_provider_usage_indicator*.py`.
     It must report no duplicate-golden errors and no drift.
5. **Repo gate:** `sase tool run check` must pass. Every lint gate must be green,
   including symvision. Any remaining test failure must carry a KNOWN/FLAKY label
   pointing at a bead listed under "Leave alone".
   - `just check-full` is **not** required. Master Gate on the landed SHA is the
     acceptance signal.

## Beads and final report

- **Close the fixed beads** with `sase bead close <id> --note "<what was verified>"`:
  sase-1dn, sase-1dh, sase-1cm, sase-1de, sase-18n and sase-13p. Skip any bead that
  turns out to be already closed or fixed upstream.
- **Notes on epics:** add a `sase bead note` on epic sase-1dr saying this commit updated
  the four sase-1dr test pins (items 4–7). Add one on sase-1c1 recording the repair
  commit.
- **Follow-ups:** use `/sase_new_task` (check for duplicates first) to file these:
  - `tool list` `last` is nondeterministic for same-second runs; it needs a monotonic
    tie-break in sase-core.
  - The crowded 80x24 top bar hides the active "Artifacts" tab label.
  - `test_ace_png_snapshots_axe_descriptions.py` uses hard-coded indexes whose goldens
    were rebaselined with the wrong rows selected after `880f1f863e`.
  - Four non-visual facades lack `__test__ = False`, so their tests run twice:
    - `tests/ace/tui/test_artifacts_beads_rendering.py`
    - `tests/ace/tui/test_kill_and_edit_launch_barrier.py`
    - `tests/ace/tui/test_tribe_panel_flicker.py`
    - `tests/test_agent_model_bundle.py`

    Only add `__test__ = False` to these four if a check shows that's safe. Some facades
    are the sole collection point for `_`-prefixed split modules.

- **Final message:**
  - list each fix
  - list anything skipped because it was already fixed upstream
  - list any new failure the re-inventory found
  - say that, once this lands, the user should watch Master Gate on the landed SHA and
    trigger Full CI (`gh workflow run "Full CI" --ref master`) to confirm both are green
