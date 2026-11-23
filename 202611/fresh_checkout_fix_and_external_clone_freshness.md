---
tier: tale
title:
  Make just fix self-bootstrapping and start external project clones at origin's tip
goal:
  A host before-commit hook running `just fix` succeeds in a fresh sase checkout, and
  `sase repo open <project>` creates external clones at the remote tip, not the
  primary's lagging HEAD.
size: small
decisions:
  docs_bootstrap:
    ask:
      Should sase-backed fix/check recipes run the full `_setup` only when the venv
      lacks sase?
    choices:
      lazy:
        Probe first; a provisioned `just fix` stays offline and never rebuilds
        sase_core_rs
      setup:
        Depend on `_setup`; every fix and commit hook fetches sase-core and may rebuild
    default: lazy
    why:
      The host runs `just fix` on every sase commit; only fresh checkouts should pay
      setup
    answer: lazy
proposed_by: bbugyi200.athena.1fh
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-23 08:00:31
status: wip
---

# Make `just fix` self-bootstrapping and start external project clones at origin's tip

## Why

A tailnet epic phase agent (`tailnet-1e.7.1.4.1.7.1.7.1.3`) spent about an hour working
in an external `sase` checkout opened with `sase repo open sase` from another project.
Its sase-core commit landed, but host completion failed. `sase stitch create` ran the
configured before-commit hook (`sase_git_fix`, which execs `just fix`) in the external
sase checkout and failed:

```
.venv/bin/python tools/render_docs --sase-only
[render_docs] keymap-macro-reference: No module named 'yaml'
[render_docs] cli-registry: No module named 'sase'
error: recipe `docs-gen` failed ...
```

The stitch layer correctly refused to retry because its inputs had not changed. The
agent's Python diff only survived because a relaunch-time eviction rescued it. Two
independent defects caused the failure. Fix both.

### Defect 1: `just fix` cannot bootstrap the application venv

`fix` runs
`fmt-py fmt-docs docs-gen fmt-md fix-keep-sorted fix-docs-budgets sync-footprint-catalog`.
The formatter recipes bootstrap `.venv-format` themselves through `_setup-format-tools`.
`docs-gen` and `sync-footprint-catalog`, and their CI twins `docs-gen-check` and
`sync-footprint-catalog-check`, depend only on `_venv`. That recipe runs a bare
`uv venv .venv` and installs nothing, yet these four recipes run tools that import the
`sase` package. So `just fix` fails deterministically in any checkout where
`just install-venv` / `just check` never populated `.venv`. Every fresh external or
linked clone is in that state. A sibling phase in the same epic succeeded only because
its agent happened to run `just install-venv` by hand in that checkout first.

Nearby recipes that import sase (`sync-feature-flags-schema`,
`sync-macro-input-schemas`, `sync-completion-spec`) already depend on `_setup`. The
default decision does not copy that, because `_setup` is heavy as a `fix` dependency:

- It runs `_refresh-sase-core-checkout`, a network fetch of sase-core.
- It runs `validate_test_environment --check-core`, which flags a Rust rebuild whenever
  the linked sase-core source changed since the last build.
- It runs `_setup-required-plugins` every time.

As a dependency of `fix`, all of that would land on every host commit hook.

### Defect 2: external project clones start at the primary checkout's lagging HEAD

`_clone_external_project` in `src/sase/main/repo_open_external.py` calls
`ensure_git_clone_at(<project primary checkout>, 2, <staging dir>)`. That function
(`src/sase/workspace_provider/_utils_checkout_clone.py`) `git clone`s the _local primary
checkout_, rewrites `origin` to the real remote, and runs `git fetch`. It never advances
the local default branch. The primary checkout only fast-forwards periodically; in the
incident it sat 13 commits behind for about ten hours. So the external clone started 13
commits behind `origin/master` and was missing the two prior phase commits the phase
depended on. The agent never noticed.

Linked repos fixed the identical problem in commit 41e5176e8f ("fix(linked-repos):
fast-forward clean stale sase-core checkouts"). There,
`materialize_linked_repo_workspace` calls `refresh_clean_linked_checkout`
(`src/sase/_linked_repo_workspaces.py`) after `ensure_git_clone_at`. The
external-project path never got the equivalent.

## Changes

### 1. Justfile: lazy app-venv bootstrap for the sase-backed fix/check recipes

- Add a private recipe (suggested name `_setup-app-import`) that depends on `_venv`,
  with a comment explaining why it exists. Its body:
  - If the app venv already has an installed `sase` distribution plus `yaml`, do
    nothing.
  - Otherwise print one `[setup] ...` notice line and delegate to the full bootstrap
    with
    `just --set venv_dir "{{ venv_dir }}" --set sase_core_dir "{{ sase_core_dir }}" _setup`.
    This mirrors how other recipes re-invoke `just` with these overrides.
- **The probe must not run from an un-isolated interpreter.** The repo root contains a
  top-level `sase/` config directory with no `__init__.py`. So
  `.venv/bin/python -c "import sase"` run from the Justfile directory _succeeds_ as a
  namespace package even when the venv is empty; this was verified. Use `-I` and check
  distribution metadata, for example
  `{{ venv_bin }}/python -I -c "import importlib.metadata as m; m.distribution('sase'); import yaml" >/dev/null 2>&1`.
- Switch `docs-gen`, `docs-gen-check`, `sync-footprint-catalog`, and
  `sync-footprint-catalog-check` from `_venv` to the new recipe.
- Do not touch `_setup`, `fix`'s formatter recipes, or recipes that do not import the
  sase package: `ratchet-core-*`, `docs-check`, `docs-pdf-check`, `docs-social-card`.

> [!decision] docs_bootstrap = setup Skip the new recipe. Change the four recipes'
> dependency from `_venv` straight to `_setup`, and have the tests assert that `_setup`
> is in their dependency chain instead of asserting the probe.

Tests: add them to `tests/test_justfile_lint_setup.py` or a focused sibling, marked
`pytest.mark.contract` and using `tests/_justfile_lint_helpers` (`dry_run`,
`copy_justfile`):

- The dry run of each of the four recipes includes the probe-then-`_setup` delegation.
- The probe invocation uses `-I`. This guards against the namespace-package false
  positive.
- One behavioral test with a copied Justfile in a tmp dir and a spy `.venv/bin/python`,
  following the existing `_install_spy_python` pattern:
  - When the probe fails, the recipe reaches the `_setup` delegation.
  - When the probe succeeds, it does not.

### 2. External project clones fast-forward to origin when first created

- In `_clone_external_project`, after `ensure_git_clone_at(...)` succeeds on the staging
  directory, call `refresh_clean_linked_checkout(str(temp_dir))` from
  `sase._linked_repo_workspaces`.
  - The staging clone is clean, and its default branch tracks `origin/<branch>`. The
    helper fast-forwards it when it is strictly behind, is a no-op otherwise, and
    swallows fetch/merge failures.
  - This runs inside `_atomic_external_clone`'s staging directory, before the rename
    into place, so the published checkout is already current.
  - Update the helper's docstring to say external project clones use it too. Keep its
    name.
- **Do not refresh a valid reopened external checkout.** Both `sase repo open`'s help
  (rendered in `docs/reference/cli/repo.md`: "without cleaning, resetting, or
  synchronizing an existing checkout") and `_materialize_external_repo`'s docstring
  promise that. Only newly created clones change, so no documentation needs editing.
- Provider-ref clones (`gh:owner/repo`) clone directly from the remote and need no
  change.

Tests: add them to `tests/main/test_repo_handler_open_external.py`, reusing the fixtures
of `test_repo_open_registered_project_clones_locally_and_reopens_without_cleaning`.
`tests/test_linked_repo_workspaces.py::test_materialize_fast_forwards_clean_clone_behind_origin`
is a model for building a primary that is behind its origin.

- New-clone freshness: set up a bare origin, a registered project primary cloned from
  it, and a new commit pushed to the bare origin from another clone, so the primary is
  strictly behind. `sase repo open` (or `open_external_repo`) for that project must then
  produce an external clone whose HEAD is origin's new tip, while the primary's HEAD
  stays unchanged.
- Reopen contract: an existing valid external clone that is behind origin keeps its HEAD
  on reopen. Extend the existing reopen test if it does not already assert this.

## Verification

- `just fix`, then `just check` through `sase tool run check`.
- Optional manual check of change 1:
  - In a throwaway clone with no `.venv`, `just docs-gen` should install the venv and
    then render.
  - A second run should print no `[setup]` notice.

## Out of scope

- `sase tool run check` validation failures inside external checkouts. A phase agent
  reported a missing `project_local_commands` binding and the attachments sidecar init.
  That is a separate issue.
- No memory notes change. No existing note describes external clone freshness or
  `docs-gen`'s venv requirements.
