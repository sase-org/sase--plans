---
tier: tale
title: Honour the AgentsViewport contract on the direct ACE read path
goal:
  ACE materializes only the bounded, correctly filtered Agents-tab prefix needed for the
  current viewport while preserving full-history and unsupported-query correctness
  fallbacks.
size: medium
proposed_by: bbugyi200.athena.sase-uv.8
bead: sase-uv.8
---

- **BEAD:**
  [sase-uv.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-uv/sase-uv.8.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-uv.8](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-uv.8.md)
- **COMMITS:**
  - [07bd0f5](https://github.com/sase-org/sase-core/commit/07bd0f589434f90c51faab4994c0ef0d4db1c31d)
    — feat(agent-scan): support windowed index reads

# Plan: Honour the AgentsViewport contract on the direct ACE read path

## Scope and provenance

Complete phase `viewport` of epic `sase-uv`, bead `sase-uv.8`. The governing design is
`plan:202608/ace_tui_responsiveness.md`; its evidence base is the research report
`research:202608/ace_refresh_loop_and_link_rail_regression/ace_refresh_loop_and_link_rail_regression.md`.
The phase is measurement-gated and crosses the Rust core boundary. Open `sase-core` with
`/sase_repo` and use only the path it prints.

The current tree has two load-bearing facts that the implementation must address:

- `AgentsViewport` already defines a prefix window
  (`start_row + visible_rows + prefetch_rows`, normally 120), but
  `DirectAgentsDataProvider.load_agents()` deletes both `search_query` and `viewport`.
- The live ACE loading path does not call `DirectAgentsDataProvider` or
  `make_agents_data_provider()` at all. Editing `_direct.py` alone would be a no-op.
  Restore provider plumbing at the real `load_agents_from_disk_with_state` boundary.

Do not resurrect the reverted daemon rollout. This is a bounded direct SQLite/index
read, with the existing source-scan path retained as the correctness fallback.

## Measurement gate

Before editing code, run `just install`; this workspace's first attempted fresh probe
could not import `sase_core_rs`. Then capture seven warm cached Tier-1
`load_tiered_agents(patch_snapshot=[])` samples and the current row count. Record the
median and compare it with the `sase-uv.1` baseline and the 300 ms budget.

The latest durable post-projection measurement on `sase-uv.7` is already a failed gate:
805 index records, list-shape warm-load median 670.54 ms. If the fresh combined-tree
measurement unexpectedly meets every epic budget, record the evidence on `sase-uv.8`,
make no product changes, run the required checks, clear/re-key any epic symbols, and
close the phase as unnecessary. Otherwise implement the bounded path below and report
before/after measurements on the bead.

## Correctness invariants

1. The viewport is a growing prefix, not an SQL offset. `start_row` contributes to
   `requested_limit`; the provider returns rows from the beginning through that limit so
   selection restoration and parent/child adjacency remain stable.
2. Active rows are never hidden merely to satisfy the requested limit. Return all
   matching active records, even if they exceed the limit, then fill the remaining
   budget from recent completed records.
3. Full-history actions, index-missing/schema-bypass fallback scans, and any search that
   cannot be represented exactly by indexed scalar data retain the existing unbounded
   behavior. A slow correct fallback is preferable to a fast incomplete result.
4. The existing Python query evaluator remains the final authority. Rust pushdown may
   narrow candidates only when parity is proved; Python reapplies the query after
   artifact and non-index sources have been combined.
5. Record limits are not row limits: one workflow record can produce a parent and many
   step rows. Bound record materialization in core, then normalize/order/filter in
   Python and cap the returned provider rows to the requested prefix.
6. Never perform provider reads, detail hydration, filesystem walks, or subprocess work
   on the Textual event loop or serial message pump. Preserve the current pump-free,
   last-request-wins refresh path and re-capture query/selection/viewport state after
   every await.

## Part 1: select the window before record materialization in `sase-core`

Extend `AgentArtifactIndexQueryWire` additively; all new fields default to today's
unbounded behavior so every non-ACE caller stays byte-for-byte compatible.

1. Add an optional TUI window limit and a structured, serde-compatible candidate-filter
   wire. Keep parsing of the user-facing agent query in Python; send only a normalized
   AST made from exact index-safe atoms and boolean nodes. Do not add a second text
   query parser in Rust.
2. Add a scalar candidate row containing the denormalized columns needed for visibility,
   query parity, and display ordering (`artifact_dir`, active/completed markers,
   project/name/CL/model/provider/type/status, start/finish/timestamp, hidden, and any
   other field justified by parity tests). Selecting these rows must not read or decode
   `record_json`.
3. For a windowed query, select visible active and recent-completed scalar candidates,
   apply the exact candidate filter, preserve every matching active record, fill the
   remaining requested prefix from completed candidates, and order candidates by the
   same top-level recency semantics used by `_agent_ordering.sort_and_reorder`.
4. Only after the candidate artifact dirs have been chosen, fetch and deserialize their
   `record_json` values, apply the existing list projection, and build clan context.
   Keep the current `select_records` path for default/unbounded queries and revalidation
   behavior. Cached reads remain read-only; stale-row repair must not be moved onto the
   cached viewport path.
5. Expose enough result metadata to distinguish a bounded prefix from complete visible
   history (requested limit, returned candidate count, and truncation/has-more state).
   Do not overload `complete_history` or trigger a Tier-2 reconcile merely because a
   valid viewport is intentionally bounded.
6. Add Rust unit tests proving:
   - default queries serialize and behave exactly as before;
   - a windowed query decodes `record_json` only for selected candidates;
   - active rows survive a smaller requested limit;
   - completed fill/order is deterministic;
   - candidate-filter AND/OR/NOT semantics match the declared exact subset;
   - list projection and clan context remain correct for the bounded result;
   - truncation/has-more metadata is accurate.

## Part 2: compile only safe query pushdown in Python

1. Mirror the additive core wire and metadata in `src/sase/core/agent_scan_wire_*`, the
   facade, and conversion tests without bumping the agent-scan schema solely for
   optional defaulted fields.
2. Add a compiler from the existing `sase.ace.agent_query` AST to the candidate-filter
   wire. Maintain an explicit support matrix with parity tests against real `Agent`
   evaluation. Scalar metadata such as project/CL/name/model/provider/type/age may be
   pushed only where the index representation is demonstrably exact. Resolve project
   display-name aliases before emitting project predicates.
3. Treat bare/text content search and state derived outside the index (`tribe`, pinned,
   derived hidden/attention/source/needs, or any other unproved atom) as not
   window-safe. Boolean expressions containing an unsupported branch are not window-safe
   as a whole (especially `OR` and `NOT`). They use the old unbounded load and the
   existing Python evaluator. Exact conjuncts may still be used as conservative
   prefilters only if tests prove they cannot remove a true match.
4. Extend `load_tiered_agents` / `_agent_loader_artifacts` with optional query-plan and
   requested-limit arguments. Empty queries and fully index-safe queries use the
   two-stage core window; full history, fallbacks, and unsafe queries do not.
5. After combining indexed records with ProjectSpec/Patch-derived agents, keep the
   existing normalization and exact Python query evaluation, then return the requested
   prefix. This ensures non-index active rows participate in ordering and can displace,
   but cannot be displaced by, lower-ranked artifact candidates.

## Part 3: make `DirectAgentsDataProvider` the live direct path

1. Route `load_agents_from_disk_with_state` through
   `make_agents_data_provider()`/`DirectAgentsDataProvider`. Add `search_query`,
   `viewport`, and an injectable provider seam for tests. Carry the provider snapshot on
   `_AgentDiskLoadResult` and app state.
2. In `DirectAgentsDataProvider.load_agents`, pass the compiled query/window into
   `load_tiered_agents`; populate `agent_snapshot` metadata with the raw query,
   requested limit, returned count/has-more state, and direct provider identity. A
   bounded result is a complete replacement of the current prefix, not a daemon page.
3. Restore a current app-to-viewport helper based on `current_idx` (or
   `_agents_last_idx` off-tab) and clamped terminal height. Use the existing defaults of
   roughly one visible screen plus two screens of prefetch. Capture it with the query
   before dispatching the worker.
4. Resolve the one-time current-project query seed before issuing the first bounded
   provider read. After each await, re-read the current query and selection. If the
   request key changed in flight, do not install a stale bounded snapshot; schedule the
   existing coalesced trailing refresh.
5. Keep cached rows visible immediately when the query changes, then schedule a
   pump-free provider refresh. Update the current test that asserts query edits never
   refresh: bounded provider search requires a refresh to find matches outside the
   loaded prefix.
6. When navigation enters the final prefetch band, coalesce one background prefix
   expansion. Do not reload per keystroke. Track the last requested/returned limit so
   the watcher is O(1), and stop expanding when the provider says there are no more
   candidates.
7. Prevent the incomplete-history merge from reattaching an old 400-row cache to an
   intentionally bounded viewport snapshot. Preserve the separate revived-row and
   explicit full-history guarantees, and ensure artifact deltas still patch the loaded
   prefix without forcing a broad rebuild.

## Part 4: lazy selected-row surfaces and regression coverage

The projection phase already introduced selected-row hydration for heavy output and
linked-repo details. Reuse it; do not add a second detail cache.

1. Audit detail, log, and relation call sites reached by a refresh. Loading a viewport
   must not hydrate every record or build per-row detail/log/relations. Selected-row
   changes may use the existing off-thread hydration/debounce paths.
2. Add provider tests for default viewport metadata, query/window forwarding,
   full-history bypass, unsafe-query fallback, and bounded direct snapshots.
3. Add loader tests comparing bounded and unbounded fixtures: the bounded result must
   equal the exact prefix of the fully normalized/filtered ordering, including workflow
   parents/steps, family follow-ups, ProjectSpec agents, active-row overflow, grouping
   inputs, dismissed rows, and project-display-name query aliases.
4. Add Textual tests for search edits (instant cached refilter plus coalesced reload),
   stale in-flight query/viewport rejection, near-end prefetch expansion, stable
   selection across expansion, no per-key refresh storm, and no eager detail hydration.
5. Add a performance regression test/bench that uses a large synthetic index and asserts
   the default direct read materializes at most the active floor plus the requested
   viewport budget, rather than all recent records.

## Verification and handoff

1. In `sase-core`, run focused agent-index/core-Python tests, then
   `cargo test -p sase_core -p sase_core_py` and its repository check command.
2. Rebuild the editable extension with `just install`, run focused Python/TUI tests,
   then run `just check` in this repo. If selection escalates or is unusual, use
   `/sase_monitor` for `just check-full` as required by the project instructions.
3. Repeat the seven-sample warm cached Tier-1 measurement with the default viewport and
   record before/after median, record count, materialized Agent count, and whether the
   300 ms budget is met. Also summarize current startup/load-kind telemetry without
   claiming quiet-host data as a causal win.
4. Run `sase bead epic-symbols sase-uv.8`. Resolve every remaining symbol or re-key its
   Justfile line to `sase-uv`/a still-open later bead before closing.
5. Close only `sase-uv.8` with
   `sase bead close sase-uv.8 --note "<what was implemented and verified>"`. Do not
   close `sase-uv` or any ancestor. Record out-of-scope discoveries only as
   `PROPOSED FOLLOW-UP:` notes on this phase.

## Out of scope

- Reintroducing a daemon-backed provider or long-lived read-model service.
- Editing SASE memory.
- Automatic pruning/vacuuming of the user's live state.
- Making an unsupported query appear bounded by silently searching only the loaded
  prefix.
- Ratcheting or releasing the pinned core revision unless the normal epic landing flow
  explicitly assigns it; if this phase adds a new binding, leave the required
  `PROPOSED FOLLOW-UP:` note for the land agent.
