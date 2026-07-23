---
tier: tale
title: Scope machine_name to the machine identity overlay
goal: Valid SASE config fragments no longer receive false missing-machine-name schema
  diagnostics while declared machine identities remain constrained and initialization-owned.
create_time: 2026-07-23 09:56:08
status: wip
---

- **PROMPT:** [202607/prompts/machine_name_schema_scope.md](prompts/machine_name_schema_scope.md)

# Scope `machine_name` to the machine identity overlay

## Problem and confirmed root cause

The public schema at `src/sase/config/sase.schema.json` is exposed through `sase path config-schema` and is
intentionally associated by editor integrations with every SASE config fragment: the user base `sase.yml`, ordinary
`sase_*.yml` overlays, machine-specific overlays, project-local `sase/sase.yml` files, legacy project `sase.yml` files,
and the bundled `src/sase/default_config.yml`. Commit `770ad01ab1` added `machine_name` as the schema's sole required
top-level property. Consequently, yaml-language-server reports `Missing property "machine_name"` for every valid
fragment that does not own the machine identity.

That per-document requirement conflicts with the configuration model:

- `sase config init` creates or selects one `sase_<machine>.yml` overlay containing `machine_name` and writes the
  machine-local selector at `~/.sase/machine_name`.
- Ordinary overlays and base/project config files are partial layers and must remain valid without duplicating the
  identity.
- The runtime intentionally supports legacy or not-yet-initialized installations with no machine identity; features that
  need it already call `require_machine_name()`, while the init planner and doctor report missing or mismatched
  selector/overlay state.
- JSON Schema validates one document and has no context about the other files in a merge stack or the local selector, so
  the shared per-file schema cannot express “one selected overlay in this installation must contain the field.”

## Intended contract

Treat `machine_name` as optional in each independently validated config document but strictly validate it whenever it is
present. Machine identity remains operationally required for initialized machine-hood features and remains owned by the
selected machine overlay. The existing initializer, selector-aware loader, `get_machine_name()` agreement check,
`require_machine_name()` failure, and doctor/init checks continue to enforce that installation-level workflow.

Do not introduce a second filename-specific schema. Ordinary and machine-specific overlays share the same `sase_*.yml`
naming space, and a filename alone does not determine whether an overlay is machine-specific; the presence of the
property does. A second schema or new editor mapping would either misclassify ordinary overlays or require machine-local
dynamic editor configuration.

## Implementation

1. Correct the shared schema in `src/sase/config/sase.schema.json`.
   - Remove the root-level `required: ["machine_name"]`.
   - Keep the `machine_name` property, string type, and `^[a-z_]+$` pattern so malformed declared identities remain
     schema errors.
   - Clarify its description: the field marks a machine-specific overlay and is established by `sase config init`, but
     is optional in base, ordinary-overlay, default, and project-local fragments.
   - Leave `additionalProperties: false` and all unrelated field contracts unchanged.

2. Make the schema regression suite validate realistic partial files instead of injecting an identity into every
   fixture.
   - In `tests/test_config_schema.py`, validate the bundled default directly, replace the current “required
     machine_name” test with coverage proving the field is optional, and retain accepted/rejected name coverage for
     configurations that declare it.
   - Remove `with_machine_name()` from `tests/_config_schema_helpers.py`.
   - Update all users in `tests/test_config_schema.py`, `tests/test_config_schema_agent_experience.py`,
     `tests/test_config_schema_automation.py`, `tests/test_config_schema_models.py`,
     `tests/test_config_schema_repositories.py`, `tests/test_config_schema_tribes.py`, and
     `tests/ace/tui/test_commits_config.py` to validate their config fragments directly. This turns the existing broad
     schema suite into protection against accidentally reintroducing any root-required field while preserving every
     nested valid/invalid assertion.
   - Add or retain an explicit representative assertion for a non-machine fragment like the screenshot's user-base
     config so the LSP failure mode is obvious from the test name and failure.

3. Align `docs/configuration.md` with the layer-aware contract.
   - State that `machine_name` is required in the selected machine identity overlay created or selected by
     `sase config init`, not in every file accepted by the public schema.
   - Update the field table and surrounding prose to call the per-document schema property optional while explaining
     that present values are pattern-validated.
   - Preserve the existing selector behavior, foreign-overlay filtering, no-default compatibility, and actionable
     initialization guidance.

## Boundaries and compatibility

- Do not change overlay selection, config merge ordering, cache invalidation, initialization, or machine-name accessors
  in `src/sase/config/core.py` or `src/sase/main/config_init_handler.py`; those behaviors already implement the
  installation-level invariant.
- Do not change the Rust config backend. It generically validates the effective merged candidate against the supplied
  schema, and an absent identity is already a supported runtime state.
- Do not change `sase-nvim` or personal Neovim configuration. Their broad schema association is correct because all
  matched files use the same set of allowable fields; correcting the schema automatically removes the false diagnostics.
- Do not add `machine_name` to base, ordinary-overlay, project-local, plugin-default, or bundled-default files as a
  workaround.

## Validation

1. Run `just install` before repository checks because workspace dependencies may be stale.
2. Run the focused schema suites covering the files changed above, including the ACE commits-config schema tests.
3. Confirm there are no remaining `with_machine_name` references and directly validate all of these against the public
   schema:
   - an empty/partial config without `machine_name`;
   - the bundled `src/sase/default_config.yml`;
   - a machine overlay with a valid lowercase/underscore identity;
   - a declared identity with invalid uppercase, hyphen, digit, or empty values, which must still fail.
4. Run `just check`, fix any failures, and rerun until clean as required for changes in this repository.
5. Inspect the final diff and re-run the focused schema tests after any cleanup to ensure the fix did not weaken nested
   schema constraints or alter runtime machine selection.

## Acceptance criteria

- yaml-language-server no longer reports a missing `machine_name` for valid base, ordinary-overlay, default, or
  project-local SASE config files using the shared public schema.
- A machine-specific overlay may declare exactly the same `machine_name` field as before, and invalid declared names
  still fail schema validation.
- Initialization and doctor checks remain the authority for establishing one selected machine identity overlay, while
  legacy/uninitialized runtime behavior remains unchanged.
- Schema tests no longer hide per-file validity by adding a synthetic machine identity to every fixture.
- User documentation clearly distinguishes per-document schema optionality from the initialized machine identity
  requirement.
- Focused tests and the full `just check` suite pass.
