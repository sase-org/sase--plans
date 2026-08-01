---
tier: tale
title: Move the Artifacts Chats sub-tab to third position
goal: ACE shows Chats third in the Artifacts strip and maps every numeric navigation
  surface to the new order.
create_time: 2026-07-25 06:35:16
status: done
---

- **PROMPT:** [prompts/202607/artifacts_chat_third.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/artifacts_chat_third.md)
- **AGENTS:**
  - [bbugyi200.athena.k1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.k1/README.md)
  - [bbugyi200.athena.k1--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.k1.md#member-code)
- **COMMITS:**
  - [93c58bd](https://github.com/sase-org/sase/commit/93c58bd88c547aadad4a04e77409777b1edc92a1) — feat(tui): reorder Artifacts tabs by usage

# Move the Artifacts Chats sub-tab to third position

## Goal

Change the ACE Artifacts sub-tab order from:

`Commits, Plans, Bugs, PRs, Chats`

to:

`Commits, Plans, Chats, Bugs, PRs`

The displayed numbering and fixed numeric shortcuts must become:

- `1` → Commits
- `2` → Plans
- `3` → Chats
- `4` → Bugs
- `5` → PRs

The default Artifacts sub-tab remains Commits. Existing pane behavior, lazy activation, project scope, and action
availability remain unchanged.

## Current design

`src/sase/ace/tui/artifact_tabs.py` owns the canonical `ARTIFACTS_SUBTAB_ORDER`. The following surfaces already derive
their order or numbering from that tuple and should continue doing so:

- the clickable numbered strip in `src/sase/ace/tui/widgets/artifacts/view.py`;
- forward/reverse wraparound cycling in `src/sase/ace/tui/actions/artifacts.py`;
- class-level fallback bindings in `src/sase/ace/tui/bindings.py`;
- runtime fixed numeric bindings in `src/sase/ace/tui/keymaps/bindings.py`;
- command-palette shortcuts in `src/sase/ace/tui/commands/catalog.py`; and
- onboarding labels and numeric hints in `src/sase/ace/tui/widgets/changespec_onboarding.py`.

The numbered shortcuts are intentionally fixed rather than configurable, so `src/sase/default_config.yml` has no numeric
Artifacts entries to edit. Keep one canonical order instead of introducing duplicate key-to-sub-tab maps.

## Implementation

1. In `src/sase/ace/tui/artifact_tabs.py`, reorder `ARTIFACTS_SUBTAB_ORDER` to
   `("commits", "plans", "chats", "bugs", "prs")`. Do not change `DEFAULT_ARTIFACTS_SUBTAB`, pane IDs, accents, action
   names, or pane composition.

2. Update behavioral assertions and test navigation to the new order:
   - In `tests/test_keymaps_app_bindings.py`, assert the new mapping for both runtime registry bindings and fallback
     bindings.
   - In `tests/ace/tui/test_artifacts_scaffold.py`, update the exact order contract, rendered tab-strip text, numbered
     jumps, command-palette key displays, and forward/reverse cycle expectations. Preserve coverage that cycling wraps,
     hidden tabs ignore digit presses, clicking retains pane state, and the Chats pane activates lazily.
   - Audit the complete test tree for numeric Artifacts selectors. Change Chats selectors from `5` to `3`, Bugs
     selectors from `3` to `4`, and PR selectors from `4` to `5`; Commits and Plans remain `1` and `2`. This includes
     focused pane tests, saved-query/onboarding tests, the Artifacts navigation benchmark, and visual-test setup code.
     Do not rewrite unrelated uses of those digits in modals, test data, or non-Artifact tab navigation.
   - In `tests/test_keymaps_e2e.py`, update the contextual query-shortcut matrix so it selects Bugs with `4`.

3. Update current documentation that presents the Artifacts sub-tabs as an ordered list:
   - In `docs/ace.md`, change the explicit numbered strip to `1 Commits · 2 Plans · 3 Chats · 4 Bugs · 5 PRs` and
     reorder its summary prose.
   - Reorder current prose listings in `docs/getting_started.md`, `docs/blog/posts/hello-sase-your-first-15-minutes.md`,
     `docs/blog/posts/structured-agentic-software-engineering.md`, and
     `docs/blog/posts/why-coding-agents-need-orchestration.md`.
   - Correct the “current navigation” warnings in `docs/images/sase_tui_tabs_infographic.prompt.md` and
     `docs/images/sase_tui_tabs_infographic.critique.md`; the retired PNG itself does not show these named sub-tabs and
     does not need regeneration. Leave genuinely historical changelog entries intact.

4. Refresh visual goldens under `tests/ace/tui/visual/snapshots/png/` only after inspecting the generated
   `.pytest_cache/sase-visual/` actual/expected/diff/source artifacts. Accept changes only where the Artifacts strip,
   its numeric labels, the active-sub-tab highlight, or order-derived onboarding/help content moved as intended; do not
   accept unrelated renderer drift.

## Validation

1. Run `just install` before repository checks, as required for an ephemeral SASE workspace.
2. Run focused behavioral coverage for:
   - `tests/test_keymaps_app_bindings.py`;
   - `tests/test_keymaps_e2e.py`;
   - `tests/ace/tui/test_artifacts_scaffold.py`;
   - the Chats loading/filtering tests;
   - saved-query, onboarding, footer, and popup-panel tests whose PR/Bugs/Chats digit selectors changed.

3. Run the Artifacts navigation slow benchmark test after updating its PR selector. The canonical tuple change adds no
   I/O, awaits, handlers, or render-path work, so preserve the existing in-memory cycling path and treat any
   key-to-paint regression as unexpected.
4. Run `just test-visual`, inspect all failures, update the intentional visual snapshots with the repository-supported
   snapshot command, and rerun `just test-visual` to prove the refreshed corpus passes.
5. Search the repository again for stale explicit sequences, labels, and contextual selectors (`5`/Chats, `3`/Bugs,
   `4`/PRs), distinguishing unrelated numeric input from Artifacts navigation.
6. Run the mandatory full `just check` and resolve any formatting, lint, type, validation, unit, integration, or visual
   failure before completion.

## Acceptance criteria

- The visible Artifacts strip reads `1 Commits`, `2 Plans`, `3 Chats`, `4 Bugs`, `5 PRs`.
- Bare digits select those panes only while Artifacts is active, with `3` opening Chats, `4` opening Bugs, and `5`
  opening PRs.
- `[` and `]` cycle through the same new order with wraparound.
- Command-palette shortcut displays and onboarding/help hints match the strip and numeric bindings.
- The Commits default, pane-specific actions, lazy activation/state retention, and project/query scoping are unchanged.
- Current documentation and intentional PNG goldens show the new order.
- `just check` passes.
