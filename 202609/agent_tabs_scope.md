---
tier: epic
title: 'Agent tabs: tab index, active-tab scope, keys, and cross-tab navigation'
goal: 'Behind the new `agent_tabs` beta flag, the Agents tab shows one agent tab at
  a time. A per-root tab index and ordered catalog come from the sase-core catalog.
  The active tab re-scopes a cached, tab-independent query result without I/O. Folds,
  sticky panels, and selection memory are kept per tab. The active tab persists across
  restarts. `[`/`]` cycle tabs, every cross-tab jump switches tabs first, and bulk
  confirmations name their scope. A minimal strip makes the scope visible. With the
  flag off, the TUI is unchanged.

  '
phases:
- id: tab-foundation
  title: Flag, ace.agent_tabs config, machine mode, and the tab index model
  depends_on: []
  size: medium
  description: 'tab-foundation: create the agent_tabs beta flag with sase flag new
    and a single check helper; add the ace.agent_tabs typed reader, schema, defaults,
    and docs; add the token-cached machine-mode accessor; extend the core adapter
    with typed tab keys and a batched catalog wrapper; give clan containers an agent_tab;
    build the pure, memoized per-root tab index model with tests. No UI wiring.'
- id: scope-stage
  title: Active-tab scope stage and tab-keyed panel state
  depends_on:
  - tab-foundation
  size: medium
  description: 'scope-stage: cache the tab-independent query result and add the active-tab
    scope stage in the inline and worker finalize paths; add the scope to PreparedApplySnapshot
    and the stale token; route every direct _agents mutation through the cached result;
    key the panel-index memo, AgentPanelFoldScope (fold persistence v3 to v4), session-sticky
    panels, and selection memory by tab scope.'
- id: tab-state-keys
  title: Tab switching, persistence, keys, minimal strip, and perf metric
  depends_on:
  - scope-stage
  size: medium
  description: 'tab-state-keys: add the synchronous tab switch with per-tab memory,
    startup selection, the emptied-tab latch, and the disappearance fallback; persist
    the active key off-thread; wire ]/[ next/prev_agents_tab and unbound pick_agents_tab
    through the full keymap surface, with legacy bracket yield; render a labels-only
    PanelTabStrip in #agents-header; add the tab-switch perf metric and AcePage state.'
- id: cross-tab-nav
  title: Switch-then-reveal for every cross-tab jump
  depends_on:
  - tab-state-keys
  size: medium
  description: 'cross-tab-nav: add one switch-then-reveal helper and route every agent-revealing
    entry point through it (shared reveal, Node Finder with off-tab chips, ,j/,J from
    the query result, relation jumps, notification jumps, link follow and trail, the
    Procs monitor jump, the run-log jump, Files open agent, revive select); record
    the tab in jump-back anchors.'
- id: scope-honesty
  title: Tab-scoped bulk confirmations, docs, and flag-on verification
  depends_on:
  - tab-state-keys
  size: medium
  description: 'scope-honesty: make bulk and cleanup confirmations name their scope
    (on <tab> or across all tabs) and flag marked agents on other tabs; document the
    keys and the flagged behavior in docs/ace.md; confirm that the flag-off goldens
    are unchanged and a live flag-on screenshot shows instant switching.'
proposed_by: bbugyi200.athena.sase-1bc.6
parent_bead: sase-1bc.6
create_time: 2026-09-27 13:46:02
status: wip
bead_id: sase-1bc.6.1
---

- **PROMPT:** [prompts/202609/agent_tabs_scope.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agent_tabs_scope.md)
- **PARENT:** [202609/agents_dynamic_tabs.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_dynamic_tabs.md)
- **BEAD:** [sase-1bc.6.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bc/sase-1bc.6.1.md)

# Plan: agent tabs, active-tab scope, keys, and cross-tab navigation

## Context

This plan implements phase `tab-scope` (bead `sase-1bc.6`) of epic `sase-1bc`. The
parent plan is `plan:202609/agents_dynamic_tabs.md`, and its **Design contract** is
binding: vocabulary, effective tab resolution, the tab catalog, the pipeline, the keys,
the configuration, and the flag scope. Every phase worker must read that contract (the
"Design contract" section and "Phase 6 — tab-scope") before starting. When code and
contract disagree, stop and record a `PROPOSED FOLLOW-UP:` note on your phase bead
rather than silently diverging.

What already exists:

- **sase-core.** The pinned `sase-core-revision.txt` already exposes these bindings, so
  no core change or pin bump is needed:
  - `canonicalize_agent_tab_name`;
  - `resolve_effective_agent_tab(root, machine_mode)`;
  - `build_agent_tab_catalog(roots, options)`.

  The catalog takes roots shaped
  `{"agent_tab": str|None, "owner": {"kind": "local"} | {"kind": "remote", "installation_id": str|None, "alias": str}}`
  and options shaped
  `{"machine_mode", "machine_order": [{"installation_id", "alias"}], "named_order": {name: int}}`.
  It returns index-aligned `keys` plus ordered `entries` (`key`, `kind`, `label`,
  `root_count`). Keys are
  `{"kind": "default" | "machine" | "unresolved_machine" | "named", ...}`. The default
  label is `main`, or `⌨ local` in machine mode.

- **Python adapter.** `src/sase/core/agent_tab.py` wraps only `canonicalize_agent_tab`.
- **Agent model.** `Agent.agent_tab` (`models/_agent_state_session.py`) is filled from
  meta, the wire, fleet rows, remote session containers, and provisional dispatch rows.
  Session follow-ups and clan joiners already copy the root's tab into their own meta.
  Synthetic clan containers (`models/_agent_tree_clan.py`) do **not** set `agent_tab`.
- **Card blocks.** They already moved to `(`/`)` (phase `card-block-keys`). The legacy
  bracket-override warning lives in `keymaps/registry.py` (about :428-461) as a local
  variable, with no accessor. `[`/`]` are currently a no-op on Agents, and
  `test_bracket_keys_do_nothing_on_agents` asserts that "until tab-scope binds them".

Code map (paths are under `src/sase/ace/tui/` unless noted; line numbers are
approximate):

- **Load and apply.**
  - `actions/agents/_loading_apply.py` (~:411-455) stores `_agents_with_children` and
    `_agents`, then calls `_select_finalize_plan`.
  - The inline refilter is `_refilter_agents` in `_loading_filter.py` (~:148-235 and
    :350).
  - The inline finalize is `finalize_agent_list` in `_loading_finalize.py` (:338). The
    query is installed at :215, :278, and :442. The panel keys are built at ~:532-545.
- **Worker.** `_compute_finalize_plan` is in `_loading_compute_finalize.py` (:325), and
  `PreparedFinalizeStaleToken` / `make_finalize_stale_token` are at :73-92 and :292.
  `PreparedApplySnapshot` is in `_loading_compute_types.py` (:78-122) and is captured in
  `_loading_apply_snapshot.py`. The stale compare is at ~:193-199.
- **Direct `_agents` mutation sites:**
  - `_dismiss_memory.py:204`;
  - `_kill_identity.py:227,235`;
  - `_named_proc_dismiss.py:101-121`;
  - `_loading_filter.py:350`.
- **Tree anchors.** `presentation_anchor_lookup` is in `models/_agent_tree_anchor.py`
  (:47), and `agent_is_tree_child` in `models/_agent_tree_fold.py`.
  `is_remote_fleet_agent` is at `actions/agents/_remote_lifecycle.py:13`. The fleet
  fields are `fleet_origin_alias` and `fleet_origin_installation_id`.
- **Panel state.**
  - `_agent_panel_index()` is in `actions/agents/_display.py` (:147-180; memo
    `(self._agents, merged, index)`).
  - `AgentPanelFoldScope` is in `models/agent_group_fold.py:42`.
  - Fold persistence (`SCHEMA_VERSION = 3`, file `ace_agents_fold_state.json`) is in
    `models/agent_fold_persistence.py`, with its TUI side in
    `actions/agents/_fold_persistence.py`.
  - `reconcile_panel_fold_registries` is in `_fold_scope.py:61`.
  - The session-sticky maps are in `_display_panel_collection.py`, and
    `_panel_selection_memory` in `_selection.py`.
  - State init is in `actions/_state_init_agents.py`.
- **Header.** `#agents-header` is composed at `_app_layout.py:131` and toggled by
  `_update_agents_header` (`actions/agents/_fleet_header.py`). `PanelTabStrip` is in
  `widgets/panel_tab_strip.py` (`set_tabs`, `set_active_tab`, and a `TabClicked`
  message).
- **Patterns to copy.**
  - Config reader: `agent_decks_settings.py`. Token cache: `models/tribe_display.py`.
  - Persistence: `models/agent_deck_persistence.py` plus
    `actions/agents/_deck_persistence.py`.
  - Perf: `util/perf.py` (`JKPerfTimer`) and the `perf_begin` pattern in
    `_panel_navigation.py:313-338`.
  - Flag check: `widgets/decks/final/flag.py`. Test toggle: `override_flags`.

Rules that bind every phase:

- **tui_perf rules** (read `sase memory read tui_perf.md`). A tab switch does no disk,
  network, or roster re-projection work. Re-read the active tab after every await.
  Refreshes go through the existing fast paths.
- **Flag gating.** With the flag off, every gated behavior is exactly today's behavior:
  the scope stage is the identity, no strip appears, and `[`/`]` do nothing on Agents.
  The gated behaviors are the scope, the strip, the `[`/`]` keys, cross-tab switching,
  and scope wording. Tests cover both flag states. Keep the Off branches explicit, so
  the epic's `finish` phase can delete them in one change.
- **Verification.** Run `sase tool run check` (never `check-full`). Golden runs go
  through `/sase_monitor` with targeted selectors, and every golden change is inspected.
- **symvision.** Symbols that a later phase of this sub-epic consumes, or that parent
  epic phases `tab-strip` / `layout-ladder` / `tab-moves` consume, may need an
  `--epic-symbol` Justfile line keyed to a still-open bead. Prefer real consumers, and
  re-key or remove any entry before your phase closes.

## Phase 1 — tab-foundation

1. **Flag.** Run `sase flag new agent_tabs -k beta` with these authored sentences:
   - `--when-enabled`: "The Agents tab shows one agent tab at a time with a tab strip,
     `[`/`]` tab cycling, and cross-tab navigation."
   - `--when-disabled`: "The Agents tab shows every agent in one roster with no tab
     strip, exactly as before agent tabs."
   - `--remove-when`: "The finish phase of epic sase-1bc deletes the Off branch."

   The parent plan authorizes this flag bead as epic scaffolding. Then:
   - paste the printed enum member and definition into
     `src/sase/feature_flags/registry.py`;
   - regenerate `src/sase/config/sase.schema.json` with
     `tools/sync_feature_flags_schema`;
   - add the row to the flag table in `docs/configuration.md`.

   Add one helper, `agent_tabs_enabled()` (`src/sase/ace/tui/agent_tabs_flag.py`, or
   next to the settings module), that lazily imports
   `current_flags().enabled(FeatureFlag.agent_tabs)`, as `widgets/decks/final/flag.py`
   does. Every gated site calls this helper.

2. **Config block `ace.agent_tabs`.** Add `src/sase/ace/tui/agent_tabs_settings.py`,
   modeled on `agent_decks_settings.py`:
   - a frozen `AgentTabsSettings` with `machine_tabs: Literal["auto","on","off"]`
     (default `auto`), `launch_from_view: bool` (default `true`), and
     `tabs: Mapping[str, AgentTabStyle]`;
   - `AgentTabStyle` has `color`, `icon`, `order: int | None`, and `description`;
   - tab names are canonicalized through `canonicalize_agent_tab`, and invalid names are
     dropped;
   - parsing fails open to defaults.

   Wire it in `actions/_state_init_late.py` beside `_agent_decks_settings`. Add the JSON
   schema under `ace.properties` (`additionalProperties: false`), the commented defaults
   block in `src/sase/default_config.yml`, and a `#### ace.agent_tabs` section plus an
   `### ace` table row in `docs/configuration.md`. Note that `launch_from_view` and the
   styling keys are consumed by later parent phases.

3. **Machine mode and ordering.** Add a token-cached accessor (the `tribe_display.py`
   `lru_cache` over `current_config_token()` pattern) returning `AgentTabsViewConfig`:
   - `machine_mode: bool`: `on` → true, `off` → false, and `auto` → true iff
     `load_dispatch_config().machines` is non-empty, quarantined records included;
   - `machine_order`: `(pinned_installation_id, alias)` pairs in
     `load_dispatch_config().machines` order, skipping records without a pinned id;
   - `pinned_by_alias`: a map used for the provisional-row fallback;
   - `named_order`: taken from `tabs.<name>.order`;
   - a hashable `token`.

   It is never computed from fleet refresh results. Tests that edit config call
   `clear_config_cache()`.

4. **Core adapter.** Extend `src/sase/core/agent_tab.py` with the following, keeping the
   module a thin wrapper:
   - an `AgentTabKey`, a frozen, hashable dataclass or `NamedTuple` (`kind`, `value`),
     with constructors for default, machine, unresolved machine, and named keys;
   - `agent_tab_key_token(key) -> str` and
     `parse_agent_tab_key_token(token) -> AgentTabKey | None`, for persistence
     (`default`, `machine:<id>`, `named:<name>`). Unresolved-machine keys have no token,
     so they are never persisted;
   - `build_agent_tab_catalog(roots, *, machine_mode, machine_order, named_order) -> AgentTabCatalog`,
     which converts the wire into `AgentTabKey`s and ordered
     `AgentTabCatalogEntry(key, kind, label, root_count)` records. It makes one binding
     call per roster.

5. **Clan container tab.** In `models/_agent_tree_clan.py`, set the synthetic
   container's `agent_tab` from its member rows, the same way `tribe` and fleet origin
   are aggregated. Joiners already copy `clan_tab`, so members agree. Take the
   declarer's (earliest) non-empty value. Check whether imported session containers
   (`models/_agent_imported_agent_session.py`) need the same inheritance from their
   anchor, and do it if so.

6. **Tab index model.** Add `src/sase/ace/tui/models/agent_tab_index.py` (pure, no I/O,
   no Textual):
   - `AgentTabIndex` exposes:
     - the ordered `catalog` entries;
     - `key_for(row) -> AgentTabKey`, which resolves child rows to their presentation
       anchor's key. It looks up by `id(row)` with a fallback by `row.identity`, so rows
       that the query or status overrides rebuild still resolve;
     - `has(key)` and `root_count(key)`;
     - a `signature` (the ordered `(key, label, root_count)` tuple) for later repaint
       gating.
   - `build_agent_tab_index(roster, view_config) -> AgentTabIndex`:
     - it walks `presentation_anchor_lookup(roster)`;
     - it collects each distinct top-level anchor once, `STARTING` roots included;
     - it derives each root's `{"agent_tab", "owner"}`. A row is remote iff
       `fleet_origin_alias` is set, with `installation_id` from
       `fleet_origin_installation_id` or else `pinned_by_alias[alias]`;
     - it makes one batched `build_agent_tab_catalog` call.
   - A memo helper keyed by `(id(roster), len(roster), view_config.token)` returns the
     cached index on a hit.

7. **Tests** (new `tests/ace/tui/models/test_agent_tab_index.py`, plus settings and
   adapter tests):
   - zero, one, two, and many tabs;
   - machine mode `on`, `off`, and `auto` (with and without machines);
   - children, session follow-ups, and clan members resolve to their root's tab;
   - clan container inheritance;
   - a provisional dispatch row and its authoritative remote row land on the same key;
   - an unknown installation id gives an unresolved key with no token;
   - token round-trip;
   - memo hit and miss;
   - config parse fail-open;
   - both flag states for `agent_tabs_enabled()`.

**Done when:** `sase tool run check` passes, the flag bead exists and its registry entry
is pasted, and nothing user-visible changed.

## Phase 2 — scope-stage

1. **Scope state.**
   - `AgentTabScope` is either an `AgentTabKey` or the `ALL_AGENT_TABS` sentinel
     (consumed later by `layout-ladder`).
   - App state (`_state_init_agents.py`) gains:
     - `_active_agent_tab: AgentTabKey`, defaulting to the default key;
     - `_agent_tab_index: AgentTabIndex | None`;
     - `_agents_query_result: list[Agent]`.
   - Add `_agent_tab_scope() -> AgentTabScope`. It returns the default key when the flag
     is off, so fold scopes stay stable when the flag is later turned on.
2. **Pipeline.** Implement the parent contract's pipeline in both paths:

   ```text
   folds → committed query → status overrides → cache → scope → selection restore → panel keys
   ```

   - **Inline** (`finalize_agent_list`): after the query and overrides, store the
     tab-independent rows in `_agents_query_result`. Then set
     `_agents = scope_agents_to_tab(result, index, scope)` before selection restore and
     `enumerate_panel_group_keys`.
   - **Worker** (`_compute_finalize_plan`): build or reuse the tab index from the plan's
     roster (`boundary.fold.unfiltered_agents`, which is the `_agents_with_children`
     equivalent) and apply the same scope. The plan carries both lists and the index.
     `_apply_finalize_plan` installs all three.
   - **Index refresh.** `_agent_tab_index` is rebuilt (memoized) whenever
     `_agents_with_children` changes: on load, on refilter, and on the mutation paths.
     It is computed off the UI thread in the worker plan and inline on the refilter
     path.
   - **Flag off.** `scope_agents_to_tab` returns its input unchanged and `_agents` is
     identical to today's.
   - `scope_agents_to_tab` keeps a child row iff its root's key matches. It is pure, so
     put it in `agent_tab_index.py`.

3. **Snapshot and stale token.** Add the active scope (as a hashable key) to
   `PreparedApplySnapshot`, captured in `_make_prepared_apply_snapshot`, and to
   `PreparedFinalizeStaleToken` / `make_finalize_stale_token`. A tab switch made while a
   worker plan is in flight then invalidates that plan.
4. **Mutation sites.** Every place that filters or replaces `_agents` directly must
   update `_agents_query_result` consistently, through one helper (for example
   `_remove_agents_from_views(identities)`, or re-scope after mutating the cached
   result):
   - dismiss memory;
   - kill identity, including its `project_clan_tree` re-projection;
   - named-proc dismiss;
   - the `_loading_filter.py:350` restore.

   Otherwise a later tab switch resurrects a removed row. Grep again for `._agents =`
   and in-place mutations before finishing.

5. **Tab-keyed panel state.**
   - **Panel index.** The `_agent_panel_index()` memo key gains the tab scope, so the
     memo becomes `(agents, merged, scope, index)`.
   - **Fold scope.** `AgentPanelFoldScope` gains `tab_scope: str`, the scope token, with
     `all` for `ALL_AGENT_TABS` and the default token by default. Every constructor site
     passes the current scope (`_fold_persistence.py`, `_fold_scope.py`,
     `_folding_panels.py`, `_loading_compute_finalize.py`, `_loading_finalize.py`, and
     the `enumerate_panel_group_keys` callers).
   - **Fold GC.** `reconcile_panel_fold_registries` prunes only scopes whose `tab_scope`
     equals the current scope, so switching tabs never garbage-collects another tab's
     folds.
   - **Fold persistence v4.**
     - `SCHEMA_VERSION = 4`. Scope objects gain a `"tab"` key.
     - v3 (and older) files decode with `tab_scope = "default"`.
     - Scopes with an unresolved-machine scope are not written.
     - Add decode tests for v1, v2, v3, and v4 plus an encode round-trip.
     - Note the rollback caveat in the module docstring: an older reader drops v4 folds
       and falls back to empty.
   - **Session-sticky panels.** `_session_mounted_panel_identities` and
     `_session_mounted_panel_backing` become per-scope. Keep a dict keyed by scope and
     select the active one, or key the entries by `(scope, PanelKey)`. The existing
     query-change clearing still clears every scope.
   - **Selection memory.** `_panel_selection_memory` becomes per-scope in the same way.
     `_sync_panel_group` prunes only the active scope's entries.
6. **Low-level re-scope.** Add `_rescope_agents_to_active_tab()`:
   1. apply the scope to the cached `_agents_query_result`;
   2. invalidate the panel cache;
   3. `_sync_panel_group`;
   4. reconcile fold registries for the new scope;
   5. refresh the display through the existing incremental or rebuild path.

   It does no I/O and no re-projection. Phase 3 builds the user-facing switch on top of
   it.

7. **Tests** (both flag states):
   - flag off: `_agents` identical to today on load, refilter, and worker-plan paths;
   - flag on, two tabs: `_agents` holds only the active tab's rows plus their children;
   - a query that empties the active tab leaves the index catalog unchanged;
   - `STARTING` roots count toward the catalog;
   - a stale worker plan is rejected after the scope changes;
   - folds on tab A survive a re-scope to B and back;
   - dismiss or kill followed by a re-scope does not resurrect the row;
   - fold persistence migration.

**Done when:** `sase tool run check` passes, and the targeted flag-off Agents goldens
(for example `tests/ace/tui/visual/*agents*`) pass unchanged through `/sase_monitor`.

## Phase 3 — tab-state-keys

1. **Switch.** Add `_switch_agents_tab(key, *, reason)`, a synchronous method on the
   Agents mixin:
   1. save the current tab's memory: the selected node identity, the focused panel key,
      and the scroll anchor (the first visible row identity or `scroll_y`);
   2. set `_active_agent_tab`;
   3. call `_rescope_agents_to_active_tab()`;
   4. restore the target tab's memory by identity, falling back to the nearest surviving
      row (index clamp), then the first row;
   5. schedule persistence.

   Other rules:
   - A no-op when the flag is off or the key equals the active key.
   - Re-read `current_tab` and the selection after any await in callers.
   - Update `_agents_last_identity` / `_agents_last_idx` so tab-return memory stays
     correct.

2. **Catalog maintenance.** After each index rebuild, `_reconcile_active_agent_tab()`
   applies these rules:
   - **Startup selection** (first catalog with the flag on):
     1. the persisted key, if that tab exists;
     2. otherwise the default tab, if it has roots;
     3. otherwise the tab holding the newest root that needs attention (stopped, failed,
        or unread);
     4. otherwise the first tab.

     If the persisted state loads after the first catalog and the user has not switched
     yet, apply it then.

   - **Latch.** If the active key existed in the previous catalog and now has no roots,
     keep it selected with an empty roster until the user navigates away. The catalog
     view the strip shows includes the latched key with count 0.
   - **Fallback.** If the active key is a machine key whose machine is no longer in
     machine mode or configuration (or machine mode turned off), switch to the default
     tab with a toast (`agent tab ⌨ apollo is gone; showing main`).

3. **Persistence.** Add `models/agent_tab_persistence.py`, modeled on
   `agent_deck_persistence.py`:
   - file `sase_home()/ace_agents_tab_state.json`, `schema_version: 1`, holding
     `{"active": <token>}`;
   - size-capped and fail-open;
   - atomic writes.

   The mixin loads off-thread from the startup loads (`_startup_loads_core.py`) and
   writes through a coalesced single writer task, flushed at quit like the deck state.
   Unresolved keys are never persisted.

4. **Keys.** Add `next_agents_tab` (`]`) and `prev_agents_tab` (`[`), both wrapping and
   both a no-op when the strip is hidden. Wire them through the full surface:
   - `default_config.yml` (with a comment) and `AppKeymaps` fields;
   - `keymaps/metadata.py` rows and the fallback `bindings.py`;
   - `_app_action_availability.py`: Agents tab only, flag on, not while the prompt input
     or a modal owns keys;
   - `_CONTEXTUAL_APP_DUPLICATES`: add `{next_agents_tab, cycle_artifacts_subtab}` and
     `{prev_agents_tab, cycle_artifacts_subtab_reverse}`;
   - the help modal rows (`modals/help_modal/agents_bindings.py`);
   - the footer (`widgets/_keybinding_bindings_agents.py`, shown only when the strip is
     visible);
   - palette metadata (`commands/_app_metadata_nav.py`) and palette availability
     (`commands/_availability_agents.py`, via a context field).

   Also add `pick_agents_tab`, unbound, with palette text "Agents: go to tab…". It opens
   a minimal option-list picker of catalog labels that switches on select. Reuse an
   existing generic picker modal if one fits; `tab-strip` replaces it with the
   searchable picker.

5. **Legacy bracket yield.** Expose the registry's legacy card-block bracket detection
   on `KeymapRegistry` (for example `legacy_card_block_brackets: frozenset[str]`). When
   card blocks are explicitly bound to `[`/`]`:
   - unbind whichever `next/prev_agents_tab` defaults collide (unless the user
     explicitly bound them);
   - extend the warning to say tab cycling yielded the brackets;
   - make sure no key is double-bound.

   Update `test_bracket_keys_do_nothing_on_agents`: with the flag off the brackets still
   do nothing; with it on they cycle tabs. Add a legacy-yield registry test.

6. **Minimal strip.**
   - Compose a `PanelTabStrip` (labels from the catalog entries; `id` is the key token,
     or a session id for unresolved keys) at the left of `#agents-header`, before
     `#agents-fleet-status`.
   - It is visible iff the flag is on and the catalog has two or more tabs, or while the
     latch holds.
   - `_update_agents_header` shows the row when the strip is visible or a diagnostic
     exists, and keeps the hidden-row path (and the flag-off path) byte-identical.
     `TabClicked` switches tabs.
   - `set_tabs` / `set_active_tab` run only when the catalog signature or active key
     changes.
   - `tab-strip` replaces this widget, so keep it minimal. Add CSS only as needed.
7. **Perf.** Wrap the switch in `jk_perf.begin("agents_tab_switch")`, then
   `mark_model_updated()` and `call_after_refresh(mark_painted)`, following the
   `_panel_navigation.py` pattern, so `SASE_TUI_PERF=1` records tab-switch key-to-paint.
8. **AcePage.** Extend `ace/testing/ace_page.py` `_extract_state` with
   `active_agent_tab` (the token) and `agent_tabs` (the labels), so tests and later
   phases can assert on them.
9. **Tests** (both flag states):
   - zero, one, two, and many tabs;
   - `]`/`[` wrap, and are a no-op when the strip is hidden;
   - per-tab selection restore by identity and the nearest-row fallback;
   - persistence round-trip, including an unresolved key that is not persisted;
   - startup selection order;
   - latch and fallback;
   - strip click;
   - a switch does no I/O: assert on stubbed loaders or persistence, with the write
     deferred off the switch path.

**Done when:** `sase tool run check` passes. The targeted flag-off Agents goldens are
unchanged. With `feature_flags.agent_tabs: true` and two tabs, a live `sase screenshot`
(for example `-p ']'` between captures) shows the strip and instant switching; attach
the PNG paths in the phase note.

## Phase 4 — cross-tab-nav

1. **Helper.** Add `_ensure_agent_tab_for(identity_or_row) -> bool`. It finds the target
   in `_agents_with_children`, including the hideable roster where the existing jump
   rungs use it, and reads its key from `_agent_tab_index`. It calls
   `_switch_agents_tab(key, reason="jump")` when the flag is on, the scope is not
   `ALL_AGENT_TABS`, and the key differs. It returns whether it switched.
2. **Shared reveal.** Call the helper inside `prepare_agent_navigation_target` /
   `reveal_agent_navigation_target` (`actions/navigation/_agent_reveal.py`), before the
   refilter and the `_agents` scan. Everything built on the shared contract then works
   across tabs:
   - `_try_reveal_agent_row` and `_reveal_agent_row`;
   - `_reveal_loaded_agent` (link follow);
   - `_reveal_last_launch_target`;
   - the Procs-pane monitor jump;
   - relation-jump activation.

   If a reveal fails after a switch, restore the previous tab, as the anchor stacks are
   restored today.

3. **Callers that scan `_agents` directly.** Route these through the helper plus the
   shared reveal (or `_try_reveal_agent_row`):
   - notification `handle_jump_to_agent` (`_notification_handlers.py`);
   - `navigate_to_agent_tab` (`_notification_navigation.py`);
   - the run-log modal `action_jump_to_agent_tab` (`modals/agent_run_log_modal.py`);
   - Files open agent (`_resolve_file_agent` / `_select_file_agent` in
     `actions/artifacts_files.py`), which must also search `_agents_with_children`, not
     only visible rows;
   - `_select_revived_agent` (`_revive_state.py`);
   - link-trail restore (`_restore_agents_link_trail_hop`).
4. **Node Finder.** It already searches `_agents_with_children`.
   - Treat rows present in `_agents_query_result` but off the active tab as reachable,
     not as not-rendered.
   - Add an off-tab chip (the tab label, muted) to those rows in
     `modals/node_finder_rendering.py` / `_node_finder_rows.py`.
   - `_jump_to_node_identity` then switches through the shared reveal.
5. **`,j` / `,J`.** When the flag is on, build the candidates from the tab-independent
   `_agents_query_result` (plus the existing collapsed-clan candidates) instead of the
   active tab's `_agents`. Select off-tab candidates through the helper plus reveal. Add
   the active scope to the `_unread_jump_candidates` cache signature.
6. **Jump-back anchors.** `AgentJumpAnchor` (`actions/navigation/jump_hints.py`) records
   the tab key. Saving and validating (`_entry_jump_agents.py`) compare against the
   anchor's tab. Restoring an anchor from another tab switches first, then resolves the
   anchor by identity where one is available.
7. **Tests.** For each entry point listed above, the target sits on a non-active tab.
   Assert that the tab switched and the row is selected (flag on), and that behavior is
   unchanged (flag off). Also cover a failed reveal restoring the previous tab, and
   back/forward across a tab switch.

**Done when:** `sase tool run check` passes.

## Phase 5 — scope-honesty

1. **Scope wording helper.** Add `_agent_bulk_scope_label()`, which returns
   `on <tab label>` (for example `on sase`, `on ⌨ apollo`) when the flag is on and the
   strip is visible, `across all tabs` at the `ALL_AGENT_TABS` scope, and `None`
   otherwise. `None` keeps today's text byte-identical.
2. **Confirmations.** Apply the label to every bulk confirmation that acts on "visible"
   rows:
   - the cleanup panel modal's "All" option and its detail lines
     (`modals/agent_cleanup_panel_modal.py`, `_kill_cleanup_panel.py`), replacing
     "loaded panels" when a label exists;
   - dismiss-all-done `D` (`_dismissing.py`);
   - kill-and-dismiss-all `K` (`_kill_all_actions.py`, `ConfirmKillAllModal`);
   - the tribe cleanup selector;
   - the custom selector, which spans the whole roster and so reads "across all tabs".
3. **Marks.** Marks stay global. Marked bulk confirmations (`_present_bulk_kill_modal` /
   `_bulk_kill_marked_agents`) add a line such as "3 of 5 marked agents are on other
   tabs" when any are off the active tab. `,u` stays global, and its toast says "across
   all tabs" when the strip is visible.
4. **Docs.** In `docs/ace.md`, add the `]`/`[` rows (flag-gated) and `pick_agents_tab`
   to the Agents keybinding tables, and add a short "Agent Tabs (beta)" note near
   `### Machines`. It covers the flag, the scope, per-tab memory, and the
   `ace.agent_tabs` pointer. The `finish` phase completes the section.
5. **Verification.**
   - Run the targeted Agents and cleanup-modal goldens through `/sase_monitor` and
     inspect them. The flag-off goldens must be unchanged.
   - Take a live flag-on `sase screenshot` with two tabs and a cleanup confirmation.
   - Before finishing, run `sase bead epic-symbols sase-1bc.6`, and make sure no
     `--epic-symbol` entry keyed to this sub-epic's beads or to `sase-1bc.6` goes stale.
     Re-key any still-needed entry to the parent epic `sase-1bc` or to a later open
     parent phase.
6. **Tests.** Wording in both flag states, the marked-off-tab line, and the custom
   selector wording.

**Done when:** `sase tool run check` passes, the goldens have been inspected, and the
evidence (screenshots, golden report, and perf JSONL sample from phase 3) is recorded in
the phase note. That evidence covers the parent phase `sase-1bc.6`'s "Done when":
flag-off goldens unchanged, `just check` passing, and a live flag-on screenshot
switching instantly.

## Out of scope

These belong to later parent phases:

- the beautiful strip, badges, overflow, arrival dots, and the searchable picker
  (`tab-strip`);
- the All-tabs level and the `o`/`O` ladder (`layout-ladder`);
- machine-tab visuals and the `local` vocabulary (`machine-tabs`);
- launch-from-view and the prompt chip (`launch-view-ux`);
- tab moves (`tab-moves`);
- removing the flag (`finish`).

No sase-core change is needed.
