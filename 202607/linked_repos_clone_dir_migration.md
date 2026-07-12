---
create_time: 2026-07-12 11:47:51
status: done
prompt: 202607/prompts/linked_repos_clone_dir_migration.md
tier: tale
---
# Migrate Normal Linked Repo Clones to `sase/repos/linked/` (with Launch-Time Clearing + Fast Restore Cache)

## Problem & Goals

Today every host-scoped clone lives directly in `<workspace_checkout>/sase/repos/<name>`, mixing two very different
kinds of repos:

- **Companion repos** — the SDD storage repos (`<project>--plans`, `<project>--research`). These are managed by the SDD
  store, must persist across agent runs, and are prepared/committed by SDD machinery.
- **Normal linked repos** — explicitly configured `linked_repos` entries (e.g. `sase-core`, `sase-github`, `sase-nvim`,
  `sase-telegram` in this project's `sase.yml`, plus the global `chezmoi` entry, and `bob-plugins` for the bob-cli
  project). Agents are supposed to access these only through the audited `sase workspace open -r "<reason>"` flow.

Because normal linked clones persist across launches, an agent can wander into a stale clone from a previous run without
ever running the audited `sase workspace open` command, and the clone's state (branch, staleness) is whatever the last
run left behind.

Goals:

1. Move normal linked repo clones to `<workspace_checkout>/sase/repos/linked/<name>`. Companion repos stay at
   `<workspace_checkout>/sase/repos/<name>`.
2. Clear the launched workspace's `sase/repos/linked/` directory during agent-launch workspace prep — exactly the one
   workspace directory being claimed for the launch, before every sase agent launch.
3. Keep `sase workspace open` fast despite the clearing: avoid paying full clone cost on every open.
4. **No backward-compatibility code.** Old clone locations are cleaned up once, out-of-band, on this machine (see Phase
   4).

## Survey of Current Machinery (what the implementation must touch)

- `src/sase/linked_repos.py` is the choke point for clone-path knowledge:
  - `LINKED_REPO_CLONES_SUBDIR = ("sase", "repos")` and `LEGACY_LINKED_REPO_CLONES_SUBDIR = (".sase", "workspaces")`.
  - `linked_repo_clone_dir()` / `resolve_linked_repo_clone_dir()` (legacy fallback) / `_prepare_linked_repo_clone_dir()`
    (legacy→canonical migration) / `_linked_repo_clone_location()` (layout matcher) /
    `materialize_linked_repo_workspace()` (clone via `ensure_git_clone_at`).
  - `_resolve_workspace_dir()` resolves each configured entry to the host-scoped clone path.
- Companion-repo consumers of the same helper (must keep resolving to `sase/repos/<name>`):
  - `src/sase/sdd/store.py`, `src/sase/sdd/_companion_init.py`, `src/sase/sdd/migrate.py` (each has a private
    `_companion_clone_dir()` that calls `resolve_linked_repo_clone_dir`).
  - `src/sase/main/sdd_handler.py` (`roots[kind]`), `src/sase/doctor/checks_config_sdd.py`
    (`_regressed_split_companion_paths`), `src/sase/llm_provider/commit_finalizer.py` (companion clone probe).
- Default companion linked-repo entries are injected by `inject_default_linked_repos()` in
  `src/sase/_linked_repo_config.py` (`<project>--plans` auto_clone, `<project>--research` lazy), marked with
  `_DEFAULT_LINKED_REPO_MARKER`. The authoritative companion repo names live in the SDD store record
  (`.sase/sdd-store.json` → `companions.plans.repo` / `companions.research.repo`).
- `sase workspace open` → `handle_open_clean()` + `resolve_checkout_path()` in
  `src/sase/main/workspace_handler_list.py`: materializes the host-scoped clone, then runs `prepare_workspace()`
  (stash_and_clean + checkout default revision + sync) and records the audited open marker.
- Agent-launch workspace prep:
  - `src/sase/axe/run_agent_runner_setup.py` — `prepare_workspace_if_needed()` (called from both the normal and the
    deferred-claim paths in `run_agent_runner.py`; skipped for retry-spawn children and home mode) and
    `prepare_linked_repo_workspaces_if_needed()` (re-materializes `auto_clone` repos at launch).
  - `src/sase/axe/run_workflow_runner.py` — inline `prepare_workspace()` call for workflow launches.
  - `src/sase/axe/run_agent_exec_retry.py` — mid-run retry/fallback re-preps (must NOT clear; see design).
- The per-clone git exclude entry `/sase/repos/` (written by `ensure_git_info_exclude_entry`) already covers any new
  subdirectory beneath `sase/repos/`, so no exclude changes are needed.
- Rust core (`sase-core`) only passes `linked_repos` metadata through opaquely (`agent_scan/scanner.rs` coerces the
  `linked_repos` list from `agent_meta.json`); it has no clone-path knowledge. **This is a Python-only change** per the
  Rust core backend boundary litmus test.
- TUI/marker/env consumers (`_agent_commits.py`, `_linked_deltas.py`, opened-workspace markers, linked-repo env vars)
  all consume resolved absolute paths from metadata, not hardcoded layout — only a stale code comment in
  `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py` mentions the literal layout.

## Design

### 1. Split the clone namespace: companions vs. normal linked repos

New layout inside every numbered host workspace checkout:

```
sase/repos/<name>                # companion repos ONLY (<project>--plans, <project>--research) — unchanged
sase/repos/linked/<name>         # normal linked repo clones — cleared on every agent launch
sase/repos/.linked-cache/<name>  # internal pristine-clone cache (see §3) — never exposed via env/markers
```

Concrete changes in `linked_repos.py`:

- `LINKED_REPO_CLONES_SUBDIR` becomes `("sase", "repos", "linked")`; add
  `COMPANION_REPO_CLONES_SUBDIR = ("sase", "repos")` and a cache subdir constant.
- Add a public `companion_repo_clone_dir(host_checkout, name)` helper; `linked_repo_clone_dir()` now returns the
  `linked/` path. All SDD/doctor/finalizer companion call sites switch to the companion helper.
- Add a companion classifier used by resolution paths (`_resolve_workspace_dir()` in `linked_repos.py` and
  `resolve_checkout_path()` in `workspace_handler_list.py`): a linked-repo name is a _companion_ iff it matches one of
  the host project's SDD companion repo basenames from the SDD store record, falling back to the default
  `{project}--plans` / `{project}--research` names when no record exists. Name-based classification (rather than the
  `_DEFAULT_LINKED_REPO_MARKER` injection marker) guarantees SDD machinery and linked-repo resolution always agree on
  one clone path even when a companion is explicitly listed in `linked_repos` config.
- **Remove all backward-compatibility path code**: `LEGACY_LINKED_REPO_CLONES_SUBDIR`,
  `_legacy_linked_repo_clone_dir()`, the legacy fallback in `resolve_linked_repo_clone_dir()` (which collapses into the
  plain path helpers), and the legacy-migration half of `_prepare_linked_repo_clone_dir()`.
  `_linked_repo_clone_location()` is updated to recognize only the two new layouts (companion 2-segment, linked
  3-segment).

### 2. Clear `sase/repos/linked/` during agent-launch workspace prep

Add `clear_linked_repo_clones(workspace_dir)` to `linked_repos.py`:

- For each child directory of `<workspace_dir>/sase/repos/linked/`, `os.rename()` it into
  `sase/repos/.linked-cache/<name>` (deleting any stale cache entry of the same name first — the outgoing clone is
  always newer). Delete any leftover stray entries so `linked/` ends up empty. Renames are O(1) on the same filesystem,
  so launch prep cost is negligible.
- Optionally prune cache entries whose names are no longer among the currently configured linked repos.

Call sites (clearing happens once per launch, for exactly the claimed workspace):

- `prepare_workspace_if_needed()` in `run_agent_runner_setup.py`, immediately after a successful `prepare_workspace()`.
  This covers both agent-runner call sites (normal launch and deferred-workspace claim) and inherits the existing skips:
  home mode, no update target, and retry-spawn children (which must preserve the parent's workspace, including its
  opened linked clones).
- The workflow-launch prep in `run_workflow_runner.py` (same position).
- Explicitly **not** called from: `handle_open_clean()` (`sase workspace open` is the restore path), the linked-repo
  prep loop in `prepare_linked_repo_workspaces_if_needed()`, and the mid-run retry/fallback re-preps in
  `run_agent_exec_retry.py` (clearing there would delete clones the running agent already opened; a retry is the same
  launch).

Ordering note: clearing runs before `prepare_linked_repo_workspaces_if_needed()`, so `auto_clone` repos (e.g.
`sase-core` here) are re-materialized fresh right afterwards — via the fast restore path below.

### 3. Fast restore: stash/restore cache instead of full re-clone

The point of clearing is that an agent never finds a ready-made linked clone without going through the audited
`sase workspace open` flow (or `auto_clone` launch prep) — not that we burn a full clone each time. So the clones
"deleted" from `sase/repos/linked/` are stashed into the internal cache, and materialization restores them by rename:

- In `materialize_linked_repo_workspace()` (used by both `sase workspace open` and `auto_clone` launch prep): when the
  target `linked/<name>` is missing and `.linked-cache/<name>` exists, `os.rename()` the cache entry back into place
  before calling `ensure_git_clone_at()`. A rename that fails with `FileNotFoundError`/`ENOTEMPTY` (parallel subagents
  racing to open the same repo) simply falls through — `ensure_git_clone_at()` already handles an existing valid clone,
  and rejects+recreates a corrupt one (`git status` probe → `rm -rf` → fresh clone).
- Freshness is guaranteed by the existing flow, not by the cache: every restore is followed by `prepare_workspace()`
  (stash_and_clean + checkout of the default parent revision + `sync_workspace`), which is exactly what makes a restored
  clone equivalent to a brand-new one.
- Cache miss (first open after the one-time cleanup, new repo, or pruned entry) pays the same cost as today: a local
  `git clone` from the primary path (object hardlinks) + fetch.
- Net disk usage is unchanged vs. today: each repo's clone exists either live in `linked/` or stashed in the cache,
  never both.

### 4. One-time cleanup of old clone locations on this machine

No code reads the old locations after this change, so stale clones are dead weight to delete once. For each project
reported by `sase project list` (currently: `actstat`, `bob-cli`, `sase`):

1. Determine the project's normal linked repo names: `linked_repos` from the project's `sase.yml` plus global config
   (`~/.config/sase/sase.yml` — currently adds `chezmoi`). Companions (`<project>--plans`, `<project>--research`) are
   excluded — they stay in place.
2. Enumerate every workspace checkout for the project (`sase workspace list --json` per project, plus the primary
   workspace dir), and delete `<checkout>/sase/repos/<name>` for each normal linked repo name found there.

Observed inventory during planning: the sase project's numbered workspaces hold `sase-core`, `sase-github`, `sase-nvim`,
`sase-telegram`, and `chezmoi` clones; one bob-cli workspace holds `bob-plugins`; actstat's workspaces hold companions
only. Primary workspace dirs hold companions only (linked repos resolve to their real primary paths there, never
clones).

Safety: perform the deletion after the code change lands, ideally while no other agents hold workspace claims (check
`sase project list` CLAIMS). Even if a running agent loses a clone, the next `sase workspace open` re-materializes it.
This cleanup also naturally "seeds" the new world: the first open of each repo per workspace pays one full clone, and
every launch after that hits the rename-fast cache.

## Implementation Phases

### Phase 1 — Path split + no-compat removal

- Rework `linked_repos.py` constants/helpers as in §1 (companion vs. linked helpers, classifier, legacy code removal,
  `_linked_repo_clone_location` update).
- Switch companion call sites (`sdd/store.py`, `sdd/_companion_init.py`, `sdd/migrate.py`, `main/sdd_handler.py`,
  `doctor/checks_config_sdd.py`, `llm_provider/commit_finalizer.py`) to the companion helper.
- Update `resolve_checkout_path()` in `main/workspace_handler_list.py` to classify the opened repo (companion vs.
  normal) against the host project.
- Fix the layout comment in `ace/tui/widgets/prompt_panel/_agent_commits.py`.

### Phase 2 — Launch-time clearing + restore cache

- Add `clear_linked_repo_clones()` + cache constants to `linked_repos.py`; wire it into `prepare_workspace_if_needed()`
  and the workflow-runner prep (§2).
- Add cache-restore to `materialize_linked_repo_workspace()` with the race fallbacks (§3).

### Phase 3 — Tests + docs

- New unit tests: path helpers and companion classification; clearing (stash-to-cache semantics, empty `linked/` result,
  no-op when absent, cache-entry replacement); restore (cache hit restores via rename without invoking `git clone`,
  corrupt cache entry falls back to fresh clone, rename-race fallback); runner wiring (clearing runs before `auto_clone`
  re-materialization; not run for retry-handoff children or mid-run retries; `sase workspace open` does not clear).
- Update existing tests that encode the old layout (`tests/test_linked_repos.py`,
  `tests/test_run_agent_runner_setup.py`, `tests/main/test_init_workspace_handler.py`, SDD store/migrate/companion
  tests, and any others found via a repo-wide `sase/repos` grep).
- Update docs that describe linked clone locations (`docs/workspace.md`, `docs/configuration.md`, `docs/sdd.md`,
  `docs/sdd_storage.md`, `docs/init.md`, `docs/cli.md`, `docs/beads.md`, `docs/commit_workflows.md`,
  `docs/project_spec.md`) — linked-clone references become `sase/repos/linked/<name>`; companion references are
  unchanged. Update the `parser_init.py` help string mentioning "linked repository clones under sase/repos/".
- Run `just check`.

### Phase 4 — One-time machine cleanup

- Execute the deletion described in §4 across all `sase project list` projects.

## Non-Goals / Explicitly Out of Scope

- No change to companion repo layout, SDD storage records, or SDD machinery behavior.
- No new CLI subcommands or options; `sase workspace open` semantics (audited reason, prepare-on-open) are unchanged.
- No Rust core (`sase-core`) changes.
- No behavior change for `workspace_num <= 1` (linked repos still resolve to their real primary paths — never clones).
- No backward-compatibility fallbacks for old clone locations (per requirements), including removal of the existing
  `.sase/workspaces` legacy-migration window.

## Risks & Edge Cases

- **`auto_clone` launch overhead**: `sase-core` (auto_clone) is cleared and re-materialized every launch — but via cache
  rename + the same `prepare_workspace()` it gets today, so launch latency is essentially unchanged.
- **Parallel opens**: two subagents opening the same linked repo concurrently race on the cache rename; the loser falls
  through to `ensure_git_clone_at()` against the winner's clone. Covered by tests.
- **Companion name collisions**: a companion repo can never be named `linked` or `.linked-cache` in practice (companions
  are `<project>--plans`/`<project>--research`); no guard needed beyond the classifier.
- **Cleanup while agents run**: deleting another workspace's linked clones mid-run degrades to a re-open/re-clone;
  scheduled for a quiet moment regardless.
- **Pre-existing red gates**: `just check` has known pre-existing failures on this machine (llm_provider
  `default_effort` tests, pyvision `ChangeSpecProjectFile`); verify failures against a clean baseline before attributing
  them to this change.
