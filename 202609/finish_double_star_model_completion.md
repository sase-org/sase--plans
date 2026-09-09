---
tier: epic
title: Finish double-star model completion landing gaps
goal:
  Close the remaining safety, native-editor, lifecycle, and visual-proof gaps found
  while landing sase-yw without redoing the already-landed shortcut implementation.
parent_bead: sase-yw
phases:
  - id: core_safety
    title: Harden shared shortcut value validation
    description:
      "core_safety: reject catalog values that cannot safely become one inline model
      directive, add focused Rust/PyO3/LSP regressions, and land a verified sase-core
      revision for downstream integration."
    size: medium
    depends_on: []
  - id: surface_proof
    title: Complete ACE and Neovim integration proof
    description:
      "surface_proof: integrate the landed core revision, preserve advisory metadata and
      scoped highlighting in ACE, complete missing lifecycle and PNG coverage, exercise
      real Neovim native completion transitions, and run the combined landing gates."
    size: medium
    depends_on:
      - core_safety
proposed_by: bbugyi200.athena.sase-yw.land
bead_id: sase-yw.3
create_time: 2026-09-09 19:52:22
status: wip
---

- **PROMPT:**
  [prompts/202609/finish_double_star_model_completion.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/finish_double_star_model_completion.md)
- **PARENT:**
  [202609/double_star_model_completion.md](https://github.com/sase-org/sase--plans/blob/main/202609/double_star_model_completion.md)
- **BEAD:**
  [sase-yw.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-yw/sase-yw.3.md)

# Finish double-star model completion landing gaps

## Context

The parent epic `sase-yw` already landed the main implementation in three repositories:

- `sase-core` commit `d4d81b64d7a002f711a6645ebcea4a66b58160aa` adds the shared
  alias/model shortcut contract, PyO3 bindings, and xprompt LSP handler.
- the SASE repository commit `4b1e5f8eb40e5e9deeae9e129f40860367fefd6e` pins released
  core `0.32.55`, adds the ACE explicit-model menu, parity tests, documentation, and one
  light-theme PNG.
- `sase-nvim` commit `9858feae71821ffb6325ee72704f2b661d0d6ad0` adds documentation and a
  request-level smoke test.

The landing audit confirmed those commits and their source are present, both original
child beads are closed, and neither child contains a `PROPOSED FOLLOW-UP:` note. Changes
that landed in the SASE repository after this epic began were also reviewed: the SDD
clone-deadline fix, test-cost recalibration, Fleet envelope decoding, dispatch
machine-init split, and usage-probe drift fix do not conflict with the shortcut. The ACE
shortcut commit already descends from the relevant Fleet and test-cost changes; later
dispatch and usage commits touch disjoint code. The only core drift after the shortcut
commit is the `0.32.55` release, which is already pinned, and there is no post-shortcut
Neovim drift.

The epic is nevertheless not ready to close. The audit found these remaining
requirements from `plan:202609/double_star_model_completion.md`:

1. `canonical_model_value` in `crates/sase_core/src/editor/model_alias_shortcut.rs`
   rejects whitespace and aliases but does not reuse the existing control-character
   validation. A malicious or malformed catalog value containing a non-whitespace
   control character such as NUL can therefore be emitted into `%m:<value>`, contrary to
   the plan's one-safe-inline-directive rule.
2. ACE drops the catalog's `bucket`, `advisory_label`, and `advisory_severity` when
   constructing `ModelCompletionMetadata`. Provider display is recovered by parsing
   `description`, which becomes incorrect when that description carries an advisory. The
   explicit menu therefore does not retain advisory data as structured metadata or
   render it deliberately.
3. Scoped short-hint matches such as `**claude/fa` highlight the short hint but do not
   independently highlight the matched provider segment. The plan requires scoped
   provider/name match highlighting without inventing a canonical-name match.
4. The explicit-model PNG suite contains only the full light-theme menu. It still lacks
   the required dark, filtered short-hint/expansion, narrow long-or-scoped,
   stacked-pane, advisory-bearing, loading, and unavailable snapshots.
5. The Neovim smoke test calls `textDocument/completion` synchronously and applies
   `textEdit` directly. It does not pass rows through Neovim's native completion
   frontend or exercise bare `**`, the live `*` to `**` transition, backspace to `*`,
   and manual completion as required.
6. The ACE explicit-model suite proves the principal path but does not directly pin
   several model-specific transitions and stale-state cases: alias-empty/loading
   second-star takeover, same-kind selection retention, third-star and space dismissal,
   explicit unavailable/retry and stale-selected-value handling, scoped/advisory
   rendering, and warm-keystroke avoidance of builders/resolvers. Shared alias tests may
   be reused where the code path is truly identical; add focused explicit coverage where
   kind transitions or model-only rendering make behavior distinct.

Do not redesign the shortcut, add configuration, add a model inventory, or duplicate
Rust filtering/edit logic in Python or Lua. Preserve `%m:` and `*alias` behavior.

## Phase `core_safety`: Harden shared shortcut value validation

Work in the `sase-core` repository opened through `sase repo open sase-core` and follow
its `AGENTS.md`.

1. Tighten explicit-model selection validation in
   `crates/sase_core/src/editor/model_alias_shortcut.rs` so a selected catalog row is
   still required to be a current filtered concrete-model result and its canonical value
   is nonempty, not alias-shaped, whitespace-free, control-character-free, and
   representable as one unquoted inline `%m:` value. Reuse an existing core validator or
   extract a focused shared predicate instead of growing a second incompatible rule.
   Preserve valid provider-qualified and nested-slash values.
2. Add focused tests at the core boundary for NUL and other relevant controls,
   whitespace, alias/provider/stale selections, valid punctuation, provider-qualified
   values, mid-token edits, and unchanged alias-wrapper compatibility. Test the actual
   serialized PyO3 dict/list/None/error behavior for malformed catalogs and unsafe
   selected values.
3. Add or extend xprompt LSP service/stdio assertions proving unsafe catalog rows cannot
   produce completion edits and recognized safe/no-match contexts remain owned
   incomplete lists. Keep plain-text edits, the single `*` trigger character, catalog
   order, and provider-scope semantics unchanged.
4. Run the focused Rust tests and the repository's required `just check` /
   `./scripts/check.sh` with a Python >=3.12 interpreter. Finish through host-owned
   finalization and record the landed revision for the dependent phase. Do not edit
   release versions manually; release-plz owns them.

## Phase `surface_proof`: Complete ACE and Neovim integration proof

This phase depends on `core_safety`. Work primarily in the SASE repository and open
`sase-core` and `sase-nvim` through `/sase_repo` before reading or editing them. Read
`lint_and_test.md` and `tui_perf.md` through `/sase_memory_read` before editing.

### Core integration

1. Confirm the `core_safety` commit is on the opened core repository's landed branch and
   ratchet `sase-core-revision.txt` to a revision that includes it. Reconcile
   `pyproject.toml` and `uv.lock` with the published-wheel floor/window only when an
   appropriate release exists; never invent an unpublished version or leave installed
   environments on a floor missing the safety fix.
2. Build the real binding with `just install` against the pinned/opened core and install
   the matching LSP with `just rust-lsp-install`. Confirm `.venv/bin/sase-xprompt-lsp`
   and the imported `sase_core_rs` expose the expected implementation before parity
   tests.

### Structured presentation and lifecycle proof

3. Extend `ModelCompletionMetadata` and the shared candidate projection so `bucket`,
   `advisory_label`, and `advisory_severity` survive as structured fields. Render a
   concise advisory in the explicit-model row or selected preview using the established
   advisory glyph/style semantics. Do not parse rendered description text to recover
   provider or advisory identity, and do not regress ordinary `%m:` or alias rows.
4. Make scoped matching highlight the provider segment and the actually matched
   model-name or short-hint segment independently. For `**claude/fa` against
   `claude/claude-fable-5` with short hint `fable`, highlight `claude` and `fa` in the
   hint without falsely highlighting `fa` in the canonical model name. Keep Rich labels
   literal and cell-width truncation stable.
5. Extend focused ACE tests for the genuinely missing model-specific cases: second-star
   takeover when alias rows are empty or loading, `*`/`**`/`***` and backspace/space
   transitions, same-kind canonical selection retention and reset, Ctrl+L consumption,
   explicit unavailable/retry, stale selected values, focus/pane/cancellation and ABA
   worker states where distinct from the shared alias path, and repeated warm typing
   that never calls the static catalog builder or a provider resolver. Use controlled
   worker barriers/state injection, not timing sleeps. Keep all
   disk/config/plugin/provider work off the event loop and Textual serial pump.
6. Expand `tests/ace/tui/visual/test_ace_png_snapshots_model_completion.py` with
   explicit-model snapshots for full dark and light menus, a filtered short-hint match
   with exact `%m:` preview, a long provider-scoped model at 70x24, a stacked pane, an
   advisory-bearing model, and truthful loading and unavailable states. Reuse a compact
   parameterization where it keeps the suite maintainable. Generate intentional goldens
   only after inspecting actual, expected, and diff PNGs for contrast, legibility,
   truncation, title/subtitle truthfulness, and height.

### Native-editor proof

7. In `sase-nvim`, upgrade `tests/lsp_model_shortcut_smoke.lua` to exercise the real
   native completion conversion/frontend rather than treating raw LSP JSON and
   `vim.lsp.util.apply_text_edits` as sufficient. Cover bare `**`, canonical-prefix and
   short-hint filtering, provider scope, catalog order, applying the chosen edit, typing
   the second star from an active alias session, backspacing to `*`, and manual
   completion. Use the explicit installed LSP command supplied by the SASE phase. Keep
   filtering and edit planning server-owned; make a Lua client fix only if the native
   test demonstrates one is required. Update the concise README smoke recipe if its
   actual steps change.

### Verification

8. Run focused core-binding/ACE/LSP parity tests, the upgraded Neovim smoke with the
   just-installed binary, `just test-visual` for the model-completion visual module, and
   each changed linked repository's required checks. In the SASE repository run
   `just check`, then run the epic landing `just check-full` through `/sase_monitor`
   with the required `TESTING`/`TESTED` status pair. Re-run installed-binary parity
   after dependency reconciliation. Report current failures; do not waive them based on
   the earlier child notes.

## Completion criteria

The remaining plan is complete when unsafe catalog values cannot create directives;
valid nested/provider-qualified values still work; ACE carries and renders advisory
metadata without parsing descriptions; scoped matches highlight only their true
provider/name-or-hint portions; the required explicit-model visual states have inspected
goldens; Neovim exercises actual native completion transitions and edits; and the pinned
installed binding/LSP plus all focused, visual, repository, and full landing gates pass.
At that point the parent `sase-yw` land agent can resume its epic-symbol cleanup, close,
symvision check, linked-plan status update, and parent-bead traversal.
