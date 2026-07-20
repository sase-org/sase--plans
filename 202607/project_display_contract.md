---
tier: tale
title: Establish the project identity and display projection contract
goal: Provide an immutable, refreshable project-display snapshot and typed identity/label
  projections that downstream human-facing renderers can consume without filesystem
  I/O or loss of canonical identity.
create_time: 2026-07-20 12:50:54
status: done
prompt: 202607/prompts/project_display_contract.md
---

# Plan: Establish the project identity and display projection contract

## Context and boundaries

Project lifecycle records already distinguish the canonical directory key from the configured name through
`ProjectRecordWire.project_name`, `ProjectRecordWire.display_name`, and `effective_project_name`. The current
`project_display_names` convenience layer collapses that distinction into mutable dictionaries, refreshes against only
the projects-root directory mtime, and lets presentation callers resolve names directly. A nested ProjectSpec rename
therefore can remain stale, while TUI consumers have no explicit immutable value to load off-thread and pass to pure
renderers.

This phase will establish the reusable Python presentation boundary only. Canonical project keys remain authoritative
for persistence, paths, joins, filters, colors, selections, and structured payloads; Rust lifecycle and statistics
aggregation contracts remain unchanged. Migrating every Statistics and remaining CLI/TUI surface belongs to the
dependent phases.

## Immutable display projection

- Refactor `sase.project_display_names` around a public immutable snapshot built in one batch from lifecycle records
  using `effective_project_name`. The snapshot will own key-to-label data that cannot be mutated after construction and
  will resolve unknown/deleted keys to the canonical key.
- Add an explicit typed projection carrying `project_key` and `project_label`, with a deterministic
  visible-label/canonical-key sort key. Duplicate display labels remain separate projections and are never treated as
  unique identities.
- Provide a fresh-load entry point for worker/off-thread refresh paths plus explicit cached-snapshot refresh and
  invalidation behavior for the existing single-key convenience APIs. Do not infer freshness from the projects-root
  directory mtime; a refresh must observe a nested ProjectSpec display-name edit even when that directory timestamp is
  unchanged.
- Invalidate the convenience cache after the supported project-name mutation path so in-process updates become visible
  immediately. Preserve graceful fallback if the lifecycle inventory cannot be read.

## Snapshot-aware presentation helpers

- Extend project-label, ChangeSpec-name, prompt/VCS-ref, and safe-stem humanization helpers to consume a supplied
  snapshot without reading, globbing, or statting the filesystem. Keep the existing root-based convenience signatures
  for non-rendering callers, backed by the explicitly managed cache.
- Document in types and docstrings that storage/query code supplies canonical keys, renderers consume labels from a
  preloaded snapshot, and interactions retain the paired key. Keep longest-prefix ChangeSpec humanization deterministic
  when project keys overlap.
- Preserve compatibility for existing attachment and signature consumers while making their data derive from the
  immutable snapshot rather than exposing mutable map state.

## Verification

- Expand focused unit coverage with the deliberately mismatched `gh_acme__widgets` → `widgets` fixture, unknown-record
  fallback, overlapping project keys, two distinct keys sharing one label, immutable snapshot behavior, and stable
  label/key sorting.
- Prove batch loading occurs once per snapshot and that snapshot-supplied helpers perform no lifecycle reload. Prove an
  explicit refresh observes a nested display-name rename without changing the projects-root mtime, and that explicit
  invalidation/mutation refresh semantics do not leak stale labels.
- Run the project-display and project-alias focused tests, then install the workspace dependencies as required and
  finish with `just check`.
