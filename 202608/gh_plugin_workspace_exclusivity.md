---
tier: epic
title: Extend the workspace-exclusivity guarantee to the sase-github plugin
goal: "The sase-github plugin acquires workspaces the same way the rest of SASE does:
  every gh__setup and GitHub-submit allocation is a single atomic claim whose result is
  checked and whose ledger record names its caller, a pinned n=<num> target that a live
  agent holds fails with that agent named instead of being taken, and no gh workflow
  step stashes, pulls, or checks out inside a checkout another live agent occupies.

  "
parent_bead: sase-q0
phases:
  - id: gh_atomic
    title: Atomic, checked workspace acquisition in the sase-github plugin
    depends_on: []
    size: medium
    description: "gh_atomic: replace the check-then-claim allocation in gh__setup and in
      the GitHub submit path with a single atomic claim, check every claim result, make
      a pinned n=<num> target a single-shot claim that names the occupant on failure,
      release the slot when materialization fails, and give every claim and release a
      caller tag.

      "
  - id: gh_guard
    title: Refuse gh workflow steps that would prepare an occupied checkout
    depends_on:
      - gh_atomic
    size: medium
    description: "gh_guard: write the per-checkout occupant record when gh__setup takes
      a workspace, clear it on release, and require the occupancy decision before
      gh__prepare stashes, gh__checkout checks out, or the GitHub submit path checks
      out, so a conflicting occupant fails the run instead of losing another agent's
      work.

      "
proposed_by: bbugyi200.athena.sase-q0.land
status: done
bead_id: sase-q0.5
create_time: 2026-09-09 19:50:35
---

- **PROMPT:**
  [prompts/202608/gh_plugin_workspace_exclusivity.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/gh_plugin_workspace_exclusivity.md)
- **PARENT:** [202608/workspace_exclusivity.md](workspace_exclusivity.md)
- **BEAD:**
  [sase-q0.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-q0/sase-q0.5.md)

# Plan: Extend the workspace-exclusivity guarantee to the sase-github plugin

## Why this exists

Epic `sase-q0` ("Guarantee one agent per workspace") landed four phases in the `sase`
repo and in `sase-core`:

- every RUNNING-field mutation is recorded to a durable ledger with before/after
  occupancy and a caller tag (`src/sase/logs/workspace_claim_ledger.py`);
- deferred allocation claims atomically **before** materializing a checkout, and a
  pinned family-attach target is a single-shot checked claim
  (`src/sase/axe/run_agent_phases.py`);
- every managed checkout carries a `.sase/occupant.json` record, and every destructive
  preparation in the runner asks `sase_core`'s `decide_workspace_occupant_conflict`
  before it cleans or resets (`src/sase/core/occupancy_guard.py`);
- a report-only `workspace.occupancy_conflicts` doctor check surfaces conflicts.

The epic plan named the per-phase VCS setup steps `gh__setup` / `gh__prepare` /
`gh__checkout` as a required guard call site — those are the steps `06e--code` ran
against the shared checkout at 13:19:19 during the incident. The `guard` phase agent
searched the `sase` repo, found no such hooks, and recorded a follow-up asking the land
agent to confirm. The land agent confirmed the opposite: those steps are real, and they
live in the **`sase-github` linked plugin repo**, which no phase of the epic ever swept.
Open it with `sase repo open sase-github -r "<reason>"` and use the printed path.

The same sweep found a second missed site in that repo: the GitHub submit path still
uses the check-then-claim pattern that phase `sase-q0.2` converted in its bare-git
sibling.

`sase-telegram`, `sase-nvim` and `sase-research-artifacts` were swept and contain no
claim, release, or workspace-preparation call sites. `sase-github` is the only gap.

## What is actually broken

### 1. `src/sase_github/scripts/gh_setup.py` — the `gh__setup` step

```python
elif n is not None:
    workspace_num = n
    workspace_dir = ensure_workspace_checkout(resolved.primary_workspace_dir, workspace_num)
else:
    workspace_num = get_first_available_axe_workspace(project_file)
    workspace_dir = ensure_workspace_checkout(resolved.primary_workspace_dir, workspace_num)

materialize_sdd_store(workspace_dir, workspace_num)
pid = os.getppid()
...
if not pre_allocated:
    claim_workspace(project_file, workspace_num, workflow_name, pid, None, pinned=not release)
```

Four defects, all of them the exact shapes the epic already fixed elsewhere:

- **Check-then-claim (`n is None`).** `get_first_available_axe_workspace` reads the
  RUNNING field under a lock, drops it, and returns a number;
  `ensure_workspace_checkout` then materializes the clone and `materialize_sdd_store`
  runs — seconds of unlocked work — before `claim_workspace` writes the row. This is the
  identical TOCTOU window `claim_next_axe_workspace` was written to eliminate and that
  phase `sase-q0.2` closed in `claim_deferred_workspace`.
- **The claim result is discarded.** `claim_workspace` returns a result object whose
  `success` is never inspected. A racer that loses the window proceeds anyway, inside
  the winner's workspace, with no error.
- **The pinned `n=<num>` branch never checks availability at all.** `#gh <ref> n=13`
  materializes and then claims workspace 13 whether or not a live agent holds it — the
  same hole phase `sase-q0.2` closed for `SASE_AGENT_DEFERRED_TARGET_WORKSPACE_NUM`.
- **No `caller_tag` and no occupant record.** These mutations land in the ledger with a
  null caller tag, so the `ledger` phase's "attributable after the fact" guarantee has a
  blind spot exactly where a `#gh` run collides with an agent. And because no
  `.sase/occupant.json` is written, a `#gh` workflow is invisible to the guard that
  protects every agent runner.

Agent launches are **not** exposed: the launcher sets `SASE_GH_PRE_ALLOCATED=1` (see
`get_pre_allocated_env_prefix` and `claim_deferred_workspace`), and that branch neither
selects nor claims. The exposure is direct `#gh` invocations and any non-pre-allocated
path.

### 2. `src/sase_github/workspace_plugin.py` — `ws_submit_changespec` (~line 539)

```python
workspace_num = get_first_available_axe_workspace(patch_file)
...
ws_dir, _ = get_workspace_directory_for_num(workspace_num, project_basename)
...
if not claim_workspace(patch_file, workspace_num, workflow_name, pid, patch_name):
    return (False, f"Failed to claim workspace #{workspace_num}")
```

The claim result _is_ checked here, so a losing racer aborts rather than proceeding —
this is less severe than `gh_setup.py`. But it is still the non-atomic pattern, and its
bare-git sibling `src/sase/workspace_provider/plugins/bare_git_submit.py:159` was
converted to `claim_next_axe_workspace_dir` by phase `sase-q0.2` (commit `75e1db1ef`).
The two plugins should not disagree about how a workspace is acquired. The subsequent
`provider.checkout(branch_name, ws_dir)` is also an unguarded destructive operation.

### 3. `src/sase_github/xprompts/gh.yml` — the `prepare`, `checkout`, and `release` steps

- `prepare` runs `git stash push --include-untracked` and `git pull --rebase` in the
  resolved workspace. `--include-untracked` takes another agent's in-flight work out of
  its tree; this is destructive preparation in the sense the `guard` phase means, and
  nothing checks occupancy first.
- `checkout` runs `provider.checkout(...)` plus another `git pull --rebase`, likewise
  unguarded.
- `release` calls `release_workspace(...)` with no `caller_tag`, and never clears the
  occupant record the setup step should have written.

## `gh_atomic`: atomic, checked workspace acquisition

Everything below happens in the `sase-github` repo, reached through
`sase repo open sase-github -r "<reason>"`; use the path it prints as the only path for
reads and writes. Mirror the shapes the `sase` repo already uses rather than inventing
new ones — read `src/sase/axe/run_agent_phases.py` (`_claim_next_deferred_workspace`,
`_claim_pinned_deferred_workspace`, `_describe_workspace_occupant`,
`_format_workspace_occupant`) and
`src/sase/workspace_provider/plugins/bare_git_submit.py` first.

### `gh_setup.py`

- **Unpinned branch (`n is None`)**: claim first, materialize second. Use
  `sase.running_field.claim_next_axe_workspace(project_file, workflow_name, pid, cl_name=..., pinned=not release, caller_tag="gh-setup")`
  to get the number under one lock, then call `ensure_workspace_checkout` and
  `materialize_sdd_store`. If either raises,
  `release_workspace(..., caller_tag="gh-setup")` the slot before the error propagates,
  so a failed setup cannot leak a workspace. Do **not** switch to
  `claim_next_axe_workspace_dir`: it resolves through `get_workspace_directory_for_num`,
  while this plugin must keep resolving through
  `ensure_workspace_checkout(resolved.primary_workspace_dir, ...)`. Take the atomic
  claim helper and keep the existing materialization.
- **Pinned branch (`n is not None`)**: make it a single-shot, checked claim. Inspect the
  `claim_workspace` result; on failure, print a message naming the current occupant
  (pid, liveness, workflow, artifacts timestamp — copy the shape of
  `_describe_workspace_occupant` / `_format_workspace_occupant`) and fail the step
  non-zero instead of continuing. Do not retry a pinned number, and do not materialize
  before the claim is held.
- **Never discard a claim result** on any branch.
- Keep the `pre_allocated` branch exactly as it is: the launcher has already claimed,
  and re-claiming there would be wrong.

### `workspace_plugin.py`

- Convert `ws_submit_changespec` to
  `claim_next_axe_workspace_dir(patch_file, workflow_name, pid, project_basename, cl_name=patch_name, caller_tag="gh-submit")`
  inside a `try/except WorkspaceClaimError`, exactly as `bare_git_submit.py:159` does,
  dropping the `get_first_available_axe_workspace` + `get_workspace_directory_for_num` +
  `claim_workspace` trio. Keep the existing `finally: release_workspace(...)` and add
  `caller_tag="gh-submit"` to it.

### `gh.yml`

- Add `caller_tag="gh-release"` to the `release` step's `release_workspace` call.

### Tests

Follow the conventions in `sase-github`'s existing `tests/test_workspace_plugin.py`:

- an unpinned setup never selects a number that is claimed in the RUNNING field at claim
  time;
- a materialization failure after a successful claim releases the slot;
- a pinned `n=<num>` target already held by a live pid fails with the occupant named and
  does not double-claim;
- `ws_submit_changespec` acquires through `claim_next_axe_workspace_dir` and still
  releases in its `finally`;
- ledger records written by these paths carry the `gh-setup` / `gh-submit` /
  `gh-release` caller tags.

## `gh_guard`: refuse gh workflow steps that would prepare an occupied checkout

Depends on `gh_atomic`, because it extends the same claim block in `gh_setup.py`. Read
`src/sase/core/occupancy_guard.py`, `src/sase/workspace_provider/occupant.py`, and
`src/sase/axe/run_agent_runner_setup.py` (`_guard_workspace_not_occupied`) first.

- **Write the occupant record** in `gh_setup.py` after the claim is held and the
  checkout is materialized, using `sase.workspace_provider.occupant.new_occupant_record`
  / `write_occupant_record`, with the same `pid` the claim used (`os.getppid()`, for the
  reason the existing comment in that file gives) plus the workflow label, project,
  workspace number, and ref as `cl_name`. Follow `claim_deferred_workspace`'s guard of
  writing the record only for a real numbered workspace, not for the primary/placeholder
  checkout.
- **Guard before handing the checkout to the workflow.** Call
  `sase.core.occupancy_guard.ensure_workspace_not_occupied(workspace_dir, project_file=..., caller=OccupancyCaller(...))`
  once the workspace is resolved and before `gh_setup.py` prints `_chdir`, so an
  occupied checkout fails the `gh__setup` step with the occupant named rather than
  letting `gh__prepare` stash someone else's work. Let `WorkspaceOccupiedError` surface
  as a non-zero step failure. One guard there covers both `prepare` and `checkout`,
  which run in that same directory immediately afterwards, so no bash-level guard is
  needed inside `gh.yml`. Run the guard on the `pre_allocated` branch too — the launcher
  has already written its own occupant record, and the decision recognises the caller's
  own lineage.
- **Guard the submit checkout**: call `ensure_workspace_not_occupied` on the resolved
  `ws_dir` in `ws_submit_changespec` before `provider.checkout(branch_name, ws_dir)`.
- **Clear the occupant record on release**: in `gh.yml`'s `release` step call
  `sase.workspace_provider.occupant.clear_occupant_record(<workspace_dir>)` under the
  same `should_release` condition. The setup step already exports `workspace_dir`, so
  thread it into the release step's template.

### Tests

- `gh__setup` writes `.sase/occupant.json` for a numbered workspace, and the release
  step clears it;
- setup refuses with `WorkspaceOccupiedError` when the checkout's occupant record names
  a different live pid, and proceeds when the record is absent, is this caller, or names
  a dead pid — the same case matrix `tests/test_core_occupancy_guard.py` uses in the
  `sase` repo;
- the refusal happens before any git mutation runs;
- `ws_submit_changespec` refuses a checkout held by a different live agent.

## Verification

- `just check` (or the equivalent gate) in the `sase-github` repo, plus its focused
  workspace/plugin suites.
- `just install` then `just check` in the `sase` repo, since `sase-github` is a required
  plugin installed into the workspace venv and a signature change there breaks
  `tools/setup_required_plugins` consumers.
- `sase doctor --check workspace.occupancy_conflicts` (or the full `sase doctor`) should
  stay clean.
- Manual: run `#gh <ref>` twice concurrently against one project and confirm from
  `~/.sase/logs/workspace_claims.jsonl` that the two runs received distinct workspace
  numbers, that both records carry the `gh-setup` caller tag, and that neither cleared
  or stashed the other's tree. Then run `#gh <ref> n=<a number a live agent holds>` and
  confirm it fails with that agent named instead of stashing its work.

## Out of scope

- The `sase` repo's own allocation and guard paths: verified complete during the
  `sase-q0` landing and unchanged by this plan.
- `WorkspaceClaimLine::parse` dropping rows with unrecognized trailing parts — filed as
  task `sase-qa`.
- `sase workspace open-clean`'s unguarded `prepare_workspace` — filed as task `sase-qc`.
- Centralizing occupant-record clearing inside `running_field/_operations.py` —
  considered and declined during the `sase-q0` landing; see that epic's landing note.
