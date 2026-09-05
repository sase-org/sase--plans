---
tier: tale
title: Restore count-only presentation for starting agents
size: small
goal:
  Keep every STARTING agent hidden from Agents-tab rows while preserving the starting
  count and normal status transitions.
proposed_by: bbugyi200.athena.0gf
create_time: 2026-09-05 17:49:19
status: wip
---

# Restore count-only presentation for starting agents

## Scope and tier

This is a small tale: one coding agent can correct two presentation-model modules and
extend the existing regression suites. The root cause is confirmed and does not require
separate implementation phases.

The user requires starting agents to appear only in the `<N> starting` status count,
never as Agents-tab nodes. This is an unconditional correction to the requested UI
contract; there is no compatibility mode or feature flag to introduce.

## Diagnosis and evidence

The supplied screenshot, `.sase/artifacts/pool/6bdee587904c-file-ref.png`, shows a
selectable `[agent] sase (STARTING)` row under a `Starting` status-group banner. Its
detail panel shows an unassigned name, workflow `ace(run)-260905_173207`, START time
`2026-09-05 17:32:07`, and no prompt file. Those observations are consistent with a
workspace-claim placeholder. They do not establish why that specific launch had not yet
acquired richer metadata.

The visibility defect is independently reproducible without loading or mutating live
agent records:

- `src/sase/ace/tui/models/agent_panels.py` defines
  `STARTING_ROW_HIDE_GRACE_SECONDS = 120.0`.
- `_agent_is_starting()` returns true only for a `STARTING` agent with a known
  `start_time` less than 120 seconds old. Despite its name, it tests temporary hiding
  eligibility, not just status.
- `agent_is_rendered_in_agents_panel()` negates that result. Therefore an old `STARTING`
  row, or one with no start time, is explicitly allowed to render.
- `build_agent_panel_index()` in `models/agent_panel_index.py` uses that predicate to
  move these rows out of `hidden_starting_indices` and into selectable panel slices.
  `_agent_info_metrics()` in `actions/agents/_display_detail_info.py` derives the
  starting count from the hidden indices, so the visible row also disappears from that
  count.
- `AgentList.update_list()` uses the same predicate before grouping. This explains both
  the node and the `Starting` banner in the screenshot. Selection, neighbor navigation,
  completion fallbacks, and display-diff helpers also share it.

A read-only probe of the current model, passing a fixed `now` into the predicate and
index builder, produced:

| STARTING age       | Rendered | Selectable total | Starting count | Headline total |
| ------------------ | -------- | ---------------- | -------------- | -------------- |
| 30 seconds         | false    | 0                | 1              | 1              |
| 119 seconds        | false    | 0                | 1              | 1              |
| 120 seconds        | true     | 1                | 0              | 1              |
| 360 seconds        | true     | 1                | 0              | 1              |
| Missing start time | true     | 1                | 0              | 1              |

The exception was introduced in commit `0083d1e10` on August 13, 2026,
`fix(ace): stop workspace-claim placeholder STARTING rows from becoming phantoms`. The
audited predecessor plan `plan:202608/phantom_starting_agent_rows.md`, design item 4,
deliberately requested this timeout as a way to expose stale claims. The existing test
`test_starting_row_past_grace_window_renders_instead_of_hiding` in
`tests/ace/tui/models/test_agent_panel_index.py` explicitly enforces it. Other hiding
tests generally use fresh timestamps, so they do not reject the exception.

That earlier change also corrected monitor artifact resolution and placeholder status
downgrades. Those loader fixes are separate from the visibility exception and must
remain intact.

## Implementation

1. Restore status-only visibility in `src/sase/ace/tui/models/agent_panels.py`: an agent
   with `status == "STARTING"` is always excluded from rendered Agents-tab rows,
   irrespective of age, missing timestamp, agent name, or metadata availability. Keep
   one shared predicate used by the existing callers. Remove the grace constant,
   elapsed-time calculation, clock-only imports, and obsolete `now` argument. Simplify
   or remove the private helper if it becomes redundant. Update the docstring to state
   the count-only contract explicitly.

2. Remove the corresponding clock plumbing from
   `src/sase/ace/tui/models/agent_panel_index.py`, including its `now` argument,
   reference-time calculation, and grace-window documentation. Update its test callers
   and retire `_WITHIN_GRACE_WINDOW`. Repository search found those explicit
   index-builder `now` calls only in `test_agent_panel_index.py`; recheck callers before
   editing. Other `now` parameters, such as date grouping in `AgentList.update_list()`,
   serve different behavior and remain necessary.

3. Preserve the existing division between loaded data and rendered rows: starting agents
   stay in the loaded agent list and keep their panel-key identities. They are excluded
   from panel slices and selectable positions, while hidden top-level agents contribute
   once each to `hidden_starting_indices`, the starting chip, and the overall agent
   headline. Existing workflow-child and family/clan counting rules continue to prevent
   double-counting. Preserve `include_hidden=True` for callers explicitly requesting
   hidden data.

4. Keep using the existing refresh and selection paths. An actual status change to
   `RUNNING`, `WAITING`, `QUEUED`, `QUESTION`, or a terminal status makes an otherwise
   eligible row available normally and removes its starting count.
   `_poll_starting_agent_transitions()` already consumes hidden starting indices; ensure
   old starting records remain eligible for this existing marker poll. A timer must
   never promote visibility while the status remains `STARTING`. No new polling,
   synchronous I/O, or cache-expiration scheme is needed.

These are Textual presentation and selection rules, so they belong in this repo. Do not
move status classification, metadata enrichment, claim cleanup, or other shared backend
behavior into Python to solve this display problem.

## Regression coverage and acceptance criteria

Extend the nearby suites rather than constructing a new live-launch test harness. Use
deterministic fixtures and no sleeps or real agent launches.

1. **All starting timestamps stay hidden.** Replace the existing timeout-exposes-row
   test with the opposite contract. Cover recent timestamps, the former two-minute
   boundary, substantially older timestamps, missing timestamps, and future timestamps.
   Exercise both the visibility predicate and panel-index output. Non-STARTING controls
   remain eligible with old or missing timestamps. Primary suites:
   `tests/ace/tui/models/test_agent_panel_index.py` and
   `tests/ace/tui/models/test_agent_panels.py`.

2. **Counts remain coherent.** Extend
   `tests/ace/tui/test_agent_panel_index_integration.py` with old and missing-time
   cases. A lone starting agent produces zero selectable rows, starting count 1, and
   headline total 1. With one visible running agent it produces one selectable row,
   starting count 1, and headline total 2. Hidden workflow children do not inflate
   top-level totals. Split and merged tribe panels both exclude starting rows; a tribe
   containing only starting rows does not gain a rendered row or status banner. Preserve
   the established empty-state behavior.

3. **The screenshot cannot recur.** Parameterize the existing hiding tests in
   `tests/ace/tui/widgets/test_agent_list_grouping_buckets.py` with stale and
   missing-time placeholders. Assert that standard and BY_STATUS modes render neither
   the agent node nor a `Starting` banner, while ordinary rows remain. Use the
   shared-filter coverage to check the remaining grouping modes where practical. Retain
   low-level STARTING status-formatting tests: formatting a status in isolation does not
   authorize displaying its row.

4. **Hidden rows are unreachable.** Extend the existing focus-snap test in the
   panel-index integration suite and appropriate navigation/completion tests with an old
   starting agent. Verify no selection, jump/neighbor target, or visible completion
   candidate can reach that row. Relevant suites include
   `tests/ace/tui/test_agent_neighbor_navigation_targets.py` and
   `tests/ace/tui/test_agent_completion_visibility.py`.

5. **Real transitions still reveal rows.** Replace the loaded list through the normal
   refresh invalidation path with a same-identity agent whose status has changed from
   old `STARTING` to `RUNNING` or `WAITING`. Assert it appears once, its starting count
   becomes zero, the headline total stays constant, and selection remains on a valid
   visible row. Keep the existing marker-poll tests in
   `tests/ace/tui/test_starting_agent_poll.py` passing; add one real-index case if
   necessary to prove that an old starting record remains a polling candidate.

The production change should normally be confined to the two model modules. Change
callers only where signature cleanup or a demonstrated regression requires it. Do not
broadly rewrite timestamp fixtures or unrelated presentation logic.

## Verification

- Read `lint_and_test.md` and `tui_perf.md` through `sase memory read` before
  implementation. Prepare the coding workspace with `just install` if needed. The
  initial planning environment lacked a usable Rust extension when resolving configured
  current time; the successful reproduction used a fixed datetime and exercised the real
  visibility/index code. An attempted planning `just check` installed a cached
  extension, then entered a release build of the Rust LSP. That check was stopped during
  dependency setup, before lint/tests, to retain the requested plan-first handoff.
  Repository checks have not passed in this turn; the coder must complete environment
  preparation and implementation verification.
- Run the focused suites above and `tests/ace/tui/test_agent_panel_title_refresh.py`.
  Include the existing monitor and placeholder-dedup regressions if their paths are
  touched; the plan does not require editing those paths.
- Run `just check` after implementation and fix attributable failures. Inspect
  `tools/select_tests --explain` if needed to ensure the relevant tests were selected.
- Run `just test-visual` for the Agents-tab presentation change; inspect any actual,
  expected, and diff images before accepting an intentional golden update.
- Use `just check-full` only when the selection broadens/escalates or at a required
  landing gate, and only through `/sase_monitor` with `TESTING` / `TESTED` statuses. Use
  that skill for any verification command that requires a long handoff.
- Before completion, confirm no grace-window code or documentation remains in the
  corrected model/tests, no new filesystem work reaches rendering/navigation, and the
  starting count is removed only by a real status change or record removal.

## Boundaries

This plan fixes why a STARTING record is visible. It does not infer from a missing
prompt that a process is dead, kill or release the screenshot's workspace claim, change
launch timing, or suppress STARTING from CLI/backend inventories. Any separately
confirmed claim-lifecycle defect requires its own evidence and scope.
