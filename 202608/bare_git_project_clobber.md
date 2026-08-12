---
tier: tale
title: Stop
goal:
  A `#git:` ref aimed at a project owned by another VCS provider fails with a clear
  error instead of silently rewriting that project's ProjectSpec into a bare-git
  project.
size: medium
proposed_by: bbugyi200.athena.yh
create_time: 2026-08-12 09:48:46
status: wip
---

# Stop `#git:` refs from converting existing projects into bare-git projects

## Problem

The `sase` project (ProjectSpec key `gh_sase-org__sase`, a GitHub project) repeatedly
gets silently converted into a bare-git project: its `WORKSPACE_DIR` is rewritten from
the real GitHub checkout to a `~/projects/git/<key>/` path, and a `BARE_REPO_DIR` field
is added. The user repairs the spec by hand; a later launch converts it again.

## Verified root cause

Reproduced end to end in an isolated `HOME`/`SASE_HOME` sandbox. Given a spec file
containing only:

```text
WORKSPACE_DIR: /home/bryan/projects/github/acme/widget/
PROJECT_NAME: widget
```

calling `resolve_git_ref("gh_acme__widget")` rewrote it to:

```text
WORKSPACE_DIR: /tmp/.../projects/git/gh_acme__widget/
PROJECT_NAME: widget
BARE_REPO_DIR: /tmp/.../.sase/repos/gh_acme__widget.git
```

That is byte-for-byte the reported symptom. The causal chain:

1. **A `#git:` tag is aimed at a GitHub project.** `~/.sase/vcs_xprompt_mru.json`
   currently holds both `#gh:gh_sase-org__sase` and `#git:gh_sase-org__sase`. Cycling
   the VCS MRU (`<ctrl+p>`) to the second entry launches the GitHub project under the
   bare-git workflow. Chat history confirms many real launches used
   `#git:gh_sase-org__sase`. This also explains the intermittency: only some launches
   land on the poisoned entry.

2. **Alias canonicalization aims a typo at the real spec.**
   `canonicalize_project_aliases_in_prompt()` is workflow-agnostic: it rewrites the
   alias inside _any_ VCS tag. Verified: `#git:sase` -> `#git:gh_sase-org__sase`, and
   likewise for `#git(sase)`, `#jj:sase`, `#git!!:sase`. So a mistyped workflow tag is
   upgraded from a harmless unknown ref into a precise reference to the real ProjectSpec
   key.

3. **`resolve_git_ref()` Mode 1 falls through instead of erroring.** In
   `src/sase/workspace_provider/plugins/bare_git_ref.py:171-187`, when the project
   directory and spec file both exist, the resolver reads `BARE_REPO_DIR` and only
   returns inside `if bare_repo_dir:`. A GitHub project has no `BARE_REPO_DIR`, so
   control **silently falls through** to Mode 2 (Patch-name search, no match) and then
   to `_init_missing_project_ref(git_ref)` at line 212 — treating an existing,
   fully-populated project as _missing_.

4. **`init_bare_git_project()` clobbers the existing spec.** In
   `src/sase/workspace_provider/plugins/bare_git_init.py:104-113` it computes
   `project_file = projects_base / project_name / <project_name>.sase`, which for
   `gh_sase-org__sase` is the **real** spec, then calls `set_bare_repo_dir()` and
   `set_workspace_dir()` on it. Both helpers update in place, which is why
   `PROJECT_NAME`, `RUNNING:`, and all Patches survive and only `WORKSPACE_DIR` changes
   while `BARE_REPO_DIR` appears. Before writing, it also creates
   `~/.sase/repos/<key>.git` and clones it to `~/projects/git/<key>/`.

5. **Nothing prunes the poisoned MRU entry.** `_is_stale_known_project_prefix()` and
   `_vcs_prefix_ref_is_gone()` in `src/sase/history/vcs_xprompt_mru.py` only ask whether
   the _project_ still exists; neither compares the tag's workflow type against the
   project's actual VCS provider. The `#git:` entry for a GitHub project is therefore
   kept forever and stays cyclable.

Residue of past conversions is still on disk: `~/projects/git/gh_sase-org__sase/` (a
clone whose only commit is `Initial commit`) and `~/.sase/repos/gh_sase-org__sase.git`
(which additionally accumulated real SDD plan commits from agents that ran against the
bogus project).

Note that `src/sase/history/vcs_xprompt_mru.py:277-282` already documents that
`resolve_git_ref` creates projects as a side effect — the destructive behavior is known,
but only guarded at that one call site.

## Goals

1. Make it structurally impossible for a `#git:` ref to convert an existing non-bare-git
   project into a bare-git project.
2. Fail loudly and actionably instead of silently, so a mistyped tag is visible.
3. Stop workflow/provider-mismatched refs from being canonicalized onto a real project
   key or retained in the VCS MRU.
4. Leave a documented cleanup path for the on-disk residue.

## Non-goals

- Changing the legitimate first-use auto-init for genuinely new `#git:<name>` projects.
- Changing the bare-git partial-init recovery added in `dff269e3a` (rebuilding a bare
  repo from a matching clone, adopting a populated clone, etc.).
- Touching the GitHub plugin in the `sase-github` linked repo. All changes here are in
  this repo.

## Implementation

### 1. Guard `resolve_git_ref()` Mode 1 (primary fix)

File: `src/sase/workspace_provider/plugins/bare_git_ref.py`

In Mode 1, when `project_dir.is_dir() and project_file_path.exists()` but
`parse_bare_repo_dir()` returns `None`, do **not** fall through to Mode 2/3. Decide
provider-awareness first:

- If the existing spec is a **bare-git** project that merely lost its `BARE_REPO_DIR`
  (its `WORKSPACE_DIR` is a git checkout whose `origin` is a local filesystem path),
  keep today's healing behavior and let it re-init.
- Otherwise (the spec belongs to another provider, e.g. GitHub) raise a new
  `ProjectProviderMismatchError(ValueError)` naming the project, its detected workflow
  type, and the tag the user should have used. Render the user-facing `PROJECT_NAME`
  rather than the directory key, per the "Show Project Names, Never ProjectSpec Keys"
  convention.

The bare-git detection currently lives in
`BareGitWorkspacePlugin._is_bare_git_project()` in
`src/sase/workspace_provider/plugins/bare_git_workspace.py`, which imports from
`bare_git_ref.py`. Extract that predicate to a module-level function (in
`bare_git_ref.py` or a small shared module) and have the plugin method delegate to it,
so no import cycle is introduced. Prefer this local predicate over
`detect_workflow_type()`, which would re-enter the plugin manager.

Define `ProjectProviderMismatchError` in a location importable by both `bare_git_ref.py`
and the ref-resolution caller without a cycle.

### 2. Guard `init_bare_git_project()` (defense in depth)

File: `src/sase/workspace_provider/plugins/bare_git_init.py`

Before any filesystem or git work (i.e. before `_assess_bare_repo` /
`_assess_clone_dir`), resolve the destination spec path and, if it already exists and is
not a bare-git project, raise `ProjectProviderMismatchError`. Doing this first means a
rejected call leaves no `~/.sase/repos/<name>.git` and no `~/projects/git/<name>/`
residue.

This also protects Mode 4 (`#git:<path>`), where a bare-repo basename can collide with
an existing project of a different provider.

### 3. Surface the error instead of swallowing it

File: `src/sase/ace/tui/actions/agent_workflow/_ref_resolution.py`

`resolve_ref_from_prompt()` catches `(ValueError, RuntimeError)` and returns `None`
(lines 78-81), which would silently drop the launch's VCS resolution. Re-raise
`ProjectProviderMismatchError` so the user sees why the launch failed, following the
existing `workflow_type == "git" and ref == "home"` re-raise precedent. Confirm the
launch paths that call this render the message rather than crashing the TUI.

### 4. Make alias canonicalization provider-aware

Files: `src/sase/project_aliases.py`, `src/sase/project_alias_prompts.py`

Do not canonicalize an alias into a VCS tag whose workflow type does not match the
resolved project's actual provider. `#git:sase` must stop becoming
`#git:gh_sase-org__sase`; leave the ref untouched so it cannot address the real spec
key. Thread the provider check in through the existing injected-dependency style
(`load_alias_map` / `load_changespec_names`) rather than importing the workspace
provider registry directly into `project_alias_prompts.py`. Keep the lookup cached per
call — this runs on every launch and on `<ctrl+p>` keystrokes.

Leave `humanize_project_refs_in_prompt()` alone; it is display-only.

### 5. Prune provider-mismatched VCS MRU entries

File: `src/sase/history/vcs_xprompt_mru.py`

Add a fourth prune class to `load_launchable_vcs_xprompt_mru()`: drop an entry whose
workflow type does not match the resolved project's actual provider.
`_workflow_and_ref_for_prefix()` already yields `(workflow_type, ref)`, and the
resolvability index already carries the alias map, so the check can reuse both. Keep it
offline and side-effect-free — in particular it must not call `resolve_ref`, for the
project-creating reason the module already documents. On any error, keep the entry
rather than nuking the MRU, matching the existing conservative pattern.

Also record the same check in `record_vcs_xprompt_usage()` so a mismatched prefix is
never written back to disk.

This removes the standing `#git:gh_sase-org__sase` entry on the next MRU load.

## Tests

Add regression coverage (new tests alongside `tests/test_bare_git_init.py`,
`tests/test_bare_git_workspace.py`, `tests/test_vcs_xprompt_mru.py`, and the project
alias tests). All tests must use an isolated `SASE_HOME` **and** an isolated `HOME`,
because the default clone path is derived from `Path.home()`.

1. `resolve_git_ref("<gh-key>")` against a spec with `WORKSPACE_DIR` and `PROJECT_NAME`
   but no `BARE_REPO_DIR` raises `ProjectProviderMismatchError`, and the spec file is
   **byte-for-byte unchanged** afterward. This is the direct regression test for the
   reported bug.
2. The rejected call creates no bare repo under `<sase_home>/repos/` and no clone under
   `<home>/projects/git/`.
3. A genuinely missing `#git:<name>` shorthand still auto-initializes a new bare-git
   project (existing behavior preserved).
4. The partial-init recovery cases in `tests/test_bare_git_init.py` still pass — a
   bare-git project whose spec lost `BARE_REPO_DIR` still heals.
5. `canonicalize_project_aliases_in_prompt("#git:sase")` leaves the ref unchanged when
   `sase` resolves to a GitHub project, while `#gh:sase` still canonicalizes.
6. `load_launchable_vcs_xprompt_mru()` prunes `#git:<gh-project>` while keeping
   `#gh:<gh-project>` and `#git:home`.
7. `resolve_ref_from_prompt()` propagates `ProjectProviderMismatchError` instead of
   returning `None`.

## Manual cleanup (requires user confirmation — do not perform unprompted)

State already damaged by this bug, to be handled with the user:

- `~/.sase/vcs_xprompt_mru.json` — the `#git:gh_sase-org__sase` entry. Fix 5 removes
  this automatically on the next MRU load; no manual edit needed.
- `~/projects/git/gh_sase-org__sase/` — stray clone, only commit is `Initial commit`.
- `~/.sase/repos/gh_sase-org__sase.git` — stray bare repo that **contains real SDD plan
  commits** (`fix_sase_core_ci` plan work). Salvage anything wanted from it before
  removal. Ask the user before deleting either path.
- Confirm `~/.sase/projects/gh_sase-org__sase/gh_sase-org__sase.sase` still has
  `WORKSPACE_DIR: /home/bryan/projects/github/sase-org/sase/` and no `BARE_REPO_DIR`. It
  was correct at the time this plan was written.

Note that `rm` is aliased to `trash` in this environment; use `command rm` for a real
deletion.

## Verification

- `just check` must pass. Run `just install` first, since workspace directories are
  ephemeral.
- Run `just check-full` before landing, since this touches ref resolution, prompt
  canonicalization, and MRU pruning, which are broadly depended upon.
- Manually confirm the fix with an isolated `HOME` + `SASE_HOME` sandbox reproducing the
  chain above: the spec must be unchanged and a clear error raised.
