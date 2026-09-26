---
tier: epic
parent_bead: sase-19x.11
title: Repair card-block landing lint and memory drift
goal:
  The scrollbar sync helper is public, and generated bead memory again matches the
  creation-reason template, without reopening closed card-block behavior.
phases:
  - id: publish-scrollbar-sync
    title: Make the block scrollbar sync helper public
    size: small
    depends_on: []
    description:
      "publish-scrollbar-sync: rename _sync_scrollbar_position to the public
      sync_scrollbar_position so panel_transitions.py can import it, and leave the
      sase-1ab legacy private import untouched."
  - id: restore-bead-memory
    title: Restore the creation-reason paragraph in generated bead memory
    size: small
    depends_on: []
    description:
      "restore-bead-memory: rerun sase memory init --no-commit so
      sase/memory/sase_beads.md regains the -w/--reason contract from its template and
      the README line and token counts match."
proposed_by: bbugyi200.athena.sase-19x.11.land
create_time: 2026-09-26 17:16:19
status: wip
---

- **PROMPT:**
  [prompts/202609/card_block_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/card_block_landing_repairs.md)
- **PARENT:**
  [202609/card_block_landing_gaps.md](https://github.com/sase-org/sase--plans/blob/main/202609/card_block_landing_gaps.md)

# Repair card-block landing lint and memory drift

Epic `sase-19x.11` is otherwise landed. These two defects were introduced by its commits
and still fail repository checks. Do not close `sase-19x.11`, do not edit its plan
status, and do not add Justfile `--epic-symbol` entries. The parent land agent resumes
after this child epic lands.

## publish-scrollbar-sync

`just symvision` fails because `_sync_scrollbar_position` in
`src/sase/ace/tui/widgets/decks/main_view_blocks.py` is imported by non-test
`src/sase/ace/tui/widgets/decks/panel_transitions.py`. Commit `bff09e3cf1`
(`sase-19x.11.1`) added that helper. Symvision's rule is to make a private function
public when a real non-test file needs it.

- Rename `_sync_scrollbar_position` to `sync_scrollbar_position`.
- Update the in-file call in `_scroll_main_to_top` and the import and call in
  `panel_transitions.py` (`_synced_block_scroll_to`).
- Do not add a pragma or an `--epic-symbol` entry. Do not move the helper into another
  module. Do not change scroll behavior.
- Do not edit `_legacy_sase_shell_syntax_enabled` or
  `src/sase/agent/legacy_sase_shell_syntax.py`. That private import belongs to open epic
  `sase-1ab` and is already recorded there.
- Re-run
  `tests/ace/tui/widgets/decks/test_deck_block_paged_pilot.py::test_block_spread_to_paged_keeps_scrollbar_in_sync`.
- Run `just symvision`. `_sync_scrollbar_position` must be gone from the error. The
  remaining `_legacy_sase_shell_syntax_enabled` error is expected and out of scope.
  `just check` may still exit non-zero solely because of that pre-existing import;
  record that and do not expand the phase.

## restore-bead-memory

Commit `f583cd5097` (`sase-19x.11.4`) regenerated `sase/memory/sase_beads.md` and
dropped the creation-reason contract that `sase-1ap.2` had published.
`sase memory init --check --diff` on this tree reports exactly:

- `sase/memory/sase_beads.md` `+6 −1`: put `-w "<why this bead was filed>"` back on the
  `sase bead create` example, and restore the paragraph that `-w/--reason` is required,
  that `@<path>` reads it, and that blank or over-2000-character reasons are rejected.
- `sase/memory/README.md` `+4 −4`: the `sase_beads.md` line and token counts, and the
  total line and token counts.

The template `src/sase/main/init_memory/templates/memory-sase-beads.template.md` already
contains that contract. Do not hand-edit the generated files.

- Run `sase memory init --no-commit`. The command commits unless `--no-commit` is set;
  the turn finalizer owns the commit.
- Confirm the resulting diff is only those two generated updates. Keep the Agent Data
  Card Block glossary strand and the `sase turn` wording.
- Run `sase memory init --check` and require a clean check.

The two phases are independent.
