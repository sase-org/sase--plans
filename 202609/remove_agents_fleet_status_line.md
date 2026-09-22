---
tier: tale
title: Remove the Agents tab fleet status line
goal:
  The Agents tab no longer shows the routine here-machine, active-count, machine-count
  status line; the row only appears when there is an actionable fleet problem.
size: small
proposed_by: bbugyi200.apollo.1h.f0.f0.f0
create_time: 2026-09-22 08:24:53
status: wip
---

# Remove the `here: athena · 2 active · 1 machine` fleet status line from the Agents tab

## Goal

On machines that have remote (federated) machines configured, the Agents tab shows a
right-aligned, dim, one-row status line directly under the agent-info / launch-context
row (below `model: opus@high · project: +sase`), e.g.:

    here: athena · 2 active · 1 machine

The user finds this line not useful and wants it gone. After this change the Agents tab
no longer renders this row in the normal (healthy) case, freeing one vertical row for
the agent list/detail panes.

## Where it lives today

- `src/sase/ace/tui/_app_layout.py` (~line 119): the Agents view composes
  `Horizontal(id="agents-header", classes="hidden")` containing
  `Static("", id="agents-fleet-status")`, between `AgentsFilterBar` and
  `#agents-content`.
- `src/sase/ace/tui/actions/agents/_fleet_header.py`: `AgentFleetHeaderMixin` with
  `_update_agents_header()` (un-hides the row whenever fleet mode is available or
  loading) and `_unified_agents_status_text()` (builds
  `here: <machine> · N active · N needs you · N machine(s) · <diagnostics> · <host feed issues>`,
  or `here: … · loading machines...`, or `here: … · <error>`).
- `src/sase/ace/tui/styles.tcss` (~lines 3722-3736 and 3772): `#agents-header`,
  `#agents-fleet-status`, and the `-onboarding-active` hide rule.
- Callers of `_update_agents_header()`: `_fleet_refresh.py` (several),
  `_fleet_projection.py` (2), `_display.py` (2, via `getattr`), `_app_watchers.py` (1,
  via `getattr`), plus test harness stubs.
- Helpers in `src/sase/ace/tui/actions/agents/_fleet_common.py` used only by the header:
  `local_machine_label`, `agent_counts_as_active`, `unified_attention_count` (and the
  private `_agent_status_key`, `_LOCAL_ACTIVE_STATUSES`, `_LOCAL_ATTENTION_STATUSES`),
  re-exported as underscore aliases from `_fleet.py`. `unified_diagnostic_text` and
  `host_feed_issue_text` are also header-only.

## Design decision: keep a problems-only row

The header is currently the only place some fleet failures surface — notably
`host_feed_issue_text` is documented as "the only place [an invalid federation host's]
feed error is guaranteed to surface", and fleet config/refresh errors
(`_agents_fleet_last_error`, e.g. `fleet config unavailable`) and stale/unknown host
diagnostics (`mac unknown`) also show only here. Deleting the row outright would make
those failures silent.

So: **remove all routine content** (`here: <machine>`, `N active`, `N needs you`,
`N machine(s)`, `partial`, and the transient `loading machines...`), and **only show the
row when there is an actionable problem**. In the healthy case (the user's screenshot)
the row is hidden entirely, which is exactly what the user asked for.

Concretely, replace `_unified_agents_status_text()` with a problems-only builder (keep
the method name or rename to e.g. `_agents_fleet_problem_text()`, implementer's choice;
update all references consistently) that returns a `·`-joined string of, in order:

1. `_agents_fleet_last_error` if set (and nothing else — mirrors today's early return),
2. otherwise `unified_diagnostic_text(projection)` if non-empty,
3. `host_feed_issue_text(projection)` if non-empty,

and returns `""` when none apply. Drop the `projection.partial` → `"partial"` fallback
(it is not actionable on its own) and drop the loading state entirely.

`_update_agents_header()` then becomes: query the two widgets (keep the early `return`
on query failure); compute the problem text only when fleet mode is available (keep the
`_fleet_mode_available()` guard); if the text is empty, add the `hidden` class and
return; otherwise remove `hidden` and `status.update(text)`. Keep the method name and
all its callers unchanged so no call site churns.

Keep the widget ids, CSS, and layout as-is (the row still needs to exist for the
problems case; it starts `hidden` so nothing changes visually when healthy).

## Dead-code cleanup

After the change these become unused — delete them rather than leave dead code
(Symvision flags unused symbols):

- `_fleet_common.py`: `local_machine_label`, `agent_counts_as_active`,
  `unified_attention_count`, `_agent_status_key`, `_LOCAL_ACTIVE_STATUSES`,
  `_LOCAL_ATTENTION_STATUSES`; remove them from `__all__`. Remove the now-unused
  `get_machine_name` import / `log` / `fleet_public_override` / `sys` / `Mapping` /
  `Agent` imports only if they truly become unused (verify with grep;
  `fleet_public_override` and `Mapping` are likely still used by other helpers — check
  before deleting).
- `_fleet.py`: remove the `_agent_counts_as_active`, `_local_machine_label`,
  `_unified_attention_count` aliases, their imports, and their `__all__` entries. Keep
  `_unified_diagnostic_text` only if still referenced anywhere; otherwise remove it too.
  Keep the `get_machine_name` re-export if anything still uses/monkeypatches it via
  `fleet_mod`; after removing the tests below, grep and delete it if unused.
- `_fleet_header.py`: drop now-unused imports (`FleetRowsProjection` default is still
  needed for the projection lookup; keep what is used).

Do NOT touch `src/sase/ace/tui/modals/machines_pane_rendering.py::local_machine_label` —
that is a separate function used by the Machines pane.

Run `just fix` / Symvision (per the `symvision` reference memory if it complains) to
confirm nothing is left dangling.

## Tests

- `tests/ace/tui/test_agents_fleet_refresh_laziness.py`:
  - Replace `test_unified_agents_status_text_labels_scope_and_staleness` with a test of
    the problems-only text: with the same fixture (2 configured hosts, a `mac`
    `host_stale` diagnostic, 2 active / 2 attention agents) the text is exactly
    `"mac unknown"` (no `here:`, no counts, no `machines`). Add a case that a healthy
    projection (no diagnostics, no host feed issues, no error) yields `""`.
  - The existing assertion at ~line 346 that
    `"apollo: feed invalid: invalid_envelope (cached 5h ago)"` is in the text must keep
    passing (update the method name if renamed).
  - Remove the `get_machine_name` monkeypatch if it is no longer meaningful.
  - The harness stub `_update_agents_header` (~line 58) and the one in
    `tests/ace/tui/test_fleet_agents_projection_family_tree.py` (~line 347) stay as-is
    (method name unchanged).
- Add a small unit test for `_update_agents_header()` hide/show behavior if cheap
  (healthy → `#agents-header` has `hidden`; error → visible with the error text); the
  visual test below covers it end-to-end, so this is optional.
- `tests/ace/tui/test_fleet_agents_feed_honesty.py`: `host_feed_issue_text` is kept, no
  change expected.
- `tests/ace/tui/visual/test_ace_png_snapshots_agents_fleet.py`
  (`test_agents_fleet_state_strip_png_snapshots`, ~lines 270-318):
  - Zero-results scene: replace `assert_page_svg_contains(page, "here: athena")` /
    `"0 active"` with an assertion that the header row is hidden (e.g.
    `page.query_one_widget("#agents-header").has_class("hidden")`), and that the SVG
    does NOT contain `here: athena` (use an existing "not contains" helper if one
    exists, otherwise assert on the widget state only).
  - Loading scene: remove the `"loading machines"` assertion; either delete the
    `agents_fleet_loading_120x40` golden scene and its PNG (it would now be identical to
    the zero-results scene), or keep it asserting the header is hidden. Prefer deleting
    the scene + its golden file under `tests/ace/tui/visual/snapshots/png/` to avoid a
    redundant golden; if the snapshot inventory / manifest lists scene names, update it
    accordingly.
  - Unavailable scene: `"fleet config unavailable"` must still be visible (now without
    the `here: athena · ` prefix) — keep that assertion.
  - Remove the `get_machine_name` monkeypatch at ~line 54 if no remaining scene needs
    it.

## Screenshots

Several Agents-tab goldens with fleet configured will change (the row disappears and
panes gain one row; the error scene loses the `here: athena · ` prefix). As the final
step after code and tests are green, run `just fix-tui-screenshots` (full inventory, no
selectors) so PNG goldens match, then spot-check by viewing
`agents_fleet_loaded_zero_results_120x40.png` (no status line) and
`agents_fleet_unavailable_120x40.png` (row shows only `fleet config unavailable`). If a
single known-flaky golden (e.g. `selected_gate_shell_output`) fails to converge after a
couple of solo retries, leave it and mention it in the summary.

## Verification

1. Targeted:
   `pytest tests/ace/tui/test_agents_fleet_refresh_laziness.py tests/ace/tui/test_fleet_agents_feed_honesty.py tests/ace/tui/test_fleet_agents_projection_family_tree.py tests/ace/tui/test_fleet_setup_guidance.py tests/ace/tui/visual/test_ace_png_snapshots_agents_fleet.py`.
2. `grep -rn "here: \|_unified_agents_status_text\|local_machine_label\|agent_counts_as_active\|unified_attention_count" src/sase/ace/tui/actions tests/ace/tui`
   shows no stale references (the machines-pane `local_machine_label` is expected).
3. `just fix`, then `just check` (known pre-existing unrelated failures such as the
   `test_agent_cleanup_panel_clan_members_e2e.py` J-focus timeouts may appear; confirm
   they are unrelated rather than chasing them).
4. `just fix-tui-screenshots` last, then spot-check the PNGs above.

## Out of scope

- The Machines pane, BY_MACHINE banners, fleet refresh logic, and the launch-context
  cluster are unchanged.
- No keymap/config changes (`default_config.yml` untouched).
- No Rust core changes: this is presentation-only TUI state.
