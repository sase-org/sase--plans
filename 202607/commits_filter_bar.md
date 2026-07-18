---
tier: epic
title: Commits filter bar with live preview and completion
goal: 'The Artifacts Commits sub-tab is filtered through a slash-triggered, completion-assisted
  inline filter bar that updates the timeline live without ever blocking the TUI,
  replacing the broken modal-based `f` flow.

  '
phases:
- id: filter-query
  title: Commit filter query language
  depends_on: []
  description: '''Commit filter query language'' section: implement the token parser,
    canonical round-trip with CommitLogFilterValues, and the in-memory commit matcher,
    with thorough unit tests.'
- id: filter-bar-widget
  title: CommitFilterBar widget and completion
  depends_on:
  - filter-query
  description: '''CommitFilterBar widget and completion'' section: build the bar widget,
    its completion dropdown, status area, and styles against the widget contract defined
    there, with widget-level tests.'
- id: pane-integration
  title: Pane wiring, live preview, and keymaps
  depends_on:
  - filter-bar-widget
  description: '''Pane wiring, live preview, and keymaps'' section: mount the bar
    in CommitsPane, implement two-tier live filtering, route the slash and f keymaps,
    delete the old modal, and update help/palette/availability surfaces.'
- id: polish-verify
  title: Visual polish and verification
  depends_on:
  - pane-integration
  description: '''Visual polish and verification'' section: add PNG snapshot goldens
    for the bar states, finalize hint/help copy and filter chips, and verify TUI responsiveness
    with the perf tooling.'
create_time: 2026-07-18 08:18:53
status: wip
bead_id: sase-6s
---

# Plan: Commits filter bar with live preview and completion

## Context

### The bug being fixed

The `f` keymap on the Artifacts → Commits sub-tab opens `CommitFiltersModal`
(`src/sase/ace/tui/modals/commit_filters_modal.py`). That modal binds only `escape` (cancel) and `ctrl+s` (apply) and
defines **no `Input.Submitted` handler**, so pressing `<enter>` inside any of its inputs is silently dropped — the user
types a filter, presses Enter, and nothing happens. (Compare `QueryEditModal`, which submits via
`on_single_line_vim_text_area_submitted`.)

Rather than patching the modal, this epic replaces it with an inline filter bar where Enter/Escape are the primary
interactions, matching how filtering works everywhere else in the app.

### Current architecture (what we build on)

- `CommitsPane` (`src/sase/ace/tui/widgets/artifacts/commits_pane.py`) owns `filters: CommitLogFilterValues`
  (`widgets/artifacts/commit_filters.py`: authors, since/until text + epoch, repos, limit=40), a `_generation` counter,
  and collects commits through a thread worker (`_schedule_collection` → `run_vcs_log` in
  `src/sase/vcs_log/collect.py`). Backend filters (since/until/authors) are pushed down to `git log`; repo filters
  restrict the resolved repo set; normal refreshes run with `no_fetch=True` (no network, no credential prompts).
- The merged timeline rows are `AggregatedCommitWire` records (`src/sase/core/vcs_log_wire.py`) carrying `repo`,
  `author_name`, `author_email`, `timestamp`, `subject`, `body` — everything needed to filter in memory.
- `/` is globally bound to `edit_query`, but `check_action` in `src/sase/ace/tui/app.py` gates it **off** for non-PRs
  artifact sub-tabs, so `/` currently does nothing on Commits. The Agents tab already reroutes `action_edit_query` to
  its own search (`actions/base.py`), establishing the routing precedent.
- `SingleLineVimTextArea` (a `VimTextArea` subclass with a `Submitted` message) is the established single-line input;
  `check_action` already releases the priority Tab binding while a `VimTextArea` is focused, so Tab is available for
  completion.
- Completion building blocks exist: `CompletionCandidate` (`widgets/file_completion.py`) and candidate/placeholder
  patterns like `widgets/vcs_repo_completion.py`.
- Date parsing: `parse_time_bound` / `DATE_HELP` in `src/sase/vcs_log/dates.py`.

### Performance ground rules (from long-term memory `tui_perf.md`)

These are hard constraints on the design:

- Keystroke paths are read-only: no subprocesses, disk I/O, or lock-taking while typing. All completion sources must
  come from already-loaded data.
- Collection stays in thread workers with generation guards (last-request-wins); debounce expensive follow-ups; never
  stack work.
- `CommitsTimeline` already guards programmatic `OptionList` updates — preserve that pattern when repainting previews.

### Rust-core boundary note

The query parser and in-memory matcher stay in Python under `src/sase/vcs_log/`: they are TUI presentation glue over the
wire records (the authoritative filtering remains the existing backend path — repo resolution + `git log` + Rust
aggregation), and they reuse the existing Python `parse_time_bound`. If another frontend later needs the same query
language, promote the parser into `sase-core` at that point; do not block this epic on it.

## Design overview

### Interaction walkthrough

1. On the Commits sub-tab, pressing `/` (or `f`, kept for muscle memory) slides a one-line filter bar into view at the
   very top of the pane, pre-filled with the canonical query for the currently active filters, and focuses it.
2. The user types a token query. A completion dropdown under the bar offers context-aware candidates; Tab (or Enter on a
   highlighted row) accepts, up/down navigate.
3. Every keystroke live-updates the timeline from in-memory data (instant), while a debounced authoritative
   re-collection runs in the background and reconciles the view when it lands.
4. Enter commits the filter: the bar closes, focus returns to the timeline, the header chips show the active query, and
   an authoritative collection is kicked immediately if one isn't already reconciled.
5. Escape closes the bar and reverts to the filters that were active when the bar opened (restoring the previous view).
6. Invalid input never crashes or commits: the offending token is reported inline in the bar's status area, Enter is
   refused with a notify, and the live preview simply keeps the last valid query's results.

### Query language

Whitespace-separated tokens plus free text; values may be double-quoted to include spaces; keys are case-insensitive:

- `repo:<name>` — repeatable and/or comma-separated; OR semantics; matches the resolved repo names and their aliases
  (same identifiers `--repo` accepts on the CLI).
- `author:<substr>` — repeatable; OR; case-insensitive substring against author name and email (existing backend
  semantics).
- `since:<date>` / `until:<date>` — exactly the existing `DATE_HELP` forms (`Nh`/`Nd`/`Nw`, `today`, `yesterday`,
  `YYYY-MM-DD`, `YYYY-MM-DDTHH:MM`).
- `limit:<N|all>` — row cap (`all` → 0/unlimited).
- Bare words — case-insensitive substring match on the commit subject (AND across words). This is a **new capability**
  applied in memory only (the backend `CommitFilters` record is unchanged); it is applied both to previews and on top of
  authoritative results.

The parsed form extends `CommitLogFilterValues` with the free-text terms and round-trips:
`values → canonical query string → values` is lossless, and the bar pre-fills from it (default limit omitted from the
canonical string).

## Commit filter query language

New module `src/sase/vcs_log/filter_query.py` (pure, no I/O):

- `parse_commit_filter_query(text) -> CommitLogFilterValues` — tokenizer + parser as specified above, reusing
  `parse_time_bound`. Errors raise a dedicated exception carrying the bad token and its span so the bar can
  highlight/report it precisely. Unknown `key:` prefixes are errors (with a "did you mean" for close matches);
  everything else is free text.
- `to_query_string(values) -> str` — canonical rendering (stable token order: repo, author, since, until, limit, then
  text; quoting only when needed).
- `commit_matches(values, entry) -> bool` (or a compiled predicate factory) — full in-memory matcher over
  `AggregatedCommitWire`: repo membership, author substring, timestamp bounds, subject text terms. Limit is applied by
  the caller after filtering. Case-insensitivity matches backend behavior.
- Extend `CommitLogFilterValues` with `text: tuple[str, ...] = ()`; `backend_filters()` is unchanged (text and repos are
  not backend fields). Update `commit_filter_chips` so chips render tokens exactly as the query language spells them
  (e.g. `repo:sase-nvim`, `since:7d`), reinforcing one vocabulary.
- A small context helper for completion: `completion_context(text, cursor) -> (kind, prefix)` classifying whether the
  cursor is at a key position or inside a `repo:`/`author:`/`since:`/ `until:`/`limit:` value, with the partial value.
  This keeps all cursor-math out of the widget.

Unit tests (`tests/vcs_log/…` or alongside existing vcs_log tests): parsing of every token kind, quoting, comma lists,
repeated keys, error spans, canonical round-trip property, matcher semantics (case-insensitivity, OR within repo/author,
AND across kinds and text terms), and completion-context classification at various cursor positions.

## CommitFilterBar widget and completion

New widget `src/sase/ace/tui/widgets/artifacts/commit_filter_bar.py`, composed of:

- A `/` sigil styled in the commits accent (`ARTIFACTS_ACCENTS["commits"]`), echoing the Vim search command line
  (`widgets/search_command_line.py`).
- A `SingleLineVimTextArea` input (Enter → `Submitted`, vim editing for free).
- A right-aligned status region rendering, as appropriate: live match count (`36 matches`), a `preview`/`exact` state
  tag, or a red parse-error message for the current input.
- A completion dropdown rendered as a small overlay `OptionList` directly below the bar, styled like the existing
  completion panels, showing at most ~8 rows with a dim kind/hint column.

Widget contract (defined here so the pane phase codes against it):

- API: `open(prefill: str)`, `close()`, `set_status(match_count, exact, error)`,
  `set_completion_sources(repos, authors)`.
- Messages: `QueryChanged(text)` on every edit, `Submitted(text)` on Enter (suppressed while a completion row is being
  accepted), `Dismissed()` on Escape when the dropdown is already closed (Escape closes the dropdown first).
- Keys while focused: up/down (or ctrl+p/ctrl+n) move the dropdown selection, Tab accepts the highlighted candidate
  (inserting e.g. `repo:sase-nvim ` with a trailing space), Escape and Enter as above. The existing `check_action`
  VimTextArea guard already keeps priority Tab bindings out of the way.

Completion candidates by context (all sourced from data pushed in via `set_completion_sources` — never fetched by the
widget):

- Key position → the five token keys with one-line hints (e.g. `since:  ·  Nh/Nd/Nw, today, YYYY-MM-DD`), plus a dim
  free-text hint row.
- `repo:` → repo names and aliases from the last collection result and resolved scope.
- `author:` → distinct author names from loaded commits.
- `since:`/`until:` → curated forms: `1h`, `24h`, `today`, `yesterday`, `3d`, `7d`, `2w`, and a `YYYY-MM-DD` template
  row.
- `limit:` → `40`, `100`, `200`, `all`.
- Candidates filter by the typed prefix; an empty match set collapses the dropdown rather than showing an empty box.

Styling lives in `src/sase/ace/tui/styles.tcss` (the `CommitFiltersModal` rules are deleted in the next phase). The bar
is `display: none` until opened; opening must not shift the timeline scroll position more than the bar's own height.

Widget-level tests: dropdown context switching as the cursor moves, candidate filtering, Tab acceptance producing the
expected text, message emission (QueryChanged/Submitted/Dismissed ordering), and Escape's two-stage behavior.

## Pane wiring, live preview, and keymaps

### Mounting and open/close semantics

- Mount `CommitFilterBar` at the top of `CommitsPane.compose` (above `#commits-info`).
- Opening: build the prefill via `to_query_string(self.filters)`, push current repo/author completion sources, remember
  the pre-open `CommitLogFilterValues` for Escape-revert, show + focus the bar.
- Enter (valid query): adopt the parsed values as `self.filters`, close the bar, focus the timeline, and run
  `_state_changed()` (which bumps the generation and schedules an authoritative collection) unless the debounced
  collection for exactly these values already reconciled.
- Enter (invalid query): keep the bar open, `notify` the parse error.
- Escape: restore the remembered values (and their last authoritative result if still cached), close the bar, focus the
  timeline.

### Two-tier live filtering

- **Tier 1 — instant preview (every `QueryChanged`, synchronous):** parse; on success, apply `commit_matches` + limit
  against the _preview base_ and repaint the timeline via the existing `update_result` path (which already guards
  programmatic `OptionList` updates and preserves selection by stable target). On parse failure, update the bar status
  only. This is pure CPU over at most a few hundred already-loaded rows — no I/O, no subprocesses, honoring the
  keystroke-path rules.
- **Preview base:** cache the most recent authoritative `VcsLogResult` together with the filter values and scope key
  (`project_scope`, `all_projects`, `include_sdd`) it was collected under, preferring the broadest coverage seen for the
  current scope (empty backend filters, largest limit). Invalidate on scope-key change. The preview is marked `exact`
  when the base fully covers the query (base collected unfiltered and not truncated at its limit, or base filters equal
  the query); it is marked `preview` otherwise — e.g. widening `since:` beyond the base or filtering to a repo whose
  rows were crowded out of a truncated merge.
- **Tier 2 — debounced authoritative collection:** ~300 ms after the last valid `QueryChanged` (reuse the
  `DetailPanelDebouncer` pattern from `util/debounce.py` with a longer delay), adopt the parsed values and schedule the
  existing generation-guarded thread-worker collection (`no_fetch=True`, so no network or credential prompts). When it
  lands it replaces the preview, updates the preview base, refreshes the bar's match count/`exact` tag, and refreshes
  completion sources (new authors/repos). The existing pending/coalescing flags already give last-request-wins;
  Escape-revert bumps the generation so stale filtered results are dropped.
- The header keeps its existing `refreshing…` indicator while tier 2 runs; text terms are also applied on top of
  authoritative results (backend can't filter subjects).

### Keymap routing and surface updates

- `check_action` (`src/sase/ace/tui/app.py`): allow `edit_query` when the commits sub-tab is active (do **not** add it
  to `NON_PRS_ARTIFACT_ACTIONS`, which would leak it to Bugs/Plans).
- `action_edit_query` (`actions/base.py`): route to the commits filter bar when on the Commits sub-tab, exactly like the
  existing Agents-tab routing.
- `action_commits_filters` (`actions/artifacts_commits.py`): repoint `pane.show_filters()` at opening the bar. The
  `commits_filters` registry action, its `f` default, and user overrides all keep working — no `default_config.yml` or
  `keymaps/types.py` changes required.
- Delete `src/sase/ace/tui/modals/commit_filters_modal.py`, its `styles.tcss` rules, and its `modals/__init__` export.
- Update `commands/_app_metadata.py` (`commits_filters` label → "Commits: filter bar") and `commands/availability.py`
  (allow `app.edit_query` on the commits sub-tab so the palette shows it there).
- Update the pane hint bar (`build_commits_hints`) to advertise the `edit_query` key (`/ filter`), and the help modal's
  Commits Pane section (`help_modal/changespecs_bindings.py`) to describe the bar and its query tokens — respecting the
  help modal's 57-char box conventions.

### Tests

Rework `test_commits_pilot_drives_filters_detail_copy_modal_and_toggles` (it currently asserts the modal and the old
footer text) into pilot coverage of: `/` opens the bar prefilled, typing narrows the timeline live (preview), the
debounced authoritative collection is scheduled with the parsed filters, Enter commits + closes + focuses the timeline,
Escape reverts, `f` opens the same bar, and `/` on Bugs/Plans stays inert. Add a regression that typing in the bar never
runs the collector synchronously on the event loop (collector invocations only via worker), and that rapid keystrokes
coalesce to one scheduled collection.

## Visual polish and verification

- Extend `tests/ace/tui/visual/test_ace_png_snapshots_commits.py` with goldens for: bar open with a prefilled query and
  match count; completion dropdown open on `repo:` with candidates; a narrowed timeline with restyled filter chips; and
  the inline parse-error state. Goldens live in `tests/ace/tui/visual/snapshots/png/`; accept intentional changes with
  `--sase-update-visual-snapshots`. Existing commits goldens will need re-acceptance if the chip restyle changes the
  header.
- Final copy pass: chips use query-token vocabulary, bar hints and help modal text are consistent, and the empty-state
  message ("No commits match the current scope and filters.") still reads correctly with the bar.
- Responsiveness verification per the perf runbook: a `SASE_TUI_PERF=1` spot-check that j/k stays under budget with the
  bar open, a typing burst in the bar produces no stall-watchdog entries (`~/.sase/logs/tui_stalls.jsonl`), and the
  commits pilot tests pass under `just test`. Run `just check` for the full gate.

## Risks and mitigations

- **Preview/authoritative disagreement confusing the user** — mitigated by the explicit `preview`/`exact` tag and the
  short debounce before reconciliation.
- **Key leakage while the bar is focused** — the input is a `VimTextArea` subclass, which already suppresses app
  priority bindings; pilot tests assert j/k/f do not fire pane actions while typing.
- **OptionList echo bugs from frequent repaints** — reuse the existing guarded `update_result` path only; never assign
  `highlighted` outside it.
- **Scope changes while the bar is open** (`p`, `d`, `a`) — the scope key invalidates the preview base; the simplest
  correct behavior (re-run the preview against the new base once its collection lands) is acceptable.
- **Stale filtered collections after Escape** — generation bump on revert drops them (existing guard).
