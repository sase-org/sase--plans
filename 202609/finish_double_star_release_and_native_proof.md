---
tier: epic
title: Finish double-star release and native-editor proof
goal:
  Make the released SASE/core pair preserve structured model presentation data and
  replace the Neovim shortcut smoke's simulated conversions with proof through the real
  native completion lifecycle.
parent_bead: sase-yw.3
phases:
  - id: release_integration
    title: Ratchet the released core contract
    description:
      "release_integration: pin SASE to the published core release that retains the
      structured provider display field, lock and install that release coherently, and
      prove the shipped binding and LSP expose the contract ACE consumes."
    size: medium
    depends_on: []
  - id: native_frontend_proof
    title: Exercise the real Neovim completion lifecycle
    description:
      "native_frontend_proof: drive double-star completion through Neovim's actual
      native popup lifecycle, cover live star/backspace/manual transitions and edit
      acceptance, and run the cross-repository landing gates against the pinned LSP."
    size: medium
    depends_on:
      - release_integration
proposed_by: bbugyi200.athena.sase-yw.3.land--1
create_time: 2026-09-09 20:00:37
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_double_star_release_and_native_proof.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_double_star_release_and_native_proof.md)

# Finish double-star release and native-editor proof

## Context

Epic `sase-yw.3` was created to finish the gaps discovered while landing double-star
explicit model completion. Its two original phases are closed, and their source work is
present on the landed branches:

- `sase-core` commit `6b84036d96ff5d5495ebaf9836440c64ee65b37d` rejects unsafe selected
  shortcut values by reusing `validate_model_value`, with Rust, PyO3, service, and stdio
  regressions.
- SASE commit `fb3ff15893d507832ab28c0e4684d0ea66d44af9` carries structured advisory
  metadata into ACE, renders scoped match spans and advisories, adds focused lifecycle
  tests, and adds the requested explicit-model PNG states.
- `sase-core` commit `937bab258773c7a959ccb3f4642a8d3c727cd5da` adds the structured
  `provider_display` field to the shared completion wire.
- `sase-nvim` commit `3d17044040ad276416b2f2c81c2cc3737b6e0d66` expands the
  model-shortcut smoke coverage.

The landing audit reviewed both phase notes, the source and commits in all three
repositories, every non-epic commit since `sase-yw.3` started, and the current branch
tips. All three checkouts equal their origin `master`. Later SASE changes are disjoint
except that `c1d8efd3c` consumes newly released fleet behavior; later core changes add
fleet contracts and release 0.32.61; there is no post-shortcut Neovim drift.

The safety, ACE presentation, focused lifecycle, and model-specific visual work is
present. The full monitored visual run did not name any model-completion PNG node among
its failures, six newly added model actuals are byte-identical to their committed
goldens, and visual inspection confirmed the full light/dark, filtered, scoped narrow,
stacked, advisory, loading, and unavailable states are legible and truthful. The same
run's unrelated visual failures were routed to existing tasks `sase-x5`, `sase-yu`, and
`sase-yv`; the core loader failure was routed to exact duplicate `sase-xv`.

Two epic-caused/integration gaps remain:

1. The SASE repository still pins `sase-core-revision.txt` to
   `5fc84c50b88098257a67c32c3785bdbd5f346f29` (release 0.32.60), declares
   `sase-core-rs>=0.32.59,<0.33.0`, and locks 0.32.59. Those versions predate `937bab2`;
   their Rust completion wire omits `provider_display`, so the round trip silently
   rehydrates it as empty and ACE falls back to the lowercase provider key. Core release
   0.32.61 at `7444d1500a5869aedadf9d11c01b1678fd98a1ea` now includes the required field
   and the later core contracts consumed by current SASE.
2. `tests/lsp_model_shortcut_smoke.lua` calls `textDocument/completion` synchronously,
   invokes the private `vim.lsp.completion._convert_results`, applies edits directly
   with `vim.lsp.util.apply_text_edits`, and replaces whole buffer lines between the
   final `*`/`**`/`*` assertions. Its one manual call triggers native completion but
   validates a separate synchronous conversion. It therefore does not prove that the
   actual native popup changes from aliases to models and back during live typing, or
   that selecting a popup item applies the edit. The README currently calls this smoke
   the headless equivalent of those manual lifecycle steps.

Do not redesign the shortcut, add configuration or inventories, duplicate Rust
filter/edit policy in Python or Lua, or change release versions in `sase-core` by hand.
Preserve `%m:`, `*alias`, existing ACE behavior, and server-owned catalog order,
filtering, and edit planning.

## Phase `release_integration`: Ratchet the released core contract

Work in the SASE repository and open `sase-core` through `/sase_repo` before using it.
The required core release already exists; this phase integrates it rather than changing
core package versions.

1. Ratchet `sase-core-revision.txt` to the landed 0.32.61 release commit (or a later
   landed release proven to contain `937bab2`). Raise the `sase-core-rs` lower bound to
   that first compatible release while preserving the `<0.33.0` window, and regenerate
   `uv.lock` through the repository's normal lock workflow. Do not hand-edit resolved
   package metadata.
2. Add or tighten a release-boundary regression that sends a model row with distinct
   `provider`, `provider_display`, short hint, and advisory fields through the actual
   installed Rust filtering binding and proves every structured field survives. Keep
   wire ownership in Rust and the local Python layer limited to conversion/presentation.
   Ensure binding-surface validation covers any current SASE consumer introduced after
   the old pin, including the post-epic remote-attention integration where applicable.
3. Run `just install` and `just rust-lsp-install` from the SASE checkout. Prove the
   imported `sase_core_rs` distribution version and `.venv/bin/sase-xprompt-lsp` both
   come from the pinned release, then run focused model wire/binding/LSP parity tests
   and the repository's required `just check`. Record the exact installed revision for
   the dependent phase.

## Phase `native_frontend_proof`: Exercise the real Neovim completion lifecycle

Work in `sase-nvim`, opened through `/sase_repo`, and use the explicit LSP executable
built from the pinned SASE/core pair in `release_integration`.

1. Upgrade `tests/lsp_model_shortcut_smoke.lua` so the acceptance path enters insert
   mode, feeds real keystrokes, waits on semantic native-completion state such as
   `pumvisible()`/`complete_info()`, selects a native popup row, and observes the live
   buffer edit. Use bounded `vim.wait` predicates and deterministic state assertions; do
   not use timing sleeps.
2. Through that real frontend path, cover bare `**` in catalog order, canonical-prefix
   and short-hint filtering, provider scope, and application of a chosen `%m:` edit.
   Start with an open alias popup after typing `*`, type the second `*` without
   resetting the buffer and prove it becomes the concrete-model popup, then backspace
   without resetting and prove it returns to aliases. Invoke manual completion on an
   existing `**` query and prove the real popup and accepted edit, not a separately
   synthesized conversion.
3. Low-level synchronous LSP assertions may remain as server-wire diagnostics, but
   private `_convert_results` and direct `apply_text_edits` calls must not stand in for
   the required native lifecycle/acceptance proof. If the real test exposes a plugin
   defect, make the smallest Lua client correction needed without moving filtering or
   edit planning out of Rust. Update the README smoke description only if the executable
   steps or support claims change.
4. Run the focused Neovim Lua/config/smoke suite and the upgraded model shortcut smoke
   with `SASE_XPROMPT_LSP_CMD` pointing to the installed matching binary. From SASE,
   rerun the model completion binding/LSP/ACE tests and the focused model-completion PNG
   module, then run `just check` and the landing `just check-full` through
   `/sase_monitor`. Inspect and report any unrelated current failures rather than
   accepting unrelated PNG drift.

## Completion criteria

This remaining plan is complete when the declared and locked minimum SASE core release
retains `provider_display` and every other structured completion field through the real
binding, the installed binding and LSP match that pin, and the Neovim smoke proves the
live alias-to-model-to-alias transitions, manual popup, and accepted edits through
Neovim's native completion frontend. The focused cross-repository checks must pass.

Do not close `sase-yw.3` or `sase-yw`, run their close-time Symvision cleanup, or mark
their linked plan files done inside these phases. The child epic's `parent_bead` link is
the handoff back to the waiting `sase-yw.3` land agent, which owns those landing steps
after this plan finishes.
