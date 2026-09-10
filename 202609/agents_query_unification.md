---
tier: epic
title: Unify the Agents tab query language with the Artifacts Agent pane
goal: "The top-level Agents tab filters with the same boolean query-profile dialect,
  Rust-backed evaluation, and FilterBar editing chrome as the Artifacts Agent pane, with
  zero idle screen-space cost and no measurable performance regression.

  "
phases:
  - id: shared-profile
    title: Shared agents-live query profile and row adapter
    depends_on: []
    size: medium
    description:
      "shared-profile: factor shared agent query field specs out of the Artifacts agents
      schema, add the agents-live boolean profile and the live-row query adapter, and
      pin Python/Rust conformance goldens for the new dialect."
  - id: live-engine
    title: Rust-backed committed-query engine behind a sunset flag
    depends_on:
      - shared-profile
    size: medium
    description:
      "live-engine: swap the Agents tab committed-query parse/evaluate path to the
      agents-live profile behind a new sunset feature flag, using an off-thread Rust
      corpus index, tree-preserving match masks, and last-good error handling."
  - id: consumers-and-pushdown
    title: Load-path pushdown parity and secondary query consumers
    depends_on:
      - live-engine
    size: medium
    description:
      "consumers-and-pushdown: compile the new dialect into the existing Rust
      candidate-filter pushdown with window-safety parity, and migrate the machines-pane
      writer, project seeding, unread-jump, neighbor, and prospective-clan consumers to
      the shared match-mask facade."
  - id: filter-bar-ui
    title: Auto-hiding FilterBar chrome on the Agents tab
    depends_on:
      - live-engine
      - consumers-and-pushdown
    size: medium
    description:
      "filter-bar-ui: replace the query-edit modal with an auto-hiding FilterBar plus a
      highlighted canonical query readout in the existing info panel, wiring live
      preview, completions, saved slots, query history, and the keybinding, footer, and
      help-modal updates."
  - id: docs-and-sweep
    title: Documentation rewrite and verification sweep
    depends_on:
      - filter-bar-ui
    size: small
    description:
      "docs-and-sweep: rewrite the user docs for the unified dialect including the
      legacy-token migration table, refresh the help-modal syntax section and
      configuration reference, and run the final perf and visual snapshot sweep."
proposed_by: bbugyi200.athena.0iy
create_time: 2026-09-10 18:01:44
status: wip
---

- **PROMPT:**
  [prompts/202609/agents_query_unification.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_query_unification.md)

# Plan: Unify the Agents tab query language with the Artifacts Agent pane

## Why

ACE currently ships **three** query dialects. The Patches dialect and the profile-driven
boolean dialect (Artifacts → Agent pane, shared with `sase agent search`) are
first-class: persistent syntax-highlighted filter rows, per-keystroke preview against
the loaded snapshot, Tab completion fed by profile schema + observed facets, inline
parse errors that never clobber the last good result, saved-query slots, history, and
Rust-backed matching (`compile_corpus_with_profile` / `evaluate_many`).

The top-level Agents tab has none of that. Its filter is a bespoke dialect
(`src/sase/ace/agent_query/`: closed key allowlist, `age>=2h` duration operators,
pure-Python evaluation) edited **blind** in a `QueryEditModal` reachable only via the
leader chord `,/` (`src/sase/ace/tui/actions/agents/_filter_actions.py:23`,
`src/sase/ace/tui/modals/query_edit_modal.py`). No preview, no completion, no match
count, no highlighting (the package's `highlighting.py` has zero consumers), errors
surface only as a transient toast at render time, and users must learn a second field
vocabulary (`type:run`, `age>2h`) that conflicts with the Agent pane's (`kind:`,
`since:`/`until:`, `min:`/`max:`).

This epic retires the bespoke dialect and gives the Agents tab the same language and
editing chrome as the Artifacts Agent pane, tuned for the live/operational row set, with
**zero idle screen-space cost** and **no perf regression** (the Agents tab is the
hottest surface in the app; see the perf contract below).

## Target design

### One dialect, two profiles

The Artifacts Agent pane's dialect is declared as an `ArtifactQuerySchema`
(`src/sase/ace/query_profile/profiles/_agents.py`, pane_id `agents`, `boolean=True`) and
compiled into a digested profile that drives parsing, canonicalization, completion,
highlighting, and Rust evaluation generically. We add a sibling **`agents-live`**
profile for the live tab. Same grammar (`AND`/`OR`/`NOT`/`!`, parentheses, implicit AND,
bare/quoted/`c"..."` strings, `key:value` terms), same shared field names with identical
semantics, plus live-only operational fields. **No Rust changes are needed**: profiles
compile through the existing generic bindings, and the Python reference evaluator stays
parity-test-only.

Field plan for `agents_live_query_schema()`:

- **Shared with the catalog profile** (same key, same value kind, same semantics):
  `name`, `family`, `clan`, `project` (exact-match strings); `kind` (enum:
  agent/member/family/clan/workflow/workflow-child); `role`, `workflow`, `model`
  (substring strings); `provider` (enum + observed facets); `status` (enum; static
  values are the live status vocabulary, merged with observed facets); `attempt` (int);
  `hidden`, `attention`, `retry` (bools); `since`/`until`/`after`/`before` (date bounds,
  `Nh/Nd/Nw/Nm`, `today`, `YYYY-MM-DD`); `min`/`max` (runtime duration bounds); `text`
  (search-only).
- **Live-only**: `cl` (substring, searchable), `machine` (exact-match, multi-value:
  canonical alias — local rows emit `here` — plus raw hostname, so the machines pane's
  programmatic terms keep working), `tribe` (exact-match string with observed facets;
  live tribes are user-defined names, not the catalog enum), `pinned`, `unread` (bools),
  `needs` (enum: `input`), `source` (enum: `axe`/`manual`).
- **Deliberately absent** (archive-only concepts): `state`, `dismissed`, `revivable`,
  `historically_viewable`, `durably_revivable`, `restartable`, `linked`, `relation`,
  `artifact`, `label`. Also `predicates=()` and `any_special=False` — do not copy the
  catalog wart where `!!!`/`@@@`/`$$$` parse but can never match. No sigils, no macros,
  no host `limit:` token (the live tab keeps its own tiered-loading window contract).

To keep the two profiles from drifting, factor the shared `QueryFieldSpec` groups out of
`_agents.py` into a private helper module in `src/sase/ace/query_profile/profiles/`
consumed by both schemas; the catalog schema's compiled output (digest, canonical
ordering, wire payload) must be byte-identical before and after the refactor.

Legacy → unified token mapping (drives inline error hints and docs):

| Legacy (`agent_query`)                                                                                                            | Unified (`agents-live`)                    |
| --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| `type:workflow` / `type:run` / `type:running`                                                                                     | `kind:workflow` / `kind:agent`             |
| `age>2h`, `age>=2h`                                                                                                               | `until:2h` (started at or before 2h ago)   |
| `age<5m`, `age:5m`                                                                                                                | `since:5m` (started at or after 5m ago)    |
| `status:foo` (substring)                                                                                                          | `status:FOO` (enum, completion-assisted)   |
| `project:foo` (substring)                                                                                                         | `project:foo` (exact; completion-assisted) |
| `tribe:` (bare, "any tribe")                                                                                                      | no direct equivalent; OR explicit tribes   |
| everything else (`cl:`, `machine:`, `pinned:`, `needs:input`, `source:axe`, `text:`, booleans, `AND`/`OR`/`NOT`, parens, quoting) | unchanged spelling                         |

### Evaluation engine (perf-critical)

Reuse the proven Artifacts machinery end to end:

- **Row projection**: a new `agent_live_query_entry()` adapter (sibling of
  `src/sase/ace/tui/widgets/artifacts/query_rows.py:agent_query_entry`) projects the
  live `Agent` model (`src/sase/ace/tui/models/agent.py`) into wire rows.
  Free-text/`text:` corpus = the existing metadata haystack (`cl_name`, `display_name`,
  `agent_name`, `status`) plus transcript content served from the existing
  `AgentContentSearchCache` (`src/sase/ace/tui/models/agent_content_search.py` —
  `(path, mtime_ns)` keying, 512 KB/file cap). Stable row id = the agent's canonical
  name.
- **Index**: build one `ArtifactQueryIndex` per loaded-agents snapshot generation via
  `compile_artifact_query_index` (`src/sase/core/query_profile_corpus_facade.py`), **off
  the event loop**, integrated with the existing background content-index rebuild in
  `src/sase/ace/tui/actions/agents/_loading_filter.py` (coalesced, last-request-wins,
  same worker discipline). Keystroke paths never stat or read files.
- **Matching**: `evaluate_artifact_query_many` returns a boolean mask; convert to a
  `frozenset` of matching agent names and apply through the existing tree-preserving
  `filter_tree_rows` (`src/sase/ace/tui/models/_agent_tree.py:544`) so parent/descendant
  retention semantics are unchanged. Cache results with the exact
  `(pane_id, generation, profile_digest, canonical_query)` key shape via an
  `ArtifactQuerySession` instance
  (`src/sase/ace/tui/widgets/artifacts/query_session.py`) owned by the Agents tab.
- **Mask facade**: expose the committed evaluation as one small object (canonical query,
  generation, matching-name set). The display filter and every secondary consumer read
  from it — one parse path, one evaluation, no per-row re-evaluation loops.
- **Load-path pushdown**: today `src/sase/ace/agent_query/pushdown.py` compiles the
  legacy AST into the Rust sqlite candidate filter (`query_agent_artifact_index`;
  pushable keys `cl`, `model`, `provider`, `project`, `type`) and decides `window_safe`.
  Replace it with a compiler from the profile-parsed AST covering the same keys (`kind`
  maps onto the indexed `type` column) with identical AND/OR/NOT composition and
  identical `window_safe` outcomes for equivalent queries, so windowed loading neither
  regresses nor over-prunes.

Perf contract (binding for every phase; the rules live in the `tui_perf.md` reference
memory, which each phase worker must read via `/sase_memory_read` before touching these
paths):

- Keystroke and render paths stay read-only: no disk I/O, no stat/glob, no subprocess.
  Index builds and transcript reads happen only in the existing off-thread workers.
- Route re-filters through `_refilter_agents()` / `_schedule_agents_async_refresh()`; do
  not add refresh code paths. Preserve the existing coalescing guards and the existing
  fast-path degradation behavior for active queries (incremental-display fallbacks stay
  exactly as conservative as today — improving them is out of scope).
- Per-keystroke synchronous work is limited to string-level
  `canonical_query_for_profile()` (same as the Artifacts panes); row matching is
  off-thread with a single pending request (last-request-wins).
- Gates: `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` p95 < 16 ms on the Agents tab
  with an active query, before/after capture per `docs/perf_runbook.md`; a quiet
  auto-refresh tick with an active query must reload no surfaces and open near-zero
  content files (`SASE_TUI_TRACE=1`).

### UI: zero idle cost, same visual grammar

- **Idle, no query**: nothing new on screen. The bar is not mounted visible; the tab
  looks exactly as today.
- **Idle, active query**: no extra row. The existing one-line `AgentInfoPanel` filter
  readout (`src/sase/ace/tui/widgets/agent_info_panel.py:414`) is upgraded to render the
  **canonical** query with the shared profile syntax highlighting
  (`profile_highlighting.highlight_query`) plus a match count (`N/M` matched/loaded).
  Clicking it opens the editor.
- **Editing**: an `AgentsFilterBar(FilterBar)` subclass
  (`src/sase/ace/tui/widgets/filter_bar.py` base; the Artifacts `AgentFilterBar` in
  `widgets/artifacts/agents_query.py:53` is the exemplar) appears above
  `#agents-content` in `#agents-view` — its own row, not inside
  `Horizontal#agents-header`, so the fleet-status line is unaffected. It uses the
  standard 3-row FilterBar look with the Agents-tab gold accent (`#FFD700`) instead of
  the Artifacts blue, `FORWARD_ARTIFACTS_PAGING = False` (no `limit:` paging on this
  tab), and disappears again on commit/dismiss. The bar must be excluded from the
  dynamic tribe-panel mount/unmount sweep in `actions/agents/_display_panel_widgets.py`,
  and its completion-overlay offset needs an Agents-tab styling variant in
  `styles.tcss`.
- **Behavior** (identical to Artifacts panes): typing previews against the loaded
  snapshot per keystroke (worker-coalesced, no timer on the list; the detail panel keeps
  its own debouncer); `Enter` commits, records history, and schedules the background
  reload; `Escape` restores the pre-session query and result; invalid input shows the
  parse error bold-red in the status lane and never clobbers the last good result; `Tab`
  accepts completions (profile keys
  - static values + observed facets); `#N`/`#` saved-query slots via
    `src/sase/ace/saved_queries.py` under a distinct `agents-live` namespace; `^`
    history navigation while the bar is open.
- **Delight detail**: when a committed or typed query fails to parse _and_ the failing
  token matches a legacy spelling from the mapping table (`age>2h`, `type:run`, …), the
  error message appends the unified replacement, e.g.
  `unknown key "age" — try until:2h`.
- **Keybindings**: bare `/` stays reserved for the detail-pane vim search (documented
  reservation). `,/` keeps working and now opens the bar. Add a direct
  `agents_filters: "f"` binding matching the `*_filters: "f"` convention of every
  Artifacts pane, after confirming `f` is unbound on the Agents tab; if it is taken,
  fall back to `,/`-only and say so in docs. Keymap changes must land in
  `src/sase/default_config.yml`, the footer must follow the conditional-keymap
  convention, and the help modal must be updated (see `src/sase/ace/CLAUDE.md` for all
  three).

### Rollout: one sunset feature flag

Phase live-engine creates one flag with `sase flag new agents_unified_query -k sunset`
(never hand-add the registry entry):

- `--when-enabled`: "The Agents tab parses, evaluates, and edits its filter with the
  shared agents-live boolean query profile through the Rust corpus engine and the
  FilterBar chrome."
- `--when-disabled`: "The Agents tab keeps the legacy agent_query dialect, the
  QueryEditModal editor, and the pure-Python evaluator."
- `--remove-when`: "The unified dialect has shipped as the default for a full release
  with no rollback need, and the legacy agent_query package has no remaining production
  callers."

Sunset default is On (new behavior live immediately), with the whole legacy branch —
`src/sase/ace/agent_query/`, `QueryEditModal`, the Python evaluator — reachable when Off
as the rollback lever. Every phase tests both flag states. Deleting the Off branch is
the flag bead's job later, **not** part of this epic. Nothing persists old-dialect
queries across sessions (the in-memory query is seeded at most with `project:<name>`,
which parses identically in the new dialect), so no data migration is needed.

## Phase: Shared agents-live query profile and row adapter

Pure library work; no user-visible change.

1. Factor the shared field-spec groups out of
   `src/sase/ace/query_profile/profiles/_agents.py` into a private shared module;
   rebuild `agents_query_schema()` from it and prove the compiled catalog profile digest
   and wire payload are unchanged (regression test that pins the digest before/after).
2. Add `agents_live_query_schema()` (new `_agents_live.py`) exactly per the field plan
   above; register pane_id `agents-live` in
   `src/sase/ace/query_profile/pane_registry.py` the same way the Procs builtin is
   registered.
3. Add `agent_live_query_entry()` — the live-row wire projection — beside the Agents tab
   models (it consumes `Agent` and `AgentContentSearchCache`; do not put TUI imports
   into `query_profile`). Multi-value `machine`, epoch date fields from start/finish
   times, runtime seconds for `min`/`max`, `needs`/`source`/`unread`/`pinned`
   derivations mirroring the legacy evaluator's semantics
   (`src/sase/ace/agent_query/evaluator.py` is the behavioral reference).
4. Conformance: extend
   `tests/ace/tui/artifacts_contract/goldens/query/profile_cases.json` and the
   registration in `tests/ace/tui/artifacts_contract/test_query_conformance.py` with
   `agents-live` cases covering every field kind, the boolean grammar, quoting/`c"..."`,
   date/duration bounds, and rejection of the removed legacy spellings (`age`, `type:`)
   — Python reference and Rust must agree byte-for-byte on canonicalization and
   row-for-row on matching.
5. Unit tests: schema shape pinned like `tests/test_query_profile_agents.py` does for
   the catalog (no sigils/macros/predicates, field kinds, searchable set = {name, cl,
   text}); adapter tests for every derived field including the machine alias/hostname
   pair and transcript-corpus inclusion.

## Phase: Rust-backed committed-query engine behind a sunset flag

Swaps what a committed query _means_ on the Agents tab; the editing surface is still the
modal (its validator/hint switch with the flag), replaced in the filter-bar-ui phase.

1. Create the `agents_unified_query` sunset flag via `sase flag new` with the three
   sentences above; wire the branch point where the committed query is parsed and
   applied.
2. On-flag committed path: parse/canonicalize with
   `parse_query_for_profile`/`canonical_query_for_profile` against the `agents-live`
   compiled profile; build the `ArtifactQueryIndex` off-thread in the existing
   content-index rebuild worker (`actions/agents/_loading_filter.py`), keyed by snapshot
   generation; evaluate through an Agents-tab `ArtifactQuerySession`; produce the mask
   facade; apply via `filter_tree_rows` in both the UI-thread finalize
   (`actions/agents/_loading_finalize.py:311`) and the worker-thread compute finalize
   (`actions/agents/_loading_compute_finalize.py:105`).
3. Error semantics: a query that fails to parse filters nothing, keeps the last-good
   result, and surfaces the message (toast for now, status-lane in the filter-bar-ui
   phase) — including the legacy-token replacement hint.
4. Load path interim: with the flag On, the legacy pushdown compiler must not see
   new-dialect strings; until the consumers-and-pushdown phase lands, a non-empty query
   takes the (already common today) full-history load path. Secondary consumers (unread
   jumps, neighbors, prospective clans) and the machines-pane/seed writers keep working:
   `machine:`/`project:` terms parse identically in both dialects, and consumers are
   pointed at the mask facade here if trivially possible, else in the next phase.
5. Tests both flag states: committed filtering end-to-end on a synthetic fleet (tree
   retention, hidden/pinned/unread/needs/source/machine/date/ runtime fields), error
   fallback, AST/result cache keying across snapshot generations, and an Off-state sweep
   proving the legacy path is untouched.
6. Perf: no new event-loop work besides string-level canonicalization; bench capture per
   the perf contract.

## Phase: Load-path pushdown parity and secondary query consumers

1. New pushdown compiler from the profile AST to the existing `candidate_filter` wire
   (`src/sase/core/agent_scan_wire_records.py`): pushable keys `cl`, `model`,
   `provider`, `project`, and `kind`→indexed `type`; identical AND/OR/NOT composition
   and `window_safe` decisions to `src/sase/ace/agent_query/pushdown.py` for equivalent
   queries (parity test over a table of equivalent legacy/unified query pairs). Wire it
   into `models/agent_loader.py:load_tiered_agents` on-flag, restoring windowed loading
   for pushdown-safe queries.
2. Migrate the remaining query writers and consumers to the unified path on-flag:
   `modals/machines_pane.py:action_show_agents` (compose terms via the profile
   canonicalizer), `actions/agents/_search_query_seed.py` (seeded `project:` term),
   `actions/agents/_unread_jump_candidates.py`, `actions/agents/_neighbors.py`,
   `actions/agents/_prospective_clan.py` (all through the mask facade; no direct
   evaluator calls left outside the engine).
3. Verify the fast-path degradation matrix is unchanged relative to master
   (`actions/agents/_display.py` active-search fallback, `_display_panel_patches.py`,
   `_loading_refresh_delta.py`): same fallback reasons fire for the same query shapes.
4. Tests: pushdown parity table, windowed-load behavior with pushdown-safe queries,
   consumer behavior on-flag and off-flag, machines-pane jump end-to-end.

## Phase: Auto-hiding FilterBar chrome on the Agents tab

1. `AgentsFilterBar(FilterBar)` configured from the compiled `agents-live` profile
   (completions, hints, negatable keys derive automatically);
   `FORWARD_ARTIFACTS_PAGING = False`; gold accent; mounted hidden in `#agents-view`
   above `#agents-content`; excluded from the tribe-panel widget sweep;
   completion-overlay offset variant in `styles.tcss`.
2. Session flow per the UI design: open via `,/`, `f` (availability-checked), or
   clicking the info-panel readout; per-keystroke preview through the engine session
   (worker-coalesced); Enter commits + records history + schedules
   `_schedule_agents_async_refresh(source="filter")`; Escape restores; parse errors
   render in the status lane with the legacy-token hint; bar hides on commit/dismiss.
3. Info-panel idle readout: canonical query with
   `profile_highlighting.highlight_query` + `N/M` match count, replacing the plain gold
   text; clickable; `seeded` marker preserved.
4. Saved slots (`#N` grammar, `agents-live` namespace in
   `src/sase/ace/saved_queries.py`) and `^` history while the bar is open, generalizing
   `actions/artifacts_query_history.py` only as far as needed.
5. Retire the QueryEditModal entry point on-flag (`_edit_agent_search_query` routes to
   the bar); Off-flag keeps the modal exactly as today.
6. Keymap/footer/help: `agents_filters` binding in `src/sase/default_config.yml` (only
   if `f` is confirmed free on the Agents tab), footer conditional-keymap rules and help
   modal per `src/sase/ace/CLAUDE.md`, help-modal Agent Query Syntax content switched to
   the unified dialect on-flag.
7. Tests: widget interaction tests (open/preview/commit/escape/error/
   completion/saved-slot/history), visual snapshot goldens for the three states (hidden,
   idle-with-query readout, editing), both flag states, and the completion-context
   behavior matching the Artifacts Agent bar (the known flat-lexer completion limitation
   is acceptable parity, not a bug to fix here).

## Phase: Documentation rewrite and verification sweep

1. `docs/ace.md`: rewrite the Agent Search section for the unified dialect (fields,
   grammar, keys, saved slots, history, the legacy→unified mapping table verbatim);
   update the Leader Mode row and the filtering overview section that contrasts the
   panes.
2. `docs/query_language.md`: the "three dialects" note becomes two (Patch + the shared
   agent boolean dialect, with the live tab's operational field additions called out);
   `docs/configuration.md` keybinding/config updates (`seed_agents_query` wording, new
   binding).
3. Help modal syntax section and footer strings final pass; confirm the 57-char box
   conventions.
4. Final sweep: full conformance goldens, visual snapshots, both-flag-state test suites,
   and the before/after perf capture (j/k bench p95 < 16 ms with an active query;
   idle-tick trace shows no query-induced file opens), recorded in the phase notes.

## Non-goals

- No changes to the Artifacts Agent pane dialect, `sase agent search`, or the
  Patches/Stitches/Beads/Plans/Files dialects (beyond the drift-proof refactor of shared
  field specs).
- No `limit:` host cap on the live Agents tab; its tiered-loading window contract stays
  authoritative.
- No improvement of the active-query fast-path degradations (incremental display, row
  patching, delta refresh) beyond exact parity with today.
- No deletion of the legacy `agent_query` package or the Off branch; that belongs to the
  `agents_unified_query` flag bead's removal.
- No Rust core (sase-core) changes; the profile system is generic and the
  candidate-filter wire is reused as-is.

## Working notes for phase workers

- Read the `tui_perf.md` reference memory (via `/sase_memory_read`) before touching any
  load, refresh, keystroke, or render path in this epic, and `lint_and_test.md` before
  finishing any phase.
- The legacy evaluator (`src/sase/ace/agent_query/evaluator.py`) is the behavioral
  reference for live-only field semantics; when unified behavior intentionally diverges
  (exact `project:`, enum `status:`, removed `age`/ `type:`/empty-`tribe:`), the
  divergence is listed in the mapping table and must be covered by tests, not silently
  changed further.
- Keymap or leader-key changes must be mirrored in `src/sase/default_config.yml`.
