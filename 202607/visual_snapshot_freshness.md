---
tier: tale
goal: Restore the ACE visual snapshot corpus to the current intentional UI and prevent
  the cross-host PNG tolerance from hiding small, material product changes.
create_time: 2026-07-15 12:24:12
status: done
prompt: 202607/prompts/visual_snapshot_freshness.md
---

# Plan: Restore ACE visual snapshot freshness

## Context and diagnosis

The current visual failures are stale expectations, not a product rendering defect. A normal `just test-visual` run
reproduces 14 failures with 183 passes and one skip. The actual captures are stable and render correctly; side-by-side
inspection ties the differences to intentional behavior already present in the product, including the
`sase ace (v0.7.1)` title, the `PARENT` ChangeSpec field, focused Vim editor/cursor-line styling, deprecated
configuration source badges, and newer runner-wait presentation.

The history shows that several feature changes updated only a subset of their affected goldens, or none at all:

- `9deb01206` changed the ACE title from PID to version without updating PNG goldens.
- `4cce6a46b` added `PARENT` to the main ChangeSpec detail widget but refreshed only an unrelated footer golden.
- `1c21d266a` introduced shared Vim/readline editor rendering while refreshing only selected Config Center states.
- `e7411b8a8` made legacy repository keys visibly deprecated and refreshed Config-tab goldens but not the edit-modal
  states that display the same source inventory.

The omissions were hidden by the repository-wide 1% ratio-only allowance in `Justfile`. An exact audit with
`SASE_VISUAL_PNG_MAX_DIFF_RATIO=0 just test-visual` finds 100 mismatches and 97 exact matches (plus one skip), including
many coherent product deltas below 1%: the title alone is about 0.10%, while the added parent row is about 0.57%. Those
snapshots continued passing until multiple small changes accumulated past the tolerance. The root problem is therefore
both an out-of-date corpus and a tolerance model that cannot distinguish low-amplitude rasterizer anti-aliasing from
small, high-contrast UI changes.

## Implementation

### 1. Make the PNG tolerance intensity-aware

Extend the PNG diff summary and tolerance decision in `tests/ace/tui/visual/png_diff.py` so the documented cross-host
allowance applies only to renderer-like anti-aliasing drift. Add an alpha-aware material-difference metric (for example,
the number of composited pixels whose color distance exceeds an empirically calibrated anti-aliasing ceiling) alongside
the existing exact changed-pixel count and ratio.

Keep the existing ratio cap for the total drift area, but require tolerated images to remain within the material-change
ceiling as well. A small high-contrast content or style change must fail even when it touches much less than 1% of the
image, while the observed macOS/Linux edge-rasterization deltas must continue to pass. Preserve an explicit override
path for deliberate renderer investigations, and report the new metric and active limits in assertion messages,
`summary.txt`, and `failure.json` so failures remain diagnosable.

Add focused unit coverage in `tests/ace/tui/visual/test_png_diff.py` for:

- exact matches and exact-mode failures;
- low-amplitude anti-aliasing-like drift within the configured ratio and material limits;
- a high-contrast one-pixel or one-glyph change that is below the ratio cap but must fail;
- explicit tolerance overrides and invalid environment configuration;
- the new summary and JSON diagnostic fields.

### 2. Realign and review the golden corpus

Regenerate the complete ACE PNG suite from the current pinned renderer rather than updating only the fourteen snapshots
that happen to exceed 1% today. Review the changed images and diff artifacts by visual family, accepting only current,
intentional states. In particular, confirm the version title, ChangeSpec `PARENT` row, Config Center deprecated-source
badge, Vim/editor focus fills, prompt/frontmatter modals, and runner-slot wait rows. Do not change product TUI behavior
merely to reproduce old screenshots.

This one-time full reconciliation should remove the hidden sub-threshold debt as well as the visible failures.
Re-running update mode on the same host must produce no further golden changes.

### 3. Document the strengthened contract

Update the `Justfile` commentary and the visual workflow documentation in `docs/development.md` (and the visual font
README if needed) to explain that the default allowance is bounded by both drift area and pixel intensity. Document how
to inspect material-difference diagnostics, how to request exact comparison, and that intentional visual changes still
require explicit golden review and acceptance.

Do not edit `memory/*.md`, `AGENTS.md`, or generated provider instruction shims without separate explicit user
authorization.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral workspace.
2. Run the focused PNG diff-helper tests, including the new low-amplitude and high-contrast tolerance cases.
3. Run `just test-visual` and require the full visual suite to pass with the normal cross-host settings.
4. On the regeneration host, run `SASE_VISUAL_PNG_MAX_DIFF_RATIO=0 just test-visual` and require an exact pass, then run
   the normal visual suite again to prove the accepted corpus is stable.
5. Run the mandatory `just check` gate and `git diff --check`.
6. Review the final binary diff inventory and representative before/after images; confirm that a second snapshot-update
   run is a no-op and that no unrelated source, memory, or instruction files changed.

## Risks and constraints

- A material-difference ceiling that is too strict will reintroduce cross-host false positives; one that is too loose
  will preserve the current blind spot. Calibrate it from known renderer-only artifacts and lock both sides of the
  boundary with unit tests.
- Bulk PNG acceptance can conceal a real regression. Group and inspect the diffs by shared visual signature rather than
  treating update mode as proof of correctness.
- Keep the change in test and documentation layers unless inspection uncovers a concrete current UI defect; the present
  evidence does not justify reverting any product behavior.
