---
tier: tale
title: Add adaptive entry jump to Launch Control
goal:
  Launch Control provides fast, polished apostrophe-hint navigation across settings,
  model aliases, and buckets while remaining safe across asynchronous row refreshes.
size: medium
proposed_by: bbugyi200.athena.02w.f0
create_time: 2026-08-15 20:01:57
status: wip
---

# Add adaptive entry jump to Launch Control

## Goal

Add the standard ACE apostrophe entry-jump interaction to **Launch Control**. Pressing
`'` should paint adaptive hints on every currently selectable setting, model alias, or
bucket; entering a hint should move the highlight there without activating the row. The
interaction must use the shared modal jump state machine, remain responsive and visually
aligned, and never leave stale hints or history pointing at a different row after an
asynchronous refresh or bucket transition.

This is a navigation enhancement only. Preserve the existing `ModelsPanel` class and
module names, `models_panel` action ID, `,m` launch chord, row IDs, edit/override/reset
semantics, provider routing, bucket drill-in behavior, and modal dimensions.

## Interaction contract

1. Add the same modal-local `apostrophe -> jump_to_entry` binding used by the model
   picker, notification modal, and saved-group revival modal. The normal Launch Control
   footer should advertise `' = Jump` alongside `j/k = Navigate` for every row context.
2. At the top level, allocate hints in exact visible selectable-row order across:
   - all six launch settings;
   - built-in alias or collapsed-bucket rows; and
   - custom alias or collapsed-bucket rows.

   Disabled launch/ownership headers, blank section spacers, and the empty-custom hint
   are presentation only and must neither receive hints nor consume hint characters.
   Inside a bucket, scope hints to that bucket's selectable alias members, again
   skipping mixed-ownership headers and the blank spacer.

3. Use the shared zero-based adaptive alphabet unchanged: `0`-`9`, `a`-`z`, `A`-`Z` for
   up to 62 targets and fixed-width two-character hints beginning at `00` for larger
   lists. Preserve case-sensitive matching and pending-prefix behavior.
4. A completed hint moves only the highlight. It must update the normal description
   strip and context-sensitive footer but must not open a bucket, launch an edit card,
   or apply any setting action. The user can press `Enter`, `l`, `e`, `o`, or another
   existing action afterward.
5. While hints are painted:
   - `Esc` or an invalid hint leaves jump mode without closing Launch Control or moving
     the cursor;
   - a second `'` pops the most recent valid pre-jump origin from the shared bounded
     back stack;
   - with no origin, the second `'` moves to the first selectable row; and
   - the footer becomes the standard concise `JUMP ' back|first  <esc> cancel` status.

   Leaving jump mode restores the row-specific normal footer. Keep the shared ten-origin
   limit and do not add a Launch-Control-specific history implementation.

## Shared jump integration and refresh safety

1. Integrate `ModelsPanel` with `KeyedPaneEntryJumpMixin[str]`; use existing stable
   Launch Control row IDs as jump keys. Supply the small host hooks for current target
   keys, highlighted key, keyed selection, and repainting rather than copying hint-map,
   prefix, matching, or back-stack logic into the panel.
2. Derive jump targets from the panel's current logical data rows in visual order, not
   from raw `OptionList` indices. This keeps disabled headers/spacers out of the key
   space and lets the existing row-ID lookup, bucket scope, and highlight restoration
   remain the source of truth.
3. Track the ordered tuple of jumpable row IDs associated with each rendered list.
   Before every list rebuild, compare the next tuple with the rendered tuple and call
   the shared invalidation API:
   - if IDs or order changed, clear both painted hints and index-based back history;
   - if IDs and order are unchanged, retain valid hints/history while repainting new
     values or countdowns; and
   - initialize the tuple on first composition without manufacturing history.

   This rule must cover the initial off-thread provider snapshot, alias/config edits,
   provider-routing refreshes, threshold/effort/limit reloads, and bucket entry/exit. In
   particular, the initial snapshot may add aliases after the six cached settings, and
   opening or leaving a bucket replaces the target identity set; neither transition may
   reinterpret an old numeric origin as a new row.

4. Repaint and keyed selection through the panel's existing `_replace_display`, stable
   `keep` ID, `_set_highlighted_index` guard, and `_update_context` paths. Do not assign
   `OptionList.highlighted` through an unguarded side path: Textual highlight echoes
   must not cause cursor jumps or stale description/footer state.
5. Keep every apostrophe/hint keystroke cache-only and synchronous. Hint allocation,
   matching, row decoration, and selection may inspect the already-loaded in-memory
   rows, but must not read config, stat/glob files, resolve providers, or start a
   worker. Existing off-thread snapshot and write workflows remain unchanged.

## Rendering and layout

1. Reuse `apply_jump_hint_prefix` so each selectable row begins with the established
   yellow `[<hint>] ` marker and retains the row's Rich styles, literal-safe content,
   `no_wrap`, and ellipsis behavior. Apply the prefix after building the normal row so
   setting, alias, warning, ownership, bucket, override, and provider styles do not
   diverge between normal and jump modes.
2. Reserve one transient fixed-width jump gutter for the whole session: four cells for
   one-character hints and five for two-character hints. Indent disabled headers and the
   empty-custom furniture by the same gutter while jump mode is active, without giving
   them visible hints, so the ownership/name/value/state grid stays aligned.
3. Subtract that gutter from the existing value/provider-model width cap only while jump
   mode is active. Preserve the 110-column modal budget and aligned state tags; do not
   widen the container or allow hint prefixes to push provenance/override chips
   off-screen. Normal mode must retain its current widths exactly. Verify both the
   standard `120x40` viewport and the existing constrained/narrow viewport behavior.
4. Keep the highlighted row and scroll visibility stable when entering or cancelling
   jump mode. Rebuilding only to add or remove hint prefixes must not change the active
   bucket, trigger a row action, or cause layout-height shifts.

## Implementation areas

- Add a focused Launch Control jump mixin/module under `src/sase/ace/tui/modals/` if
  that keeps the already-large display facade cohesive. It should adapt
  `KeyedPaneEntryJumpMixin[str]`, route normalized key events, and call the existing
  display hooks; the shared `pane_entry_jump.py` state machine should not need
  feature-specific behavior.
- Wire the mixin and `apostrophe` binding in `models_panel.py`, including only the small
  per-instance rendered-target snapshot needed for refresh invalidation.
- Update `models_panel_display.py` to expose stable ordered target IDs, invalidate
  changed target sets before repaint, decorate selectable prompts, reserve decorative
  hint gutters, route keyed selection through guarded highlight restoration, and switch
  the footer between normal and jump variants.
- Extend `models_panel_rendering.py` only with pure width/gutter or decoration helpers
  needed to keep the grid aligned. Avoid coupling pure renderers to Textual widgets or
  adding config/provider work to render functions.
- Update the Launch Control section in `docs/ace.md`, its key table, and the shared
  apostrophe-jump overview, which currently enumerates only three supporting modals.
  Document that hints include settings, aliases, and buckets; skip section furniture;
  select without activating; and use `''` for back/first behavior.

## Tests and validation

1. Add focused mounted Launch Control jump tests that prove target IDs and hints follow
   visual order at the top level and inside built-in, custom, homogeneous, and mixed
   buckets while headers, spacers, and the empty-custom hint are excluded.
2. Exercise real key events, including uppercase hint normalization and a list large
   enough to require a two-character pending prefix. Assert completion moves the
   highlight by stable ID, refreshes description/footer context, and never invokes the
   selected row's action.
3. Cover `Esc`, invalid input, `''` with and without history, bounded back-stack reuse,
   normal/jump footer text, retained highlight/scroll position, and the guarded
   programmatic-highlight path.
4. Add refresh tests for both sides of the identity rule: value-only rebuilds with the
   same ordered IDs retain valid jump state/history, while the initial async alias load,
   alias addition/removal/reorder, and bucket entry/exit clear stale hints and origins.
   Include a refresh during active two-character prefix input so no prefix survives an
   identity change.
5. Add or update Launch Control PNG snapshots for a top-level hint session and a
   drilled-in mixed bucket. Include at least one constrained-width assertion and inspect
   actual/expected/diff artifacts to verify yellow hints, uniform transient gutters,
   aligned headers/data/state tags, footer containment, and absence of clipping. Avoid
   regenerating unrelated goldens.
6. From an installed workspace (`just install` first), run the focused panel navigation,
   rendering, keymap, shared jump, and visual tests. Regenerate only intentional Launch
   Control goldens with the visual update flag, rerun those snapshots without it, then
   run the required `just check`. If the diff broadens a visual/data selection gate,
   follow the repository's governed scoped/full verification behavior rather than
   bypassing it.

## Acceptance criteria

- Pressing `'` in Launch Control paints adaptive hints on every visible setting, alias,
  or bucket and on no disabled/decorative row; hints select but never activate entries.
- Case-sensitive one- and two-character hints, cancellation, `''` back/first behavior,
  bounded history, and jump footer text match the other shared modal jump surfaces.
- Top-level and drilled-in bucket target scopes are correct, and any row-ID/order change
  clears stale hints/history before repaint while value-only refreshes remain stable.
- Hint mode preserves the Launch Control grid, semantic styles, state-column visibility,
  highlight, scroll position, modal dimensions, and responsive behavior at normal and
  constrained widths.
- No keystroke path performs I/O or provider/config resolution, no highlight echo causes
  a cursor jump, and all existing launch-setting, alias, bucket, override, edit, reset,
  provider-routing, and close actions behave exactly as before.
- Documentation, focused tests, reviewed PNG snapshots, and the required repository
  verification accurately describe and protect the completed interaction.
