---
tier: tale
title: Fix sase repo open tier-1 resolution for display-named projects
goal:
  sase repo open resolves sidecar, linked, and primary repos again in projects whose
  display name differs from their canonical project key, and the unknown-repo error
  lists the real repo names.
size: small
proposed_by: bbugyi200.kellys_mbp.0g
create_time: 2026-09-13 18:30:28
status: wip
---

# Fix `sase repo open` Tier-1 Resolution For Display-Named Projects

## Problem

`sase repo open <name>` fails for every tier-1 target (sidecars, linked repos, and the
primary) in any project whose display name differs from its canonical project key:

```
❯ sase repo open beads -r none
Unknown repo 'beads' for project 'gh_sase-org__sase'. Valid repos: gh_sase-org__sase. To open an external repo, use gh:owner/repo (or owner/repo).
```

This was reported on athena but reproduces identically on every machine (confirmed on
kellys_mbp): it is a code regression, not machine state. `sase repo list` still shows
all repos (including the `beads` sidecar), and `sase repo path beads` still works via
its legacy SDD fallback, but the shared name matcher used by `sase repo open` matches
nothing.

## Root Cause

Commit `2b811499c4` ("fix(repo): route provider aliases to configured checkouts",
2026-09-11) rewrote `match_repo_record` in `src/sase/main/repo_handler_common.py` to
pre-filter inventory records with:

```python
if record.project == host_ctx.project_name
```

The two sides of that comparison hold different name forms:

- `RepoRecord.project` is set by `_collect_project_repos` in
  `src/sase/repo_inventory.py` to `effective_project_name(host)`, i.e. the **display
  name** (`sase`). The raw key is stored separately in `RepoRecord.project_key`
  (`gh_sase-org__sase`).
- `host_ctx.project_name` comes from `resolve_project_context` →
  `infer_project_name_from_cwd` (`src/sase/bead/project_name.py`), which reads the
  workspace checkout marker (`project_name: "sase"`) and then canonicalizes it through
  `_canonicalize_project_ref` / `resolve_project_alias_ref` to the **canonical project
  key** (`gh_sase-org__sase`).

For a project whose display name differs from its key, the filter therefore matches zero
records, tier-1 resolution returns `None`, tier-2 (registered-project match) correctly
excludes the host project, tier-3 (`parse_external_repo_ref("beads")`) fails, and the
user gets the "Unknown repo" error. The error's "Valid repos:" list is built in
`_unknown_repo_error` (`src/sase/main/repo_open_external.py`) with the same broken
`record.project == host_ctx.project_name` filter, which is why it lists only the raw
project key (added unconditionally) and no actual repos.

Before `2b811499c4`, `match_repo_record` matched sidecar/linked records by name/slug
with no project filter at all, so this never mattered. The regression escaped the test
suite because every fixture in `tests/main/repo_handler_helpers.py` uses
`project="demo"` with `project_key="demo"` and `project_name="demo"` — the display name
and the key are identical, so the bad comparison still matches.

## Affected Call Sites

All three comparisons of `RepoRecord.project` against `host_ctx.project_name`:

1. `src/sase/main/repo_handler_common.py` — `match_repo_record` (the `configured` list
   comprehension, currently line ~114). Breaks all tier-1 opens; also used by
   `match_repo_path_record` in `src/sase/main/repo_handler_path.py`.
2. `src/sase/main/repo_handler_common.py` — `_host_checkout_path` (currently line ~391).
   Same mismatch: the primary record is never found, so `_standard_external_path`
   silently degrades during external-collision checks.
3. `src/sase/main/repo_open_external.py` — `_unknown_repo_error` (currently line ~237).
   Produces the misleading "Valid repos: <project-key-only>" message.

## Fix

1. In `src/sase/main/repo_handler_common.py`, add one small module-level predicate:

   ```python
   def record_belongs_to_host_project(
       record: RepoRecord,
       host_ctx: ProjectContext,
   ) -> bool:
       """Match host-project records by canonical key or display name."""

       return host_ctx.project_name in {record.project, record.project_key}
   ```

   `record.project_key` always carries the canonical key
   (`ProjectRecordWire.project_name`) and `record.project` the display name, so checking
   both accepts whichever form the resolved project context holds.

2. Use the predicate at all three sites:
   - `match_repo_record`: replace `record.project == host_ctx.project_name` in the
     `configured` comprehension with `record_belongs_to_host_project(record, host_ctx)`.
   - `_host_checkout_path`: replace `record.project == host_ctx.project_name` with the
     predicate call.
   - `_unknown_repo_error` in `src/sase/main/repo_open_external.py`: import
     `record_belongs_to_host_project` from `sase.main.repo_handler_common` (a top-level
     import is safe; `repo_handler_common` does not import `repo_open_external`) and
     replace `record.project == host_ctx.project_name` with the predicate call.

   Do not simply drop the project filter: `handle_open_command` scopes its inventory to
   one project, but the helpers are shared and must stay correct for multi-project
   inventories.

## Regression Tests

1. Extend `tests/main/repo_handler_helpers.py` (defaults must preserve existing tests):
   - `project_context(tmp_path, *, project_name: str = "demo")` — pass the name through
     to `ProjectContext.project_name`.
   - `repo_record(..., project: str = "demo", project_key: str | None = None)` —
     `project_key` defaults to `project` when not given.

2. Add tests to `tests/main/test_repo_handler_open_resolution.py` modeling the real
   shape: `host_ctx.project_name = "gh_sase-org__sase"` (canonical key) with records
   carrying `project="sase"` (display name) and `project_key="gh_sase-org__sase"`:
   - A display-named sidecar record (name `beads`, slug `sase--beads`) resolves via
     `_match_repo_record("beads", ...)` and via its slug. This test MUST fail before the
     fix and pass after.
   - `open_external_repo("missing", ...)` (mirroring the existing
     `test_unknown_repo_lists_valid_candidates`) lists the display-named project's
     sidecar/linked names in "Valid repos:" instead of only the project key.
   - A display-named variant of
     `test_repo_name_resolution_matches_github_alias_from_linked_origin` (linked record
     with a git origin, matched via `gh:owner/repo`), which also exercises the
     remote-tier path through `_ensure_no_external_collision` → `_host_checkout_path`.

## Verification

- Run the targeted tests first:
  `pytest tests/main/test_repo_handler_open_resolution.py tests/main -k repo`.
- Then run the standard agent gate: `just check`.
- Live-command acceptance: the `sase` CLI is a uv editable install pointing at the
  project's primary checkout, so the running command only heals once this fix lands on
  master and each machine's primary checkout pulls it. Within the implementation
  workspace, the unit tests above are the acceptance mechanism; after landing, verify
  from any sase workspace that `sase repo open beads -r "<reason>"` prints the sidecar
  checkout path and exits 0, and that a bogus name's error lists the real repo names.

## Non-Goals

- `sase repo open beads -p sase` fails earlier with "Project 'sase' has no
  WORKSPACE_DIR" because explicit `-p` values are not alias-canonicalized in
  `resolve_project_context`. That behavior predates this regression and is out of scope.
- Other `record.project == <name>` comparisons outside the repo-open path (e.g.
  `sase/core/artifact_file_protection.py`) are out of scope; they receive their project
  argument from different resolution paths.
