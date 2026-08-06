---
tier: tale
title: Resolve Artifacts project scope through project refs, not raw ProjectSpec keys
goal:
  Every Artifacts pane resolves its project scope for `#gh` projects whose ProjectSpec key differs from the configured
  project name, so the Beads pane stops reporting "No bead store is available for this project."
proposed_by: bbugyi200.athena.tz
create_time: 2026-08-06 09:11:09
status: done
---

# Plan: Resolve the Artifacts project scope through project refs

## Problem

The Beads sub-tab of the Artifacts tab renders

```
! sase: No bead store is available for this project.
0/0 tasks · 0/0 epics · 0/0 phases · Load errors: sase
```

for the `sase` project, which definitely has a bead store at `sase/repos/beads` in its primary repo.

## Root cause

`ACEApp.artifacts_project_scope` carries a **ProjectSpec directory key** (`gh_sase-org__sase`) by contract, but one of
its two producers seeds it with a **user-facing project ref** (`sase`).

**Consumers all key off the ProjectSpec key.** Verified in this workspace:

```
resolve_projects('gh_sase-org__sase')
  -> (PlansProject(project='gh_sase-org__sase', display_name='sase',
                   workspace_dir='/home/bryan/projects/github/sase-org/sase/'),)
project_beads_dir('gh_sase-org__sase')
  -> /home/bryan/projects/github/sase-org/sase/sase/repos/beads

resolve_projects('sase')
  -> (PlansProject(project='sase', display_name='sase', workspace_dir=None),)   # synthetic fallback
project_beads_dir('sase')
  -> None                                                                       # -> the error banner
```

The same holds for the artifact-file index, which stores keys:

```
query_artifact_files(project='sase')               -> 0 rows
query_artifact_files(project='gh_sase-org__sase')  -> rows, project='gh_sase-org__sase'
```

**Producer 1 (correct).** The project picker sets the scope from `InventoryProjectChoice.project_key`, i.e.
`record.project_name`, the key — `src/sase/ace/tui/actions/artifacts.py:301`. The single-enabled-project auto-scope also
uses keys (`enabled_projects` is built from `record.project_name` at `src/sase/ace/tui/actions/artifacts.py:164`).

**Producer 2 (defective).** Startup seeds the scope straight from the parsed top-level query token —
`src/sase/ace/tui/actions/_state_init.py:184`:

```python
self.artifacts_project_scope: str | None = get_sole_project_filter(
    self.parsed_query
)
```

`get_sole_project_filter` returns the literal `project:` token value (`src/sase/ace/query/introspection.py:119-150`).
Per the repo's "Show Project Names, Never ProjectSpec Keys" convention, query tokens hold the user-facing project name,
and the ChangeSpecs matcher compares them against `changespec.project_query_name`
(`src/sase/ace/query/matchers.py:56-67`), so `project:sase` is a perfectly valid query — it just is not a key.

The user's persisted query is exactly that. `~/.sase/last_query.txt` contains `project:sase`, and
`~/.sase/query_history.json` shows both forms have been used (`project:sase` and, earlier, `project:gh_sase-org__sase` —
which is why the pane used to work).

**Why it never self-heals.** `_ensure_artifacts_project_choices` loads the project inventory off-thread on the first
non-PRs Artifacts pane activation (`src/sase/ace/tui/actions/artifacts_navigation.py:66`), and its completion handler
re-applies the scope — but re-applies the _same unnormalized value_ (`src/sase/ace/tui/actions/artifacts.py:266-269`):

```python
else:
    self._set_artifacts_project_scope(
        self.artifacts_project_scope,   # still 'sase'
        picked=False,
    )
```

Note that this handler already holds `result.project_ref_display`, a `ProjectRefDisplaySnapshot` whose
`project_key_for_ref` does exactly the required mapping. Verified against live records:

```
project_key_for_ref('sase')              -> 'gh_sase-org__sase'
project_key_for_ref('gh_sase-org__sase') -> 'gh_sase-org__sase'
project_key_for_ref('SASE')              -> 'gh_sase-org__sase'   (casefolded)
project_key_for_ref('nope')              -> None
```

**Why this went unnoticed.** For `#git` projects the directory key equals the project name, so key and ref are the same
string and every consumer works. The divergence only appears for `#gh` projects, where the key is `gh_<org>__<repo>` and
the name is the repo's `PROJECT_NAME:`. The defect has existed since the Artifacts tab was scaffolded (`a62647069`); it
only became reachable once query tokens started carrying display names.

### Blast radius

The reported Beads failure is one symptom of a scope contract violation that degrades most of the Artifacts tab whenever
the startup query names a `#gh` project by its display name:

| Site                                                                                   | Behavior under a ref-valued scope                                                                                                                                          |
| -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `widgets/artifacts/beads_data.py:65-68` (via `resolve_projects` / `project_beads_dir`) | Hard error banner: "No bead store is available for this project." — **the reported symptom**                                                                               |
| `widgets/artifacts/plans_data.py:120-122`, `:170-177`                                  | Same error, plus "No document sidecar is available for this project." (the synthetic `PlansProject` has `workspace_dir=None`, so `project_document_roots` returns `{}`)    |
| `widgets/artifacts/files_data.py:89` -> `query_artifact_files(project=...)`            | Silent empty pane — the index stores keys                                                                                                                                  |
| `tui/artifacts_bugs.py:244-251` `_load_project_beads`                                  | Silent empty local bug links: `get_project_beads_dirs_for_project(ref)` returns `None`, `or []` swallows it, and the error string stays empty                              |
| `tui/artifacts_bugs.py:83-108` `_resolve_bug_scope`                                    | **Already correct** — it matches the scope against `{project_name, effective_project_name, *aliases}` casefolded. This is the precedent the rest of the tab should follow. |

## Fix

Normalize at the producer, and harden the loader that surfaces the reported error. Do not change the query token itself:
`project:sase` is correct and must keep working.

### 1. Normalize the query-derived scope once the inventory is known

In `_ensure_artifacts_project_choices`'s completion handler (`src/sase/ace/tui/actions/artifacts.py`, the `else` branch
around lines 266-269), resolve the current scope through the snapshot already in hand:

```python
else:
    scope = self.artifacts_project_scope
    normalized = result.project_ref_display.project_key_for_ref(scope)
    self._set_artifacts_project_scope(normalized or scope, picked=False)
```

Requirements:

- Fall back to the raw ref when `project_key_for_ref` returns `None` (unknown, ambiguous, or typo'd project). Do **not**
  fall back to `None`: that would silently widen the scope to all projects and hide the mistake. Keeping the raw ref
  preserves today's explicit "no store for this project" diagnostic.
- Normalizing changes `artifacts_project_scope`, so `_set_artifacts_project_scope` takes its
  `project != self.artifacts_project_scope` branch and calls `_clear_all_artifacts_marks()`. That is acceptable and
  intended: this fires once, during the first Artifacts activation that triggered the inventory load, before the user
  can have marked anything. It is also what makes the panes see `changed=True` and reload with the corrected scope.

This single change fixes the Beads, Plans, Files, and Bugs panes together, because they all read the shared scope.

### 2. Make `resolve_projects` ref-tolerant (defense in depth)

`resolve_projects` in `src/sase/ace/tui/widgets/artifacts/plans_data_sources.py:25-60` currently matches only
`record.project_name == project`. Widen the match to key, configured label, and aliases — mirroring the
`_resolve_bug_scope` precedent — using the record batch it has already loaded:

```python
ProjectRefDisplaySnapshot.from_records(records).project_key_for_ref(project)
```

Then select the record whose `project_name` equals that key. Keep the existing synthetic
`PlansProject(project, project, None)` fallback for refs that resolve to nothing, so an unknown project still produces
today's clear error rather than an empty pane.

This costs no extra disk IO (the records are already in memory) and makes the Beads and Plans loaders correct for any
caller, including a scope that reaches them before normalization lands.

### 3. Resolve the ref in the Bugs pane's bead-link lookup

`_load_project_beads` in `src/sase/ace/tui/artifacts_bugs.py:244-251` passes the raw scope to the key-only
`get_project_beads_dirs_for_project`. Resolve the ref to a key first (reusing the same helper `_resolve_bug_scope`
already proves out), so bug rows keep their local bead links. Preserve the existing degraded-but-usable behavior: a
missing store must not raise.

### Rust core boundary

No `../sase-core` change is required. The ref-to-key projection already lives in this repo as
`ProjectRefDisplaySnapshot` in `src/sase/project_display_names.py`, built from wire records that the Rust-backed
`sase.core.project_lifecycle_facade` already supplies. This plan reuses that projection rather than reimplementing
identity resolution, which keeps the change on the presentation/adapter side of the boundary. Whether ref resolution
should eventually move into `sase_core` is a separate question and is explicitly out of scope here.

## Tests

Add regression coverage that would fail today. Every new test must use a project whose key differs from its name (e.g.
key `gh_acme__widget`, name `widget`) — a `#git`-style fixture where key equals name cannot catch this bug.

- `tests/ace/tui/test_artifacts_plans_data.py` — `resolve_projects` returns the real record (with its `workspace_dir`)
  when scoped by display name, by alias, and by a differently-cased ref; still returns the synthetic fallback for an
  unknown ref.
- `tests/ace/tui/test_artifacts_beads_loading.py` — `load_beads_snapshot` scoped by display name loads beads and reports
  **no** entry in `snapshot.errors`. Assert the absence of the "No bead store is available for this project." message
  specifically, so the test names the reported symptom.
- `tests/ace/tui/test_artifacts_plans_loading.py` — the same scope produces neither the bead-store error nor the "No
  document sidecar is available" error.
- `tests/ace/tui/test_artifacts_scaffold.py` (or the nearest home for `artifacts_project_scope` behavior) — after the
  project-choices load completes, a scope seeded from a display-name query token has been normalized to the ProjectSpec
  key, and an unresolvable ref is left untouched rather than reset to `None`.
- `tests/ace/tui/test_artifacts_bugs.py` — a display-name scope still resolves local bead links.

Existing tests patch `_resolve_projects` at `tests/ace/tui/test_artifacts_beads_loading.py:50,98`,
`tests/ace/tui/test_artifacts_plans_data.py:79,128`, and `tests/ace/tui/test_artifacts_plans_loading.py:49`; extend
those seams rather than inventing new ones.

## Verification

1. `just install` first — this is an ephemeral workspace clone.
2. `just check`. If the scoped test lane escalates or selects unusually, run `just check-full`.
3. Manual confirmation against the real symptom: with `~/.sase/last_query.txt` holding `project:sase`, launch
   `sase ace`, open Artifacts -> Beads, and confirm the pane lists the `sase` project's beads with no "Load errors"
   line. Check the Plans and Files panes in the same session.

## Out of scope

- Changing what `project:` query tokens contain, or rewriting the persisted query. Display-name tokens are the
  documented convention and must keep working.
- Migrating ref resolution into `sase_core`.
- Any change to the Beads pane's own filters (`-status:closed` and friends were behaving correctly; the pane had zero
  beads to filter).
