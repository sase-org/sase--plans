---
tier: tale
title: Fix New Project wizard never offering GitHub Create
goal: The New Project Source stage pins the Create row for GitHub on open, so `n`
  starts repository creation.
size: small
decisions:
  explain_n:
    ask: When `n` finds no pinned Create row, should it show a toast explaining why
      instead of doing nothing?
    choices:
      notify: Toast why `n` can't create yet (checking, or unsupported, press R);
        adds one docs sentence
      silent: Keep `n` a silent no-op without a Create row; only the race fix and
        its tests land
    default: notify
    why: the silent no-op is why this race was hard to diagnose
    answer: notify
proposed_by: bbugyi200.athena.10y
proposed_with: claude/opus
decided_by: auto
status: done
---

# Fix: New Project wizard never offers GitHub "Create", so `n` does nothing

## Diagnosis

**User's suspicion, confirmed in part:** `n` (new) in the New Project wizard's Source
stage really does nothing for the GitHub provider. The `n` binding itself works. It
selects the pinned **CREATE** row, and that row never appears. In the screenshot the
list starts at `ON THIS MACHINE` with no `CREATE` group above it.

**Root cause:** create-support discovery runs before the provider's inventory loads,
then caches the wrong answer.

1. `src/sase/ace/tui/modals/projects_pane_new.py` opens `NewProjectModal` without
   `candidate_inputs`, so the modal bootstraps them asynchronously.
2. `NewProjectModalProvidersMixin._configure_providers`
   (`new_project_modal_providers.py`) calls `_maybe_discover_create_support(selected)`
   before `_start_provider_load(selected)`. `_observe_create_support` reads
   `self._inputs.get(workflow_type)`, which is still `None`, so it computes
   `namespace = ""`.
3. sase-github's `describe_gh_repo_creation("")` returns `None` by contract ("Returns
   None for empty namespaces"). The modal caches
   `_create_support["gh"] = "unsupported"`.
4. After the inventory loads, `_refresh_owner_set` calls
   `_maybe_discover_create_support(workflow_type)` without `force`. That call hits the
   cached `"unsupported"` result, so it never asks again with the real owner.
5. `NewProjectSource.action_create_new` returns silently when `_create_row is None`.

**Evidence.** A headless Textual pilot reproduced this against the user's installed
build: sase `c68a30365` (the same build as the TUI in the screenshot) and sase-github
0.2.24, which implements `ws_describe_repo_creation`. Apollo, the screenshot host, has
the same versions.

- On open, `describe_repo_creation` was called once, with `[("gh", "")]`. The result was
  `_create_support == {"gh": "unsupported"}` and `create_row is None`. `n` from the list
  left the stage at `source`.
- After `R`, which forces rediscovery, the hook was called again with
  `("gh", "bbugyi200")`. It returned a form, the Create row was pinned, and `n` entered
  `create-configure`.
- Workaround until the fix ships: in the Source stage press `R`, then `n` from the list.

**Why tests missed it:** every wizard test passes `candidate_inputs=` up front, so the
namespace is already known when discovery runs. They also stub `describe_repo_creation`
as `lambda _wt, _ns: object()`, which ignores the namespace. No test covers the real
bootstrap path together with GitHub's rule that an empty namespace returns `None`.

Note: a workspace `.venv` may pin an older sase-github (0.2.21) that lacks
`ws_describe_repo_creation`. That only matters for live repros; the tests stub the hook.

## Changes

### 1. Key create-support discovery on the namespace (`new_project_modal_providers.py`)

- Add a helper `_create_namespace(workflow_type) -> str | None`:
  - Return `None` when `self._inputs.get(workflow_type)` is `None`, meaning the
    inventory has not loaded yet.
  - Otherwise return `inputs.account`, or else the first named owner whose
    `status != "error"`, or else `""`.
  - Skipping error owners keeps the synthetic
    `OwnerResult(name="provider", status="error")` that `_load_provider` builds on a
    load failure from becoming a bogus `"provider"` org namespace. Until now the
    empty-namespace race hid that bug.
- Change `_create_support` from `dict[str, str]` to `dict[str, tuple[str, str]]`
  (`workflow_type -> (namespace, status)`). Update both the `TYPE_CHECKING` annotation
  in the mixin and the initializer in `NewProjectModal.__init__`
  (`new_project_modal.py`).
- `_maybe_discover_create_support(workflow_type, *, force=False)`:
  - Compute `namespace = self._create_namespace(workflow_type)`. If it is `None`, return
    without caching anything; the load path triggers discovery once the inventory lands.
  - Keep the in-flight guard. The observer re-checks the namespace when it completes.
  - Reuse the cache only when `not force` and the cached namespace equals the current
    one. Otherwise, after dropping the cache entry on `force`, spawn
    `_observe_create_support(workflow_type, namespace)`.
- `_observe_create_support(workflow_type, namespace)`:
  - Describe the namespace it was given; do not re-read the inputs.
  - Discard the loading flag and keep the existing teardown and mount guards.
  - Store `(namespace, "supported" | "unsupported")`.
  - If `self._create_namespace(workflow_type)` changed while the call was in flight,
    call `_maybe_discover_create_support(workflow_type)` again instead of applying the
    result. Otherwise call `_apply_create_support`.
- `_apply_create_support`: read the status from the tuple. Keep the rest unchanged.
- `_load_provider`: right after the cache-only inputs are installed
  (`self._show_inputs_if_selected(workflow_type)`), call
  `self._maybe_discover_create_support(workflow_type)`. The Create row then appears as
  soon as cached inventory arrives, not after every owner revalidates live.
- Keep the existing calls in `_configure_providers`, `select_provider`,
  `action_refresh_provider`, `_finish_provider_load`, and the end of
  `_refresh_owner_set`. With the namespace-aware cache they are correct:
  - When inputs are not loaded yet, they do nothing.
  - When the inputs were provided (as in the tests), discovery still runs immediately.
  - After a refresh, they re-discover only if the namespace changed.

This stays presentation and TUI glue in Python. It changes no Rust/core boundary.

### 2. Explain an unavailable `n` (only if `explain_n = notify`)

> [!decision] explain_n = notify
>
> - `NewProjectSource`: add a `CreateUnavailable(Message)`. In `action_create_new`, when
>   the list is focused and `_create_row is None`, post it instead of returning
>   silently. Keep the existing early return for non-list focus.
> - `NewProjectModal` (`new_project_modal.py`): add
>   `@on(NewProjectSource.CreateUnavailable)`. `@on` handlers must stay on the concrete
>   modal; see its module docstring. The handler delegates to a new providers-mixin
>   helper `_explain_create_unavailable()`, which calls `self.notify(...)` with the
>   provider display name:
>   - If the inventory is not loaded, or `workflow_type in _create_support_loading`:
>     "Checking whether {Provider} can create repositories…" (information severity).
>   - If the status is `"unsupported"`: "{Provider} can't create repositories for
>     {namespace or 'this account'} — press R to retry" (warning severity).
> - Docs: in `docs/reference/tui/projects-tab.md`, Source step (item 1), add one short
>   sentence saying `n` explains why when no Create row is available. Before editing,
>   read the `docs_writing.md` reference memory with `/sase_memory_read`.
>
> With `explain_n = silent`, skip this section entirely.

## Tests

Add `tests/ace/tui/modals/test_new_project_create_support.py`. Reuse `ModalApp`,
`inputs`, and `provider` from `tests/ace/tui/modals/_new_project_source_helpers.py`,
`repo_form` from `_new_project_create_helpers.py`, and `wait_for` from
`sase.ace.testing.wait`. Use a describe stub that mirrors GitHub's contract: `None` for
`""`, and `repo_form()` with the namespace replaced for any other namespace. Record
every call.

1. **Bootstrap path pins the Create row (the regression).**
   - Construct `NewProjectModal()` with no `provider_metadata` or `candidate_inputs`.
   - Monkeypatch `workspace_provider.get_all_workflow_metadata` to return
     `(provider("gh", "GitHub"),)`.
   - In `sase.ace.tui.modals.new_project_modal_providers`, patch `load_preferences` to
     return `Preferences(last_provider="gh")` and `load_candidate_inputs` to return
     `inputs()` for both cache-only and live calls.
   - Stub `parse_repo_ref` and `probe_repo` as the Source tests do. Set `SASE_HOME` to
     `tmp_path`.
   - Assert, using `wait_for`, that `source.create_row is not None`.
   - Assert `""` was never passed to describe, and that `"bbugyi200"` was.
   - Focus `#new-project-source-list`, press `n`, and wait for
     `modal._state.stage == "create-configure"`.
2. **A cached result for another namespace is not reused.**
   - Provide `candidate_inputs={"gh": inputs(account="", owners=[])}`. Discovery
     describes `""` and the row stays unpinned.
   - Set `modal._inputs["gh"] = inputs()` and call
     `modal._maybe_discover_create_support("gh")`, mirroring the end-of-refresh call.
   - Assert `"bbugyi200"` gets described and the Create row is pinned.
3. **Namespace changes while discovery is in flight.**
   - Use a gated describe (see `_gated_describe` in `test_new_project_create_races.py`).
   - Start discovery with `account=""`, then swap in `inputs()` before releasing the
     gate.
   - Assert a follow-up describe for `"bbugyi200"` happens and the row ends up pinned.
4. **A failed provider load never describes the synthetic error owner.**
   - Make `load_candidate_inputs` raise on the bootstrap path.
   - Assert describe is never called with `"provider"`.
5. **(explain_n = notify)** With describe always `None`, focus the list and press `n`.
   - Assert the stage stays `source` and a notification mentions `R` (patch
     `modal.notify` or inspect the app's notifications).
   - Assert pressing `n` while the filter has focus still types into the filter and
     posts no toast.

Existing wizard tests pass `candidate_inputs` up front and must keep passing unchanged,
including the PNG snapshots in
`tests/ace/tui/visual/test_ace_png_snapshots_new_project_source.py`. Inputs are already
present there, so discovery still runs right away with the same namespace, `""` for
`_empty_inputs()`.

## Verification

- Run the new test file plus `tests/ace/tui/modals/test_new_project_source.py`,
  `test_new_project_create*.py`, and the New Project PNG snapshot tests.
- Then follow the `lint_and_test.md` reference memory, which means `just check` through
  `sase tool run`.
- Optional live check with a gh-authenticated sase-github of at least 0.2.23: drive
  `NewProjectModal()` through the bootstrap path in a headless pilot. Expect describe
  calls of `[("gh", "bbugyi200")]` with no `""` call, a pinned Create row without
  pressing `R`, and `n` from the list entering `create-configure`.
