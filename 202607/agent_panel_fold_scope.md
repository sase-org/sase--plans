---
create_time: 2026-07-14 09:41:54
status: wip
prompt: 202607/prompts/agent_panel_fold_scope.md
tier: tale
---
# Fix Agents-Tab Group Folds Leaking Across Panels

## Goal

Make group folding on the `sase ace` Agents tab respect the rendered agent panel that owns the group. Collapsing or
expanding a focused `Done` banner in `(untagged)`, for example, must not change the independently rendered `Done`
banners in `#chop`, `#research`, or any other tag panel.

The change should preserve the existing grouping-mode behavior, bulk fold semantics, focus restoration, jump targets,
refresh pipeline, and TUI navigation performance.

## Root Cause

Each tag panel builds its grouping tree independently and emits panel-local banner keys. In status grouping, every panel
can therefore contain the same key, such as `("Done",)`. The application currently owns one active
`AgentGroupFoldRegistry` per grouping mode and passes that same registry to every panel. Because the registry stores
only the local banner tuple, collapsing `("Done",)` in one panel makes every other panel's tree read that key as
collapsed too.

Other panel-sensitive behavior already recognizes this identity boundary: the focused banner is interpreted in the
focused panel, and jump targets include a panel index because equal banner keys may exist in different panels. Fold
state needs the same panel dimension.

## Intended Behavior

- Lowercase `h` / `l` on an Agents-tab group changes only that group in the currently focused rendered panel.
- Equal group keys in sibling panels remain independent in every grouping mode (`STANDARD`, `BY_DATE`, and `BY_STATUS`).
- Uppercase `H` / `L` remains the all-panels operation: it applies one snapshot-based visible layer to every rendered
  panel without recursively stepping rows that become visible during the same keypress.
- Fold state remains isolated by grouping mode as it is today, and is additionally isolated between split tag panels and
  the single merged `All agents` panel. Toggling either setting restores the state previously used in that layout rather
  than leaking a same-named group from another layout.
- Query/filter/refresh reconciliation removes stale folds only from the active panel layout. A group disappearing from
  one panel must not remove or preserve state merely because a sibling panel has the same local key.
- The ChangeSpecs tab continues to use the neutral, single-tree `GroupFoldRegistry` unchanged.

## Design

### 1. Introduce explicit Agents-panel fold scope

Add an Agents-specific fold-state owner above the existing neutral `GroupFoldRegistry`. It should map an explicit panel
scope to one ordinary per-tree registry. The scope must include both the panel key (`None` for untagged or a tag name)
and whether the UI is in merged-panel mode, so the split untagged panel cannot collide with the merged `All agents`
panel.

Keep `GroupRow.group_key` and all grouping-tree keys unchanged and local to a panel. Prefixing or encoding the panel in
the tuple would couple rendering labels, hierarchy traversal, parent-key slicing, marking, killing, and focus logic to
an application layout concern. Instead, central helpers should resolve the correct per-panel registry before a tree is
built or a fold is mutated.

Continue to keep one Agents fold-state owner per `GroupingMode`, preserving the current mode round-trip behavior while
adding panel isolation within each mode.

### 2. Route every Agents tree consumer through the scoped registry

Update the Agents display, focused-panel navigation, neighbor visibility, jump-target generation/history validation,
group lookup, and panel-switch logic to build each panel's tree with that panel's registry view. The widget-level tree
builder should continue to receive a normal single-tree registry, keeping the model and widget APIs simple and leaving
ChangeSpecs unaffected.

Centralize scope/view selection so callers cannot accidentally fall back to the mode-wide owner. The navigation-stop
cache should key on the selected panel registry's version (plus its existing panel/layout inputs), so a fold mutation
invalidates the affected hot path without adding per-keypress tree construction or synchronous work.

### 3. Make fold actions panel-correct

Refactor focused fold helpers to derive the focused panel's agent slice and registry together. Use that context for:

- finding the focused agent's enclosing banners;
- collapsing/expanding a selected banner;
- re-anchoring after expansion;
- snapping focus after collapse; and
- resolving parent-banner escalation on repeated `h` presses.

For `H` / `L`, take one render-order snapshot per currently rendered panel, mutate that panel's registry, and preserve
the existing one-layer rule. Workflow folds remain keyed by workflow identity, but only workflows visible in the
corresponding panel snapshots should be stepped. After a bulk change, re-anchor the cursor against the focused panel
only.

### 4. Reconcile scoped fold state safely during refresh

Replace the deduplicated, cross-panel known-key projection used by the prepared finalize pipeline with a panel-scoped
projection for the active split/merged layout. Carry the panel-layout flag through the prepared snapshot and stale
token, alongside the existing grouping mode, so a worker result prepared before a panel-layout toggle cannot prune the
wrong fold scopes when it reaches the UI thread.

When applying the projection, prune vanished keys independently inside each active scope and remove active-layout scopes
whose panels disappeared. Preserve scopes belonging to inactive grouping modes and the inactive split/merged layout,
matching the existing expectation that cycling away and back restores fold intent.

Keep this enumeration in the existing prepared/off-thread path and retain the synchronous fallback only where the
current finalize pipeline already uses it; do not introduce disk I/O or new full-refresh paths into fold actions.

## Test Plan

Add focused regression coverage using multiple tag panels that each contain a `Done` banner:

1. Collapse `Done` in the focused untagged panel and assert that tagged panels' `Done` groups remain expanded and their
   agents remain visible.
2. Switch focus to a sibling panel with the same banner key and verify navigation, highlight, jump targeting, and `l`
   operate on that panel's independent state.
3. Cover the same-key isolation at the fold-state model level, including separate split-untagged and merged-panel
   scopes.
4. Verify `H` / `L` still apply one visible layer across all rendered panels, including panels whose tree depths differ,
   without stepping newly exposed descendants in the same press.
5. Verify grouping-mode round trips preserve mode-specific, panel-specific fold state, and merged/split toggles do not
   leak state between layouts.
6. Verify refresh garbage collection removes a stale `Done` fold from only the panel where it vanished, even while
   `Done` remains present in another panel; also verify a prepared finalize result is rejected after the panel layout
   changes.
7. Extend display/navigation cache tests to prove each widget/tree receives the appropriate scoped registry and held
   `j`/`k` navigation still reuses cached stops until the focused scope's version changes.

Run the targeted Agents grouping, folding, panel display, navigation/jump, and prepared-apply tests first. Then run the
slow Agents `j`/`k` benchmark to check the hot path remains within its established behavior, followed by `just install`
and the required full `just check`. No visual snapshot changes are expected because the rendered styling is unchanged;
any PNG delta should be treated as a regression unless separately explained.

## Non-Goals

- Do not change group labels, grouping order, tag inheritance, or panel sizing.
- Do not persist fold state across TUI process restarts; this change only corrects ownership of the existing in-memory
  state.
- Do not alter ChangeSpecs grouping/folding behavior or the neutral registry used there.
