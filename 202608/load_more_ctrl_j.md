---
tier: epic
status: done
title: Ctrl+J loads more list entries and Ctrl+K unloads them
goal: "Every ACE list that today pages with Ctrl+K loads the next page with Ctrl+J and
  unloads that page with Ctrl+K, using one configurable page size that defaults to 100.
  Every Artifacts sub-tab speaks the same limit:N query token, starts with it in its
  default query, and uses those two keys to raise or lower the cap.

  "
phases:
  - id: config
    title: Page-size config and shared limit helpers
    depends_on: []
    size: small
    description:
      "config: add ace.page_size (default 100) and the shared limit-token helpers every
      later phase uses."
  - id: modals
    title: Rebind existing load-more panels
    depends_on:
      - config
    size: medium
    description:
      "modals: switch prompt-history, alias-history, and revive-agent paging to Ctrl+J /
      Ctrl+K with plus-or-minus page-size, not doubling."
  - id: query-limit
    title: Host-owned limit token on every Artifacts pane
    depends_on:
      - config
    size: medium
    description:
      "query-limit: accept limit:N on every Artifacts dialect, inject it into each
      pane's default query, and apply it as a post-match cap."
  - id: artifacts-keys
    title: Artifacts Ctrl+J and Ctrl+K
    depends_on:
      - query-limit
    size: medium
    description:
      "artifacts-keys: bind Ctrl+J / Ctrl+K on the Artifacts tab to raise or lower the
      committed limit and grow snapshots when the cap outruns the loaded page."
proposed_by: bbugyi200.athena.086
bead_id: sase-r6
create_time: 2026-09-09 19:50:49
---

- **PROMPT:**
  [prompts/202608/load_more_ctrl_j.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/load_more_ctrl_j.md)
- **BEAD:**
  [sase-r6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-r6/README.md)

# Plan: Ctrl+J loads more list entries and Ctrl+K unloads them

## Goal

Give ACE one paging chord pair and one page size:

- **Ctrl+J** loads `ace.page_size` more entries into the current list (default **100**).
- **Ctrl+K** undoes that step: unload the last `ace.page_size` entries, never dropping
  below one page.
- Every Artifacts sub-tab — Stitches, Patches, Beads, Files, and every document-provider
  pane (Plans, Research, and third-party kinds) — starts with a `limit:<page_size>`
  token in its default query. The same chords rewrite that token.

Do not add a feature flag. This replaces the existing load-more chord and adds a
permanent config field; users are meant to keep both.

Do not move this into `sase-core`. `limit:` is a host-owned presentation cap, not a
row-matching field. Follow the Stitches pattern: parse it in Python, strip it before
`evaluate_artifact_query_many`, and slice after the match. Adding `limit` as a
filterable profile field would make Rust look for a `limit` property on rows.

## Non-goals

These chords already mean something else. Leave them alone:

| Surface                   | Keys                | Keep doing                                                                                     |
| ------------------------- | ------------------- | ---------------------------------------------------------------------------------------------- |
| Prompt textarea           | `Ctrl+K`            | Open prompt history from a single-line prompt                                                  |
| Prompt textarea           | `Ctrl+J`            | Insert newline / split list items                                                              |
| Agents tab                | `Ctrl+J` / `Ctrl+K` | Next / previous agent metadata section (already gated to the Agents tab in `check_app_action`) |
| Wait modal                | `Ctrl+J` / `Ctrl+K` | Next / previous field                                                                          |
| Saved-agent-group revival | `PageDown`          | Load more saved groups                                                                         |

Do not remap Launch Control `H`, do not change `llm_provider.model_alias_history_limit`
(that remains the **initial** alias-history window), and do not change Files' first-page
disk fetch (`FILES_FIRST_PAGE_LIMIT`, currently 500) except to grow it when the query
cap outruns the loaded snapshot.

## Current load-more inventory

These are the only surfaces that bind `Ctrl+K` to **load more entries**:

1. **Prompt history modal** (`src/sase/ace/tui/modals/prompt_history_modal.py`) — page
   size 250, cursor paging, intercepts `Ctrl+K` while the filter input is focused.
2. **Alias-history modal** (`src/sase/ace/tui/modals/alias_history_modal.py`) — starts
   at `model_alias_history_limit` (10) and **doubles** on each `Ctrl+K` via
   `doubled_alias_history_limit`.
3. **Revive-agent modal** (`src/sase/ace/tui/modals/revive_agent_modal.py`) — page size
   250, archive paging, intercepts `Ctrl+K` in `on_key`.

Artifacts Stitches already parses `limit:N` / `limit:all` in `sase.vcs_log.filter_query`
and slices after matching (`commits_filtering.py` strips `limit` from the Rust row
query). Other Artifacts dialects reject `limit:` as an unknown key. Beads default to
`-status:closed`; Plans and Files default to an empty filter; Patches start from the ACE
query string; Stitches default is `sidecar:false merges:hide since:24h` with no cap.
App-level `Ctrl+J` / `Ctrl+K` already no-op on the Artifacts tab because
metadata-section navigation is Agents-only.

## Shared semantics

`N = ace.page_size` (integer `>= 1`, default 100).

**Load more (Ctrl+J)**

- Numeric cap `L`: set the cap to `L + N`.
- Unlimited (`limit:all`, `limit:0` on Stitches, or a user-deleted token): no-op;
  everything matching is already visible.
- Prompt-history / revive: load one more page of `N` records. No-op when exhausted or a
  load is already in flight.
- Alias history: set `limit_per_alias` to `current + N` and reload through the existing
  worker. Stop doubling.

**Unload (Ctrl+K)**

- Numeric cap `L`: set the cap to `max(N, L - N)` when `L >= N`, else leave it (never
  jump a user-typed `limit:20` up to 100, and never show an empty list by going to 0).
- Unlimited: set `limit:N` (first unload introduces the default page).
- Prompt-history / revive: drop the last loaded page of `N` items and rewind the
  cursor/page stack so the next load-more fetches that page again. No-op when only the
  first page is loaded.
- Alias history: `limit_per_alias = max(initial, current - N)` where `initial` is
  `model_alias_history_limit`. Reload. No-op at the initial window.

Preserve highlight/selection when the selected row is still visible; otherwise select
the new last row. Intercept the chords while a modal filter input is focused, the same
way prompt-history already intercepts `Ctrl+K`. Never run catalog, git, or snapshot IO
on the UI thread; reuse each surface's existing worker/proc path (`tui_perf.md`).

## Phase `config`

Add the config field and the helpers later phases import. No TUI behavior change yet.

1. `ace.page_size: 100` in `src/sase/default_config.yml` under `ace:`, with a comment
   that it is the Ctrl+J / Ctrl+K step and the default Artifacts `limit:` value.
2. Matching property on `ace` in `src/sase/config/sase.schema.json`: integer,
   `minimum: 1`, `default: 100`. `tests/test_config_schema.py` already requires the
   bundled default to validate.
3. Getter `get_ace_page_size()` next to other ACE config readers. Invalid or missing
   values fall back to 100. Tests that mutate config call `clear_config_cache()`.
4. Shared module (suggested: `src/sase/ace/query/limit_token.py`) with no Textual
   imports:
   - `extract_limit(query) -> tuple[str, int | None]` — remainder plus cap; `None` means
     unlimited (`all`, `0`, or absent). Reject negated `-limit:`, empty values, and
     non-integers the same way Stitches does.
   - `ensure_limit(query, n) -> str` — append `limit:n` when no limit token is present;
     leave an explicit user token alone.
   - `replace_limit(query, n) -> str` — write a numeric `limit:n`, preserving the rest
     of the query and canonical token order for that dialect when a serializer exists.
   - `adjust_limit(current, page_size, direction) -> int` — the plus-or-minus / floor
     rules above.

5. Unit tests for extract/ensure/replace/adjust, including quotes, `limit:all`,
   duplicate tokens (last wins or error — pick one and pin it; Stitches errors on
   duplicates if it already does), and the floor rules.
6. Document `ace.page_size` in `docs/configuration.md` next to the other `ace` fields.

## Phase `modals`

Depends on `config`. Does not touch Artifacts queries.

1. Prompt history: bind `Ctrl+J` to `load_more` and `Ctrl+K` to a new `unload` action
   (priority bindings plus `on_key` intercepts for both, replacing the current `Ctrl+K`
   intercept). Initial page size and increment both come from `get_ace_page_size()`.
   Keep a stack of loaded pages/cursors so unload drops only the last page and the next
   load-more can refetch it. Update `tests/ace/tui/modals/test_prompt_history_modal.py`
   (today it asserts `Ctrl+K` is load-more and not `Ctrl+D`).
2. Alias history: stop calling `doubled_alias_history_limit`. Load-more adds `N`; unload
   subtracts `N` down to the initial config limit. Update footer markup in
   `alias_history_rendering.py`, `tests/test_alias_history_modal.py`, and `docs/ace.md`
   (the Launch Control history table currently says Ctrl+K doubles the window).
3. Revive-agent: same chord swap and page size `N`. Keep the existing off-thread
   `_load_more_async` seam; add `_unload` that trims the in-memory agent list and
   restores the previous page cursor. Update
   `tests/ace/tui/modals/test_revive_agent_modal.py` and the `^k: +N more` hint in
   `_hints_text`.
4. Do not change `test_prompt_history_trigger.py` or `test_keymaps_e2e.py`'s prompt-bar
   `Ctrl+K` (those open history, they do not page it).
5. CHANGELOG under Features: ACE list paging is now Ctrl+J / Ctrl+K with a configurable
   page size.

## Phase `query-limit`

Depends on `config`. Can run in parallel with `modals`. After this phase, users can type
`limit:100` on every Artifacts pane even before the new chords land.

`limit:` is a **host-owned cap**, not a row property:

- Extract it from the query string before dialect parse / Rust eval.
- Match rows against the remainder.
- Slice the matched list to the cap (Stitches already does this in `_filtered_result`;
  copy that shape).
- Serialize it back into the visible query and chips.

Do **not** add `QueryFieldSpec(key="limit")` as a filterable row field on Beads / Plans
/ Files / Patches / provider profiles. Stitches may keep its existing parser field
because it already treats limit as a cap and strips it in `_row_query_string`.

Wire the shared token through every pane's filter session:

| Pane               | Parser today                                    | Default query today                   | Work                                                                                          |
| ------------------ | ----------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| Stitches           | `parse_commit_filter_query` already has `limit` | `sidecar:false merges:hide since:24h` | `ensure_limit` at resolve time; update the schema/comment that says omitted limit is uncapped |
| Beads              | `sase.bead.filter_query` — unknown key today    | `-status:closed`                      | Accept `limit`; `default_bead_filter_values()` includes `limit:N`; completion + help          |
| Plans              | `sase.plan_search.filter_query`                 | empty                                 | Same                                                                                          |
| Files              | `files_filtering.py`                            | empty                                 | Same; display slice is independent of `FILES_FIRST_PAGE_LIMIT`                                |
| Patches            | boolean `parse_query` / profile                 | ACE query string                      | Extract `limit:` before the boolean parse so it is never a `PropertyMatch`; apply after eval  |
| Document providers | profile-driven flat parse                       | empty                                 | Same extractor as other flat panes so Plans/Research/custom kinds inherit it                  |

Default-query rule: at pane init, `ensure_limit(default, get_ace_page_size())`. If a
user-configured Stitches `ace.artifacts.stitches.default_query` already contains
`limit:`, leave it. Changing `ace.page_size` then changes the Ctrl+J step and any
default that had no explicit limit; it does not rewrite a user-authored `limit:40`.

Also:

- Filter-bar `KEY_COMPLETIONS` / value completions (`40`, `100`, `200`, `all` as
  Stitches already offers).
- Help copy in `patches_artifact_bindings.py` for Beads / Files / Plans / provider
  sections (Stitches already documents `limit:N / limit:all`).
- Query-profile tests that currently enumerate filterable keys must **not** start
  treating `limit` as a row field.
- Coverage labels already distinguish truncated vs exact on Stitches; reuse that for
  other panes when `len(matched) > limit`.
- `docs/query_language.md` and `docs/ace.md` editing-queries / Artifacts sections:
  `limit:` is a host cap on every Artifacts pane.

Keep collection controls separate from row membership (already an Artifacts invariant).
Stitches may still pass the numeric cap into git collection; Files still fetch a first
page of 500 then a full index; Plans may still extend the deep archive. The cap only
decides how many matched rows the list shows.

## Phase `artifacts-keys`

Depends on `query-limit`.

1. Add configurable app keymaps in `src/sase/default_config.yml`:

   ```yaml
   artifacts_load_more: "ctrl+j"
   artifacts_unload: "ctrl+k"
   ```

   Thread them through `AppKeymaps`, `keymaps/metadata.py`, `bindings.py` /
   `DEFAULT_BINDINGS`, and the keymap loader tests.

2. `check_app_action`: enable the two actions only on the Artifacts tab (any sub-tab).
   Metadata-section actions stay Agents-only, so the shared chords do not double-fire.
   Disable both while the prompt textarea is focused so prompt newline / open-history
   keep priority (the prompt widget already intercepts these keys; do not regress
   `test_keymaps_e2e.py`).

3. One pair of app actions (`action_artifacts_load_more` / `action_artifacts_unload`)
   that:
   - Read the active pane's committed query (or live query if that pane's filter session
     is open).
   - `extract_limit` → `adjust_limit` → `replace_limit`.
   - Commit through the pane's existing query-commit path so chips, history, reveal-lens
     restoration, and selection restore all run. If the filter editor is open, update
     its text to match.
   - No-op with no query rewrite when already at the floor (unload) or already unlimited
     (load more).

4. When the new cap exceeds the loaded snapshot, grow data off-thread:
   - **Files:** if `limit >` loaded row count and the snapshot is not `complete`,
     request `full=True` (existing path) then slice.
   - **Stitches:** existing collection already follows `values.limit`.
   - **Plans:** if deep-archive coverage is a lower bound, let the existing archive
     worker extend; do not load the archive on the UI thread.
   - **Beads / Patches / providers:** in-memory snapshots; just re-slice.

5. Footer / help / command palette: show the chords on Artifacts surfaces. Update
   `docs/ace.md` Artifacts key tables.

6. Tests (prefer pane unit / AcePage tests over full visual unless pixels change):
   - Default query for each built-in pane contains `limit:<page_size>`.
   - Ctrl+J on Beads `-status:closed limit:100` becomes `limit:200` and the list grows
     by at most 100.
   - Ctrl+K from 200 returns to 100 and is a no-op at 100.
   - User-typed `limit:all` then Ctrl+K becomes `limit:100`.
   - Custom `ace.page_size: 25` is honored.
   - Agents-tab Ctrl+J still cycles metadata sections.
   - Prompt-bar Ctrl+K still opens history.
   - Query history can return to the pre-Ctrl+J query.

7. PNG snapshots: update goldens only when the default query or footer text changes
   pixels (alias-history footer, Stitches default-query shots that do not pin their own
   query). Use `--sase-update-visual-snapshots` for intentional diffs; inspect
   `.pytest_cache/sase-visual/` first.

## Verification

Each phase runs `just install` then `just check` on its tree. The combined landing tree
runs `just check-full` through `/sase_monitor` (`TESTING` / `TESTED`). If PNG goldens
change, run `just test-visual` on the affected files.

## Docs and copy

- `docs/configuration.md` — `ace.page_size`
- `docs/ace.md` — alias-history table, prompt-history modal paging, revive hints,
  Artifacts `limit:` and the new chords. Leave the prompt-input "Ctrl+K opens history"
  row and the Agents-tab metadata section rows unchanged.
- `docs/query_language.md` — host-owned `limit:` on Artifacts panes
- CHANGELOG Features for the new chords and config field
- Stitches schema description currently says omitted `limit:` is uncapped; defaults are
  now capped, user-deleted tokens stay uncapped
