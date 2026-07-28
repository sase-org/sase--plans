---
tier: tale
title: First-class project filters for Artifacts Commits
goal: The Commits query is the sole visible and authoritative source of project scope
  from startup through picker changes.
create_time: 2026-07-28 06:24:40
status: wip
---

- **PROMPT:** [202607/prompts/commits_project_filter.md](prompts/commits_project_filter.md)
- **AGENTS:**
  - [bbugyi200.athena.mp](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.mp/README.md)
  - [bbugyi200.athena.mp--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.mp.md#member-code)
- **COMMITS:**
  - [fc72269](https://github.com/sase-org/sase/commit/fc72269b639c93cc29cd617d5b3dd3da4d91cd3d) — feat(ace): add project facet to commits filters

# First-class project filters for Artifacts Commits

## Goal

Make the persistent Commits query the sole visible and authoritative representation of project filtering:

- add a singular, non-negatable `project:<project>` facet to the Commits query language;
- make the Commits project picker replace that facet while preserving every other committed filter, and make its “All
  projects” choice remove the facet;
- ensure every automatically selected current/explicit project is present in the query before the first Commits
  collection, while an absent `project:` facet means a true all-project collection;
- remove the separate Commits scope label and any other persistent project-scope presentation so the filter row is the
  only project-filter indicator.

This remains a tale because the parser, pane state, picker route, tests, help, and documentation form one tightly
coupled behavior change that one follow-up agent can implement and verify atomically.

## Design and implementation

1. Extend the pure Commits query contract in `src/sase/vcs_log/filter_query.py`.
   - Add a single `project` field to `CommitLogFilterValues` and recognize `project:` in parsing, completion context,
     canonical token rendering, equality/hash/cache keys, and round trips.
   - Canonicalize `project:` before `repo:` so the outer project scope is immediately visible. Quote values through the
     existing token helpers.
   - Keep the facet singular and non-negatable: the project picker selects one project, and the current VCS-log resolver
     accepts one explicit project scope. Reject duplicate, comma-list, empty, and negated project tokens with exact
     token spans instead of silently inventing multi-project semantics.
   - Do not push project into `CommitFilters` or the in-memory commit matcher; it selects the repository constellation
     before provider collection, unlike repo/author/date/subject filters.

2. Make the query, rather than independent pane flags, own Commits collection scope in
   `src/sase/ace/tui/widgets/artifacts/commits_collection.py`,
   `src/sase/ace/tui/widgets/artifacts/commits_filtering.py`, and `src/sase/ace/tui/widgets/artifacts/commits_pane.py`.
   - Derive each collection spec and cache/snapshot scope from `filters.project`: a value calls `run_vcs_log` with that
     `project_scope` and `all_projects=False`; no value calls it with `all_projects=True` and no project scope. Remove
     or collapse the mutable `project_scope`/`all_projects` state so it cannot disagree with the visible query.
   - Ensure project changes use the existing `_commit_filter_values` reconciliation path. This must preserve the other
     query facets, invalidate stale generations/cache scopes, update the persistent read-only filter text immediately,
     retain selection when possible, and schedule collection only through the existing off-thread/coalesced worker.
   - Adapt the configured `a` compatibility action to the same query state: clearing an active `project:` means all
     projects, and restoring the last known automatic/picked project re-adds the visible token. If there is no known
     project to restore, direct the user to `p` rather than creating hidden current-project scope.
   - Keep fetch task labels/files derived from the effective project filter without reintroducing project scope in the
     header.

3. Preload and maintain the effective project token through the Artifacts lifecycle in
   `src/sase/ace/tui/actions/_state_init_late.py`, `src/sase/ace/tui/actions/_startup_mount.py`,
   `src/sase/ace/tui/actions/artifacts.py`, and `src/sase/ace/tui/widgets/artifacts/view.py`.
   - Resolve the current registered project read-only during the existing pre-mount state/config setup, outside
     keystroke/render handlers, and merge it into the configured Commits default only when the default has no explicit
     `project:`. An explicit project from the ACE query/shared initial scope takes precedence over cwd inference.
   - Pass the merged values into `CommitsPane` before compose/mount so its visible filter bar and first collection
     agree; do not briefly collect a hidden current-project scope under a query without `project:`.
   - Preserve the existing lazy, off-thread project inventory read. If its one-enabled-project fallback selects a
     project later, route that selection through the pane’s query mutation method so the canonical token appears before
     the corresponding recollection.
   - When `p` is used on Commits, seed the picker highlight from the current `project:` value. On selection, replace the
     project facet while retaining sidecar/repo/author/date/limit/text filters; on “All projects,” remove it. Keep the
     existing shared-scope behavior for Plans, Chats, and Bugs.
   - Feed the already loaded project inventory into the Commits filter bar as in-memory `project:` value completions
     (canonical keys, with display labels where supported) without adding disk access to typing or completion paths.

4. Remove redundant project-scope presentation and update user guidance.
   - Simplify `build_commits_info_header` and its callers in `src/sase/ace/tui/widgets/artifacts/commits_rendering.py` /
     `src/sase/ace/tui/widgets/artifacts/commits_pane.py` so the header retains Commits, limit, and refresh status but
     never renders “Scope,” “Current project,” “All projects,” or a project display name.
   - Add `project:` to `CommitFilterBar` key/value completion hints in
     `src/sase/ace/tui/widgets/artifacts/commit_filter_bar.py`.
   - Update the mandatory ACE `?` help content in `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py` to
     describe `project:` and the query-backed picker/all-project behavior within the modal’s width constraints.
   - Update `docs/ace.md`, the `ace.artifacts.commits.default_query` documentation in `docs/configuration.md`, and the
     config schema description if needed so startup precedence, singular project syntax, picker behavior, and “no
     project token means all projects” are explicit.

## Verification

1. Expand `tests/test_vcs_log_filter_query.py` for project parse/serialize order, quoting, round trips, duplicate/list/
   negation errors with exact spans, unknown-key suggestions, and completion classification. Extend
   `tests/ace/tui/widgets/test_commit_filter_bar.py` for project key/value completion from warm in-memory sources.

2. Expand `tests/ace/tui/test_commits_config.py`, `tests/ace/tui/test_commits_pane_filters.py`,
   `tests/ace/tui/test_commits_pane_interactions.py`, and `tests/ace/tui/test_artifacts_scaffold.py` to prove:
   - an explicit configured or ACE-query project is present before mount and drives the first collection;
   - an inferred current project is preloaded only when no explicit default project exists;
   - no project token produces `all_projects=True`, never a hidden cwd scope;
   - `p` replaces/removes only `project:` and preserves every other committed token;
   - the compatibility all-project toggle rewrites the visible query;
   - project changes invalidate/recollect through the worker path, the picker highlights the query value, and other
     Artifacts panes retain their existing shared-scope behavior;
   - Commits headers and help contain no duplicate project-scope indication.

3. Update the affected Commits PNG snapshot assertions/goldens in
   `tests/ace/tui/visual/test_ace_png_snapshots_commits.py` only for the intentional query/header/completion changes,
   inspecting generated actual/expected/diff artifacts before accepting them.

4. Run focused parser, config, filter-bar, Commits pane, Artifacts scaffold, help, and visual snapshot tests during
   implementation. Because implementation changes files in this repository, finish with `just install` and `just check`,
   and resolve all lint, type, unit, and exact visual snapshot failures before handoff.
