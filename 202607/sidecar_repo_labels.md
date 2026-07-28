---
tier: tale
title: Label sidecar commits and deltas by sidecar role, not `sdd`
goal:
  The ACE ARTIFACTS section names sidecar commit and delta groups by their sidecar role (plans, research, beads),
  matching every other repo surface, and the `sdd` literal no longer appears as a repository label.
create_time: 2026-07-28 13:18:38
status: wip
---

- **PROMPT:** [202607/prompts/sidecar_repo_labels.md](prompts/sidecar_repo_labels.md)

# Label sidecar commits/deltas by sidecar role, not `sdd`

## Problem

The ACE Agents-tab ARTIFACTS section groups an agent's commits and deltas by source repository. Bead-store commits
(`chore(beads): claim …`, `chore(beads): close …`) and their deltas (`issues.jsonl`, `events/streams/*.jsonl`) render
under a group named `sdd`.

That is wrong three separate ways:

1. **Wrong repo identity.** Those commits land in the beads sidecar (`<org>/<project>--beads`), not in anything named
   `sdd`.
2. **Dead terminology.** `sdd` is the deprecated "Spec-Driven Development" label that is being removed from the
   codebase.
3. **Inconsistent vocabulary.** The same three repos have three different spellings today:
   - `sase repo list` / `sase repo open` / the `/sase_repo` skill call them `plans`, `research`, `beads`.
   - ACE's own cwd-derived fallback (`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py:300`) also produces
     `plans` / `research` / `beads`.
   - The persisted-marker path produces `<org>/<project>--plans`, `<org>/<project>--research`, and — for beads — `sdd`.

   Which spelling a group gets depends on which code path happened to fire, so the same repository can appear under two
   different names in the same panel.

## Root cause

Verified by resolving the store in a live numbered workspace: `sdd_store_label()` returns `<org>/<project>--plans` for
plans, `<org>/<project>--research` for research, and the literal `sdd` for beads.

The chain:

1. `src/sase/sdd/_commit_store.py:165` — `sdd_commit_targets()` builds the beads target store with `remote_url=None`.
   The plans target keeps `store.remote_url` and the research target is given `store.research_remote_url`
   (`_commit_store.py:158`), but `SddStore` has no `beads_remote_url` field at all (`src/sase/sdd/_store_types.py:70`),
   so there is nothing to hand the beads target.
2. `sdd_store_label()` (`src/sase/sdd/_commit_store.py:293`) therefore skips its `remote_url` branch for beads and falls
   through to `_sdd_store_record_label()`.
3. `_sdd_store_record_workspace_dir()` (`src/sase/sdd/_commit_store.py:340`) derives a workspace directory from the
   clone path: for `<workspace>/sase/repos/beads` it returns the **numbered agent workspace**. But the store record only
   ever lives in the **primary checkout** — `_sdd_store_record_path()` (`src/sase/sdd/_store_records.py:110`) is
   documented to take `primary_workspace_dir`, and `_resolve_sdd_storage()` (`src/sase/sdd/_store_resolution.py:159`)
   reads it through `get_primary_workspace_dir()`. A numbered workspace has no `.sase/sdd-store.json`, so the read
   returns `None`.

   This miss happens for **all three** kinds, not just beads. Plans and research are simply masked by their `remote_url`
   short-circuit in step 2; beads has no such cover, so it is the one that exposes the defect.

4. `sdd_store_label()` falls through to its terminal `return "sdd"` (`src/sase/sdd/_commit_store.py:307`).
5. The commit marker is persisted with `repo_name: "sdd"` — and `record_sdd_commit_result_marker()` carries the same
   literal default at `src/sase/workflows/commit/commit_tracking.py:490`.
6. ACE renders the explicit marker name in both surfaces: the Commits group
   (`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py:364`, where the explicit name overrides the correct
   cwd-derived role) and the Deltas group (`src/sase/ace/tui/widgets/prompt_panel/_agent_deltas.py:291`).

There is a related latent bug in the same family: `_dirty_sdd_store_repos()`
(`src/sase/llm_provider/commit_finalizer_state.py:155`) picks its fallback name by list index — `index == 1` means
`research`, anything else means `plans` — so the beads target at index 2 would be labelled `plans`. It is currently
invisible only because `sdd_store_label()` never returns `None` for sidecar storage.

### Why this shipped green

`tests/test_sdd_commit_store.py:223` (`test_split_beads_auto_commit_marker_uses_beads_repo_name`) already asserts the
intended behaviour — that a beads commit marker names the beads repo. It passes because it writes the store record into
the same directory that `_sdd_store_record_workspace_dir()` derives, i.e. it models a topology where the numbered
workspace _is_ the primary checkout. `tests/test_commit_artifacts.py:329` and `tests/test_commit_artifacts.py:275` do
the same. Real agents never run in that topology, so the tests encode the intent but not the production layout.

## Decision

**Sidecar commit and delta groups render the sidecar role: `plans`, `research`, `beads`.**

Rationale:

- It matches every other repo-facing surface: `sase repo list`, `sase repo open`, the `/sase_repo` skill, the sidecar
  role defaults in `src/sase/_linked_repo_config.py:239`, ACE's own cwd fallback, and what
  `commit_finalizer_state.py:158` was already reaching for.
- It removes the `sdd` literal from a user-visible surface.
- It is unambiguous in context: the ACE panel is scoped to one agent in one project, and the primary group already shows
  the project name.
- External repos keep their `owner/repo` form, which preserves the existing visual distinction between sidecars and
  external clones.

This does change the plans and research groups from `<org>/<project>--plans` / `<org>/<project>--research` to `plans` /
`research`. That is deliberate — leaving them as remote slugs while beads renders as `beads` would just trade one
inconsistency for another.

**The legacy `separate_repo` store keeps its current labelling** (`remote_url` → record `repo` → `sdd`). That store
really is a single repo named `<project>--sdd` holding all kinds, so `sdd` is its actual name there. Renaming the
`--sdd` sidecar suffix belongs to the wider SDD-terminology removal, not here.

## Changes

### 1. Carry the sidecar role on derived target stores

- `src/sase/sdd/_store_types.py` — add `sidecar_role: str | None = None` to the `SddStore` dataclass, alongside
  `research_dir` / `beads_dir`.
- `src/sase/sdd/_commit_store.py` — in `sdd_commit_targets()`, set `sidecar_role="plans"`, `"research"`, and `"beads"`
  on the three derived target stores. Non-sidecar storage leaves it `None`.

### 2. Derive the label from the role

- `src/sase/sdd/_commit_store.py` — in `sdd_store_label()`, when `store.storage == SDD_STORAGE_SIDECAR_REPOS` return
  `store.sidecar_role`. Keep the existing `separate_repo` branch (remote URL → record `repo` → `"sdd"`) untouched, so
  `_repo_label_from_remote_url()` stays live and Symvision does not flag it as unused.

### 3. Stop the store-record lookup from silently missing

- `src/sase/sdd/_commit_store.py` — `_sdd_store_record_workspace_dir()` currently returns the workspace it derived from
  the clone path, which is the numbered workspace. Resolve the primary from it via `get_primary_workspace_dir()` before
  calling `read_sdd_store_record()`, while keeping the _derived_ (numbered) workspace for the `sidecar_repo_clone_dir()`
  comparison inside `_sdd_store_record_label()` — the clones live under the numbered workspace, the record lives in the
  primary.

  Steps 1–2 mean this no longer affects the beads label, but the same silent miss is what produced the bug and it still
  governs the `separate_repo` path. Fix it so the class of failure does not reappear.

### 4. Fix the primary-workflow marker path

- `src/sase/workflows/commit/commit_tracking.py:394` — `_sdd_repo_name_for_commit_cwd()` should return the role `kind`
  directly for a sidecar clone rather than preferring `sidecar.repo`, which also removes its dependence on the record
  read for the sidecar case. Resolve the primary before `read_sdd_store_record()` for the remaining legacy branch.
- `src/sase/workflows/commit/commit_tracking.py:490` — drop the `or "sdd"` default in
  `record_sdd_commit_result_marker()`. Default to the store root's directory name instead, so an unresolvable repo
  degrades to something true rather than to dead terminology.

### 5. Remove the index-based fallback in the commit finalizer

- `src/sase/llm_provider/commit_finalizer_state.py:155` — replace the `index == 1` heuristic with
  `target_store.sidecar_role`, which is now authoritative.

### 6. Normalize already-persisted `sdd` markers on the ACE display path

Markers written before this change are on disk and keep `repo_name: "sdd"`, so the Agents tab would keep showing the
wrong name for every completed agent still in the list.

- `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py:352` — in `_repo_attribution_for_commit_record()`, when the
  explicit `repo_name` is exactly the legacy placeholder `"sdd"` **and** the cwd resolves inside a sidecar clone, prefer
  the cwd-derived role over the explicit name. Key the normalization to that literal only, so an explicit
  `<org>/<project>--sdd` from a real `separate_repo` store is still honoured.
- `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py:300` — reuse `SDD_CANONICAL_DIRS`
  (`src/sase/sdd/_paths.py:8`) instead of the inline `("plans", "research", "beads")` tuple, so the kind list has one
  definition.

This fixes history without a data migration, and the Deltas groups follow automatically because
`agent_commit_linked_delta_groups()` builds them from the same `CommitDiffInfo.repo_name`.

### 7. Tests

- Add a shared test helper that builds the **real** topology: a primary checkout holding `.sase/sdd-store.json`, plus a
  separate numbered workspace whose `.sase/checkout.json` records `primary_workspace_dir`. This is the missing fixture
  that let the bug ship.
- Rewrite `tests/test_sdd_commit_store.py:223` to use that topology and expect `beads`.
- Add a regression test in `tests/test_sdd_commit_store.py`: with the record present **only** in the primary checkout,
  `sdd_store_label()` returns `plans` / `research` / `beads` for the three targets and never `"sdd"`. This is the
  assertion that fails on today's code.
- Update `tests/test_commit_artifacts.py:275` and `tests/test_commit_artifacts.py:329` to the real topology and the role
  names.
- Add an ACE test in `tests/ace/tui/widgets/test_agent_display_commit_metadata.py`: a marker with `repo_name: "sdd"` and
  cwd `<workspace>/sase/repos/beads` renders the group as `beads`.
- Keep `tests/ace/tui/widgets/test_agent_display_commit_metadata.py:321`
  (`test_meta_commits_record_repo_name_overrides_cwd_group`, legacy `.sase/sdd` cwd with an explicit
  `<org>/<project>--sdd`) passing unchanged — that is the case the narrow normalization must not swallow.
- Review and update as needed: `tests/test_sdd_commit.py:152`, `tests/test_sdd_commit.py:202`,
  `tests/test_done_agent_loader.py:451`, `tests/llm_provider/test_commit_finalizer_auto_sdd_status.py:330`,
  `tests/ace/tui/widgets/test_linked_deltas.py:344`.

## Verification

1. `just install` then `just check` — this workspace may be stale, so install first.
2. Resolve the store in a numbered workspace and assert every sidecar target labels as its role:

   ```python
   from pathlib import Path
   from sase.sdd.store import resolve_sdd_store
   from sase.sdd._commit_store import sdd_commit_targets, sdd_store_label

   store = resolve_sdd_store(str(Path.cwd()), <workspace_num>)
   print([sdd_store_label(t) for t, _ in sdd_commit_targets(store, None)])
   # expected: ['plans', 'research', 'beads']   (today: [..., ..., 'sdd'])
   ```

3. In `sase ace`, select an agent that made bead commits and confirm the ARTIFACTS Commits and Deltas groups both read
   `beads`. Confirm a previously-completed agent (whose marker still says `sdd` on disk) also now renders `beads`.

## Out of scope

The wider SDD-terminology removal: renaming the `src/sase/sdd/` package, `.sase/sdd-store.json`, the `SASE_SDD_*`
environment variables, the `<owner>/<repo>--sdd` sidecar suffix, the in-tree `sdd/` directory, the `Sdd*Wire` types, and
`docs/sdd.md`. This plan changes only what the ACE ARTIFACTS section shows a user, plus the marker-writing paths that
feed it.
