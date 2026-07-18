---
tier: epic
title: Plans filter bar with live filtering and completion
goal: 'The Artifacts Plans sub-tab is filtered through a slash-triggered, completion-assisted
  inline filter bar that narrows proposals, epic phase trees, and the plan archive
  instantly from in-memory data — with a debounced deep-archive search that makes
  truncated archive results authoritative — matching the Commits filter bar in interaction,
  power, and polish.

  '
phases:
- id: filter-query
  title: Plan filter query language and search index
  depends_on: []
  description: '''Plan filter query language and search index'' section: extract the
    shared token toolkit from the commits query module, implement the plans query
    parser, canonical round-trip, in-memory matcher, completion context, and snapshot
    search index, with thorough unit tests.'
- id: filter-bar-widget
  title: Shared filter bar and PlanFilterBar widget
  depends_on:
  - filter-query
  description: '''Shared filter bar and PlanFilterBar widget'' section: extract the
    generic bar/completion machinery from CommitFilterBar into a reusable base widget
    and build the plans-accented PlanFilterBar with its completion schema, keeping
    commits behavior unchanged, with widget-level tests.'
- id: pane-integration
  title: Pane wiring, live filtering, and keymaps
  depends_on:
  - filter-bar-widget
  description: '''Pane wiring, live filtering, and keymaps'' section: mount the bar
    in ArtifactsPlansPane, implement instant in-memory tree filtering with matched-phase
    auto-expansion, route the slash and f keymaps, and update hint, help, palette,
    and availability surfaces, with pilot tests.'
- id: deep-archive
  title: Deep archive search reconciliation
  depends_on:
  - pane-integration
  description: '''Deep archive search reconciliation'' section: add the debounced,
    worker-based deep archive fetch through the plan-search facade that upgrades truncated
    preview results to exact, with last-request-wins coalescing and staleness tests.'
- id: polish-verify
  title: Visual polish and verification
  depends_on:
  - deep-archive
  description: '''Visual polish and verification'' section: add PNG snapshot goldens
    for the bar states, finalize chips, hints, and help copy, and verify TUI responsiveness
    with the perf tooling.'
create_time: 2026-07-18 10:09:08
status: wip
---

# Plan: Plans filter bar with live filtering and completion

## Context

### Motivation

The Artifacts → Plans sub-tab has no filtering of any kind. The only narrowing is structural (three fixed sections, the
per-project archive cap) and the only scope control is the project picker. Meanwhile the Commits sub-tab just gained a
slash-triggered inline filter bar (epic `sase-6s`, plan `202607/commits_filter_bar.md`) with completion, live preview,
and canonical query round-trip. This epic brings the same interaction — the same `/` muscle memory, the same query-token
vocabulary, the same completion dropdown — to the Plans sub-tab, tuned to what plan rows actually are: a small,
fully-in-memory pipeline of proposals, epic/phase bead trees, and archived plan markdown.

### Current architecture (what we build on)

- `ArtifactsPlansPane` (`src/sase/ace/tui/widgets/artifacts/plans_pane.py`) owns `project_scope: str | None` (`None` =
  all enabled projects), a sticky `_expanded_epics` set, and an immutable `PlansSnapshot` loaded off-thread by
  `load_plans_snapshot` (`plans_data.py`) via `run_worker(..., thread=True)` with `_loading`/`_reload_pending`
  coalescing and an mtime-based `source_key` cache. `_refresh_options()` rebuilds a plain `OptionList` (`#plans-list`)
  from `build_plan_options` (`plans_list.py`), preserving selection by option id under the `_syncing_options` guard.
- **Everything needed to filter is already resident in the snapshot.** `PlanProposal` carries `title`, `tier`,
  `timestamp`, `frontmatter` (incl. `goal`), full `content`/`body`, `agent`, `plan_path`. Epic/phase rows carry the full
  bead `Issue` (`id`, `title`, `status`, `tier`, `assignee`, `created_at`, `description`, `notes`, `design`,
  `changespec_name`) plus derived `ready_ids`/`blocked_ids` sets. Archive rows carry the plan-search `Plan` (`title`,
  `name`, `relpath`, `status`, `created_at`, `summary`, full `body`, `frontmatter`). Unlike Commits, live filtering
  needs **no backend re-query**: it is pure presentation over loaded rows.
- The one coverage gap is the archive: `_load_project_archive` browses the newest 50 plans per project (100 merged), and
  `snapshot.archive_truncated` records when older plans were cut. The Rust-backed facade
  `sase.plan_search.facade.search` already supports deeper, sorted, limit-capped browsing per plans root — the raw
  material for an authoritative deep-search tier.
- The Commits epic already built the reusable pieces this epic generalizes: the token query language
  (`src/sase/vcs_log/filter_query.py`: tokenizer with quoting/escapes, span-carrying `CommitFilterQueryError`, canonical
  `to_query_tokens` round-trip, `completion_context`), the bar widget (`widgets/artifacts/commit_filter_bar.py`:
  `SingleLineVimTextArea` input, overlay completion `OptionList`, status region, Tab/arrows/Escape handling,
  `_apply_completion` cursor math), and its `styles.tcss` rules.
- `/` is globally bound to `edit_query` (`default_config.yml`), gated by `check_action` in `src/sase/ace/tui/app.py`,
  which currently whitelists it on non-PRs artifact sub-tabs only for `commits`. `action_edit_query` (`actions/base.py`)
  reroutes per surface (Agents search, Commits bar) — the routing precedent to extend. `f` is unbound on the Plans
  sub-tab (`commits_filters: "f"` is gated to commits), so Plans can mirror the `/` + `f` pairing.

### Performance ground rules (from long-term memory `tui_perf.md`)

Hard constraints on this design:

- Keystroke paths are read-only: no disk I/O, subprocesses, or lock-taking while typing. All matching runs over a
  prefolded in-memory index; all completion sources come from the loaded snapshot.
- Slow work runs in thread workers with coalescing/staleness guards (last-request-wins); expensive follow-ups are
  debounced; never stack work.
- Programmatic `OptionList` updates stay inside the existing `_syncing_options` guard; detail-panel updates stay behind
  `DetailPanelDebouncer`; filtered repaints cancel artifacts jump mode via the same
  `_cancel_artifacts_jump_mode_for_model_change("plans")` hook the existing model-changing paths use.

### Rust-core boundary note

The query parser, matcher, and search index stay in Python: they are TUI presentation glue over records the backend
already ships (`plan_search` wire mirrors, bead `Issue`, proposal inventory), and authoritative archive coverage still
flows through the existing Rust `plan_search` binding. If another frontend later needs the same query language — or true
full-corpus relevance search — promote the language into `sase-core`'s plan-search crate at that point; do not block
this epic on it.

## Design overview

### Interaction walkthrough

1. On the Plans sub-tab, pressing `/` (or `f`) slides a one-line filter bar into view at the very top of the pane,
   pre-filled with the canonical query for the currently committed filter (empty by default), and focuses it.
2. The user types a token query. A completion dropdown under the bar offers context-aware candidates; Tab (or Enter on a
   highlighted row) accepts, up/down navigate, Escape closes the dropdown first, then the bar.
3. Every keystroke live-narrows all three sections from in-memory data: section headers show `matched/total` counts,
   epics with matching phases auto-expand to reveal them, and the bar's status region shows the live match count with an
   `exact`/`preview` coverage tag.
4. Enter commits the filter: the bar closes, focus returns to the list, the pane header shows the active query as chips,
   and the filter keeps applying across subsequent snapshot refreshes.
5. Escape closes the bar and reverts to the filter, expansion state, and selection that were active when the bar opened.
6. Invalid input never crashes or commits: the offending token is reported inline in the bar's status area, Enter is
   refused with a notify, and the list keeps showing the last valid query's results.

### Query language

Same lexical rules as the commits language (whitespace-separated tokens, double quotes preserve spaces, case-insensitive
keys, unknown `key:` prefixes are errors with a "did you mean"), with a plan-shaped vocabulary:

- `kind:<proposal|epic|phase|archive>` — repeatable and/or comma-separated; OR semantics; selects row kinds.
- `status:<value>` — repeatable; OR; matched case-insensitively against the row's status-label set: proposals expose
  `proposed`; epic/phase rows expose their bead status (`open`, `in_progress`, `closed`) plus the derived pipeline
  states the section legend already renders (`ready`, `blocked`, and the launched state), computed from the same
  snapshot sets and glyph logic as the rows themselves; archive rows expose their frontmatter `status` string.
- `tier:<value>` — repeatable; OR; matched against the row's tier labels (bead tier `plan`/`epic`, proposal tier,
  archive plan kind/frontmatter tier `tale`/`epic`).
- `project:<name>` — repeatable; OR; case-insensitive exact match on the project key or its display name. Most useful in
  all-projects scope, valid in any scope.
- `since:<date>` / `until:<date>` — exactly the existing `DATE_HELP` forms (`Nh`/`Nd`/`Nw`, `today`, `yesterday`,
  `YYYY-MM-DD`, `YYYY-MM-DDTHH:MM`), reusing `parse_time_bound`; bounds the row's creation/arrival time (proposal
  `timestamp`, bead `created_at`, archive plan `created_at`), normalized to epoch at index-build time. Rows without a
  parseable timestamp are excluded when a bound is set.
- Bare words — case-insensitive substring terms, AND across words, matched against the row's full haystack: title,
  identifiers (bead id, plan name, relpath), goal/summary, description/notes, full markdown body, assignee, agent, and
  project. Free text is what makes `sase-6s` find the epic and `filter bar` find this plan.

Deliberate deviation from commits: there is **no `limit:` key**. The pipeline list is small and bounded; archive
coverage is handled by the deep-search tier rather than a user-facing row cap.

The parsed form is a frozen `PlanFilterValues`; `values → canonical query string → values` is lossless with stable token
order (kind, status, tier, project, since, until, then text). An empty query means "no filter".

### Filtering semantics (tree rules)

- Proposal and archive rows show iff they match.
- An epic row shows if it matches **or any of its phases match**; epics shown only as ancestors render auto-expanded (an
  "effective expansion" overlay — the sticky `_expanded_epics` set is never mutated by filtering).
- A phase row shows if it matches, or if its parent epic matches and is expanded (normal expansion rules apply beneath a
  matching epic).
- Section headers stay visible with `matched/total` counts (e.g. `── Epics (3/12)`); an empty section keeps its dim
  empty label. The match count in the bar counts rows that match in their own right (ancestors shown for context are not
  counted).

### Coverage model (exact vs preview)

Proposals, epics, and phases are always fully loaded, so results over them are always exact. The archive is exact unless
`snapshot.archive_truncated` is set **and** the query can reach archive rows (i.e. `kind:` does not exclude `archive`).
While that gap exists the bar's tag reads `preview`; the deep-archive tier (phase 4) reconciles it to `exact`. This
reuses the commits bar's `exact`/`preview` vocabulary so the two bars speak one language.

## Plan filter query language and search index

Extract the generic machinery and build the plans language on top of it. All modules are pure (no I/O):

- **New shared toolkit `src/sase/filter_tokens.py`** — move the reusable, domain-free parts out of
  `src/sase/vcs_log/filter_query.py`: the tokenizer (`_Token`, quoting/escape handling, strict/lenient modes), the
  span-carrying error base, unquoted-index/split/quote-value helpers, and the generic completion-context scaffolding
  (parameterized by the key set and repeatable keys). `vcs_log/filter_query.py` becomes a thin commits-specific layer
  over the toolkit with its public API (`parse_commit_filter_query`, `to_query_tokens`, `compile_commit_matcher`,
  `completion_context`, `CommitFilterQueryError`) unchanged — `tests/test_vcs_log_filter_query.py` must keep passing
  without modification (contract guard for the refactor).
- **New module `src/sase/plan_search/filter_query.py`** — the plans language:
  - `PlanFilterValues` frozen dataclass: `kinds`, `statuses`, `tiers`, `projects`,
    `since_text`/`until_text`/`since`/`until`, `text` tuples with an `is_empty` helper.
  - `parse_plan_filter_query(text) -> PlanFilterValues` — validates `kind:` values against the four row kinds, dates via
    `parse_time_bound`, rejects unknown keys with close-match suggestions, raises a span-carrying
    `PlanFilterQueryError`.
  - `to_query_tokens(values)` / `to_query_string(values)` — canonical rendering, quoting only when needed; lossless
    round-trip.
  - `plan_completion_context(text, cursor) -> (kind, prefix)` — cursor classification over the plans key set (keeps
    cursor math out of widgets).
- **New adapter `src/sase/ace/tui/widgets/artifacts/plans_filtering.py`** — bridges snapshot rows to the language:
  - `PlanFilterRecord`: one neutral record per row (row kind, project key + display name, prefolded status-label set,
    tier-label set, epoch timestamp or `None`, prefolded haystack strings, identity linking back to the option-id/row).
  - `build_plan_filter_index(snapshot) -> PlanFilterIndex` — builds records for every proposal, epic, phase, and archive
    row, including the derived ready/blocked/launched labels computed from the same snapshot sets the row glyphs use.
    Casefolding of bodies happens exactly once here.
  - `compile_plan_matcher(values) -> Callable[[PlanFilterRecord], bool]` — cheap reusable predicate: kind membership,
    status/tier/project OR-sets, timestamp bounds, AND substring text terms. The pane caches the index per snapshot
    identity and builds it lazily on first filter use (an action path, not a keystroke path); per-keystroke work is then
    pure substring checks over prefolded strings.

Unit tests (`tests/test_plan_filter_query.py`, extending the patterns of `tests/test_vcs_log_filter_query.py`): parsing
of every token kind, quoting, comma lists, repeated keys, error spans and did-you-mean, canonical round-trip property,
matcher semantics (OR within a key, AND across keys and text terms, case-insensitivity, timestamp bounds,
missing-timestamp exclusion), completion context at various cursor positions, and index construction from a synthetic
snapshot (derived status labels included). Plus the untouched commits query suite passing as the extraction guard.

## Shared filter bar and PlanFilterBar widget

Generalize the bar so commits and plans share one implementation:

- **New base module `src/sase/ace/tui/widgets/filter_bar.py`** — hosts the behavior currently in `commit_filter_bar.py`:
  the key-capturing `SingleLineVimTextArea` subclass, the overlay completion `OptionList`, the candidate/metadata
  plumbing, `open(prefill)`/`close()`/`set_status(...)`, Tab/up/down/ctrl+p/ctrl+n handling, two-stage Escape,
  programmatic-highlight guards, and the `_apply_completion`/token-bounds cursor math. Subclasses supply: widget/child
  ids, accent styling hooks, the completion-context function, key-candidate table, and value-candidate sources. Each
  concrete bar declares its **own** `QueryChanged`/`Submitted`/`Dismissed` message classes so Textual handler names stay
  per-widget (`on_commit_filter_bar_*` in `CommitsPane` keeps working verbatim).
- **`CommitFilterBar` becomes a thin subclass** with identical public contract, ids, messages, and behavior.
  `tests/ace/tui/widgets/ test_commit_filter_bar.py` and the commits pane pilot suite must pass unchanged — that is the
  refactor's acceptance bar.
- **New `src/sase/ace/tui/widgets/artifacts/plan_filter_bar.py`** — `PlanFilterBar` with `plan-filter-*` ids and the
  plans accent (`ARTIFACTS_ACCENTS["plans"]`, `#AF87FF`) on sigil, input border, dropdown border, and highlight.
  Completion schema:
  - Key position → the six keys with one-line hints (e.g. `status:  ·  open, in_progress, closed, ready, blocked`,
    `since:  ·  Nh/Nd/Nw, today, YYYY-MM-DD`) plus the dim non-selectable `free text · title, body, id (AND)` hint row.
  - `kind:` → the four row kinds. `tier:` → `tale`, `epic`, `plan`.
  - `status:` → the static bead vocabulary + derived states, plus distinct archive statuses observed in the snapshot
    (pushed in via `set_completion_sources` — the widget never computes them).
  - `project:` → project keys and display names from the snapshot.
  - `since:`/`until:` → the same curated forms as commits (`1h`, `24h`, `today`, `yesterday`, `3d`, `7d`, `2w`, and the
    `YYYY-MM-DD` template row that suppresses the trailing space).
  - Candidates filter by typed prefix; an empty match set collapses the dropdown rather than showing an empty box.
- **Styling** (`src/sase/ace/tui/styles.tcss`): refactor the `CommitFilterBar` rules into shared `FilterBar` geometry
  (hidden by default, height 3, base/overlay layers, status min-widths, dropdown offset/size) plus small per-bar accent
  sections — commits keeps gold `#FFD700`, plans gets purple `#AF87FF`. Opening must not shift the list by more than the
  bar's own height.

Widget-level tests (`tests/ace/tui/widgets/test_plan_filter_bar.py`): context switching as the cursor moves across
`kind:`/`status:`/`project:` tokens, candidate filtering and Tab acceptance producing the expected text and cursor
position, message ordering (QueryChanged/Submitted/Dismissed), Escape's two-stage behavior, and completion-source
normalization.

## Pane wiring, live filtering, and keymaps

### Mounting and session semantics

- Mount `PlanFilterBar` as the first child of `ArtifactsPlansPane.compose` (above `#plans-info`).
- New pane state: `self.filters: PlanFilterValues` (empty default), the cached `PlanFilterIndex`, and session fields
  mirroring the commits pane: `_filter_session_open`, `_filter_restore_values`, `_filter_restore_expanded`,
  `_filter_restore_selection`, `_live_filter_values`, `_filter_query_error`.
- `show_filters()`: build the prefill via `to_query_string(self.filters)`, build/reuse the snapshot's filter index, push
  completion sources, remember the restore state, show + focus the bar, and render the current match count.
- Live preview (every `QueryChanged`, synchronous): parse; on success run the compiled matcher over the index and
  repaint via the existing `_refresh_options` path extended with a filter parameter; on failure update only the bar's
  error status and keep the last valid view. This is pure CPU over at most a few hundred prefolded rows — no I/O on the
  keystroke path.
- Enter (valid): adopt the values as `self.filters`, close the bar, focus `#plans-list`; the filter persists across
  snapshot refreshes (each new snapshot gets a fresh index and the same matcher applied). Enter (invalid): keep the bar
  open, `notify` the parse error. Escape: restore remembered values, expansion, and selection; close; focus the list.
- Filtered repaints preserve selection by option id when the row is still visible (falling back to the first selectable
  row), keep using the `_syncing_options` guard, debounce detail updates through the existing `DetailPanelDebouncer`,
  and cancel artifacts jump mode like other model-changing paths.

### List building

Extend `build_plan_options` (`plans_list.py`) with an optional filter argument (the compiled matcher + index, or a
precomputed visible-row set): apply the tree rules from the design overview, render `matched/total` header counts while
a filter is active, and merge the effective-expansion overlay for epics with matching phases. `#plans-status`
(`build_plans_status`) shows matched counts while a filter is applied; `#plans-info` (`build_plans_scope`) gains
active-filter chips rendered from `to_query_tokens` — one vocabulary everywhere.

### Keymap routing and surface updates

- `check_action` (`src/sase/ace/tui/app.py`): extend the commits-only `edit_query` exception to also allow the `plans`
  sub-tab.
- `action_edit_query` (`actions/base.py`): add a Plans branch routing to `pane.show_filters()`, directly after the
  commits branch.
- New registry action `plans_filters` with default key `f` (`src/sase/ace/tui/keymaps/types.py` field + registry entry,
  `src/sase/default_config.yml` under the plans keymap block), added to `PLANS_ARTIFACT_ACTIONS` and implemented as
  `action_plans_filters` in `actions/artifacts_plans.py` → `pane.show_filters()` — mirroring the `/` + `f` pairing on
  Commits.
- `commands/_app_metadata.py`: add `plans_filters` → "Plans: filter bar". `commands/availability.py`: extend the
  `app.edit_query` sub-tab gate to include `plans`, and add `app.plans_filters` to `_PLANS_ARTIFACT_COMMANDS`.
- `build_plans_hints` (`plans_rendering.py`): add the `edit_query` key with a `filter` label. Help modal Plans Pane
  section (`modals/help_modal/changespecs_bindings.py`): document `/`/`f` and the query-token vocabulary, respecting the
  57-char box conventions.

### Tests

Pilot coverage in `tests/ace/tui/test_artifacts_plans.py` (reusing its `_snapshot` fixtures): `/` opens the bar
prefilled and focused; typing narrows all three sections live, auto-expands an epic whose phase matches, and updates
header counts; Enter commits + closes + focuses the list and the chips render; Escape restores filters, expansion, and
selection; `f` opens the same bar; `/` on the Bugs sub-tab stays inert; keys like `e`/`w`/`s` do not fire pane actions
while the bar is focused. Add a regression that typing never invokes `load_plans_snapshot` (keystroke path is disk-free)
and that the filter re-applies after a snapshot refresh lands mid-session.

## Deep archive search reconciliation

Close the archive-truncation gap so a committed or in-progress query is authoritative:

- **Trigger:** a filter session is open (or a non-empty filter is committed), the parse succeeded,
  `snapshot.archive_truncated` is set, and the query can reach archive rows. Otherwise the in-memory result is already
  exact and no work runs.
- **Debounce:** ~300 ms after the last valid query change (a `DetailPanelDebouncer` with the same delay the commits pane
  uses), so rapid keystrokes coalesce to one fetch.
- **Fetch (thread worker):** for each project in scope with a plans root, call
  `sase.plan_search.facade.search(None, kinds=("tale", "epic"), source=SOURCE_REPO, sort="recent", limit=<deep cap, ~500/project>, repo_root=...)`
  — a browse-mode fetch, deliberately **without** pushing the query or statuses down to Rust, so final membership is
  decided by exactly the same Python matcher as tier 1 (one matching semantics, no drift). Map results to archive
  records, index them, and apply the matcher. This is a pure local scan through the existing Rust binding: no network,
  no prompts, no locks, worker-only.
- **Reconciliation:** last-request-wins via a request token (scope, filter values, snapshot identity). When the fetch
  lands and is still current, replace the displayed archive section with the deep matches (deduped by plan path against
  loaded rows), update the match count and header counts, and upgrade the tag `preview` → `exact` — unless the deep
  fetch itself hit its cap, in which case the tag stays honest (e.g. `newest 500 searched`). Escape drops pending
  results via token mismatch; a committed filter keeps its reconciliation when it lands. Deep results are display-layer
  only: the base snapshot stays canonical, and clearing the filter returns to the standard archive listing.
- **Cache:** a small bounded map (like the commits pane's 32-entry authoritative cache) keyed by the request token,
  invalidated on snapshot change, so re-typing a recent query reconciles instantly.

Tests: the worker only runs via the debouncer (a typing burst yields one fetch); stale tokens are dropped after
scope/snapshot/values change; merge dedups by path and preserves recency order; tag transitions (`preview` → `exact`,
capped-fetch honesty); Escape discards pending deep results; no fetch runs when the archive is not truncated or `kind:`
excludes the archive.

## Visual polish and verification

- Extend the plans PNG suite (`tests/ace/tui/visual/`, alongside `test_ace_png_snapshots_artifacts_plans.py`) with
  goldens for: the bar open with a prefilled query and match count; the completion dropdown open on `status:` with
  candidates; a narrowed list showing an auto-expanded epic, `matched/total` headers, and active-filter chips; and the
  inline parse-error state. Goldens live in `tests/ace/tui/visual/snapshots/png/`; accept intentional changes with
  `--sase-update-visual-snapshots`. Existing plans goldens need re-acceptance only if the header/status lines change.
- Final copy pass: chips, hints, help modal, and completion hints share the query-token vocabulary; empty-section labels
  still read correctly under an active filter; the plans and commits bars are visually parallel (same geometry, per-tab
  accents).
- Responsiveness verification per the perf runbook: a `SASE_TUI_PERF=1` spot-check that j/k stays under budget with the
  bar open and a filter active, a typing burst produces no stall-watchdog entries (`~/.sase/logs/tui_stalls.jsonl`), and
  the full plans pilot and visual suites pass under `just test`. Run `just check` for the full gate.

## Risks and mitigations

- **In-flight commits polish (`sase-6s.4`)** touches commits goldens, hint/help copy, and possibly `styles.tcss` — the
  shared-widget extraction (phase 2) edits the same neighborhood. Mitigation: land this epic's phases after `sase-6s.4`
  closes (or rebase over it); the unchanged commits widget and pane test suites are the guard that the refactor is
  behavior-preserving.
- **Handler-name breakage in the widget refactor** — mitigated by declaring message classes per concrete bar and by the
  existing commits pilot tests.
- **Large markdown bodies on the keystroke path** — mitigated by the prefolded per-snapshot index built once on an
  action path; keystrokes only run substring checks.
- **`OptionList` echo bugs from frequent repaints** — reuse the existing `_syncing_options`-guarded `_refresh_options`
  path only; never assign `highlighted` outside it.
- **Tier-1/tier-2 disagreement** — avoided structurally: the deep fetch is browse-mode and the same Python matcher
  decides membership in both tiers; the coverage tag never claims `exact` when a cap was hit.
- **Scope changes while the bar is open** (project picker) — the filter index and deep-search tokens key on scope +
  snapshot identity; the simplest correct behavior (re-run the preview once the new snapshot lands) is acceptable.
- **Derived status drift** (ready/blocked/launched) — the index computes labels from the same snapshot sets and glyph
  logic as row rendering, and a unit test pins them together.
