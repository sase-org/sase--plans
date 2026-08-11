---
tier: tale
title: Preview which folds the Agents-tab `-` keymap would re-expand
goal:
  When the next `-` press would restore folds, every lane and clan it would re-expand is
  marked with a gold `▿` on its row and a `▿N` chip in the panel border title, and the
  markers clear the instant the next press would sweep instead.
size: medium
proposed_by: bbugyi200.athena.xz
create_time: 2026-08-11 08:39:28
status: done
---

- **PROMPT:**
  [prompts/202608/fold_restore_preview.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/fold_restore_preview.md)
- **AGENTS:**
  - [bbugyi200.athena.xz](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.xz.md)
- **COMMITS:**
  - [3e19e7c](https://github.com/sase-org/sase/commit/3e19e7cd15c71a2eacaa95fb5bb265a0c89c7b1f)
    — feat(ace): show fold restore preview markers

# Fold Restore Preview for the Agents-tab `-` Keymap

## Goal

Make it visually obvious — before the user presses anything — exactly which agent lanes
and clans the next `-` press would re-expand in the Agents tab, using the same "armed
restore" visual language the `=` keymap already established for panel isolation.

## Background: what `-` does today

`-` is bound to `collapse_panel_folds` (`src/sase/default_config.yml:462`,
`src/sase/ace/tui/bindings.py:43`). Its implementation is
`AgentPanelFoldSweepMixin.action_collapse_panel_folds` in
`src/sase/ace/tui/actions/agents/_folding_panel_sweep.py`:

1. Resolve the target panel (whole-panel focus, else `_panel_group.focused_key`).
2. Resolve every open canonical **lane** fold (`resolve_panel_lane_collapse_target`,
   `_folding_lanes.py`) and every open canonical **clan** fold
   (`resolve_panel_clan_collapse_target`, `_folding_clans.py`) in that panel.
3. If any are open → collapse them all and store a `PanelFoldSweepRecord`
   (`src/sase/ace/tui/models/agent_panels.py:78`) holding each fold key with the
   `FoldLevel` it had before, keyed by panel.
4. If none are open → **reverse**: `_restore_panel_fold_sweep` re-expands exactly the
   folds that panel's last sweep closed, filtered at press time by
   `_live_sweep_record_entries` to fold owners that are still live in the panel and
   still `COLLAPSED`, then drops the record.

Grouping banners (`Done`, `Running`, project/Patch/name-root banners) are never touched
by `-`; only structural lane and clan folds are.

Two predicates already expose this state and are used by the footer
(`_display_detail_footer.py:130-147` → `_keybinding_bindings.py:221-224`, which renders
`- collapse folds` vs `- restore folds`):

- `_panel_has_collapsible_folds(panel_key)` — would `-` sweep?
- `_panel_fold_sweep_restore_available(panel_key)` — would `-` restore?

**The gap:** the footer says the _direction_ of the next press but nothing says _which
folds_ come back. The user has to remember what they swept.

## Background: how `=` already solves the analogous problem

`=` (`isolate_panels`, `_folding_panels.py:151`) arms a session-local
`PanelIsolationRevert`. `_panel_isolation_marked_keys()` returns the set of live panels
whose fold state the remembered layout would change. Both panel-refresh paths
(`_display_panel_widgets.py:100-101` and `:290-291`) and the title-only repaint
(`_display_panel_collection.py:_refresh_agent_panel_titles`) call it and thread
`isolation_restore_marked=key in isolation_marked_keys` into `agent_panel_border_title`
(`_display_panel_titles.py:85-138`), which appends a `↺` glyph styled
`_PANEL_ISOLATION_RESTORE_STYLE = "bold #D7AF5F"` to the panel border title.

This plan mirrors that architecture one level down: from panels to the rows inside them.

## Design

### D1. The marked set — one resolver, mirroring `_panel_isolation_marked_keys`

Add to `AgentPanelFoldSweepMixin` (`_folding_panel_sweep.py`):

```python
def _panel_fold_restore_marked_keys(self) -> dict[PanelKey, frozenset[str]]:
    """Return the fold keys `-` would re-expand, per panel."""
```

A panel contributes an entry **iff all of the following hold**, which is exactly the
condition under which pressing `-` on that panel restores rather than sweeps:

1. It has a `PanelFoldSweepRecord` in `_panel_fold_sweep_records`.
2. It is still a live panel key in `_panel_group.panel_keys`.
3. It is **not** collapsed (`panel_is_collapsed` from `._panel_fold_intent`) — `-`
   refuses to act on a collapsed panel ("Panel is collapsed"), and a collapsed panel
   renders no rows, so marking it would be a lie.
4. `_panel_has_collapsible_folds(panel_key)` is `False` — if anything is still open, the
   next `-` press _sweeps_, and the old record will be replaced rather than restored.
5. `_live_sweep_record_entries(...)` is non-empty after filtering to still-live,
   still-collapsed owners.

Return `{panel_key: frozenset(fold_key for fold_key, _level in live_entries)}`.

**Performance gate (mandatory).** Rule 6 of `sase/memory/tui_perf.md` — read it before
touching the refresh path — says full agent-list rebuilds are the most expensive UI
operation, so this must add ~zero cost in the common case. Condition 1 is a dict lookup
and `_panel_fold_sweep_records` is empty except right after a sweep, so the method
**must return `{}` immediately when the records dict is empty/absent**, before touching
`_panel_group`, `rendered_panel_slice`, or either collapse-target resolver. Only panels
that actually hold a record may pay for conditions 3-5. Do not iterate all panels and
probe each one. No new refresh path is introduced: every arm/disarm already runs through
`_refilter_agents` / `_refresh_agents_display(list_changed=True)`.

### D2. Row marker glyph and style

Marked rows get a trailing **`▿`** (U+25BF WHITE DOWN-POINTING SMALL TRIANGLE) rendered
in **`bold #D7AF5F`**.

**Why not `↺`** (the glyph `=` uses): `↺` is already taken twice in the agent-row
grammar and would be ambiguous or unreadable there.

- `_REVERTED_GLYPH = "↺"` styled `bold #D7875F` (`_agent_list_styling.py:121`) is the
  _reverted_ badge, rendered immediately before the display name of a reverted agent.
- `↻N` is the retry/attempt badge — both as a leading row badge and, critically, as
  `_attempt_count_suffix` _inside the fold annotation itself_
  (`_agent_list_helpers.py`), producing `×3 ↻2`. A gold `↺` two cells from a dim `↻2` is
  not a distinction a user should have to make.

`▿` is unused in the row grammar (audited against `_agent_list_styling.py`:
`⚡ ◌ ◆ ✏️ ? ! ↺ ❑ ≡ │ └─ ↳ 🐍 🐚 ▌ ▎ ▸ ━ ─` plus `▲`/`▼` for embedded workflows and
`❖`/`▸` in panel titles). It reads as a disclosure triangle — "this opens downward" —
which is literally what the action does. It is hollow where the embedded-workflow `▼` is
solid, they live in different row zones, and an embedded-workflow step row is never a
canonical sweep target, so the two never compete.

**Why `#D7AF5F`**: it is exactly `_PANEL_ISOLATION_RESTORE_STYLE`. The invariant to
establish and document is:

> **Gold `#D7AF5F` marks state that the key you just pressed will put back.** `=` marks
> whole panels with `↺` in the border title; `-` marks individual folds with `▿` on the
> row and in the border title. Location and glyph together say which key owns the mark;
> the shared color says "press it again to undo".

To keep them from drifting, promote the color to one shared constant — e.g.
`ARMED_RESTORE_STYLE = "bold #D7AF5F"` in `_display_panel_titles.py` (or a small shared
module) — and have `_PANEL_ISOLATION_RESTORE_STYLE` and the new row/title constants
reference it. Rendering the two affordances in different golds is a bug.

### D3. Row marker placement

In `format_agent_option` (`_agent_list_render_agent.py`), append the marker in the
fold-annotation zone: immediately **after** the `if fold_annotation:` block and
**before** the family count chip. Rendered as a leading space plus the glyph, so a
marked row reads:

```
  ≡ workflow (RUNNING) ×3 ▿ [R2 D1] my-lane
```

The marker **must not** be folded into the annotation string and **must not** depend on
the annotation being non-empty: `compute_fold_annotation` deliberately returns `""` for
an anonymous single-child lane
(`agent.is_anonymous and agent.appears_as_agent and total == 1 and attempts_count == 0`),
and such a lane _can_ be a canonical sweep target. Those rows must still show `▿`.

Leave the `×N` annotation's own dim-cyan styling untouched. Rejected alternative:
recoloring `×N` gold while armed. It adds a second, redundant color channel, makes rows
flicker hue on every arm/disarm, and steals a stable structural signal (hidden-child
count) for a transient one.

### D4. Panel title chip

`agent_panel_border_title` gains a `fold_restore_marked_count: int = 0` parameter. When

> 0 it appends `▿N` in the shared gold — e.g. `❖ @epic ▿3 · 5 [R2 D1]` — placed in the
> same slot the isolation `↺` uses (after the label, before the `·` count separator).
> When both are armed the title shows `↺` then `▿3`; they are different glyphs for
> different keys, so this composes without ambiguity.

This exists so the affordance survives scrolling (marked rows can be off-screen in a
long panel) and so the _scope_ of `-` is legible when more than one panel holds a record
— `-` only ever acts on the focused panel, and the per-panel chip makes the per-panel
bookkeeping visible. Suppressed on collapsed panels by construction (D1 condition 3).

Do **not** change the footer labels (`- collapse folds` / `- restore folds`). They are
already documented and correct; the count now lives in the title chip, in one place.

## Implementation

Work outward from state to pixels. All paths are relative to the repo root.

### 1. State resolver

`src/sase/ace/tui/actions/agents/_folding_panel_sweep.py`

- Add `_panel_fold_restore_marked_keys()` per D1 to `AgentPanelFoldSweepMixin`, with the
  cheap empty-records early return first.
- Reuse the existing `_panel_fold_sweep_records`, `_live_sweep_record_entries`, and
  `_panel_has_collapsible_folds` helpers; do not duplicate their logic.

### 2. Shared restore accent

`src/sase/ace/tui/actions/agents/_display_panel_titles.py`

- Introduce the shared `ARMED_RESTORE_STYLE` constant and redefine
  `_PANEL_ISOLATION_RESTORE_STYLE` in terms of it (keep the existing name — it is
  asserted by `tests/ace/tui/test_agent_panel_titles.py`).
- Add the `▿` glyph constant and the `fold_restore_marked_count` parameter to
  `agent_panel_border_title` per D4.

`src/sase/ace/tui/widgets/_agent_list_styling.py`

- Add `_FOLD_RESTORE_GLYPH = "▿"` and `_FOLD_RESTORE_GLYPH_STYLE`, sourced from the same
  gold. Add a comment tying it to the `-` sweep and to the panel-title marker, matching
  the file's existing commenting style for `_REVERTED_GLYPH` etc.

### 3. Row rendering

`src/sase/ace/tui/widgets/_agent_list_render_agent.py`

- Add `fold_restore_marked: bool = False` to `format_agent_option` and to
  `cached_format_agent_option` (pass it through to both the key builder and the
  formatter).
- Render per D3.

`src/sase/ace/tui/widgets/_agent_list_render_cache.py`

- Add `fold_restore_marked` to `agent_render_key`'s signature **and to the returned key
  tuple**. This is not optional: the module docstring states a cache hit must only
  happen when the output would be byte-identical, and omitting it would leave stale
  markers on screen after an arm or disarm.

### 4. Widget plumbing

`src/sase/ace/tui/widgets/_agent_list_build.py`

- `build_list` gains `fold_restore_marked_keys: set[str] | None = None`. In the
  per-agent loop it already computes `fold_key = agent_fold_key(agent)`; derive
  `restore_marked = bool(fold_key and fold_key in (fold_restore_marked_keys or ()))` and
  pass it to `cached_format_agent_option`.
- Store it in `widget._row_render_ctx[i]` under `"fold_restore_marked"` so the
  single-row fast path keeps it.
- `patch_row` reads `ctx.get("fold_restore_marked", False)` and passes it through.
  (`patch_row` handles status/runtime mutations, never fold-state changes, so carrying
  the previous value forward is correct.)

Key the marker by **fold key**, not by row index. Fold keys are stable across the
global→local index remapping that `agent_list.update_list` and
`_refresh_panel_widgets_impl` perform for `jump_hints`; using indices would require
mirroring that remap in three places.

`src/sase/ace/tui/widgets/agent_list.py`

- `update_list` gains the same `fold_restore_marked_keys` parameter, documented in its
  docstring, and forwards it to `build_list`. No index remapping needed (see above).

### 5. App-side wiring

`src/sase/ace/tui/actions/agents/_display_panel_widgets.py`

- In **both** `_refresh_panel_widgets_impl` and `_refresh_affected_panel_widgets`,
  resolve the marked map once per refresh next to the existing
  `marked_keys_fn = getattr(self, "_panel_isolation_marked_keys", None)` lookup, using
  the same defensive `getattr` + `callable` pattern (the panel mixins are consumed by
  lightweight test apps that do not implement every method).
- Pass `fold_restore_marked_keys=restore_marked.get(key)` into `update_list` and
  `fold_restore_marked_count=len(restore_marked.get(key, ()))` into
  `self._agent_panel_title(...)`.

`src/sase/ace/tui/actions/agents/_display_panel_collection.py`

- Add the `fold_restore_marked_count` parameter to `_agent_panel_title` and forward it
  to `agent_panel_border_title`.
- Update `_refresh_agent_panel_titles` to resolve the map and pass the count, exactly as
  it already does for `isolation_marked_keys`.

### 6. Help modal

`src/sase/ace/tui/modals/help_modal/agents_bindings.py:174-177`

- The `-` entry currently reads `"Collapse panel folds ⇄ restore"`. Update it to name
  the marker, e.g. `"Collapse panel folds ⇄ restore ▿"`, respecting the 32-character
  description cap documented in `src/sase/ace/CLAUDE.md` ("Help Popup Maintenance" and
  "Help Modal Box Formatting"). Truncate wording rather than exceeding the cap.

### 7. Docs

- `docs/ace.md` (the `-` paragraph around line 1444): after the sentence describing the
  reverse, document that armed panels mark each fold `-` would re-expand with a gold `▿`
  on the owner row plus `▿N` in the panel title, and that the markers clear the moment
  the next press would sweep instead of restore.
- `docs/agent_families.md` (the `-` paragraph around line 518): same, in that file's
  more compact register, alongside the existing `↺`-marker sentence for `=`.

## Tests

Follow the existing structure — `tests/ace/tui/test_agent_panel_fold_sweep.py` uses the
`AgentPanelCollapseApp` harness from `._agent_panel_collapse_helpers` with
`_named_workflow_lane` / `_clan_container` fixtures; reuse them.

**Resolver** (`tests/ace/tui/test_agent_panel_fold_sweep.py`):

1. No records → `_panel_fold_restore_marked_keys() == {}` (and, to pin the perf gate,
   that it short-circuits without consulting collapse-target resolution — e.g. via
   monkeypatched sentinels or by asserting the empty result on an app whose panel group
   is absent).
2. After a sweep → the swept panel maps to exactly the swept lane + clan fold keys.
3. After the reversing press → back to `{}`.
4. Manually expanding one swept fold → `{}` for that panel (the next `-` sweeps, so
   nothing may stay marked), even though other record entries are still collapsed.
5. A record whose owners all disappeared → `{}`, matching
   `_panel_fold_sweep_restore_available`.
6. A collapsed panel with a live record → `{}`.
7. Two panels, one swept → only the swept panel is marked; the other stays empty.

**Row rendering** (extend the nearest existing agent-row rendering test module):

8. `format_agent_option(..., fold_restore_marked=True)` appends `▿` with the shared gold
   style; `False` appends nothing.
9. A row whose `compute_fold_annotation` is `""` (anonymous single-child lane) still
   renders `▿` (D3).

**Cache** (`tests/` module covering `agent_render_key`):

10. Two keys differing only in `fold_restore_marked` are unequal.

**Title** (`tests/ace/tui/test_agent_panel_titles.py`):

11. `fold_restore_marked_count=3` renders `▿3` in the shared gold, using the existing
    `_assert_title_span` helper.
12. Isolation `↺` and fold `▿3` compose in one title without clobbering each other.

**Visual** (`tests/ace/tui/visual/test_ace_png_snapshots_agents_tribe_panel.py`):

13. New PNG snapshot `agents_panel_fold_sweep_armed_120x40`, modeled directly on the
    existing `agents_tribe_panel_isolation_armed_120x40` case (~line 230): drive to the
    Agents tab, focus a panel with an expanded lane and clan, press `-`, wait for the
    sweep record to arm, assert `assert_page_svg_contains(page, "▿")` and the
    `("-", "restore folds")` footer binding, then `ace_png_visual.assert_page_png(...)`.
    Generate the golden with `just test-visual --sase-update-visual-snapshots` and
    **eyeball the resulting PNG** before accepting it — this feature is judged on how it
    looks, so confirm the glyph renders in Fira Code at the pinned size, sits where D3
    says, and reads as gold against the surrounding dim cyan.

If `▿` turns out to render poorly in the pinned Fira Code fixture, fall back to `⌄`
(U+2304) and, only if that also fails, `▾` (U+25BE); keep the shared gold and the
placement in either case, and note the substitution in the docs updates.

## Verification

```bash
just install
just check
```

`just check-full` if the scoped selection escalates or reports anything unusual.
`just test-visual` for the PNG suite (it is excluded from `just test`), inspecting
`.pytest_cache/sase-visual/` artifacts on any failure.

## Out of scope

- Changing what `-` collapses or restores. This plan is presentation-only; the sweep,
  the record, the filtering, and the notifications keep their current semantics.
- Marking grouping banners. `-` never collapses them.
- Changing the `=` isolation marker beyond factoring its color into a shared constant.
- Changing the footer labels.
