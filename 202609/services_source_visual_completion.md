---
tier: epic
title: Complete Services source-panel visual and performance verification
goal: Cover the promised source combinations and terminal sizes with reviewed PNG
  goldens, and verify navigation and refresh cost before sase-1af lands.
parent_bead: sase-1af
phases:
- id: source_visual_verification
  title: Cover Services source panels and verify visual and navigation cost
  depends_on: []
  description: 'source_visual_verification: add reviewed Services source-combination
    PNGs and record navigation and refresh measurements.'
  size: medium
proposed_by: bbugyi200.apollo.sase-1af.land
create_time: 2026-09-26 11:43:20
status: wip
bead_id: sase-1af.5
---

- **PROMPT:** [prompts/202609/services_source_visual_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/services_source_visual_completion.md)
- **PARENT:** [202609/routine_source_nav_sections.md](https://github.com/sase-org/sase--plans/blob/main/202609/routine_source_nav_sections.md)
- **BEAD:** [sase-1af.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1af/sase-1af.5.md)

# Remaining work

Epic sase-1af has shipped the Rust declaring-source contract, Python projection, four
Services panels, and health badges with first-sight builtin folding. Its
`density_polish` phase also required Services PNG inspection at 120×40, 100×30, and a
narrow layout for Builtin+User, all three source groups, and Builtin-only, plus a
key-to-paint and idle-refresh comparison. The current
`test_ace_png_snapshots_services_panels.py` has only the older two-panel fixture at
120×40 and 70×36. Its fixture supplies no `routine_origins`, so it cannot exercise
source grouping or the builtin fold and health presentation. There is no 100×30 or
Builtin-only Services golden. The phase note explicitly reported this gap.

## Implementation

1. Update the Services PNG fixture data in
   `tests/ace/tui/visual/_ace_axe_png_snapshot_tree_fixtures.py` so routine origins are
   explicit and realistic. Cover Builtin+User, all three routine sources, and
   Builtin-only. Include a failed or missing-script builtin job so the folded parent
   health badge is visible; keep the scheduler and service-proc state deterministic.
   Preserve the existing empty-routines case.
2. Update `tests/ace/tui/visual/test_ace_png_snapshots_services_panels.py` to capture
   the specified 120×40, 100×30, and narrow layouts and source combinations. Assert
   panel order, source-specific titles, collapsed builtin rows, health badge, selected
   focus, and `J`/`K` behavior as appropriate. Rename stale two-panel/Scheduled Routines
   descriptions. Use targeted `just fix-tui-screenshots -- <selector>` to generate the
   affected PNGs, inspect the report, and inspect every created/changed Services golden
   visually. If layout problems surface, fix the Services presentation code within this
   scope and add focused behavioral coverage.
3. Follow `tui_perf.md`: capture key-to-paint and idle-refresh traces on representative
   Services navigation before and after any presentation fix, or compare the current
   trace to a checked-out pre-source-panel baseline when no code fix is needed. Confirm
   no synchronous config/filesystem work enters paint or keystroke paths and no material
   `j`/`k` latency regression. Record the measured evidence in the child bead close
   note.
4. Run `just fix`, focused Services tests and targeted visual maintenance, then the
   required `sase tool run check` (`just check` gate). Do not run `just check-full`.
   Distinguish the separately owned stale `sase-19x.4` Symvision entries and unrelated
   full-suite infrastructure failures from changes in this tale; report any remaining
   gate result precisely.

## Acceptance

- Reviewed Services PNGs show the correct fixed panel order, source membership, builtin
  fold, failure badge, focus treatment, and legible titles at 120×40, 100×30, and narrow
  widths.
- The selected cases include Builtin+User, User+Plugin+Builtin, and Builtin-only; the
  empty-routines state remains covered.
- Navigation and refresh measurements show no material regression or render-path I/O.
  Focused tests and the required file-change verification are reported with their exact
  outcomes.
