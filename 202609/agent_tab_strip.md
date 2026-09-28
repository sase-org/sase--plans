---
tier: tale
title: Complete the Agents tab strip
goal:
  The flagged Agents tab strip is fully usable, visually verified, and the assigned
  phase bead is closed.
size: medium
proposed_by: bbugyi200.athena.sase-1bc.7
bead: sase-1bc.7
create_time: 2026-09-28 08:35:11
status: wip
---

- **PARENT:**
  [202609/agents_dynamic_tabs.md](https://github.com/sase-org/sase--plans/blob/main/202609/agents_dynamic_tabs.md)
- **BEAD:**
  [sase-1bc.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bc/sase-1bc.7.md)

# Complete the Agents tab strip (sase-1bc.7)

Implement phase 7 of `plan:202609/agents_dynamic_tabs.md` in the current `sase`
checkout. Keep the `agent_tabs` flag boundary and the existing phase 6 tab index, scope,
keys, and persistence. This plan covers only `sase-1bc.7`; do not close the parent epic
or another phase.

1. Build a focused `AgentTabStrip` and immutable descriptor from the existing
   `PanelTabStrip` behavior. Reuse terminal-cell click ranges, resize reflow, and
   tooltip plumbing. Render the contract's accent labels, machine glyph, half-block
   active pill, neutral count, `S`/`F`/`U` attention badges, incomplete `N+`, divider,
   and right-side health or diagnostic text. Derive named accents from configured
   overrides or the established project palette. Keep the one-tab and flag-off header
   hidden and pixel-identical to the current view.
2. Project descriptors from the existing ordered catalog and the committed,
   tab-independent query result so counts and attention are query-aware while tab
   existence is not. Include the latched emptied tab. Track new-root arrivals for
   inactive tabs, clear each dot on visit, and repaint only when the catalog, counts,
   attention, health, active key, or arrival state changes. Keep rendering and tab
   switches free of disk, network, and roster projection work.
3. Add full, compact, and micro tiers, followed by an active-centered overflow window at
   narrow widths. Give `‹N` and `N›` their own terminal-cell click targets, tint them
   when hidden tabs need attention, and open the tab picker from either. Upgrade the
   existing picker to searchable rows with glyph, accent label, count, attention, and
   machine information; retain its palette and `pick_agents_tab` entry points.
4. Integrate strip tabs with apostrophe jump hints through the existing navigation
   target and entry jump mode. Make the active/latch state and the three empty causes
   visible: a genuinely empty tab, a query hiding its roots (including matches elsewhere
   and a clear-filter hint), and an unavailable feed (with the Admin Center Machines
   route). Keep the All-tabs presentation contract for the later layout phase in mind
   without implementing that phase's ladder.
5. Add behavior tests for cell-width click mapping, tier transitions, overflow
   selection, picker search, badges and arrival clearing, empty causes, and repaint
   gating. Add and inspect visual goldens for hidden, standard, attention, narrow
   overflow, all three empty states, a 32-character name, and picker. Capture and
   inspect live wide and narrow screenshots with the flag on. Run the relevant focused
   tests, `just fix`, `just fix-tui-screenshots` and inspect its report and each changed
   golden, then `just check` (the default gate). Read `lint_and_test.md` before
   finishing.
6. Before closing, run `sase bead epic-symbols sase-1bc.7` and resolve every remaining
   symbol or re-key its Justfile line to an open bead. Close only `sase-1bc.7` with
   `sase bead close sase-1bc.7 --note "<verified behavior and checks>"`. Record any
   discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes on this phase, including
   a check failure that reproduces unchanged on the clean base tree; do not create a
   bead.
