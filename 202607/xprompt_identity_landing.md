---
tier: epic
title: Finish canonical xprompt project identity and land sase-ac
goal: 'Every consumer of a project xprompt namespace uses the canonical user-facing
  project name, no surface advertises or resolves the ProjectSpec directory-key spelling,
  and epic sase-ac is closed with its plan file marked done.

  '
phases:
- id: browser_identity
  title: Repair project-local definition paths and browser namespaces
  depends_on: []
  size: small
  description: 'browser_identity: fix the `project_local_config:` definition-path
    regression and stop `get_all_project_local_prompts()` from emitting directory-key
    namespaces into the ACE xprompt browser and doctor.

    '
- id: prompt_bar_identity
  title: Canonicalize the prompt-bar VCS-tag workspace lookup
  depends_on: []
  size: small
  description: 'prompt_bar_identity: normalize the project the prompt-bar xprompt
    selector derives from the VCS tag before looking up its workspace, so project-local
    `sase.yml` xprompts stop being silently dropped.

    '
- id: workflow_identity
  title: Canonicalize and register-back project workflow loading
  depends_on: []
  size: medium
  description: 'workflow_identity: give `get_all_workflows()` the same canonical-identity
    and registry-backed resolution `get_all_xprompts()` received, so a project''s
    `.yml` workflow xprompts resolve by canonical name and outside the checkout.

    '
- id: identity_cache
  title: Invalidate the xprompt identity cache on project mutations
  depends_on: []
  size: small
  description: 'identity_cache: wire `project_identity`''s process-lifetime caches
    into the existing project mutation invalidation paths so a long-running `sase
    ace` does not keep resolving stale project identities.

    '
- id: land
  title: Land epic sase-ac
  depends_on:
  - browser_identity
  - prompt_bar_identity
  - workflow_identity
  - identity_cache
  size: small
  description: 'land: close bead sase-ac, run symvision and clear anything the expired
    epic whitelist reports, and mark both plan files done.'
parent_bead: sase-ac
parent: plans:202607/xprompt_project_identity.md
create_time: 2026-07-28 09:13:39
status: wip
---

# Plan: Finish canonical xprompt project identity and land sase-ac

## Context

Epic `sase-ac` (plan `plans:202607/xprompt_project_identity.md`) established the **canonical user-facing project name**
as the single xprompt namespace identity. Its five phases are closed and the headline defect is genuinely fixed —
verified from a working directory outside the checkout:

```
canonical_xprompt_project("gh_sase-org__sase")     -> "sase"
known_project_namespaces()                          -> {"sase": .../sase-org/sase, ...}
gather_structured_entries() project entries         -> sase/docs, sase/gact, sase/reads, sase/remember, sase/sync
                                                       (all tagged project="sase"; no gh_sase-org__sase/* duplicates)
build_structured_xprompts_catalog(project="sase")   -> includes them
build_structured_xprompts_catalog(project="gh_sase-org__sase") -> includes them
get_all_xprompts(project="sase")                    -> sase/reads, sase/sync, ...   (from /tmp)
process_xprompt_references("#sase/sync")            -> expands   (from /tmp)
```

The Rust core mirror (`sase-core` commit `2034123`, phase `sase-ac.5`) is consistent: `known_workspaces` is keyed by
canonical name and both the `project_local_config:` emitter and its definition-path resolver read that same map.

What remains are **consumers on the Python side that were not migrated with the catalog**. They still key project
workspaces by the ProjectSpec directory key while the strings now flowing through the system are canonical names. One of
them is a regression the epic introduced; the rest are surfaces where the epic's stated goal — "project-local xprompts
are namespaced with exactly one user-facing name **everywhere**, and those references resolve regardless of the process
working directory" — is not yet met.

### Reference: the two spellings

For the `sase` project: directory key `gh_sase-org__sase`, canonical user-facing name `sase`. Any project registered
with `#gh:<org>/<repo>` has this split; `#git:<name>` projects do not, which is why these defects are invisible in most
fixtures.

### Reference: the helpers to use

Both live in `src/sase/xprompt/project_identity.py` and were added by `sase-ac.1`:

- `canonical_xprompt_project(ref: str | None) -> str | None` — directory key, `PROJECT_NAME`, or alias -> canonical
  name. Unknown refs pass through unchanged; never raises.
- `known_project_namespaces() -> dict[str, Path]` — enabled project workspaces re-keyed by canonical name. This is the
  drop-in replacement for `get_known_project_workspaces()` at any call site that is looking up a **namespace**.

`get_known_project_workspaces()` itself stays as-is: it is the directory-key-keyed primitive and has legitimate
non-namespace callers (`sase.history.vcs_xprompt_mru`, `sase.agent.multi_prompt_vcs`, `sase.agent.launch_cwd_common`,
`sase.agent.launch_cwd_bead_work`, `sase.xprompt._parsing_vcs_refs`, `prompt_completion_root`). Do not change those —
they resolve VCS project refs and launch working directories, not xprompt namespaces, and the epic explicitly scoped the
`#gh:<org>/<repo>` -> `gh_<org>__<repo>` key scheme out.

---

## Repair project-local definition paths and browser namespaces

Two defects on the same user-facing surface (Admin Center -> XPrompts browser, plus prompt jump/preview and doctor).

### Defect 1 — `project_local_config:` definition paths no longer resolve (regression)

`load_project_local_xprompts()` (`src/sase/xprompt/loader_sources.py:521`) stamps each `sase.yml`-defined xprompt with
`source_path = f"project_local_config:{project}"`. Since `sase-ac.2`, the gatherers pass the **canonical name**, so that
string is now `project_local_config:sase`.

`resolve_source_to_file_path()` (`src/sase/ace/tui/modals/xprompt_browser_helpers.py:243-255`) still resolves it through
`get_known_project_workspaces()`, which is keyed by directory key:

```
resolve_source_to_file_path("project_local_config:sase")               -> None   # what the catalog now emits
resolve_source_to_file_path("project_local_config:gh_sase-org__sase")  -> .../sase/sase/sase.yml  # dead spelling
```

Consequence: for every project whose directory key differs from its `PROJECT_NAME`, opening or editing the definition of
a project-local `sase.yml` xprompt is broken. Affected callers: `xprompt_browser_actions.py:52,129`,
`xprompt_select_modal.py:380`, `_prompt_jump_target.py:267`, `_prompt_preview_target.py:271,289`.

**Fix.** Resolve through `known_project_namespaces()` instead. Keep the existing local import style. Accept the
directory-key spelling too if that is cheap (route `project_name` through `canonical_xprompt_project()` first, then look
it up), so any stale persisted source string still opens.

Note `classify_source()` in the same module humanizes the parsed project through `project_display_name_for(key)`. That
now receives a canonical name, which `project_display_name_for` passes through unchanged for unknown keys, so the
displayed category text stays correct. No change needed there — but do not "clean it up" into something that assumes a
directory key.

### Defect 2 — the browser still advertises the unresolvable directory-key namespace

`get_all_project_local_prompts()` (`src/sase/xprompt/loader.py:86`) is the one gatherer `sase-ac.2` did not migrate. It
still iterates `get_known_project_workspaces()`, so it namespaces with the directory key:

```
get_all_project_local_prompts() -> gh_sase-org__sase/docs, gh_sase-org__sase/gact, gh_sase-org__sase/reads,
                                   gh_sase-org__sase/remember, gh_sase-org__sase/sync
```

`load_browser_items()` (`src/sase/ace/tui/modals/xprompt_browser_catalog.py:38-41`) merges that dict **over** the
canonical entries from `get_all_prompts(project=...)`, so the ACE xprompt browser lists both spellings for the same
file, and the `gh_sase-org__sase/...` rows insert a reference that expands to nothing. This is precisely the "duplicate,
non-resolvable namespace" defect the epic's root-cause section named, still live on the browser. It also feeds
`check_config_xprompt_definitions()` (`src/sase/doctor/checks_config_xprompts.py:139`), inflating its loaded count with
phantom duplicates.

**Fix.** Iterate `known_project_namespaces()` in `get_all_project_local_prompts()`. The dict merge in
`load_browser_items()` then collapses to one row per file, so no change is needed there — confirm that with a test
rather than assuming it.

### Tests

Extend `tests/test_project_local_xprompts.py`. Note that
`TestClassifySourceProjectLocal::test_project_local_config_resolve` currently patches
`sase.xprompt.loader.get_known_project_workspaces` with `{"myproj": tmp_path}` — key equal to name, which is why the
regression slipped through. Update that patch target alongside the fix.

- A fixture project registered with directory key `gh_org__proj` and `PROJECT_NAME: proj`, holding a `sase/sase.yml`
  with an `xprompts:` entry: `resolve_source_to_file_path("project_local_config:proj")` returns that `sase.yml` path.
- The same call with `project_local_config:gh_org__proj` also resolves (if you implement the accept-both behavior) or
  returns `None` (if you do not) — assert whichever you chose, deliberately.
- `get_all_project_local_prompts()` contains `proj/thing` and does **not** contain `gh_org__proj/thing`.
- `load_browser_items()` over that fixture yields exactly one row named `proj/thing`.
- Both new assertions must fail with the source change reverted.

## Canonicalize the prompt-bar VCS-tag workspace lookup

`src/sase/ace/tui/actions/agent_workflow/_prompt_bar_requests.py:354-365` seeds the prompt-bar xprompt selector with
"extra prompts" taken from the project's `sase.yml`:

```python
vcs_project = extract_project_from_vcs_tag(vcs_tag)   # user-facing name, e.g. "sase"
if vcs_project:
    workspaces = get_known_project_workspaces()       # keyed by directory key
    ws_dir = workspaces.get(vcs_project)              # -> None for key != name
```

This is the same boundary defect `sase-ac.4` fixed in `_xprompt_arg_assist_project_from_text()`, in a sibling path the
`tui` phase did not cover. The whole block is wrapped in a bare `except Exception: pass`, so the failure is silent:
under a `#gh:sase` tag the selector simply never offers the project's `sase.yml` xprompts.

**Fix.** Route `vcs_project` through `canonical_xprompt_project()` and look it up in `known_project_namespaces()`. Keep
the local-import style and the surrounding error handling; this runs on a TUI interaction path.

### Tests

Extend the existing prompt-bar request tests (find them under `tests/ace/tui/actions/`). With a fixture project whose
directory key differs from its `PROJECT_NAME`, a prompt whose leading VCS tag names the project by its user-facing name
must put that project's `sase.yml` xprompts into the selector's extra prompts. Assert the test fails with the change
reverted.

## Canonicalize and register-back project workflow loading

`sase-ac.3` made `get_all_xprompts()` canonicalize its project and consult the registry, but its sibling
`get_all_workflows()` (`src/sase/xprompt/workflow_loader.py:589`) was left on the old contract:

- it does not canonicalize `effective_project`;
- `_load_workflows_from_project_workspace()` (line 554) looks the project up in `get_known_project_workspaces()`, keyed
  by directory key.

Every caller supplies a user-facing name — `detect_project()` returns the workspace-provider name, and the ACE catalog
path now supplies a canonical name — so the lookup never hits for a key != name project and the function silently
returns `{}`:

```
get_all_workflows(project="sase")               -> no sase/* entries
get_all_workflows(project="gh_sase-org__sase")  -> no sase/* entries
```

Consequence: a `.yml` workflow xprompt in a project's `sase/xprompts/` directory does not resolve by canonical name and
does not resolve outside the checkout, while a `.md` xprompt in the same directory now does. That split contradicts the
epic goal. It is latent rather than currently visible only because the `sase` repo's `sase/xprompts/` happens to hold
two `.md` files and no `.yml` — do not use that as a reason to skip this phase; add a `.yml` fixture and cover the
contract.

**Fix.** Mirror `sase-ac.3`:

- In `get_all_workflows()`, compute `effective_project` through `canonical_xprompt_project()`, exactly as
  `get_all_xprompts()` (`src/sase/xprompt/loader.py:156-158`) does. Keep `detect_project()` as the fallback source.
- In `_load_workflows_from_project_workspace()`, look up `known_project_namespaces()`.
- Guard the cost the same way `_load_registered_project_xprompts()` (`src/sase/xprompt/loader.py:107-127`) does: skip
  the registry read when the current checkout's detected identity already canonicalizes to the requested project, and
  read only the one requested project. `get_all_workflows()` is on hot paths.
- Preserve the documented first-wins precedence: the existing `workflows.setdefault(...)` inside
  `_load_workflows_from_project_workspace()` and the update ordering in `get_all_workflows()` must keep CWD/project
  filesystem sources winning over the registry copy. Update the `get_all_workflows()` docstring's priority paragraph to
  describe the registry source, matching what `sase-ac.3` did for `get_all_xprompts()`.

Read `sase/memory/tui_perf.md` before adding any caching here.

### Tests

Extend `tests/test_project_local_xprompts.py` (or the workflow-loader test module, whichever the existing fixtures fit
better) with a fixture project registered as directory key `gh_org__proj` / `PROJECT_NAME: proj` whose workspace holds
`sase/xprompts/flow.yml`:

- With cwd outside that workspace, `get_all_workflows(project="proj")` contains `proj/flow`.
- `get_all_workflows(project="gh_org__proj")` and any configured alias select the same entry.
- With cwd inside a second checkout of the same project defining a different `flow.yml`, the cwd copy wins.
- A disabled project's `flow.yml` stays unresolved (do not widen the `include_states=("enabled",)` default).

## Invalidate the xprompt identity cache on project mutations

`src/sase/xprompt/project_identity.py` memoizes with two process-lifetime caches that nothing ever clears:

```python
@lru_cache(maxsize=1)
def _identity_registry() -> tuple[dict[str, str], ProjectDisplaySnapshot] | None:
    return load_project_alias_map(), load_project_display_snapshot()

@lru_cache(maxsize=512)
def _canonical_xprompt_project(ref: str) -> str: ...
```

Note it calls `load_project_display_snapshot()` — the module's documented **fresh-load** entry point for off-thread
refresh — rather than the managed cached accessor, then wraps it in its own cache. So the identity projection is frozen
at first use for the life of the process, and the existing invalidation contract does not reach it:
`sase.project_aliases._set_project_name_locked()` (line 165) already calls `invalidate_project_display_snapshot()` after
a `PROJECT_NAME` change, and `set_project_aliases_locked()` mutates the alias map that the same tuple caches.

Consequence for a long-running `sase ace`: register, rename, or re-alias a project and its xprompt namespace keeps
resolving to the pre-mutation identity until restart. For a newly registered project, `known_project_namespaces()` picks
up the workspace but `label_for()` misses the new key, so it falls back to the directory key — silently reproducing the
exact bug this epic fixed, for exactly the projects most likely to be in flux. The tests already work around this by
reaching into module privates (`project_identity._identity_registry.cache_clear()` in
`tests/test_xprompt_project_identity.py`, `tests/test_xprompt_catalog_structured.py`, and
`tests/ace/tui/widgets/_completion_helpers.py`), which is the smell.

**Fix.** Add a public `invalidate_xprompt_project_identity()` to `project_identity.py` that clears both caches, and call
it from the paths that already invalidate project identity state:

- `sase.project_display_names.invalidate_project_display_snapshot()` — the natural chokepoint; `sase.project_aliases`
  already does a deferred import of `project_display_names` inside a function, so use the same deferred-import style to
  avoid an import cycle.
- `sase.project_aliases.set_project_aliases_locked()` and its siblings around lines 85-132, which change the alias map
  without touching display labels.

Check for a project create/register/enable/disable path that should invalidate too; if one exists, wire it, and if the
sweep shows the chokepoint already covers it, say so in the bead note rather than leaving it implicit.

Then replace the private `cache_clear()` reach-ins in the three test helpers listed above with the new public function.

Keep the degradation contract from `sase-ac.1` intact: the helper sits on the ACE keystroke path, must never raise, and
must fall back to the input ref on any registry read failure.

### Tests

- After `canonical_xprompt_project()` has cached a result, registering a new project and calling
  `invalidate_xprompt_project_identity()` makes the next call reflect the new registry state.
- A `PROJECT_NAME` change through `sase.project_aliases` makes the next `canonical_xprompt_project()` call return the
  new name without an explicit invalidation call from the test.
- An alias change through `set_project_aliases_locked()` makes the new alias resolve on the next call.

## Land epic sase-ac

Run this only after the four phases above are complete.

1. `sase bead close sase-ac`. If the close is rejected, the named phases were not actually complete — finish or reopen
   them rather than forcing. Force only with `--force --reason ... --resolution canceled|superseded` and only as a
   deliberate recorded outcome.
2. Run `just symvision`. The `sase-ac` epic-symbol whitelist entries expire at close, so anything they were suppressing
   surfaces now. Remove the stale entries and any unused code it reports. (As of this plan, `just symvision` is clean
   and no `sase-ac` entries remain in the `Justfile` — but re-check after the close, since expiry is what changes.)
3. Set `status: done` in the frontmatter of `plans:202607/xprompt_project_identity.md` (the epic's plan file, resolved
   under `sase/repos/plans/202607/`).
4. Set `status: done` in the frontmatter of this plan file.

## Verification

- `just install` first — workspace virtualenvs are ephemeral and this workspace may be stale.
- `just check` for each phase.
- Cross-checking probe, run from a directory outside the checkout (`cd /tmp`), which must show canonical names only:

  ```python
  from sase.xprompt.loader import get_all_project_local_prompts
  from sase.ace.tui.modals.xprompt_browser_helpers import resolve_source_to_file_path
  from sase.xprompt.workflow_loader import get_all_workflows

  sorted(get_all_project_local_prompts())            # no gh_*__* namespaces
  resolve_source_to_file_path("project_local_config:sase")   # not None
  ```

- Manual confirmation in `sase ace` with `#gh:sase` as the VCS workflow: the xprompt browser lists `sase/reads` and
  `sase/sync` exactly once each with no `gh_sase-org__sase/...` rows, and opening a project-local `sase.yml` xprompt
  definition works.

## Out of scope

- The `#gh:<org>/<repo>` -> `gh_<org>__<repo>` directory-key scheme itself, and renaming
  `ProjectRecordWire.project_name`. Both were already scoped out by the `sase-ac` plan.
- `get_known_project_workspaces()` callers that resolve VCS project refs or launch working directories rather than
  xprompt namespaces: `sase.history.vcs_xprompt_mru`, `sase.agent.multi_prompt_vcs`, `sase.agent.launch_cwd_common`,
  `sase.agent.launch_cwd_bead_work`, `sase.xprompt._parsing_vcs_refs`, and
  `sase.ace.tui.widgets.prompt_completion_root`.
- `sase.ace.tui.modals.config_pane_widget`'s workspace lookup: `ConfigCenterModal` is constructed without a `project`
  argument (`src/sase/ace/tui/actions/base.py:584`), so `self._project` is always `None` and the branch is dead in
  practice. Leave it alone.
- The Rust core (`sase-core`). Its `known_workspaces` map is already canonical-keyed end to end, including the
  `project_local_config:` emitter and its definition-path resolver, so no parity change is required for this work.
