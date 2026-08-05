---
tier: epic
title: Stop silently losing agent bead closes written in ephemeral workspaces
goal: A `sase bead close` run by an agent inside a numbered workspace either reaches
  the canonical bead store or fails loudly. The CLI never prints `✓ Closed` for a
  mutation that exists only in a workspace-local sidecar clone, workspace eviction
  can never destroy unpublished canonical bead commits, and the post-completion finalizer
  verifies published state instead of the local projection.
phases:
- id: publish
  title: Make every bead-store mutation publication-verified before the CLI reports
    success
  depends_on: []
  size: medium
  description: 'publish: after a bead mutation commits, verify the commit actually
    reached the canonical remote; force a synchronous publish when it did not, and
    fail the command loudly when it still cannot publish, instead of returning 0 on
    a workspace-local-only write.'
- id: evict
  title: Refuse to evict a workspace sidecar clone that holds unpublished bead commits
  depends_on: []
  size: medium
  description: 'evict: teach the launch-time workspace bead safety net about the sidecar-repos
    layout and wire it into the eviction path so `clear_workspace_repos` can never
    trash a `sase--beads` clone with unpushed canonical bead commits.'
- id: finalize
  title: Verify published bead state in the post-completion finalizer
  depends_on:
  - publish
  size: small
  description: 'finalize: make the finalizer''s bead safety net publish and verify
    rather than only commit, and stop instructing agents to confirm a close with a
    command that reads the local projection.'
proposed_by: bbugyi200.athena.t9
create_time: 2026-08-05 15:45:37
status: wip
bead_id: sase-fb
---

- **PROMPT:** [prompts/202608/bead_close_publication_loss.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/bead_close_publication_loss.md)
- **BEAD:** [sase-fb](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fb/README.md)

# Plan: Stop silently losing agent bead closes written in ephemeral workspaces

## Problem

An epic phase worker closed its assigned bead, the CLI confirmed the close, the agent's own verification step confirmed
the close a second time, the agent reported completion — and the bead was still `in_progress` for the project owner, who
had to close it by hand.

This is not a reporting mistake by the agent. The close really happened; it happened in a git clone that was deleted 123
seconds later, and every read-back the agent performed was served from that same doomed clone.

The failure is silent at every layer, and the prescribed phase-worker workflow makes it _more_ likely, not less (see
root cause 2).

## Evidence

Reconstructed from `tool_calls.jsonl` in the agent's artifact directory, the managed-sync logs in
`~/.sase/bead_push_logs/`, the `sase--beads` sidecar history, and the workspace sidecar clone's git reflog. All times
UTC, 2026-08-05.

| Time                | Event                                                                                                                                                                |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 19:09:18 – 19:18:19 | Agent runs four `sase bead note <bead> 'PROPOSED FOLLOW-UP: …'` commands. All four land in the canonical store.                                                      |
| 19:18:25            | Managed sync worker runs against the **workspace** bead clone: `integrated: true, pushed: true`. Integration marker refreshed.                                       |
| 19:19:18            | Agent runs `sase bead close <bead> --note "<4.2 KB verification note>"`.                                                                                             |
| 19:19:41            | Command exits 0, printing `✓ Closed` and `+ Noted`.                                                                                                                  |
| 19:19:42            | A managed sync worker starts — against the **primary** checkout, not the workspace clone: `integrated: false, pushed: true`. Nothing new is published.               |
| 19:20:03            | The post-completion finalizer's verification runs `sase bead show <bead>`, which prints `[CLOSED]`, `Closed at: 2026-08-05T19:19:19Z`.                               |
| 19:20:42            | Agent completes and reports success.                                                                                                                                 |
| 19:21:22            | The workspace's `sase--beads` clone reflog records `clone: from github.com:sase-org/sase--beads.git` — the clone holding the close commit was evicted and recreated. |
| 19:28:19            | The project owner closes the bead by hand.                                                                                                                           |

`sase bead history <bead>` today shows the four `note_appended` events (19:09:18, 19:09:29, 19:09:41, 19:18:19) and then
jumps straight to `issue_closed` at **19:28:19** — the owner's manual close. There is no 19:19:19 close event, and the
4.2 KB close note the agent wrote is gone. The mutation was not merely delayed; it was destroyed.

## Root causes

### 1. The bead mutation's publish hands off to a lane that pushes a different checkout

`bead_store_mutation` in `src/sase/bead/cli_common.py` commits with `push_after_commit=False`, then calls
`_push_committed_bead_store()` → `push_sdd_store_after_commit()` in `src/sase/sdd/_commit_store.py`. For a sidecar-repos
store — which is exactly what the `sase--beads` sidecar is — that function takes the enqueue-only branch:

```python
if store.storage == SDD_STORAGE_SIDECAR_REPOS and store.sidecar_role:
    ...
    enqueue_sidecar_push_publication(
        project_key=target.project_key, project=target.project, sidecar_kind=store.sidecar_role
    )
    return
```

The queued request records `project_key`, `project`, and `sidecar_kind`. It does **not** record which checkout holds the
unpublished commit. When `sase_chop_sidecar_publication.py` drains it, `_resolve_sdd_store(target)` resolves the store
from `target.primary_checkout` and pushes the **primary** sidecar checkout. That checkout has nothing new, so the drain
reports success while the agent's commit sits untouched in the workspace clone. The 19:19:42 sync log is that drain.

### 2. The only mechanism that ever publishes a workspace bead clone is a TTL-gated freshness refresh

`src/sase/main/entry.py` runs `schedule_current_bead_refresh()` in a `finally` after every `sase bead` subcommand. That
path is a _freshness_ optimization, not a publication guarantee:

```python
def _maybe_schedule_bead_refresh(beads_dir):
    if bead_refresh_mode() != "background":
        return None
    repo_root = _find_git_root(beads_dir)
    if repo_root is None or _integration_is_fresh(repo_root):
        return None
    return push_bead_work_launch_async(beads_dir)
```

`integration_is_fresh` compares the marker mtime against `_DEFAULT_REFRESH_TTL_SECONDS = 120`
(`src/sase/sdd/_integration_marker.py`). The four notes landed only because the TTL happened to be stale when they ran.
The close ran 73 seconds after the 19:18:25 sync marked integration — inside the TTL — so no push was scheduled at all.

This inverts the intended safety property: **the more recently an agent touched its bead, the less likely its close is
to be published.** The `bd/work_phase_bead` prompt tells phase workers to record `PROPOSED FOLLOW-UP:` notes and _then_
close, so the prescribed workflow reliably warms the TTL immediately before the one mutation that carries a completion
contract.

### 3. Nothing verifies publication, and eviction destroys the evidence

- `handle_bead_close` prints `✓ Closed` from `mutation.project.last_mutation_outcome` — the local write. Publication is
  never consulted.
- `sase bead show` reads the same local store, so the agent's verification and the finalizer's "verify bead `<id>` is
  closed before invoking the commit skill" instruction (`src/sase/commit_instructions.py`) both confirm a state that
  exists nowhere else.
- `prepare_launch_workspace_repos` in `src/sase/axe/runner_workspace.py` calls `clear_workspace_repos()`
  (`src/sase/_linked_repo_workspaces.py`), which renames `<workspace>/sase/repos` into `<workspace>/.sase/trash/` and
  deletes it in the background, then re-clones. That is the 19:21:22 reflog entry.
- The guard built for precisely this hazard, `_protect_unpushed_sidecar_bead_commits`, never fires for this layout. Its
  `_top_level_beads_dir(repo_root)` only checks `<repo_root>/beads` and in-tree bead state at `<repo_root>`; the sidecar
  store lives at `<repo_root>/sase/repos/beads` in a _separate_ git repository, so the helper returns `None` and the
  guard returns `True` having protected nothing. Worse, `prepare_launch_workspace_repos` — the function that actually
  evicts — never calls the guard at all.

## Scope of the bug

This is not specific to one bead. Every phase-bead and task-bead worker closing its own bead from a numbered workspace
is exposed whenever the integration marker is fresh at close time. The observable symptom is always the same: the agent
truthfully reports a close that the owner never sees.

## Design decisions

1. **Verify above the publication layer, not inside it.** The fix is implemented in the bead CLI mutation lane
   (`bead_store_mutation` / `_apply_mutation_side_effects`), which asks "did the commit I just made reach the remote?"
   after the configured push policy has run. This keeps the fix correct whether publication is queued, async, or inline,
   and keeps it off the code that epic `sase-fa` is actively rewriting (see Sequencing).

2. **Bead CLI mutations publish synchronously.** A mutation that carries a completion contract must not depend on a
   detached worker or a background drain. The mutation lane forces a synchronous `push_bead_work_launch` against the
   store that actually holds the commit when the post-policy check finds unpublished bead commits. Async remains
   acceptable for non-CLI callers (launch checkpoints already have their own push paths).

3. **Loud failure beats silent success.** When a bead mutation commits locally but cannot be published, the command
   prints an unmistakable stderr diagnostic naming the unpublished commit count, the store path, the managed-sync log
   path, and the remediation command — and exits non-zero for mutating verbs. An agent that believes it closed its bead
   when it did not is worse than an agent that is told the close failed.

4. **Eviction is a hard barrier, not a best-effort warning.** A workspace sidecar clone with unpublished canonical bead
   commits is never trashed. Publish first; if publication fails, retain a recovery ref and refuse the eviction,
   matching the existing `_protect_unpushed_sidecar_bead_commits` contract rather than inventing a second one.

5. **No new sidecar publication mechanism.** This epic adds verification and a barrier. It does not introduce another
   queue, lane, or chop.

## Sequencing against epic `sase-fa`

Epic `sase-fa` ("Revert async sidecar publication so `sase commit` publishes sidecars inline again") is in progress and
owns the enqueue-only branch described in root cause 1: its `commit` phase restores inline publication and its `queue`
phase drops the `sidecar_push` request kind entirely.

- Do **not** re-fix root cause 1 here. When `sase-fa` lands, `push_sdd_store_after_commit` falls through to
  `push_bead_work_launch(store.repo_root)` and the wrong-checkout push disappears.
- Root causes 2 and 3 survive `sase-fa` untouched. A TTL-gated freshness refresh is still not a publication guarantee,
  an unverified `✓ Closed` is still a lie when the push fails for any reason (network, non-fast-forward, lock
  contention), and the eviction guard is still blind to the sidecar-repos layout.
- The `publish` phase must therefore be written to sit _above_ whatever `push_sdd_store_after_commit` does at
  implementation time. Read that function before starting; if `sase-fa`'s `commit` phase has already landed, the
  verification still applies unchanged.
- If a phase agent finds its target code already rewritten by `sase-fa` in a way that fully satisfies one of this epic's
  acceptance criteria, it records that on its phase bead rather than duplicating the change.

## Phases

### Phase `publish` — publication-verified bead mutations

**Files:** `src/sase/bead/cli_common.py`, `src/sase/main/bead_fast_path.py`, `src/sase/bead/sync.py`,
`src/sase/bead/_sync_diagnostics.py`, `src/sase/bead/cli_crud.py`.

1. Add a publication-verification helper to the bead sync surface — for example
   `verify_bead_store_published(beads_dir) -> BeadPublicationStatus` — built on the existing
   `unpushed_bead_commit_count(repo_root, beads_dir)` in `src/sase/bead/_sync_diagnostics.py`. It reports the resolved
   repo root, the store path, the unpublished commit count, and whether the store even has an upstream to publish to. A
   store with no upstream, an in-tree store, or a read-only store is "not applicable" and must not be treated as a
   failure.
2. In `bead_store_mutation` (`src/sase/bead/cli_common.py`), after `_push_committed_bead_store()` returns, run the
   verification. When unpublished bead commits remain, force one synchronous `push_bead_work_launch(beads_dir)` against
   the store that holds the commit and re-verify.
3. When commits still remain unpublished, emit a single high-visibility stderr diagnostic naming the unpublished commit
   count, the store path, the managed-sync log path when one exists, and the remediation command; propagate a non-zero
   exit for mutating verbs.
4. Apply the same verification to the Rust fast-path lane in `_apply_mutation_side_effects`
   (`src/sase/main/bead_fast_path.py`), which calls `auto_commit_bead_store(message)` and today swallows every
   exception. Both lanes must share the helper — a mutation must not be verified on one path and unverified on the
   other.
5. Make `_print_close_results` (`src/sase/bead/cli_crud.py`) truthful: `✓ Closed` is printed only for a
   verified-published close. An unpublished close prints a distinct, unmissable failure line.
6. Tests: a regression test that reproduces the exact scenario — a bead store clone with an upstream, a fresh
   integration marker (so the TTL gate suppresses the background refresh), a close mutation, and an assertion that the
   command either publishes the commit or exits non-zero with the diagnostic. Add a companion test for the fast-path
   lane and one for the no-upstream/in-tree "not applicable" case, which must stay silent and exit 0.

### Phase `evict` — eviction barrier for unpublished bead commits

**Files:** `src/sase/axe/runner_workspace.py`, `src/sase/_linked_repo_workspaces.py`.

1. Fix `_top_level_beads_dir` (`src/sase/axe/runner_workspace.py`) so it discovers the sidecar-repos layout at
   `<workspace>/sase/repos/beads` in addition to `<repo_root>/beads` and the in-tree store. The sidecar clone is its own
   git repository, so `_protect_unpushed_sidecar_bead_commits` must resolve the sidecar's own repo root via
   `bead_store_git_root` rather than assuming the store lives inside the primary workspace repo;
   `_beads_dir_belongs_to_repo` needs a matching adjustment so the sidecar case is accepted instead of rejected.
2. Call the guard from the eviction path. `prepare_launch_workspace_repos` invokes
   `clear_workspace_repos(workspace_dir, workspace_num)` before re-cloning and never consults the guard;
   `clear_workspace_repos` renames `<workspace>/sase/repos` into `<workspace>/.sase/trash/` and deletes it in the
   background. The guard must run before that rename and must be able to refuse it.
3. On refusal, keep the established contract: attempt a synchronous publish first; if commits remain, create the
   recovery ref via `_retain_current_head_recovery_ref` and fail the preparation with a diagnostic naming the recovery
   ref, rather than warning and proceeding.
4. Tests: a test that seeds a workspace sidecar bead clone with an unpushed commit whose upstream is unreachable, runs
   the eviction path, and asserts the clone is neither trashed nor re-cloned and the recovery ref exists. A companion
   test asserting that a fully published sidecar clone is evicted normally, so the barrier does not wedge routine
   launches.

### Phase `finalize` — verified completion

**Files:** `src/sase/llm_provider/commit_finalizer.py`, `src/sase/commit_instructions.py`,
`src/sase/default_config.yml`.

1. The finalizer's bead safety net (`src/sase/llm_provider/commit_finalizer.py`, the helper that commits leftover
   external bead state with `chore(beads): sync bead state`) currently commits and stops. Extend it to publish and
   verify using the `publish` phase's helper, so a finalizer-created bead commit cannot repeat this failure.
2. `build_commit_instruction_message` (`src/sase/commit_instructions.py`) tells the agent to "verify bead `<id>` is
   closed before invoking the commit skill". Reword so the verification is meaningful — direct the agent at published
   state and at the diagnostic the `publish` phase emits, rather than at a bare `sase bead show` that reads the local
   projection.
3. Review the `bd/work_phase_bead` and `bd/work_task` prompt bodies in `src/sase/default_config.yml` for the same
   local-read assumption and update them only if the wording actually misleads. Do not restate mechanics the CLI now
   enforces.
4. Tests: assert the finalizer safety net publishes and surfaces an unpublished-state failure, and update any
   instruction-text assertions the rewording touches.

## Acceptance criteria

1. Running `sase bead close <id> --note "…"` in a numbered workspace whose bead-store integration marker is fresh either
   publishes the close to the canonical `sase--beads` remote before returning, or exits non-zero with a diagnostic
   naming the unpublished commit count, the store path, and the remediation command.
2. `✓ Closed` never appears for a close that exists only in a workspace-local clone.
3. Workspace preparation cannot trash a sidecar bead clone that holds unpublished canonical bead commits; it publishes
   them, or retains a recovery ref and fails.
4. Both the Python mutation lane and the Rust fast-path mutation lane are verified by the same helper.
5. Regression tests reproduce the TTL-fresh close-in-workspace scenario and the eviction scenario, and fail against the
   current code.
6. `just check` passes.

## Non-goals

- Redesigning sidecar publication. Epic `sase-fa` owns that; this epic verifies whatever it produces.
- Changing the bead event model, the `issues.jsonl` projection, or bead ID resolution.
- Retroactively recovering the lost close event and its note. The commit was deleted with the clone; the bead has since
  been closed manually.
- Reworking the TTL-gated background refresh into a publication mechanism. It stays a freshness optimization;
  publication becomes the mutation lane's own responsibility.

## Adjacent issue found while diagnosing (out of scope)

`_MUTATING_VERBS` in `src/sase/main/bead_fast_path.py` is `{"+1", "close", "create", "open", "ref", "rm", "update"}` and
does not include `note`, so `_is_mutating_verb("note")` is `False` and a `sase bead note` fast-path run skips
`assert_bead_store_write_sandboxed` even though it writes an event and commits. This is a write-sandbox guard bypass,
not a cause of the loss described above. The land agent should file it through `/sase_new_task` rather than folding it
into any phase here.
