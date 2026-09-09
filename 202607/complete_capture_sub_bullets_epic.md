---
tier: tale
title: Complete and land the capture-sub-bullets epic
goal:
  The remaining CLI acceptance coverage is added, the completed feature is revalidated
  against concurrent changes, and epic gh_bobs-org__bob-cli-2 is closed and finalized
  without force.
bead: gh_bobs-org__bob-cli-2
create_time: 2026-09-09 19:53:07
status: wip
---

- **PARENT:**
  [202607/capture_sub_bullets.md](https://github.com/sase-org/sase--plans/blob/main/202607/capture_sub_bullets.md)
- **BEAD:** gh_bobs-org__bob-cli-2

# Complete and land the capture-sub-bullets epic

## Goal

Finish the one remaining acceptance-coverage gap for epic bead `gh_bobs-org__bob-cli-2`,
revalidate the complete feature across `bob-cli` and the linked `chezmoi` repository,
then perform the epic landing sequence without forcing the bead closed.

## Audit context

- The epic has four closed phase beads: `gh_bobs-org__bob-cli-2.1` through `.4`, all
  with resolution `done`.
- Full current and historical note review found no `PROPOSED FOLLOW-UP:` entries on any
  child bead.
- The implementation commits are `31a10c59` (scanner), `0dc8d666` (sub-bullet writing),
  `851d7a16` (`capture-tasks`), `8831506c` (bob-cli documentation), and linked-chezmoi
  commit `745988aa` (Hammerspoon picker).
- The actual scanner, capture writer, discovery command, CLI wiring/documentation,
  Hammerspoon request model, picker, and tests have been reviewed against
  `@plan:202607/capture_sub_bullets.md`.
- `just all` passes in `bob-cli`; `just fmt-lua` and `just test-hammerspoon` pass in
  `chezmoi` with 14 successes.
- The only non-epic commit found after the epic began is chezmoi commit `0f8691c1`,
  which changes only `home/dot_config/aliases.sh`; it neither duplicates nor conflicts
  with the epic and needs no integration change.
- The linked epic plan deliberately leaves the older Pomodoro block-ID error text out of
  scope and calls correcting its false underscore claim a worthwhile separate follow-up.
  No existing bead matched that work.

## Implementation

1. Complete the planned CLI integration coverage in `tests/cli.rs`.
   - Extend `capture_sub_bullet_errors_are_actionable_in_human_and_json_modes` with an
     invalid route-character case, such as `body @bad.route^parent`, expecting exit code
     2 and `sub-bullet capture route must contain only A-Z, a-z, 0-9, '_' or '-'`.
   - Add an invalid block-ID-character case, such as `body @cash^bad.id`, expecting exit
     code 2 and
     `sub-bullet capture block ID must be non-empty and contain only A-Z, a-z, 0-9 or '-'`.
   - Keep both cases inside the existing human/JSON loop so each contract is exercised
     in both formats and the vault remains unmodified on error.

2. Revalidate the completed epic.
   - Run the focused CLI test for
     `capture_sub_bullet_errors_are_actionable_in_human_and_json_modes`.
   - Run `cargo fmt --check`, `git diff --check`, and `just all` in `bob-cli`.
   - Reopen the configured `chezmoi` repo with `sase repo open` before inspection,
     confirm commit `745988aa` remains present and the checkout clean, then run
     `just fmt-lua` and `just test-hammerspoon`.
   - Recheck the commit range beginning at `31a10c59` in `bob-cli` and the
     timestamp-equivalent range in `chezmoi` so any commits that land while this plan is
     executing are reviewed for use of, duplication of, or conflict with the epic's
     scanner/capture APIs and picker flow. Integrate any newly relevant change and rerun
     the affected checks.

3. Land and close the epic. This is the final phase and must be completed in this order.
   - File the plan's worthwhile out-of-scope correction as a task bead with a
     description naming `gh_bobs-org__bob-cli-2` as its source: fix the Pomodoro
     block-ID usage error so it no longer claims underscore is accepted when
     `collect_done::is_block_id_byte` rejects it. Create it as `open`, refine if needed,
     then set it to `ready`. Do not file child-note follow-ups because the child
     histories contained none.
   - Close `gh_bobs-org__bob-cli-2` without `--force`, using a detailed `--note` that
     records: all four child beads and implementation commits verified; the source and
     acceptance coverage reviewed; `just all`, `just fmt-lua`, and
     `just test-hammerspoon` results; the unrelated `0f8691c1` integration judgment plus
     any later-commit review; the newly created follow-up bead ID; and that no
     `PROPOSED FOLLOW-UP:` entries were omitted because none existed.
   - Only after the close succeeds, check whether a `symvision` recipe is available. If
     it is, run `just symvision`, remove only stale `gh_bobs-org__bob-cli-2` whitelist
     entries and unused code it reports, rerun focused checks plus `just symvision`
     until clean, and then rerun `just all`. If no recipe is available, record that fact
     in the final handoff.
   - Open the configured plans sidecar with `sase repo open plans`, edit the linked epic
     plan `202607/capture_sub_bullets.md`, and change only its frontmatter `status: wip`
     to `status: done`.
   - Verify `sase bead show gh_bobs-org__bob-cli-2` reports `CLOSED` with resolution
     `done`, the linked plan reports `status: done`, all relevant diffs pass
     `git diff --check`, and no unexpected working-tree changes exist.

## Constraints

- Never force-close merely to make the close command succeed. If the descendant guard
  rejects the close, inspect the named beads and finish or deliberately reopen them.
- Preserve unrelated user changes in every checkout and sidecar.
- Use `apply_patch` for source and plan-file edits.
- Use the SASE bead workflow for bead creation/update/close and the SASE repo workflow
  before accessing linked or sidecar repositories.
