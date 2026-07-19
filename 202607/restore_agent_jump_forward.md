---
tier: tale
title: Restore jump-forward navigation on every ACE tab
goal: 'Ctrl+Shift+O walks forward through the current tab''s entry-jump stack on Agents,
  Artifacts/PRs, and Axe, while Ctrl+K remains available for the Agents metadata-section
  navigation that replaced the old binding.

  '
create_time: 2026-07-19 08:06:39
status: done
prompt: 202607/prompts/restore_agent_jump_forward.md
---

# Plan: Restore jump-forward navigation on every ACE tab

## Context and intended behavior

ACE already maintains independent back and forward entry-jump stacks for ChangeSpecs, Axe entries, and the richer Agents
anchors (rows, grouping banners, and split panels). `Ctrl+O` walks backward through those stacks, but the shared forward
action is still bound to `Ctrl+K`. When Agents metadata-section navigation adopted `Ctrl+J` / `Ctrl+K`, ACE resolved the
collision by suppressing jump-forward on the Agents tab even though the Agents forward-stack behavior remained
implemented and tested.

Use `Ctrl+Shift+O` as the default forward counterpart to `Ctrl+O` on every main tab. Keep `Ctrl+K` unchanged for
previous-metadata-section navigation on Agents and for unrelated, locally scoped text-editor and modal behaviors. User
overrides of `jump_to_entry_forward` must continue to flow through the existing keymap registry. This is presentation
and TUI navigation wiring, so it remains in the Python frontend rather than changing the Rust core boundary.

## Default keymap and display normalization

- Change the bundled `ace.keymaps.app.jump_to_entry_forward` default and the static fallback binding from `ctrl+k` to
  `ctrl+shift+o`, keeping `default_config.yml` as the runtime source of truth and `bindings.py` in parity with it.
- Preserve the action identifier and configuration field so existing user overrides remain compatible; do not migrate or
  repurpose the Agents `prev_agent_metadata_section: ctrl+k` setting.
- Generalize key-display formatting for nested modifiers so the new chord appears consistently as `Ctrl+Shift+O`, not
  `Ctrl+SHIFT+O`, in help, command-palette, and footer-capable display surfaces. Retain existing formatting for single
  modifiers, aliases such as Ctrl+Space, named keys, and compound alternatives.

## Restore all-tab dispatch and discoverability

- Remove the runtime `AceApp.check_action` exclusion that disables `jump_to_entry_forward` on Agents. Continue using the
  existing Agents anchor restore path and artifact-file-viewer guard; the keystroke must add no I/O, refresh path, or
  other work beyond the established in-memory action.
- Broaden command metadata for `jump_to_entry_forward` from ChangeSpecs/Axe to all three tabs, and replace its stale
  `ctrl+k` search alias with the new chord. Leave metadata-section commands Agents-only.
- Put the configured back/forward pair back in the Agents help Navigation section alongside the separate `Ctrl+J` /
  `Ctrl+K` metadata-section pair. The ChangeSpecs and Axe help builders already consume the configured pair and should
  automatically show the new default once the registry changes.

## Documentation

- Update the canonical ACE navigation reference to describe `Ctrl+O` / `Ctrl+Shift+O` as back/forward jump-stack
  navigation on Artifacts/PRs, Agents, and Axe, while continuing to document `Ctrl+J` / `Ctrl+K` as Agents metadata
  navigation.
- Extend the jump-stack explanation to cover walking forward after a back-jump and update the current
  ChangeSpecs-in-practice guide so it no longer advertises the obsolete `Ctrl+R` / `Ctrl+K` PR-history pairing.
- Do not rewrite unrelated `Ctrl+K` documentation or bindings for prompt history, revive/load-more modals, editors, or
  other locally scoped surfaces.

## Verification

- Update registry and binding tests to assert `ctrl+o` / `ctrl+shift+o` are the unique default back/forward actions and
  that `ctrl+k` remains assigned to previous Agents metadata navigation.
- Update command-catalog and availability tests to require the forward command's `Ctrl+Shift+O` display and all-tab
  scope, while retaining Agents-only scope for metadata-section commands.
- Update help-display tests so all three tab guides expose `Ctrl+O / Ctrl+Shift+O` for jump-stack navigation and the
  Agents guide also exposes `Ctrl+J / Ctrl+K` for metadata sections. Add focused coverage for multi-modifier display
  formatting.
- Add or adapt an end-to-end key-dispatch regression on the Agents tab that presses `ctrl+shift+o` and proves it reaches
  the forward action. Keep the existing Agents row/banner/panel round-trip tests as behavioral coverage, updating stale
  key-name descriptions where appropriate.
- Run the focused keymap, command, help, and entry-jump tests, then run `just install` followed by the required
  `just check`. Re-scan current-source keymap references afterward to ensure no main-tab jump-forward surface still
  advertises `Ctrl+K`, without altering intentional local uses.

## Risks and constraints

- Multi-modifier chords can be represented differently by terminal protocols; use Textual's established
  `ctrl+shift+<key>` spelling (already used elsewhere in ACE) and cover real binding dispatch through the TUI test
  harness.
- The two `Ctrl+K` app bindings currently coexist only because tab gating makes them mutually exclusive. Moving
  jump-forward to a unique chord must not weaken duplicate-key validation or change how custom keymap collisions fall
  back to defaults.
- Agents jump restoration includes stable panel/banner anchors and an artifact-viewer navigation guard. Re-enable the
  existing action wholesale rather than introducing a separate Agents implementation, so those invariants and the
  keystroke-path performance constraints remain intact.
