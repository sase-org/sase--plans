---
tier: tale
title: Show configured project names, never ProjectSpec keys
goal: Every user-facing project reference in the Artifacts Commits filter surface
  renders the configured PROJECT_NAME, and a memory gotcha keeps the rule from regressing.
create_time: 2026-07-28 08:46:38
status: done
---

- **PROMPT:** [202607/prompts/project_name_display.md](prompts/project_name_display.md)
- **AGENTS:**
  - [bbugyi200.athena.mp.f0--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.mp.f0.md#member-code)
  - [bbugyi200.athena.mp.f0--plan](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.mp.f0.md#member-plan)
- **COMMITS:**
  - [4fb5980](https://github.com/sase-org/sase/commit/4fb5980600a819238d77e4add6bc3487378d5d94) — fix(ace): show configured project names in commits UI

# Show configured project names, never ProjectSpec keys

## Goal

The Commits filter query, its completions, its picker round trip, and its task labels must present a project by its
configured `PROJECT_NAME:` (ex: `sase`), never by the ProjectSpec directory key (ex: `gh_sase-org__sase`). Concretely:

- the startup-seeded `project:` token holds the configured name whenever the resolved project has one;
- every system-written project ref (picker selection, one-enabled-project fallback, `a` restore, completions) is a name,
  and any key or alias the user commits is canonicalized to the name when the warm inventory knows it;
- the key remains the canonical identity everywhere it is not shown: shared scope state, `project_file` metadata,
  backend resolution, and picker selection results;
- `sase/memory/gotchas.md` gains one short entry codifying the rule, and `sase memory init` regenerates the derived
  instruction files.

Unknown or ad-hoc refs (projects absent from the inventory, or an inventory read that fails) must degrade to the ref the
caller supplied so nothing breaks and nothing raises.

This is a tale: one behavior contract spanning the Commits startup seed, pane, picker route, copy, docs, tests, and one
memory entry, implementable and verifiable atomically by one follow-up agent.

## Background

`project:` became a first-class Commits facet in commit `fc72269b`, but every producer of its value writes the canonical
directory key:

- `src/sase/ace/tui/actions/_state_init_late.py` merges `ensure_project_file_and_get_workspace_num()`'s third element,
  which is the workspace-derived directory key, into the startup query. This is the token in the reported screenshot.
- `src/sase/ace/tui/widgets/artifacts/commits_pane.py::set_project_scope` receives a resolved `display_name` from
  `src/sase/ace/tui/actions/artifacts.py::_set_artifacts_project_scope` and discards it (`del display_name`), writing
  the key into `filters.project` for both picker selections and the one-enabled-project fallback.
- `_ensure_artifacts_project_choices` forwards `tuple(result.display_names)` to `set_commits_project_sources`, which
  iterates the dict's **keys**, so `project:` completions offer directory keys.

The backend already accepts either form: `src/sase/vcs_log/resolve.py::_resolve_explicit_project_repos` matches
`project_scope` against `record.project_name`, `effective_project_name(record)`, and aliases, so switching the visible
token to the configured name does not change which repositories are collected. No Rust-core change is required: this
adds no backend behavior, only a presentation projection over records `list_project_records` already returns.

## Design and implementation

1. Add one shared, reusable ref-to-label projection in `src/sase/project_display_names.py`.
   - Add a pure resolver that maps a project ref to its user-facing label given already-loaded inputs: exact directory
     key to its label; alias to its owning key's label; a value that already equals a known label (case-insensitively)
     to that label; anything else returned unchanged. Empty/`None` refs stay `None`.
   - Add one loader that performs a single `list_project_records` read and returns an immutable projection object
     carrying both the `ProjectDisplaySnapshot` and the alias map (build the alias map from the same records via
     `sase.project_alias_records.project_alias_map_from_records` with `strict=False`, as read paths do, rather than a
     second read through `load_project_alias_map`). Inventory failures degrade to an empty projection whose lookups are
     the identity, and never raise.
   - Export the new names in `__all__`. `sase/xprompt/project_identity.py::_canonical_xprompt_project` implements the
     same mapping today; make it delegate to the new pure resolver **only if** `tests/test_xprompt_project_identity.py`
     stays green unmodified. If its documented fallbacks cannot be preserved exactly, leave that module alone and add a
     one-line comment pointing at the shared resolver instead of duplicating semantics silently.

2. Canonicalize the startup seed in `src/sase/ace/tui/actions/_state_init_late.py`.
   - Keep `merge_commits_startup_project` pure and unchanged in signature semantics; after the merge, project the
     winning `values.project` (from the ACE query, the configured `default_query`, or cwd inference) through the new
     resolver so the composed pane starts with the configured name.
   - Perform the inventory read **only** when the merged values actually carry a project, so a non-project cwd with no
     configured/ACE project still adds zero startup work (TUI perf rule 9). One `list_project_records` call measures
     ~1.5-3 ms warm on this host; keep it synchronous and pre-mount so the first paint and the first collection agree (a
     post-mount rewrite would flash the key), and keep it out of every keystroke, completion, and render path.
   - Put the impure loader behind one clearly named function so the audit can name its `(path, function, loader)` site,
     and register that site by adding the owning file to `_PROJECT_METADATA_FLOW_FILES` and the triple to
     `_ALLOWED_METADATA_LOADERS` in `tests/test_project_display_presentation_audit.py` with a rationale. Verify the
     owning file contains no other unregistered loader call.

3. Make every system-written project ref a name in `src/sase/ace/tui/widgets/artifacts/commits_pane.py`.
   - `set_project_scope` writes `display_name or project` into `filters.project` instead of discarding `display_name`,
     and records the same visible ref in `_last_project_scope` so `toggle_all_projects` restores a name.
   - Key `_project_files` by the visible ref actually written into the query so `fetch_commits` keeps finding its
     `project_file`; its tracked-task `display_name`, task file argument, dedup key, and duplicate message must then
     read `Fetch commits (sase)`, never the key. Preserve the existing dedup/label shape otherwise.
   - `set_project_completion_sources` accepts the label-keyed inventory (labels for completion, plus the ref-to-label
     mapping needed by step 5) without adding any disk access.

4. Feed labels, not keys, from the inventory load in `src/sase/ace/tui/actions/artifacts.py`.
   - `set_commits_project_sources` must receive display labels (deduplicated, stable order) and `project_files` re-keyed
     by label; `src/sase/ace/tui/widgets/artifacts/view.py` just forwards them.
   - Leave the shared scope contract intact: `artifacts_project_scope` and the Plans/Chats/Bugs
     `set_project_scope(project=<key>, display_name=<label>)` calls keep canonical identity plus a projected label, and
     those panes keep their existing `display_name or key` rendering.
   - When opening the picker from Commits, map the query's visible ref back to a canonical `project_key` (label, key, or
     alias, case-insensitively, from the already loaded choices) before passing `current_project=`, because
     `InventoryProjectPicker` highlights by `choice.project_key`. An unresolvable ref must fall back to the existing
     all-projects highlight rather than raising or highlighting the wrong row.

5. Canonicalize user-typed refs on commit only, in `src/sase/ace/tui/widgets/artifacts/commits_filtering.py`.
   - When a submitted query commits (`_commit_filter_values` on the submit path), project `values.project` through the
     pane's warm in-memory mapping before generation bump, `filters` assignment, cache/scope-key computation, and the
     `to_query_string` write-back, so the visible text, `self.filters`, and `_authoritative_results` keys never
     disagree. A committed `project:gh_sase-org__sase` therefore redisplays as `project:sase`.
   - Never canonicalize during live preview, per-keystroke reconciliation, or completion; those paths stay read-only and
     must not touch disk (TUI perf rules 8 and 11). If the inventory has not landed yet, keep the user's text verbatim;
     the backend resolves keys and aliases either way.

6. Update user-facing copy and documentation.
   - `src/sase/ace/tui/widgets/artifacts/commit_filter_bar.py`: `KEY_COMPLETIONS` and `VALUE_HINTS` must say project
     _name_ (ex: `single project name; omitted means all projects`, value hint `project name`), not "project key".
   - `src/sase/ace/tui/modals/help_modal/changespecs_bindings.py`: `project:KEY` becomes `project:NAME`; respect the
     57-char box and 32-char description limits in `src/sase/ace/CLAUDE.md`.
   - `docs/ace.md` and the `ace.artifacts.commits.default_query` entry in `docs/configuration.md` (and the config schema
     description if it repeats the wording): `project:` accepts a configured project name, directory key, or alias, and
     committed values are canonicalized to the configured name; keep the existing singular/non-negatable and "no token
     means all projects" statements.

7. Add the memory gotcha and regenerate derived files.
   - The user explicitly requested this in the originating prompt: _"add a useful and concise ... section (use the
     format used in that file--I think we use description lists) to the sase/memory/gotchas.md file that ensures we
     never make a mistake like this again."_ That request, restated in the approval that hands this plan off, is the
     in-conversation user permission the memory rule requires; if the implementing agent is not satisfied that it has
     that permission, it must ask the user with `/sase_questions` rather than silently skip step 7.
   - Append one definition-list entry in the file's existing style (bold term, two trailing spaces, wrapped body), kept
     short because every token costs context. Suggested content, to be tightened while editing:

     **Show Project Names, Never ProjectSpec Keys** User-facing text must render a project's configured `PROJECT_NAME:`,
     never its ProjectSpec directory key (`sase`, not `gh_sase-org__sase`). Project through `sase.project_display_names`
     (or a `display_name` the caller already resolved) and fall back to the key only when no name is configured. This
     covers query tokens the TUI writes back into filter bars, completion candidates, picker rows, task labels, and
     notifications; canonical keys stay in identity, storage, and lookup paths.

   - Then run `sase memory init` to regenerate `AGENTS.md`, the provider instruction shims, and the memory README. That
     regeneration is mandatory and needs no separate approval.

## Verification

1. Unit-test the shared resolver (extend `tests/test_project_display_names.py`): directory key, configured name, alias,
   unknown ref, empty/`None`, and inventory-failure degradation; assert the loader performs exactly one
   `list_project_records` read.

2. Startup coverage in `tests/ace/tui/test_commits_config.py` (plus the startup/scaffold tests as needed), using the
   deliberately mismatched `tests/_project_display_case.py` model (`gh_acme__widgets` vs `widgets`) so a key can never
   pass by accident: the pre-mount query and first collection carry `project:widgets` for an inferred current project,
   for a configured `default_query` key, and for an ACE-query key, while precedence between those three is unchanged;
   with no resolvable project, no inventory read happens and no token appears.

3. Interaction coverage in `tests/ace/tui/test_commits_pane_filters.py` and
   `tests/ace/tui/test_commits_pane_interactions.py`: picker selection and the one-enabled-project fallback write the
   label; `a` removes and restores the label; opening the picker from a label-bearing query highlights the matching
   `project_key`; committing a typed key or alias rewrites the visible token to the name while preserving every other
   facet; a typed unknown ref survives verbatim; `fetch_commits` submits a name-bearing display name, dedup key, and the
   correct `project_file`; other Artifacts panes keep their existing shared-scope behavior.

4. Completion coverage in `tests/ace/tui/widgets/test_commit_filter_bar.py`: `project:` candidates are configured names,
   and the new hint copy is asserted.

5. Add autouse isolation for the new inventory read in `tests/ace/tui/conftest.py`, matching the existing
   `_isolate_commits_current_project` fixture, so ACE tests never depend on the host's `~/.sase/projects`; tests that
   need a project inject their own deterministic inventory.

6. Extend `tests/test_project_display_presentation_audit.py` with the registered loader site from step 2. Note that the
   existing structural audit only catches canonical _attributes_ reaching sink calls, so it cannot catch a plain scope
   string written into a query facet; the behavior tests above carry that guarantee.

7. Update `tests/ace/tui/visual/test_ace_png_snapshots_commits.py` only for intended changes (project completion
   candidates and the `project name` hint text), inspecting `.pytest_cache/sase-visual/` actual/expected/diff artifacts
   before accepting goldens with `--sase-update-visual-snapshots`.

8. Because this changes files in this repository, finish with `just install` then `just check`, and resolve every lint,
   mypy, Symvision, unit, and exact visual-snapshot failure before handoff. Confirm `sase validate` passes after
   `sase memory init` regenerates the instruction files.
