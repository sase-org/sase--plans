---
tier: epic
title: Model override/edit picker alias ordering and effort-aware selections
goal: 'Model overrides and persistent alias edits present builtin models before all
  aliases, collect a defaulted reasoning-effort choice, and preserve explicit model
  or alias effort suffixes through resolution, storage, display, and launch.

  '
phases:
- id: alias-effort-semantics
  title: Preserve effort through model references and overrides
  depends_on: []
  size: medium
  description: '"Preserve effort through model references and overrides" section:
    complete alias@effort normalization, precedence, temporary override persistence,
    launch threading, Rust/Python parity, and focused tests.

    '
- id: model-picker-effort-flow
  title: Reuse the effort chooser in model override/edit flows
  depends_on:
  - alias-effort-semantics
  size: medium
  description: '"Reuse the effort chooser in model override/edit flows" section: reorder
    the model catalog, generalize the existing effort picker, and wire effort-aware
    listed and Custom selections through override and edit.

    '
- id: model-picker-effort-verification
  title: Visual integration and full validation
  depends_on:
  - alias-effort-semantics
  - model-picker-effort-flow
  size: small
  description: '"Visual integration and full validation" section: update and inspect
    ACE visual coverage, exercise responsive navigation, and run Rust and Python project
    checks to completion.

    '
create_time: 2026-07-24 16:42:30
status: done
bead_id: sase-8z
---

- **PROMPT:** [202607/prompts/model_override_effort.md](prompts/model_override_effort.md)

# Model Override/Edit Picker: Alias Ordering and Effort-Aware Selections

## Goal

Make the Models panel's override and persistent-edit workflows present the builtin provider/model catalog before the
complete model-alias catalog, ask for reasoning effort after a concrete model or alias is chosen, default that chooser
to the configured `llm_provider.default_effort`, and preserve effort suffixes through custom input, alias resolution,
temporary override state, display, and launch.

The semantic result of `@default@medium`, when `@default` targets `codex/gpt-5.6-sol`, is the structured
provider/model/effort result `codex/gpt-5.6-sol@medium`. An outer suffix must take precedence over effort already
carried by the alias target. Unknown trailing `@token` text must remain part of a model identifier, matching the
existing known-effort-only split rule.

## Current behavior and constraints

- `model_picker_rows.build_model_rows()` currently prepends alias rows before provider/model rows whenever the Models
  panel supplies alias context.
- The Models panel already has `DefaultEffortLevelModal`, generated from `EFFORT_LEVELS_ORDERED`, but it is specialized
  for editing or temporarily overriding the global default and has no highlighted/default-confirmed selection.
- `split_model_effort()` in Python and Rust already recognizes the trailing known-effort token in `@alias@effort`; the
  Python alias resolver also already gives the outer suffix precedence while walking configured alias chains. Missing
  regression coverage and callers that discard the structured effort make this behavior incomplete in practice.
- Free-form panel safety checks currently treat `default@medium` as the alias name, so an otherwise valid custom
  `@default@medium` is rejected.
- `TemporaryLLMOverride` records only provider/model/raw input. The write path calls the two-field resolver, so a custom
  or picker-generated effort suffix is stripped during resolution and is not available when the override later shapes a
  launch.
- Alias display snapshots explicitly clear effort whenever a temporary model override is active, and the primary-default
  launch fast path similarly omits override-borne effort.
- Keep model/alias resolution and state I/O off Textual render and keystroke paths. Continue constructing the picker
  from the Models panel's cached `AliasView` snapshot.
- The shared grammar must remain in parity with `sase-core`; use the linked repository opened through
  `sase repo open sase-core`.

## Phase 1: Preserve effort through model references and overrides

Slug: `alias-effort-semantics`

Dependencies: none

### Work

1. Add explicit Python and Rust regression coverage for the shared known-effort-only split grammar:
   `@default@medium -> (@default, medium)`, unknown suffixes remain untouched, and a single leading token such as
   `@high` is still an alias/model token rather than an effort-only value.
2. Extend model-alias resolution tests around `resolve_model_alias_with_effort()` /
   `resolve_model_provider_with_effort()` to cover:
   - direct `@alias@effort`;
   - aliases whose configured target has no effort;
   - aliases whose configured target already carries a different effort, with the outer suffix winning;
   - chained aliases and selector members without changing their existing selector-consumption behavior. Keep the
     established structured return contract (clean target plus effort) instead of reattaching suffixes inside
     provider/model fields.
3. Introduce one suffix-aware alias-reference normalization helper for panel and display callers. Use it in
   `alias_reference_rejection()` and `AliasView.references` so safety validation and dependency/cycle detection operate
   on the clean alias name while retaining the selected effort for the eventual value. Do not interpret an arbitrary
   unknown suffix as effort.
4. Extend `TemporaryLLMOverride` and its JSON serialization/deserialization with an optional canonical effort field.
   Resolve writes with `resolve_model_provider_with_effort()`, retain the original `raw_model`, validate any persisted
   effort, and keep old v1/v2 records without the field readable as `effort=None`. Preserve eager snapshot semantics for
   alias targets rather than re-resolving `raw_model` on every launch.
5. Thread stored override effort through every effective path: non-default alias resolution, the no-directive
   primary-default launch path, `resolve_effective_default_provider_model_with_effort()`, `AliasView` construction, and
   override notifications/indicators where the selected effort is user-visible. An explicit outer `@alias@effort`
   continues to win over stored/configured alias effort; otherwise override-borne effort wins over the
   machine-wide/configured default under the existing precedence rules.
6. Update focused tests for temporary override migration/round trips, configured and overridden alias views, default and
   non-default launches, and indicator/render labels. Document any user-visible state or resolution contract change in
   `docs/llms.md` / `docs/ace.md`.

### Acceptance

- Resolving `@default@medium` yields provider `codex`, model `gpt-5.6-sol`, and effort `medium` on a configuration where
  `@default` targets that Codex model.
- If `@focused` targets `claude/opus@high`, `@focused@medium` resolves with effort `medium`, while plain `@focused`
  retains `high`.
- Setting either a default or non-default temporary model override from `model@medium` or `@alias@medium` survives a
  state round trip and affects the next launch at `medium`.
- Old override state without an effort field behaves exactly as before.

## Phase 2: Reuse the effort chooser in model override/edit flows

Slug: `model-picker-effort-flow`

Dependencies: `alias-effort-semantics`

### Work

1. Reorder `build_model_rows()` so the optional follow-up-default row remains first, all builtin provider/model groups
   appear next, the complete flat alias snapshot appears below them, and `Custom...` remains last. Adjust separators,
   filtering, highlight restoration, and jump-hint tests so provider and alias groups remain coherent and disabled
   self/cycle rows remain searchable but unselectable.
2. Refactor the existing `DefaultEffortLevelModal` presentation into a reusable effort-level chooser rather than
   creating a second effort UI. Preserve the global default-effort edit/override flows while allowing a model-selection
   context with:
   - the same canonical ordered vocabulary and visual tones;
   - `j`/`k` (and arrow) navigation plus Enter confirmation and existing numeric shortcuts;
   - an optional provider-default/no-suffix choice when no configured default exists;
   - an initial highlighted choice set specifically from the configured `llm_provider.default_effort`, not a temporary
     global effort override. Guard any programmatic OptionList highlight assignment so Textual highlight echoes cannot
     cause cursor jumps.
3. Add pending model/effort state to both Models-panel workflows. After a listed builtin model or alias is selected,
   open the reusable effort chooser, append the confirmed known effort as the trailing suffix, then continue to the
   existing destination:
   - temporary override -> duration/until picker -> background state write;
   - persistent edit -> validation/diff preview -> optional commit offer. Cancellation at the effort step performs no
     write and does not advance to duration or preview.
4. Make `Custom...` explicitly advertise the supported suffix grammar. For a single custom model/provider-model/alias:
   - preserve and honor a valid explicitly typed suffix;
   - apply suffix-aware alias rejection/cycle checks to the clean alias;
   - when no suffix is supplied, open the same effort chooser using the configured default selection. Persistent
     selector expressions keep their existing per-member effort grammar and go directly through selector validation; do
     not reinterpret the last member's suffix as one effort for the whole pool/fallback.
5. Ensure values handed to persistent preview and temporary override writes are canonical raw strings such as
   `gpt-5.6-sol@xhigh` or `@default@medium`. Update success messages to include the effort without changing the
   established duration wording.
6. Add interaction tests for listed model and alias selection, default highlight/Enter behavior, no-config provider
   default, cancellation at every new transition, manual concrete and alias suffixes, alias-cycle rejection with
   suffixes, custom values without suffixes, and selector preservation.

### Acceptance

- The picker shown in the supplied screenshot lists builtin provider/model groups before the complete alias section,
  with `Custom...` last.
- Selecting either a builtin model or a selectable alias immediately opens the reused effort chooser; pressing Enter
  without moving chooses the configured default effort.
- A manually entered `codex/gpt-5.6-sol@medium` reaches the override write or edit preview with `@medium` intact.
- A manually entered `@default@medium` passes alias safety validation, resolves to the configured default model with
  `medium`, and overrides any alias-defined effort.

## Phase 3: Visual integration and full validation

Slug: `model-picker-effort-verification`

Dependencies: `alias-effort-semantics`, `model-picker-effort-flow`

### Work

1. Add or update ACE PNG scenarios for the reordered alias-enabled model picker, the new effort step after builtin-model
   selection, the effort step after alias selection, and a narrow supported terminal size. Inspect actual, expected,
   diff, and source artifacts before accepting intentional golden changes.
2. Exercise filtering and rapid navigation in the picker/effort modal without adding config reads, resolver calls, disk
   I/O, or asynchronous work to keystroke/render paths. Retain cached `AliasView` construction and selective modal
   transitions.
3. From the main `sase` workspace, run `just install` before checks, then run focused Python/TUI tests,
   `just test-visual`, `just rust-check` for the linked `sase-core` changes, and the mandatory `just check`. Resolve
   every failure and rerun until clean.
4. Review the final diff across both repositories for accidental version edits, generated instruction/memory changes,
   stale snapshots, or unrelated modifications.

### Acceptance

- Focused behavior tests, Rust parity tests, visual snapshots, `just rust-check`, and `just check` all pass.
- The supported narrow layout keeps titles, choices, and footer inside the viewport, and navigation remains immediate.
- No SASE memory files or generated agent-instruction shims are changed.

## Risks and mitigations

- **Ambiguous `@` in model IDs:** split only canonical trailing effort levels; preserve all unknown/internal `@` tokens
  verbatim.
- **Selector suffix misclassification:** distinguish a single model reference from a pool/fallback expression before
  applying whole-value effort handling.
- **Effort precedence regressions:** test explicit outer suffix, configured alias/member effort, temporary
  model-override effort, global temporary effort, configured default effort, and provider default in descending order.
- **State compatibility:** make the stored field optional and cover v1/v2 records that predate it.
- **TUI cursor jumps/performance:** keep catalogs immutable during interaction, guard programmatic highlights, and avoid
  I/O/resolution in handlers and rendering.

## Non-goals

- Changing the canonical effort vocabulary or provider-specific supported levels.
- Changing load-balancing/fallback selection or cursor-consumption semantics.
- Re-resolving stored temporary alias targets dynamically after configuration changes.
- Modifying SASE memory files or generated provider instruction shims.
