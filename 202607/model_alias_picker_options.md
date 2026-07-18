---
tier: tale
title: Model-alias targets in the Models-panel picker
goal: 'Models-panel Edit and Override flows offer safe, searchable @alias targets
  as first-class, polished choices while preserving concrete-model behavior and accurately
  communicating what each choice will do.

  '
create_time: 2026-07-18 07:43:07
status: done
prompt: 202607/prompts/model_alias_picker_options.md
---

# Plan: Model-alias targets in the Models-panel picker

## Context and outcome

The Models panel already displays every implicit and configured model alias, and the configuration layer already accepts
alias-to-alias values such as `big_epic_lander: "@coder"`. The Edit and Override actions nevertheless open the shared
`ModelPickerModal` with only provider-grouped concrete models. As a result, filtering for `@coder` produces “No matching
models” and forces the user through `Custom...`, even though SASE knows the alias, its description, its chain, and its
effective provider/model.

Make aliases feel native in the two Models-panel picker flows. A user should be able to press `e` or `o`, see a compact
alias section before the provider model sections, filter by `@coder` or `coder`, understand the alias’s current target,
and select it with the same navigation and jump controls as a model. Persistent Edit stores the selected `@alias`
reference; temporary Override keeps its existing behavior of resolving the selected alias to a concrete provider/model
when the override is written. This plan does not add alias creation/deletion, change runtime alias precedence, or expose
aliases in unrelated callers such as the approval-options model picker.

## Product and visual design

- Add an opt-in alias group at the top of the picker, separated from the provider groups with a styled `ALIASES` header
  and count. Render each choice with the `@name` as the primary token, a subtle arrow, and the provider-themed effective
  model badge already familiar from the Models panel. Keep columns calm and aligned, truncate long values deliberately,
  and use muted secondary text for provenance/override context rather than crowding the primary token.
- Reuse the established semantic colors: alias cyan for the token, provider palettes for effective targets, and
  restrained warning/error styling for an unavailable choice. Preserve the modal’s centered proportions and stable
  footer, and verify both the normal 120-column presentation and a narrower viewport so long custom aliases and
  provider-local model paths degrade with ellipsis instead of wrapping or shifting the layout.
- Make filtering span `@name`, bare name, alias kind, configured description, and the displayed provider/model. A query
  that matches only an alias should show that alias and its group without also claiming that there are no matches. When
  neither aliases nor models match, use a context-aware empty message while retaining `Custom...` as the escape hatch.
  Update the input placeholder when alias choices are enabled so the capability is discoverable.
- Preserve current keyboard behavior: the filter retains focus, `j`/`k` and arrows navigate all selectable rows, Enter
  returns the exact raw value (`@coder`), apostrophe jump hints follow the visible selectable order, and Escape first
  clears a filter and then cancels. Disabled safety rows remain visible and searchable with a concise reason so an
  expected alias does not simply disappear.

## Alias catalog and safety contract

- Extend the picker with a small, typed, opt-in alias-selection context instead of global behavior or several loosely
  related booleans. The context carries a snapshot of the Models panel’s already-loaded `AliasView` catalog, the alias
  being changed, and whether the operation is a persistent reference or a temporary concrete override. Existing callers
  that omit the context must produce byte-for-byte-equivalent model/default/custom rows and return values.
- Build alias rows once when the modal opens, then perform filtering, rendering, selection, and jump-map construction
  entirely in memory. Do not read config, inspect the override state file, discover providers, or resolve aliases from
  input/highlight/key handlers. Reuse the panel snapshot (and, if the display layer needs an additional picker-facing
  field, compute it during the normal alias aggregation) so this feature cannot turn typing or navigation into a
  disk-scaled path.
- Display the target that the selected token actually represents, including the special `@default` rule: a nested
  explicit `@default` reference uses the configured/provider default and does not inherit the machine-wide default
  temporary override. Non-default aliases should show an active temporary override when that override would participate
  in resolution. Distinguish this current/effective hint from the stored value so the UI never implies that a persistent
  alias reference is copied into a concrete model.
- For persistent Edit, derive each alias’s immediate dependency from its configured `@reference` or implicit fallback
  and traverse the small alias graph with a bounded, visited-set algorithm. Disable the alias being edited, any
  candidate whose chain reaches it (which would introduce a direct or transitive cycle), and a candidate whose existing
  chain is cyclic. Label the reason (`current alias` or `would create a cycle`) in the row. Apply the same validation to
  an `@...` value entered through `Custom...`, including rejecting an unknown alias token, so the free-form path cannot
  bypass the picker’s guard. Concrete custom model strings remain unchanged and still flow to the Rust-backed
  preview/validation step.
- For temporary Override, allow other known aliases and disable only the target alias itself, which would be a confusing
  self-snapshot. Document and preserve the existing eager behavior: after duration selection, the worker resolves the
  raw `@alias` against fresh state, stores the resulting concrete provider/model together with the raw token, and
  reports the actual resolved model. No persistent alias graph is mutated, so persistent-cycle exclusions do not apply
  to this mode.

Keep row construction, dependency classification, matching, and Rich-text rendering as pure helpers with focused unit
coverage. The TUI mixins should only assemble the selection context, route the returned string into the existing
preview/duration workflows, and notify on a rejected free-form alias.

## Models-panel integration

- In both `ModelsPanelAliasEditMixin.action_edit` and `ModelsPanelOverrideMixin.action_override`, pass the panel’s flat
  alias-view snapshot plus the selected target alias into the picker using the appropriate operation semantics. Bucket
  membership is display-only; all bucket members remain individually addressable in the picker, in the deterministic
  alias order established by the alias data layer.
- Persistent selection of an alias should open the existing `AliasEditPreviewModal` directly with
  `ConfigEditOp.set_value("@alias")`, so the effective before/after view, source-preserving diff, chezmoi application,
  and commit/push offer remain the final confirmation path. There is no second free-form dialog for a first-class alias
  choice.
- Temporary selection of an alias should proceed directly to the existing duration and exact-time pickers. Keep
  cancellation/back behavior and worker error handling unchanged, and ensure the eventual state record retains the raw
  alias token while its provider/model fields contain the write-time resolution.
- Refreshing the Models panel after an edit/override continues to replace its alias snapshot and preserve the selected
  row. Do not add a parallel refresh path or synchronous work to an action handler.

## Documentation and compatibility

- Update the Models-panel documentation to mention the alias group, filtering, unavailable-cycle affordance, and the
  difference between a persistent dynamic alias reference and a temporary concrete snapshot. Add one persistent example
  (`@big_epic_lander` → `@coder`) and one temporary example using an alias.
- Keep provider short aliases visually/searchable only as they are today; they must not be confused with `@model_alias`
  choices or returned with an `@` prefix. Keep `Follow-up default`, all sentinels, concrete model IDs, provider
  grouping/order, and the behavior of non-Models-panel picker callers backward compatible.

## Verification

- Expand picker unit tests for opt-in construction, stable ordering, aligned styled rendering, exact `@alias` return
  values, filtering with and without `@`, description/effective-model search, context-aware empty states, jump hints,
  disabled-row navigation, and the default no-alias compatibility path.
- Cover the alias dependency classifier with direct self-reference, multi-hop-to-target, implicit fallback (including
  provider-coder and big-epic-lander chains), pre-existing cycle, safe sibling, and long-chain cases. Exercise the same
  guard through `Custom...` so unknown and unsafe references cannot reach the persistent preview.
- Add Models-panel flow tests proving Edit persists the raw `@alias`, Override carries it through duration selection and
  resolves it in the existing worker, cancellation remains a no-op, bucketed aliases are offered, and unrelated picker
  callers remain alias-free.
- Add a deterministic ACE PNG snapshot reached through the Models-panel Edit flow with the alias group visible and
  `@coder` filtered/highlighted. Inspect actual/expected/diff artifacts before accepting the intentional golden, and add
  a narrow styled geometry assertion for truncation, footer visibility, and zero row wrapping.
- Run `just install` before verification in the implementation workspace, then run the focused picker/Models-panel tests
  and `just test-visual`. Finish with the repository-required `just check`; investigate any visual or performance
  regression rather than weakening exact local snapshot comparisons.

## Acceptance criteria

- From any alias row in Models, both `e` and `o` expose first-class, searchable `@alias` choices; selecting `@coder` no
  longer requires `Custom...`.
- The picker communicates each choice’s accurate current target and operation semantics, renders cleanly at supported
  widths, and retains all existing keyboard and escape-hatch behavior.
- The UI cannot create a persistent direct/transitive alias cycle through either a listed choice or free-form `@alias`,
  and cannot create a misleading temporary self-snapshot.
- Persistent edits still use preview/confirm and store the alias reference; temporary overrides still store a concrete
  write-time resolution and leave persistent config untouched.
- Existing ModelPickerModal callers and concrete-model flows are unchanged, all targeted/visual tests pass, and
  `just check` succeeds.
