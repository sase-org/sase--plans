---
tier: tale
title: Resolve VCS-backed artifact files against their owning project's repositories
goal:
  Opening a VCS-backed artifact file (Agents tab `a`, Artifacts tab, clipboard, CLI,
  doctor) materializes its bytes from the row's own project's repository regardless of
  the process cwd, so same-named sidecars in other projects (e.g. `research`) can no
  longer shadow it.
size: medium
proposed_by: bbugyi200.apollo.3a
create_time: 2026-09-30 06:41:00
status: wip
---

# Resolve VCS-backed artifact files against their owning project's repositories

## Problem

Pressing `a` on the Agents tab to open an image produced by a SASE agent sometimes fails
with:

> Could not open artifact file: content unavailable for
> `research@aaeb404f…:202609/bob_cli_2o_plan_budget_now_explained/…_infographic.png`

The row is a byte-free, VCS-backed `ArtifactFile` (`path=None`, `vcs_repo="research"`,
`vcs_sha`, `vcs_relpath`, `sha256`, `project="gh_bobs-org__bob-cli"`). Opening it must
materialize the bytes from a git checkout that contains the recorded commit.

## Root cause (confirmed by reproduction)

`sase.core.artifact_file_vcs.materialize_artifact_file(row, repositories=...)` picks the
checkout list by matching `row.vcs_repo` **by bare name** against a caller-supplied
repository list. Every caller builds that list from
`launch_artifact_ref_context(is_home_mode=False)`, which derives the project from the
**process cwd**:

- ACE Agents tab `a` / Artifacts tab / clipboard `Y`:
  `materialize_artifact_file_entries` in
  `src/sase/ace/tui/models/artifact_file_clipboard.py` and `_materialized_path` in
  `src/sase/ace/tui/widgets/artifacts/files_detail.py`.
- `sase artifact` CLI (`src/sase/artifact_cli/references.py::resolved_file_path`),
  prompt resolution
  (`src/sase/artifact_ref_prompt_resolution.py::materialized_artifact_path`), and doctor
  verify (`src/sase/core/artifact_file_doctor.py::verify_artifact_file_index` via
  `_default_repositories`).

The row's own `project` field is ignored. Several projects define a sidecar with the
same name (`research` exists in both `sase` → `sase--research` and `bob-cli` →
`bob-cli--research`). On this machine the TUI process runs with cwd = the `sase` primary
checkout, so `research` resolves to the **sase** research clones, which do not contain
the bob-cli commit → Rust returns `missing` → `OSError("content unavailable for …")`.

Reproduced directly: materializing that row with cwd = sase checkout fails; with cwd =
bob-cli checkout it succeeds. Inverse case: from a non-project cwd,
`collect_repo_inventory()` (no project filter) sorts `bob-cli` before `sase`, so
`research` resolves to bob-cli and **sase** research artifacts break instead. Which rows
fail depends on where the TUI was launched, not on the row.

Why "only sometimes": the Rust materializer (`sase_core::artifact_file::vcs`) checks a
content-addressed, sha256-verified cache
(`~/.sase/artifacts/vcs-cache/<aa>/<sha256><suffix>`) before touching any checkout. A
row that was materialized once from a correct context opens from any context afterwards.
Only cross-project rows never materialized from their own project fail. Additionally,
when **no** repository of that name exists in the cwd project, the Python bridge returns
`None` before asking Rust, so even a valid cache entry is never consulted.

The Rust resolver/materializer contract is correct: the `file:` ref resolver only
returns a locator, and `artifact_file_materialize_vcs` takes explicit `checkout_paths`.
Checkout selection lives entirely in the Python bridge, alongside the Python repository
inventory (`sase.repo_inventory.collect_repo_inventory`). This fix is Python-only; **no
sase-core change and no `sase-core-revision.txt` bump**.

## Fix

Make VCS materialization resolve `row.vcs_repo` against the row's **owning project**
inventory first. Keep the caller-supplied (cwd-derived) repositories only as a fallback
for legacy rows. Every caller benefits because the change is centralized in the bridge.

### 1. Lean per-project repository helper (`src/sase/artifact_ref_context.py`)

- Extract the inline
  `repositories = tuple(ArtifactRefRepository(...) for record in repository_records)`
  block of `artifact_ref_context()` into a public helper, e.g.
  `artifact_ref_repositories(*, project: str | None, workspace_num: int) -> tuple[ArtifactRefRepository, ...]`.
  It calls `collect_repo_inventory(project=project)` and reuses
  `_repository_checkout_paths` / `_repository_checkout_path`.
- `artifact_ref_context()` must call the new helper and keep its current best-effort
  behavior (log at debug and return `()` on inventory failure). Its output must not
  change.
- The new helper must let callers distinguish "project not found / inventory failed"
  from success. Either raise and let the bridge catch, or return `None` for failure.
  Pick one and document it in the docstring.
- Export it in `__all__`.
- This lookup costs about 0.1s per project, cheaper than the full launch context (about
  0.4s). Do not build the full `artifact_ref_context` (SDD store, bead stores, sidecar
  policies) just to get repositories.

### 2. Owner-first resolution in the bridge (`src/sase/core/artifact_file_vcs.py`)

- Add a small resolver that maps a row to checkout paths, memoized per
  `(project, workspace_num)` for the resolver's lifetime (per call or batch, **not** a
  process-global cache, so long-lived TUI processes pick up new clones), e.g.
  `ArtifactFileRepositoryResolver(fallback=...)`. `fallback` may be an iterable of
  repositories or a zero-arg callable returning one. Evaluate it lazily and at most
  once, so callers that normally resolve by owner never pay for
  `launch_artifact_ref_context`.
- Resolution order for a VCS-backed row:
  1. If `row.project` is set, look up that project's repositories (lazy import of
     `sase.artifact_ref_context.artifact_ref_repositories` inside the function to avoid
     a `sase.core` → `sase` import cycle, matching the existing lazy import in
     `artifact_file_doctor._default_repositories`). Match `row.vcs_repo` against `name`
     or `aliases` using the existing `_repository_for_name`. Derive the workspace number
     used for checkout ordering from the producing workspace: read the checkout marker
     at `row.workspace_dir` via `sase.workspace_provider.marker.find_marker_from_cwd` /
     `read_marker`, and use `marker.workspace_num` when the marker's project matches.
     Otherwise use `0`, the primary. This only orders candidates; all existing clones
     are still tried.
  2. If the row has no project, the project is unknown
     (`RepoInventoryProjectNotFoundError` or any inventory failure), or that project has
     no repository of that name, use the fallback repositories (today's behavior).
- `materialize_artifact_file(row, *, repositories=(), resolver=None)` keeps its current
  call shape so existing callers and tests keep working. When `resolver` is `None`,
  build a one-shot resolver whose fallback is `repositories`.
- When no repository resolves, **still invoke the Rust binding with an empty
  `checkout_paths` list**. The verified content cache can then answer (`cached`), and
  otherwise the result is `missing`. Do not return `None` before consulting the cache.
- On a `missing` result, `log.debug` the row id, owner project, and the Rust-reported
  `checkouts_tried` to aid future diagnosis.

### 3. Callers

- `src/sase/ace/tui/models/artifact_file_clipboard.py::materialize_artifact_file_entries`:
  build **one** resolver for the batch with a lazy fallback
  `lambda: launch_artifact_ref_context(is_home_mode=False).repositories`, and pass it to
  every `materialize_artifact_file` call. Include the owning project in the `OSError`
  locator when present, e.g.
  `content unavailable for research@<sha>:<relpath> (project gh_bobs-org__bob-cli)`, so
  the toast says which repository was searched.
- `src/sase/ace/tui/widgets/artifacts/files_detail.py::_materialized_path`: use the same
  resolver pattern with a lazy launch-context fallback.
- `src/sase/core/artifact_file_doctor.py::verify_artifact_file_index`: create one
  resolver per verify run, with the explicit `repositories` argument or lazy
  `_default_repositories` as fallback, so multi-project indexes verify each row against
  its own project. Today every foreign-project VCS row is reported as unresolvable.
- `artifact_cli/references.py` and `artifact_ref_prompt_resolution.py` need no signature
  change. They get owner-first resolution through `materialize_artifact_file`. Confirm
  by reading them.

## Tests

Use the real-git helpers pattern (`tests/ace/tui/_artifact_file_vcs_fixtures.py`,
`tests/artifact_file_facade/test_vcs.py`). Monkeypatch the owner-project lookup seam,
not `collect_repo_inventory` internals where avoidable.

- `tests/artifact_file_facade/test_vcs.py`:
  - **Regression**: two scratch repos both registered under the name `research` (project
    A lacks the commit, project B has it). Row has `project="B"`. Caller `repositories`
    = A's `research`. The row materializes the exact bytes from B.
  - Rows without `project` still use caller repositories (existing tests cover; keep
    them passing).
  - An unknown owner project (lookup raises / not found) falls back to caller
    repositories.
  - An owner project that has no repo of that name falls back to caller repositories.
  - With no repository resolvable at all, a pre-populated verified cache entry is still
    returned. A fake binding or real cache path must show the binding is called with
    empty `checkout_paths`.
  - The resolver memoizes: two rows from the same project trigger one owner lookup, and
    a lazy callable fallback is never invoked when every row resolves by owner.
- `tests/artifact_file_facade/test_doctor.py`: verify resolves a VCS row whose project
  differs from the explicit/fallback repositories.
- `tests/ace/tui/test_artifact_file_vcs_clipboard.py` or
  `tests/ace/tui/actions/test_artifact_file_vcs_open.py`: a real-bytes regression
  through `materialize_artifact_file_entries`. The launch-context repositories point at
  the wrong same-named repo, and the row's owning project resolves to the right one. It
  materializes, and the `OSError` message includes the project when it genuinely fails.
- Keep `install_materialization_context` working for existing tests. Rows it creates
  have no `project`, so they exercise the fallback path. Extend the fixture only if
  needed.

## Verification

- Run `just check` per the `lint_and_test` memory note. Run only what that note
  prescribes. Do not run `just check-full` unless instructed.
- Manual smoke (optional, from the sase checkout, after `just check`): with cwd = the
  sase primary checkout and `SASE_*_WORKSPACE_NUM` unset, call
  `materialize_artifact_file_entries` on a cross-project VCS row from
  `~/.sase/artifacts/index.jsonl`. Use a row whose `project` differs from the cwd
  project and whose sha256 has no entry under `~/.sase/artifacts/vcs-cache/`. It should
  materialize instead of raising. Do not delete existing cache entries to force this.

## Out of scope

- Moving repository inventory into sase-core.
- Changing the Rust materializer, its cache layout, or the artifact index schema.
- Renaming sidecars to avoid name collisions. Rows already record `project`, which is
  sufficient to disambiguate.
