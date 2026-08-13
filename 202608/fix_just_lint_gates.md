---
tier: tale
title: Fix the two red just lint gates (terminology audit and symvision)
goal:
  just lint passes every stage end to end, and task beads sase-kq and sase-kt are closed
  as obsolete.
size: small
proposed_by: bbugyi200.athena.za
create_time: 2026-08-13 09:05:59
status: wip
---

# Fix `just lint`: classify the legacy `changespec` guard-provider fixtures and privatize `tribe_config_key`

## Problem

`just lint` is deterministically red on current master at two independent stages.

**Stage `_lint-patch-stitch-terminology`** fails with exactly three unclassified
`changespec` tokens:

```
main:tests/test_validate_sase_core_rs_tool.py:430: changespec: defect (unclassified) "changespec",
main:tests/test_validate_sase_core_rs_tool.py:504: changespec: defect (unclassified) "changespec",
main:tools/validate_sase_core_rs:606: changespec: defect (unclassified) "changespec": {"name_prefix": "x"},
```

**Stage `_lint-symvision`** fails with one unused public symbol:

```
Unused public functions/classes. ...
  tribe_config_key in src/sase/ace/tui/models/tribe_display.py
```

The symvision failure is masked today because the terminology stage runs first and
aborts the recipe; it was confirmed independently by running `just _lint-symvision`
directly. Every other `lint` stage was also run individually and passes: `keep-sorted`,
`ruff`, `mypy` plus `typecheck_extensionless_tools`, `pyscripts-260801`,
`check_test_wait_helpers`, `validate_changelog`, and `toobig`. These two are therefore
the complete set of `just lint` failures.

Both failures are already tracked as ready task beads: `sase-kq` (terminology audit,
`+8` corroborations) and `sase-kt` (symvision).

## Root Cause

### 1. Legacy `changespec` guard-provider mirrors are unclassified

The bundled config schema advertises `changespec` as a **legacy alias** of the `patch`
chop guard provider — see `docs/axe.md:500` and `docs/configuration.md:2117`
("`changespec` is a legacy alias"), and the four `inhibit_if` enums in
`src/sase/config/sase.schema.json`.

Those schema enum lines are classified by the dedicated
`_is_config_schema_legacy_chop_provider_enum` predicate in
`src/sase/patch_stitch_audit.py`. But two _mirrors_ of that same enum carry no
classification signal at all:

- `tools/validate_sase_core_rs:606` — the `AXE_CHOP_GUARD_PROVIDER_PAYLOADS` table,
  which deliberately holds one minimal `inhibit_if` payload per advertised provider so
  that adding a provider to the schema without extending the table fails
  `_validate_axe_chop_guard_providers` instead of silently skipping it.
- `tests/test_validate_sase_core_rs_tool.py:430` and `:504` — the `_write_guard_schema`
  default provider list and the schema-drift test's provider list, which reproduce the
  advertised enum independently of the tool's table.

With no signal, `_classify_candidate` falls through every rule to
`("defect", "unclassified")`.

The audit's designed extension point for exactly this case is an **adjacent declaring
comment**: `_iter_candidates` passes a ±1-line context window into
`_classify_candidate`, and `_declares_compatibility_boundary` matches
`_DECLARED_COMPATIBILITY_MARKERS` (`legacy`, `compat`, `alias`, `deprecated`,
`retained`, ...) case-insensitively against that context. This contract is already
pinned by two existing tests, `test_classifier_accepts_explicit_compatibility_comment`
and `test_classifier_accepts_test_tree_declared_legacy_alias` in
`tests/test_patch_stitch_terminology_audit.py`.

### 2. `tribe_config_key` has no non-test consumer

`tribe_config_key` (`src/sase/ace/tui/models/tribe_display.py:108`) is called only from
its own defining file, at lines 115 (`tribe_display_for`) and 130
(`tribe_identity_colors`). Its only other references are
`tests/ace/tui/models/test_tribe_display.py:111-112` and its `__all__` entry at
line 212. Per `sase/memory/symvision.md`, test references can never keep a public symbol
alive, so step 2 of that note's decision hierarchy applies: make it private. It must not
be deleted — both in-file callers are live and depend on its `None` → `"default"`
mapping.

## Approach

### Step 1 — Classify the tool's legacy guard-provider payload

In `tools/validate_sase_core_rs`, add one comment line directly above the
`"changespec": {"name_prefix": "x"},` entry in `AXE_CHOP_GUARD_PROVIDER_PAYLOADS`,
declaring it the retained legacy alias of the `patch` provider (for example:
`# Legacy alias retained for the ``patch`` provider.`).

Word the comment so it contains a `_DECLARED_COMPATIBILITY_MARKERS` term but **not** an
audited token itself, so the fix does not introduce a new candidate line. The token is
already visible on the line immediately below, so the comment stays readable.

Expected classification: `legacy-compatibility-boundary` via the
`legacy_compatibility_boundary` rule (`tools/` paths are not matched by the earlier
`audit_contract`, `stable_documentation_reference`, or `compatibility_test_or_fixture`
predicates).

### Step 2 — Classify the two test fixture provider lists

Apply the same treatment at both sites in `tests/test_validate_sase_core_rs_tool.py`:
the `_write_guard_schema` default list (~line 430) and the
`test_validate_axe_chop_guard_providers_fails_on_schema_drift` list (~line 504).

Expected classification: `legacy-data-test-fixture` via the
`compatibility_test_or_fixture` rule, which is evaluated before
`legacy_compatibility_boundary` for `tests/` paths.

Keep both literal lists intact and duplicated. The drift guard's value comes from the
fixture mirroring the schema _independently_ of the tool's payload table, so do not
factor them into a shared constant and do not derive either list from
`AXE_CHOP_GUARD_PROVIDER_PAYLOADS`.

### Step 3 — Privatize `tribe_config_key`

In `src/sase/ace/tui/models/tribe_display.py`:

- Rename `tribe_config_key` → `_tribe_config_key` (line 108).
- Update both in-file callers (lines 115 and 130).
- Drop the `"tribe_config_key"` entry from `__all__` (line 212), preserving the
  remaining sorted order.

In `tests/ace/tui/models/test_tribe_display.py`, update
`test_tribe_config_key_maps_none_to_default` (lines 110-112) to call
`display._tribe_config_key`, leaving its assertions unchanged. The test imports the
module (`import sase.ace.tui.models.tribe_display as display`) rather than importing the
symbol, and test paths satisfy symvision's private-symbol guard, so this does not trade
one finding for another — confirm that by re-running the linter rather than by
inspection.

## Verification

Run in this order and require each to pass:

1. `just install` first. Ephemeral `sase_<N>` workspaces drift; this tree also needed
   its `sase-core` linked checkout refreshed (via the `/sase_repo` skill) to satisfy the
   `sase-core-rs>=0.26.6,<0.27.0` floor before `just install` would succeed.
2. `just _lint-patch-stitch-terminology` — expect no `defects:` block, and expect the
   retained-token summary to still report non-zero counts for the three `required=True`
   rules (`legacy_compatibility_boundary`, `legacy_serialized_data`,
   `stable_public_path`) so no `stale_rules` error is raised.
3. `just _lint-symvision` — expect clean output.
4. `just lint` — expect every stage to pass end to end. This is the acceptance
   criterion.
5. Focused suites named by the two beads:
   `pytest tests/test_validate_sase_core_rs_tool.py tests/test_patch_stitch_terminology_audit.py tests/ace/tui/models/test_tribe_display.py`
6. `just check`, as the repo requires after any file change.

## Bead Closeout

Once `just lint` passes end to end, close both obsoleted task beads with a note naming
the verified command output:

```bash
sase bead close sase-kq --note "<what was verified>"
sase bead close sase-kt --note "<what was verified>"
```

Both are `ready` task beads with no descendants, so neither close cascades. Do not open
new beads for this work.

## Non-Goals And Risks

- **Not** renaming or removing the schema's `changespec` guard provider. It is a
  documented legacy alias in the public config surface; retiring it is a separate
  compatibility decision.
- **Not** adding new rules or predicates to `src/sase/patch_stitch_audit.py`. The
  declaring-comment mechanism is the sanctioned path and is already covered by tests;
  adding a hard-coded path predicate would grow the audit's special-case surface for no
  gain.
- **Not** deleting `tribe_config_key` or its test.
- Main risk: a comment whose wording drifts outside the ±1-line context window or omits
  a marker term silently leaves the defect in place. Mitigation is step 2 of
  Verification — the audit's own output, not inspection, is the proof.
