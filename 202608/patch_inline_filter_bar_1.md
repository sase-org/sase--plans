---
tier: tale
size: medium
title: Cut Patch over to the shared inline filter bar
goal:
  Route the Patches pane's boolean query through the shared profile-driven engine and
  Rust corpus, replace QueryEditModal with a persistent inline PatchFilterBar that owns
  completion, live match count, coverage, Escape rollback and the "#N" save grammar,
  bind "f" to the bar, make "p" rewrite only the committed project scope token, and
  prove slots, history and selection restore never touch a hidden pane.
proposed_by: bbugyi200.athena.sase-m6.6.1.6
bead: sase-m6.6.1.6
create_time: 2026-08-15 23:51:59
status: wip
---

- **PARENT:** [202608/unified_artifacts_query_1.md](unified_artifacts_query_1.md)
- **BEAD:**
  [sase-m6.6.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-m6/sase-m6.6.1.6.md)

# Plan: Cut Patch over to the shared inline filter bar

Implements phase `patch_bar` of the child epic
`plan:202608/unified_artifacts_query_1.md` (bead `sase-m6.6.1.6`). Phases `profile`,
`rust_engine`, `python_reference`, `persistence` and `flat_panes` have landed on
`master`; this phase is the last migration step before `conformance`.

## What already exists (do not redo)

- `patches_query_schema()` (`src/sase/ace/query_profile/profiles.py`) already declares
  the Patch dialect at `boolean=True` with its sigils (`+ ^ ~ &`), `%` status macros,
  the three host predicates and `any_special`. `compile_builtin_contract` already stamps
  the compiled profile onto the Patches `ArtifactsPaneContract`, so "configure the Patch
  contract with `boolean=true`" is done.
- Rust already owns the generic side: `QueryCorpus::from_rows`, `patch_rows_from_specs`
  (which computes the **transitive** ancestor chain), `patch_query_profile()`, and the
  `compile_corpus_with_profile` / `compile_query_with_profile` /
  `parse_query_with_profile` / `canonicalize_query_with_profile` bindings. **No
  `sase-core` change is expected in this phase.** If one turns out to be needed, open
  the repo with `/sase_repo` (`sase repo open sase-core -r "<reason>"`) and use only the
  printed path.
- `sase/core/query_profile_corpus_facade.py` (`compile_artifact_query_index`,
  `evaluate_artifact_query_many`, `ArtifactQueryIndex/Result/CacheKey`) and
  `sase/ace/query/profile_reference*.py` (boolean + flat reference parser,
  canonicalizer, evaluator) are the shared engine this phase plugs Patch into.
- Pane-keyed persistence landed: `saved_queries.json`, `query_history.json` and
  `query_selections.json` are `{pane_id: …}`, `QueryRecord` carries
  `source`/`canonical`/`profile_digest`, and `_load_saved_query`, `action_prev_query`,
  `action_next_query`, `action_open_saved_query_picker` already early-return on
  non-Patches panes instead of hard-switching tabs. This phase **verifies and tests**
  that, it does not rebuild it.

## Defects this phase must fix

Two real parity gaps exist today between Rust's Patch corpus and the Python profile
path. Both are in scope because this phase is what puts the Python path in production
for Patch.

1. **`ancestor:` collapses to the row's own name in Python.** `_coerce_patch_query_row`
   sets `raw_fields["ancestor"] = (name, parent)`, but the `ancestor` field is not
   `repeatable`, so `_raw_values(..., repeatable=False)` keeps only the first element.
   Reproduced on `master`: for `grand <- mid <- kid`,
   `evaluate_query_many("ancestor:grand", …)` (Rust) returns `[True, True, True]` while
   `evaluate_artifact_query_many("ancestor:grand", …, patch_profile)` returns
   `[True, False, False]`.
2. **Row cardinality is wrongly tied to query `repeatable`.** `repeatable` describes the
   _query_ syntax (`key:a,b` in flat mode); it must not truncate a row's multi-valued
   data. Rust's `strings_from_json` keeps every element of a JSON array regardless of
   the field spec, so Python must too.

Fix both **without changing `patches_query_schema()`**, so the Patch profile digest
stays stable and already-saved Patch views do not become `is_stale`.

## Design

### 1. Corpus-aware Patch rows (shared by both evaluators)

In `src/sase/ace/query/profile_evaluator.py`:

- Change `_raw_values` so a non-string `Sequence` always yields every element,
  regardless of `repeatable`; keep the serialized-string split
  (`_parse_serialized_sequence`) gated on `repeatable`. Document that `repeatable` is
  query syntax, not row cardinality.
- Add `coerce_artifact_query_rows(profile, entries) -> tuple[ArtifactQueryRow, ...]`. It
  materializes `entries`; when `profile.pane_id == "patches"` and every entry is
  patch-like (`_is_patch_row`), it builds a `name.casefold() -> parent` map once and
  gives each row the full transitive ancestor chain (own name first, cycle-guarded,
  stopping on an unknown parent) — mirroring `ancestor_names` in sase-core's
  `crates/sase_core/src/query/row.rs`. Otherwise it falls back to per-entry
  `coerce_artifact_query_row`.
- `build_query_context_for_profile` calls `coerce_artifact_query_rows`, so the Python
  reference evaluator (and therefore `evaluate_artifact_query_many` in
  `sase/core/query_facade.py`) agrees with Rust on ancestry.
- Keep `stable_id` derivation for Patch rows unique across projects: use the Patch's
  project query name plus its name, joined by `\x1f`, matching how `patch_row_target`
  composes `(project_name, name)`. Export a small helper so the TUI can compute the same
  id for a `Patch` without importing Textual code.

Export `coerce_artifact_query_rows` from `sase/ace/query/profile_reference.py`.

In `src/sase/core/query_profile_corpus_facade.py`:

- `compile_artifact_query_index` uses `coerce_artifact_query_rows` instead of a
  per-entry comprehension (no signature change).
- `ArtifactQueryResult` gains `matched_mask: tuple[bool, ...]`, populated by
  `evaluate_artifact_query_many` alongside `matched_row_ids`. Every existing consumer
  keeps using `matched_row_ids`; Patch uses the positional mask so filtering never
  depends on id uniqueness. Update `tests/ace/tui/test_artifacts_query_session.py`'s
  hand-built `ArtifactQueryResult` accordingly.

### 2. Patch filtering on the profile-driven Rust corpus

`src/sase/ace/tui/actions/patch/_loading.py`:

- Replace `QueryCorpus` with `ArtifactQueryIndex` throughout the mixin:
  `_PreparedPatchLoad.query_corpus` -> `query_index`,
  `_compile_query_corpus_for_patches` -> `_compile_patch_query_index`,
  `_validate_query_corpus_for_patches` -> `_validate_patch_query_index`,
  `_apply_prepared_query_corpus` -> `_apply_prepared_patch_query_index`,
  `_get_query_corpus_for_patches` -> `_get_patch_query_index`. Keep the same "built for
  this exact `list` object" guard by storing `id(patches)` next to the index, and keep
  `index.validate()`'s row-count check.
- The index is built with `pane_id="patches"`, the pane profile
  (`compiled_profile_for_builtin_pane("patches")`, cached on the app), and a
  monotonically increasing per-app generation counter so cache keys can never collide
  across reloads. Construction stays inside `_prepare_patch_load_from_disk`, which
  already runs under `asyncio.to_thread` on the async path.
- `_filter_patches_impl` evaluates through `evaluate_artifact_query_many(query, index)`
  and keeps rows by `result.matched_mask`. It filters on the **display** query (see §4),
  not necessarily the committed one. Wrap it in a bounded (32-entry) `OrderedDict` cache
  keyed by `ArtifactQueryCacheKey` so repeated keystrokes and Escape rollback are free;
  clear the cache whenever the index is replaced.
- Hide-toggle logic (`query_explicitly_targets_terminal` / `…_submitted`,
  `build_query_context`) is unchanged, but reads the display parsed query.
- Parse Patch queries with `parse_query_for_profile(text, profile)` (the profile-aware
  reference parser, whose `ProfileQueryError` subclasses `QueryParseError`) instead of
  `sase.ace.query.parse_query` in every TUI Patch path: `actions/base.py`,
  `actions/patch/_query.py`, `actions/navigation/_tree.py`, and startup query load.
  `canonical_query_string` stays `to_canonical_string(self.parsed_query)`, which now
  means the profile canonical form. The Rust parser still runs for every evaluation
  inside `evaluate_artifact_query_many`, so both engines execute on the hot path.
- `sase/core/query_corpus_facade.py` and `sase.ace.query.parse_query` stay as
  compatibility entry points for non-TUI consumers (CLI, doctor, tests). Removing them
  belongs to the `conformance` phase.

### 3. `PatchFilterBar` and a highlighted closed display

New `src/sase/ace/tui/widgets/artifacts/patch_filter_bar.py`:

- `PatchFilterBar(FilterBar)` with `PERSISTENT = True`,
  `ACCENT = ARTIFACTS_ACCENTS["patches"]`, ids
  `patch-filter-row/-sigil/-input/-status/-completion` and `patch-filter-candidate`
  prefix. It is constructed with the pane contract's compiled profile, so its keys,
  value hints and free-text hint come from the profile.
- `_completion_context` override for the boolean grammar. Add a pure helper
  `patch_completion_context(text, cursor, *, profile)` to a new
  `src/sase/ace/query/completion.py` (domain-only, no Textual import) that returns
  `(kind, prefix, negated=False)` for: start of input, after whitespace, after `(`,
  after `!`/`AND`/`OR`/`NOT`, inside a bare word (`"key"`), after `key:`
  (`kind == key`), after a sigil (`+ ^ ~ &` -> that sigil's field), and after `%`
  (`kind == "macro"`). Quoted strings return `("text", …)`.
- `_candidates_for` override that, on top of the inherited key/value rows, offers: sigil
  shorthands with their field hints, `%<letter>` macros labelled with their status
  value, the predicate shorthands (`!!!`, `!!`, `@@@`, `!@`, `$$$`, `!$`, `*`), and —
  only when the query is empty — the `#` / `#N` save grammar, so the bar advertises the
  save path the modal hid. Value rows come from `set_observed_facets(index.facets)`.
- `_closed_display_text` override returning `build_query_text(text)`
  (`sase/ace/tui/widgets/patch_detail.py`) plus the saved-slot indicator the
  `SearchQueryPanel` renders today, read from the app's `_saved_queries` cache for the
  `"patches"` pane.

In `src/sase/ace/tui/widgets/filter_bar.py`, add the opt-in closed-display seam:

- `DISPLAY_ID` class var and a `_closed_display_text(text) -> Text | None` hook that
  returns `None` in the base class (so Stitches/Beads/Plans/Files are unchanged).
- When the hook returns text, `compose` also yields a `Static` sibling of the editor;
  `open()`/`close()`/`set_query()` toggle which of the two is displayed (editor while
  editing, `Static` when closed). When the hook returns `None` the widget behaves
  exactly as today.

CSS in `src/sase/ace/tui/styles.tcss`: a `PatchFilterBar` block mirroring
`CommitFilterBar` (`display: block`, height 3, `#00D7AF` sigil/border/highlight) plus a
`.filter-bar-display` rule matching `.filter-bar-input` metrics. Drop the now-unused
`#search-query-panel` rule.

`ArtifactsPatchesPane.compose` yields `PatchFilterBar(id="patch-filter-bar", profile=…)`
where `SearchQueryPanel(id="search-query-panel")` is today. `SearchQueryPanel` itself
stays exported (it is re-exported from `changespec_detail.py` and covered by
`tests/ace/tui/widgets/test_search_query_panel_cache.py`); only the Patches pane stops
mounting it. Update `actions/_startup_mount.py`'s widget-cache tuple and
`actions/patch/_display.py`'s `_get_search_query_widget` to resolve the bar, and have
`_refresh_display_impl` push both the query text and the live status into it.

### 4. Patch filter session

New `src/sase/ace/tui/widgets/artifacts/patches_filter_session.py` with
`PatchesFilterSessionMixin`, mixed into `ArtifactsPatchesPane` (same shape as
`BeadsFilterSessionMixin`). It owns only session state and message handling and
delegates all Patch-state mutation to the app:

- `show_filters()` — idempotent focus if already open; otherwise record the committed
  source query and the selected `ArtifactEntryTarget`, push `index.facets` into the bar,
  and `bar.open(app.query_string)`.
- `on_patch_filter_bar_query_changed` — parse with `parse_query_for_profile`; on
  `QueryParseError` call `bar.set_status(None, exact=False, error=exc)` and leave the
  list untouched; otherwise install the live query, re-filter, and
  `bar.set_status(len(app.patches), exact=True, error=None)`.
- `on_patch_filter_bar_submitted` — if the text matches `^#(\d)?(.*)$`, run the save /
  delete grammar and keep the bar open; otherwise commit and close.
- `on_patch_filter_bar_dismissed` — drop the live query, re-filter to the committed one,
  restore the remembered selection, close, focus the Patch list.

App-side helpers, extracted from today's `action_edit_query.on_dismiss` closure into
`src/sase/ace/tui/actions/patch/_filter_session.py`:

- `_set_patch_live_query(source, parsed)` / `_clear_patch_live_query()` — set/clear
  `_live_patch_query` (source + parsed) and re-filter `self._all_patches` in place. No
  disk read, no history push, no `save_last_query`, no selection write.
- `_display_patch_query()` / `_display_patch_parsed_query()` — the live pair when a
  session is open, else `query_string` / `parsed_query`. `_filter_patches_impl` and
  `_refresh_display_impl` read these.
- `_commit_patch_query(source)` — verbatim today's "normal query update" branch: parse,
  compare canonical, `_save_selection_for_current_query`, push
  `QueryRecord(source, canonical)` onto the `"patches"` history stack, assign
  `query_string`/`parsed_query`, `_load_patches()`,
  `_restore_selection_for_current_query`, `_save_current_query()`, notify.
- `_save_patch_query_slot(text)` — verbatim today's `#N` branch (delete on bare `#N`,
  next-free slot on bare `#`, move-notify when the query already occupies another slot),
  parsing through the profile parser and stamping `current_profile_digest("patches")`
  via the existing `save_query` call.

`action_edit_query` (`actions/base.py`) loses the `QueryEditModal` push entirely: on the
Patches pane it now resolves the pane and calls `show_filters()`, matching the branches
the other panes already have. `QueryEditModal` itself stays — the Agents tab still uses
it from `actions/agents/_filter_actions.py`.

Marks, fold state and the Patch banner group key are untouched by live filtering beyond
what `_apply_patches`/`_refresh_display` already do; `_patch_banner_focus_still_valid`
is re-checked after each live re-filter, exactly as `_apply_reloaded_patches` does.

### 5. `p` rewrites only the project scope

New pure helper `src/sase/ace/query/project_scope.py`:

- `project_scope_of(query) -> str | None` — the value of the first top-level `project:`
  term (either `project:x` or `+x`).
- `rewrite_project_scope(query, project) -> str` — replace the first top-level
  `project:` term's value in place (preserving whichever spelling the user typed), drop
  any further top-level `project:` terms, append `project:<name>` when none exists, and
  remove the term entirely (plus one adjacent explicit `AND`, preferring the preceding
  one) when `project is None`.
- `PROJECT_SCOPE_NESTED` sentinel: when a `project:` term exists only at paren depth
  > 0, the rewrite is refused so a grouped expression is never silently rewritten.

It walks `tokenize_query_for_display` (`sase/ace/query/highlighting.py`), whose output
covers the source contiguously including whitespace, so every other committed token
survives byte-for-byte.

Enabling `p` on Patches goes through the contract's declared-fact route, not an
availability special case: `_app_action_availability.check_app_action` already gates
`pick_artifacts_project` on `PaneCapability.PROJECT_SCOPE`, and the Patches builtin
adapter in `_artifact_tab_contract_adapters.py` declares `project_scoped=False`. Flip
that fact to `True` so the `project_scope_from_declaration` rule turns the capability ON
with an auditable verdict. `test_builtin_contract_snapshots`'s
`assert not contract.has(PaneCapability.PROJECT_SCOPE)` for `patches` flips with it.

`action_pick_artifacts_project` (`actions/artifacts.py`) drops the
`current_artifacts_pane_key == "patches"` early return. For Patches it seeds
`current_project` from `project_scope_of(self.query_string)` mapped through
`choices.project_ref_display.project_key_for_ref`. On pick it does two things and
nothing else: `_set_artifacts_project_scope(project, picked=True)` (the shared scope
every other pane already follows) and
`_commit_patch_query(rewrite_project_scope(self.query_string, <display name or None>))`.
The Patch query is rewritten **only** on an explicit pick — startup and implicit scope
propagation never inject a `project:` token into a user's committed Patch query — so the
rewrite is handled in `_on_picked` rather than plumbed through
`ArtifactsView.set_project_scope` (which deliberately does not enumerate the Patches
pane). `_clear_all_artifacts_marks` already skips `patches`, so Patch marks survive a
scope change unchanged. A refused nested rewrite notifies "project scope is inside a
grouped expression — edit the query with `<f>`" and changes nothing.

### 6. Keymap: `f` opens the bar

`f` on Patches is `edit_hooks` today, so the load-bearing `f` move needs a home for hook
editing. Move `edit_hooks` to `F`, which is unbound on both surfaces that use it
(Patches and the Agents "fork chat as agent" alias); `F` is otherwise only
`stitches_fetch`, gated to the Stitches pane exactly as `f` is gated today.

- `src/sase/default_config.yml`: add `patches_filters: "f"` beside the other `*_filters`
  entries and set `edit_hooks: "F"`.
- `keymaps/app_keymaps.py`: add the `patches_filters: str` field.
- `keymaps/metadata.py`: add `("patches_filters", "Patch Filters", False)`.
- `tui/bindings.py`: add `Binding("f", "patches_filters", "Patch Filters", show=False)`
  and rebind `edit_hooks` to `F`.
- `_app_action_availability.py`: `patches_filters` is available only on the Artifacts
  tab's Patches pane.
- `commands/_app_metadata.py` + `commands/availability.py`: register
  `app.patches_filters` so the command palette can reach it.
- `_artifact_tab_actions.py`: add `"patches_filters"` to
  `CAPABILITY_HOST_ACTIONS[PaneCapability.FILTER_SESSION]`.
- `action_patches_filters` lives next to the other pane filter actions and calls the
  pane's `show_filters()`.
- Help modal: `patches_bindings.py` keeps its `edit_hooks` row (the key renders from the
  registry), and `patches_artifact_bindings.py` gains a "Patch Pane" section documenting
  `f` / `/`, the sigils, `%` macros, predicate shorthands, `#N` saves, and Enter/Esc —
  mirroring the existing "Stitch Pane" section — plus an updated
  `pick_artifacts_project` row ("Pick; Patches/Stitches rewrite `project:`").

No manual `CHANGELOG.md` edit: that file is release-please generated and the
`just check` gate only validates its structure. The conventional-commit subject carries
the note.

## Files touched (expected)

```
src/sase/ace/query/completion.py                            (new)
src/sase/ace/query/project_scope.py                         (new)
src/sase/ace/query/profile_evaluator.py
src/sase/ace/query/profile_reference.py
src/sase/core/query_profile_corpus_facade.py
src/sase/ace/tui/widgets/filter_bar.py
src/sase/ace/tui/widgets/artifacts/patch_filter_bar.py      (new)
src/sase/ace/tui/widgets/artifacts/patches_filter_session.py(new)
src/sase/ace/tui/widgets/artifacts/panes.py
src/sase/ace/tui/widgets/artifacts/__init__.py{,.pyi}
src/sase/ace/tui/actions/patch/_loading.py
src/sase/ace/tui/actions/patch/_filter_session.py           (new)
src/sase/ace/tui/actions/patch/{_query.py,_display.py,_core.py,__init__.py}
src/sase/ace/tui/actions/{base.py,artifacts.py,_startup_mount.py,_state_init_navigation.py}
src/sase/ace/tui/actions/navigation/_tree.py
src/sase/ace/tui/{_app_action_availability.py,bindings.py,styles.tcss}
src/sase/ace/tui/_artifact_tab_contract_adapters.py
src/sase/ace/tui/keymaps/{app_keymaps.py,metadata.py}
src/sase/ace/tui/commands/{_app_metadata.py,availability.py}
src/sase/ace/tui/_artifact_tab_actions.py
src/sase/ace/tui/modals/help_modal/patches_artifact_bindings.py
src/sase/default_config.yml
```

## Tests

New and updated coverage, all under `tests/`:

- `tests/test_query_profile_reference.py` — transitive `ancestor:` on a grandparent
  chain; multi-valued non-`repeatable` row fields survive coercion; Patch `stable_id` is
  unique for equal names in different projects.
- New `tests/test_query_profile_patch_parity.py` — for a Patch corpus with an ancestry
  chain, reverted siblings, error/running suffixes and every sigil/macro, assert
  `evaluate_query_many` (Rust, legacy corpus), the profile-driven
  `evaluate_artifact_query_many` (Rust corpus facade) and
  `evaluate_query_many_for_profile` (Python reference) return identical masks, and that
  `parse_query` and `parse_query_for_profile` canonicalize identically across the
  existing golden query corpus.
- Rewrite `tests/ace/tui/test_changespec_query_corpus_routing.py` for
  `ArtifactQueryIndex`: index built once per Patch list identity, rebuilt on a new list,
  stale index rejected, filtering keyed off `matched_mask`, evaluation cache hit on a
  repeated query.
- New `tests/ace/tui/test_patch_filter_bar.py` — boolean completion context table
  (start, after space, after `(`, after `!`, mid-word, after `key:`, after each sigil,
  after `%`, inside quotes); sigil/macro/predicate/`#` candidate rows; live typing
  re-filters without touching `query_string`, history or `last_query.txt`; Enter
  commits, pushes history and restores the selection; Escape restores the committed
  query and selection; a parse error paints the status and leaves the list alone;
  `#3 status:draft` saves to slot 3 and `#3` deletes it, both without changing the
  active query.
- New `tests/ace/tui/test_patch_project_scope.py` — `rewrite_project_scope` adds,
  replaces (both `project:` and `+` spellings), removes with its adjacent `AND`,
  preserves every other token, and refuses a nested `project:`; `p` on Patches drives
  the picker and commits the rewritten query; an implicit (non-picked) scope change
  leaves the committed Patch query alone.
- `tests/ace/tui/artifacts_contract/test_contract_compiler.py` — the Patches contract
  now declares `PROJECT_SCOPE` ON, with the `project_scope_from_declaration` verdict.
  Re-run the whole `tests/ace/tui/artifacts_contract/` suite; any conformance or
  capability golden that enumerates Patches capabilities updates with it.
- New `tests/ace/tui/test_patch_query_pane_isolation.py` — with the Beads pane active, a
  saved-slot key, `^`/`_`, and the saved-query picker leave the Patch query,
  `query_history.json` and `query_selections.json` untouched and never switch sub-tabs.
- `tests/test_keymaps_defaults.py` picks up the new `patches_filters` field
  automatically; extend `tests/ace/tui/test_keymaps_app_bindings.py` (or the closest
  existing binding test) with `f -> patches_filters` on Patches and `F -> edit_hooks`.
- Visual: retire `tests/ace/tui/visual/snapshots/png/query_edit_modal_120x40.png` and
  its `test_query_edit_modal_png_snapshot`, replacing them with
  `patch_filter_bar_closed_120x40` (persistent bar with the highlighted committed query
  and slot indicator) and `patch_filter_bar_completion_120x40` (bar open with the
  completion menu). Refresh `changespec_selected_row_120x40` and any other Patches
  golden the bar shifts, via `--sase-update-visual-snapshots`, inspecting the
  actual/expected/diff artifacts in `.pytest_cache/sase-visual/` before accepting.

## Verification

1. `just install` first (ephemeral workspace).
2. `just check` — whole-repo lint gates plus the diff-scoped test lane. If the scoped
   run escalates or reports an unusual selection, hand `just check-full` to
   `/sase_monitor` with a `--next` action instead of running it inline.
3. `just test-visual` for the Patch goldens; accept only intentional diffs.
4. `SASE_TUI_PERF=1` spot check that Patch `j`/`k` navigation and per-keystroke
   filtering stay under the 16 ms p95 gate and that no disk, Git or profile-compile work
   appears in the typing path (the index is built in `_prepare_patch_load_from_disk`,
   and repeat queries hit the bounded result cache).
5. Manual sanity via `/sase_run`-style launch is not required; the TUI tests drive the
   bar through `AcePage`.

## Out of scope

- The rest of the Artifacts keymap unification (`y`, `R`, `s`, `L`, `o`) — parent epic's
  `keymap` phase.
- Deleting `QueryEditModal`, `sase.ace.query.parse_query` or
  `sase/core/query_corpus_facade.py`; the `conformance` phase removes obsolete paths
  only where compatibility permits.
- Widening `ArtifactEntryWire.properties`, the Agents tab's `ace/agent_query`, and
  retiring the legacy persistence readers.
- Cross-pane parity/golden expansion and the p95 measurement matrix, owned by
  `sase-m6.6.1.7`.
