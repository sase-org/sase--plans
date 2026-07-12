---
create_time: 2026-07-08 18:21:34
status: done
prompt: 202607/prompts/agent_panel_sdd_commits.md
tier: tale
---
# Show SDD-repo commits in the Agents-tab COMMITS panel

## Problem / product context

On the **Agents** tab of the `sase ace` TUI, selecting an agent shows a metadata/prompt panel that includes a
**COMMITS:** section listing the commits that agent made (short SHA + subject), grouped by repository. Today that
section lists commits made to the **primary** workspace repo (and any linked repos the commit `cwd` maps to), but it
does **not** list commits made to the separate **SDD repo** (e.g. `sase-org/sdd`).

The most visible gap: when a plan is approved, the tale/epic/legend plan file is auto-committed into the separate SDD
repo, but that commit never appears in the plan agent's COMMITS panel. The same is true for any SDD-repo commit made on
an agent's behalf (e.g. the commit finalizer syncing SDD store changes, the exec-plan accept flow committing the
generated prompt/plan, and bead-store commits).

We want SDD-repo commits that an agent made — or that were auto-made for it — to show up as commits in that agent's
COMMITS panel, attributed to the SDD repo, alongside its primary/linked-repo commits.

## Root cause

The Agents-tab COMMITS panel is driven by a **per-agent, run-scoped marker pipeline** that is entirely separate from the
SDD commit path:

1. **Display.** `append_agent_commits_section(text, agent)` in `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`
   renders the COMMITS section from `agent.step_output["meta_commits"]` (a `list[dict]` of
   `{message, sha, cwd, diff_path}` records). Commits are grouped into repos by mapping each record's `cwd` to a repo
   name (`_repo_name_for_commit_cwd`): inside `agent.workspace_dir` -> primary repo; inside a linked repo's workspace
   dir -> that linked repo; otherwise the cwd basename (with a trailing `_<n>` stripped).

2. **Population.** `meta_commits` comes from a `commit_results.json` marker file written into the agent's
   `SASE_ARTIFACTS_DIR`. It is read by `read_commit_results_metadata()` / `extract_step_output_and_diff_path()` in
   `src/sase/axe/run_agent_helpers_state.py` (at agent done-time) and backfilled for already-done agents by
   `_enrich_missing_commit_metadata()` in `src/sase/ace/tui/models/_loaders/_done_loaders.py`.

3. **Producer.** The **only** writer of `commit_results.json` is the main-repo `sase commit` workflow:
   `write_result_marker()` in `src/sase/workflows/commit/commit_tracking.py` (via `CommitWorkflow._run_tracking_steps`).

The SDD commit path — `commit_sdd_files()` / `commit_sdd_store_files()` in `src/sase/sdd/_commit.py` — makes real git
commits to the SDD repo and even tags them with a `SASE_AGENT=<name>` trailer (so the revert-agent feature can discover
them via `git log` polling), but it **never writes a `commit_results.json` marker**. Because the panel reads only
`meta_commits`, SDD commits are invisible there.

Note there are two independent commit surfaces, and only one is affected:

- `sase vcs log` (CLI timeline) already treats the separate SDD repo as a first-class repo
  (`src/sase/vcs_log/resolve.py::_resolve_sdd_repo`) and the Rust core aggregator's `AggregatedCommitWire.repo` label
  already anticipates an "SDD store label". That path is **repo-wide git-log**, not per-agent, and is unaffected here.
- The Agents-tab panel is **per-agent** and driven by `meta_commits`. This is the surface we must fix.

## Boundary note (Rust core)

Per the repo's Rust-core boundary rule, shared backend/domain behavior belongs in `../sase-core`. This fix stays in
Python because the entire `meta_commits` machinery (marker format, marker reading, repo grouping/attribution) already
lives in Python and is **not** part of the Rust `agent_scan` scanner (that scanner reads a fixed set of marker files and
does not read `commit_results.json`). Extending the existing Python pipeline is consistent with the current
architecture; no `sase-core` wire/API change is required. If per-agent commit attribution is later promoted into core,
that is a separate migration and out of scope here.

## High-level design

Record SDD-repo commits made in an agent context as `commit_results.json` marker records in that agent's artifact dir,
so they flow through the existing `meta_commits` pipeline into the panel, attributed to the SDD repo.

Four pieces:

### 1. A narrow recorder for SDD commit markers

Add a helper (in `src/sase/workflows/commit/commit_tracking.py`, next to the existing marker code, or a small sibling
module) that appends an SDD commit to the agent's `commit_results.json` **list only** — it must reuse
`_upsert_commit_results_marker()` and must **not** overwrite the single `commit_result.json` file (that single file
backs the "primary" commit's `meta_commit_*` fields and `agent_meta.commit_result`; SDD commits are additional, not
primary).

The record should carry the fields the reader/display already understand plus an explicit repo label:

- `cwd` = the SDD repo dir (so it is naturally distinct from primary/linked cwds; dedup key `(cwd, result)` keeps
  re-runs idempotent),
- `result` / `commit_result` = the new SDD commit SHA,
- `message` = the commit message (subject drives the panel line),
- `repo_name` (new field) = the SDD repo label (e.g. `sase-org/sdd`, mirroring `sase vcs log`'s `_sdd_label`, falling
  back to `"sdd"`),
- optional `diff_path` (nice-to-have; not required for the panel, see below).

### 2. Emit the marker from the SDD commit path

Make `commit_sdd_files()` capture the new commit SHA (currently it returns only a bool; add an internal
`git rev-parse HEAD` after a successful commit, or thread the SHA out) and call the recorder when an agent artifact dir
is resolvable. Resolve the artifact dir in this priority:

- an explicitly passed `artifacts_dir` (for host-process callers, see piece 3), else
- the `SASE_ARTIFACTS_DIR` env var (covers in-agent-process callers automatically).

Also resolve the SDD repo label once (from the `SddStore` record / store label) so it can be stamped on the marker;
`commit_sdd_store_files()` already has the `SddStore` and is the better layer to compute the label, then pass it down.

This single choke point covers all agent-attributable SDD commits uniformly:

- **In-agent-process** (env var is set): commit finalizer (`src/sase/llm_provider/commit_finalizer.py`), exec-plan
  accept/generate (`src/sase/axe/run_agent_exec_plan_*.py`), bead-store commits (`src/sase/bead/`).
- **Host-process** plan approval (env var not set): pass the plan agent's artifact dir explicitly — see piece 3.

Only agent-attributable commits should be recorded; non-agent contexts (no resolvable artifact dir) simply skip
recording, exactly as `write_result_marker` already no-ops when `SASE_ARTIFACTS_DIR` is unset.

### 3. Attribute the plan-approval commit to the plan agent

The plan-approval archive-and-commit happens in the host/CLI process, not the plan agent's process, so
`SASE_ARTIFACTS_DIR` is not set to the plan agent. Both approval call sites already carry the plan agent's identity in
the notification's `action_data` (`agent_timestamp`, `agent_project_file` / `project_dir`, `agent_name`):

- `src/sase/plan_approval_actions.py::_archive_plan_for_approval`
- `src/sase/ace/tui/actions/agents/_notification_modals.py::_archive_plan_for_approval`

Resolve the plan agent's artifact dir from that data (project + workflow dir + timestamp, using the canonical
`projects_root/<project>/artifacts/<workflow>/<timestamp>` layout / existing resolver) and pass it into
`commit_sdd_store_files(..., artifacts_dir=...)`.

### 4. Surface the marker in the display (attribution + late writes)

- **Repo label passthrough.** Teach `_commit_result_list_record()` (`src/sase/axe/run_agent_helpers_state.py`) to carry
  the new `repo_name` field, and teach `_agent_commits.py` (`_repo_name_for_commit_cwd` / the grouping in
  `_persisted_commit_groups` and `agent_commit_diffs`) to prefer an explicit `repo_name` on the record when present,
  falling back to the existing cwd-based mapping. (Without this, an SDD-dir cwd of `.../.sase/sdd` would fall back to
  basename `"sdd"` — acceptable, but the explicit label matches `sase vcs log` and is robust across storage layouts.)

- **Late-written markers for done agents.** For in-agent-process SDD commits made during the run (before `done.json`),
  `extract_step_output_and_diff_path()` already captures them at done-time — no change needed. For the **plan-approval**
  case the marker is written _after_ the plan agent is done, and `_enrich_missing_commit_metadata()` today (a) only runs
  when a _primary_ commit already exists (`meta_commit_message` / `meta_new_commit`) and (b) skips when `meta_commits`
  is already populated. Relax this so a done agent's `meta_commits` is (re)hydrated/merged from `commit_results.json` on
  load even when the agent made no primary commit and even when `meta_commits` was already set at done-time (merge by
  `(cwd, sha)`; the file is small JSON). Keep the read guarded/lazy to respect TUI load performance (see
  `memory/tui_perf.md`).

## Scope decisions

- **Only separate-repo SDD commits are new.** In-tree SDD content already rides inside the agent's primary commit (so it
  already shows), and `local` (unversioned) SDD has no commits. Recording should therefore be a no-op unless a real
  commit was created (the existing `commit_sdd_files` "returns true only when a new commit is created" contract already
  gives us this gate).
- **Panel needs SHA + subject only.** `diff_path` on the SDD marker is optional; capturing an SDD diff (to also feed the
  DELTAS / diff surfaces via `agent_commit_diffs`) can be a follow-up. Landing the COMMITS-panel visibility first keeps
  the change tight.
- **Attribution target = the agent whose work produced the commit** (plan agent for a tale plan; the running agent for
  finalizer/exec-plan/bead commits). Family grouping in the panel already follows the agent's own record, so no extra
  family logic is needed.

## Suggested phasing

1. **Phase 1 — in-process SDD commits.** Recorder helper (piece 1) + emit from
   `commit_sdd_files`/`commit_sdd_store_files` via the `SASE_ARTIFACTS_DIR` fallback (piece 2) + repo-label
   passthrough/display attribution (piece 4, first bullet). This makes finalizer / exec-plan / bead SDD commits appear
   with no enrichment change, since they are captured at done-time.
2. **Phase 2 — plan-approval retroactive.** Pass the plan agent's artifact dir from both `_archive_plan_for_approval`
   call sites (piece 3) + relax `_enrich_missing_commit_metadata` to re-hydrate `meta_commits` for done agents (piece 4,
   second bullet). This delivers the headline "tale plan file" case.

## Testing

- Unit-test the recorder: appending an SDD marker upserts `commit_results.json` (dedup by `(cwd, sha)`), leaves
  `commit_result.json` untouched, and stamps `repo_name`.
- Unit-test display attribution: a `meta_commits` record with `repo_name` groups under that label; without `repo_name`,
  an SDD cwd falls back to `"sdd"`; primary/linked grouping is unchanged.
- Unit-test enrichment: a done agent with no primary commit but an SDD marker in `commit_results.json` gets
  `meta_commits` hydrated on load; an agent with both primary and SDD markers shows both, ordered primary-first.
- Integration-ish test for `commit_sdd_files`: committing in a temp SDD git repo with `SASE_ARTIFACTS_DIR` set writes
  the marker; unset -> no marker.
- Run `just check` (after `just install`) before finalizing. Add/adjust PNG snapshot goldens only if the panel's
  rendered COMMITS layout changes intentionally.

## Key files

- Display: `src/sase/ace/tui/widgets/prompt_panel/_agent_commits.py`
- Marker read + `meta_commits` build: `src/sase/axe/run_agent_helpers_state.py`
- Done-agent enrichment: `src/sase/ace/tui/models/_loaders/_done_loaders.py`
- Marker writer / recorder home: `src/sase/workflows/commit/commit_tracking.py`
- SDD commit path: `src/sase/sdd/_commit.py` (`commit_sdd_files`, `commit_sdd_store_files`)
- Plan-approval commit sites: `src/sase/plan_approval_actions.py`,
  `src/sase/ace/tui/actions/agents/_notification_modals.py`
- SDD label reference: `src/sase/vcs_log/resolve.py::_sdd_label`, `src/sase/sdd/store.py` (`SddStore` record)
