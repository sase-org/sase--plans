---
tier: tale
title:
  Finish and close epic sase-1d7 (unread ack reliability and Agents TUI responsiveness)
goal:
  Fix the leftovers the sase-1d7 land agent found while verifying the epic (a fleet
  skip-path repaint regression, a test broken by the change-only runtime tick, unused
  public symbols, stale retention docs, and small correctness and dead-code cleanups),
  then close epic sase-1d7 and mark its plan file done.
size: medium
proposed_by: bbugyi200.athena.sase-1d7.land
bead: sase-1d7
create_time: 2026-09-30 20:02:22
status: done
---

- **PARENT:**
  [202609/unread_ack_reliability_and_tui_responsiveness.md](https://github.com/sase-org/sase--plans/blob/main/202609/unread_ack_reliability_and_tui_responsiveness.md)
- **BEAD:**
  [sase-1d7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1d7/README.md)

# Finish and close epic sase-1d7

## Context

Epic `sase-1d7` ("Make unread acks stick and keep the Agents TUI responsive") has all 13
phase beads closed (`sase-1d7.1` … `sase-1d7.13`, commits 279bc272d1 … 788a9311f8). Its
land agent verified every phase against the epic plan and the code at master 788a9311f8.
Phases 1–10 and 12–13 are implemented as planned. The items below are what is left, and
every one of them was caused by this epic. Follow-up triage is already finished and
recorded on the epic bead in the note that starts `LAND TRIAGE (sase-1d7.land`. Do not
re-triage, and do not create beads for anything in this plan.

Read the epic first: `sase bead read sase-1d7 -r "Need the epic scope and land notes"`.
Before touching TUI code, read `sase memory read tui.md tui_perf.md -r "<why>"`.

Decisions the land agent already made (do not reopen them):

- `read_unread_completion_index` having no production caller is intentional. The
  `sase-1d7.12` phase plan (`core_unread_ack_index.md`) scopes it to callers without a
  snapshot and to the parity test; reconciles project their in-memory snapshot instead.
- The help-modal row for `mark_all_unread_done_agents_read` is truncated to 32 columns.
  The pre-epic text was truncated the same way, so the rendered output did not change.
  Leave the text alone so PNG goldens do not churn.
- The poll path reconciles against a snapshot the monotonic cache may have rejected.
  This is low risk: the pending-ack fence still protects every in-flight ack, and the
  single-flight guarded read makes out-of-order generations very unlikely. No change.
- `test_tui_app_import_stays_under_startup_budget` (3501 vs the 3485 cap) is tracked by
  `sase-13p`. Do not raise the cap. Step 3 only removes this epic's own contribution.

## 1. Symvision: unused public symbols (epic note #1)

`just symvision` at 788a9311f8 lists three unused public symbols in
`src/sase/ace/tui/actions/agents/_unread_set_generation.py`, all from `sase-1d7.9`:

- Rename `get_unread_set_generation` → `_get_unread_set_generation`. It is only used
  inside this file.
- Rename `has_unread_probe_cache_key` → `_has_unread_probe_cache_key`. It is only used
  inside this file.
- Delete the alias `note_unread_set_changed`, which has no callers.
- Drop all three from `__all__`.

`bump_unread_set_generation`, `cached_has_unread_probe` and `unread_jump_cache_key` stay
public because other modules use them. After this, `just symvision` should list only
`sase-1dn`'s three tool symbols (`HandoffSubmitResult`, `StarterResolution`,
`owner_ref`). Those belong to `sase-1cx`, not this epic; leave them alone.

## 2. Repair the rail runtime-tick test broken by `sase-1d7.10`

`tests/ace/tui/test_agents_node_rail_wiring.py::test_runtime_tick_skipped_in_rail` has
failed since 11ba54b381. The change-only tick (`AgentList.patch_active_runtime_rows` →
`patch_runtime_suffix_row`) returns 0 when the rendered runtime text has not changed,
and the test calls `_patch_agent_runtime_rows()` at effectively the same instant it
painted. The rail behavior itself is correct. Make the test advance its tick clock:

- After `await _goto_agents(page, 4)`, monkeypatch `local_now` in
  `sase.ace.tui.actions.agents._display_panel_patches` with an advancing clock:
  `base = local_now()`, `ticks = itertools.count(1)`,
  `lambda: base + timedelta(days=next(ticks))`.
- Add a docstring sentence explaining that the tick is change-only, so each call
  advances a day.

The land agent ran exactly this change: the file passed 10/10.

## 3. Keep the epic's new modules off the TUI startup import closure

`_unread_bulk_scope` and `_unread_set_generation` (both new in this epic) load when
`sase.ace.tui.app` is imported. Defer both:

- `src/sase/ace/tui/actions/agents/_unread_state.py`:
  - Remove the top-level
    `from ._unread_set_generation import bump_unread_set_generation`.
  - Add a module-level private wrapper next to `log`:
    `def _bump_unread_set_generation(app: Any, *, removed: Any = None) -> int`. It
    imports `bump_unread_set_generation` locally and returns its result.
  - Switch the six `bump_unread_set_generation(self…)` call sites to the wrapper.
- `src/sase/ace/tui/actions/agent_workflow/_leader_mode.py`: move
  `from ..agents._unread_bulk_scope import BULK_READ_UNDO_WINDOW_SECONDS` from the top
  of the module into the `BulkUnreadToggleOutcome.MARKED_READ` branch of the
  `mark_all_unread_done_agents_read` handler.

To check, run
`python -c "import importlib,sys; importlib.import_module('sase.ace.tui.app'); print(len(sys.modules))"`.
The land agent measured 3501 → 3499, and neither module should appear in `sys.modules`.

## 4. Fleet skip path must repaint rows and banners whose display changed (`sase-1d7.11` regression)

`_try_skip_unchanged_fleet_refresh` in
`src/sase/ace/tui/actions/agents/_fleet_projection.py` copies host-level volatile fields
onto the live rows with `_patch_fleet_volatile_row_fields` and then calls only
`_update_agents_header()`.

Agent rows and group banners render several of those fields, so they now go stale until
some unrelated repaint:

- Rows: `_append_fleet_summary` in `widgets/_agent_list_render_agent.py` shows "feed
  invalid", unhealthy `fleet_connection_health`, and unhealthy `fleet_freshness`.
- Banners: `_authoritative_machine_summary` / `_host_feed_status_label` in
  `models/agent_groups/_tree_summary.py` show host counts and "stale · cached N ago".

The runtime tick's suffix-only fast path reuses the stored left text, so it does not
repaint them either. Before this phase every refresh reprojected, so a host going stale
showed immediately. The module comment above `_FLEET_VOLATILE_ROW_FIELDS` claims these
fields "stay fresh"; that is false today.

Fix it without bringing back per-poll reprojection:

1. Add a named tuple constant of the fields that are actually rendered:
   - `fleet_freshness`, `fleet_connection_health`, `fleet_host_status`,
     `fleet_host_feed_error`, `fleet_diagnostic`
   - the six `fleet_host_*_count` fields
   - `fleet_host_cache_age_seconds`, but only when the fresh row's `fleet_freshness` is
     `"stale"` (the banner shows the age only then)

   Leave out `fleet_observed_at_unix`. Live rows do not render it, and gone rows keep a
   stable value.

2. Make `_patch_fleet_volatile_row_fields` also report which live rows changed one of
   those rendered fields. Returning their indices into the live list works.
3. On the skip path, when none changed, keep today's cost: only the header patch. When
   some changed:
   - Map them to panel keys through the cached panel index
     (`self._agent_panel_index().keys_per_agent[idx]`).
   - Rebuild only those panels with `self._refresh_affected_panel_widgets(keys)`. The
     row and banner render caches already key on these fields, so only the affected rows
     and banners re-render. If that returns `False`, fall back to
     `_refresh_agents_display(list_changed=True)`.
   - Refresh the info/detail panel once when the selected row's `fleet_diagnostic`
     changed.
4. Rewrite the module comment so it says exactly what happens now.

Tests: add them to the existing split modules under
`tests/ace/tui/test_agents_fleet_refresh_laziness*.py` (the skip-path tests live in
`test_agents_fleet_refresh_laziness_projection.py`; put new ones in a new sibling module
if that file would pass about 600 lines).

- A skip-path refresh where only `fleet_observed_at_unix` / cache age changed (host
  fresh) rebuilds no panel and does not call `project_clan_tree`.
- A host going from fresh to stale on the skip path repaints that host's row summary and
  banner: the rendered text contains "stale", `project_clan_tree` is not called, and
  only that panel is rebuilt.
- Existing laziness tests stay green.

Record the idle-bench `fleet_refresh unchanged` number before and after
(`tests/perf/bench_tui_trace.py` idle scenario; see `docs/perf_runbook.md`). It must
stay near the `sase-1d7.11` result (p50 about 0.05 ms).

## 5. Retention docs still say 14 days (`sase-1d7.13`)

`sase-1d7.13` shortened live retention of dismissed notification rows to 3 days in
sase-core, but three user-facing texts still say 14:

- `docs/axe.md` (about line 515, `notification_store_compact`: "older than 14 days")
- `docs/notifications.md` (about line 1909: "a 14-day retention window")
- `src/sase/default_config.yml` (about line 1514, the `notification_store_compact` job
  description: "past the 14-day retention window")

Change each one to 3 days. Leave `docs/axe.md` about line 534 alone; it is the
artifact-directory retention, which is unrelated. Grep `docs/` and `src/` for any other
"14-day"/"14 days" text about notification retention.

## 6. Small correctness fixes

1. **`,u` off-tab perf sample never closes.** In `_leader_mode.py`, the
   `mark_all_unread_done_agents_read` handler calls `_begin_leader_perf(self, ",u")` but
   calls `_finish_leader_perf` only inside the `current_tab == "agents"` branch. Call it
   on both paths, the way the `,j`/`,J` handlers do. Add a test that an off-tab `,u`
   finishes its sample (follow the existing leader perf tests from `sase-1d7.3`; check
   `tests/ace/tui/test_unread_trace_spans.py` and the leader keymap tests for the
   pattern).
2. **Remove the legacy-signature retry in `_unread_ack_writer.py`.** In the batch
   completion helper, the `except TypeError:` branch re-calls
   `_complete_unread_notification_dismissal` without `matched_ids`/`generation`. A real
   `TypeError` raised mid-completion would then record no generation, and that op's
   pending-ack entry would never retire. First grep `tests/` to confirm no test double
   still overrides the old signature; update any that does. Then delete the branch and
   let the generic `except Exception` log it.
3. **Accurate chrome-helper counters.** In `_unread_chrome.py`, the failed-row path
   increments `panel_rebuilds` even when no rebuild ran (`panel_key is None` or no
   `_refresh_affected_panel_widgets`). Count a rebuild only when one ran. Expose the
   other case as a separate `patch_failed` counter on the `unread.chrome_apply` span.

## 7. Dead code and stale text left by the epic

1. `src/sase/ace/tui/actions/agents/_unread_navigation.py` (about line 120):
   `_repaint_changed_unread_rows` always returns `True` now, so
   `if changed and not self._repaint_changed_unread_rows(before_unread): return` cannot
   return. Make it `if changed: self._repaint_changed_unread_rows(before_unread)`.
2. `src/sase/ace/tui/actions/agents/_unread_state.py`:
   - `_has_bulk_read_undo_available` and `_bulk_read_undo_window_open` have identical
     bodies. Keep `_has_bulk_read_undo_available`, which the leader, footer and tests
     use; point the two internal callers at it; delete `_bulk_read_undo_window_open`.
   - Delete `_mark_all_unread_done_agents_read`, which has no callers left in src or
     tests now that the bulk toggle supersedes it. Grep first.
3. `_notification_snapshot_version` is write-only. The epic plan's pending-ack-fence
   phase said to reuse or replace it, and store generations replaced it. Delete:
   - its init in `actions/_state_init_agents.py`
   - its bump in `actions/agents/_notification_provider.py`
     (`_set_notification_snapshot_cache`)
   - the two test-double attributes in `tests/ace/tui/test_pending_ack_fence.py` and
     `tests/ace/tui/test_unread_ack_pipeline.py`
4. Stale comments and docstrings:
   - `_state_init_agents.py`: the coalescing-writer comment still says the queue holds
     "plus snapshot copies". It now holds `(op_id, keys, identities)` and is drained by
     one `ack_agent_completions` Rust call per batch.
   - `_unread_state.py` `_clear_agent_unread_and_dismiss_notification` docstring: the
     dismiss is now queued on the coalescing ack writer, and the indicator resyncs
     asynchronously (not "refreshed").
5. `src/sase/ace/tui/actions/agents/_unread_jump_candidates.py`: the unread predicate
   lambda duplicates `is_bulk_ack_unread_target` in `_unread_bulk_scope.py`, whose
   docstring says it also gates the jump candidates. Use
   `lambda agent: is_bulk_ack_unread_target(agent, unread_ids)` with a local import, and
   drop imports that become unused.
6. `docs/perf_runbook.md` (about lines 833–835): add `unread.chrome_apply` to the jq
   span filter.

## 8. Verification

- Run the touched suites directly first:
  - `tests/ace/tui/test_agents_node_rail_wiring.py`
  - `tests/ace/tui/test_agents_fleet_refresh_laziness*.py`
  - `tests/ace/tui/test_agent_unread_*.py`
  - `tests/ace/tui/test_unread_*.py`
  - `tests/ace/tui/test_pending_ack_fence.py`
  - `tests/ace/tui/test_agent_bulk_ack_scope_undo.py`
  - `tests/ace/tui/test_runtime_tick_caches.py`
  - `tests/ace/tui/test_agents_roster_generation.py`
  - the leader keymap tests
- Then run `sase tool run check` (never `just check-full`). Known red on master that is
  not this epic's:
  - `test_app_import_budget`: sase-13p
  - `tests/ace/tui/widgets/test_agent_header_panel.py` collection ImportError: sase-1dh
  - sase-1dn's three tool symbols in symvision
  - the parallel flakes sase-1al and sase-1du
  - nodes tracked by sase-18s and sase-1de

  Anything labeled UNKNOWN is yours.

- Step 4 changes repaint timing but should not change any static golden. If a PNG golden
  that touches fleet rows or banners fails in `just check`, run
  `just fix-tui-screenshots -- <selector>` and inspect the report before accepting
  anything.

## 9. Close out epic sase-1d7 (final step, same turn as the code)

1. Run `sase bead epic-symbols sase-1d7`. The land agent saw "No --epic-symbol entries
   for sase-1d7". Resolve or delete any entry that appears now, per the Symvision
   epic-whitelist policy in `sase memory read symvision.md -r "<why>"`.
2. Close the epic. Write the note from what you actually verified; at minimum it must
   cover:
   - all 13 phases verified against the plan and code
   - follow-up triage recorded in the LAND TRIAGE note
   - the lean-index decision
   - the fixes from steps 1–7
   - your `sase tool run check` result with any known-red items and their owning beads

   ```bash
   sase bead close sase-1d7 --note "<verification summary>"
   ```

   Never use `--force` to make the close succeed.

3. Run `just symvision` and confirm none of the remaining items belong to `sase-1d7`.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. That is the
   `PLAN` path `sase bead read sase-1d7` prints
   (`plan:202609/unread_ack_reliability_and_tui_responsiveness.md` in the plans repo);
   it is currently `status: wip`.
5. `sase-1d7` has no `parent_bead`, so the landing is complete after this. No parent
   phase or plan needs closing.
