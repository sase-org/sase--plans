---
tier: tale
title: Add Focus and Fleet agent views
goal:
  ACE presents local and enrolled-machine agents through responsive Focus and Fleet
  views with honest freshness, durable follows, and no unnecessary remote work.
size: medium
proposed_by: bbugyi200.athena.sase-xe.11
bead: sase-xe.11
create_time: 2026-09-06 21:18:05
status: wip
---

- **PARENT:** [202609/remote_dispatch_fleet.md](remote_dispatch_fleet.md)
- **BEAD:**
  [sase-xe.11](https://github.com/sase-org/sase--beads/blob/main/pages/sase-xe/sase-xe.11.md)

# Add Focus and Fleet agent views

Implement bead `sase-xe.11` as one bounded TUI feature pass. Preserve the existing
Agents list and detail machinery, but add a mode axis that projects local and remote
fleet records into that machinery without putting remote I/O on the Textual event loop.
The implementation must remain inert when `remote_dispatch` is disabled or no machine is
enrolled.

## 1. Bridge enrolled machines into the federation read facade

- Extend the dispatch/federation adapter so the supported `dispatch.machines` records
  and `LocalCredentialStore` can be projected into `FederationHostConfig` values.
  Preserve compatibility with the existing `dispatch.remote_hosts` test surface, but
  make enrolled machine aliases, connection plans, installation pins, and stored bearer
  tokens the source used by ACE.
- Keep configuration inspection pure and local. An empty enrolled-machine set, a
  disabled `remote_dispatch` flag, quarantined/invalid records, or missing credentials
  must produce a disabled/degraded snapshot without starting the federation worker or
  touching the network. Expose safe per-machine diagnostics for the UI rather than
  leaking credential or path details.
- Add focused unit coverage in `tests/test_dispatch_federation.py` (and dispatch tests
  where appropriate) for enrolled-record projection, alias/origin preservation, missing
  credentials, quarantine, feature-off behavior, and the zero-machine/no- supervisor
  invariant.

## 2. Model fleet snapshots and adapt remote wire rows to Agents data

- Add an ACE fleet-view model/adapter near the existing agent data providers. Parse
  `FederationReadResponseWire` host results for summary, catalog, and followed-batch
  operations into immutable host/view snapshots containing rows, pagination cursors,
  running/count data, connection status, freshness, partial/error state, and safe
  diagnostics.
- Validate/project each `ResolvedAgentSummaryWire` through the existing Rust contract
  binding and retain its logical/exact locators, revision, capabilities, content
  metadata, bounded intent, and machine origin. Derive origin-qualified stable row
  handles so equal human names from different installations never collide. Extend
  `AgentState` and row-handle construction only with presentation/identity metadata
  needed by the shared list and detail paths; do not recreate lifecycle or count rules
  in Python.
- Build Focus from the existing local agent snapshot plus the viewer-local durable
  follow snapshot resolved through `followed_batch`; build Fleet from paged `catalog`
  host results. Use `fleet_count_focus_and_fleet` for logical counts and propagate
  unknown origins, partial results, and observed timestamps into header and machine
  presentation.
- Add unit tests for wire parsing, cross-origin identity stability, followed-only Focus
  projection, duplicate logical rows, partial/offline hosts, paging, bounded intent, and
  count/freshness propagation.

## 3. Add demand-tiered, cancellable remote hydration

- Introduce one app-owned fleet controller/mixin that cooperates with the current
  `_schedule_agents_async_refresh` flow. It should publish cached/local Focus content
  immediately, then use `spawn_pump_free_task` for remote summary/batch/catalog/detail
  work only while the Agents tab and relevant mode/row are active.
- Tier reads as: header summary for visible Agents chrome; followed-batch for Focus;
  first catalog page plus additional pages requested by `AgentsViewport` for Fleet; and
  detail/content metadata only for the selected remote row after the existing selection
  debounce. Re-read active tab, mode, selection identity, and generation after every
  await before applying results.
- Coalesce refreshes, cancel superseded mode/detail/page tasks, and cancel all owned
  tasks during teardown. Retain usable cached rows during refresh/failure and mark them
  aging/stale/offline/partial honestly. Never perform network or worker startup from a
  key handler, mount path, or no-machine startup.
- Cover demand gating and races with deterministic async tests: no machines and
  feature-off perform no federation calls; Focus does not fetch Fleet catalogs; Fleet
  pages only at viewport demand; rapid mode/selection changes cannot apply stale
  results; task cancellation and teardown leave no late UI mutation.

## 4. Add the Agents header, modes, machine sections, and row affordances

- Add a compact `#agents-header` `Horizontal` to `_app_layout.py`, using the existing
  `PanelTabStrip` for `Focus` and `Fleet`. Hide the entire strip until at least one
  valid machine is enrolled. Show logical running-count chips in the tab labels and a
  restrained partial/freshness indicator without presenting stale numbers as live.
- Add `current_agents_subtab` as a normalized reactive and keep independent mode state
  for selected stable identity, list scroll offset, detail/panel selection, and fold
  registries. Switching modes must restore that mode's state while continuing to use the
  single existing `AgentList`, panels, filter, and navigation implementation.
- Keep Focus's current local grouping and add followed remote rows without regressing
  local family/clan behavior. In Fleet, group rows under collapsible machine headers
  showing alias, connection/freshness state, and count summary; render the full remote
  catalog with restrained columns. Extend list rendering/cache keys for a follow star,
  machine accent rail, safe origin label, and bounded one-line intent preview. Preserve
  status semantics and make stale/offline/unknown data visually distinct but readable.
- Provide honest zero/loading/unavailable states. When no machine is enrolled, retain
  the current local Agents experience and expose a discoverable `Connect a machine`
  action/hint. When machines exist but no rows match, distinguish empty Focus, empty
  Fleet, loading, partial, and unavailable states.
- Add focused behavioral tests for mode-state restoration, machine folds, selection
  across refresh/paging, hidden header/no-machine startup, freshness labels, and shared
  list/detail behavior. Add deterministic ACE PNG fixtures and snapshots for followed,
  partial, offline, empty, loading, unavailable, and zero-machine states, including a
  constrained-width case.

## 5. Add follow and mode actions through every keymap surface

- Add named `AppKeymaps` actions for forward/reverse Agents-mode cycling and
  follow/unfollow, with defaults in `src/sase/default_config.yml`. Add availability
  predicates, binding metadata, command-palette metadata, and Agents help rows so every
  configurable action remains schema-complete and discoverable. Keep `[` and `]`
  behavior unchanged; availability-gate direct mode selection if digit shortcuts are
  retained from the accepted design.
- Implement follow/unfollow as viewer-local `follow_store` mutations keyed by the remote
  logical locator. Update the current projections immediately, emit a concise toast,
  offer Undo using the inverse local operation, and reconcile on the next followed-batch
  refresh. Add a `View in Focus` action for a Fleet row and a `Connect a machine` action
  that routes to the existing enrollment command guidance; keep remote actions disabled
  for local rows, missing capability, feature-off, or invalid machine state.
- Test default/configured bindings, help and command-palette coverage, action
  availability, immediate follow/unfollow and Undo, Fleet-to-Focus selection transfer,
  and non-collision with existing Agents and Artifacts bindings.

## 6. Verify performance, visuals, flags, and phase completion

- Run narrow unit and Textual tests while iterating, then run the Agents `j`/`k`
  navigation benchmark with representative Fleet sections and enough rows/pages to
  detect per-keystroke remote work or full-list rebuild regressions. Record/compare the
  existing benchmark metrics and keep navigation within the repository's current
  thresholds.
- Exercise both `remote_dispatch` flag states and assert identical local-only startup
  behavior when disabled. Review generated PNGs for clipping, hierarchy, stale/partial
  honesty, focus visibility, and narrow-terminal degradation; update only intentional
  baselines.
- Because project files changed, follow the repository verification memory: install the
  linked Rust core wheel first if required by the ephemeral checkout, run the relevant
  focused suites and PNG lane, then run the default `just check` lane. Do not substitute
  `just check-full` inside the agent turn.
- Before completion, inspect `sase bead epic-symbols sase-xe.11` and resolve each
  remaining symbol or re-key it to the appropriate still-open parent/follow-on phase.
  Record any genuinely out-of-scope discovery only as a `PROPOSED FOLLOW-UP:` note on
  `sase-xe.11`. Close only `sase-xe.11`, with a note naming the focused tests, visual
  snapshots, benchmark, feature-flag cases, `just check`, and epic-symbol result that
  were actually verified; do not close or mutate the parent epic lifecycle.
