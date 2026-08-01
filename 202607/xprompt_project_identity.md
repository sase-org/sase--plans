---
tier: epic
title: Fix project-local xprompt completion by canonicalizing project identity
goal: 'Typing `#sase/` in the ACE prompt bar under a `#gh:sase` VCS tag completes
  the project''s local xprompts (`#sase/reads`, `#sase/sync`), project-local xprompts
  are namespaced with exactly one user-facing name everywhere, and those references
  resolve regardless of the process working directory.

  '
phases:
- id: identity
  title: Canonical project-identity resolver
  depends_on: []
  size: small
  description: 'identity: add a single Python helper that normalizes any project spelling
    (directory key, PROJECT_NAME, alias) to the canonical user-facing namespace name,
    plus its unit tests.

    '
- id: catalog
  title: Namespace and filter the xprompt catalog by user-facing name
  depends_on:
  - identity
  size: medium
  description: 'catalog: make the xprompt catalog namespace, tag, and filter project-local
    xprompts by the canonical user-facing project name so requesting project `sase`
    returns `sase/reads`.

    '
- id: resolver
  title: Resolve registry-backed project xprompts independent of cwd
  depends_on:
  - identity
  size: medium
  description: 'resolver: teach the xprompt loader to read a registered project''s
    checkout xprompts through the project registry so `#sase/reads` expands even when
    the process cwd is not that checkout.

    '
- id: tui
  title: Normalize the ACE completion project boundary
  depends_on:
  - catalog
  size: small
  description: 'tui: normalize the project identity the ACE prompt bar derives from
    the VCS tag and the prompt context, and add the end-to-end regression test for
    the reported completion failure.

    '
- id: core_parity
  title: Mirror the identity fix in the Rust core catalog
  depends_on:
  - catalog
  size: medium
  description: 'core_parity: apply the same canonical-name namespacing and filtering
    to the Rust xprompt catalog in the sase-core repo so the LSP and gateway agree
    with Python.

    '
create_time: 2026-07-28 07:41:09
status: done
bead_id: sase-ac
---

- **PROMPT:** [prompts/202607/xprompt_project_identity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/xprompt_project_identity.md)
- **BEAD:** [sase-ac](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ac/README.md)

# Plan: Fix project-local xprompt completion by canonicalizing project identity

## Problem

In `sase ace`, with `#gh:sase` as the VCS xprompt workflow, typing `#sase/` in the prompt input widget shows no
completion menu. The expected candidates are `#sase/reads` and `#sase/sync`, defined in this repo's `sase/xprompts/`
directory.

## Root cause

SASE has **two different spellings of a project's identity**, and the xprompt catalog uses one while every other layer
uses the other.

A `#gh:<org>/<repo>` project has:

- a **ProjectSpec directory key** — `gh_sase-org__sase` — the `~/.sase/projects/<key>/` directory name, exposed as
  `ProjectRecordWire.project_name`;
- a **user-facing name** — `sase` — the spec's `PROJECT_NAME:`, exposed as `ProjectRecordWire.display_name`. This is
  what the user types (`#gh:sase`) and what `detect_project()` returns from inside the checkout.

The completion pipeline runs like this:

1. `PromptTextArea._xprompt_arg_assist_project_from_text()` (`src/sase/ace/tui/widgets/_xprompt_arg_hints.py:213`) reads
   the leading VCS tag `#gh:sase` and calls `extract_project_from_vcs_tag()`, which yields the **user-facing name**
   `"sase"`.
2. That string reaches `build_xprompt_assist_entries(project="sase")` (`src/sase/ace/tui/prompt_catalog.py:97`) and then
   `build_structured_xprompts_catalog(project="sase")`.
3. `gather_structured_entries()` (`src/sase/xprompt/_catalog_sources.py:82`) sweeps `get_known_project_workspaces()`,
   whose keys are **directory keys** (`src/sase/xprompt/loader_sources.py:485` returns `record.project_name`). It
   therefore namespaces the checkout's `sase/xprompts/reads.md` as `gh_sase-org__sase/reads` and tags the entry
   `project="gh_sase-org__sase"`. `classify()` (`src/sase/xprompt/_catalog_sources.py:170`) applies the same directory
   key when it maps a source path back to a known workspace.
4. `filter_structured_catalog_entries()` (`src/sase/xprompt/_catalog_structured.py:114`) drops every entry whose
   `entry.project` is neither `None` nor the requested project. `"gh_sase-org__sase"` never equals `"sase"`, so **all**
   project-local entries are discarded.
5. `build_xprompt_completion_candidates()` receives zero project entries, produces zero candidates, and
   `_try_auto_xprompt_completion()` (`src/sase/ace/tui/widgets/_file_completion_open.py:301`) returns `False` without
   opening a menu.

Reproduced directly (with `sase ace`'s real working directory, the primary checkout):

```
gather_structured_entries():
  gh_sase-org__sase/reads   project=gh_sase-org__sase   bucket=project
  gh_sase-org__sase/sync    project=gh_sase-org__sase   bucket=project
  sase/reads                project=gh_sase-org__sase   bucket=project
  sase/sync                 project=gh_sase-org__sase   bucket=project

build_structured_xprompts_catalog(project=None)                -> both spellings
build_structured_xprompts_catalog(project="sase")              -> []          <-- the bug
build_structured_xprompts_catalog(project="gh_sase-org__sase") -> both spellings
```

Two defects follow from the single mismatch:

- **Filter miss.** The requested project is a user-facing name; the tagged project is a directory key. They can never
  match, so the menu is always empty.
- **Duplicate, non-resolvable namespace.** The same file is namespaced twice — once as `sase/reads` (discovered through
  the cwd, because `get_all_xprompts()` auto-detects `detect_project()`) and once as `gh_sase-org__sase/reads`
  (discovered through the registry sweep). The dedup key in `gather_structured_entries()` is `(source_path, name)`, so
  the differing names keep both. Only the `sase/...` spelling actually resolves; `#gh_sase-org__sase/sync` expands to
  nothing. The catalog therefore advertises a name that is not a real reference.

A third, related defect is exposed by the same mismatch and matters for the fix to be useful:

- **cwd-dependent resolution.** `get_all_xprompts()` (`src/sase/xprompt/loader.py:103`) reaches project-local xprompts
  only through filesystem sources relative to the current working directory plus `~/sase/xprompts/<project>/`; it never
  consults the project registry. `load_xprompts_from_project()` (`src/sase/xprompt/loader_sources.py:395`) reads only
  the `home_project` scope. The registry-driven `load_project_file_xprompts()` that does read a project's checkout is
  called _only_ from the catalog gatherers. Consequence: run from a directory outside the checkout,
  `process_xprompt_references("#sase/sync")` returns the literal text unchanged. Since prompt expansion happens in the
  launching process (`src/sase/agent/multi_prompt_launch_plan.py:79`,
  `src/sase/ace/tui/actions/agent_workflow/_launch_body_single.py:282`), a completed `#sase/reads` silently fails to
  expand whenever `sase ace` was not started inside the checkout.

## Approach

Establish one canonical project namespace identity — the **user-facing name** (`display_name`, falling back to the
directory key when a project has no `PROJECT_NAME:`) — because that is the only spelling that resolves today and the
only one users type. Normalize at every boundary that accepts a project string, and make the registry-backed loader
reachable from the resolver.

Existing building blocks to reuse rather than reinvent:

- `sase.project_aliases.load_project_alias_map()` — `alias`/`PROJECT_NAME` -> directory key.
- `sase.project_aliases.resolve_project_alias_ref()` — one ref -> directory key.
- `sase.project_display_names.load_project_display_snapshot()` / `ProjectDisplaySnapshot.label_for()` — directory key ->
  user-facing label.
- `ProjectRecordWire.display_name`, already populated with `sase` for this project.

---

## Canonical project-identity resolver

Add one helper module (suggested `src/sase/xprompt/project_identity.py`, or a home in
`src/sase/project_display_names.py` if that reads better to the implementer) exposing:

- `canonical_xprompt_project(ref: str | None) -> str | None` — accept a directory key, a `PROJECT_NAME`, or a configured
  alias and return the canonical user-facing namespace name. Return the input unchanged when it names no known project
  (so ad-hoc namespaces such as `bd/` and `research/` keep working), and `None` for `None`/empty.
- `known_project_namespaces() -> dict[str, Path]` — the `get_known_project_workspaces()` sweep re-keyed by canonical
  name.

Requirements:

- Resolve through the alias map first (ref -> directory key), then the display snapshot (directory key -> label); a ref
  that is already canonical must round-trip to itself.
- Degrade to the input string on any registry read failure. This helper sits on the ACE keystroke path, so it must never
  raise and must not perform unbounded I/O per call — reuse the cached snapshot accessors and add memoization if
  profiling in the `tui` phase shows a cost. See `sase/memory/tui_perf.md` before adding any caching.
- The `home` project stays excluded, matching `load_project_alias_map()` today.

Unit tests: directory key -> name, `PROJECT_NAME` -> name, alias -> name, canonical name -> itself, unknown ref ->
itself, `None` -> `None`, and a project with no `PROJECT_NAME:` -> its directory key.

## Namespace and filter the xprompt catalog by user-facing name

In `src/sase/xprompt/_catalog_sources.py`:

- `gather_entries()` (line 31) and `gather_structured_entries()` (line 63): iterate `known_project_namespaces()` instead
  of `get_known_project_workspaces()`, so `load_project_local_xprompts()` / `load_project_file_xprompts()` namespace
  with `sase/` and the entries are tagged `project="sase"`.
- `classify()` (line 144): when mapping an absolute source path back to a known workspace (line 170), tag the canonical
  name rather than the directory key.
- Dedup: with both discovery routes now producing `sase/reads` from the same `source_path`, the existing
  `(source_path, name)` key collapses them to one entry. Confirm this with a test rather than assuming it — a project
  whose xprompts are reachable both ways must appear exactly once.

In `src/sase/xprompt/_catalog_structured.py`:

- `filter_structured_catalog_entries()` (line 114): normalize the requested `project` through
  `canonical_xprompt_project()` before comparing, so a caller passing a directory key, an alias, or the user-facing name
  all select the same entries. Do the same for the `project` filter in `build_xprompts_catalog` if it applies an
  equivalent comparison.

Tests (extend `tests/test_xprompt_catalog_structured.py`, `tests/test_xprompt_catalog_classification.py`,
`tests/test_project_local_xprompts.py`):

- A fixture project registered with directory key `gh_org__proj` and `PROJECT_NAME: proj`, holding
  `sase/xprompts/thing.md` in its workspace, yields exactly one catalog entry named `proj/thing` tagged
  `project="proj"`.
- `build_structured_xprompts_catalog(project="proj")` includes it; so do `project="gh_org__proj"` and any configured
  alias.
- No entry named `gh_org__proj/thing` is produced.
- An unregistered namespace (`bd/next`) is unaffected.

## Resolve registry-backed project xprompts independent of cwd

Make a canonical project namespace resolve from anywhere, so a completed `#sase/reads` always expands.

In `src/sase/xprompt/loader.py`, `get_all_xprompts()` (line 103): when `effective_project` names a registered project,
also load that project's workspace xprompts via the registry — i.e. the `load_project_local_xprompts()` +
`load_project_file_xprompts()` pair that `gather_structured_entries()` already uses — keyed by the canonical name from
the `identity` phase. Keep the documented first-wins precedence in the docstring (lines 110-114): cwd/project filesystem
sources must still win over the registry-sourced copy, so a workspace clone's edited xprompt keeps overriding the
primary checkout's version. Update that docstring to describe the new source.

Guard the cost: `get_all_xprompts()` is called on hot paths. Load the registry copy only for the one requested project,
not for every registered project, and only when the project is not already satisfied by a filesystem source.

Consider whether `load_xprompts_from_project()` (`src/sase/xprompt/loader_sources.py:395`, currently `home_project`
scope only) is the right seam for this or whether a sibling function is cleaner; either is acceptable as long as
precedence is preserved.

Tests (extend `tests/test_project_local_xprompts.py`, `tests/test_xprompt_known_project_refs.py`):

- With cwd set outside the fixture project's workspace, `get_all_prompts(project="proj")` contains `proj/thing` and
  `process_xprompt_references("#proj/thing")` expands to the body.
- With cwd inside a second checkout of the same project that defines a different `thing.md`, the cwd copy still wins.
- A `#proj/thing` reference for a _disabled_ project stays unresolved (do not widen `get_known_project_workspaces()`'s
  `include_states=("enabled",)` default).

## Normalize the ACE completion project boundary

In `src/sase/ace/tui/widgets/_xprompt_arg_hints.py`, `_xprompt_arg_assist_project_from_text()` (line 213) has two
sources that disagree:

- the VCS tag branch (line 222) yields a user-facing name (`sase`);
- the `_prompt_context` fallback (line 229) yields `PromptContext.project_name`, which is populated from the ProjectSpec
  path and is a **directory key**.

Route both through `canonical_xprompt_project()` so the app-level catalog cache in
`src/sase/ace/tui/actions/_startup_prompt_catalog.py` is keyed consistently and one warm build serves both entry points.
Check `src/sase/ace/tui/prompt_catalog.py` for the same normalization need at `_project_xprompt_dirs()` (line 212) and
`prompt_source_watch_paths()` (line 145), which build per-project watch paths from the raw project string.

Regression tests:

- Extend `tests/ace/tui/widgets/test_auto_xprompt_completion.py` with the reported case: prompt text `#gh:sase #sase/`
  with the cursor at end opens the xprompt completion menu containing `#sase/reads` and `#sase/sync`, seeded from a
  fixture project registered with a directory key that differs from its `PROJECT_NAME`.
- Add the equivalent soft-completion assertion (`tests/ace/tui/widgets/test_prompt_live_completion.py`) so `<ctrl+l>`
  and the inline suggestion agree with the menu.
- Assert the directory-key spelling is _not_ offered as a candidate.

## Mirror the identity fix in the Rust core catalog

`crates/sase_core/src/xprompt_catalog.rs` in the sibling `sase-core` repo is an independent implementation of the same
gather/classify/filter contract, used by `sase_xprompt_lsp` and the gateway. It carries the identical defect:

- `known_project_workspaces()` (around line 1679) keys by the `~/.sase/projects/<dir>` directory name — the directory
  key.
- The registry sweep (around lines 977-995) namespaces and tags project xprompts with that key.
- The project filter (around lines 300-301) drops entries whose `entry.project` differs from the requested project.

Open the repo with the `/sase_repo` skill (`sase repo open sase-core -r "<reason>"`) and use the path it prints; do not
clone or web-fetch it. Per this repo's Rust core backend boundary rule, keep the wire/API and behavior in step with the
Python fix.

Work items:

- Resolve each project directory's `PROJECT_NAME:` (and aliases) from its ProjectSpec, and use that canonical name for
  both the namespace prefix and the `project` tag.
- Normalize the requested project in the filter so key, name, and alias spellings all select the same entries.
- Add Rust unit tests mirroring the Python cases: a project whose directory key differs from its `PROJECT_NAME` yields
  `proj/thing` (never `gh_org__proj/thing`), and filtering by key, name, or alias returns it.
- Check `tools/validate_sase_core_rs` in this repo and any catalog parity fixtures for expectations that pin the
  directory-key spelling, and update them together with the Rust change.

## Verification

- `just install` first (workspace virtualenvs are ephemeral), then `just check`.
- Manual confirmation in `sase ace`: with `#gh:sase` set as the VCS workflow, type `#sase/` in the prompt input widget
  and confirm `#sase/reads` and `#sase/sync` appear in the completion menu, that accepting one inserts a reference which
  expands at launch, and that the `#gh_sase-org__sase/...` spelling no longer appears.

## Out of scope

- Changing the `#gh:<org>/<repo>` -> `gh_<org>__<repo>` directory-key scheme itself.
- Renaming `ProjectRecordWire.project_name`, despite it holding the directory key rather than the user-facing name.
- Any change to how `sase/xprompts/` files are discovered on the filesystem (the content-layout source order in
  `sase/memory/xprompts.md` stays as documented).
