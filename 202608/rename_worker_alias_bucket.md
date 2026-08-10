---
tier: tale
size: medium
title: Rename the phase_worker alias bucket and its size aliases to worker
goal:
  The built-in model-alias bucket is named `worker`, its five members are
  `@xsmall_worker`, `@small_worker`, `@medium_worker`, `@large_worker`, and
  `@xlarge_worker`, every runtime, config, schema, doc, memory, and Rust plan-schema
  surface uses the new names, and `sase doctor` gives actionable migration guidance for
  the retired `phase_worker` spellings.
proposed_by: bbugyi200.athena.sase-il.land.w1
create_time: 2026-08-10 11:02:37
status: wip
---

# Plan: Rename the phase_worker alias bucket and its size aliases to worker

## Context and invariants

SASE ships one built-in Models-panel bucket, `phase_worker`, holding five implicit role
aliases: `xsmall_phase_worker`, `small_phase_worker`, `medium_phase_worker`,
`large_phase_worker`, and `xlarge_phase_worker`. This plan renames the bucket to
`worker` and drops `_phase` from each member name, yielding `<size>_worker`.

Invariants this rename must preserve:

- **Resolution behavior is unchanged.** `xsmall_worker` still falls back to `@cheaper`,
  `small_worker` to `@cheap`, `large_worker` to `@smart`, and `xlarge_worker` to
  `@smartest`; `medium_worker` keeps its independent concrete target. Descriptions,
  fallback-graph validation, role ordering, and bucket-coalescing behavior are
  unchanged.
- **Only the five bucket members are renamed.** `default`, `epic_lander`,
  `big_epic_lander`, `smart`, `smartest`, `cheap`, `cheaper`, and `cheapest` keep their
  names, and `epic_lander` / `big_epic_lander` stay outside the bucket.
- **Alias names and bucket names remain separate namespaces.** `worker` is already a
  _retired implicit alias_ name carried in `REMOVED_IMPLICIT_ALIAS_GUIDANCE`. The new
  `worker` bucket must not be conflated with it: keep the retired-`@worker` guidance and
  reword it so a reader can tell the retired alias from the new bucket. Likewise
  `phase_worker` stays a retired builtin alias key in `_RETIRED_BUILTIN_ALIAS_NAMES`.
- **This is a breaking config change**, so it lands with a `feat!:` commit and a
  `BREAKING CHANGE:` footer.

Confirmed during planning: the chezmoi-managed `home/dot_config/sase/sase.yml` and
`sase_athena.yml` configure no `*_phase_worker` builtin overrides and no `phase_worker`
bucket metadata, so no live user configuration breaks. The doctor guidance below exists
for durability and for other installs, not to repair this machine.

Explicit decision to confirm at approval: the shipped bucket description string stays
`"Size-specific phase-agent aliases."`. Only the identifier changes; the aliases still
serve phase agents. Say so at approval if the description should be reworded too.

## Implementation

1. Rename the alias names and their name constants in the model-alias policy layer.
   - `src/sase/llm_provider/model_alias_defaults.yml`: rename the five `aliases` keys to
     `<size>_worker`, leaving each entry's `fallback`/`target`/`description` untouched,
     and update the `@medium_phase_worker` mention in the file header comment.
   - `src/sase/llm_provider/model_alias_policy.py`: rename
     `XSMALL_PHASE_WORKER_MODEL_ALIAS_NAME` through
     `XLARGE_PHASE_WORKER_MODEL_ALIAS_NAME` to `<SIZE>_WORKER_MODEL_ALIAS_NAME` with
     `<size>_worker` values, update the module-header comment block that documents the
     implicit alias set, and keep `_ROLE_ALIAS_NAME_CONSTANTS` in the same order as the
     YAML entries — the loader compares the constant name set against the YAML key set
     and raises an installation-defect error on any mismatch.
   - `src/sase/llm_provider/config.py`: update the five compatibility re-exports.
   - `src/sase/llm_provider/__init__.py`: update the bucket-constant imports and
     `__all__` entries.

2. Rename the built-in bucket in the alias view.
   - `src/sase/llm_provider/alias_view.py`: `PHASE_WORKER_BUCKET_NAME` becomes
     `WORKER_BUCKET_NAME = "worker"` and `PHASE_WORKER_BUCKET_DESCRIPTION` becomes
     `WORKER_BUCKET_DESCRIPTION`. Update `_ROLE_ALIAS_ORDER`, the
     `_BUILTIN_BUCKET_SPECS` fixed-member tuple, the `builtin_bucket_for_alias` lookup,
     and the `build_models_panel_rows` docstring. `BUILTIN_MODEL_ALIAS_BUCKET_NAMES`
     becomes `frozenset({"worker"})`.

3. Update the size-to-alias routing map and the remaining runtime call sites.
   - `src/sase/bead/work.py`: the five imports and the `PhaseSize`-to-alias mapping.
   - `src/sase/xprompt/model_completion.py`: the five imports and the completion
     ordering tuple.
   - `src/sase/xprompt/_directive_values.py`: the `@medium_phase_worker` example in the
     directive-value help text.
   - `src/sase/ace/tui/modals/approve_options_modal.py`: the medium-worker constant
     import and its use in the default follow-up label.
   - `src/sase/ace/tui/modals/models_panel_edit_helpers.py` and
     `src/sase/ace/tui/widgets/alias_overrides_indicator.py`: docstring examples and
     prose naming `medium_phase_worker` / `<size>_phase_worker`.
   - `src/sase/axe/run_agent_exec_plan_accept.py`: the `%model:@medium_phase_worker`
     docstring example.

4. Give `sase doctor` actionable migration guidance for the retired spellings.
   - `src/sase/doctor/checks_config_common.py`: add the five `<size>_phase_worker` names
     to `REMOVED_IMPLICIT_ALIAS_GUIDANCE`, each pointing at its `<size>_worker`
     replacement, and reword the existing `worker`, `coder`, and `phase_worker` entries
     that currently say "phase-worker alias" or `@medium_phase_worker`.
   - `src/sase/doctor/checks_config_model_aliases.py`: add the five old names to
     `_RETIRED_BUILTIN_ALIAS_NAMES` so their stale targets are not double-reported, and
     emit a focused per-key warning when `model_aliases.builtin.<size>_phase_worker` is
     configured, telling the user to move the target to
     `llm_provider.model_aliases.builtin.<size>_worker`. Update the `worker_models`,
     `coder`, provider-coder, and `phase_worker` messages that name
     `medium_phase_worker`, and keep reading the shipped medium default from
     `implicit_alias_targets()` rather than hardcoding a model string.
   - Give stale `model_aliases.buckets.phase_worker` metadata a targeted message naming
     the `worker` rename instead of falling through to the generic "has metadata but no
     custom aliases reference this bucket" warning.
   - Update the module docstring's summary of what the check reports.

5. Update shipped configuration and the config schema.
   - `src/sase/default_config.yml`: the `@<size>_phase_worker` prose in the
     `model_aliases` comment block, the five commented `builtin` examples, the "ACE
     always supplies `phase_worker`" sentence, and the commented
     `buckets: phase_worker:` example.
   - `src/sase/config/sase.schema.json`: the `builtin` description's alias list and its
     `<size>_phase_worker` tale-routing sentence, the per-alias `bucket` property
     description, and the `buckets` map description.

6. Update the size memory template and regenerate the memory notes. **This step needs
   explicit user permission that this plan does not grant.**
   - `src/sase/main/init_memory/templates/memory-sase-sizes.template.md` names
     `@<size>_phase_worker` twice. `sase/memory/sase_sizes.md` is generated from that
     packaged template, and `just check` runs `sase validate`, which fails on
     memory-init drift — so the template edit and the regeneration are inseparable.
   - Per this repo's memory gotcha, approving a plan is _not_ permission to edit
     `sase/memory/*.md`, `AGENTS.md`, or the provider shims. Before touching them the
     implementing agent must ask the user directly with `/sase_questions` for permission
     to update `sase/memory/sase_sizes.md` and run `sase memory init` (which also
     refreshes `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`, and the
     memory README) — unless the user already granted it in that conversation.
   - If permission is withheld, stop and report: the rename cannot pass `just check`
     while the generated size memory still advertises the old alias names.

7. Update the documentation set.
   - `docs/llms.md`: the builtin alias table and its descriptions, the bucket section
     that explains what `phase_worker` folds together, the `model_aliases.builtin` /
     `buckets` YAML examples, the temporary-override and effort-write examples, the
     Models-panel walkthrough, and the migration callout about retired names.
   - `docs/configuration.md`: the alias list, the YAML examples, the bucket section, the
     size-routing paragraphs, and the retired-alias callout — keeping the historical
     `@worker` / `@other` / `@phase_worker` retirement notes readable now that `worker`
     also names a bucket.
   - `docs/ace.md`: the Models-panel alias list, the built-in bucket description, the
     filter example, and the override/effort walkthrough steps that open `phase_worker`.
   - `docs/beads.md` and `docs/sdd.md`: the size-to-alias routing paragraphs.
   - `docs/xprompt.md`: the `%model(...)` per-alias override examples.
   - Extend the existing retired-name notes with `<size>_phase_worker` ->
     `<size>_worker` so upgraders can find the mapping.
   - Do **not** hand-edit `CHANGELOG.md`; release-please generates it. Land the work as
     a `feat!:` commit whose body carries a `BREAKING CHANGE:` footer naming the old and
     new bucket and alias names.

8. Update tests and visual fixtures.
   - Roughly 62 test files carry the old names. Update shared fixtures and helpers first
     (`tests/_model_alias_defaults_fixture.py`, `tests/_models_panel_helpers.py`,
     `tests/_model_picker_modal_helpers.py`,
     `tests/ace/tui/visual/_ace_models_panel_png_snapshot_fixtures.py`), then the
     `tests/llm_provider/` alias-policy/resolution/override/panel-row suites, the
     Models-panel navigation/rendering/edit/bucket/override suites, the model picker,
     completion, directive-extraction, launch-preview, config-schema, and
     `tests/test_bead/` suites.
   - `tests/doctor/test_checks_config_model_aliases.py`: keep the existing retired-name
     coverage and add cases for a configured
     `model_aliases.builtin.medium_phase_worker`, an alias value referencing
     `@medium_phase_worker`, and stale `model_aliases.buckets.phase_worker` metadata.
   - Rename the visual tests `test_models_panel_phase_worker_drilled_in_png_snapshot`
     and `test_models_panel_phase_worker_override_drilled_in_png_snapshot` plus their
     snapshot IDs to the `worker` spelling, `git mv` the two matching goldens under
     `tests/ace/tui/visual/snapshots/png/`, and regenerate every Models-panel,
     completion, and override-pill golden whose rendered text contains an alias or
     bucket name.
   - Leave identity and history strings alone: agent names, archived plan/bead data, and
     anything under the gitignored `sase/repos/` sidecar checkouts.

9. Rename the alias names in the Rust plan-schema description (separate repo).
   - `crates/sase_core/src/plan/validate.rs` defines `PHASE_SIZE_DESCRIPTION`, which
     names all five `@<size>_phase_worker` aliases and is user-visible through
     `sase plan validate --explain` and `--explain --json`. Open the repo with
     `/sase_repo open sase-core`, rename the five aliases in that string, and update the
     matching `assert_eq!(phase_size.description, ...)` expectation in the same file.
   - Land it as its own conventional commit/PR in `sase-core`. release-plz publishes the
     resulting `sase-core-rs` patch release, which already satisfies this repo's
     `sase-core-rs>=0.23.0,<0.24.0` floor in `pyproject.toml`, so no floor bump is
     needed. Until that release reaches the workspace, `--explain` still prints the old
     alias names; do not add a Python-side workaround for the gap.

## Verification

1. Run `just install` first — the workspace directory is ephemeral and its virtualenv
   may be stale.
2. While iterating, run the focused suites: `tests/llm_provider/`,
   `tests/doctor/test_checks_config_model_aliases.py`, the Models-panel and model-picker
   suites, `tests/test_xprompt_model_completion.py`,
   `tests/test_config_schema_models.py`, and `tests/test_bead/`.
3. Run `sase doctor -C config.model_aliases` against a scratch config that sets
   `model_aliases.builtin.medium_phase_worker`, an `@medium_phase_worker` reference, and
   `model_aliases.buckets.phase_worker` to confirm each warning is present and reads
   correctly.
4. Search the tree for `phase_worker` and `PHASE_WORKER` across `src`, `tests`, `docs`,
   `sase/memory`, and `.github`. Every surviving hit must be deliberate: retired-name
   migration guidance, the doctor tests that cover it, or a documentation upgrade note.
5. Run `just check-full`, required because this change touches the broadening set (alias
   registry, config schema, TUI surfaces, docs, generated memory, and visual fixtures).
   For PNG failures, inspect the actual/expected/diff artifacts in
   `.pytest_cache/sase-visual/` before accepting anything, then rerun
   `just test-visual`.
6. In the `sase-core` checkout, run `cargo fmt --all -- --check`,
   `cargo clippy --workspace --all-targets -- -D warnings`, and
   `cargo test -p sase_core`.
7. Finish with `git diff --check` and a review of `git status` in both repos for stray
   generated or sidecar changes, and confirm the commit message carries the
   `BREAKING CHANGE:` footer.
