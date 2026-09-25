---
tier: epic
title: Keep bead note previews compact in split Context cards
goal: "Make the agent-authored bead note preview readable and bounded by three visible
  body lines in narrow and split Main deck Context cards.

  "
parent_bead: sase-18z
phases:
  - id: responsive_preview
    title: Bound note previews to the visible Context-card width
    size: medium
    depends_on: []
    description: "responsive_preview: Pass the available card width to note rendering,
      keep the preview within three physical body lines, and preserve attribution,
      overflow, and hints.

      "
  - id: visual_proof
    title: Verify narrow note previews in the running TUI
    size: small
    depends_on:
      - responsive_preview
    description:
      "visual_proof: Add width-sensitive coverage, refresh and inspect the targeted PNG,
      and capture and inspect a live note preview when local data supports it."
proposed_by: bbugyi200.athena.sase-18z.land
create_time: 2026-09-25 10:45:24
status: wip
---

- **PROMPT:**
  [prompts/202609/bead_note_split_layout.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/bead_note_split_layout.md)
- **PARENT:**
  [202609/agent_bead_note_previews.md](https://github.com/sase-org/sase--plans/blob/main/202609/agent_bead_note_previews.md)

# Remaining work for sase-18z

The two original phases are complete: `sase-core` commit `c558f88942a1` adds schema-2
note previews, and sase commit `43f724619f` pins that revision and renders them. The
focused facade and TUI suite passed (65 tests). However, the checked-in
`agents_bead_note_preview_120x40.png` shows the right-hand Context card rewrapping each
nominally 80-cell preview line into several physical lines. The note consumes most of
the card and exceeds the plan's three visible body-line limit in a split layout. The
renderer currently computes note line length against `REASON_LINE_CELL_LIMIT` without
the actual card width.

Implement only this remaining layout repair:

1. Make preview wrapping and truncation account for the available Context-card width in
   spread, paged, narrow, and split layouts. Keep at most three visible note-text lines,
   a clear full-detail continuation for truncated content, and an accurate earlier-note
   count. Preserve attribution, safe plain-text rendering, the existing bead hint
   target, and cached off-thread loading. Avoid per-keypress file or bead-store reads.
2. Add focused width-sensitive tests that count physically visible body lines at narrow
   and wide widths, including wide Unicode and a long author/role. Refresh the targeted
   PNG golden and inspect it for legibility. Capture and inspect a live TUI screenshot
   with a real agent-authored note if the local workflow can display one; record
   concrete evidence if it cannot.
3. Run `just fix`, the targeted visual capture, and guarded `sase tool run check`. Do
   not run `just check-full`. Report any unrelated failure separately.

The parent epic's land agent will recheck the work and handle epic symbols, closure, and
the original plan-file status after this child epic lands.
