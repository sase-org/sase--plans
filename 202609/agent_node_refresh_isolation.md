---
tier: tale
title: Keep agent nodes stable during background refreshes
goal:
  Eliminate transient provider and relationship flicker by isolating worker-owned row
  graphs until UI publication, while preserving startup responsiveness and incremental
  panel updates.
size: medium
proposed_by: bbugyi200.athena.0n0
create_time: 2026-09-18 13:13:21
status: wip
---

# Keep agent nodes stable while background refreshes prepare replacement rows

## Outcome and scope

The Agents tab must keep displaying a coherent, already-published row graph while a
background refresh computes its replacement. Provider icons, family/clan counts, status,
hierarchy, and runtime metadata must not briefly disappear or change before the
completed result is applied. Preserve incremental BY_STATUS panel updates and bounded
startup loading.

This is one coordinated, medium-sized implementation for one coding agent. The confirmed
defect is mutable Textual presentation objects shared across the worker/UI boundary. It
belongs in this repository's TUI models and loading glue; no shared backend semantics,
Rust API, feature flag, CLI, or configuration change is needed. Do not change startup
quiet thresholds or force an archive scan before first paint.

## Prior work and diagnosis

Read context: `bead:sase-12p`, its phase notes, and
`plan:202609/by_status_panels_and_stale_tui.md`. That epic fixed unconditional panel
rebuilds under BY_STATUS and surfaced stale imported TUI code. Its tests and soak
measured panel lifetime and rebuild reasons. Those properties can hold while a mounted
panel renders partially mutated rows, so this finding does not invalidate the earlier
improvement.

The following path was verified on source revision `e0f1a8d43`:

1. `AgentLoadingApplyMixin._make_prepared_apply_snapshot` in
   `src/sase/ace/tui/actions/agents/_loading_apply.py` copies the outer cached and
   capacity lists with `list(...)`. The contained `Agent` objects remain the same
   objects used by the live lists and widgets.
2. `_loading_disk_full.py` and `_loading_disk_delta.py` pass that snapshot to
   `prepare_loaded_agents_worker_boundary` using `asyncio.to_thread`.
3. `merge_incomplete_load_after_complete_history` in `_loading_compute_merge.py` retains
   cached rows missing from a partial update, then runs
   `_normalize_relationships_after_merge` on the merged objects. This occurs for exact
   artifact deltas even before complete history, bounded prefixes with `has_more`, and
   incomplete loads after complete history.
4. `models/_agent_status_apply.py:apply_status_overrides` clears and reconstructs
   `followup_agents`, resets `wait_display_source`, and changes derived status and
   metadata. `models/_agent_ordering.py:sort_and_reorder` clears `runtime_children` and
   `family_container`, then rebuilds links. `models/_agent_tree.py` creates new
   synthetic clan containers. An old displayed clan container can therefore remain empty
   for the entire interval between worker normalization and UI apply.
5. `widgets/_agent_list_helpers.py:ordered_row_providers` recursively reads
   `runtime_children`. `_agent_list_render_agent_prefix.py` renders those providers;
   `_agent_list_render_cache.py:agent_render_key` includes them in the cache key.
   Runtime row patches and other UI rendering can observe the intermediate graph. The
   race affects more than icons: family classification depends on `followup_agents`, and
   aggregate runtime/counts depend on the same links.

### Reproduction already performed without implementation changes

An in-memory probe used the real `Agent` model, `sort_and_reorder`,
`_make_prepared_apply_snapshot`, and `prepare_loaded_agents_worker_boundary`. The live
fixture was a clan in tribe `epic` with two running members whose providers were Claude
and Codex. Complete history was already marked seen. A nonempty exact delta updated an
unrelated row. A thread-event barrier paused the real `_clear_runtime_children`
immediately after its clearing operation; the main thread read/rendered the old
container before, during, and after worker preparation, before any apply. External
project-name and runner-capacity lookups were stubbed; merge and relationship
normalization were real.

| Input ownership                              | Before worker | During worker | Worker finished, before apply | Prepared replacement |
| -------------------------------------------- | ------------- | ------------- | ----------------------------- | -------------------- |
| Production shared row objects                | Claude, Codex | empty         | empty                         | Claude, Codex        |
| Shallow copy of each row                     | Claude, Codex | empty         | empty                         | Claude, Codex        |
| Detached graph using one deep-copy operation | Claude, Codex | Claude, Codex | Claude, Codex                 | Claude, Codex        |

A direct merge probe also rendered the actual prefix as `🎭 🤖 ` before the clear and as
an empty string afterward. No provider metadata changed in the source fixture, and no
widget was unmounted. The detached-graph case is an experimental control, not a
production fix already made. The user's exact live visual event has not been captured;
this is a deterministic defect matching the reported symptom.

### Startup-delay hypothesis

The delayed arrival of some rows is consistent with the existing staged loader:
`_loading_disk_viewport.py` requests a bounded prefix; `_loading_refresh_polling.py`
arms one-shot unwindowed prefix completion after two seconds of input quiet. Some
repair/fallback or unsupported-query loads arm full-history reconciliation after 30
seconds of input quiet. These are scheduling thresholds, not guaranteed arrival times;
ongoing input or refresh work can defer them. Healthy Tier 1 loads need not run
full-history reconciliation.

Startup enrichment and recurring partial refreshes use related loading machinery, but
delayed startup discovery is not necessary for the confirmed race: the probe reproduces
with complete history already loaded and an unrelated artifact delta. Retain legitimate
progressive discovery; fix publication ownership rather than making first paint wait for
all history. Validate actual startup convergence below.

## Implementation

### 1. Turn the reproduction into a failing regression

Add focused tests near `tests/test_agents_tab_apply_boundary.py` and
`tests/test_agents_tab_artifact_delta_merge.py`, using the production snapshot and
worker entry points. Use bounded thread events/barriers with guaranteed cleanup, not
timing-sensitive sleeps. Render the existing live row while relationship normalization
is paused and again after preparation but before apply.

Cover a synthetic clan plus a multi-provider family with real shell relationships. A
delta to another row must leave the displayed graph's provider order, child links,
family classification, status, and relevant counts unchanged throughout preparation.
Verify the new result is correct after apply. Include a worker exception/cancellation
path proving abandoned preparation cannot damage the currently displayed graph.
Cancellation of an `asyncio.to_thread` await does not stop its thread: release and join
the test worker and verify isolation through its eventual completion.

### 2. Establish an explicit worker-owned row graph

Create a small, centralized TUI graph-snapshot/ownership boundary before any worker
mutation of cached UI objects. Audit the entire prepared pipeline, including cached rows
retained by the incomplete merge, capacity rows, cached proc-shell carryover, and any
cached loader objects supplied to mutating preparation. Both broad and exact-delta
worker paths must obey the same rule. Direct/synchronous apply helpers must also
preserve old row values needed for the subsequent display diff.

Use a cycle-safe graph copy with a shared memo across overlapping input collections, or
an equivalently proven detached projection. Copy each reachable mutable object once;
preserve intentional aliases within the new graph. A plain list copy,
`copy.copy(agent)`, or `dataclasses.replace(agent)` alone is insufficient. Handle
`runtime_children`, `followup_agents`, `family_container`, `wait_display_source`, retry
links, and mutable metadata touched by preparation. Do not serialize with `asdict` or
JSON: runtime back-pointers form cycles and carry presentation state. An optimized copy
must be justified by profiling and ownership tests, not by an incomplete manually
enumerated set of current icon dependencies.

Keep graph detachment and normalization off the event loop on async refreshes. Avoid
introducing an archive-sized copy into `_make_prepared_apply_snapshot`, which also
serves UI-side stale-token checks. Keep cheap UI capture separate from worker ownership
if needed. Ensure graph size, rather than repeated traversal through every alias, bounds
the copy cost. Avoid redundant copies for incoming rows already owned exclusively by the
current worker.

The worker must not mutate the displayed graph at any point, including final query,
fold, status-override, or runner-capacity preparation. Publish the prepared graph only
through the existing UI apply/finalize path, preserving selection by identity, current
fold/query validation, capacity generation checks, live-hint carryover, coalescing, and
navigation gates. Retain old row values until the incremental display diff has compared
them. Do not use a UI-blocking lock, suppress clock ticks, retain provider badges
forever, or hide the race behind extra panel rebuilds/debouncing.

Document the graph ownership contract at the snapshot/worker boundary so that
`frozen=True` on a dataclass containing mutable rows is not mistaken for isolation.

### 3. Preserve loading and legitimate updates

Exercise these load sequences through production merge/apply entry points:

- Bounded startup prefix, deferred prefix completion, then another bounded refresh.
- Query/fallback reconciliation to complete history followed by a Tier 1 refresh.
- Repeated exact deltas, including one to a different clan in the same tribe.
- A real child/provider addition, provider correction, terminal transition, dismissal,
  and explicit artifact deletion. Real changes must appear when the new graph lands;
  omitted rows in an incomplete load must retain the existing merge semantics.

Use fake monotonic time to cover the two-second and conditional 30-second paths. Assert
eventual expected membership, preservation of already discovered rows, and no loss of
existing badges while a follow-up is pending. Do not interpret legitimately new
providers or newly discovered historical rows as flicker. Preserve full-load authority
to remove absent rows and exact-delta authority to apply deletions.

Add graph tests with a family back-pointer cycle and a row shared by the visible and
capacity rosters. Assert internal alias preservation and absence of mutable aliases back
to live objects. Compare explicit observable fields; avoid recursive dataclass equality
as a graph oracle. Existing tests that depend on Python object identity should retain
identity requirements only where they express a real contract.

### 4. Verify the visible result and cost

Extend `tests/perf/test_agents_display_rebuild_guard.py` or a closely scoped Textual
integration test with BY_STATUS, a stable query such as `NOT machine:apollo`, and an
`@epic` panel containing multiple provider-bearing family/clan nodes. Exercise a real
background worker with the normalization barrier while forcing the existing row
render/patch path. Assert unchanged old row text/badges and panel widget identities
before publication, correct text after publication, preserved selection/folds, and no
full panel rebuild for stable membership. Genuine status/membership changes must still
take the established correct fallback.

Run focused tests for apply-boundary isolation, incomplete and artifact-delta merges,
provider rendering/runtime ticks, startup prefix/reconciliation, refresh coalescing, and
the existing panel rebuild guard. Measure preparation/copy overhead on a representative
large graph with shared family links and on repeated small deltas. Use the existing TUI
performance tooling/runbook to compare before and after: first-paint bounds must remain
intact, graph copying must stay off the pump, and navigation should retain the existing
p95 <16 ms target. Report actual measurements and fixture size. Quiet ticks must still
reload no unchanged surfaces. Do not add flaky absolute wall-clock assertions to unit
tests.

Run `just fmt` (or `just fix`) and `just check` after implementation. Follow
`lint_and_test.md` for any required escalation; long verification runs use
`/sase_monitor`, and `just check-full` is only run through that skill.

Finally, validate a freshly started TUI running the fixed source. Prefer an isolated
session using the host's normal query/grouping, without disturbing the user's live
session. Read `tui_screenshot.md` before live screenshot tooling. Record startup load
sources and initial-to-converged membership, then observe repeated partial refreshes
under churn after convergence. Capture intermediate rendered-row/provider evidence
across at least ten relevant refreshes, alongside panel lifetime and rebuild reasons; a
single screenshot or panel-mount count alone cannot prove this fix. Record the actual
imported revision and whether this is a disposable session or the user's session. If
live churn is unavailable, use controlled replay and explicitly report the
live-verification limit rather than claiming a live soak.

## Acceptance

- The deterministic pre-fix race fails on the original code and passes on the fix:
  published row graphs remain unchanged until application, even if preparation fails, is
  canceled, or is paused mid-normalization.
- All relevant provider icons and family/clan presentation fields remain stable during
  unchanged refreshes; legitimate source updates become visible on apply.
- Startup progressively converges without retreating to earlier partial data, and the
  report distinguishes staged discovery from the independently reproduced race.
- Existing BY_STATUS panel stability, query/fold/selection behavior, dismissal and
  deletion semantics, refresh coalescing, and startup responsiveness remain intact.
- Focused regressions and required repository checks pass. The implementation report
  includes measured cost and fresh-process render evidence, with any observation limits
  stated plainly.
