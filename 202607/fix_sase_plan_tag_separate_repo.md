---
create_time: 2026-07-11 09:25:08
status: done
prompt: 202607/prompts/fix_sase_plan_tag_separate_repo.md
tier: tale
---
# Fix Missing `SASE_PLAN` Commit Tag for Separate-Repo (Companion) SDD Stores

## Problem

When a sase agent commits after a tale is approved, the `sase commit` command is supposed to append a `SASE_PLAN=<path>`
tag to the commit message so the code commit points back at the approved plan (tale). This stopped happening: recent
agent commits carry `SASE_AGENT=`/`SASE_MACHINE=` tags but no `SASE_PLAN=` tag.

The last tagged commit in this repo is `5e9300e11` (2026-07-08 03:09, `SASE_PLAN=sdd/tales/202607/...`). Every agent
commit after that morning is missing the tag.

## Root Cause

Commit `4637a8aa1` ("refactor(sdd): introduce storage policy resolver (sase-5j.1)", 2026-07-07 23:55) changed the gate
in `handle_sase_plan()` (`src/sase/workflows/commit/commit_hooks.py`) from:

```python
version_controlled = get_effective_sdd_config(cwd)   # True for any version-controlled SDD setup
```

to:

```python
sdd_in_tree = resolve_sdd_store(cwd, 1).is_in_tree   # True ONLY for SDD_STORAGE_IN_TREE
```

Every consequential step in `handle_sase_plan` — copying the plan into the repo, adding frontmatter, **appending the
`SASE_PLAN=` tag**, and staging the plan file — is now gated on `sdd_in_tree`.

Hours later (2026-07-08 07:08 per the workspace's `.sase/sdd-store.json`), the SDD store was migrated to the mandatory
GitHub companion repo model (`storage: separate_repo`, e.g. `sase-org/sase--sdd`, cloned at `.sase/sdd/`). From that
moment `is_in_tree` is always `False`, so `handle_sase_plan` degenerates to only the `sed` "status: wip → done" flip and
the tag is never appended. The timeline matches the observed regression exactly.

Note the tag machinery itself is fine: `update_trailing_commit_tags()` renders canonical key `PLAN` as `SASE_PLAN=`, and
the plan-approval flow (`run_agent_exec_plan_accept.py`) still correctly writes the plan into the companion store via
`write_sdd_files()`, commits it there via `commit_sdd_store_files()`, and exports
`SASE_PLAN=<workspace>/.sase/sdd/tales/<YYYYMM>/<name>.md` for the coder follow-up. Only the commit-hook gating is
wrong.

## Desired Behavior

The `SASE_PLAN` tag should be appended for store-backed (separate-repo/local) SDD storages too, with the value expressed
**relative to the repo that contains the plan file**:

- Companion/local store (`.sase/sdd/`): `SASE_PLAN=tales/<YYYYMM>/<plan_name>.md` (relative to the SDD store root — the
  `sdd/` prefix is intentionally dropped, per user request).
- In-tree store (unchanged): `SASE_PLAN=sdd/tales/<YYYYMM>/<plan_name>.md` (relative to the code repo root, which is
  where the file actually lives in that layout).

(The date shard is the existing `YYYYMM` layout produced by `get_yyyymm()`/`write_sdd_files()`.)

Tag readers are unaffected: `sase vcs log` chip rendering (`sase.vcs_log.tags.commit_tag_view`) treats tag values as
opaque strings, and no code resolves the tag value back to a filesystem path. This change is Python-only commit workflow
glue; no `sase-core` (Rust) changes are needed.

## Design

### 1. Rework `handle_sase_plan()` (`src/sase/workflows/commit/commit_hooks.py`)

Resolve the store once (`store = resolve_sdd_store(cwd, 1)`) and branch on layout instead of skipping everything for
non-in-tree storages:

- **Plan already inside the SDD store** (the standard approved-tale flow — approval already copied + committed it to the
  companion repo): skip the copy, compute the tag value relative to the store root, append the tag, keep the status
  flip. Do NOT set `payload["_plan_path"]` (the plan does not live in the code repo, so there is nothing to stage
  there).
- **Plan not in the store or code repo** (e.g. no-commit approvals hand off the archived `~/.sase/plans/...` copy):
  mirror the old in-tree parity behavior for store-backed storages — copy the plan into
  `<store>/tales/<YYYYMM>/<name>.md` (reuse the existing prettier-format + create-time frontmatter + prompt-link steps,
  with the prompt-link inference searching the store root instead of the code repo root), commit it to the store via the
  existing `commit_sdd_store_files()` helper, then tag with the store-relative path.
- **In-tree storage**: behavior unchanged (copy into `sdd/tales/`, frontmatter, repo-relative tag, `_plan_path`
  staging).

The "mark plan done" `sed` continues to run for all storages. For store-backed storages the resulting dirty file is
already swept into the companion repo by the existing commit-finalizer machinery (`auto_commit_done_sdd_plan_status` /
"chore(sdd): sync uncommitted SDD store changes"), so no new commit step is needed for the status flip itself.

### 2. Store-relative tag helper (`src/sase/workflows/commit/plan_paths.py`)

Add a tag-value helper (e.g. `format_sase_plan_tag_value(plan_path, *, repo_root, store)`) that:

- returns the path relative to `store.sdd_dir` when the plan is under the store root;
- falls back to detecting the `.sase/sdd/` path components (like the existing `_local_sdd_reference`) and stripping
  them, so the tag stays correct even if the exported `SASE_PLAN` points at a sibling workspace's store clone;
- otherwise falls back to the existing repo-relative/`~` logic.

Important: leave `format_sase_plan_reference()` itself untouched — it feeds the ChangeSpec COMMITS-drawer `plan:`
display, where `.sase/sdd/tales/...` must remain a workspace-resolvable path (the ACE TUI expands it to open the file).
Only the git commit tag switches to store-relative form.

### 3. Tests

- New/extended coverage in `tests/workflows/test_commit_workflow.py` (fixtures in `tests/_commit_workflow_fixtures.py`)
  for `handle_sase_plan` with a separate-repo store record:
  - plan inside `.sase/sdd/tales/<YYYYMM>/` → message ends with `SASE_PLAN=tales/<YYYYMM>/<name>.md`, `_plan_path` not
    set, status flipped to done;
  - plan only in the `~/.sase/plans/` archive → copied into the store `tales/` shard, committed to the store repo, and
    tagged store-relative;
  - in-tree storage → existing `SASE_PLAN=sdd/tales/...` + `_plan_path` staging behavior still holds (regression guard).
- Unit tests for the new helper in `tests/workflows/test_plan_paths.py` (store-relative, sibling-workspace fallback,
  non-store fallback).

### 4. Docs

Update the `SASE_PLAN` mentions to describe the storage-dependent tag value:

- `docs/sdd.md` (commit tag paragraph)
- `docs/commit_workflows.md` (plan-handling step + `SASE_PLAN` env var table row)

## Out of Scope

- The hardcoded `workspace_num=1` argument to `resolve_sdd_store()` in the commit hook (resolves correctly via the
  project `WORKSPACE_DIR` / marker lookup; unrelated to this regression).
- Changing the ChangeSpec drawer `plan:` display format.
- Any `sase-core` (Rust) changes — tag values are opaque to all readers.
