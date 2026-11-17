---
tier: tale
title: Finish TUI startup imports and background scheduling
goal:
  Complete sase-1t3.8 with a smaller import closure and deferred monitor and AXE work
  while preserving recovery and first-use behavior.
size: medium
proposed_by: bbugyi200.athena.sase-1t3.8
proposed_with: codex/gpt-6-astra
bead: sase-1t3.8
create_time: 2026-11-17 06:10:18
status: wip
---

- **PARENT:**
  [202611/tui_storm_trust_heap_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202611/tui_storm_trust_heap_recovery.md)
- **BEAD:**
  [sase-1t3.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1t3/sase-1t3.8.md)

# Finish the startup critical path for sase-1t3.8

## Outcome and scope

Complete assigned phase `sase-1t3.8` of `plan:202611/tui_storm_trust_heap_recovery.md`.
Remove the remaining eager action imports, avoid unnecessary roster-triggered monitor
reconciliation, and keep full AXE snapshots deferred until Services is visited. Preserve
the existing startup release, refresh coalescing, service lifecycle, and monitor
settlement behavior.

This is a single medium tale because much of the original phase has already landed.
There are no earlier implementation notes or unfinished attempts on this bead. The
remaining work is import wiring and TUI scheduling with existing backend APIs; a new
backend cache or index is not required.

The parent's decisions are final: `hotfix_reinstall=yes`, `trust_ack_compat=cutover`,
and `tui_perf_memory=no`. The first two belong to other phases. Do not edit any memory
note, install globally, or restart the live TUI. Read the applicable memory through
`/sase_memory_read`; use `/sase_repo` before opening another repository. Shared domain
changes, if an unexpected defect makes them necessary, belong in Rust core, not a new
Python implementation.

## Evidence and work already landed

Planning measurements used sase HEAD `694cdec86b`, Python 3.14.7 and Textual 8.0.1. The
host was busy (load averages approximately 7.7/8.2/8.4). These are diagnostic samples,
not a quiet-host startup p95 or a claim about the installed live TUI.

| Probe                                                                  | Current result                                                                                                                        | Consequence                                                                                                    |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `.venv/bin/python tools/tui_import_closure`                            | 1.84 s import CPU; 2,838 first-party modules; 3,548 total; 12,051,331 normalized source bytes                                         | Below the current 12,162,484 cap, but the action import closure remains expensive                              |
| Fresh `-X importtime`                                                  | `actions.agents._core` cumulative 1.309 s; `gate_turn` 0.395 s; `notification_gates` 0.387 s; the `macro.models` package edge 0.238 s | Attribute and defer the heavy package/action edges; these nested times must not be added                       |
| `load_axe_config()`                                                    | Cold 468 ms, warm 18.7 and 18.4 ms; 14 routines, 62 chops                                                                             | The old 2–3 s idle config cost does not reproduce; do not add the proposed general config cache                |
| Header-only `collect_axe_status_data` with reused `AxeStatusReadCache` | Cold 92.9 ms, warm 30.8 ms; instrumented warm sample 65.1 ms; reported `file_opens=0`                                                 | Preserve skip-on-unchanged-token behavior; do not optimize the old multi-second collector without new evidence |
| `_reconciliation_project_records(None)`                                | 171 and 162 ms, 47 candidates                                                                                                         | Bounded active-monitor index querying already exists; no new query API is needed                               |

The collector's `file_opens` counts its instrumented cache reads, not every file opened
by config, service status, or proc loading. The profiled warm sample spent about 41 ms
in config loading and 20 ms in proc snapshot loading; the profile adds overhead. No live
monitors were settled during these measurements.

Existing implementation to retain:

- `_startup_loads_core.py` holds hidden work until visible readiness, with an existing
  45-second safety release. `_loading_disk_support.py` records deferred reconciliation
  and `_flush_deferred_monitor_reconcile` releases it afterward.
- `_run_axe_startup_init_body` in `actions/axe_display/_refresh_full.py` already loads
  headers only and requests a full follow-up for initially/currently visible Services.
  It also owns auto-start and `--restart-axe`; retain those semantics.
- `src/sase/monitor/store.py::_reconciliation_project_records` already calls the Rust
  artifact index with `only_monitors`, active-only selection, and a 1,000-row limit,
  retaining a bounded fallback. Keep its existing tests and behavior.
- Recent hijack and agents-sync changes already defer their heavy modules. Keep every
  existing `_DEFERRED_MODULES` exclusion in the import-budget test.

The original consolidated research reference in the parent plan currently resolves as
missing. Use the audited parent plan's recorded historical figures and the measurements
above; do not reconstruct or fetch the sidecar through another path.

## Implementation

### 1. Remove heavy implementation from the app import closure

Re-run the import measurement at the coding agent's HEAD and use a fresh `-X importtime`
subprocess to attribute remaining paths. Start with `actions/agents/_core.py`,
`_agent_enter_action.py`, `_agent_enter_badges.py`, `_agent_enter_labels.py`,
`_agent_enter_index.py`, `_notification_dispatch.py`, `_notification_modal_flow.py`, and
`_toasts.py`.

The package initializers matter: `sase.gate_turn.__init__` eagerly imports models, which
import status and notification models; `sase.notification_gates.__init__` eagerly
imports execution, validation, poller, and service modules; importing
`sase.macro.models` first executes the large macro package initializer. Merely changing
an import to a deeper module may therefore save nothing.

Use ordinary use-site imports and `TYPE_CHECKING` for annotation-only edges. Where a
lightweight first-paint helper necessarily crosses an eager package initializer, use the
existing explicit lazy-export pattern to preserve its public API while deferring
execution/model closures. Keep labels and cheap state helpers available. Do not redesign
the mixin hierarchy, introduce a generic import hook, copy domain logic into the TUI, or
defer required work into a slow synchronous keypress handler. Heavy first-use action
work must retain its existing worker or durable-proc boundary. Read `macros.md` before
changing macro package wiring.

Add fresh-process assertions for the heavy implementation modules actually removed from
startup, plus focused first-use tests for changed public exports and affected gate
actions. Existing patch seams, `__all__`, lazy `__dir__`, and type checking must still
work. Lightweight `gate_turn` or notification modules need not be banned wholesale if
their data is required at first paint.

Lower the normalized-source footprint baseline in
`tests/ace/tui/test_app_import_budget.py` to the measured post-change baseline,
documenting the revision and attributed removed edges. Keep metric version 1, the fixed
4,096-byte allowance, strict `<`, existing elapsed-time/total-module limits, and every
existing deferred-module exclusion. Update the baseline-dependent guard tests
consistently. Never loosen a budget to accommodate this work.

### 2. Avoid futile monitor reconciliation without losing recovery

In `actions/agents/_loading_disk_support.py`, separate an empty in-memory monitor
signature from an immediate need to scan. Roster loads with no monitor candidates must
not start a worker on every load or directly after startup release. Preserve the
existing post-visible-ready gate, navigation/load deferral, single-flight behavior,
changed-signature trailing pass, feedback-loop suppression, and guard cleanup on
failure/cancellation.

An empty displayed roster is not proof that the global monitor store is empty: query
filters, bounded loads, dismissed/hidden records, and other projects can hide
candidates. Preserve recovery through a coalesced, low-frequency, post-ready backstop
using the existing 30-second reconciliation cadence and bounded backend query. Do not
mark an incomplete or query-scoped roster globally authoritative. Use the existing
startup/task/timer lifecycle for that backstop, ensure teardown cancels it, and do no
disk or liveness probing while building the signature. Positive monitor signature
changes may request the existing worker immediately after the startup/activity gates
allow it. Keep actual eligibility and settlement decisions in the current backend; do
not create a second Python state machine.

Extend `tests/ace/tui/test_monitor_reconcile_off_read_path.py`: an empty roster burst
starts no immediate reconcile; startup holds still apply; an incomplete or filtered
empty roster eventually receives the bounded backstop; a newly appearing monitor
triggers work; unchanged bursts coalesce; changed rows get one trailing pass; dead
supervisors still settle and request one refresh; spawn failure and teardown
release/cancel correctly. Update old test harnesses that used an empty roster as a
stand-in for a positive candidate.

### 3. Keep full AXE collection behind Services visitation

Close the remaining bypass in `actions/event_refresh/_auto_refresh_surfaces.py`:
currently `axe_full = sanity_due or self.current_tab == "services"` performs full
collection on a hidden, never-visited Services tab. Track Services visitation separately
from `_axe_first_load_done` (headers loaded) and `_axe_full_snapshot_ready` (full data
successfully applied), initializing and setting it in the normal startup/tab-switch
state paths. Initial Services counts as a visit. Before the first visit, even a sanity
refresh is header-only and does not request all chop log tails.

After a visit, keep the established full sanity reconciliation and visible-Service
refresh behavior. A first visit schedules one full request through the existing guard,
including when a header load is already in flight. Inspect `_schedule_axe_async_refresh`
and the trailing request in `_run_axe_status_refresh`: the scheduler currently
unconditionally promotes requests to full. Preserve the requested header/full strength
when coalescing, so a hidden header request cannot accidentally escalate itself, and a
real Services full request cannot be lost. Re-capture current selection/tab after awaits
where needed. Keep selected-tail loading, pinned history offsets, loading indicators,
and service lifecycle intact.

Extend the existing startup/refresh tests to cover initially hidden Services, sanity
before a visit, first visit during an in-flight header refresh, rapid tab switches,
initially visible Services, post-visit sanity, and failed/spawn-failed loads. On stable
tokens with no watcher dirtiness and before the sanity deadline, assert zero collector
calls, including with the watcher absent. Config/service token changes and lifecycle
transitions must still cause a header refresh. This call-count contract is the
deterministic idle-CPU guard; do not add timing-sensitive unit tests or a second cache
keyed only on config while service state can change.

## Verification and completion

1. Re-read phase notes and inspect HEAD before editing, preserving already-landed fixes.
   Record any changed baseline and any safely dropped work in the bead.
2. Run focused import-budget and affected gate first-use tests, monitor scheduling and
   `tests/monitor/test_monitor_store_reconcile_queries.py`, AXE collector, startup
   observability/lifecycle, and `test_event_handlers_auto_refresh_clean_ticks.py` plus
   the debounce/token tests. Reuse existing fixtures; never reconcile real monitors for
   a benchmark.
3. Compare before/after import CPU and normalized footprint with the same interpreter.
   Use a separate TUI or controlled fixture capture for startup/idle traces following
   `tests/perf/capture_tui_startup.py` and the existing performance runbook. Do not
   restart the live instance. Record host load, Textual/core versions, visible-ready
   anchor, `axe.collect` snapshot mode/call counts, and monitor timing. Parent targets
   remain process-to-`on_mount` p95 below 3 s and cheap quiet ticks; report measured
   results honestly if host load prevents a meaningful p95. Already-landed header-only
   startup means the historical 3–4 s AXE saving must not be claimed again.
4. Run formatting and `sase tool run check` as required by `lint_and_test.md`. Use
   `/sase_monitor` if a command cannot finish in the turn. `just check-full` is not
   requested. If rendered output changes, follow the targeted screenshot rules; no
   visual redesign is intended.
5. Record independent clean-base evidence for any unchanged check failure and append
   `PROPOSED FOLLOW-UP:` on `sase-1t3.8`, citing an existing tracking bead if present.
   Do not create task beads. Such proven pre-existing failures do not leave this phase
   open.
6. Before completion, run `sase bead epic-symbols sase-1t3.8` (none existed during
   planning); resolve any new entries or re-key them to an appropriate still-open bead.
   Close only `sase-1t3.8` with a note of measured results and verification. Do not
   close `sase-1t3` or any ancestor plan. End through `/sase_final` so the host commits
   the implementation.
