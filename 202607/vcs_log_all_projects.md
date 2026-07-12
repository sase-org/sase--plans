---
create_time: 2026-07-10 10:51:49
status: done
prompt: 202607/prompts/vcs_log_all_projects.md
tier: tale
---
# Plan: Add all-project scope to `sase vcs log`

## Goal

Add `-a` / `--all` to `sase vcs log` so one invocation builds a single chronological timeline from every repository
known through every registered SASE project, including each project's primary checkout, configured linked repositories,
and separate versioned SDD store. Move the existing author filter's short option from `-a` to `-A` while retaining
`--author`.

The default command must remain scoped to the current project constellation. Global scope must stay provider-neutral,
read-only during repository discovery, deterministic, and resilient when individual project records, checkouts,
linked-repo configurations, or VCS providers are unavailable.

## User-facing behavior

- `sase vcs log -a` and `sase vcs log --all` discover registered projects across all lifecycle states (active, inactive,
  and sibling), matching the user-visible SASE project inventory while excluding the system-managed `home`
  pseudo-project.
- Global discovery works regardless of the caller's current directory; it does not require the caller to be inside a
  recognized SASE workspace.
- Each usable project contributes its primary repository, linked repositories resolved with `materialize=False`, and any
  existing separate-repository SDD history. In-tree/unversioned SDD storage remains excluded because it has no distinct
  commit history.
- `-A PATTERN` and `--author PATTERN` select authors exactly as `-a` / `--author` do today. `-a` is no longer an author
  alias.
- `--all` and `--current-only` are mutually exclusive because they request contradictory scopes. Keep `--repo`
  compatible with `--all`, applying it to the expanded, deduplicated global repository catalog.
- Existing date, author, fetch, branch/ref, tag, ordering, format, and limit semantics remain unchanged. The default
  limit is still the newest 40 commits from the merged candidate histories; `--limit 0` remains the way to request an
  unlimited timeline.
- Repository labels in pretty, full, oneline, and JSON output remain stable and unique. Prefer a registered project's
  effective display name when a checkout is both a standalone project and another project's linked repo; retain simple
  linked-repo labels when unique; qualify colliding linked/SDD labels with their owning project. Filtering should
  recognize the displayed label and unambiguous source aliases so common names such as `sase-core` and `chezmoi` remain
  convenient.
- A physical checkout is queried at most once even when it is registered independently, linked from multiple projects,
  or reached through symlinked/equivalent paths. This prevents duplicate commits, duplicate fetches, and name-key
  collisions in remote state and rendering.
- Bad project records and per-project linked/SDD resolution failures become project-qualified warnings and do not hide
  healthy projects. Repeated equivalent warnings and shared-repo discoveries should be collapsed so global output stays
  useful.

## Implementation approach

1. **Extend the CLI contract.** Update `src/sase/main/parser_vcs.py` to add the alphabetically listed `-a` / `--all`
   flag, change author to `-A` / `--author`, and express the all/current-only conflict through argparse. Update
   `src/sase/main/vcs_handler.py` to pass the new scope bit through the log pipeline. Preserve the existing long option
   and every non-scope default.

2. **Add an explicit global repository-resolution path.** Refactor `src/sase/vcs_log/resolve.py` so the existing
   single-project/fallback resolver remains intact while an all-project path:
   - obtains all lifecycle records through the Rust-backed
     `list_project_records(sase_projects_dir(), "all", include_home=False)` inventory;
   - validates each record's project file and primary workspace path, surfacing unusable records as warnings rather than
     substituting the caller's checkout;
   - reuses the current primary, linked-repo, and SDD helpers for each project without materializing workspaces or
     creating SDD stores;
   - carries project provenance internally while it canonicalizes paths, promotes registered-primary metadata over
     linked aliases, deduplicates candidates, assigns collision-free labels, and produces a deterministic final order
     (registered primaries, linked-only repos, then SDD repos, sorted stably within each class);
   - applies repeatable `--repo` filters after global naming/deduplication and reports unmatched selectors consistently.

   This stays in the Python resolver because it is filesystem/config/plugin orchestration, while reusing the Rust core's
   authoritative project inventory and commit aggregation rather than duplicating either domain model.

3. **Thread scope through collection without changing aggregation semantics.** Update `src/sase/vcs_log/collect.py` so
   `run_vcs_log()` requests the selected resolver scope and continues fetching at most the requested merged limit per
   unique repo before using the existing Rust aggregator for the global top-N timeline. Preserve
   resolution-warning-before-collection-warning ordering and per-repository failure isolation.

4. **Make non-Git/local-only providers first-class in the global timeline.** Treat an absent or explicitly unsupported
   remote-log resolver as a local-history capability rather than a repository failure. Such providers should still be
   queried through the standard provider-neutral `log()` hook and their commits marked with unknown remote presence;
   Git-capable providers retain fetch-cache, ref-union, and ahead/behind behavior. Providers that do not implement the
   log hook should produce an isolated, actionable warning without affecting other VCS types.

5. **Keep output auditable.** Preserve all current render formats and commit-label behavior using the unique final repo
   names. Include the selected all-project scope in the JSON query metadata so machine consumers can distinguish a
   global result from the default constellation, and ensure warnings, remote-state lookup, colors, and legends remain
   correct when many projects resolve or some have no selected commits.

6. **Update documentation.** Revise `docs/vcs.md`, the `sase vcs` CLI flag table in `docs/configuration.md`, and the
   command examples/index where appropriate to document `--all`, the `-A` author migration, lifecycle coverage,
   deduplication, provider-neutral/local-only behavior, interaction with `--repo` and `--current-only`, and the
   distinction between global candidate scope and the merged `--limit` cap.

## Verification

- Extend `tests/main/test_vcs_parser.py` to cover default `all=False`, both all-project spellings, the new `-A` author
  alias and retained `--author`, the reassignment of `-a`, the all/current-only parse error, and alphabetical/help text
  expectations.
- Extend `tests/test_vcs_log_resolve.py` with mocked active/inactive/sibling project records covering global invocation
  outside a workspace, primary/linked/SDD expansion, unusable-record warnings, deterministic ordering, canonical/symlink
  path deduplication, promotion of a linked checkout to its registered project identity, repeated shared linked repos,
  label collisions, warning deduplication, and global `--repo` filtering. Retain the existing tests as regression
  coverage for default and fallback scope.
- Extend `tests/test_vcs_log_collect.py` to verify scope forwarding, one provider call per deduplicated checkout, merged
  top-N behavior across globally resolved repos, warning order/failure isolation, and local-log fallback when remote
  comparison is unsupported.
- Extend `tests/test_vcs_log_render.py` for unique labels and the global JSON query marker, including a mix of
  remote-capable and local-only repositories.
- Run focused parser/resolver/collector/render tests, then run the repository-required `just check` after
  `just install`.

## Non-goals

- Do not add `--all` to `sase vcs list` as part of this change; the requested surface is `sase vcs log`.
- Do not materialize missing linked or numbered workspaces, create SDD stores, mutate project lifecycle state, or
  silently register new projects.
- Do not add VCS-specific branches to the CLI. New/external VCS plugins participate through the existing
  provider-neutral log contract.
