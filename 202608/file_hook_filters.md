---
tier: tale
title: Nest file-hook filters under filters
goal:
  File-hook configuration, runtime models, and public inventories group every
  event-selection field under filters, and the research-highlights hook is migrated
  without changing its behavior.
proposed_by: bbugyi200.athena.vq.f1
create_time: 2026-08-08 11:44:42
status: wip
---

# Plan: Nest file-hook filters under `filters`

## Goal

Make `file_hooks` structurally distinguish hook execution settings from event-selection
criteria. Every selection field—`projects`, `sidecars`, `path_globs`,
`agent_name_globs`, and `ops`—must live under an optional `filters` mapping, while
`name`, `description`, `command`, and `timeout` remain properties of the hook itself.
Migrate the configured `research-highlights` hook in the linked chezmoi repository to
the new shape without changing what it matches or executes.

## Current behavior

- `src/sase/config/sase.schema.json` defines all five filter fields directly on each
  `file_hooks` item.
- `src/sase/config/file_hooks.py` separately validates the runtime YAML because config
  layers are not validated against the JSON schema while loading. Its known-key set,
  project-local auto-scoping check, parser, immutable `FileHookConfig`, and event
  matcher all assume flat fields.
- `sase file-hook list --json` serializes `FileHookConfig` with `asdict`, so its
  version-2 machine-readable payload also exposes the filters flat. The human table
  displays the same fields in a Filters column.
- The execution-batch wire format stores only already-matched run information; it does
  not serialize filter configuration.
- The user-layer `research-highlights` declaration in the linked `chezmoi` repository
  still uses the flat filter fields.

## Schema and compatibility decisions

1. Introduce a frozen `FileHookFilters` value object containing the five optional
   selection fields, and make `FileHookConfig.filters` the single runtime property for
   them. This keeps the runtime model, public JSON inventory, and authored YAML aligned
   instead of parsing a nested document into an unrelated flat shape.
2. Treat `filters` as optional and an empty mapping as unrestricted. Preserve the
   existing default semantics for every omitted field.
3. Move `projects` under `filters` along with the four fields shown in the requested
   config example. It is also an event-selection dimension; the example omits it only
   because the global `research-highlights` hook has no project restriction.
4. Make the old top-level filter fields invalid rather than accepting two equivalent
   layouts. Runtime loading must skip such a hook with a targeted migration warning,
   just as it currently does for the obsolete `globs` key. The JSON schema must reject
   old and mixed layouts through `additionalProperties: false` at both levels.
5. Increment `FILE_HOOK_LIST_JSON_SCHEMA_VERSION` from 2 to 3 because the public JSON
   entry shape changes from flat fields to a nested `filters` object. Keep the human
   table labels and matching semantics unchanged.
6. Do not bump `BATCH_SCHEMA_VERSION`: pending batches contain no config-filter object,
   and changing their accepted version would discard otherwise executable work.

## Implementation

### 1. Define and parse the nested runtime model

Update `src/sase/config/file_hooks.py` to:

- Add `FileHookFilters` with optional `projects`, `sidecars`, `path_globs`,
  `agent_name_globs`, and `ops` fields, including direct-construction validation of
  operation names.
- Replace the five flat `FileHookConfig` fields with one `filters` field and update the
  matcher to read through it.
- Restrict hook-level keys to `name`, `description`, `command`, `filters`, and
  `timeout`; independently restrict nested keys to the five supported filter names.
- Require `filters`, when present, to be a mapping. Parse its string-list and operation
  values with field paths/messages that identify `filters.<name>`.
- Emit an explicit migration diagnostic when any former top-level filter key is seen,
  and continue to reject unknown nested keys so typos cannot broaden a hook to match
  every event.
- Preserve project-local auto-scoping by detecting whether `filters.projects` is absent,
  then injecting the detected user-facing project into the runtime filter object. An
  explicitly supplied `filters.projects` continues to win.
- Export the new type from `sase.config.file_hooks` and `sase.config` alongside the
  existing file-hook API.

Update direct constructors and accesses in the execution engine and tests to use the
nested value object. Matching results and the batch payload should otherwise remain
byte-for-byte equivalent for equivalent configuration.

### 2. Change the public schema and inventory projection

Update `src/sase/config/sase.schema.json` so a file hook has one optional `filters`
object referencing a dedicated definition whose properties are the five filter fields.
Both the hook and filter objects must disallow extra properties, and the existing item
types, operation enum, descriptions, and omission defaults must be preserved.

Update `src/sase/main/file_hook_handler.py` to:

- Render the human Filters column from `hook.filters` without changing its labels or
  wildcard display.
- Serialize version-3 JSON entries with a nested `filters` object and no legacy flat
  filter keys.

Use focused contract tests to prove that the JSON schema accepts nested filters and an
omitted/empty `filters` object, rejects old top-level fields, rejects unknown nested
fields and invalid nested types/operations, and continues to validate the packaged
default config.

### 3. Cover runtime migration and unchanged behavior

Refactor `tests/test_file_hooks.py`, `tests/test_file_hook_cli.py`, and
`tests/test_file_hook_engine.py` around `FileHookFilters`, then add assertions for:

- successful loading of the new nested YAML shape;
- omitted filters and project-local implicit project scoping;
- explicit nested project scope overriding implicit scoping;
- actionable rejection of every legacy top-level filter location, malformed `filters`,
  and unknown nested filter names;
- unchanged path, agent-name, project, sidecar, and operation matching semantics;
- version-3 list JSON containing exactly one nested filter object while the human list
  still shows all five dimensions; and
- unchanged version-1 execution batches for a matched hook.

Keep loader failures fail-soft: an invalid hook is warned about and skipped, never
silently converted into an unrestricted hook and never allowed to fail the commit or
artifact operation that triggered loading.

### 4. Document and migrate the real configuration

Update the `file_hooks` section of `docs/configuration.md` to show `projects`,
`sidecars`, `path_globs`, `agent_name_globs`, and `ops` beneath `filters`, distinguish
hook-level fields from filter subfields in the reference table, and document the
breaking migration and project-local auto-scope behavior in terms of `filters.projects`.

In the linked `chezmoi` repository, update `home/dot_config/sase/sase.yml` so the
existing `research-highlights` filter values are nested under `filters`. Preserve its
current command `bob highlights create --research-root .`, description, timeout, and all
filter values. Apply only that chezmoi-managed config target after the updated SASE
package is installed so the live config is never left in a shape the installed loader
cannot understand.

## Verification

1. Run `just install` in the SASE repository before validation, as required for an
   ephemeral workspace.
2. Run the focused config, loader/matcher, CLI, and execution-engine tests covering the
   files above, including the schema acceptance/rejection cases and list JSON version.
3. Run `just check` in the SASE repository. Escalate to `just check-full` if scoped
   selection broadens or reports an unusual selection, per repository instructions.
4. Parse the edited chezmoi YAML independently and inspect its diff to confirm that only
   the intended structural nesting changed.
5. Apply the one chezmoi SASE config target, then run `sase file-hook list --json` and
   assert that `research-highlights` is present with schema version 3, the nested filter
   values from the requested example, the unchanged research-root command and timeout,
   and no top-level filter keys.
6. Check both repositories' final diffs/statuses and confirm no unrelated files were
   changed. Do not commit unless separately requested or a post-completion finalizer
   explicitly asks for commits.

## Acceptance criteria

- The supported YAML contract places all five event filters exclusively under
  `file_hooks[].filters`.
- Equivalent nested configuration selects exactly the same events as the old flat
  configuration, including implicit project scoping for project-local hooks.
- Old or malformed layouts fail safely with useful diagnostics and can never broaden a
  hook accidentally.
- The JSON schema, runtime loader/model, list JSON, tests, and documentation agree on
  the nested structure; versioned outputs are updated only where their wire shape
  actually changes.
- The chezmoi source and applied SASE config use the new structure, and the effective
  `research-highlights` hook retains its current behavior.
