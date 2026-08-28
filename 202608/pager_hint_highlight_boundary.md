---
tier: tale
title: Limit pager hint highlighting to the bracketed key
goal:
  Pager hint backgrounds cover only the bracketed jump key while icons retain their
  intended foreground styling.
size: small
proposed_by: bbugyi200.athena.0fd
---

- **AGENTS:**
  - [bbugyi200.athena.0fd](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.0fd.md)
- **COMMITS:**
  - [65e0974](https://github.com/sase-org/sase/commit/65e09744a0180e86a21574be01dfc6882d90b969)
    — fix(pager): bound hint capsule styles

# Limit pager hint highlighting to the bracketed key

## Outcome

Render every pager jump hint as the same plain-text sequence it uses today—
`[<hint>]<kind icon><non-breaking space><target>`—while confining the capsule background
to `[<hint>]`. The icon and its following non-breaking space must retain their existing
kind-aware foreground treatment without inheriting the hint's yellow background.

## Root cause and scope

The current screenshot shows the capsule background extending through the kind icon for
bead, file, plan, and URL targets. In `src/sase/pager/_labels.py`, `_label_prefix()`
creates a Rich `Text` with `_LABEL_STYLE` or `_LABEL_MATCH_STYLE` as the object's base
style, then appends the icon with a foreground-only span. Rich combines that appended
span with the base style, so the icon and non-breaking space keep the base background
even though they have their own foreground style.

This is a presentation-only correction. It must not change link scanning, hint
allocation or matching, target resolution, key handling, text content, marker choice,
spacing, wrapping, or navigation behavior. No keymap or configuration change is needed.

## Implementation

1. Refactor `_label_prefix()` in `src/sase/pager/_labels.py` so the returned Rich `Text`
   starts without a capsule base style. Append `[<hint>]` with the state-specific hint
   style, then append the kind icon and non-breaking space with an explicit icon style.
   Preserve all existing state semantics:
   - a normal live hint uses `_LABEL_STYLE` only on `[<hint>]`;
   - a pending-prefix match uses `_LABEL_MATCH_STYLE` only on `[<hint>]`;
   - a pending-prefix non-match remains visually dim, including its marker, but gains no
     capsule background outside `[<hint>]`;
   - a dangling hint and marker remain dim, and the existing `⊘`, `(missing)`, target
     styling, and destination-kind accents remain unchanged.
2. Strengthen `tests/pager/test_labels.py` with style-boundary assertions, not only
   plain-text assertions. Resolve the effective Rich style at representative offsets and
   prove that:
   - every character of `[<hint>]` has the expected capsule background;
   - the immediately following icon and non-breaking space have no capsule background
     and retain the expected marker foreground/emphasis;
   - the target itself has no capsule background;
   - matching and non-matching pending-prefix states do not reintroduce background
     bleed, and dangling presentation remains dim. Keep the existing assertions on the
     exact plain-text prefix so this style-only fix cannot alter spacing or marker
     placement.
3. Extend `tests/pager/visual/test_app_png_snapshots.py` with a compact labeled document
   containing representative target kinds (at minimum a file path and URL), rendered at
   the pager suite's narrow and wide sizes. Add the corresponding PNG goldens under
   `tests/pager/visual/snapshots/png/`, inspect the generated actual/expected/diff
   artifacts, and accept only the intended change: yellow behind `[<hint>]`, never
   behind the icon or non-breaking space. Existing zero-link and section-rule goldens
   must remain byte-identical.

## Verification

1. Run the focused semantic tests in `tests/pager/test_labels.py`, plus the pager app
   tests that exercise pending-prefix and dangling-label transitions.
2. Generate the new labeled snapshots with `--sase-update-visual-snapshots`, visually
   inspect them and any diff artifacts, then rerun `just test-visual` without update
   mode to prove exact snapshot equality.
3. Run `just check` for the repository-wide lint gates and diff-scoped test lane. This
   narrow rendering change does not require `just check-full` unless scoped selection
   escalates or reports an unusual selection.

## Acceptance criteria

- In normal and matching-prefix states, the computed background color ends at the
  closing `]` for both one- and two-character hints.
- The icon and non-breaking space preserve their target-kind foreground style and have
  no yellow capsule background.
- Pending non-matches and dangling links retain their current dim semantics without
  changing text, marker, target, or `(missing)` output.
- Hint activation, copy/edit/follow actions, label assignment, wrapping, and footer
  behavior remain unchanged.
- Focused tests, exact PNG snapshots, and `just check` pass.
