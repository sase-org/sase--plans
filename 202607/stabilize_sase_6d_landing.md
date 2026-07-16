---
tier: tale
title: Stabilize canonical-path visual coverage and land sase-6d
goal: 'The canonical xprompt-save PNG coverage is deterministic under the full parallel
  suite, all epic integration checks pass, and epic sase-6d is closed with its post-close
  Symvision cleanup and plan metadata finalized.

  '
create_time: 2026-07-16 16:51:34
status: wip
prompt: 202607/prompts/stabilize_sase_6d_landing.md
---

# Plan: Stabilize canonical-path visual coverage and land sase-6d

## Context

The landing audit verified all nine child beads and their commits across the main SASE repository, `sase-core`,
`sase-nvim`, chezmoi, actstat, and bob-cli. The canonical project and home trees are deployed, the legacy live-home
trees are absent, both architecture diagrams pass full-resolution review, and the propagated memory-map PNGs are
byte-identical. Later non-epic commits were reviewed on the main and linked/base branches; none require a canonical-path
integration change.

The full `just check` run passed formatting, every linter, SASE validation, and committed-plan validation, then found
one nondeterministic PNG failure:

- `tests/ace/tui/visual/test_ace_png_snapshots_xprompt_save.py::test_xprompt_save_snippet_mode_png_snapshot`
- Only the focused name-input cursor differed: 298 of 1,520,532 pixels (`0.000195984037`).
- An isolated exact-pixel rerun passed without changing the golden.
- Existing prompt-stack visual helpers explicitly set `cursor_blink = False` to make focused captures independent of
  Textual's wall-clock blink phase, but the shared xprompt-save `_open` helper does not.

Treat this as remaining integration work from the epic's Phase 9 visual refresh. Preserve the reviewed UI and existing
goldens; make the test setup deterministic rather than accepting whichever blink phase happened to be captured.

## Phase 1: Stabilize and revalidate the visual coverage

Update the shared xprompt-save visual setup so its focused name `Input` has cursor blinking disabled before the modal is
declared visually idle. Use the widget's concrete type instead of an untyped attribute suppression where practical, and
apply the stabilization in the shared helper so all four xprompt-save snapshots receive the same deterministic focused
cursor behavior.

Run the complete xprompt-save visual module at least three consecutive times to prove that the cursor no longer
alternates, then run `just check`. Do not update PNG goldens unless the stabilized output consistently demonstrates a
separate, intentional UI change; if that happens, inspect expected/actual/diff artifacts at full resolution and document
the reason before accepting it. Also run `just rust-check` so the linked Rust layout/catalog implementation remains
validated with the Python adapter used by the suite.

Before landing, confirm the main checkout and every repository opened by the audit have no unintended changes. Preserve
unrelated user work and keep generated cache/artifact files out of source control.

## Phase 2: Land epic sase-6d

This is the final phase and its ordering is mandatory:

1. Re-run `sase bead show sase-6d` and confirm all nine children remain closed.
2. Close the original epic with `sase bead close sase-6d`.
3. After the close succeeds, run `just symvision`. Epic-symbol whitelist entries for `sase-6d` expire only at this
   point. If Symvision reports stale whitelist entries or now-unused code, use the repository's audited
   `sase_memory_read` procedure for `symvision.md`, remove exactly the reported stale entries/unused code, and rerun
   proportionate tests plus `just check` for any source changes.
4. Open the plans sidecar through the required `sase_repo` workflow and set `status: done` in the frontmatter of the
   original epic plan linked by the bead, `$SASE_SDD_PLANS_DIR/202607/canonical_sase_directories.md`. Do not mark this
   stabilization plan as a substitute for finalizing the original epic plan.
5. Verify `sase bead show sase-6d` reports the epic closed, the original plan reports `status: done`, post-close
   Symvision is clean, required checks pass, and all affected worktrees contain only the intentional landing changes.
