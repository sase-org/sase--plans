---
tier: tale
title: Fix sase-telegram CI stale project-display test seam
goal:
  sase-telegram CI is green again on 3.12 and 3.13 against current sase master, with the
  display-map test seam in one place.
size: small
decisions:
  drift_watch:
    ask:
      Run sase-telegram CI daily (plus manual dispatch) so sase-master drift surfaces
      within a day?
    choices:
      daily:
        Add a daily cron and workflow_dispatch to ci.yml; free on a public repo, red
        runs send email
      skip: Leave ci.yml untouched; drift stays hidden until the next sase-telegram push
    default: daily
    why:
      this breakage sat unseen for 14 days and scheduled minutes are free on a public
      repo
    answer: daily
proposed_by: bbugyi200.athena.1g8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 16:40:40
status: wip
---

# Plan: Fix sase-telegram CI by retargeting the stale project-display test seam

## Problem

Every `CI` run on `sase-org/sase-telegram` master since 2026-11-20 (commits `60a94ab`,
`ceffb63`, `bffd4f6`, `4f2939f`, plus release-please PR #41 for 0.4.30) fails in the
`Run tests` step. Lint passes. Each run shows the same 4 setup errors and no assertion
failures (e.g. `881 passed, 4 errors`):

```
ERROR tests/test_formatting.py::TestBuildForkCopyText::test_tag_prompt_renders_tag_form
ERROR tests/test_inbound.py::TestBeadProjectContext::test_extract_project_from_tag_prompt
ERROR tests/test_inbound.py::TestBeadProjectContext::test_records_project_context_for_tag_prompt
ERROR tests/test_inbound.py::TestBeadProjectContext::test_fork_wait_copy_text_uses_tag_form
AttributeError: <module 'sase.project_display_names' ...> does not have the attribute
'_project_display_name_map_cached'
```

The other matrix leg (3.12 or 3.13) is just fail-fast cancellation of the same failure.

## Root cause

- CI checks out **sase master** and **sase-core master** unpinned (`.sase-deps/`), so
  sase-telegram tests run against whatever sase looks like at run time.
- sase commit `19f497f82d` (2026-11-06, "refactor(project-display): split
  project_display_names into private modules") turned `sase.project_display_names` into
  a thin public facade over `sase._project_display_cache`,
  `sase._project_display_labels`, and `sase._project_display_snapshot`. It removed the
  private `_project_display_name_map_cached` function. The facade docstring says private
  names now live in, and must be imported from, the split modules. That commit
  retargeted sase's own monkeypatches to
  `sase._project_display_labels.cached_project_display_snapshot`, but sase-telegram has
  two copies of a `_fake_tag_catalog` fixture that still
  `patch.object(sase.project_display_names, "_project_display_name_map_cached", ...)`:
  - `tests/test_formatting.py`, a class-level fixture on `TestBuildForkCopyText` (around
    line 130).
  - `tests/test_inbound.py`, a module-level fixture just above
    `class TestBeadProjectContext` (around line 2843).
- sase-telegram CI did not run between 2026-10-19 (green) and 2026-11-20, so the first
  sase-telegram push after the sase split was the first red run.

Product code (`src/sase_telegram/formatting.py`) only imports public facade names
(`project_display_name_for`, `humanize_*`), so it is unaffected. This is purely
test-seam drift.

### Correct replacement seam

In `sase/_project_display_labels.py`, every humanizer resolves its map through
`_display_names()`. That function calls `cached_project_display_snapshot`, a name bound
in the `sase._project_display_labels` module namespace (imported from
`sase._project_display_cache`). `_display_names()` is typed `Mapping[str, str]`, so a
plain `{"sase": "sase"}` dict still satisfies every consumer. The correct target is
therefore:

```python
import sase._project_display_labels as display_names_module
patch.object(display_names_module, "cached_project_display_snapshot",
             return_value={"sase": "sase"})
```

Patching the facade (`sase.project_display_names.cached_project_display_snapshot`) would
NOT work: the labels module holds its own reference.

This was verified on a scratch copy of sase-telegram `4f2939f` against current sase
master. Before the change, the 4 errors reproduce exactly. After retargeting both
fixtures as above, those tests pass (19/19 in the two classes) and the full suite
reports `885 passed`. `ruff check` and `ruff format --check` stay clean on the edited
files. (mypy only covers `src/`.)

## Implementation

All edits are in the **sase-telegram** linked repo. Open it first with
`sase repo open sase-telegram -r "<reason>"`, work only in the printed path, and read
its `AGENTS.md`.

### 1. Consolidate the fixture into `tests/conftest.py` with the new seam

Two copy-pasted fixtures meant one sase refactor broke two places. Replace them with one
shared, non-autouse fixture in `tests/conftest.py` that keeps the exact name
`_fake_tag_catalog`. All four consumers request it by that name (one in
`test_formatting.py`, three in `test_inbound.py`), so no test signatures change.

- Body: identical to the current fixtures (a single `ProjectTagCatalog` with one
  `ProjectTagTarget(key="sase", name="sase", tag="+sase", workflow_type="git", vcs_ref="#git:sase", provider_display="Git (bare)", state="enabled")`,
  `accent_palette=()`; patch both `load_project_tag_catalog` and
  `peek_project_tag_catalog` on `sase.project_tags.catalog`; `yield fake`). Change only
  the third patch, which now targets
  `sase._project_display_labels.cached_project_display_snapshot` with
  `return_value={"sase": "sase"}`.
- Keep the sase imports inside the fixture body, as the current copies do, so importing
  conftest stays cheap. Annotate the return as `Iterator[ProjectTagCatalog]`; `Iterator`
  is already imported in conftest. Use a string or `TYPE_CHECKING` import for the
  annotation if needed, to keep the sase import lazy.
- Keep the existing docstring and add one sentence: the display-map seam lives in
  `sase._project_display_labels` (not the public `sase.project_display_names` facade)
  because the humanizers resolve `cached_project_display_snapshot` from that module's
  namespace. If sase moves it again, `patch.object` fails loudly at setup.

### 2. Delete the two local fixture copies

- `tests/test_formatting.py`: remove the `@pytest.fixture() def _fake_tag_catalog(self)`
  method from `TestBuildForkCopyText`. Then drop any imports that become unused (e.g.
  `patch`), but only if nothing else in the file uses them. Check with `ruff`; note
  `F401` is ignored in this repo's ruff config, so check manually.
- `tests/test_inbound.py`: remove the module-level
  `@pytest.fixture() def _fake_tag_catalog()` above `class TestBeadProjectContext`.
  `patch` is used widely in that file, so keep it.

A test-class method fixture would shadow conftest. Removing the class copy is what lets
the conftest fixture take effect for `test_formatting.py`, so do not leave it behind.

### 3. Decision `drift_watch`

> [!decision] drift_watch = daily In `.github/workflows/ci.yml`, extend `on:` with a
> daily `schedule` (e.g. `- cron: "17 9 * * *"`, off the top of the hour) and a bare
> `workflow_dispatch:`. Leave `push`/`pull_request` and the job body unchanged. The
> scheduled run exercises sase-telegram master against current sase/sase-core master, so
> upstream refactors like `19f497f82d` show up within a day. `workflow_dispatch` lets a
> human re-run master CI without a push.

> [!decision] drift_watch = skip Do not touch `.github/workflows/ci.yml`.

## Verification

Run from the sase-telegram checkout root:

1. `just install-venv`. In a linked workspace, the Justfile auto-resolves the enclosing
   sase checkout as the `sase` source and the sibling linked `sase-core` for the maturin
   build, matching what CI tests against. If `sase-core` is not linked in the workspace,
   open it with `sase repo open sase-core` first.
2. Targeted:
   `.venv/bin/pytest tests/test_formatting.py::TestBuildForkCopyText tests/test_inbound.py::TestBeadProjectContext`.
   All must pass (19 tests), with no `AttributeError` at setup.
3. Full gate: `sase tool run check` (guarded recipe; do not run bare `just check`).
   Expect ruff + mypy clean and the full suite green (~885 passed).
4. `rg "_project_display_name_map_cached" tests src` returns nothing, and
   `rg "def _fake_tag_catalog" tests` matches only `tests/conftest.py`.
5. If `drift_watch = daily`, sanity-check the workflow YAML, e.g. with
   `python -c "import yaml,sys; yaml.safe_load(open('.github/workflows/ci.yml'))"`. Note
   that PyYAML parses the `on:` key as boolean `True`; that is expected.

After the host lands the change on sase-telegram master, the push-triggered `CI` run
should go green on both 3.12 and 3.13 (check with `actstat --only-failures` or
`gh run list -R sase-org/sase-telegram -w CI -L 3`). release-please refreshes PR #41
(0.4.30) on that push, which re-runs its CI. Do not block the turn waiting on CI; use
`/sase_monitor` if a wait is needed.

## Out of scope

- Pinning sase/sase-core revisions in sase-telegram CI (a cross-repo release-process
  change).
- Changing sase's facade or re-exporting private seams for downstream tests.
- The `PTBDeprecationWarning` about `retry_after` in `test_telegram_client.py` (a
  warning, not a failure).
