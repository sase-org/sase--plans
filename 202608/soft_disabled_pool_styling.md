---
tier: tale
title: Amber soft-disabled members in alias routing summaries
goal:
  Make Launch Control alias selector summaries communicate whether a soft-disabled
  member will be used through an amber member treatment and truthful routing glyphs
  instead of a literal soft chip, without changing selector behavior.
size: small
proposed_by: bbugyi200.athena.09f
create_time: 2026-08-21 09:42:01
status: wip
---

# Plan: Amber soft-disabled members in alias routing summaries

## Context and scope

Launch Control's highlighted-alias description currently renders a soft-disabled
selector member as an ordinary green available target followed by a yellow `soft`
suffix. That splits one routing state across two visual signals and always leaves the
member with a checkmark, even when round-robin routing is deliberately sparing it.

Replace that compact pool/fallback-member treatment only. Provider-management surfaces
(the disabled-provider title line, Provider Routing rows and descriptions, model picker,
completion provenance, notifications, and tooltips) should retain explicit soft-disable
wording because they do not all have a selected-member marker and must remain
understandable without color. No provider-disable or alias-resolution semantics change
in this tale.

## Visual and behavioral contract

Use the already-resolved `ModelAliasSelectorMember.sparing` and `.selected` fields as
the sole source of truth. Do not reimplement the sibling-availability policy in the
renderer: `.selected` already accounts for round-robin pools, all-soft pools, ordered
fallbacks, last-resort tails, weights, CLI availability, and hard disables.

| Member state                            | Compact rendering           | Treatment                                      |
| --------------------------------------- | --------------------------- | ---------------------------------------------- |
| Preferred and selected                  | `→ ✓ provider/model@effort` | Existing green treatment                       |
| Preferred and eligible but not selected | `✓ provider/model@effort`   | Existing dim-green treatment                   |
| Soft-disabled and selected              | `→ ✓ provider/model@effort` | Arrow, checkmark, and complete target in amber |
| Soft-disabled and not selected          | `× provider/model@effort`   | X and complete target in amber                 |
| Hard-disabled or otherwise unavailable  | `× provider/model@effort`   | Existing red treatment                         |
| Invalid                                 | `! raw-value`               | Existing warning treatment                     |

The amber target includes the provider, model, and effort suffix as one semantic unit;
the literal trailing ` soft` chip disappears. Keep separators, `pool:` / `fallback:`
labels, and optional weight chips neutral so the exceptional provider state remains
legible without flooding the line with color. Preserve the current dimming and arrow
suppression when an alias selector is suspended by an override; that line is an inactive
availability preview rather than the active routing decision.

This colored glyph grammar intentionally distinguishes an amber X (selectable but spared
for this resolution) from a red X (unavailable). A selected soft member receives the
amber checkmark whenever the resolver actually chose it, including an all-soft pool or
an ordered fallback whose semantics permit it.

## Implementation

1. In `src/sase/ace/tui/modals/models_panel_rendering_descriptions.py`, make selector
   member rendering choose the soft member's marker from the authoritative selected
   state, apply the existing soft amber palette to the selected arrow plus the entire
   marker/target span, and remove the appended `soft` text. Keep the existing rendering
   branches for preferred, unavailable, invalid, weighted, last-resort, and
   override-suspended members intact.
2. Update the Launch Control documentation in `docs/ace.md` so the alias-description
   grammar describes amber selected/skipped soft members instead of a `soft` chip, while
   retaining the documented explicit labels on provider-management and model-selection
   surfaces.
3. Replace the existing soft-chip assertion in
   `tests/test_models_panel_alias_rendering.py` with focused selected and spared cases.
   Assert exact plain text (including absence of the `soft` suffix), the amber styling
   of the glyph and full target, the selected arrow treatment, and unchanged green/red
   behavior for ordinary available and unavailable members. Include an all-soft or
   equivalent selected-soft case so the checkmark is driven by `.selected`, not an
   inferred count of preferred siblings.
4. Strengthen the soft-disabled Launch Control PNG scenario in
   `tests/ace/tui/visual/test_ace_png_snapshots_models_panel.py`: supply alias views
   whose selector-member metadata agrees with the provider-disable snapshot, highlight
   the affected pool row so its description is visible, and exercise a yellow selected
   soft member beside a red unavailable member. Regenerate only the corresponding
   `models_panel_provider_soft_disabled_120x40.png` golden and inspect its actual,
   expected, and diff output before accepting it.

## Verification

- Run the focused alias-rendering tests covering pool/fallback descriptions.
- Run the dedicated Models-panel visual snapshot case, update its golden intentionally,
  and inspect the rendered image to confirm the entire selected soft target and both
  routing glyphs read as one amber state with no dangling `soft` label.
- Run `just install` before repository verification, then `just check` as required for
  changes in this repository. If scoped selection escalates or reports unusual coverage,
  follow the repository guidance for `just check-full` through `/sase_monitor`.

## Acceptance criteria

- A soft-disabled member selected by the existing resolver renders with an amber arrow,
  amber checkmark, and amber `provider/model@effort` target, with no `soft` suffix.
- A soft-disabled member spared by the existing resolver renders with an amber X and
  amber full target, with no `soft` suffix.
- Hard-unavailable members remain red, ordinary available members remain green, and
  routing choice, pool counts, cursor consumption, fallback order, and weights do not
  change.
- The refreshed visual snapshot makes the selected soft and unavailable states
  distinguishable at a glance, and the focused tests verify the Rich style spans rather
  than relying only on plaintext.
