---
tier: tale
title: Agents filter indicator shows bracketed counts and an edit-query key hint
goal:
  "The Agents-tab info panel renders an active filter as `filter: <query>
  [matched/loaded] (/)`, with the key hint drawn from the edit_query keymap and omitted
  when that key is unbound."
size: small
proposed_by: bbugyi200.apollo.1h.f0.f0.f0.w2.w0.w0
create_time: 2026-09-22 15:18:12
status: wip
---

# Plan: Agents filter indicator: `[matched/loaded]` counts and an edit-query key hint

## Goal

The Agents-tab info panel (top-right header) currently renders an active filter as:

```
filter: NOT machine:apollo  15/86
```

Change it to:

```
filter: NOT machine:apollo [15/86] (/)
```

- Drop the double space between the query and the counts; use a single space.
- Wrap the `matched/loaded` counts in square brackets.
- Append a ` (/)` hint showing the key that edits the query. The key comes from the
  keymap registry (`edit_query`, `slash` by default), not a hard-coded `/`, and follows
  the same pattern the panel already uses for the `view: … (key)` and `group: … (key)`
  hints.

This is presentation-only Textual rendering, so it stays in this repo. No Rust core,
wire, or keymap config changes are needed. `edit_query` already exists in
`src/sase/default_config.yml` as `"slash"`, so `default_config.yml` stays unchanged.

## Current Code

`src/sase/ace/tui/widgets/agent_info_panel.py`, `AgentInfoPanel._build_display_text`:

```python
self._search_query_click_span = None
if self._search_query_rich is not None:
    self._append_separator(text)
    text.append("filter: ", style="dim italic")
    click_start = text.cell_len
    text.append_text(self._search_query_rich)
    if self._search_query_seeded:
        text.append(" seeded", style="dim")
    if self._search_query_match_count is not None:
        matched, loaded = self._search_query_match_count
        text.append(f"  {matched}/{loaded}", style="dim")
    self._search_query_click_span = (click_start, text.cell_len)
elif self._search_query:
    self._append_separator(text)
    text.append("filter: ", style="dim italic")
    text.append(self._search_query, style="bold #FFD700")
    if self._search_query_seeded:
        text.append(" seeded", style="dim")
if self._search_query and self._search_query_partial_history:
    text.append("  filtered on recent history; loading full history...", ...)
```

The existing key-hint pattern a few lines below (view and group segments):

```python
grouping_key = self._registry.app.choose_agent_grouping
if not is_unbound_key(grouping_key):
    key = key_display_name(grouping_key)
    text.append(f" ({key})", style="dim")
```

`key_display_name("slash")` returns `"/"`. `is_unbound_key` and `key_display_name` are
already imported in this module. The bare-`/` binding is
`Binding("slash", "edit_query", ...)` in `src/sase/ace/tui/bindings.py`, and on the
Agents tab `action_edit_query` delegates to `_edit_agent_search_query`. That makes
`self._registry.app.edit_query` the correct key to show.

## Implementation

### 1. `src/sase/ace/tui/widgets/agent_info_panel.py`

1. Add a small private helper next to `_append_separator`, for example:

   ```python
   def _append_edit_query_hint(self, text: Text) -> None:
       edit_key = self._registry.app.edit_query
       if is_unbound_key(edit_key):
           return
       text.append(f" ({key_display_name(edit_key)})", style="dim")
   ```

2. In the rich branch, change the count rendering to
   `text.append(f" [{matched}/{loaded}]", style="dim")`: one leading space, square
   brackets, and the whole chip kept `dim` as it is today.
3. Keep `self._search_query_click_span = (click_start, text.cell_len)` right after the
   counts, so the clickable span still covers the query, the optional ` seeded` tag and
   ` [M/N]`. Then call `_append_edit_query_hint(text)` after setting the span. The hint
   stays outside the click span. That keeps the "click the query segment" contract
   unchanged, and the hint reads as a key legend, like the view and group hints.
4. In the plain `elif self._search_query:` fallback branch (no rich text, no counts),
   also call `_append_edit_query_hint(text)` after the optional ` seeded` tag. Both
   render paths then advertise the key the same way (`filter: status:FAILED (/)`).
5. If the rich branch has no `match_count`, it still gets the hint and simply shows no
   bracket chip (`filter: q (/)`).
6. Leave the partial-history notice unchanged. It follows the hint, giving
   `filter: status:FAILED [1/5] (/)  filtered on recent history; loading full history...`.

Resulting shapes:

| State                  | Rendered                                 |
| ---------------------- | ---------------------------------------- |
| rich + counts          | `filter: NOT machine:apollo [15/86] (/)` |
| rich + seeded + counts | `filter: project:demo seeded [1/1] (/)`  |
| rich, no counts        | `filter: status:FAILED (/)`              |
| plain fallback         | `filter: status:FAILED (/)`              |
| `edit_query: unbound`  | the same, minus the ` (/)` suffix        |

### 2. Unit tests

Update the existing assertions for the new format:

- `tests/ace/tui/widgets/test_agent_info_panel_filter.py`
  - `"filter: status:FAILED  3/12"` becomes `"filter: status:FAILED [3/12] (/)"`
  - `"filter: project:demo seeded  1/1"` becomes
    `"filter: project:demo seeded [1/1] (/)"`
  - the partial-history assertion becomes
    `"filter: status:FAILED [1/5] (/)  filtered on recent history; loading full history..."`
  - `test_search_query_click_span_covers_only_the_query_segment`: the expected span text
    becomes `"status:FAILED [1/5]"`. Also assert that the text after the span starts
    with `" (/)"`, so the hint is proven to sit outside the span.
  - the plain-fallback test: assert `"filter: status:FAILED (/)"`.
- `tests/ace/tui/widgets/test_agent_info_panel_grammar.py`
  (`test_filter_and_partial_history_join_with_dot_and_keep_click_span`)
  - `" · filter: status:FAILED  1/5  filtered on recent history;"` becomes
    `" · filter: status:FAILED [1/5] (/)  filtered on recent history;"`
  - the expected click-span text becomes `"status:FAILED [1/5]"`.

Add new tests to `test_agent_info_panel_filter.py`:

- `edit_query` unbound omits the hint: build the panel, then
  `panel.set_keymap_registry(load_keymap_registry({"keymaps": {"app": {"edit_query": "unbound"}}}))`
  (same pattern as `test_unbound_refresh_omits_hint` in the grammar tests). Render a
  rich query with counts and assert that `"filter: status:FAILED [1/5]"` is present and
  `"(/)"` is absent.
- A rebound `edit_query` (for example `"e"`) renders ` (e)` instead of ` (/)`.
- The bracketed count and the hint are both styled `dim`. Use the existing
  `style_for_plain_segment` helper from `._agent_info_panel_helpers`, or
  `style_at_plain_index`, whichever fits.

Run `rg -n '[0-9]+/[0-9]+' tests/ace/tui | rg -i 'filter'` and
`rg -n 'filter: ' tests/ace/tui` to find any other test that asserts the old
double-space format (for example `tests/ace/tui/test_agents_tab_current_project_seed.py`
asserts substrings such as `"filter: project:sase seeded"`, which still match). Update
only the assertions the new format actually breaks.

### 3. Visual PNG goldens

At least `agents_filter_bar_idle_readout_120x40` (from
`tests/ace/tui/visual/test_ace_png_snapshots_agents_filter_bar.py`) renders the filter
readout and will change. Other goldens that show an active Agents filter may change too.
Follow the SASE TUI memory (`tui.md` → `tui_screenshot.md`, read via
`/sase_memory_read`) golden-maintenance procedure: regenerate with
`just fix-tui-screenshots` (targeted selectors after `--` first, e.g. the filter-bar
test file, then a full run to catch any other affected goldens), using `/sase_monitor`
as that memory requires. Inspect every changed golden in the retained report. The only
visible difference should be the filter readout text (`[M/N] (/)`) and whatever that
small width change shifts in the header's right-aligned segments. Do not accept
unrelated diffs.

## Verification

1. Read `lint_and_test.md` via `/sase_memory_read` and run `just check` (not
   `just check-full`).
2. Targeted pytest:
   `pytest tests/ace/tui/widgets/test_agent_info_panel_filter.py tests/ace/tui/widgets/test_agent_info_panel_grammar.py tests/ace/tui/test_agents_tab_current_project_seed.py`
3. Visual goldens are regenerated and inspected as described above.
4. Optional live sanity check: `sase screenshot` of the Agents tab with a committed
   query shows `filter: <query> [M/N] (/)` in the top-right header.

## Out of Scope

- No change to what clicking the query does or to the `FilterClicked` message.
- No change to the partial-history notice wording or to the `seeded` tag.
- No keymap default changes, so `src/sase/default_config.yml` stays unchanged.
