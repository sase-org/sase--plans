---
tier: epic
status: done
title: Finish star model alias completion verification
goal:
  Close the behavioral and visual readiness gaps left by sase-yf so the shipped
  star-triggered alias menu satisfies its original interaction, cache, and rendering
  contract under deterministic tests.
phases:
  - id: alias_behavioral_hardening
    title: Prove and harden alias interaction and catalog behavior
    size: medium
    depends_on: []
    description:
      "alias_behavioral_hardening: add the missing widget and catalog coverage for
      navigation, selection retention, token replacement, undo/redo, structured-context
      precedence, effective merged aliases, cold-load responsiveness, stale-worker
      rejection, invalidation, failure/retry, and stacked-pane isolation; fix any
      contract defects those deterministic tests expose."
  - id: alias_visual_completion
    title: Complete alias rendering and reviewed PNG coverage
    size: medium
    depends_on:
      - alias_behavioral_hardening
    description:
      "alias_visual_completion: finish the original visual contract, including the
      canonical expansion arrow, Unicode loading ellipsis, prefix highlighting, narrow
      width priority, selected-row action hints, and full/filtered/narrow/stacked PNG
      scenarios across relevant themes; inspect generated images and preserve ordinary
      model-menu snapshots unless a shared intentional fix requires review."
proposed_by: bbugyi200.athena.sase-yf.land
parent_bead: sase-yf
bead_id: sase-yf.3
create_time: 2026-09-09 19:52:25
---

- **PROMPT:**
  [prompts/202609/finish_star_model_alias_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_star_model_alias_completion.md)
- **PARENT:** [202609/star_model_alias_completion.md](star_model_alias_completion.md)
- **BEAD:**
  [sase-yf.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yf/sase-yf.3.md)

# Finish star model alias completion verification

## Context

Epic `sase-yf` added the reusable Rust `*alias` detection/edit contract in `sase-core`
commit `be8f55233bd9648a84b111da1377baab8732da81` and the ACE prompt integration in SASE
commit `a95d7c1ddfcd34d95b56de75d3ad2265f8e3b1a0`. The linked SASE plan requires
substantially more behavioral and visual evidence than those commits contain. The
existing focused suites pass, and a direct widget probe confirms basic accept/undo/redo,
but the current implementation has only six shortcut-specific widget tests and no
shortcut-specific PNG scenario. It also renders `Enter ->` and three ASCII dots where
the accepted contract specifies `Enter →` and a Unicode ellipsis, and it does not apply
match highlighting to the alias prefix.

This plan covers only that remaining epic work. Do not close `sase-yf`, retire its epic
symbols, run the parent landing-only Symvision pass, or mark the parent plan done; the
resumed `sase-yf` land agent owns those steps after this child epic lands.

## Phase 1: Prove and harden alias interaction and catalog behavior

Expand `tests/ace/tui/widgets/test_model_alias_completion.py` and the model-catalog
tests with deterministic fixtures that exercise the actual `PromptTextArea` and real
Rust binding. Cover all of the following, fixing production behavior whenever a test
reveals a mismatch:

- Bare-star opening at prompt start, after an ASCII space, and on a later logical line;
  first-row Enter and Ctrl+L acceptance; case-insensitive filtering; navigation;
  preservation of a still-matching selected alias; backspace-to-bare-star; no-match
  dismissal; and manual Ctrl+T with automatic directive menus disabled.
- Full-token replacement from a caret in the middle of the token, including text after
  the token; preservation of existing spaces, tabs, newlines, Unicode, and CRLF; exact
  caret placement; one undoable expansion edit; undo restoring the original star token
  without deleting earlier prose; redo restoring both expansion and caret; and proof
  that acceptance never submits or inserts a newline.
- Ordinary-writing and ownership boundaries: `a*b`, `path/*`, escaped stars, `**bold**`,
  list-marker spaces, completed emphasis, inline/fenced code including an unclosed fence
  and cursor-at-end boundaries, disabled prompt regions, YAML and structured/read-only
  frontmatter, placeholders, Jinja, directive/xprompt arguments, paths/references,
  Tab/snippet behavior, list editing, selection, and NORMAL mode.
- Teardown and stale acceptance after cursor movement, focus loss, Escape, prompt text
  load, mode change, pane switch, star deletion, and catalog/candidate change. Returning
  the cursor alone and undo must not reopen a dismissed menu.
- One real merged-config fixture containing built-in, plugin, and user aliases, a user
  override of a plugin alias, and degraded target metadata. Assert one effective row per
  alias name in stable catalog order, no provider/concrete-model rows, and successful
  ordinary `%m:@alias` resolution after expansion.
- A controllable worker barrier, never timing sleeps, that proves a cold catalog load
  does not block the next key event; Enter/Ctrl+L during loading is consumed; completion
  refreshes only for the still-current text, cursor, pane, mode, and request; stale,
  dismissed, unmounted, or switched-pane workers cannot revive or retarget a menu;
  failure produces one quiet unavailable state; explicit Ctrl+T retries; config
  invalidation schedules a new build; and repeated warm typing/filtering/navigation
  never calls the static builder or launch/provider resolver.

Keep the Rust core as the source of truth for trigger and edit planning. If a defect is
in shared behavior, open `sase-core` with `/sase_repo`, fix and verify it there, let
host-owned finalization land/release it, then ratchet the SASE core revision and
dependency floor through established tooling. Do not add a Python fallback.

## Phase 2: Complete alias rendering and reviewed PNG coverage

Finish the menu presentation against the original `sase-yf` contract:

- Render the selected subtitle as `Enter → %m:@<alias> · <description>` using literal
  Rich text. Omit the description cleanly when absent. At narrow widths preserve the
  expansion affordance first, drop/truncate description before the directive preview,
  and ellipsize an exceptionally long alias only after optional detail is gone.
- Use the exact `Loading model aliases…` loading label, retain a quiet non-selectable
  unavailable row, and ensure neither state claims a selectable expansion.
- Highlight the typed alias prefix through the existing shared match-highlighting helper
  without changing ordinary `%model` menu behavior. Preserve alias identity, kind
  badges, target/effort/availability/override metadata, selection pointer, row cap,
  scrolling, height budget, and cursor readout.
- Keep `Enter accept alias` in the prompt action hint only while a selectable alias menu
  is active and restore the prior prompt-mode hint on every close path. Loading and
  unavailable states must not advertise acceptance.
- Add deterministic PNG scenarios and goldens for a full bare-star menu, a filtered
  selection with expansion preview, a narrow terminal with long aliases/descriptions,
  and an active stacked pane. Exercise light and dark themes where contrast differs. Run
  the dedicated visual suite, inspect every rendered PNG and diff artifact, and accept
  only intentional changes. Existing ordinary model-completion snapshots should remain
  byte-stable unless a justified shared-rendering change is reviewed.

Update user documentation only if the verified behavior or exact labels differ from the
already-landed text; avoid broad adjacent rewrites.

## Verification

Run `just install` before SASE verification. Run the focused shortcut, directive-model,
catalog, and panel tests while iterating, then the dedicated PNG suite and `just check`.
Follow `lint_and_test.md` exactly. In `sase-core`, if modified, run its repository-wide
`just check` rather than a crate-only substitute. Record inspected PNGs and exact test
results in phase notes so the resumed parent lander can re-audit them.
