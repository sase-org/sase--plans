---
create_time: 2026-07-10 12:59:31
status: done
prompt: 202607/prompts/vcs_log_phantom_projects.md
tier: tale
---
# Plan: Stop `sase vcs log` from reporting phantom projects and SDD repos

## Problem

`sase vcs log -a` (and `-a -S`) reports repositories that do not correspond to any real registered project or SDD store.
In the current environment it shows `bob-plugins`, `sase-core`, `sase-github`, `sase-nvim`, and `sase-telegram` as
top-level registered projects, and with SDD inclusion it emits phantom repos plus warnings such as:

```
⚠ bob-plugins/sdd: No remote 'origin' configured. Install a VCS plugin (e.g. sase-github) for this repository.
⚠ dotfiles_1/sdd: No remote 'origin' configured. ...
⚠ sase-core/sdd: ... (also sase-github, sase-nvim, sase-telegram)
```

None of these `<project>/sdd` repos are real SDD stores, and the five "projects" above were never registered by the user
(`sase project list` does not show them).

## Root cause (two compounding defects)

### 1. Sibling bookkeeping records are treated as registered projects

`_resolve_all_project_repos()` in `src/sase/vcs_log/resolve.py` calls
`list_project_records(sase_projects_dir(), "all", include_home=False)`. The `"all"` state filter includes
`PROJECT_STATE: sibling` records. Sibling records are **auto-materialized bookkeeping specs**, not user registrations:
the workspace-open machinery (`_materialize_sibling_project_context` / `_heal_sibling_project_context` in
`src/sase/main/workspace_handler_context.py`) stamps a sibling ProjectSpec for every configured linked repo an agent
works in, so SASE can track running agents, changespecs, and skill uses there.

Evidence that they are not projects:

- `sase project list` (default `--state active`) hides them; only an explicit `-s sibling`/`-s all` reveals them.
- The Rust core attaches a "project is sibling" warning to each such record — which the resolver currently
  **suppresses** (a filter in `_resolve_record_candidates`) in order to let them through.

Consequences of including them:

- Their checkouts are classified as `kind="primary"` ("registered primaries") instead of surfacing as the linked repos
  they actually are. The dedup preference then promotes the sibling route over the owning project's linked-repo route.
- Each sibling record gets **project-level SDD-store expansion**, which is what manufactures the phantom
  `sase-core/sdd`-style entries (combined with defect 2 below).

Excluding sibling records loses no repositories: every sibling record exists precisely because its checkout is a
configured linked repo of a real project (verified: `bob-plugins` is `bob-cli`'s configured linked repo; the four
`sase-*` repos are the `sase` project's linked repos), so global discovery still reaches all of them through linked-repo
resolution — with correct `linked` provenance.

Other `list_project_records(..., "all", ...)` callers were audited and intentionally need sibling records (alias and
display-name resolution, agent launch targets, xprompt loading, ace project discovery, doctor checks); they are out of
scope.

### 2. A provider storage _policy_ is mistaken for an existing SDD store

`_resolve_sdd_repo()` in `src/sase/vcs_log/resolve.py` includes an SDD repo whenever `resolve_sdd_store(...)` reports
`storage == "separate_repo"` and the `.sase/sdd` directory exists. But `_resolve_sdd_storage()`
(`src/sase/sdd/store.py`) falls back to the **provider policy** when no store record exists — for GitHub-backed
checkouts the policy is `separate_repo`, so the storage field only says where a store _would_ live, not that one was
ever materialized.

The authoritative signal that a companion SDD store exists is a **materialized store record** (`.sase/sdd-store.json`,
via `read_sdd_store_record()` + `is_materialized_record()`), optionally cross-checked with `clone_matches_record()`
(`src/sase/sdd/_store_adoption.py`) to confirm the local clone actually belongs to that record. Checkouts with real
stores (sase, actstat, bob-cli) all have this record; the six phantom `<name>/sdd` entries are stale legacy `.sase/sdd`
clones (no record, no `origin` remote) left over from before companion-repo storage became required.

This defect is independent of `--all`: it also affects default-scope `sase vcs log -S` inside an affected project (e.g.
the phantom `dotfiles_1/sdd` belongs to the _active_ dotfiles project) and `sase vcs list`, which resolves through the
same helper with SDD inclusion hardcoded on.

## Fix

### 1. Restrict the global inventory to registered projects

In `src/sase/vcs_log/resolve.py`:

- Change the all-project inventory request from `"all"` to `("active", "inactive")` so sibling bookkeeping records are
  never enumerated as projects. Registered-but-inactive projects remain included: they are real user registrations with
  real history.
- Remove the now-unneeded "project is sibling" warning suppression in `_resolve_record_candidates`.
- Update the module/function docstrings that describe the scope as "active, inactive, and sibling".

Update the user-facing wording to match: the `--all` help text in `src/sase/main/parser_vcs.py` ("all active, inactive,
and sibling projects") and the `sase vcs log` sections of `docs/vcs.md` (and the flag description in
`docs/configuration.md` if it repeats the lifecycle wording). The new wording should say global scope covers every
registered (active or inactive) project, and that sibling checkouts still appear as linked repos of their owning
projects.

### 2. Require a materialized SDD store before logging one

Add a small read-only helper to the SDD package (exported from `sase.sdd`), e.g.
`materialized_sdd_clone(primary_workspace_dir) -> Path | None`, which returns the primary's `.sase/sdd` clone path only
when:

- `read_sdd_store_record()` yields a record for which `is_materialized_record()` is true, and
- the local clone directory exists and `clone_matches_record()` accepts it (so a stale clone pointing at the wrong
  remote is not presented as the companion store's history).

Rewire `_resolve_sdd_repo()` in `src/sase/vcs_log/resolve.py` to use this helper instead of inferring existence from
`resolve_sdd_store()`'s policy-derived storage value. `resolve_sdd_store()` itself must keep its current semantics —
materialization paths rely on the policy fallback to decide where a store _will_ live.

Because the label helper reads the same record, materialized stores keep their proper `owner/repo--sdd` labels and the
`<project>/sdd` fallback label disappears along with the phantoms.

This keeps the fix at the right layer: the project inventory stays Rust-backed (only the state filter argument changes),
and "does a materialized SDD store exist?" is answered once inside the Python SDD domain package rather than being
re-derived by each consumer (`sase vcs log`, `sase vcs list`, and any future frontend).

## Expected behavior after the fix

- `sase vcs log -a` lists actstat, bob-cli, dotfiles_1, nova, and sase as registered primaries; bob-plugins, sase-core,
  sase-github, sase-nvim, sase-telegram, and chezmoi appear (once each) as linked repos of their owning projects.
- `sase vcs log -a -S` adds only the three real companion stores (`bbugyi200/actstat--sdd`, `bobs-org/bob-cli--sdd`,
  `sase-org/sase--sdd`); all six "No remote 'origin'" SDD warnings disappear.
- Default-scope `-S` and `sase vcs list` no longer surface stale, never-materialized `.sase/sdd` clones anywhere.

## Verification

- `tests/test_vcs_log_resolve.py`: update the mocked-lifecycle fixtures so sibling records are asserted to be _excluded_
  from global enumeration while their checkouts still resolve via the owning project's linked config (kind `linked`);
  keep active + inactive inclusion coverage; add cases for a primary with a stale non-materialized `.sase/sdd` clone
  (skipped, no warning about it), a materialized record with a matching clone (included, proper label), a materialized
  record whose clone is missing or mismatched (skipped), and default-scope skipping of stale clones.
- New unit coverage for the `materialized_sdd_clone()` helper in the SDD test suite.
- `tests/main/test_vcs_parser.py`: refresh the `--all` help-text expectation.
- vcs list coverage: assert a stale clone no longer produces an SDD row.
- Run the focused resolver/collector/parser/render and SDD store suites, then `just install` + the repository-required
  `just check`.
- Real-environment smoke test: `sase vcs log -a -S -N` must show no `<project>/sdd` warnings and only the three real
  companion SDD repos.

## Non-goals

- Do not delete or migrate the stale `.sase/sdd` clones on disk; cleanup/adoption belongs to `sase sdd init` and
  (potentially) a future doctor check, not to a read-only log command.
- Do not change other `list_project_records` consumers or sibling-record materialization itself.
- Do not rename the `dotfiles_1` project or touch the `nova` project; both are genuinely registered (the odd
  `dotfiles_1` name is a pre-existing registration artifact, worth a separate look).
- No Rust core changes: the inventory API already supports per-state filtering.
