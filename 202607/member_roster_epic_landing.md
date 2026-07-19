---
tier: tale
title: Complete and land the numbered member-roster epic
goal: 'Repair the post-start Statistics fixture regression that prevents the full
  repository gate from passing, revalidate the completed member-roster epic, and finish
  its bead, Symvision, and durable-plan landing sequence.

  '
create_time: 2026-07-18 20:58:12
status: done
prompt: 202607/prompts/member_roster_epic_landing.md
---

# Plan: Complete and land the numbered member-roster epic

Epic `sase-6w` is functionally implemented by its four closed child beads. The land audit confirmed the roster renderer,
fold-aware family detail, digit jump navigation, help/footer integration, and PNG coverage in the actual source and
bead-tagged commits. Focused verification passed 85 relevant tests. The branch is `master`, aligned with
`origin/master`, and has no current ChangeSpec.

The integration audit started at the first epic commit, `657ebce13`, and reviewed every later non-epic commit. The only
direct overlap was the Statistics-tab change, which updated the Agents help modal before the epic's final test commit
reconciled that surface. A later forced-name-reuse fix preserves clan and family container reservations and is
compatible with the epic's identity-keyed jump maps.

The repository-wide `just check` gate nevertheless found one remaining post-start regression in the Statistics fixtures.
Both `tests/ace/tui/test_statistics_pane.py` and `tests/ace/tui/visual/_ace_config_center_statistics_helpers.py` use the
sample skill name `sase_beads`. The `pyscripts-260619` structure checker performs a fixed-substring search for the root
executable `tools/sase_bead`, so it mistakes those fixture values for references that should use the closer
`tests/ace/tui/tools/` directory. The checker is a dated vendored script and its local `tools/AGENTS.md` explicitly
forbids editing it in this repository.

## Phase 1: Repair the post-start Statistics fixture regression

Replace the colliding `sase_beads` sample value in both Statistics fixtures with another representative, real SASE skill
name that does not contain the `sase_bead` executable basename. Preserve the relative counts and ordering so the
Statistics behavior under test remains unchanged. Update only the PNG goldens whose rendered skill label changes; do not
broadly accept unrelated visual differences.

Verify the diagnosis directly with `just _lint-pyscripts`, then run the focused Statistics tests and affected visual
snapshots. Keep the numbered-roster implementation unchanged unless those checks expose an actual interaction.

## Phase 2: Revalidate the epic and integration window

Rerun the focused roster/family/jump/help suites used by the land audit, then run `just check` to cover formatting, all
linters (including Symvision), SASE validation, committed-plan validation, unit tests, and PNG visual snapshots. Review
the final diff to ensure it contains only the Statistics fixture/golden repair and any evidence-driven cleanup.

The relevant focused suite is:

```bash
.venv/bin/pytest -q \
  tests/ace/tui/widgets/test_member_roster.py \
  tests/ace/tui/widgets/test_agent_display_clan.py \
  tests/ace/tui/widgets/test_agent_display_family.py \
  tests/ace/tui/test_member_jump_navigation.py \
  tests/ace/tui/test_agents_panel_fold_mode.py \
  tests/ace/tui/widgets/test_keybinding_footer_idempotent.py \
  tests/ace/tui/modals/test_help_modal.py \
  tests/test_keymaps_defaults.py
```

## Phase 3: Close and land `sase-6w`

Only after Phase 2 is green, close the epic with:

```bash
sase bead close sase-6w
```

After closing, run `just symvision` because any epic-scoped symbol exemptions expire at closure. If it reports stale
epic whitelist entries or genuinely unused code, follow the Symvision memory workflow, remove those entries/code, and
rerun `just symvision` plus `just check` after source changes.

Finally, open the plans sidecar through the required repository-access workflow and change only the linked epic plan's
frontmatter from `status: wip` to `status: done` at `${SASE_SDD_PLANS_DIR}/202607/member_roster_sections.md`. Confirm
`sase bead show sase-6w` reports the epic closed and inspect both repositories' final status. This close, post-close
Symvision cleanup, and durable plan status update must remain the final phase.

## Risks and guardrails

- Do not edit `tools/pyscripts-260619`; it is vendored from the dotfiles repo.
- Do not regenerate the entire PNG corpus blindly. Accept only Statistics snapshots whose visible fixture label
  intentionally changed.
- Do not close the epic until `just check` passes.
- Preserve the existing member-roster feature code unless a failing focused test demonstrates remaining work.
