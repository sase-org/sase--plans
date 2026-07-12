---
create_time: 2026-07-12 15:34:39
status: wip
prompt: 202607/prompts/companion_sdd_finalizer_prompting.md
tier: tale
---
# Prompt Agents to Commit Companion SDD Repo Changes (Stop Generic Finalizer Sweeps)

## Problem

Companion SDD repos (`sase--plans`, `sase--research`) are accumulating generic
`chore(sdd): sync uncommitted SDD store changes` commits, and those commits show up in the ACE agent detail COMMITS
section with **no** file entries in the "Deltas:" metadata field and **no** viewable diff in the file panel.

Evidence gathered during diagnosis:

- One workspace's research companion clone contains **five** consecutive
  `chore(sdd): sync uncommitted SDD store changes` commits; the most recent one (`2d3849c`, `SASE_AGENT=research.a.cdx`)
  silently swept a **664-line agent-authored research report** (`202607/lumberjack_chop_configuration_audit.md`).
- A plans companion clone contains sync commit `a5fc275` (`SASE_AGENT=6w`) that swept an agent's edit to an SDD plan
  file (`202607/remove_file_panel_trim.md`).

Desired behavior (per user): except for sase's own targeted auto-commits (bead mutations, plan Add/Complete transitions,
done-plan status closeouts), companion repos should be treated **just like linked repos**: the commit finalizer should
detect dirty companion clones and prompt the agent to commit them with a meaningful message via the `/sase_git_commit`
skill, and the resulting commits should surface diffs in the TUI (Deltas field + file panel).

## Root Causes

### RC1 — The finalizer sweeps companion repos instead of prompting

`run_commit_finalizer()` (`src/sase/llm_provider/commit_finalizer.py`) calls
`_auto_commit_separate_sdd_store_if_possible(project_dir)` **unconditionally** — once before the first prompting pass
and again after every pass. That helper calls
`commit_sdd_store_files(store, "chore(sdd): sync uncommitted SDD store changes", auto_commit_type="sdd")` with
`paths=None`, which commits **everything** dirty in both the plans and research clones.

Meanwhile `collect_dirty_state()` (`src/sase/llm_provider/commit_finalizer_state.py`) only inspects the main workspace
and configured/opened **linked** repos. Companion clones (which live at the fixed workspace subpaths `sase/repos/plans`
and `sase/repos/research`) are never part of `DirtyState`, so the agent is never prompted — the sweep always wins, even
for large agent-authored content.

The sweep was introduced by `88c49a798` ("fix(sdd): auto-commit separate store mutations") as a fallback to sync bead
mutations. Bead mutations now auto-commit synchronously with proper `chore(beads): …` messages on both the normal CLI
path (`sase/bead/cli_crud.py`, `auto_commit_bead_store`) and the Rust fast path
(`src/sase/main/bead_fast_path.py::_apply_mutation_side_effects`), so the blanket sweep mostly catches agent-authored
files these days.

### RC2 — SDD auto-commit markers carry no diff, so the TUI can't render Deltas

`commit_sdd_files()` (`src/sase/sdd/_commit_store.py`) records the commit via `_record_sdd_commit_marker()` →
`record_sdd_commit_result_marker()` **without ever capturing a diff** (`diff_path=None` in the marker written to
`commit_results.json`).

On the display side (`src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`):

- The COMMITS section (`_persisted_commit_groups`) renders records without a `diff_path` — which is why the sync commit
  is visible in the screenshot.
- `agent_commit_diffs()` — the sole source for the "Deltas:" section (`_agent_deltas.py::agent_delta_entries` /
  `agent_commit_linked_delta_groups`) and for file-panel per-commit diffs — **skips any record whose `diff_path` is
  missing**.

Hence: commit visible, deltas/diff invisible. This affects not just the sweep but _all_ SDD auto-commits (bead
mutations, plan Add/Complete), which also never show diffs today.

### RC3 — Prompted companion commits would be misattributed as primary commits

`write_result_marker()` (`src/sase/workflows/commit/commit_tracking.py`) does not record a `repo_name`. Display-side
attribution falls back to cwd matching: `_commit_cwd_is_primary()` treats any cwd **inside the agent workspace tree** as
primary, only excluding paths under `agent.linked_repos`. Companion clones live _inside_ the workspace tree
(`sase/repos/plans`, `sase/repos/research`) and are not linked repos, so once agents start committing there via
`sase commit`, those commits would be grouped/merged into the **primary** repo's Deltas. This must be fixed for the
prompted flow to display correctly.

## Design

### Phase 1 — Finalizer: treat external SDD store repos like linked repos

Files: `src/sase/llm_provider/commit_finalizer_state.py`, `commit_finalizer_types.py`, `commit_finalizer_prompting.py`,
`commit_finalizer.py`.

1. Extend `_DirtyRepoKind` with a new kind (`"sdd"`).
2. In `collect_dirty_state()`, resolve the project's SDD store (`resolve_sdd_store`); when storage is `separate_repo` or
   `companion_repos`, run `git_changed_files()` on each existing clone root (plans `repo_root`, and `research_dir` when
   present) and append a `DirtyRepo` per dirty clone. Name each repo with `sdd_store_label()` (e.g.
   `sase-org/sase--plans`), falling back to the companion kind (`plans` / `research`).
3. In `build_dirty_details()`, render these with a distinct label (e.g.
   `SDD companion repo sase-org/sase--research: <path>`) and include them in the same `cd <path>` + `/sase_git_commit`
   instruction block used for linked repos (including the "verify `git status` is clean afterwards" instruction).
4. Narrow `_auto_commit_separate_sdd_store_if_possible()` to a **bead-state-only** safety net: commit only the bead
   state directory inside the plans store (`commit_sdd_store_files(..., paths=[<beads dir>], auto_commit_type="beads")`)
   with a descriptive message (e.g. `chore(beads): sync bead state`). Rationale: bead state is machine-managed and its
   per-mutation auto-commits are explicitly best-effort (exceptions swallowed), so leftovers there should never block or
   prompt an agent. Everything else — plan file edits, research reports, media — flows through the prompting passes.
5. Reorder so the narrowed sweep runs **before** dirty-state collection in each round, so the collected `DirtyState`
   reflects post-sweep reality. Dirty companion repos remaining after `max_passes` fail the finalizer exactly like
   linked repos (`CommitFinalizerError`).
6. Pass `artifacts_dir` explicitly to `commit_sdd_store_files()` from the finalizer instead of relying on the
   `SASE_ARTIFACTS_DIR` env fallback.

Notes:

- Legacy separate-repo stores (`.sase/sdd/.git`) get identical treatment (they flow through the same storage-kind
  check).
- The existing targeted auto-commits are untouched: bead mutation commits, plan Add/Complete commits in
  `workflows/commit/commit_hooks.py`, plan approval commits, and the narrow `auto_commit_done_sdd_plan_status()`
  closeout.

### Phase 2 — Capture diffs for all SDD auto-commit markers

Files: `src/sase/sdd/_commit_store.py`, `src/sase/workflows/commit/commit_tracking.py`.

1. In `commit_sdd_files()`, after staging the changed files (post `git add`, once `diff --cached` confirms staged
   content), capture the staged diff scoped to the files being committed (`git diff --cached -- <changed_files>`).
2. Persist the diff text to `$SASE_ARTIFACTS_DIR/commit_diffs/NNN.diff` using the same sequential-naming scheme as
   `capture_pre_commit_diff()`. Refactor the "allocate next diff path + write text" portion of
   `capture_pre_commit_diff()` into a shared helper in `commit_tracking.py` so both call sites use one implementation.
   Skip capture when no artifacts dir is available (matching the marker's existing no-op behavior).
3. Thread the resulting path through `_record_sdd_commit_marker()` → `record_sdd_commit_result_marker(diff_path=...)`
   (the parameter and marker schema already exist; they are just never populated).

Result: bead auto-commits, plan Add/Complete commits, and remaining bead-state sync commits all gain "Deltas:" entries
and viewable per-commit diffs in the TUI — `agent_commit_diffs()` picks them up with zero display-side changes, and
non-primary records group under the store label via `agent_commit_linked_delta_groups()`.

### Phase 3 — Correct attribution for prompted companion commits

Files: `src/sase/workflows/commit/commit_tracking.py`, `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`.

1. When `sase commit` runs with cwd inside an external SDD store clone, record an explicit `repo_name` (via
   `sdd_store_label()` / the store record) in the marker written by `write_result_marker()`. The display side already
   honors explicit `repo_name` and marks such records non-primary.
2. As a fallback for records without explicit `repo_name` (historical data), teach `_commit_cwd_is_primary()` /
   `_repo_name_for_commit_cwd()` to recognize the fixed companion clone subpaths (via `companion_repo_clone_dir()`
   relative to the agent workspace) as non-primary, mirroring the existing linked-repo carve-out.
3. Confirm `_persist_commit_diff_path()` already refuses to let a companion commit replace the primary
   `commit_diff_path` (its docstring says it does; keep a regression test).

### Phase 4 — Live file-panel deltas for dirty companion clones (running agents)

File: `src/sase/ace/tui/widgets/file_panel/_linked_deltas.py`.

Extend `_eligible_linked_workspace_candidates()` to also include the agent workspace's companion clone directories
(fixed subpaths, when they exist and are git repos), so uncommitted companion changes appear as a grouped live diff in
the Deltas/file panel _while the agent runs_ — matching linked-repo behavior and making the pre-commit state reviewable.

**The implementer MUST run `sase memory read tui_perf.md -r "..."` before this phase** — it touches the live-diff
refresh path, which is perf-sensitive. Reuse the existing per-repo caching (`_linked_delta_cache`) as-is; companion
candidates should be cheap constant-path lookups.

## Boundary Check

No Rust core (`sase-core`) changes expected: the finalizer runtime, SDD commit helpers, marker recording
(`commit_results.json` assembly in `axe/run_agent_helpers_state.py`), and TUI display are all Python in this repo, and
the marker schema already includes `diff_path`/`repo_name` fields. Re-verify during implementation that nothing consumes
these markers from `sase_core_rs`.

## Testing

- `tests/llm_provider/test_commit_finalizer_auto_sdd_status.py`: update sweep tests — non-bead companion dirt is no
  longer auto-committed; bead-state dirt still is (with the new message).
- New finalizer tests: dirty companion clones produce `DirtyRepo(kind="sdd")` entries, prompt details include the store
  label + `cd` + `/sase_git_commit` instructions, and dirt remaining after `max_passes` raises `CommitFinalizerError`.
- `_commit_store` tests: staged-diff capture writes `commit_diffs/NNN.diff` and the marker records that `diff_path`; no
  capture without an artifacts dir; diff is scoped to committed files when other files in the store are also dirty.
- `tests/test_commit_artifacts.py`: `write_result_marker` records `repo_name` for SDD-store cwds;
  `_persist_commit_diff_path` still protects the primary diff.
- Prompt-panel display tests: SDD markers with `diff_path` yield Deltas entries grouped under the store label;
  companion-cwd records without explicit `repo_name` are treated as non-primary.
- Linked-deltas tests: companion clone candidates produce live delta groups for running agents.

## Risks / Compatibility

- Agents will now be prompted (and can fail the finalizer) for companion dirt they didn't create — e.g. leftovers from
  an interrupted prior run in the same workspace. This matches linked-repo semantics and is the explicitly requested
  behavior; the bead-state safety net keeps machine-managed noise out of the prompts.
- All agent runtimes are treated uniformly; the finalizer prompting is provider-neutral and the `/sase_git_commit` →
  `sase commit` rule already applies to every runtime.
- Non-agent contexts (TUI, daemons, plan approval) are unaffected: the sweep only ever ran inside the finalizer, and
  targeted auto-commit paths keep their behavior (now with diffs).
