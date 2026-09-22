---
tier: epic
title: Fix the bug and CI task beads that survived 2026-09-20 triage
goal: 'Every bug and CI task bead filed by agents on 2026-09-20 that is neither a
  duplicate nor already fixed is repaired and closed: `just check` passes on a clean
  master with no known-failure caveat, the ACE PNG corpus matches its goldens, and
  the nine product defects behind those beads (ToolRun ledger addressability and retention,
  the release core-floor smoke, doctor model advisories, the Agents view surfaces,
  the notification modal footer, gate reachability, and workspace-preparation diagnostics)
  are fixed.

  '
phases:
- id: symvision
  title: Clear the 26 unused public symbols that abort every lint run
  depends_on: []
  size: medium
  description: 'symvision: privatize, wire up, or delete the 26 unused public symbols
    reported by `just _lint-symvision` so `just check` reaches its later stages for
    every other phase.

    '
- id: latch
  title: Restore the complete-history latch reset on a changed query key
  depends_on: []
  size: medium
  description: 'latch: decide whether the keep-the-larger-cache change or the test
    expectation is wrong for an incomplete tier1 load under a changed history query
    key, then fix that side.

    '
- id: queue_weight
  title: Settle the land segment's queue weight
  depends_on: []
  size: medium
  description: 'queue_weight: decide from the documented weighted-capacity contract
    whether the rendered land segment must claim weight 2.0, then fix the renderer
    or the test accordingly.

    '
- id: import_budget
  title: Land the TUI import count strictly under its budget
  depends_on: []
  size: small
  description: 'import_budget: the app import now sits exactly at the 3290 module
    cap against a strict comparison, so cut at least one startup import or change
    the boundary semantics with a recorded rationale.

    '
- id: tmp_paths
  title: Stop eleven ACE tests asserting a full pytest tmp path
  depends_on: []
  size: medium
  description: 'tmp_paths: replace the rendered-text path assertions in the zoom file-list
    and commit-view tests with assertions that do not depend on the length of the
    pytest basetemp.

    '
- id: row_weight_golden
  title: Settle the clan-collapse agent-row label weight
  depends_on: []
  size: medium
  description: 'row_weight_golden: decide whether the tribeless DONE agent row lost
    its bold label or the golden is stale, then fix the renderer or refresh that one
    snapshot.

    '
- id: agents_view
  title: Repair the Agents view surfaces the metadata-only default left behind
  depends_on:
  - symvision
  - latch
  - import_budget
  - tmp_paths
  - row_weight_golden
  size: medium
  description: 'agents_view: dispatch the zoom modal''s LLM Calls visibility message
    and repaint the Agents header view hint on every view-picker exit path.

    '
- id: notify_footer
  title: Make the notification modal footer fit the modal
  depends_on:
  - symvision
  - latch
  - import_budget
  - tmp_paths
  - row_weight_golden
  size: medium
  description: 'notify_footer: give the notification modal''s hint line a width-aware
    tier ladder so close and +1 stay visible at 120 columns in all three hint variants.

    '
- id: toolrun_cli
  title: Disclose the run id on a failed launch and gate the floor smoke
  depends_on:
  - symvision
  size: small
  description: 'toolrun_cli: print the wrapper header for a run whose child never
    starts, and add the ToolRun smoke to the release core-floor job now that the floor
    contains it.

    '
- id: toolrun_retention
  title: Reclaim quarantined ToolRun stores
  depends_on:
  - symvision
  size: medium
  description: 'toolrun_retention: extend sase-core''s tool_run retention selection
    to quarantined corrupt stores and surface them through the existing preview and
    apply reports.

    '
- id: doctor_pools
  title: Warn on every advisory-flagged pool member
  depends_on:
  - symvision
  size: medium
  description: 'doctor_pools: expand each alias pool''s members in the model-advisory
    check so its verdict no longer depends on the round-robin cursor.

    '
- id: gate_shell_row
  title: Keep the declared shell block through gate creation
  depends_on:
  - symvision
  - queue_weight
  size: medium
  description: 'gate_shell_row: find and fix the seam that records continuation_mode
    none for a custom gate whose request declares a shell block, so its gate-shell
    row is registered and the gate stays listed.

    '
- id: gate_undismiss
  title: Make notification dismissal recoverable
  depends_on:
  - symvision
  - queue_weight
  size: medium
  description: 'gate_undismiss: add an undismiss state transition to the notification
    action surfaces so a dismissed live gate can be reached again.

    '
- id: workspace_error
  title: Surface why workspace preparation failed
  depends_on:
  - symvision
  size: medium
  description: 'workspace_error: carry the underlying git or update failure out of
    prepare_workspace into the raised error and the run log instead of discarding
    it.'
proposed_by: bbugyi200.athena.0oe
create_time: 2026-09-20 17:14:05
status: done
bead_id: sase-14n
---

- **PROMPT:** [prompts/202609/fix_triaged_bug_and_ci_beads.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/fix_triaged_bug_and_ci_beads.md)
- **BEAD:** [sase-14n](https://github.com/sase-org/sase--beads/blob/main/pages/sase-14n/README.md)

# Plan: Fix the bug and CI task beads that survived 2026-09-20 triage

## Problem

Agents on athena and apollo filed 24 `bug` and `ci` task beads on 2026-09-20. Triage on
that date verified each one against master `ec7dbbfdf` (`== origin/master`) and disposed
of nine:

- Already closed before triage: `sase-13k`, `sase-13l`, `sase-140`.
- Closed as verified fixed: `sase-13m` (contract manifest agrees with marker selection
  again), `sase-13v` and `sase-13y` (both signatures of each gone after `9a56fc129`),
  `sase-13z` (sase-telegram `tests/test_receiver.py` is 19/19 green after `f99521c`).
- Closed as duplicates, with their unique evidence carried forward as notes on the
  survivor: `sase-13q` into `sase-13o`, `sase-13r` into `sase-13p`.

Fifteen beads survived, each independently reproduced during triage at `ec7dbbfdf`. They
are unrelated defects with one shared consequence: five of them keep `just check` or the
visual corpus red for every agent working on anything else, and `just check` aborts at
the symvision stage before it ever runs a test, so the lint phase gates all the others.
Flake-typed beads are deliberately out of scope; they are tracked separately and none of
them appear here.

This epic is one phase per defect (two for the bead that carries two independent
defects), ordered so the gate-clearing work lands first.

## Conventions for every phase

- Verify your own phase with `sase tool run check` (the recorded form of `just check`).
  Do **not** run `just check-full`; this plan does not ask for it, and a `just check`
  pass with a `just check-full` failure is a test-infrastructure bug rather than phase
  work.
- Re-measure before you fix. Every phase below records what triage observed at
  `ec7dbbfdf`, and several beads' own descriptions are now stale in the ways the notes
  describe. Trust the bead's notes over its title and description.
- Each phase owns the task bead(s) named in its section and closes them with
  `sase bead close <id> --note "<what you verified>"`, quoting the command and result
  that proves the fix. The one exception is `sase-14g`, which two phases repair: neither
  phase closes it, and the land agent closes it once both phases are verified.
- Do not create task beads. Record anything out of scope as a
  `PROPOSED FOLLOW-UP: <summary — detail>` note on your own phase bead.
- Where a phase must choose between fixing production code and changing a test
  expectation, make the choice explicitly and record the reasoning in the commit message
  and the bead close note. Do not loosen an assertion, raise a budget, or regenerate a
  golden merely to turn a gate green.
- Phases that change rendered TUI output run `just fix-tui-screenshots` and inspect the
  run's report and every update group before accepting it. Generation is not approval.

## Clear the 26 unused public symbols that abort every lint run

Fixes `sase-13s`. Read `sase memory read symvision.md` first; that note owns this gate's
policy and its failures are the ones most often fixed wrongly by deleting the reported
symbol.

`just _lint-symvision` fails on clean master with "Unused public functions/classes" for
26 symbols. Triage confirmed the count and the exact list at `ec7dbbfdf`; the bead's
title says 27 and its description lists 28, because `is_gear_eligible_row` and
`service_enablement_chip` were fixed after it was filed. The live list is:

- `src/sase/sdd/_store_clone_admission.py`: `acquire_remote_clone_permit`,
  `configured_remote_clone_concurrency`, `remote_clone_lock_dir`,
  `try_remote_clone_permit`
- `src/sase/sdd/_store_clone_remote.py`: `can_retry_without_reference`,
  `clone_attempt_telemetry`, `clone_attempt_timeout`, `matching_clone_reference`,
  `remote_clone_args`
- `src/sase/ace/tui/models/_agent_runner_slot_capacity.py`: `appears_as_agent`,
  `capacity_agent_family`, `capacity_record_is_live`, `capacity_run_started_at`,
  `capacity_timestamp`, `family_shell_id`, `family_shell_kind`, `family_shell_state`,
  `parsed_artifact_path`, `participates_in_runner_slots`, `tui_hold_membership_tribes`,
  `workflow_dir_name`
- `src/sase/service/host_support.py`: `gateway_builtin_argv`, `sase_command`
- `src/sase/service/host_reporting.py`: `observation`, `write_host_status`
- `src/sase/completion/runtime_cache_generation.py`: `coherent`

These are the residue of module splits, so most are used only inside their own defining
file and should become private. Test-only references do not count as use: where a symbol
is referenced only from tests (`remote_clone_lock_dir` and `matching_clone_reference`
are the reported examples), privatize it and update the test to the private name rather
than adding a pragma. Delete a symbol only after confirming it has no caller at all. Do
not add pragmas and do not add `--epic-symbol` whitelist entries to the Justfile: this
is not an in-flight epic's symbol surface, and whitelist entries left behind are their
own recurring problem.

Done when `just _lint-symvision` is clean, `just check` passes, and no reported symbol
was silenced by a pragma or a whitelist entry.

## Restore the complete-history latch reset on a changed query key

Fixes `sase-13n`. The node
`tests/ace/tui/test_lazy_tier2_reconcile_apply.py::test_changed_query_incomplete_load_after_reconcile_rearms`
fails deterministically with `assert True is False` on
`app._agents_seen_complete_history` at line 210, after `apply_load` receives an
incomplete tier1 load whose `history_query_key` differs from the cached complete-history
key. Triage reproduced it at `ec7dbbfdf` in isolation; the bead carries three
independent +1s, a six-file bisect to `9231c9352` ("stop incomplete bounded loads from
replacing a larger cache"), and a later note confirming it on a pristine `git archive`
export of `68d9e0f655`. `9231c9352` never touched this test file.

Decide which side is wrong and say why:

- If the rule "a changed committed query cannot reuse another query's full-history
  latch" is still correct, `9231c9352`'s keep-the-larger-cache behaviour dropped it and
  the fix belongs in the loading code (`_loading_apply.py` and its merge helpers), not
  the test.
- If keeping the larger cache across a changed query key is intended, the test's
  expectation is stale and must be updated with a written justification for why a latch
  set under one query key is sound under another.

Do not loosen the assertion to accommodate both. The other ten tests in that file pass
and must keep passing.

Done when the full file passes, `just check` passes, and the commit message records
which side was wrong and why.

## Settle the land segment's queue weight

Fixes `sase-13o` (and therefore its duplicate `sase-13q`, already closed). Two nodes in
`tests/test_capacity_gate_to_admission.py` fail with `assert None == 2.0` on
`land.queue_weight`: `test_epic_gate_capacity_reaches_weighted_admission` and
`test_omitted_capacity_preserves_land_weight_and_global_budget`. Triage reproduced both
at `ec7dbbfdf`.

Triage also found the root cause, which the bead was filed without, and ruled out every
candidate it had suspected — it is not sase-core skew, not the installed plugin set, and
not user xprompt config:

- `src/sase/bead/work.py` builds the land segment and calls
  `_queue_capacity_lines(...)`, which calls `format_queue_directive(capacity=capacity)`
  and never passes a weight. So the land segment renders `%q:<capacity>` and expands to
  `queue_weight=None`, `queue_weight_explicit=False`.
- `src/sase/xprompt/queue_directive.py` already accepts and renders `weight=`, so the
  field exists and is simply never populated.
- With capacity omitted, `_queue_capacity_lines` returns no line at all, which is why
  the second node fails identically. Its name asserts the land weight survives an
  omitted capacity, so the expected weight cannot be derived from capacity.
- The prompt body cannot supply it either: `bd/land_epic` is a built-in xprompt from
  `src/sase/default_config.yml`, expanding it yields no `%queue` directive, and
  `tests/test_bead_xprompt_tags.py::test_bead_worker_builtin_xprompts_do_not_author_wait_directives`
  forbids built-in bead-worker xprompts from authoring queue or wait directives. The
  weight has to come from the renderer.

The decision to make: is a land agent contractually supposed to claim weight `2.0`
against the weighted-load budget? The assertions arrived with `14ddd4d82` ("docs(queue):
document weighted capacity and verify gate-to-admission path"), so settle it from that
commit's `docs/xprompt.md` changes and the current weighted-capacity section there.
Either the contract is real, and `_queue_capacity_lines` must emit the land weight
independently of capacity while leaving phase segments non-explicit; or `14ddd4d82`
documented an intent that was never implemented, and the test is wrong and must be
corrected with that finding recorded. Whichever side you fix, keep
`phase.queue_weight_explicit is False` and the admission behaviour the rest of the file
asserts.

Done when both nodes pass, `just check` passes, and the documented contract and the code
agree.

## Land the TUI import count strictly under its budget

Fixes `sase-13p` (and therefore its duplicate `sase-13r`, already closed). **Read this
bead's notes before touching anything: the fix it describes has already half landed and
its numbers are stale.**

The two eager `sase.service` import edges the bead blames are gone —
`src/sase/ace/tui/actions/axe_display/_data.py` no longer imports `sase.service.*` at
module level, `src/sase/ace/tui/widgets/bgcmd_list.py` imports its service types under
`TYPE_CHECKING` only, and the test's `deferred_modules` assertion passes. What remains
is a boundary saturation, not the reported two-module overrun: at `ec7dbbfdf`,
`tests/ace/tui/test_app_import_budget.py::test_tui_app_import_stays_under_startup_budget`
fails `assert 3290 < 3290` against `_MAX_MODULE_COUNT = 3290` and a strict comparison.

Re-measure first, then choose one of two remedies and record why:

- Cut at least one module from the `sase.ace.tui.app` import closure so the count lands
  strictly under the cap. This is the preferred direction and is what closed `sase-136`
  intended by saying not to raise the cap.
- Or change the guard's boundary to `<=` with a written rationale for why exactly-at-cap
  is an acceptable steady state, keeping the cap value itself unchanged.

Raising `_MAX_MODULE_COUNT` is not an option. Keep the existing `deferred_modules`
assertion and the CPU-time check that `sase-136` introduced intact.

Done when the node passes, `just check` passes, and the choice is justified in the
commit message.

## Stop eleven ACE tests asserting a full pytest tmp path

Fixes `sase-13u`. Eleven nodes assert that a full pytest tmp path appears in rendered
panel text, and the renderer truncates long paths with an ellipsis, so they fail in
every SASE agent workspace (whose pytest basetemp is roughly 140 characters before the
file name) and pass under a short basetemp. Triage confirmed both halves at `ec7dbbfdf`:
11 failed under the agent basetemp, and the same selection is 20 passed with
`--basetemp=/tmp/zc`.

The nodes are the ten file-navigation tests in
`tests/ace/tui/test_agents_zoom_panel_files.py` (first failure at line 74,
`assert str(first_path) in rendered`) plus
`tests/ace/tui/modals/test_commit_view_modal.py::test_commit_view_modal_toggles_plan_and_restores_cached_diff`
(line 527, a title line that ends in `…` mid-path).

The tests are right about the product behaviour; only their expectation depends on path
length. Follow the precedent of closed `sase-rv`, which fixed this class in
`tests/main/test_skills_handler.py` by asserting on structured data instead of rendered
text: assert on the panel or model state that names the file, or on a stable suffix such
as the basename, or give the fixture a short tmp path. Do not shorten or disable the
renderer's truncation to make tests pass, and do not weaken the assertions to the point
that they would pass while the wrong file is displayed.

Done when all 11 nodes pass under the default agent basetemp, still pass under
`--basetemp=/tmp/zc`, and `just check` passes.

## Settle the clan-collapse agent-row label weight

Fixes `sase-14k`.
`tests/ace/tui/visual/test_ace_png_snapshots_agents_panel_clan_collapse.py::test_selected_panel_clan_collapse_precedes_status_group_png_snapshot`
drifts from
`tests/ace/tui/visual/snapshots/png/agents_selected_panel_clan_collapse_120x40.png` on a
clean tree. Triage reproduced it independently at `ec7dbbfdf`: a targeted
`just test-visual` run on that file reports `created=0 updated=1 unchanged=0 stale=0`.

The difference is semantic, not renderer noise: the agent row for the tribeless DONE
fixture agent named `home` is bold in the golden and regular weight in the current
render, 354 pixels confined to that label's bounding box, every other pixel identical.
The bead's lead, not yet bisected, is that the golden was last refreshed by `8c83b8f0e`
and `7442af7af` ("insert arriving agent rows in place and settle the column in one
frame") landed after it and touches agent-row rendering.

Bisect that lead, then decide: either the row-name weight regressed and the renderer is
wrong, or the weight change was intended and this one golden is stale. Only if the
change was intended may you refresh the snapshot, and then only through
`just fix-tui-screenshots` with the report and the update group inspected. This phase's
scope is this node; the older broad drift tracked by `sase-x5` and `sase-10u` is not
part of it, and closed `sase-ce` refreshed this same golden among six others, so treat a
second unexplained drift here as evidence for the renderer side.

Done when the targeted `just test-visual` run reports no drift for this node,
`just check` passes, and the commit message says which side was wrong.

## Repair the Agents view surfaces the metadata-only default left behind

Fixes `sase-149` and `sase-14a`. Both were proposed by phase `sase-12z.5.2` and left for
later because each changes rendered output; triage confirmed both still present at
`ec7dbbfdf`.

`sase-149`: `src/sase/ace/tui/modals/zoom_panel_modal.py` defines a bare
`on_llm_calls_visibility_changed` (line 345) that delegates to the shared helper.
Textual derives the handler name from the message class, and `LLMCallsVisibilityChanged`
collapses to `on_llmcalls_visibility_changed`, so the method never dispatches and the
zoom modal never learns the LLM Calls panel has content. `9a56fc129` fixed the identical
latent bug on `AgentDetailPanelMixin` in
`src/sase/ace/tui/widgets/_agent_detail_panels.py` by adding
`@on(LLMCallsVisibilityChanged)` above the same-named method, with a regression test at
`tests/ace/tui/test_agent_view_picker.py::test_agents_llm_calls_panel_message_enables_picker_layouts`.
Decide whether the zoom modal needs the same wiring or whether the handler is genuinely
dead and should be deleted. If you wire it up, mirror that regression test for the modal
and refresh any zoom goldens it moves.

`sase-14a`: the Agents header `[view: ... (p)]` hint is not repainted when the view
picker closes without a change. In
`src/sase/ace/tui/actions/agents/_agent_view_picker.py`,
`_refresh_agent_view_surfaces()` runs only on the changed branches —
`_apply_agent_view_mode_result` returns early when `set_panel_mode` reports no change,
`_apply_agent_layout_result` returns early when the chosen layout is already current,
and an Esc dismissal reaches neither. A countdown tick that fires while the modal is
open drops the hint entirely until the next tick. Refresh the info panel from the
picker's dismiss callback so every exit path repaints, then remove the test-side
workaround in `tests/ace/tui/visual/_ace_agents_png_snapshot_helpers.py` where
`choose_agent_metadata_view` refreshes the info panel itself, and confirm the visual
helpers still produce stable frames without it.

Done when both beads' reproductions no longer reproduce, the new dispatch regression
test exists (or the handler's deletion is justified), the visual helper workaround is
gone, any moved goldens are refreshed with the report inspected, and `just check`
passes.

## Make the notification modal footer fit the modal

Fixes `sase-13j`. The Notifications modal's hint line is a single `Label` with
`height: 1` and `text-align: center` (`NotificationModal #notification-hints` in
`src/sase/ace/tui/styles.tcss`) that auto-sizes to its content, so nothing wraps,
reflows, or sheds entries and everything past the modal's edge is clipped. Triage
measured the three variants at `ec7dbbfdf` against roughly 108 visible cells at 120
columns: `DEFAULT_HINT_TEXT` 208, `QUESTION_HINT_TEXT` 138, `GATE_HINT_TEXT` 153 — each
_wider_ than when the bead was filed (205/135/150), so the drift is still growing. Users
lose `q: close` and `+: +1` entirely and cannot discover tab navigation from the footer.

This is a footer design change, not a wording tweak: the bead records that three agents
in a row tried to shorten their own fragment and the line stayed clipped, because the
entries that fall off predate any of those fragments. Implement a width-aware tier
ladder:

- Model the footer as an ordered list of `(key, label)` fragments, each with a priority,
  instead of three hard-coded strings. `q: close` and `+: +1` are the highest priority
  and must survive every tier; low-frequency entries shed first.
- Choose the tier from the measured available width at render time and on resize, and
  build the text in `src/sase/ace/tui/modals/notification_modal_options.py` (which
  selects the variant around lines 275-289) from the constants in
  `src/sase/ace/tui/modals/notification_modal_constants.py`.
- Cover all three variants — `DEFAULT_HINT_TEXT`, `QUESTION_HINT_TEXT`,
  `GATE_HINT_TEXT`. A second hint row, or moving shed keys into the help modal, is
  acceptable as the lowest tier provided the surviving line always fits and `q: close`
  is always present.
- Add a test that pins the invariant by measurement, not by string equality: at 120
  columns each rendered variant's cell width must not exceed the modal's content width,
  and the close and +1 hints must be present.

Refresh the twelve `notification_*_120x40.png` goldens this moves, inspecting the report
and each update group.

Done when the measured width of every variant fits at 120 columns, the invariant test
exists, the goldens are refreshed deliberately, and `just check` passes.

## Disclose the run id on a failed launch and gate the floor smoke

Fixes `sase-141` and `sase-147`. Both are `sase-135` follow-ups, both small, and neither
touches the other's files.

`sase-141`: a `sase tool run` whose child never starts still records a settled ToolRun,
but the wrapper never prints its `sase tool run <run-id>` header, so the run is
unaddressable. Triage reproduced it at `ec7dbbfdf`:
`sase tool run -- /nonexistent/binary_xyz` prints only
`executable not found: /nonexistent/binary_xyz`, while the ledger gained a corresponding
row. Print the header before attempting the spawn, or on the spawn-failure path, so
every recorded run id reaches the user exactly once. The epic's guarantees must survive:
the child's own stdout and stderr and its exit status stay byte-exact and unchanged (127
for a missing executable, 126 for a non-executable file), and the header stays on
stderr. The relevant code is `src/sase/tool/executor.py`'s spawn-failure path (its
exit-code mapping is around lines 675-682) and the header in `src/sase/tool/render.py`.
Add a regression test for both the 127 and the 126 case that asserts the id is printed
once and is resolvable with `sase tool show`.

`sase-147`: step 1 of this bead is **already done** — `pyproject.toml` now declares
`sase-core-rs>=0.34.70,<0.35.0`, comfortably past the `0.34.63` floor that first
contains the 14 `tool_run_*` bindings, so the release workflow's ratchet landed it. Only
step 2 remains: add `tools/smoke_sase_core_rs_tool_runs` (which exists, and runs locally
through `just smoke-tool-runs`) to the release-core-floor smoke list in
`.github/workflows/ci.yml`, alongside the six sibling smokes at roughly lines 520-525.
The ordering constraint the bead sets out is satisfied by the floor already being
raised; confirm the declared floor in `pyproject.toml` before adding the smoke rather
than assuming it.

Done when the failed-launch header is printed and tested for both exit codes, the smoke
is in the release-core-floor list, `just check` passes, and both beads are closed.

## Reclaim quarantined ToolRun stores

Fixes `sase-143`. When the ToolRun store detects a corrupt SQLite database it renames it
to `runs.sqlite.corrupt-*` under `sase_home()/tools/` and starts fresh, and nothing ever
removes those files, so every corruption event permanently leaks a full copy of the
ledger. Triage confirmed the gap at the linked sase-core checkout: quarantine exists
(`quarantine_corrupt_store` and `corrupt_store_quarantine_path` in
`crates/sase_core/src/tool_run/store.rs`), but the retention path — `retention_preview`,
`retention_apply`, and the `retention` helper in the same file — selects only retained
log files, event files, and projection rows, and never mentions quarantined stores.

Per the Rust core backend boundary, the selection logic belongs in sase-core. Open that
repo with your `/sase_repo` skill and extend `crates/sase_core/src/tool_run/` so
quarantined stores are selected under an explicit horizon and surfaced through the
existing preview/apply retention report, keeping the honest physical-byte accounting the
epic requires. `src/sase/core/disk_footprint_reap_tool_run.py` stays a thin adapter, and
`src/sase/core/disk_footprint_inventory.py` keeps owning `tool_run_retention`. Add Rust
tests beside the existing `corrupt_store_is_quarantined_on_write_and_recreated` and
`retention_protects_unsettled_runs` cases, and a Python test that
`sase disk reap --owner tool_run_retention` previews and then reclaims a quarantined
file.

Both repositories are part of this phase's commit obligation. If the fix needs a
sase-core release and a floor raise before the Python side can rely on it, say so in
your phase note rather than pinning a floor that has no containing release.

Done when a quarantined store is previewed and reclaimed at an explicit horizon,
sase-core and sase tests cover it, and `just check` passes.

## Warn on every advisory-flagged pool member

Fixes `sase-14e`. `sase doctor -C llm.model_advisory` is the only consent surface for a
route to a model that trains on its inputs and outputs, and it is silent most of the
time. `_resolved_routes()` in `src/sase/doctor/checks_providers_advisory.py` builds its
routes from `build_alias_views(overrides={})` and reads one `view.model` per alias — the
pool's _currently selected_ member, which depends on the machine-global round-robin
cursor in `~/.sase/llm_lb.json` — and never expands the pool. Triage confirmed the code
is unchanged at `ec7dbbfdf`. The bead's repro shows the same tree and config yielding
`OK` on a fresh cursor and `WARN` after three consumes.

For every alias, parse the target with `parse_model_alias_selector` (in
`src/sase/llm_provider/load_balancing.py`, which also declares `members` and
`fallback_members` on `ModelAliasSelector`) and check every member, resolving each to
`(provider, model)`, so the verdict is deterministic and cursor-independent. Keep the
per-`(source, model)` de-duplication. Do not change what else counts as a route (tier
mappings, launch-default settings) and do not suppress a finding. Update the "Model
Advisories" sentence in `docs/llms.md` that says the check warns whenever one of those
pools _currently selects_ the model. Add a regression test that pins the cursor away
from the advisory-flagged member and still expects `WARN`.

Done when the check warns independently of cursor state, the docs sentence matches the
new behaviour, the regression test pins it, and `just check` passes.

## Keep the declared shell block through gate creation

The first of the two defects behind `sase-14g`; do not close that bead in this phase.

A custom gate declared a full shell block in its `request.json`
(`PYPI-DELETE -> PYPI-DELETED`, `next.fork=family`), but `.creation_result.json`
recorded `continuation_mode="none"` and `sase gate list --all` returned 103 rows without
it, so the shell block was accepted at request time and dropped at creation time. The
gate stayed live and answerable (`acceptance.disposition=accepted_failed`,
`can_supersede=true`) yet invisible, which in the reporting case stalled an irreversible
human-gated PyPI deletion.

Triage narrowed where to look without settling the fix. `GateSpec` does carry the block:
`src/sase/notification_gates/model_request.py` declares `shell: GateShellSpec | None`
and parses it, and `continuation_mode` defaults to the literal `"none"` when the request
omits it (around line 217). `src/sase/notification_gates/service.py` then writes
`spec.continuation_mode` straight through into the creation result and the envelope
(around lines 333, 450, and 519). So the likely gap is that nothing derives or validates
`continuation_mode` from the presence of a shell block for the custom-gate path, the way
the built-in gate kinds set theirs explicitly (`gate_shell` in `src/sase/sudo/gate.py`,
and the per-kind constants used by the plan, question, task-triage, and flag gates).
Confirm that before changing anything.

Fix it so a request that declares a shell block cannot be created with a continuation
mode that discards it: either derive the mode from the block, or reject the inconsistent
pair at validation time with a clear error. A silent drop is the bug. Add a test that
creates a custom gate with a shell block and asserts both the recorded continuation mode
and the presence of a gate-shell row for it.

Done when a shell-block custom gate registers its gate-shell row and appears in
`sase gate list --all`, the inconsistent combination can no longer be created silently,
and `just check` passes.

## Make notification dismissal recoverable

The second of the two defects behind `sase-14g`; do not close that bead in this phase.
This phase is independent of the gate-creation fix, and either fix alone leaves a
reachable gate, which is why the bead treats them as one problem.

Dismissal is a one-way door. Triage confirmed at `ec7dbbfdf` that
`sase notify apply-state` accepts exactly `{dismiss, mute, read, snooze, unmute}` with
no undismiss, and that no `undismiss` symbol exists anywhere in `src/sase`. In the
reporting case both of the gate's notifications had been dismissed, so the live gate was
unreachable from the notification panel as well as the gate list, leaving
`sase gate answer <id>` from the CLI as the only route in — and only for someone who
already knew the id.

Add an undismiss transition and expose it everywhere dismissal is offered: the
`sase notify apply-state` action set, the notification store state transition beside the
existing dismissal handling in `src/sase/notification_gates/dismissal.py`, and the ACE
notification surfaces that offer dismiss today. Read `sase memory read cli_rules.md`
before adding the CLI action, and update `src/sase/default_config.yml` if you add or
change a keymap. Cover it with tests that a dismissed notification can be restored and
that a restored gate notification is reachable again.

Done when a dismissed notification can be undismissed from the CLI and the TUI, the
round trip is tested, and `just check` passes.

## Surface why workspace preparation failed

Fixes `sase-14m`. `prepare_workspace_if_needed` in
`src/sase/axe/run_agent_runner_setup.py` turns every preparation failure into a bare
`RuntimeError("Failed to prepare workspace")`, and the helper it checks,
`prepare_workspace(...)` in `src/sase/axe/runner_workspace_prepare.py`, returns a bare
`bool`. Triage confirmed both at `ec7dbbfdf`. The underlying git or update error is
never surfaced, logged, or attached to the failed run, so the operator gets a traceback
naming only the guard that raised it. In the reporting case one of three soak probes was
lost with no recoverable cause, and its run log ends at
`Updating workspace to origin/master...`. Secondary cost: the failed run holds its
workspace until the agent is dismissed in the TUI, so a slot leaves the pool with no
explanation.

Make `prepare_workspace` report why it failed — raise a typed error, or return a result
carrying the failing command, its exit status, and its stderr — and include that reason
both in the raised `RuntimeError` message and in the run log, so the next occurrence is
diagnosable from the artifact alone. Keep the existing callers working: the helper has
more than one, so change them all rather than leaving a second bare-bool path behind.
Add a test that a preparation failure propagates the underlying error text.

Before you start, read the `wip` tale plan
`202609/workspace_origin_and_push_failure_diagnosis.md`. It covers adjacent ground —
stale workspace-clone origins and a false "dirty work vanished" finalizer diagnosis
whose real cause was a rejected push — and if part of this reason-carrying work already
exists there, integrate with it instead of building a second mechanism. It is not a
duplicate of this bead: that plan is about the finalizer and clone origins, this is
about discarding the preparation error.

Done when a failed preparation names the underlying git or update error in both the
exception and the run log, a test pins it, and `just check` passes.
