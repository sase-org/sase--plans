---
tier: tale
title: Honor the ACE Agents viewport contract
goal:
  ACE materializes only the searched visible-plus-prefetch agent window while preserving
  exact queries, selection, hierarchy, and lazy selected-row details.
size: medium
proposed_by: bbugyi200.athena.sase-uv.8
bead: sase-uv.8
create_time: 2026-09-09 19:59:41
status: wip
---

- **PARENT:**
  [202608/ace_tui_responsiveness.md](https://github.com/sase-org/sase--plans/blob/main/202608/ace_tui_responsiveness.md)
- **BEAD:**
  [sase-uv.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-uv/sase-uv.8.md)

# Honor the ACE Agents viewport contract

## Context

Phase `sase-uv.8` is the measurement-gated final reduction in the ACE Agents-tab loader.
The approved epic design observes that `AgentsViewport` already defines a 40-row visible
window plus 80 prefetched rows, but `DirectAgentsDataProvider` discards both `viewport`
and `search_query`, while the live TUI disk path bypasses the provider and asks
`load_tiered_agents` to materialize up to 1,000 active plus 200 completed artifact
records. The baseline phase measured 1.0--1.4 seconds for 468 agents. After the
projection prerequisite, the same unbounded list-shaped path still measured a 670.54 ms
warm median over about 805 records, above the epic's 300 ms budget, so the viewport
phase remains necessary.

A fresh measurement in this workspace must follow `just install`, because the current
editable `sase_core_rs` extension is not built. Repeat the warm measurement before
editing implementation code. If it unexpectedly meets every epic budget on the installed
current tree, record that evidence on `sase-uv.8` and close the phase as unnecessary;
otherwise implement the bounded path below and use the same harness for the before/after
comparison.

The change crosses the repository boundary deliberately: selection, filtering, and
ordering that determine which records are materialized are shared backend behavior and
belong in `sase-core`; provider and Textual state wiring remain in `sase`.

## Desired behavior

- An ordinary direct-provider refresh materializes at most the requested visible-plus-
  prefetch window, plus the minimal relationship closure needed to render coherent
  workflow/family/clan rows, rather than the complete recent index result.
- The window is chosen after hidden/dismissed visibility rules, search filtering, and a
  deterministic newest-first root ordering. Repeated reads of an unchanged index return
  the same boundary.
- Existing structured Agents queries retain exact results. Predicates supported by
  indexed columns are applied before record JSON is decoded; predicates that require
  Python-only state or hydrated content use ordered cursor paging and exact post-filter
  evaluation, stopping once the requested window is full rather than silently dropping
  matches beyond the first raw page.
- Full-history/revive operations retain their explicit unbounded semantics. Source- scan
  fallback stays bounded and reports that the result is incomplete. Exact artifact
  deltas continue to use their narrow merge path.
- Query edits show the cached in-memory result immediately, schedule a coalesced
  background re-window, and reject an in-flight result if the query, selected identity,
  or viewport generation changed while it was loading.
- Detail output, logs, and relations continue to hydrate only for the selected projected
  row; the list refresh must not add eager full-record hydration.

## Implementation

1. Extend the `sase-core` artifact-index query wire with a backward-compatible optional
   window/cursor and an index-filter representation. In
   `crates/sase_core/src/agent_scan/index.rs`, select visible active/completed
   candidates in one deterministic newest-first order, apply index-representable query
   predicates before decoding `record_json`, fetch one extra candidate to determine
   `has_more`, and return stable continuation metadata. Expand only the lineage records
   required for coherent parent/child presentation without counting that closure against
   the user window. Keep existing callers unchanged when the new fields are absent, and
   add Rust tests for boundary stability, active/completed interleaving, visibility,
   filtering, continuation, and relationship closure.

2. Mirror the new wire fields and result metadata through
   `src/sase/core/agent_scan_wire_records.py`,
   `src/sase/core/agent_scan_wire_conversion.py`, and the existing facade/PyO3 serde
   bridge. Translate the parsed Python Agents-query AST into the validated core filter
   form for supported metadata predicates. For content-, pin-, or other local-only
   predicates, page ordered core candidates and run the existing exact evaluator until
   the requested number of rendered matches is available or the cursor is exhausted.
   Invalid queries must keep the current parse-error behavior and must not accidentally
   narrow the result.

3. Thread `search_query` and `AgentsViewport` through `load_tiered_agents`, the Tier-1
   artifact snapshot selector, and `DirectAgentsDataProvider` instead of deleting them.
   Make the direct provider's `AceSnapshot` metadata truthful (`requested_limit`, query,
   continuation/truncation state, and page count), preserve the bounded source-scan
   fallback, and leave explicit `full_history=True` calls unwindowed. Route the live
   Agents-tab disk load through the provider contract so the implementation is actually
   exercised rather than remaining an unused facade.

4. Capture the current query and an `AgentsViewport` before dispatching the worker load
   from `actions/agents/_loading_disk.py`; derive the window end from the selected row,
   visible row count (with the contract defaults as the safe fallback), and prefetch.
   Re-capture state after the await and discard/schedule a last-request-wins refresh
   when the inputs drift. Keep the immediate cached `_refilter_agents()` response on
   query edits, then schedule the coalesced provider refresh so matches outside the old
   window become visible. Preserve selection by identity across page changes and keep
   the existing delta and full-history paths explicit.

5. Add focused Python tests for provider argument propagation and snapshot metadata,
   bounded Tier-1/fallback/full-history behavior, exact search across more than one core
   page, query-change stale-result rejection, selection preservation, and the guarantee
   that ordinary list refreshes do not call full-record hydration. Update existing wire
   and loader tests for backward-compatible defaults rather than weakening their
   assertions.

## Verification

1. Run the linked `sase-core` repository's `just check`, which includes both core and
   PyO3 binding tests.
2. Reinstall the editable core binding in `sase`, run the focused provider/loader/query/
   selection tests, and run `just check` in `sase`.
3. Using the same live index and cached-freshness harness for both paths, record at
   least five warm samples for the legacy unbounded query and the default 120-row
   viewport. Record materialized record/agent counts and median latency on `sase-uv.8`;
   the bounded warm `load_tiered_agents` median must be below the epic's 300 ms target
   or any remaining stage must be explained with a profile.
4. Run the Agents large-list keystroke bench and relevant detail-hydration tests to
   confirm row movement remains within its established ceiling and selected-row detail
   behavior is unchanged.
5. Before closing only `sase-uv.8`, run `sase bead epic-symbols sase-uv.8`, resolve
   every remaining symbol or re-key it to a still-open bead, and close with a note
   containing the exact checks and measurements above.
