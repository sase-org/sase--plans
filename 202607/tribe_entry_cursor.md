---
tier: tale
title: Show which TRIBE MEMBERS row the panel-entry key selects
goal:
  The tribe summary in the agent metadata panel marks, with an exact and always-correct cursor, the TRIBE MEMBERS row
  that pressing the panel-entry key (`l` / `Esc`) will select.
create_time: 2026-07-29 06:55:52
status: done
---

- **PROMPT:** [prompts/202607/tribe_entry_cursor.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/tribe_entry_cursor.md)
- **AGENTS:**
  - [bbugyi200.athena.nu--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.nu.md#member-code)
  - [bbugyi200.athena.nu--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.nu.md#member-plan)
- **COMMITS:**
  - [2110954](https://github.com/sase-org/sase/commit/2110954eb1d6833a03228298d5716681d484980a) — feat(tui): show tribe panel entry target

# Plan: Panel-entry cursor for the TRIBE MEMBERS roster

## Problem

When whole-panel focus is on an expanded tribe panel, the metadata panel renders the tribe document: `TRIBE` header,
`NEEDS ATTENTION`, a numbered `TRIBE MEMBERS` roster, then the enrichment sections. Pressing `l` (or `Esc`) "enters" the
panel and lands on some row, but the document gives no hint which one. The user must press the key to find out, and the
answer is not obvious: it is the panel's _remembered_ selection when that row is still rendered, otherwise the panel's
first rendered row — which is not necessarily roster entry `00`, because the roster is in panel order while the entry
destination follows the _grouped, fold-aware render order_.

Add a visual indicator to the `TRIBE MEMBERS` section marking the destination row.

## Where the behavior actually lives (read this before designing anything)

- `l` on the Agents tab → `AgentFoldingMixin.action_expand_or_layout` (`src/sase/ace/tui/actions/agents/_folding.py:54`)
  → `_expand_fold` (`src/sase/ace/tui/actions/agents/_folding_agent_groups.py:179`).
- With whole-panel focus on an **expanded** panel, `_expand_fold` delegates to `_exit_expanded_panel_focus`
  (`src/sase/ace/tui/actions/agents/_selection.py:136`). `Esc` reaches the same method from
  `src/sase/ace/tui/actions/_event_keyboard.py:88`. These are the only two callers, so one resolver covers both keys.
- `_exit_expanded_panel_focus` picks its destination with exactly this rule:

  ```python
  stops = self._panel_navigation_stops(include_panel_focus=True)
  remembered = self._panel_selection_memory.get(focus.panel_key)
  destination = remembered if remembered in stops else (stops[0] if stops else None)
  ```

  A stop is `("agent", global_idx)` or `("banner", group_key)` (a _collapsed group banner_, e.g. the `Done` banner when
  grouping by status). `include_panel_focus=True` is mandatory: without it `_panel_navigation_stops`
  (`src/sase/ace/tui/actions/agents/_navigation_order.py:110`) returns `[]` while whole-panel focus is active.

- With whole-panel focus on a **collapsed** panel, `l` expands the panel and _keeps_ whole-panel focus
  (`_folding_agent_groups.py:194-202`); it does not enter. The footer already says `l expand panel` in that state
  (`src/sase/ace/tui/widgets/_keybinding_bindings.py:194-206`).
- `_panel_selection_memory` (`src/sase/ace/tui/actions/agents/_selection.py:71`) is only written while whole-panel focus
  is _not_ active, so it is frozen for as long as the tribe document is on screen. The _stops_ are not frozen: a 10 s
  auto-refresh, a kill/dismiss, a fold change (`h`/`H`), or a grouping change (`o`) can all move or delete the
  destination.
- The tribe document is built by `build_tribe_detail_text`
  (`src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py:577`) from an `AgentTribeSummarySnapshot`
  (`src/sase/ace/tui/models/agent_tribe_summary.py`), which `_focused_tribe_summary`
  (`src/sase/ace/tui/actions/agents/_selection.py:187`) builds from the focused panel's agent slice. The roster itself
  is rendered by the shared `append_member_roster` (`src/sase/ace/tui/widgets/prompt_panel/_member_roster.py:172`),
  which clan panels also use.
- Roster entries (`snapshot.units`) are the panel's **top-level units** — `tribe_unit_roots` keeps rows for which
  `agent_is_tree_child` is false — so clan containers, families, workflows, and lone lanes. A navigation stop can point
  at a _nested_ row (a family member, a workflow child) whose owning unit is found with `presentation_anchor_lookup`
  (`src/sase/ace/tui/models/_agent_tree.py:80`).

## Design

### Core invariant: one resolver, no re-derivation

The single biggest reliability risk is a second implementation of "where does `l` go" drifting from the real one. So:
extract the destination rule from `_exit_expanded_panel_focus` into one shared function and have **both** the key
handler and the renderer call it. The indicator is then correct by construction, not by agreement.

New module `src/sase/ace/tui/actions/agents/_panel_entry_target.py`:

```python
def resolve_panel_entry_stop(owner, panel_key) -> PanelSelectionStop | None:
    """Return the stop that entering `panel_key` selects, or None."""
```

It contains the `remembered if remembered in stops else stops[0]` rule and nothing else. `_exit_expanded_panel_focus` is
rewritten to call it (behavior must be byte-identical, including the `stops_fn(include_panel_focus=True)` / `TypeError`
fallback shim it currently carries).

### What gets shown

Only when the focused panel is **expanded** (`snapshot.panel_collapsed is False`). On a collapsed panel `l` expands
rather than enters, so showing an entry cursor there would be a lie; render nothing.

1. **Row cursor.** Every roster row gains a fixed two-cell gutter _before_ its number chip. The destination row gets
   `❯ `; every other row gets two spaces, so the column never shifts as the target moves. Nested child rows (rendered
   from `FoldLevel.FULLY_EXPANDED`) get the same two-cell pad prepended to their `   └─` branch prefix so the tree stays
   aligned under its unit.

   ```text
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ▸ ❖ TRIBE MEMBERS · 16 · l ❯ ns
        00  nt · workflow · ▶ RUNNING · opus · 7m51s
   ❯    01  ns · family · ▶ WORKING TALE · mixed · 10m36s [R1 D1]
        02  nr · family · ▶ RUNNING · mixed · 18m51s [R2 D1]
   ```

2. **Heading affordance.** The `TRIBE MEMBERS` heading gains a trailing ` · l ❯ <label>` clause. It names the key so the
   glyph is self-explanatory on first sight, reuses `❯` so the heading and the row read as one statement, and stays
   correct in the cases where no row can carry the cursor.

### Glyph, colors, and why

- `❯` (U+276F) appears nowhere in `src/sase` today (verified). It cannot be confused with the fold glyphs `▸ ▾ ▼ ◆`
  (`_fold_language.py:14`) or the status glyphs `▲ ◐ ▶ … ⏳ ✗ ✓` (`src/sase/agent/status_buckets.py:17`).
- Cursor style: `bold #FFFFFF`. The roster's semantic colors are already spent — accent yellow for the number chips,
  status colors, `#5FD7FF` unread, `#00D700` marks — so a neutral bright white cursor reads as pure chrome and cannot be
  mistaken for a status. It is also the one style that survives the PNG/SVG snapshot suite and monochrome terminals.
- Heading clause styles: `" · "` dim; the key label `bold #87D7FF` (the document's field-label blue, so it reads as a
  key cap); `"❯ "` `bold #FFFFFF` (identical to the row cursor); the destination label `bold {TRIBE_IDENTITY_COLOR}`
  (the same style unit labels already use in `_append_triage_line`).
- The key label is **not** hardcoded: `expand_or_layout` is rebindable (`src/sase/default_config.yml:268`). Resolve it
  app-side with `footer_key_display(self._keymap_registry.app.expand_or_layout)` and carry it on the entry target;
  default to `"l"` so pure-model tests and any pre-registry render still produce sensible text.

### Destination cases (all four must be handled)

| Destination stop                                                            | Row cursor                                                | Heading clause                                                                                     |
| --------------------------------------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `("agent", idx)` where the row _is_ a top-level unit                        | on that unit                                              | `· l ❯ ns`                                                                                         |
| `("agent", idx)` where the row is nested inside a unit                      | on the **owning unit** (via `presentation_anchor_lookup`) | `· l ❯ ns › ns--2`                                                                                 |
| `("banner", group_key)` — a collapsed group banner                          | none (no roster row is the destination)                   | `· l ❯ Done (group)`                                                                               |
| `None` (empty panel), or unit past the 100-entry `MEMBER_ROSTER_LIMIT` tail | none                                                      | omit the clause when there is no destination; keep it when the destination merely was not rendered |

The `unit › member` form reuses the existing `_tribe_member_label` idiom in `_agent_display_tribe.py:511`. The group
label comes from `banner_label` (`src/sase/ace/tui/models/agent_groups/_tree.py:480`); factor out
`banner_label_for_group_key(group_key)` there and have `banner_label(group)` delegate to it, since the resolver has only
the key.

### Rejected alternatives (do not re-litigate these while implementing)

- **Recolor the target's number chip instead of adding a gutter.** Zero extra columns and it looks great, but it is a
  colour-only signal: it dies in a monochrome terminal and is weak for colour-vision deficiency. The gutter is
  shape-based and survives everything.
- **A second, dim cursor on the exact nested child row when it is rendered.** More precise, but two cursors in one
  roster invite "which one is it?". The heading clause already names the exact row, and the roster's own unit of meaning
  is the numbered member — which is what the user asked to indicate.
- **Putting the number (`l ❯ 03 np`) in the heading clause.** Couples the clause to `MemberJumpNumbering.width` and
  breaks for hidden-tail targets. The label alone is unambiguous.
- **Showing the cursor on collapsed panels** (where `l` expands instead of entering). An indicator that points at a row
  the key will not select is worse than no indicator.
- **Re-deriving the destination from the snapshot** instead of sharing `_exit_expanded_panel_focus`'s rule. Guaranteed
  drift.

### Perf constraints (see `sase/memory/tui_perf.md`)

`_focused_tribe_summary` runs on both the immediate/cheap path (`_apply_tribe_summary(..., cheap=True)`, which renders
the header only and returns before the roster) and the 150 ms-debounced full path. `resolve_panel_entry_stop` calls
`_panel_navigation_stops(include_panel_focus=True)`, which rebuilds the panel tree.

- Gate it: give `_focused_tribe_summary` a `with_entry_target: bool = True` keyword and pass `with_entry_target=False`
  from the cheap call site in `_display_detail_render.py:108-113`. The cheap render never draws the roster, so it must
  not pay for the target.
- No new work may land on the immediate highlight path (rule 7). The debounced path pays exactly one extra tree build
  per tribe render.
- Note for the implementer: `_nav_stops_cache` holds a single entry keyed partly on `include_panel_focus`, so the
  `include_panel_focus=True` call evicts the plain one. That is harmless here — while whole-panel focus is active the
  plain variant short-circuits to `[]` without building a tree.

### Rust core boundary

Nothing here crosses it. The destination depends on Textual whole-panel focus, per-panel selection memory, in-panel fold
state, and grouping mode — all TUI presentation state with no meaning to another frontend. All work stays in this repo.

## Implementation

1. **`src/sase/ace/tui/models/agent_groups/_tree.py`** — add
   `banner_label_for_group_key(group_key: tuple[str, ...]) -> str`, make `banner_label(group)` delegate, export the new
   name.

2. **`src/sase/ace/tui/models/agent_tribe_summary.py`**
   - Add:

     ```python
     @dataclass(frozen=True, slots=True)
     class TribeEntryTarget:
         """The roster destination the panel-entry key selects."""

         unit_identity: AgentIdentity | None
         label: str
         kind: Literal["unit", "member", "group"]
         key_label: str = "l"
     ```

   - Add pure resolvers `tribe_entry_target_for_row(agents, row, *, key_label)` (uses `tree_parent_lookup` +
     `presentation_anchor_lookup` over the same panel slice the snapshot is built from, then `_row_name` /
     `_relative_child_label` for the label) and `tribe_entry_target_for_group(label, *, key_label)`.
   - Add `entry_target: TribeEntryTarget | None = None` as the **last** field of `AgentTribeSummarySnapshot` (all
     existing fields are default-less, so a trailing default is legal and every existing constructor keeps working), and
     an `entry_target` keyword on `build_agent_tribe_summary_snapshot`.
   - Export the new names from `__all__`.

3. **`src/sase/ace/tui/actions/agents/_panel_entry_target.py`** (new) — `resolve_panel_entry_stop(owner, panel_key)` as
   described. Keep it dependency-light so it can be unit-tested against a stub owner.

4. **`src/sase/ace/tui/actions/agents/_selection.py`**
   - `_exit_expanded_panel_focus` calls `resolve_panel_entry_stop`; delete the inlined rule. No behavior change.
   - `_focused_tribe_summary(*, with_entry_target: bool = True)`: when enabled and the panel is expanded, resolve the
     stop, map it to a `TribeEntryTarget` (agent stop → `self._agents[idx]` → `tribe_entry_target_for_row` over the
     panel slice; banner stop → `banner_label_for_group_key` → `tribe_entry_target_for_group`), resolve `key_label` from
     `self._keymap_registry.app.expand_or_layout` via `footer_key_display`, and pass it into the builder. Resolve
     nothing when `focus.collapsed`.

5. **`src/sase/ace/tui/actions/agents/_display_detail_render.py`** — thread `with_entry_target=not cheap` through
   `_apply_tribe_summary` into the `_focused_tribe_summary` call.

6. **`src/sase/ace/tui/widgets/prompt_panel/_member_roster.py`**
   - `MemberRosterEntry` gains `is_entry_target: bool = False`.
   - `append_member_roster` gains `entry_cursor: bool = False`. When true, every numbered row is prefixed with the
     two-cell gutter and every child branch prefix is padded by two cells; when false the output is **byte-identical to
     today**, so clan rosters and their goldens are untouched.
   - Export a module constant for the glyph/style so tests and the tribe module share one definition.

7. **`src/sase/ace/tui/widgets/prompt_panel/_agent_display_tribe.py`**
   - `_roster_entries` sets `is_entry_target` on the unit whose `identity == snapshot.entry_target.unit_identity`.
   - Pass `entry_cursor=not snapshot.panel_collapsed` to `append_member_roster` — driven by panel state, not by whether
     a target exists, so the gutter does not jitter when the destination is a group banner.
   - Build the heading clause and pass it into `append_member_roster` (add an optional `heading_suffix: Text | None`
     parameter to `append_member_roster`/`_append_roster_heading`; `None` keeps clan headings unchanged).

8. **`src/sase/ace/tui/modals/help_modal/agents_bindings.py:138`** — per `src/sase/ace/CLAUDE.md`, keep the `?` popup in
   sync: change the `expand_or_layout` description to `"Expand fold / enter panel (❯)"` (29 chars, within the 32-char
   cap).

## Tests

Behavioral first — the point of this feature is that the mark is _never wrong_.

1. **`tests/ace/tui/test_agent_panel_entry_indicator.py`** (new, app-level, the load-bearing suite). For each scenario:
   assert on the rendered tribe document which row carries `❯`, then actually press `l`, then assert the row the app
   landed on is the marked one.
   - Fresh panel, no selection memory → first rendered row; prove it is _not_ assumed to be roster entry `00` by using a
     grouping/fold arrangement where render order differs from panel order.
   - After descending, moving to a later row, and returning to whole-panel focus → the remembered row.
   - Remembered row removed (killed/dismissed/filtered out) between the render and the keypress → falls back to the
     first stop, and a re-render moves the cursor.
   - Grouping toggled (`o`) so render order changes → cursor follows.
   - Destination is a nested family member → cursor sits on the owning family unit and the heading clause reads
     `unit › member`.
   - Destination is a collapsed group banner → no row cursor, heading clause names the group, and `l` does land on the
     banner.
   - `Esc` agrees with `l` (both use the shared resolver).
   - Collapsed panel → no cursor, no clause; after one `l` the panel is expanded, still whole-panel focused, and the
     cursor appears.
   - Rebound key (`expand_or_layout` set to something other than `l` in the keymap) → the clause shows the bound key.

2. **`tests/ace/tui/widgets/test_agent_display_tribe.py`** — document-level: gutter present on every row and only one
   `❯`; two-space gutter keeps non-target rows aligned; child branch prefixes stay aligned under their unit at
   `FULLY_EXPANDED`; heading clause text for each of the four destination cases; nothing rendered when
   `panel_collapsed=True`; target whose unit is past `MEMBER_ROSTER_LIMIT` renders the clause but no cursor.

3. **`tests/ace/tui/widgets/test_agent_display_clan_roster.py`** — a regression assertion that clan rosters render with
   **no** gutter and no heading clause, pinning the `entry_cursor=False` default.

4. **Pure model tests** for `tribe_entry_target_for_row` (top-level unit, family member, clan member → clan container,
   workflow child, row with no anchor) and for `resolve_panel_entry_stop` (remembered-and-present, remembered-and-gone,
   no memory, no stops).

5. **Visual (`just test-visual`)** — the expanded-tribe goldens shift by two columns and gain the cursor; re-accept with
   `--sase-update-visual-snapshots` **only after** eyeballing the diffs in `.pytest_cache/sase-visual/`. Expect churn in
   at least `agents_sole_selected_panel_120x40`, `agents_tribe_panel_selected_expanded_120x40`,
   `agents_tribe_panel_level_{1,2,3,4}_120x40`, `agents_tribe_panel_display_config_120x40`,
   `agents_tribe_panel_isolation_armed_120x40`, and `agents_selected_panel_clan_collapse_120x40`. Collapsed-panel
   goldens (`agents_collapsed_panel_*`) must **not** change — if they do, the collapsed-panel suppression is broken. Add
   one new golden showing a _resumed_ (non-first) destination, since that is the case the feature exists for.

## Acceptance criteria

- With whole-panel focus on an expanded tribe panel, exactly one `TRIBE MEMBERS` row is marked with `❯`, and pressing
  the panel-entry key selects a row inside that member — in every scenario in test suite 1.
- The heading clause names the destination and the currently bound key.
- Collapsed panels render neither cursor nor clause.
- Clan rosters render exactly as before (no gutter, no clause).
- `_exit_expanded_panel_focus` has no destination logic of its own; `resolve_panel_entry_stop` is the only place the
  rule exists.
- Nothing is added to the immediate/cheap detail path; `pytest -s -m slow tests/ace/tui/bench_tui_jk.py` shows no
  regression in Agents-tab p95.
- `just install && just check` passes, and `just test-visual` passes with reviewed goldens.

## Out of scope

- Changing what `l` / `Esc` actually do. This plan only makes the existing destination visible.
- Any indicator in the left agent list, the footer, or clan/family/workflow documents.
- Persisting `_panel_selection_memory` across restarts (it is session-local today; that stays true).
- Editing any file under `sase/memory/`, `AGENTS.md`, or the generated provider instruction shims.
