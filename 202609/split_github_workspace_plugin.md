---
tier: tale
title: Split sase-github workspace_plugin.py into a workspace/ package
goal:
  Every file, including src/sase_github/workspace_plugin.py, is 700 lines or fewer after
  splitting the 1,923-line module into cohesive workspace/ submodules, with no behavior
  change and a green `sase tool run check` in sase-github.
size: medium
proposed_by: bbugyi200.athena.0q2.r0
create_time: 2026-09-23 11:05:58
status: wip
---

# Plan: Split `sase_github/workspace_plugin.py` into a `workspace/` package

## Goal

`src/sase_github/workspace_plugin.py` in the **sase-github** linked repo is 1,923 lines.
It holds the `GitHubWorkspacePlugin` hookimpl class plus every helper behind it: `#gh`
ref resolution, project-record naming, repo and namespace completion, SDD sidecar
discovery and creation, cloning, PR submission, and mail prep.

Split it into cohesive modules so that **every file, including `workspace_plugin.py`, is
≤ 700 lines**. The target is ~310 lines at most.

This is a **pure structural refactor**. Runtime behavior, hook signatures, error
messages, and subprocess argv must not change. The only intentional non-move edits are:

1. Renaming the helpers that other modules now import (see "Naming rule").
2. Turning 5 hook bodies into thin delegations.
3. Deleting 2 dead private helpers.

Work in the sase-github checkout: run `sase repo open sase-github -r "<reason>"` and use
the printed path. Read its `CLAUDE.md` first. That repo's commit becomes part of your
final declaration.

## Target layout

`workspace_plugin.py` stays where it is: the entry point is
`sase_github.workspace_plugin:GitHubWorkspacePlugin`, and keeping the file preserves
`git log --follow`. The implementation moves into a new sibling subpackage,
`src/sase_github/workspace/`.

| File                       | Owns                                                                                                                                | ~Lines |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------ |
| `workspace_plugin.py`      | `GitHubWorkspacePlugin` with every `@hookimpl`, plus `_HOSTED_URL_RE`; the SDD, mail, and submit hooks delegate                     | ~310   |
| `workspace/__init__.py`    | Docstring only: "Implementation modules behind `sase_github.workspace_plugin.GitHubWorkspacePlugin`." No re-exports.                | ~3     |
| `workspace/gh_cli.py`      | Shared `gh` plumbing: `non_interactive_gh_env`, the `looks_like_*_error` output classifiers, `DEFAULT_GH_TIMEOUT_SECONDS`           | ~75    |
| `workspace/remotes.py`     | Origin inspection, GitHub remote URLs, remote matching, and SSH→HTTPS clone                                                         | ~150   |
| `workspace/projects.py`    | Home-rooted layout (`~/.sase/projects`, `~/projects/github/...`), SASE project-record lookup, canonical `gh_<owner>__<repo>` naming | ~210   |
| `workspace/refs.py`        | `#gh` ref resolution: `resolve_gh_ref`, `peek_gh_ref`, and their private resolvers                                                  | ~185   |
| `workspace/completion.py`  | Prompt completion: `gh repo list` candidates and local owner namespaces                                                             | ~225   |
| `workspace/sdd_repo.py`    | SDD sidecar repo identity (suffix, candidates, description) and `gh` operations on it (probe, create, label)                        | ~310   |
| `workspace/sdd_sidecar.py` | SDD hook flows: preflight, create-or-verify, and materialize, plus their record and staging helpers                                 | ~310   |
| `workspace/submit.py`      | Patch submission: `PR_URL_RE`, `submit_patch`, PR state checks, `gh pr merge`                                                       | ~290   |
| `workspace/mail.py`        | Interactive mail prep (`prepare_mail`)                                                                                              | ~65    |

The import graph must stay acyclic and layered like this. Nothing in `workspace/` may
import `workspace_plugin`.

- `gh_cli`, `remotes`, `projects`, `mail` → only `sase.*` or stdlib
- `refs` → `projects`, `remotes`
- `completion` → `gh_cli`, `projects`
- `sdd_repo` → `gh_cli`
- `sdd_sidecar` → `sdd_repo`, `remotes`
- `submit` → `gh_cli`
- `workspace_plugin` → all of the above

### Why this split (alternatives rejected)

- **Group by hook family, not by layer.** Each module maps to one family of hooks, so a
  reader chasing a hook opens one file. The shared leaf modules (`gh_cli`, `remotes`,
  `projects`) exist only because more than one family needs them.
- **Two SDD modules.** SDD alone is about 620 lines once the three hook bodies move out
  of the class, which is too close to the cap for one file. It splits cleanly into
  "which repo, and `gh` calls against it" (`sdd_repo`) and "hook orchestration"
  (`sdd_sidecar`). The dependency runs one way.
- **Hook bodies move out of the class.** Keeping all bodies inline would leave
  `workspace_plugin.py` at ~620 lines, right at the cap, and mix routing with
  orchestration. Short hooks (≤ ~30 lines) stay inline; only `ws_preflight_sdd_sidecar`,
  `ws_create_sdd_remote`, `ws_materialize_sdd_store`, `ws_prepare_mail`, and `ws_submit`
  delegate.
- **Rejected: turning `workspace_plugin.py` into a package.** The user asked to shrink
  this file, and a package loses `--follow` history for it.
- **Rejected: flat top-level modules.** That would put ~15 modules in `sase_github/`,
  with generic names like `submit.py` and `mail.py` sitting beside `plugin.py`, which is
  the VCS plugin and also does PR and mail work.

## Symbol map

Current line numbers are approximate hints; match by symbol name. Moved code is copied
verbatim, except for the renames in the next section and the dedent of hook bodies.

### `workspace/gh_cli.py`

- `_non_interactive_gh_env` → `non_interactive_gh_env`
- `_looks_like_auth_error` → `looks_like_auth_error`. Keep the HTTP 403 comment.
- `_looks_like_not_found_error` → `looks_like_not_found_error`
- `_looks_like_already_exists_error` → `looks_like_already_exists_error`
- `_looks_like_network_error` → `looks_like_network_error`
- `_GH_REPO_LIST_TIMEOUT_SECONDS = 10` → `DEFAULT_GH_TIMEOUT_SECONDS = 10`. Both the
  repo list timeout and the `_sdd_network_timeout` fallback use it, as they do today.

### `workspace/remotes.py`

- `_read_git_origin` → `read_git_origin`
- `_read_github_origin` → `read_github_origin`. Import `GitHubRemote` under
  `TYPE_CHECKING`.
- `_remote_matches_repo` → `remote_matches_repo`
- `_github_ssh_url` → `github_ssh_url`
- `_github_https_url` stays private.
- `_clone_gh_repo` → `clone_gh_repo`
- `_remove_failed_clone_target` and `_format_clone_failure` stay private.

### `workspace/projects.py`

- `_projects_base` → `projects_base`
- `_github_workspace_dir` → `github_workspace_dir`
- `_normalized_workspace_dir` stays private.
- `_list_project_records` → `all_project_records`
- `_list_enabled_project_records` → `enabled_project_records`

  These two are renamed rather than just de-underscored. A function named
  `list_project_records` would be shadowed by its own function-local
  `from sase.core.project_lifecycle_facade import list_project_records`.

- `_canonical_project_owner` → `canonical_project_owner`
- `_find_project_record_for_workspace` → `find_project_record_for_workspace`
- `_find_project_record_for_alias` → `find_project_record_for_alias`
- `_allocate_canonical_project_name` → `allocate_canonical_project_name`
- `_project_file_for` → `project_file_for`
- `_ensure_useful_repo_name` → `ensure_useful_repo_name`
- Private: `_is_valid_project_name`, `_canonical_project_name_base`, `_project_refs`
- `ProjectRecordWire` stays a `TYPE_CHECKING` import.
- `preferred_project_spec_path` stays a module-level import.

### `workspace/refs.py`

- `peek_gh_ref` and `resolve_gh_ref` keep their public names.
- Private: `_resolved_ref_for_record`, `_peek_repo_path_ref`, `_resolve_repo_path_ref`,
  `_resolve_existing_named_ref`
- Keep these as module-level imports, because tests patch them here:
  - `find_all_patches`
  - `preferred_project_spec_path`
  - `ResolvedRef`
  - `get_default_branch`, `parse_workspace_dir`, `set_workspace_dir`

### `workspace/completion.py`

- `_list_github_repo_candidates` → `list_github_repo_candidates`
- `_repo_candidates_error` → `repo_candidates_error`. The hook uses it for
  `unsupported_namespace`.
- `_list_github_ref_namespaces` → `list_github_ref_namespaces`
- Private: `_DEFAULT_REPO_COMPLETION_LIMIT`, `_VcsRepoErrorKind`,
  `_repo_completion_limit`, `_repo_entries_from_gh_json`, `_string_field`,
  `_optional_string_field`, `_classify_gh_repo_list_error`, `_pluralize_project_count`

### `workspace/sdd_repo.py`

- `_SddRepoProbe` → `SddRepoProbe`
- `_sidecar_sdd_candidates` → `sidecar_sdd_candidates`
- `_sdd_sidecar_suffix` → `sdd_sidecar_suffix`
- `_probe_github_repo_detail` → `probe_github_repo_detail`
- `_create_github_sdd_repo` → `create_github_sdd_repo`
- `_ensure_github_sdd_label` → `ensure_github_sdd_label`
- Private: `_SDD_SIDECAR_LABEL*` constants, `_validate_sdd_sidecar_suffix`,
  `_sdd_sidecar_description`, `_probe_github_repo`, `_create_github_sdd_label`,
  `_sdd_network_timeout`
- **Delete** `_sidecar_sdd_repo`. It is dead: its definition is its only reference, and
  no test uses it.

### `workspace/sdd_sidecar.py`

New hook-flow functions, named after the hooks without the `ws_` prefix. Each takes
`(primary_workspace_dir: str, options: dict[str, object])`. The hooks' `workspace_dir`
argument is unused by all three bodies today, so the hooks keep it in their signatures
and do not forward it.

- `preflight_sdd_sidecar(...) -> SddSidecarPreflight | None`: the body of
  `ws_preflight_sdd_sidecar`, minus `del workspace_dir`.
- `create_sdd_remote(...) -> dict[str, object] | None`: the body of
  `ws_create_sdd_remote`.
- `materialize_sdd_store(...) -> dict[str, object] | None`: the body of
  `ws_materialize_sdd_store`. Replace
  `self.ws_create_sdd_remote(primary_workspace_dir, workspace_dir, options)` with
  `create_sdd_remote(primary_workspace_dir, options)`. That is a plain method call
  today, not a pluggy dispatch, so this is equivalent.

  The name matches `sase.sdd.store.materialize_sdd_store` in core but does not collide,
  because the module differs.

- Private helpers moved here: `_SDD_STORE_SCHEMA_VERSION`,
  `_sdd_repo_target_from_options`, `_discover_sidecar_sdd_repo_for_create`,
  `_require_sdd_creation_authorization`, `_sdd_store_record`, `_path_has_content`,
  `_clone_sdd_repo`
- **Delete** `_discover_sidecar_sdd_repo` (the variant without `_for_create`). It is
  dead in the same way.
- Keep the duplicated found/not-found branches in `create_sdd_remote` as they are. Do
  not deduplicate them in this change.

### `workspace/submit.py`

- `_PR_URL_RE` → `PR_URL_RE`. `ws_extract_change_identifier` imports it.
- New
  `submit_patch(patch_file: str, patch_name: str, project_basename: str, console: object | None) -> tuple[bool, str | None]`:
  the body of `ws_submit` after the `detect_workflow_type(...) != "gh"` gate. The gate
  stays in the hook, like every other hook's "is this mine?" check.
- Private: `_extract_pr_number`, `_check_pr_state`, `_check_existing_pr`,
  `_submit_via_pr_merge`
- Keep these as module-level imports, because tests patch them here:
  - `find_all_patches`, `Patch`
  - `get_default_branch`, `parse_workspace_dir`

### `workspace/mail.py`

- `_prepare_mail_git` → `prepare_mail`. Same parameters, same function-local imports.

### `workspace_plugin.py` (what stays)

- The module docstring. Add one sentence pointing at `sase_github.workspace`.
- `_HOSTED_URL_RE`, and the full `GitHubWorkspacePlugin` class. These hook bodies stay
  inline with call sites renamed:
  - `ws_get_workflow_metadata`, `ws_detect_workflow_type`, `ws_get_change_label`
  - `ws_resolve_ref`, `ws_clone_external_repo`, `ws_peek_ref`
  - `ws_list_repo_candidates`, `ws_list_ref_namespaces`
  - `ws_extract_change_identifier`, `ws_generate_submitted_check_script`
  - `ws_supports_reviewer_comments`, `ws_get_workspace_directory`
  - `ws_format_commit_description`
- These hooks become thin delegations:
  - `ws_preflight_sdd_sidecar`, `ws_create_sdd_remote`, `ws_materialize_sdd_store`
  - `ws_prepare_mail`: keeps its `ws_detect_workflow_type` gate, then calls
    `prepare_mail`.
  - `ws_submit`: keeps the legacy-name mapping and the `detect_workflow_type` gate, then
    calls `submit_patch`.
- Keep every hook signature exactly as it is, including the
  `# Legacy hook argument name retained for compatibility.` comments and parameter
  names. Pluggy matches hook arguments by name.
- Import `clone_gh_repo`, `get_default_branch`, `resolve_gh_ref`, and `peek_gh_ref` at
  module level into `workspace_plugin`. The class looks them up there, and tests patch
  `sase_github.workspace_plugin.{clone_gh_repo,get_default_branch,resolve_gh_ref}`.
- Prune imports that are no longer used. **Lint will not catch leftovers:** this repo's
  ruff config ignores `F401` (unused import) and `F821` (undefined name). Check by hand
  that `json`, `shutil`, `sys`, `Sequence`, `Mapping`, `Any`, `Literal`, `Path`,
  `Patch`, `find_all_patches`, `preferred_project_spec_path`, `set_workspace_dir`,
  `non_interactive_git_env`, and the `TYPE_CHECKING` block are gone unless still used.

## Naming rule

- A helper that another module imports loses its leading underscore. The final names are
  listed in the symbol map above.
- A helper used only inside its new module keeps its underscore.
- Tests may keep importing module-private helpers from their new home. That already
  happens today.

## Invariants that must hold

1. **Keep function-local imports function-local.** Tests patch
   `sase_github.config.get_github_orgs`, `.get_default_github_host`, and
   `.load_merged_config`. They also patch
   `sase.workspace_provider.detect_workflow_type`, `sase.running_field.*`, `sase.ace.*`,
   `sase.vcs_provider.get_vcs_provider`,
   `sase.workspace_provider.submission_utils.finalize_submission`, and
   `sase.core.project_lifecycle_facade.list_project_records`. Those patches only work
   because the imports run at call time. Moving an import to module level would also add
   to sase's plugin-load startup cost.
2. **Keep module-level imports module-level** in the module that now looks the name up.
   The list is in the symbol map above.
3. `from sase_github.workspace_plugin import GitHubWorkspacePlugin`, `resolve_gh_ref`,
   and `peek_gh_ref` still work. So do the entry point, `sase_github/__init__.py`, and
   the `publish.yml` smoke check, with no edits to them. Add no other compatibility
   re-exports for the old private names: nothing outside this repo imports them. The
   sase core repo only mentions the class in a test docstring.
4. Every new module starts with a one-line module docstring and
   `from __future__ import annotations`. It is fully typed, because mypy runs with
   `disallow_untyped_defs`.
5. Messages, argv lists, timeouts, and return shapes stay byte-for-byte the same.

## Test updates (no behavior changes to tests)

Tests stay in their current files. Update imports and patch targets only. A patch must
name the module where the name is **looked up**.

### Imports

- `tests/test_workspace_plugin.py`: keep `GitHubWorkspacePlugin` from
  `sase_github.workspace_plugin`. Import the rest from their new homes:
  - `clone_gh_repo` from `remotes`
  - `sidecar_sdd_candidates` and `probe_github_repo_detail` from `sdd_repo`
  - `_extract_pr_number` from `submit`
  - `github_workspace_dir` and `enabled_project_records` from `projects`
  - `peek_gh_ref` and `resolve_gh_ref` from `refs`

  Rename the call sites in the test bodies to match.

- `tests/test_submit_with_recorded_pr.py`: import `_check_pr_state`,
  `_extract_pr_number`, and `_submit_via_pr_merge` from `sase_github.workspace.submit`.
  Update the module docstring's function names only if they changed.

### Patch retargets

| Current target                                                                                                     | New target                                                               |
| ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| `sase_github.workspace_plugin._clone_gh_repo`                                                                      | `sase_github.workspace_plugin.clone_gh_repo`                             |
| `…workspace_plugin.get_default_branch` in `test_external_repo_hook_clones_and_reports_canonical_result`            | unchanged                                                                |
| `…workspace_plugin.get_default_branch` in `TestHostAwareWorkspace` and `TestResolveGhRef`                          | `sase_github.workspace.refs.get_default_branch`                          |
| `…workspace_plugin.find_all_patches` in `TestResolveGhRef`                                                         | `sase_github.workspace.refs.find_all_patches`                            |
| `…workspace_plugin.find_all_patches` / `.parse_workspace_dir` in `test_gh_workspace_claims.py` (`ws_submit` tests) | `sase_github.workspace.submit.find_all_patches` / `.parse_workspace_dir` |
| `…workspace_plugin._list_enabled_project_records`                                                                  | `sase_github.workspace.completion.enabled_project_records`               |
| `…workspace_plugin._repo_completion_limit`                                                                         | `sase_github.workspace.completion._repo_completion_limit`                |
| `…workspace_plugin._sdd_network_timeout`                                                                           | `sase_github.workspace.sdd_repo._sdd_network_timeout`                    |
| `…workspace_plugin.resolve_gh_ref` (`TestWsResolveRef`)                                                            | unchanged                                                                |
| `…workspace_plugin.Path.home` (`_home_patches` and `TestHostAwareWorkspace`)                                       | `sase_github.workspace.projects.Path.home`                               |

`Path.home` is a class attribute, so the patch is process-wide. The retarget is needed
anyway because `workspace_plugin` will no longer import `Path`.

### The 51 `sase_github.workspace_plugin.subprocess.run` patches

`X.subprocess` is the shared stdlib module, so every one of these patches is
process-wide. Retarget them for honesty:

- **One module's command is being faked:** use
  `sase_github.workspace.<module>.subprocess.run`, naming the module that issues the
  command:
  - `remotes`: `git clone`, origin reads
  - `completion`: `gh repo list`
  - `sdd_repo`: `gh repo view/create`, `gh label create`
  - `submit`: `gh pr view/merge`
- **`TestDetectWorkflowTypeForProject`:** keep
  `sase_github.workspace_plugin.subprocess.run`, because `ws_detect_workflow_type` stays
  in the class.
- **Negative guards and cross-module flows:** use `"subprocess.run"` directly. This
  covers the "must not spawn subprocesses" tests (`TestPeekGhRef`, the namespace
  completion guard) and flows such as SDD materialization, which runs `gh` probe, label,
  and `git clone`. It states the process-wide intent, and `refs.py` and `sdd_sidecar.py`
  do not import `subprocess`.
- The target module must actually import `subprocess`, or `patch` raises
  `AttributeError`.

## Docs

- `docs/architecture.md`: in the "GitHubWorkspacePlugin (`workspace_plugin.py`)"
  section, add a short module-map table matching "Target layout". Change "The
  `resolve_gh_ref()` function" to name `sase_github/workspace/refs.py`.
- `README.md`: in the "Project Structure" tree, add the `workspace/` package with one
  line per module.
- sase-github `CLAUDE.md`: under "Architecture", add a bullet for `workspace_plugin.py`
  and the `workspace/` package.
- Do not edit `CHANGELOG.md`. release-please owns it.

## Verification

Run everything from the sase-github checkout.

1. If `.venv` is missing, run `just install`.
2. Run `just fmt`.
3. Run `sase tool run check` (ruff + mypy + pytest). A raw `just check` is refused
   there. It must pass with the same test count as before the change: run
   `sase tool run check` once before editing to record the baseline.
4. Size check:
   `wc -l src/sase_github/workspace_plugin.py src/sase_github/workspace/*.py`. Every
   file must be ≤ 700 lines.
5. Staleness check:
   `grep -rn "workspace_plugin\._\|workspace_plugin import _" src tests` must return
   nothing.
6. Entry-point smoke:
   `.venv/bin/python -c "import sase_github; from sase_github.workspace_plugin import GitHubWorkspacePlugin, resolve_gh_ref, peek_gh_ref; print('ok')"`.
7. Self-review the move:
   `git diff -M --color-moved=zebra --color-moved-ws=allow-indentation-change`. Moved
   blocks should render as moves. Any non-move hunk should be a rename, a delegation, an
   import, or one of the two dead-helper deletions.

Suggested commit subject for the host finalizer:
`refactor(workspace): split workspace_plugin.py into a workspace/ package`

## Out of scope

- Splitting `tests/test_workspace_plugin.py` (2,355 lines) to mirror the new modules. It
  is a good follow-up, but it is a separate change.
- Any deduplication or logic cleanup, such as the repeated `stderr`/`stdout` joining or
  the two branches in `create_sdd_remote`.
- `plugin.py` (the VCS plugin, 535 lines). It is already under the cap.
