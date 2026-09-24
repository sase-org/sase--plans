---
tier: tale
title: Return just symvision to green on master
goal:
  just _lint-symvision exits 0 on master with every flagged symbol fixed by deletion,
  privatization, a genuine public rename, or a justified epic-symbol entry, and the
  tracking task beads are closed.
size: medium
proposed_by: bbugyi200.athena.0qz
bead: sase-17m.3.1
create_time: 2026-09-24 12:32:41
status: wip
---

- **BEAD:**
  [sase-17m.3.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-17m/sase-17m.3.1.md)

# Plan: Return `just symvision` to green on master

## Goal

`just symvision` (the `_lint-symvision` stage of `just lint` / `just check`) fails on
master, so every agent's `just check` stops before the test lane. Fix every failure
properly, following `sase/memory/symvision.md` (read it first with
`sase memory read symvision.md -r "<why>"`). Delete dead code, make single-file helpers
private, rename genuine cross-module helpers to public names, and whitelist with
`--epic-symbol` only when a named, still-open epic phase will consume the symbol. Do not
patch Symvision, add blanket pragmas, or add new `--exclude-decorator` flags.

## What is actually failing (investigation summary)

Symvision runs its checks in order and stops at the first failing category. The visible
failure hides two more categories behind it.

1. **Visible: "Private functions/classes should not be imported" (~70 lines).** Three
   `toobig` routine splits (agents `toobig-5y.*`, not epic work) moved `_`-prefixed
   helpers into sibling modules and import them across files:
   - `8cc58fb06` split `llm_provider/usage/refresh.py` into `_refresh_*`
   - `d7faebddf` split `llm_provider/usage/presentation.py` into `_presentation_*`
   - `ebbfbc91b` split `ace/tui/modals/plugins_browser_install.py`

   Symvision matches private names by _name_, so every other same-named private def in
   the repo is listed too. That covers `_optional_text` in 15 files, `_number`,
   `_string_list`, and `_format_number` in 3 files. It also covers `_failure_count` in
   `doctor/checks_providers.py`, `_normalize_origin` in `alias_history.py`, and
   `_provider_rows` in `stats/_perf_view_latency.py`. These are noise and clear once the
   split modules stop importing privates across files. Do **not** rename them.

   These failures are already tracked by task beads **sase-17c** (plugins_browser),
   **sase-17j** (presentation) and **sase-17l** (refresh), each with several +1s.

2. **Hidden: "Private functions/classes must be used in the file where they are
   defined".** Eight dead privates (list below).
3. **Hidden: "Unused public functions/classes".** Twenty-four symbols (list below). Most
   were added by phases of in-progress epics. They could not see their own symvision
   errors because stage 1 was already red.

**Epic review.** I reviewed the 18 open epics created in the last five days. None of
them caused stage 1. Stages 2 and 3 are almost entirely epic work:

| Epic                                           | Status      | Hidden symvision symbols it introduced                                                                                                                                                                                                                                                                        |
| ---------------------------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sase-17d / sase-17d.10.1 (Agents tab decks)    | in progress | `_SectionLayoutPublisher`, `CardSeparator`, `DeckPanelSnapshot`, `NodeSpineExpandRequested`, `spine_geometry`, `default_card_id`, `enter_zoom`, `exit_zoom`, `lower_bound_rows`, `slow_tool_overflow_hint`                                                                                                    |
| sase-17x (`:` Command Line)                    | in progress | `CompletionSpecCacheError`, `command_line_spec_key`, `command_line_spec_path`, `ensure_command_line_spec`, `policy_table_paths`                                                                                                                                                                               |
| sase-17p (ToolRun hand-off)                    | in progress | `MonitorToolHandoff`, `parse_monitor_tool_words`, `envelope_from_resolved`, `read_output_tail` (from sase-17p.4, `df8ed5134`), `_infer_attribution` (a wrapper left behind by `c0591adf2`)                                                                                                                    |
| sase-17m / sase-17m.3.1 (agent session rename) | in progress | `_stored_family_role`, `_split_agent_family_name`, `_agent_family_suffix`, `_reserved_agent_family_names`, `_allocate_agent_family_child_name`, `agent_session_suffix_token`, `is_agent_session_member`, `agent_session_role_for_suffix`, `allocate_agent_session_child_suffix`, `normalize_runtime_group_by` |

The other recent epics (sase-142, sase-14j, sase-142.5, sase-14t, sase-158, sase-158.6,
sase-165, sase-169, sase-16e, sase-16g, sase-17y, sase-17z) touch none of the flagged
symbols. Two more flagged symbols come from closed task beads, not epics:
`streams_at_rev` (sase-17q, `1ba4e80e3`) and `_loose_object_count` (sase-17r,
`a054efc58`).

The older task beads **sase-16u** (`ExpandedLaunchSegments`) and **sase-16l**
(`delete_paths_in_background`) no longer reproduce. `caca6b60f` and `c87c9d1aa` fixed
them, and both beads have notes saying so. They are still open.

No linked repo (sase-telegram, sase-github, sase-research-artifacts) imports any symbol
this plan renames, privatizes, deletes, or drops from a facade.

## Step 0 — Sync and re-derive the failure list

Several of these epics are landing concurrently. `sase-17d.10.1.1` is closed but its
commit may not be on master yet, and `17d.10.1.2`, `17x.3`, `17p.5`, `17m.4` and `17m.5`
are active. So:

1. Rebase onto the latest `origin/master`, then run `just install`.
2. Reproduce with `just _lint-symvision`.
3. Reveal the masked stages with this throwaway probe. Write it to a temp file outside
   the repo and never commit it. It skips earlier stages so later ones print.

   ```python
   # stage-2 probe: pretend no private is imported cross-file
   import sys
   import symvision.cli as cli
   orig = cli.batch_search_usage
   def patched(symbols, infos, tracked=None):
       if symbols and all(s.startswith("_") for s in symbols):
           return set()
       return orig(symbols, infos, tracked)
   cli.batch_search_usage = patched
   # stage-3 probe: uncomment to also skip the in-file private check
   # cli.batch_check_private_in_file = lambda info: (list(info.private_symbols), [])
   sys.argv = ["symvision"] + sys.argv[1:]
   sys.exit(cli.main())
   ```

   Run it as `.venv/bin/python <probe> src/sase` with the same `--exclude-decorator` /
   `--epic-symbol` arguments as the Justfile recipe.

If a newly landed commit changed a symbol's situation, re-apply the same decision rules
rather than following this plan blindly. Examples: 17d.10.1.2 already changed
`slow_tool_overflow_hint`, or a new unused public appeared. Record any deviation in your
final summary.

## Step 1 — Stage 1: cross-module privates in the three split packages

General rule: inside the already-private `_refresh_*` / `_presentation_*` modules, drop
the underscore only on helpers that more than one module genuinely calls. Move a helper
that only one other module uses into that module. Remove every private re-export from
the facades. Keep the facades' _public_ late-import patch seams: tests monkeypatch
`refresh.eligible_usage_providers`, `refresh.admit_provider_usage_refresh` and friends,
`refresh.usage_probe_floor`, `refresh.usage_cli_fingerprint`, and
`presentation.provider_usage_format_remaining_text`. Keep every module under 500 lines
(toobig limits are 1000/850/700).

### 1a. `src/sase/llm_provider/usage/refresh.py` + `_refresh_*`

- `_UsageRefreshProviderResult` (`_refresh_model`) → rename to
  `UsageRefreshProviderResult`. Update the imports in `_refresh_submit` and
  `_refresh_execution`, and keep the facade re-export, adding it to `__all__`. It is the
  element type of `UsageRefreshReceipt.providers` and tests import it from `refresh`
  (`test_models_panel_usage_modal`, `test_axe_chop_usage_refresh`,
  `ace/tui/test_refresh_panel_dispatch`, `main/test_usage_command`,
  `llm_provider/test_usage_presentation`).
- `_release_started`, `_run_inline_batch`, `_submit_started_proc` (`_refresh_execution`,
  imported by `_refresh_submit`) → drop the underscore.
- `_resolve_requested_providers` (`_refresh_eligibility`, used only by
  `_refresh_submit`) → move it into `_refresh_submit` and keep it private. Keep its late
  `eligible_usage_providers` import from the facade.
- `_provider_has_probe_capability` (`_refresh_eligibility`, late-imported only by
  `_refresh_triggers`) → move it into `_refresh_triggers`, call it directly, and keep it
  private.
- `_mark_usage_refresh_due` (`_refresh_triggers`, which resolves it through the facade)
  and `_referenced_provider_ids` (`_refresh_eligibility`, same pattern) → call them
  directly in their own module. Delete the late self-imports through the facade.
- `_provider_cli_ready`, `_admit_one`, `_disabled_receipt`, `_normalize_execution`,
  `_normalize_origin`, `_live_inline_providers`, `_record_inline_crash`,
  `_runner_payload` → used only in their own module, so just delete them from the facade
  import.
- Tests: retarget private imports and monkeypatch targets to the defining module. For
  example,
  `monkeypatch.setattr("…usage._refresh_eligibility._referenced_provider_ids", …)` in
  `test_usage_eligibility` and `test_agy_usage_probe`. In `test_usage_hot_cadence`,
  patch `_refresh_triggers._provider_has_probe_capability` and import `_admit_one` from
  `_refresh_submit`. In `test_usage_refresh`, patch
  `_refresh_triggers._mark_usage_refresh_due`. In `test_usage_adaptive_admission`,
  import `_admit_one` from `_refresh_submit`. In `test_usage_eligibility`, import
  `_provider_cli_ready` from `_refresh_eligibility`. Grep `tests/` for every name above
  to catch string-form patch targets.

### 1b. `src/sase/llm_provider/usage/presentation.py` + `_presentation_*`

- Shared helpers used by several modules → make them public in `_presentation_shared`.
  Choose names that don't collide with existing public defs elsewhere in `src/sase`:
  - `_number` → `finite_number` (a public `number` already exists in
    `stats/_view_payload.py`)
  - `_optional_text` → `nonblank_text` (a public `optional_text` already exists in
    `main/monitor/common.py`)
  - `_failure_count` → `failure_count`
  - `_provider_collector_health` → `provider_collector_health`
  - `_format_remaining_text` → `format_remaining_text`, keeping its late
    `presentation.provider_usage_format_remaining_text` seam
- Move single-consumer helpers into their only consumer and keep them private:
  - `_provider_rows` → `_presentation_snapshot`
  - `_string_list` → `_presentation_labels`
  - `_window_rows` and `_format_number` → `_presentation_render`
  - `_age_from_timestamp`, `_remaining_label`, `_window_source_label` (from labels) →
    `_presentation_render`

  A public `remaining_label` already exists in `models_panel_provider_state.py`, so
  keeping this one private avoids a name clash.

- Label and snapshot helpers used by render → drop the underscore:
  - `_provider_window_style` and `_window_status_label_for_provider` in
    `_presentation_labels`
  - `_display_provider_rows` and `_usage_diagnostic_to_json` in `_presentation_snapshot`
- `_collector_health_style` is only used by the public one-line wrapper
  `collector_health_style` and by in-file callers. Fold its body into
  `collector_health_style` and delete the private function. Point the in-file callers at
  the public name.
- `_collector_health_label` stays private in `_presentation_labels`.
- Delete both `_collector_health_*` imports from `presentation.py`.
- Tests (`tests/llm_provider/test_usage_presentation.py` ~L129/132): import
  `_collector_health_label` from `_presentation_labels` and call
  `presentation.collector_health_style`.
- Check that no function-local variable shadows any newly public name. None does today.

### 1c. `src/sase/ace/tui/modals/plugins_browser_install_*`

- Move these into `plugins_browser_install_combined.py`, their only real consumer, and
  keep them private:
  - `_CombinedInstallOutcome`, from `_previews`. `_messages` names it only in a type
    hint, so drop that use or restructure it.
  - `_combined_install_message`, from `_messages`
- Move `_source_variant_label` (+ `_SOURCE_VARIANT_LABELS`) from `_messages` into
  `plugins_browser_install_single.py`, its only consumer.
- `_install_many_skipped_message` is used by both `single` and `combined` → rename it to
  `install_many_skipped_message` in `_messages`.
- No tests reference these names. Re-run the plugins browser tests anyway.

## Step 2 — Stage 2: dead private symbols

- **Delete** the five "deprecated wrapper" privates in `src/sase/plan_chain.py`:
  - `_stored_family_role`
  - `_split_agent_family_name`
  - `_agent_family_suffix`
  - `_reserved_agent_family_names`
  - `_allocate_agent_family_child_name`

  `29ca9ee46` moved every caller to the `_…session…` equivalents, and nothing calls
  these. Keep the _public_ `agent_family_*` wrappers; about 20 source files still use
  them.

- **Delete** `_infer_attribution` in `src/sase/main/proc_handler.py`. `c0591adf2`
  replaced its only call with `sase.procs.attribution.infer_proc_attribution`. Keep the
  `Path` import, which is still used.
- **Delete** `_loose_object_count` in `src/sase/sdd/_store_maintenance.py`. `a054efc58`
  replaced its only caller with `_loose_object_stats`.
- **Delete** the unused `_SectionLayoutPublisher` Protocol in
  `src/sase/ace/tui/widgets/prompt_panel/_section_view.py`, and drop `Protocol` from
  that file's typing import if nothing else uses it.

## Step 3 — Stage 3: unused public symbols

Privatize means: add `_`, rename in-file uses and docstrings, remove the name from
`__all__`, and repoint tests at the private name. Where the plan says so, repoint tests
at the public entry point instead. Delete means: remove the symbol, its now-dead
helpers, and its tests.

**sase-17d decks.** No open phase consumes any of these, so none gets an epic-symbol.

- `CardSeparator` (`widgets/decks/separators.py`) → privatize. `test_deck_spread_pure`
  builds it through `main_separator_for(...)` instead.
- `DeckPanelSnapshot` (`models/agent_deck_persistence.py`) → privatize and update
  `test_agent_deck_persistence`.
- `NodeSpineExpandRequested` (`widgets/decks/node_spine.py`) → **do not just add `_`**.
  Textual derives the handler name from the class name, so that would silently break
  `on_node_spine_expand_requested` in `actions/agents/_deck_layout_actions.py`. Instead
  nest it as `class NodeSpine(Static): class ExpandRequested(Message)`, post
  `self.ExpandRequested()`, and keep the handler name `on_node_spine_expand_requested`.
  Add a small unit test asserting
  `NodeSpine.ExpandRequested.handler_name == "on_node_spine_expand_requested"`.
- `spine_geometry` → privatize and update `test_deck_collapse_zoom`.
- `default_card_id` (`widgets/decks/model.py`) → privatize. `test_deck_model` uses
  `resolve_active_card(ids, None, partial=False)`, which gives the same results.
- `enter_zoom` and `exit_zoom` (`widgets/decks/layout.py`) → privatize and update both
  docstrings that mention them. `test_deck_collapse_zoom` and
  `test_agent_deck_persistence` use `toggle_zoom`.
- `lower_bound_rows` (`widgets/decks/render_mode.py`) → privatize and update
  `test_deck_render_mode`.
- `slow_tool_overflow_hint` (`widgets/prompt_panel/_agent_slow_tools.py`) → privatize
  and update `test_agent_slow_tools`. sase-17d.10.1.2 edits this function's hint list,
  so rebase carefully.

**sase-17x Command Line.**

- `ensure_command_line_spec` → `--epic-symbol 'sase-17x(ensure_command_line_spec)'`.
  Phase sase-17x.9 (completion-popup) consumes it. `plan:202609/command_line_panel.md`
  says: "run `ensure_command_line_spec` and then `load_command_line_grammar` in a thread
  worker".
- `CompletionSpecCacheError` → `--epic-symbol 'sase-17x(CompletionSpecCacheError)'`. It
  is the failure type that `ensure_command_line_spec` raises, and the 17x.9 worker must
  handle it. Making it private would leave a public function raising an unnameable
  error.
- `command_line_spec_key` and `command_line_spec_path` → privatize. No phase consumes
  them. Update `tests/completion/test_spec_contract.py`.
- `policy_table_paths` (`completion/run_policy.py`) is used only by the drift test.
  Delete it from `src` and remove it from `__all__`. Build the same union inside the
  test from `_RUN_POLICY_TABLE`, `_WRITES_TRUE_OVERRIDES`, `_WRITES_FALSE_OVERRIDES` and
  `_STDIN_PATHS`. sase-17x.3 may be editing `test_spec_contract.py`, so rebase
  carefully.

**sase-17p ToolRun hand-off.** Phases 17p.5 and 17p.6 don't use these, so none gets an
epic-symbol.

- `MonitorToolHandoff` and `parse_monitor_tool_words` (`monitor/tool_handoff.py`) →
  privatize. `monitor/start.py` only uses attributes of the returned value. Update
  `tests/monitor/test_monitor_tool_handoff.py`.
- `envelope_from_resolved` (`tool/handoff.py`) → privatize and update
  `tests/tool/test_handoff.py`. `resolved_from_envelope` stays public.
- `read_output_tail` (`tool/control.py`, used only in its own file) → privatize and
  remove it from `__all__`.

**sase-17m agent session rename.**

- `agent_session_suffix_token`, `is_agent_session_member`,
  `agent_session_role_for_suffix`, `allocate_agent_session_child_suffix`
  (`plan_chain.py`) → one `--epic-symbol 'sase-17m(<name>)'` entry each. They are the
  canonical replacement names. Today only the deprecated public `agent_family_*`
  wrappers call them, and those wrappers still have about 20 callers. Phase sase-17m.4
  (runtime cutover: "Do not keep internal aliases just to shrink the diff") and
  sase-17m.5 (ACE surfaces) move those callers onto these names and delete the wrappers.
  Moving the callers here would collide head-on with those phases.
- `normalize_runtime_group_by` (`stats/query.py`) → privatize. In
  `tests/test_agent_session_durable_json.py`, drop the direct import and asserts; the
  same test already checks the behavior through `query_run_stats`.

**Closed task-bead leftovers.**

- `streams_at_rev` (`bead/_stream_integrity_git.py`) → delete it. `1ba4e80e3` replaced
  all its callers with `stream_paths_at_rev` + `batch_show_texts`. Then remove any
  imports that become unused.

## Step 4 — Justfile

In the `_lint-symvision` recipe, add the six `--epic-symbol` entries from Step 3 (two
for sase-17x, four for sase-17m). Above them, add a comment in the existing style naming
the consuming phases: 17x.9 consumes the two sase-17x symbols, and 17m.4/17m.5 consume
the four sase-17m symbols and remove their entries. Confirm the list with
`sase bead epic-symbols`.

## Step 5 — Bead bookkeeping (after verification passes)

- Close the three tracking task beads with what you verified:
  `sase bead close sase-17c sase-17j sase-17l --note "<commit + just _lint-symvision clean>"`.
  If `close` takes only one ID, close them one at a time.
- Close sase-16u and sase-16l as no longer reproducing. Name the fixing commits
  (`caca6b60f` privatized `_ExpandedLaunchSegments`; `c87c9d1aa` fixed
  `delete_paths_in_background`) and say that a symvision run on the new master is clean.
- Append a note (`sase bead note <id> "<text>"`) to each consuming or affected phase
  bead so in-flight workers see the change:
  - **sase-17x.9:** it consumes `ensure_command_line_spec` / `CompletionSpecCacheError`
    and must remove both `sase-17x(...)` Justfile entries.
  - **sase-17m.4 and sase-17m.5:** migrating the `agent_family_*` callers must remove
    the four `sase-17m(...)` entries.
  - **sase-17d.10.1.2:** `slow_tool_overflow_hint` is now private, and
    `NodeSpineExpandRequested` is now `NodeSpine.ExpandRequested`.
  - **sase-17p:** `MonitorToolHandoff`, `parse_monitor_tool_words`,
    `envelope_from_resolved`, and `read_output_tail` are now private.

  Do not close or edit any epic or phase bead's status.

- Recurrence: a `toobig` routine split breaking symvision with cross-module privates has
  now happened six times (sase-13l, sase-10n, sase-zk, sase-17c, sase-17j, sase-17l). A
  red stage 1 also hid epic regressions for a day. Use `/sase_new_task` to check for an
  existing task and file or corroborate one. It should cover two things: the toobig
  split routine prompt (configured outside this repo, in the user's chezmoi-managed SASE
  config) must require a `just _lint-symvision` result no worse than its base; and
  agents that see symvision already red should still compare the full masked-stage
  output. Do not edit the chezmoi repo in this tale.

## Verification

1. `just _lint-symvision` exits 0.
2. The Step 0 probe prints nothing for stages 2 and 3.
3. `sase bead epic-symbols` lists exactly the six new entries, and symvision accepts
   them: beads open, symbols public and not yet imported.
4. Run the focused tests for every touched test file and module group:
   - `tests/llm_provider/test_usage_*.py`
   - `tests/test_models_panel_usage_modal.py`
   - `tests/test_axe_chop_usage_refresh.py`
   - `tests/ace/tui/test_refresh_panel_dispatch.py`
   - `tests/main/test_usage_command.py`
   - plugins browser tests
   - `tests/ace/tui/widgets/decks/`
   - `tests/**/test_agent_slow_tools.py`
   - `tests/completion/test_spec_contract.py`
   - `tests/monitor/test_monitor_tool_handoff.py`
   - `tests/tool/test_handoff.py`
   - `tests/tool/test_lifecycle_controls.py`
   - `tests/test_agent_session_durable_json.py`
   - plan_chain tests
   - `tests/sdd_store/`
   - bead stream-integrity tests
5. `just fix`, then `sase tool run check`. Its lint stage must be fully green. Do not
   run `just check-full`.
6. No PNG goldens should change, because every edit is a rename, move, or deletion with
   no visual effect. If any golden changes, stop and investigate.
