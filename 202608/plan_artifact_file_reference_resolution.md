---
tier: tale
title: "Resolve logical plans: references when synthesizing plan artifact files"
goal:
  Synthesized plan artifact files always carry a real filesystem path, so the agent metadata panel stops falsely marking
  an existing epic plan as missing and stops rendering it as a duplicate row.
proposed_by: bbugyi200.athena.th
create_time: 2026-08-05 17:11:33
status: done
---

- **PROMPT:**
  [prompts/202608/plan_artifact_file_reference_resolution.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/plan_artifact_file_reference_resolution.md)

# Plan: Resolve logical `plans:` references when synthesizing plan artifact files

## Problem

When a phase agent is selected in the ACE Agents tab, the metadata panel renders the agent's epic plan twice, and the
second rendering is wrong:

```
  Epic Plan: plans:202608/bead_close_publication_loss.md      <- correct, resolvable link
  ...
  ARTIFACTS · 7 files · 1 artifact file
    Files:
      ▤ plans:202608/bead_close_publication_loss.md (missing)  <- wrong: file exists, and this row
                                                                  should not be rendered at all
```

The plan file exists at `~/.sase/plans/202608/bead_close_publication_loss.md`. The `(missing)` marker is false, and the
row is a duplicate of the `Epic Plan:` line directly above it.

## Root cause

`ArtifactFile.path` is consumed everywhere as a **filesystem path**. For plan rows it can instead hold a **logical plan
reference** (`plans:<month>/<name>.md`), and nothing resolves it.

The chain:

1. Agents launched against an epic plan persist a logical reference in `agent_meta.json`:
   `"sdd_plan_path": "plans:202608/bead_close_publication_loss.md"`. This is the canonical persisted form introduced by
   the `sase-9z` plan-reference unification epic (`b3a4bc282 fix(bead): persist canonical plan references`), and it is
   correct on its own terms — `plans:` references are resolved against the plan roots returned by
   `sase.sdd.plan_refs.resolve_plan_roots()`, namely the active SDD store plans root and the machine-local
   `~/.sase/plans` archive.
2. `sase.core.artifact_file_helpers.selected_plan_path()` reads `sdd_plan_path` and hands it to
   `select_canonical_plan_path()`. With `plan_committed: true` that function returns `sdd_plan_path` verbatim — still a
   logical reference, never resolved.
3. `sase.core.artifact_file_defaults.synthesize_default_artifact_files()` builds the plan row directly from that value:
   `ArtifactFile(kind="plan", path="plans:202608/bead_close_publication_loss.md")`. The logical reference is now sitting
   in a field every downstream consumer treats as a filesystem path.
4. `sase/ace/tui/widgets/prompt_panel/_artifact_files.py::_resolve_actual_path()` sees a non-absolute string and joins
   it onto the agent's workspace directory, producing `<workspace>/plans:202608/bead_close_publication_loss.md`.
   `os.path.exists()` is `False`, so `append_artifact_file_paths()` appends the `(missing)` suffix.

Verified against the live artifact directory for the affected agent
(`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/05/20260805154735`):

```
kind='plan' path='plans:202608/bead_close_publication_loss.md' source_path=None is_vcs_backed=False
_resolve_actual_path(...) -> <workspace>/plans:202608/bead_close_publication_loss.md   exists=False
```

while the shared resolver handles the same reference correctly:

```
resolve_plan_roots(...)     -> (<workspace>/sase/repos/plans, ~/.sase/plans)
resolve_plan_reference(...) -> status='exact' resolved=~/.sase/plans/202608/bead_close_publication_loss.md
```

### Why the row should not be rendered at all

`sase/ace/tui/widgets/prompt_panel/_agent_display_header_summary.py` already contains a filter whose entire purpose is
to suppress the plan artifact-file row when the plan is already shown in the plan/bead section:

```python
resolved_artifact_file_paths = resolve_artifact_file_paths(agent)
if plan_enrichment.resolved_plan_paths:
    plan_paths = {Path(path).resolve(strict=False) for path in plan_enrichment.resolved_plan_paths}
    resolved_artifact_file_paths = [
        artifact_file
        for artifact_file in resolved_artifact_file_paths
        if Path(artifact_file.actual_path).resolve(strict=False) not in plan_paths
    ]
```

`plan_enrichment.resolved_plan_paths` is produced by `sase/ace/tui/models/_agent_associated_plan_paths.py`, which _does_
route through the shared resolver, so it holds the real absolute path. The artifact row's `actual_path` holds the bogus
workspace-joined string. The two never compare equal, so the filter silently fails and the duplicate row survives.

Fixing the root cause therefore fixes both symptoms at once: the row's `actual_path` becomes the real plan path, the
existence check passes, and the dedup filter then removes the row so only the `Epic Plan:` line remains.

### Collateral damage from the same defect

All confirmed by running against the affected agent's artifact directory:

- `created_at`, `mime_type`, `sha256`, `size_bytes` are all `None` — `file_created_at()` stats a path that cannot exist.
- `artifact_file_clipboard_path()` returns
  `ArtifactFilePathCopy(text='~/.../<workspace>/plans:202608/bead_close_publication_loss.md', origin='stored', missing=False)`.
  Copying the path yields a nonexistent path **and** wrongly reports `missing=False`.
- `append_artifact_file_paths()` only assigns an `[N]` open hint when `exists` is true, so the row cannot be opened from
  the panel.
- `artifact_file_dedupe_key()` resolves the reference against the _current process CWD_, so the dedupe key for this row
  differs depending on where the reader runs.

### Scope boundaries (checked, deliberately out of scope)

- **Archived plan paths are fine.** `plan_path.json` and `done.json`/`agent_meta.json` `plan_path` entries hold absolute
  paths (e.g. `{"plan_path": "/home/bryan/.sase/plans/202608/stored_prompt_duality.md"}`). Only `sdd_plan_path` carries
  logical references.
- **The Artifacts → Files subtab is unaffected.** It renders `query_artifact_files()`, which reads only the JSONL index;
  synthesized plan rows are never written there. Confirmed: no `plans:` string appears anywhere in
  `~/.sase/artifacts/index.jsonl`, and that agent has no rows in it.
- **VCS-backed rows are unaffected.** `is_vcs_backed` requires an empty `path`; plan rows always set `path`.
- **Do not change `select_canonical_plan_path()` semantics.** It has three other callers
  (`_agent_associated_plan_phase.py`, `_agent_associated_plan_summary.py`, `_loaders/_meta_enrichment_common.py`) that
  intentionally pass and receive _references_, resolving them later through the shared resolver. It must stay a pure
  chooser over reference strings.

## Approach

Restore the invariant that `ArtifactFile.path` always holds a filesystem path, by resolving logical plan references
through the existing shared, Rust-backed resolver at the single point where the plan row is synthesized.

Resolution belongs in `sase.core.artifact_file_helpers.selected_plan_path()`. That function has exactly one caller
(`synthesize_default_artifact_files()`), so the change is fully contained, and it already reads the `done` and
`agent_meta` dicts that carry `workspace_dir`. Resolving there — _before_ the value reaches
`select_canonical_plan_path()` — also repairs a latent secondary bug: that function's ambiguous branch calls
`_path_exists(sdd_plan_path)`, which is unconditionally `False` for a logical reference, so when `plan_committed` is
absent it wrongly prefers a stale archived path over a live SDD plan.

Do **not** reimplement reference parsing or root search. Call
`sase.sdd.plan_refs.resolve_plan_reference(value, workspace_dir=..., workspace_num=...)` and use the returned
`PlanReferenceResolution`. This keeps the Rust core boundary intact — the grammar and root search already live in
`sase_core`, exposed via the `plan_reference_resolve` binding.

## Implementation

### 1. Resolve the reference in `selected_plan_path()`

File: `src/sase/core/artifact_file_helpers.py`

- Add a module-private helper that turns a possibly-logical plan reference into a filesystem path:
  - Return the value unchanged when it is already an absolute path (fast path — this is the overwhelmingly common case
    and must cost nothing).
  - Otherwise resolve via `sase.sdd.plan_refs.resolve_plan_reference()`, using the `workspace_dir` recovered from
    `done`/`agent_meta` and the workspace number from
    `sase.sdd.plan_refs.workspace_context_for_plan_resolution(workspace_dir)`.
  - Use `PlanReferenceResolution.best_path` rather than `.resolved_path`. `best_path` falls back to the first ordered
    candidate on a miss, so a genuinely deleted plan still yields a plausible real location and an **honest**
    `(missing)` marker instead of a fabricated workspace-joined path.
  - Return the original value unchanged when there is no workspace context, when resolution yields nothing, or when the
    resolver raises. Never let this degrade an existing working case.
- **Import `sase.sdd.plan_refs` lazily inside the function.** `artifact_file_helpers` is imported broadly across
  `sase.core`; a module-level import would add `sase.sdd.store` + Rust binding load to that import graph. (Checked: no
  actual import cycle exists today — `sase.sdd` does not import any `artifact_file` module — but keep the import lazy
  anyway, matching the existing lazy-import style in `_capture_decisions()`.)
- Apply the helper to `sdd_plan_path` before passing it to `select_canonical_plan_path()`. Leave `archived_plan_path`
  alone; those are already absolute.

### 2. Confirm no other synthesis path needs it

File: `src/sase/core/artifact_file_defaults.py`

`synthesize_default_artifact_files()` should need no edit once `selected_plan_path()` returns a real path. Verify the
plan row's `created_at` becomes populated (`file_created_at()` now stats a real file), which is the cheapest signal that
the fix took effect.

### 3. Performance guard

`selected_plan_path()` is reached from `artifact_file_paths()` during detail-header summary construction. Per the TUI
performance rules, render paths must not stat or glob.

- The absolute-path fast path means agents without a logical reference pay **zero** additional cost.
- For agents with a logical reference the added work is one bounded `resolve_plan_reference()` call. This path already
  reads three JSON files per call (`done.json`, `agent_meta.json`, `plan_path.json`), and its results are cached by
  `_artifact_file_page_cache` in `src/sase/ace/tui/actions/agents/_panel_artifact_files.py`, so one additional bounded
  resolution is consistent with the existing cost profile.
- Do not add a new cache in `sase.core` unless measurement shows a regression. If one is needed, key it on
  `(reference, workspace_dir, workspace_num)` and bound its size.
- Verify no regression with `SASE_TUI_PERF=1` and a j/k sweep over the Agents tab with a phase agent selected; p95
  key-to-paint must stay under 16 ms.

## Tests

### New: synthesis resolves logical plan references

File: `tests/artifact_file_facade/test_synthesis.py`

Follow the existing fixture style (`agent_dir`, `write_json` from `.helpers`) and monkeypatch plan roots the same way
`tests/test_plan_reference_resolver_integration.py` does:

```python
monkeypatch.setattr("sase.sdd.plan_refs.resolve_plan_roots", lambda *_args: (store_root, local_root))
```

Cover:

1. `agent_meta.json` with `sdd_plan_path: "plans:202607/example.md"` and `plan_committed: true`, with the plan present
   under the local plans root → the synthesized `kind="plan"` row has `path` equal to the resolved absolute path, and
   `created_at` is populated.
2. An **absolute** `sdd_plan_path` is passed through byte-for-byte unchanged (regression guard for the fast path).
3. Resolution miss (reference names a plan that does not exist under any root) → synthesis still produces a row and does
   not raise; the row's `path` is a real candidate location or the original reference, never a workspace-joined
   pseudo-path.
4. No `workspace_dir` in `done.json`/`agent_meta.json` → no raise, value passed through.

### New: the plan row is suppressed in the metadata panel

File: `tests/ace/tui/widgets/test_agent_display_artifact_file_metadata.py` (or a sibling using the existing
`_agent_display_helpers.make_agent` fixture)

Assert the end-user-visible behavior from the bug report: for a phase agent whose epic plan is a `plans:` reference, the
rendered ARTIFACTS section contains **no** `(missing)` suffix for the plan and **no** duplicate plan row, because
`_agent_display_header_summary`'s `resolved_plan_paths` filter now matches. This is the test that would have caught the
reported symptom.

### Extend: cross-surface resolver agreement

File: `tests/test_plan_reference_resolver_integration.py`

This file exists specifically to assert that every surface resolves a plan reference identically, and it is parametrized
over `typed` / `absolute` / `sdd` / `sidecar` / `month-relative` reference forms. **Artifact-file synthesis is missing
from it — that omission is why this bug shipped.** Add synthesis as a covered surface so any future surface that forgets
to resolve is caught here.

## Verification

1. `just install` (workspace virtualenvs are ephemeral and may be stale).
2. `just check`.
3. Targeted runs:
   ```bash
   .venv/bin/python -m pytest tests/artifact_file_facade/test_synthesis.py \
     tests/test_plan_reference_resolver_integration.py \
     tests/ace/tui/widgets/test_agent_display_artifact_file_metadata.py -q
   ```
4. End-to-end confirmation against real data — synthesize the affected agent's artifact files directly and confirm the
   plan row now carries an existing absolute path:
   ```bash
   .venv/bin/python -c "
   from sase.core.artifact_file_facade import list_artifact_files
   import os
   d='$HOME/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202608/05/20260805154735'
   for a in list_artifact_files(d):
       if a.kind=='plan':
           print(a.path, os.path.exists(a.path), a.created_at)
   "
   ```
   Expected: `~/.sase/plans/202608/bead_close_publication_loss.md True <timestamp>`.
5. Open `sase ace`, select a phase agent whose epic plan is a `plans:` reference, and confirm the ARTIFACTS section
   shows the plan neither as `(missing)` nor as a duplicate of the `Epic Plan:` line.

## Success criteria

- `ArtifactFile.path` holds a filesystem path for every synthesized plan row; logical `plans:` references never leak
  into it.
- The false `(missing)` marker is gone, and the duplicate plan row is suppressed by the pre-existing dedup filter.
- Plan-row `created_at` is populated; clipboard copy yields a real path with an accurate `missing` flag.
- A genuinely absent plan still renders `(missing)` — honestly.
- Absolute `sdd_plan_path` values and all non-plan artifact rows are byte-for-byte unchanged.
- No new plan-reference parsing or root-search logic is added outside the shared Rust-backed resolver.
- `just check` passes; no TUI key-to-paint p95 regression.
