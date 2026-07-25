---
tier: tale
title: Default @medium_phase_worker to @default@high and render alias effort overlays
goal: 'The implicit @medium_phase_worker alias defaults to "@default@high" — the model
  @default resolves to, run at high reasoning effort — the alias-reference-with-effort
  form resolves and validates correctly everywhere, and the ace Models panel makes
  that form legible on the alias row and in its description strip.

  '
create_time: 2026-07-25 09:59:00
status: done
---

- **PROMPT:** [202607/prompts/medium_phase_worker_default_alias.md](prompts/medium_phase_worker_default_alias.md)

# Plan: Default `@medium_phase_worker` to `@default@high`

## Problem and behavioral contract

`medium_phase_worker` is the only size role whose implicit default is a hard-pinned concrete target:
`src/sase/llm_provider/model_alias_policy.py` sets `MEDIUM_PHASE_WORKER_MODEL_ALIAS_DEFAULT = "codex/gpt-5.6-sol"` with
a `@high` suffix and registers it in `IMPLICIT_ALIAS_TARGETS`. Every other size role points at another alias
(`@cheaper`, `@cheap`, `@smart`, `@smartest`), so it tracks the user's provider choice; medium instead forces one
vendor's model on every user, including users who have no Codex CLI installed.

Change the implicit default to `@default@high`: use whatever model `@default` resolves to, but override the reasoning
effort to `high`. On the current default-provider setup (`@default` → `codex/gpt-5.6-sol`) the effective medium model is
byte-for-byte what it is today; on any other default provider, medium now follows that provider instead of forcing
Codex.

The alias grammar already permits this. `split_model_effort` peels a known trailing `@<level>` off _any_ model string,
including `@default@high` (the Rust mirror in `sase-core` has a `split_model_effort("@default@medium")` test), and
`_resolve_model_alias_result` already threads an alias-borne effort through indirection with outermost-wins precedence.
Three gaps make the form a second-class citizen today, and this plan closes them:

1. **The fallback map cannot express it.** `ROLE_ALIAS_FALLBACKS` values are bare `@<alias>` references, and
   `implicit_model_alias_fallback` returns the name with the `@` stripped and nothing else, so an effort overlay would
   leak into the alias name (verified: after adding `"@default@high"` to the map,
   `implicit_model_alias_fallback("medium_phase_worker")` returns `"default@high"`).
2. **The Models panel cannot show it.** Verified by patching `IMPLICIT_ALIAS_TARGETS["medium_phase_worker"]` to
   `"@default@high"`: the row's state tag renders a bare `implicit` — no `→ @default`, no effort provenance — and the
   description strip says only `effort: high`, with nothing tying that effort to the reference that produced it. The
   panel would advertise the new default as if it were a hard pin.
3. **`sase doctor` rejects it.** `check_config_model_aliases` strips only the leading `@` before looking the referenced
   alias up, so a user who copies the new default into `llm_provider.model_aliases.builtin.medium_phase_worker` gets a
   false warning. Verified verbatim:
   `model_aliases.builtin.medium_phase_worker -> @default@high references unknown alias '@default@high'`.

Contract for the finished work:

- `@medium_phase_worker` resolves to `@default`'s provider/model at effort `high` and moves whenever `@default` moves,
  including when `default` is user-configured.
- Effort precedence is unchanged: an outer effort wins, so `@medium_phase_worker@low` resolves at `low`.
- Nested-`@default` semantics are unchanged: an active _temporary_ override on `@default` does not move
  `@medium_phase_worker`, exactly as it does not move `@coder` or `@smart` today. Do not "fix" this.
- An explicit configured value for `medium_phase_worker` still wins over the implicit default, at every existing
  precedence layer (launch override, temporary override, config, implicit).
- Any alias — builtin or custom, implicit or configured — may use the `@<alias>@<effort>` form and gets the same
  resolution, validation, and panel rendering. Nothing about this is medium-specific except which default ships.

### Rust core boundary

No `sase-core` change. Confirmed against the linked checkout: the Rust core mirrors only the effort _vocabulary_ and the
`split_model_effort` rule (`crates/sase_core/src/effort.rs`, which already covers `@default@medium`), and it has no
`model_aliases` config validation and no alias-resolution model at all — alias policy, resolution, and presentation are
Python-resident. Phase-size routing in `crates/sase_core/src/plan/validate.rs` names `@medium_phase_worker` but says
nothing about its target, so it stays as-is.

## Teach the policy layer alias references that carry effort

In `src/sase/llm_provider/model_alias_policy.py`:

- Add `MEDIUM_PHASE_WORKER_MODEL_ALIAS_EFFORT = "high"` and redefine `MEDIUM_PHASE_WORKER_MODEL_ALIAS_DEFAULT` as
  `f"@{DEFAULT_MODEL_ALIAS_NAME}@{MEDIUM_PHASE_WORKER_MODEL_ALIAS_EFFORT}"`. Compose it from the existing constants
  rather than writing the literal `"@default@high"`, so the two names stay the single source of truth.
- Move the `medium_phase_worker` entry out of `IMPLICIT_ALIAS_TARGETS` and into `ROLE_ALIAS_FALLBACKS`, keyed to that
  constant. That map is the right home: `IMPLICIT_ALIAS_TARGETS` is documented as the concrete/selector defaults for
  aliases that _do not_ alias a role, and `@default@high` does alias a role. It also restores the invariant that
  `IMPLICIT_ALIAS_TARGETS` values never contain `@<alias>` references, which keeps the selector-parsing paths that read
  that map simple.
- Update the module docstring and the `ROLE_ALIAS_FALLBACKS` comment: a role fallback is an `@<alias>` reference that
  may carry a trailing `@<effort>`, meaning "that alias's model, at this effort". Note that the effort is an overlay,
  not a new target, and that an outer effort still wins.
- Leave `ROLE_ALIAS_DESCRIPTIONS[medium_phase_worker]` alone — the role's purpose has not changed.

In `src/sase/llm_provider/model_alias_config.py`:

- Fix `implicit_model_alias_fallback` to split a trailing effort off before stripping the `@`, so it keeps returning a
  bare alias name (`"default"`) and never `"default@high"`. Every existing caller depends on that contract.
- Add `implicit_model_alias_fallback_reference(name)` returning the raw reference (`"@default@high"`) and
  `implicit_model_alias_fallback_effort(name)` returning just the overlay (`"high"`). Both are thin readers over the
  same policy lookup, including the `<provider>_coder` → `@coder` case, which has no overlay and must keep returning
  `None` for the effort.
- Re-export all three from the `src/sase/llm_provider/config.py` façade next to the existing
  `implicit_model_alias_fallback` export, because resolution reaches them through the `config.` monkeypatch seam and
  `alias_view` imports from `.config`.

In `src/sase/llm_provider/model_alias_resolution.py`:

- In `_resolve_model_alias_result`, the fallback branch currently rebuilds the reference as `f"@{fallback}"`, which
  would discard the overlay. Switch it to `config.implicit_model_alias_fallback_reference(bare)` and assign that string
  directly to `current`. The loop's existing `split_model_effort` call and its `if effort is None` guard then apply the
  overlay with the correct outermost-wins precedence — no new effort plumbing.
- Make the same substitution in `validate_model_alias_selector_value`, where the fallback target is rebuilt the same
  way. Behavior is identical today (validation splits effort anyway), but it keeps one source of truth for "what does
  this implicit alias point at".
- Keep the `bare in seen` cycle guard on the fallback branch exactly as-is.

## Surface the reference and its effort in the Models panel

In `src/sase/llm_provider/alias_view.py`, add one derived `AliasView` property, `reference_effort`: the effort overlay
carried by this row's _immediate_ `@alias` edge. For a configured row, take it from `configured_value` via
`normalize_model_alias_reference`, and return `None` when that helper reports no alias — a concrete pin such as
`claude/opus@high` carries an effort, but not a _reference_ effort, and conflating the two would mislabel every pinned
row. For an unconfigured row, return `implicit_model_alias_fallback_effort(self.name)`. Document it in the `AliasView`
docstring alongside `reference_provider`/`reference_model`.

Leave `references` (configured-only, name-only) and `implicit_fallback` (bare name) unchanged: the state tag and
`model_picker_rows._alias_dependencies` both already read them, and keeping their contracts stable is what lets the rest
of this change stay small.

In `src/sase/ace/tui/modals/models_panel_rendering.py`:

- Give `_append_reference` an optional effort argument and, when set, append it with the existing
  `_append_effort_suffix` helper so the reference reuses the panel's established ` @ <level>` idiom and its
  `_EFFORT_CONNECTIVE_STYLE`/`_EFFORT_LEVEL_STYLE` colors. The `@alias` token keeps `_REFERENCE_TAG_STYLE`, so the
  scan-the-state-column property that the reference accent was introduced for still holds.
- Pass `view.reference_effort` from both `state_tag` call sites — the configured-reference branch and the
  implicit-fallback branch — so a user who writes `medium_phase_worker: "@default@high"` explicitly sees the same thing
  as the shipped default. The medium row becomes:

  ```
  role          medium_phase_worker    CODEX(gpt-5.6-sol) @ high     implicit → @default @ high
  ```

  Width stays inside the panel budget: gutter 2 + kind 13 + 1 + name 22 + 1 + badge (≤32, ~25 here) + gap 3 + state 26 ≈
  96 of the ~110-cell budget, and the existing whole-row ellipsis remains the narrow-viewport safety net. Do not widen
  `PROVIDER_MODEL_CELL_MAX` or the modal.

- In `description_text_for_view`, the `elif view.effort:` branch currently renders
  `effort: high · no default configured`, which states the level without saying where it came from. When there is no
  active override and `view.reference_effort` is set, render the literal syntax between the level and the default
  comparison:

  ```
  effort: high (via @default@high) · overrides default medium
  ```

  Compose the `@<name>@<effort>` token from `view.references or view.implicit_fallback` plus `view.reference_effort`
  rather than echoing `configured_value`, so user whitespace or casing cannot leak into the strip. Skip the clause
  entirely when an override is active — the override already owns the state tag, and `view.effort` then describes the
  override, not the reference.

Leave the badge column, `BucketView.model_counts`, the pool/fallback member lines, and the override chip untouched.

## Accept the form in doctor and config surfaces

In `src/sase/doctor/checks_config_model_aliases.py`, split the trailing effort off an `@<alias>` target before resolving
the referenced name, so `@default@high` validates as `@default`. Keep both existing arms intact: a retired name still
gets its `REMOVED_IMPLICIT_ALIAS_GUIDANCE` message, and a genuinely unknown name still reports, but now names the alias
it actually parsed (`@nope`, not `@nope@high`). Apply the same split to the `REMOVED_IMPLICIT_ALIAS_GUIDANCE` lookup so
`@phase_worker@high` is still recognized as stale rather than merely unknown.

In `src/sase/default_config.yml`, change the commented example to `medium_phase_worker: "@default@high"` with a comment
noting it follows `@default` at high effort, and extend the alias-value grammar comment above the example so the
`@<alias>` reference bullet says the reference may carry a trailing `@<effort>` that overrides the referenced alias's
effort. Keep the block's `keep-sorted`-safe shape.

In `src/sase/config/sase.schema.json`, extend the `model_aliases.builtin` description the same way: the
`'@<alias>' references` clause should state that a reference may carry a trailing `@<effort>`. The existing
trailing-effort sentence currently reads as scoped to selector members only.

## Documentation

Update every statement that names the medium default as a concrete Codex model, and document the new panel rendering:

- `docs/llms.md`: the builtin-alias example (~570), the implicit-defaults table row for `@medium_phase_worker` (~695),
  the size-routing prose (~708, ~923, ~953), and the two config examples (~724, ~947 — retarget the "medium bead phases
  run on Codex" example so it still demonstrates a concrete override without implying it is the default).
- `docs/configuration.md`: the builtin example (~802), the phase-routing paragraph (~850), and the alias-value grammar
  paragraph (~769) so it documents `@<alias>@<effort>`.
- `docs/ace.md`: the Models-panel state-tag list (~1467) gains the `implicit → @<fallback> @ <effort>` form, and the
  override-interaction paragraph (~1620) drops the concrete medium model in favor of `@default@high`.
- `docs/beads.md` (~429) and `docs/sdd.md` (~219): phase-routing sentences.
- `docs/xprompt.md`: the alias-override examples (~1141) and the size table (~1155). Leave the `%m:` grammar example at
  ~1357 alone — it demonstrates an explicit provider/model, which is still valid.

Do not touch `sase/memory/*.md`, `AGENTS.md`, or the generated provider instruction shims; no memory file records the
medium default, and editing them needs explicit user permission that this task does not carry.

## Regression coverage

- `tests/llm_provider/test_config_role_aliases.py`: `implicit_model_alias_fallback("medium_phase_worker")` is now
  `"default"` (it currently asserts `None`), the constant is `"@default@high"`, the new effort accessor returns
  `"high"`, the new reference accessor returns `"@default@high"`, and a `<provider>_coder` alias still reports a
  `"coder"` fallback with `None` effort.
- Alias resolution (extend `tests/llm_provider/test_config_alias_resolution.py` or the nearest existing home):
  `@medium_phase_worker` resolves to `@default`'s provider/model with effort `high`; it follows a user-configured
  `default`; `@medium_phase_worker@low` resolves at `low`; a configured `medium_phase_worker` still wins over the
  implicit default; and an active temporary `@default` override leaves `@medium_phase_worker` on the configured default.
- `tests/llm_provider/test_alias_view.py`: the medium row is still `kind == "role"` and unconfigured, now has
  `implicit_fallback == "default"` and `reference_effort == "high"`, and role ordering is unchanged. Add a case proving
  a configured concrete pin with an effort suffix (`claude/opus@high`) reports `reference_effort is None`.
- `tests/test_models_panel_alias_rendering.py`: `test_state_tag_implicit_size_phase_worker` now expects
  `"implicit → @default @ high"`; add a configured-`@default@high` state-tag case, a description-strip case for the
  `(via @default@high)` clause against a configured and an unconfigured default effort, and a case proving the clause is
  suppressed while an override is active.
- `tests/test_model_picker_aliases.py`: the medium row now has a dependency edge to `default`, so a persistent edit of
  `@default` must offer `@medium_phase_worker` as `would create a cycle`. This is a real gap closed by the move; assert
  it rather than working around it.
- `tests/doctor/test_checks_config_model_aliases.py`: a configured `medium_phase_worker: "@default@high"` produces no
  problem; `"@nope@high"` still reports an unknown-alias problem naming `@nope`; `"@phase_worker@high"` still gets the
  retired-alias guidance.
- `tests/test_config_schema_models.py`: the builtin-alias fixture (~68) should exercise `"@default@high"`.
- `tests/_models_panel_helpers.py`: retarget the `medium_phase_worker` row in `make_phase_worker_bucket_views` (~229)
  onto the same provider/model as the helper's `default` row with `effort="high"`, and fix any assertion in the bucket,
  navigation, and override-flow panel tests that this shifts.
- Visual: in `tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py`, change the `medium_phase_worker` fixture
  (~135) from `codex/gpt-5.6-sol` to the fixture's own `@default` model (`claude/claude-fable-4-10`), keeping
  `effort="high"`, so the golden demonstrates the new form instead of preserving the retired pin. Run the dedicated
  visual suite, inspect the actual/expected/diff artifacts under `.pytest_cache/sase-visual/`, accept only the
  intentional models-panel goldens with `--sase-update-visual-snapshots`, then rerun the visual suite normally.

## Verification

Run `just install` first — workspace virtualenvs are ephemeral and this repo requires it before other recipes. Then run
the focused model-alias, alias-view, Models-panel rendering, model-picker, doctor, and config-schema tests; then
`just test-visual` for the golden refresh described above; then `just check`. Because file changes land in this repo,
`just check` is mandatory before reporting completion.

Finish by grepping `src/`, `tests/`, and `docs/` for `gpt-5.6-sol@high` and for prose that still calls the medium phase
default a fixed Codex model, and re-run any focused test touched by that cleanup.
