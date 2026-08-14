---
tier: epic
title: Full model-pool support in the Models panel alias editor
goal: 'Model alias definitions overridden from the ACE Models panel can specify round-robin
  pools and ordered fallback chains through a guided, keyboard-driven builder instead
  of hand-typed free text, the temporary-override path refuses selector expressions
  instead of silently persisting a corrupted single target, and the docs describe
  the shipped behavior.

  '
phases:
- id: override-selector-reject
  title: Reject selector expressions in the temporary-override path
  depends_on: []
  size: small
  description: 'override-selector-reject: detect selector syntax with parse_model_alias_selector
    in the panel''s Custom Override Model flow and refuse it with a message pointing
    at the persistent Edit key, instead of letting a pool string split on its first
    slash and persist a bogus single-target override.

    '
- id: selector-edit-plumbing
  title: Parse-based selector detection and prefilled custom input
  depends_on:
  - override-selector-reject
  size: small
  description: 'selector-edit-plumbing: add a shared TUI selector helper module over
    the existing llm_provider API, replace the substring selector sniffing in the
    alias Edit flow with real parsing plus per-member safety checks, and give the
    custom-model input an initial value so editing an existing selector no longer
    means retyping it.

    '
- id: selector-builder
  title: Guided pool and fallback builder modal
  depends_on:
  - selector-edit-plumbing
  size: medium
  description: 'selector-builder: add a SelectorBuilderModal that assembles selector
    members from the existing model picker and effort ladder with add, remove, reorder,
    per-member effort, and pool-versus-fallback toggling, gate member selection against
    nested selectors, and route a new picker entry row through it into the unchanged
    preview and write path.

    '
- id: selector-docs
  title: Documentation sync for selector authoring
  depends_on:
  - selector-builder
  size: small
  description: 'selector-docs: correct the ACE Models panel documentation that claims
    the custom input only accepts concrete model strings, document the builder row
    and its keys, state that temporary overrides refuse selectors, and cross-check
    the selector semantics and configuration references.'
proposed_by: bbugyi200.athena.014
create_time: 2026-08-14 10:49:15
status: wip
bead_id: sase-lz
---

- **PROMPT:** [prompts/202608/models_panel_pool_authoring.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/models_panel_pool_authoring.md)
- **BEAD:** [sase-lz](https://github.com/sase-org/sase--beads/blob/main/pages/sase-lz/README.md)

# Full model-pool support when overriding model alias definitions in the Models panel

## Problem

The ACE Models panel (`,m`) is the authoritative place to edit model alias definitions
(`docs/llms.md:1170`). Alias values may be selector expressions — `A | B`
(availability-filtered round-robin pool) or `A || B` (ordered fallback chain) — and
selectors are explicitly **config-only**: `%model` values, launch-scoped alias
overrides, and temporary overrides all remain single targets (`docs/llms.md:1140`).

Pools are therefore _only_ reachable through the panel's persistent **Edit** (`e`) path,
and today that path supports them barely and unevenly:

1. **Authoring is hand-typing only.** `e` opens `ModelPickerModal`, which is
   single-select. The only way to reach a selector is to choose `Custom...` and type the
   whole expression, including exact `provider/model` ids and per-member `@effort`, from
   memory. The picker's catalog — aliases plus provider-grouped models with provider
   badges and advisories — is unusable for building one.
2. **No prefill.** `CustomModelInputModal`
   (`src/sase/ace/tui/modals/custom_model_input_modal.py:29`) accepts no initial value,
   so changing one member of a three-member pool means retyping the entire expression.
3. **The effort ladder is skipped for selectors.**
   `ModelsPanelAliasEditMixin._on_edit_custom_picked`
   (`src/sase/ace/tui/modals/models_panel_alias_edit.py:127`) short-circuits on
   `"|" in raw_model` straight to the preview, so `DefaultEffortLevelModal` never runs
   and per-member effort must be typed by hand.
4. **Detection is a bare substring test.** That same `"|" in raw_model` branch also
   bypasses `alias_reference_rejection`, so member-level cycle/unknown-alias problems
   surface late, from `validate_model_alias_selector_value`, in a different message
   shape than the picker's own rejections.
5. **The temporary-override path silently corrupts a typed selector.**
   `ModelsPanelOverrideMixin._on_custom_picked`
   (`src/sase/ace/tui/modals/models_panel_override.py:151`) has no selector detection
   and never validates. A typed pool flows through `split_model_effort`, gets a
   _whole-string_ `@effort` appended in `_on_override_model_effort_picked`, then reaches
   `set_alias_override` → `resolve_model_provider_with_effort`
   (`src/sase/llm_provider/registry.py:382`), which splits on the first `/`. Verified in
   this repo:

   ```
   'claude/opus | codex/gpt-5.5'       -> ('claude', 'opus | codex/gpt-5.5', None)
   'claude/opus | codex/gpt-5.5@high'  -> ('claude', 'opus | codex/gpt-5.5', 'high')
   ```

   `TemporaryLLMOverride` (`src/sase/llm_provider/temporary_override_state.py:45`) is
   structurally single-target, so this persists a bogus override with a nonexistent
   model name and no error — the user gets a broken override instead of a clear "use
   `e`".

6. **Docs contradict the code.** `docs/ace.md:2682` says `Custom...` "remains available
   for concrete model strings", never mentioning that selectors are accepted at all.

What already works and must not regress: `validate_model_alias_selector_value`
(`src/sase/llm_provider/model_alias_resolution.py:400`) is a thorough authoritative
validator (empty members, cycles, unknown aliases, nested selectors); the write path via
`alias_model_edit_path` correctly targets `llm_provider.model_aliases.builtin.<alias>`
and `llm_provider.model_aliases.custom.<alias>.model`; and panel rendering already
displays pool chips and member lists
(`src/sase/ace/tui/modals/models_panel_rendering.py:170,300`).

## Goal

Make selector authoring a first-class, discoverable, keyboard-driven flow in the Models
panel Edit path; make the temporary-override path reject selectors with an actionable
message instead of corrupting them; and bring the docs in line.

## Design constraints

- **Rust core backend boundary.** Selector parsing, validation, and resolution already
  live in `src/sase/llm_provider` (`load_balancing.py`, `model_alias_resolution.py`,
  `model_alias_policy.py`) and behind the Rust-backed config planner. Every change in
  this epic is presentation-only Textual glue that _calls_ those APIs. Do **not**
  reimplement selector semantics in the TUI, and do not add a second validator.
  `validate_model_alias_selector_value` stays the single authority at the write
  boundary.
- Mixed `|` / `||` in one value stays invalid; nested selectors stay invalid; members
  follow the single-target grammar with an optional trailing `@effort`.
- Selectors remain config-only. Nothing in this epic makes a temporary override, a
  launch-scoped alias override, or a `%model` value selector-valued.
- The Models panel's in-panel keys (`e`, `o`, `x`, `r`, `Ctrl+E`, `Ctrl+R`) are
  hardcoded `BINDINGS` on the modal, not leader-keymap config; only the `,m` leader
  chord lives in `src/sase/default_config.yml`. If any phase nevertheless introduces a
  configurable binding, it must update `src/sase/default_config.yml`.
- **Memory files are out of scope.** No phase may touch `sase/memory/*.md`, `AGENTS.md`,
  or the generated provider instruction shims (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
  `QWEN.md`). Those require explicit user permission that this plan does not carry.
- Every phase runs `just install` first (workspaces are ephemeral), then `just check`
  before reporting done. The final phase additionally runs `just check-full` through
  `/sase_monitor`, never inline.

## Phases

### Phase `override-selector-reject`

**Size:** small **Depends on:** (none)

Stop the temporary-override path from silently persisting a corrupted selector.

Change `ModelsPanelOverrideMixin._on_custom_picked`
(`src/sase/ace/tui/modals/models_panel_override.py:151`) to detect selector syntax with
`parse_model_alias_selector` from `sase.llm_provider.load_balancing` before any other
handling:

- A `ModelAliasSelectorError` (for example a mixed `|`/`||` value) is reported verbatim
  via `self.notify(..., severity="warning")` and the flow returns without writing.
- A successfully parsed selector is also refused, with an actionable message naming the
  alternative — for example:
  `Pools and fallbacks are config-only; a temporary override on @<alias> takes a single target. Press e to set @<alias> to a selector.`
- Anything else keeps today's behavior exactly (effort ladder, then duration picker).

Deliberately unchanged: `_on_model_picked` receiving an `@alias` row for an alias that
_owns_ a selector. That is the documented snapshot semantic — the override bypasses the
expression for its lifetime (`docs/llms.md:1143`) — and must keep working.

**Tests:** extend `tests/test_models_panel_override_flows.py` with cases for a typed `|`
pool, a typed `||` fallback, and a mixed `|`/`||` value, asserting each notifies, writes
nothing, and leaves the panel screen focused. Add a regression case asserting an
`@alias` override on a pool-owning alias still snapshots a single concrete target.

**Done when:** a typed selector in Custom Override Model is refused with a message
pointing at `e`; no `~/.sase/llm_override.json` entry is written for it; `just check`
passes.

### Phase `selector-edit-plumbing`

**Size:** small **Depends on:** `override-selector-reject`

Replace substring sniffing with real parsing, share it, and make the free-text path
editable rather than retype-only.

1. Add `src/sase/ace/tui/modals/models_panel_selector.py` — thin presentation glue over
   the existing `llm_provider` API, no new semantics:
   - `parse_selector_for_display(value)` → wraps `parse_model_alias_selector` and turns
     `ModelAliasSelectorError` into a user-facing message, so both the Edit and Override
     paths share one detection point.
   - `member_rejection(context, member)` → applies the picker's existing
     `alias_reference_rejection` (`src/sase/ace/tui/modals/model_picker_rows.py`) to a
     single selector member.
   - `compose_selector(mode, members)` → delegates to `ModelAliasSelector.normalized` so
     the canonical `A | B` / `A || B` spelling has exactly one producer.
2. Refactor the Phase `override-selector-reject` guard onto `parse_selector_for_display`
   so there is a single detection helper in the TUI.
3. In `ModelsPanelAliasEditMixin._on_edit_custom_picked`
   (`src/sase/ace/tui/modals/models_panel_alias_edit.py:127`), replace
   `if "|" in raw_model:` with the parse-based check, and on a parsed selector run
   `member_rejection` for each member, notifying with the first rejection before falling
   through. `_open_model_edit_preview`'s `validate_model_alias_selector_value` call
   stays as the authoritative final gate — this only moves the picker's own safety
   feedback earlier and into a consistent message shape.
4. Give `CustomModelInputModal`
   (`src/sase/ace/tui/modals/custom_model_input_modal.py:29`) an `initial: str = ""`
   constructor argument that seeds the `SingleLineVimTextArea` and leaves the cursor at
   end of line. Have the Edit path pass the alias's current value
   (`AliasView.configured_value`, else `AliasView.implicit_value`) so editing an
   existing selector starts from that expression. The Override path keeps `initial=""` —
   its default is a fresh single target.

**Tests:** extend `tests/test_models_panel_edit.py` for parse-based detection (a mixed
`|`/`||` value still rejects with the parser's message; a member that references an
unknown or cycle-forming alias is rejected before the preview opens) and for the
prefill. Add coverage that `CustomModelInputModal(initial=...)` seeds the editor and
that an empty `initial` preserves today's placeholder behavior.

**Done when:** the Edit `Custom...` input opens prefilled with the alias's current
value; selector detection everywhere in the panel goes through the shared helper;
`just check` passes.

### Phase `selector-builder`

**Size:** medium **Depends on:** `selector-edit-plumbing`

Add the guided builder — the phase that actually makes pools first-class.

1. **New modal** `src/sase/ace/tui/modals/models_panel_selector_builder.py`:
   `SelectorBuilderModal(ModalScreen[str | None])`, constructed with the alias name, the
   alias's current value, the panel's `AliasSelectionContext`, the
   `EffectiveDefaultEffortSnapshot`, and the panel's `now`.
   - State: a `ModelAliasSelectorMode` (default `round_robin`) and an ordered member
     list, both seeded by parsing the current value. A single-target current value seeds
     a one-member list; an empty value seeds an empty list.
   - Rendering: a header naming the mode (`round-robin pool` / `ordered fallback`) and
     showing the live normalized expression from `compose_selector`; an `OptionList` of
     member rows with position, provider/model badge (reuse
     `provider_model_badge_markup` from `sase.ace.tui.provider_styles`), effort suffix,
     and availability marker; and a validation line re-driven by
     `validate_model_alias_selector_value` after every mutation.
   - Modal-local `BINDINGS`, matching the existing hardcoded panel style: `a` add
     member, `d` remove, `J`/`K` reorder, `E` set or clear that member's effort, `t`
     toggle `|` ↔ `||`, `enter` confirm, `esc` cancel. Confirm dismisses with the
     normalized expression string.
   - `a` pushes the **existing** `ModelPickerModal` with the panel's alias context and
     then, for a member without a typed `@effort`, the **existing**
     `DefaultEffortLevelModal` — no duplicated catalog or effort ladder.
   - Confirm is blocked while the member count is below two or the validator reports
     errors; the footer states which.
2. **Member-level safety in the picker.** Selector members may follow an alias chain but
   may not reach another pool or fallback. Extend `AliasSelectionOperation` in
   `src/sase/ace/tui/modals/model_picker_rows.py` with a `member` operation whose
   `_alias_disabled_reason` also disables alias rows that own a selector — derivable
   from `AliasView.selector_mode` — with a concise reason (`reaches a pool` /
   `reaches a fallback`). This surfaces up front what
   `validate_model_alias_selector_value` would otherwise only report at the preview.
3. **Entry point.** Add a `Pool / fallback...` row next to `Custom...` in
   `build_model_rows` behind a new `include_selector_option` flag and a
   `SELECTOR_SENTINEL`. Only the Edit path sets the flag; the Override picker must not
   show it, since selectors are config-only there.
4. **Wiring.** `ModelsPanelAliasEditMixin.action_edit` passes
   `include_selector_option=True`; `_on_edit_model_picked` handles `SELECTOR_SENTINEL`
   by pushing `SelectorBuilderModal` seeded from the highlighted view; its callback
   routes the returned expression to the unchanged `_open_model_edit_preview`
   (validation → preview → write → commit offer).
5. **Presentation.** Add the modal's container/row rules to
   `src/sase/ace/tui/styles.tcss` following the existing models-panel modal conventions,
   and surface the builder's keys wherever the panel's modal keys are already listed.

**Tests:**

- New `tests/test_models_panel_selector_builder.py`: seeding from an existing `|` pool,
  from an existing `||` fallback, from a single target, and from empty; add / remove /
  reorder; mode toggle; per-member effort set and clear; confirm emits the normalized
  expression; confirm blocked below two members and blocked on a validation error; a
  member that would reach a nested selector is rejected.
- Extend `tests/test_model_picker_aliases.py` for the new row, the
  `include_selector_option` gating (present for Edit, absent for Override), and the
  `member` operation's disabled reasons.
- Extend `tests/test_models_panel_edit.py` for the `SELECTOR_SENTINEL` route reaching
  `AliasEditPreviewModal` with the composed expression.
- Visual: add a golden for the builder (for example
  `models_panel_selector_builder_120x40.png`) to the models-panel edit snapshot suite
  (`tests/ace/tui/visual/test_ace_png_snapshots_models_panel_edit.py`), generated with
  `--sase-update-visual-snapshots`, then confirm with `just test-visual`. Re-check the
  existing `models_panel_alias_picker_*` goldens, which the new picker row will shift.

**Done when:** a user can press `e`, choose `Pool / fallback...`, assemble a
multi-member selector entirely from the picker with per-member effort, reorder it,
toggle pool vs fallback, and write it — without typing a `|` by hand; `just check` and
`just test-visual` pass.

### Phase `selector-docs`

**Size:** small **Depends on:** `selector-builder`

Bring the documentation in line with the shipped behavior.

- `docs/ace.md`, Models Panel section: correct the claim at `docs/ace.md:2682` that
  `Custom...` "remains available for concrete model strings" — it also accepts selector
  expressions, and `Custom...` now opens prefilled with the alias's current value.
  Document the `Pool / fallback...` builder row: its keys, seeding behavior, the
  two-member minimum, per-member effort, and pool-vs-fallback toggling. Update the `e`
  row of the panel's key table to mention selector authoring. In the _Temporary
  overrides_ subsection, state that Override refuses selector expressions and points at
  `e`.
- `docs/llms.md` (selector semantics, roughly lines 1120–1160): keep the "Selector
  expressions are config-only" statement authoritative, and add that the Models panel
  can now author selectors directly and that a temporary override rejects one outright
  rather than accepting it.
- `docs/configuration.md:1418`: cross-check the pool-row description against the new
  authoring flow and adjust only if it now reads as incomplete.
- Explicitly out of scope: `sase/memory/*.md`, `AGENTS.md`, and the generated provider
  instruction shims. If this work looks like it warrants a memory update, record a
  `PROPOSED FOLLOW-UP:` note on the phase bead instead of editing those files.

**Verification:** run `just check-full` through `/sase_monitor`
(`sase monitor start --command 'just check-full' …` with a `--next` action), never
inline — it routinely outruns a single agent turn.

**Done when:** the three doc files describe the shipped flow with no remaining
contradiction, and `just check-full` is green.

## Risks and notes

- The shared `ModelPickerModal` is used by callers beyond the Models panel. Both new
  picker knobs (`include_selector_option`, the `member` alias operation) must default to
  today's behavior so unrelated callers are untouched.
- PNG goldens are pixel-exact locally. The new picker row shifts existing models-panel
  picker snapshots; regenerate deliberately and review the diffs in
  `.pytest_cache/sase-visual/` rather than blanket-accepting.
- The two-member minimum is a builder-level affordance, not a new rule: a one-member
  "selector" is simply a single target, which the existing single-target path already
  handles.
- No Rust core changes are expected. If any phase finds that selector semantics
  genuinely need to move or change, that belongs in the sibling `sase-core` repo per the
  `rust_core_backend_boundary` memory and should be raised before proceeding.
