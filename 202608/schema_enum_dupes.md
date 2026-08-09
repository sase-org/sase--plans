---
tier: tale
title: Repair duplicate enum values that invalidate sase.schema.json
goal:
  yaml-language-server accepts src/sase/config/sase.schema.json again, restoring LSP
  validation and completion for sase.yml files, and a regression test fails if any
  bundled JSON schema regains a duplicate enum value.
proposed_by: bbugyi200.athena.wd
create_time: 2026-08-09 07:53:13
status: wip
---

# Plan: Repair duplicate enum values that invalidate sase.schema.json

## Problem

Editing `sase/sase.yml` in Neovim surfaces a diagnostic on line 1:

```
Schema '/home/bryan/projects/github/sase-org/sase/src/sase/config/sase.schema.json'
is not valid:
/definitions/axeChop/properties/inhibit_if/oneOf/0/items/properties/provider/enum :
must NOT have duplicate items (items ## 0 and 1 are identical)
```

This is not a problem with the YAML file being edited. The bundled schema itself is
rejected, so yaml-language-server discards it wholesale: **all** validation, completion,
and hover for every `sase.yml` file is silently lost, and the single error is anchored
to line 1 (which is why it renders on the `commit_hooks:` line).

## Root cause

`src/sase/config/sase.schema.json` declares the chop guard provider enum four times, and
every occurrence lists `patch` twice:

| Line | Location                                                                                                                                                   |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 174  | `definitions.axeChop.properties.inhibit_if.oneOf[0].items.properties.provider.enum`                                                                        |
| 184  | `definitions.axeChop.properties.inhibit_if.oneOf[1].propertyNames.enum`                                                                                    |
| 1198 | `properties.axe.properties.lumberjacks.additionalProperties.properties.chops.items.oneOf[1].properties.inhibit_if.oneOf[0].items.properties.provider.enum` |
| 1212 | the matching `propertyNames.enum` under the same `chops` subtree                                                                                           |

Each reads `["patch", "patch", "agent_hood", "agent_clan"]`.

The duplicate was introduced by the Patch terminology migration, in two steps:

1. `c7026e50e` (`feat(tui): rename ACE patch surface`) deliberately widened the enum to
   `["patch", "changespec", "agent_hood", "agent_clan"]` so both the new and the legacy
   provider name validated.
2. `77d18c3e1` (`feat(cli): adopt Patch terminology across workflows`) applied a blanket
   `changespec` -> `patch` rename across 444 files. That rename hit the deliberate
   dual-accept pair and collapsed it into `["patch", "patch", ...]`.

So this is a rename over-application, not a hand-authored typo. The fix is to restore
`changespec`, **not** to delete the duplicate: `src/sase/axe/chop_policy.py:124` still
accepts it on purpose:

```python
# Legacy compatibility: persisted chop policies may still use
# the old provider name.
guard.get("provider") in {"patch", "changespec"}  # legacy provider name
```

Dropping `changespec` from the schema would leave the LSP flagging persisted chop
configs that the runtime still honors. The identical pattern survives correctly in
`src/sase/xprompt/vcs_project_completion.py:56`
(`VcsProjectEntryKind = Literal["project", "patch", "changespec"]`), which confirms the
intended shape.

## Why CI and the test suite did not catch this

Two independent gaps:

1. **No test meta-validates the schema.** The `tests/test_config_schema*.py` family only
   validates _instances_ against the schema (`Draft7Validator(schema()).validate(...)`).
   A structurally invalid schema still happily validates instances, so every one of
   those tests passes.
2. **Python's `jsonschema` cannot detect this even if asked.** Draft-07 removed
   `uniqueItems` from the metaschema's `enum` definition, so
   `Draft7Validator.check_schema(schema)` returns cleanly on the current broken file
   (verified). yaml-language-server meta-validates with **Ajv** instead
   (`yamlSchemaService.js:216`), whose draft-07 metaschema does enforce enum uniqueness.

The consequence for this plan: the regression guard must assert enum uniqueness
**explicitly** by walking the schema. Calling `check_schema()` alone would be a test
that passes against the known-broken file and therefore proves nothing.

## Verification performed while diagnosing

- Reproduced the exact untruncated diagnostic by compiling the schema against Ajv's
  draft-07 metaschema using the same Ajv that ships inside the installed
  yaml-language-server. The first reported error matches the screenshot character for
  character.
- Confirmed `Draft7Validator.check_schema()` passes on the broken file (the gap above).
- Confirmed the candidate fix makes the schema valid under Ajv, with zero errors. The
  secondary `must be array` / `must match a schema in anyOf` errors in the raw Ajv
  output are cascading `allErrors` noise from the same four sites, not separate defects.
- Scanned the whole schema for duplicate JSON object keys: none.
- Validated `src/sase/xprompts/workflow.schema.json` against the same metaschema:
  already clean. It should still be covered by the new guard.

## Scope

### 1. Restore `changespec` in the four enums

In `src/sase/config/sase.schema.json`, change all four occurrences of

```json
["patch", "patch", "agent_hood", "agent_clan"]
```

to

```json
["patch", "changespec", "agent_hood", "agent_clan"]
```

Lines 174, 184, and 1198 are single-line array literals. Line 1212 sits inside a
multi-line `propertyNames.enum` block, so match it in its wrapped form rather than
assuming one-line formatting. Preserve the file's existing indentation and formatting
exactly; do not reformat the schema.

JSON has no comments, so the legacy value cannot be annotated inline the way
`chop_policy.py` and `vcs_project_completion.py` annotate theirs. The regression guard
in step 3 is what protects these sites from collapsing again.

### 2. Fix the same collapse in the test helper

`tests/ace/tui/widgets/test_vcs_project_completion.py:45` was hit by the identical
rename collapse:

```python
kind: Literal["project", "patch", "patch"] = "project",
```

Replace the inline duplicate `Literal` with the canonical exported alias:

```python
kind: VcsProjectEntryKind = "project",
```

importing `VcsProjectEntryKind` from `sase.xprompt.vcs_project_completion` alongside the
existing imports in that module. Using the alias rather than re-spelling
`Literal["project", "patch", "changespec"]` prevents this helper from drifting again.

### 3. Add the regression guard

Add a test — `tests/test_config_schema_validity.py` is a natural name, sitting beside
the existing `test_config_schema*.py` family — that:

- Walks **every** bundled JSON schema, i.e. both `src/sase/config/sase.schema.json` (via
  `sase.config.inventory.load_config_schema`) and
  `src/sase/xprompts/workflow.schema.json`, resolving paths through the package rather
  than hard-coded repo-relative strings.
- Recursively visits every `enum` array at any depth (including inside `definitions`,
  `oneOf`/`anyOf`/`allOf` branches, `items`, `properties`, `additionalProperties`, and
  `propertyNames`) and asserts each has no duplicate members. Compare members by
  `json.dumps(member, sort_keys=True)` so non-string enum values are handled.
- Reports the failing JSON-pointer path and the duplicated value in the assertion
  message, so a future failure names the offending site directly instead of just saying
  "duplicate found".
- Also calls `Draft7Validator.check_schema(...)` on each schema. This is weaker than the
  enum check for this specific bug but cheaply catches other structural breakage. Do not
  rely on it as the duplicate-enum guard.

Confirm the new test **fails** against the unfixed schema before applying the fix
(revert temporarily or assert against a fixture copy), then passes after. A regression
test that never went red is not a regression test.

`jsonschema` is already a declared dependency in `pyproject.toml`, and
`tests/_config_schema_helpers.py` already exposes a `schema()` helper — reuse it rather
than re-reading the file.

## Out of scope

- Removing `changespec` legacy support. That is a separate deprecation decision
  affecting `chop_policy.py`, `vcs_project_completion.py`, and persisted chop policies
  on disk.
- Auditing the remaining ~440 files touched by `77d18c3e1` for other rename fallout. A
  targeted sweep was already run for this defect class (duplicate `enum` members,
  duplicate `Literal`/set members, duplicate JSON keys) and turned up exactly the four
  schema sites plus the one test helper addressed here.
- Any change to the Neovim or chezmoi schema association. The association works
  correctly; it is the schema content that is invalid.

## Verification

1. `just install` first — workspace virtualenvs are ephemeral.
2. `just check`.
3. Confirm the schema is valid under the real consumer, not just Python. From the
   yaml-language-server install directory, compile `src/sase/config/sase.schema.json`
   against Ajv's bundled draft-07 metaschema and assert zero errors. The
   yaml-language-server binary resolves via `which yaml-language-server`; its
   `node_modules` provides `ajv` and `ajv/dist/refs/json-schema-draft-07.json`. This
   step is the one that actually proves the reported symptom is gone, since Python's
   validator cannot see the defect.
4. Reopen `sase/sase.yml` in Neovim and confirm the line-1 diagnostic is gone and that
   completion inside `axe.lumberjacks.*.chops` works again.
