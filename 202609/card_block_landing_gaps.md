---
tier: epic
parent_bead: sase-19x
title: Finish card-block landing gaps
goal:
  Card-block transitions, legacy Reply visuals, navigation latency, and glossary context
  satisfy the remaining acceptance requirements found while landing sase-19x.
phases:
  - id: scrollbar-transition
    title: Keep the scrollbar in sync across card-block mode changes
    size: medium
    depends_on: []
    description:
      "scrollbar-transition: remove the block-spread to block-paged scrollbar
      desynchronization and its snapshot workaround; add a focused regression."
  - id: legacy-reply-visual
    title: Verify the legacy followup Reply block heading visually
    size: small
    depends_on:
      - scrollbar-transition
    description:
      "legacy-reply-visual: inspect the still-reachable legacy followup_agents Reply
      heading and per-phase blocks in a targeted visual capture."
  - id: block-performance
    title: Verify and improve card-block navigation latency
    size: medium
    depends_on:
      - scrollbar-transition
    description:
      "block-performance: profile block cycling and sticky-Reply j/k, remove avoidable
      work, and record controlled latency against the original target and baseline."
  - id: glossary
    title: Add the Agent Data Card Block glossary term
    size: small
    depends_on: []
    description:
      "glossary: add a concise card-block strand and link it from Agent Data Card,
      integrating the concurrent sase-turn terminology change."
proposed_by: bbugyi200.athena.sase-19x.land
create_time: 2026-09-26 14:47:52
status: wip
---

- **PROMPT:**
  [prompts/202609/card_block_landing_gaps.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/card_block_landing_gaps.md)
- **PARENT:**
  [202609/agent_data_card_blocks.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_data_card_blocks.md)

# Finish card-block landing gaps

## Context

Epic `sase-19x` has ten closed phases. Its landed code and 87 focused card-block tests
confirm the model, Reply builders, block modes, navigation keys, rail, cutover, and user
docs. The `card_blocks` flag definition and `decks/flag.py` have been removed; the
reopened flag bead `sase-1ad` was closed after `tools/check_feature_flags` exited 0.
`sase bead epic-symbols sase-19x` reports no entries.

The land audit found four remaining items:

1. `sase-19x.9` note #2 records that a block-spread to block-paged transition leaves
   `ScrollBar.position` stale until a later explicit scroll. The micro PNG test in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_deck_blocks.py` works around it
   with a down-and-back scroll at the end of the test.
2. `sase-19x.4` note #1 requests visual inspection of the legacy `followup_agents` Reply
   heading after per-phase blocks were added. Existing seven block-state PNGs cover
   session Reply blocks; there is no explicit legacy followup Reply golden.
3. The cutover bench accepted a 100 ms guard while the original plan targeted j/k p95
   below 16 ms. Its pre-cutover flag-off arm also missed 16 ms under host load, and
   block-cycle steady-state median was about 50 ms. The remaining task is to distinguish
   host floor from avoidable feature cost and make block cycling as fast as the current
   rendering architecture allows without changing semantics.
4. `sase-19x.10` note #1 supplies Agent Data Card Block glossary text; epic note #1 asks
   the land agent to apply proposed memory changes directly. Concurrent epic
   `sase-1ab.5` is renaming sase shells to sase turns in glossary memory. Work from the
   latest landed wording, and do not reintroduce retired terminology.

These are the only child phases. Closing `sase-19x`, running the final Symvision pass,
and marking its linked plan done belong to the resumed parent land agent.

## Phase acceptance

### scrollbar-transition

- Reproduce the stale position in a focused Textual pilot that changes a Reply card from
  block-spread to block-paged, including a narrow split.
- Repair the widget/state transition so `ScrollBar.position` tracks the scroller in the
  first settled frame without a synthetic down-and-back scroll.
- Remove the workaround from the micro visual test, run its targeted visual update, and
  inspect the retained report and PNG diff. Preserve the existing reading anchor and
  `[ / ]` navigation behavior.

### legacy-reply-visual

- Build a deterministic non-session root with legacy `followup_agents` (including a
  gate), exercise both hint and non-hint paths, and inspect a targeted PNG or live
  capture of the `AGENT REPLY · N` heading and per-phase dividers.
- Refresh only goldens whose rendered content really changed. Record why no existing
  golden changed if the legacy path had no prior visual coverage.

### block-performance

- Profile the 10-shell, approximately 500-line-per-shell fixture in
  `tests/ace/tui/bench_tui_jk_blocks.py` using the existing TUI perf hooks. Compare
  sticky-Reply j/k with the prior flag-off numbers in `sase-19x.9` note #3, and measure
  block-cycle separately under a controlled, low-contention run.
- Identify whether `show_main_document` or a full preferred-card re-show performs
  avoidable measurement/render work on a block-only page swap. Use a narrow state update
  only if it preserves follow/arrival state, reading anchors, rail, and split
  independence; add a meaningful regression assertion for any changed path.
- Run the benchmark after the change. Meet the original 16 ms j/k p95 when the host
  permits; if the unchanged flag-off baseline also exceeds it, document the measured
  host floor and show no feature regression. Keep the block-cycle median no worse than
  the recorded approximately 50 ms and prefer a measured improvement.

### glossary

- Follow the `sase_memory_write` procedure. Add `Agent Data Card Block` (aliases
  `card block`, `card blocks`) as a glossary strand with the phase note's one-unit,
  stable-identity, newest-landing, `[ / ]`, and rail semantics.
- Update `Agent Data Card` to link the new term and state that a card can contain
  optional, non-nesting blocks. Integrate with `sase-1ab.5` so the strand uses the
  currently shipped `sase turn` terminology.
- Run `sase memory init` and inspect the generated instructions.

## Verification

Each code phase runs focused tests for its changed behavior, then `sase tool run check`
(`just check`), never `just check-full`. UI phases use targeted visual maintenance and
inspect every changed golden. The glossary phase republishes memory and runs the same
file-change check. Report any failure introduced outside these changes through the
normal SASE task-triage workflow, without expanding this child plan.
