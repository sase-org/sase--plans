---
tier: epic
title: One workspace per agent family
goal:
  Close the workspace-collision hole in which a `#gh:`/`#git:` agent works in a second,
  turn-scoped VCS workspace lease that the rest of SASE never learns about. Make the VCS
  workflow adopt the runner's existing numbered claim instead of allocating another one,
  propagate the launcher's pre-allocation across shell follow-up launches, rebind the
  runner's single workspace identity when a VCS workflow legitimately does allocate,
  refuse to release or un-occupy a workspace the agent's family still holds, and add
  doctor and regression coverage that fails when one live pid holds two numbered claims.
phases:
  - id: adopt-runner-workspace
    title: VCS setup adopts the runner's existing workspace
    depends_on: []
    size: medium
    description:
      "adopt-runner-workspace: make the `#git:` and `#gh:` setup steps adopt the live
      numbered RUNNING claim already held by the calling runner instead of allocating a
      second workspace, treating adoption exactly like the existing launcher
      pre-allocation branch (no second claim, no duplicate occupant record,
      `should_release=false`), while leaving explicit `n=<num>` pinning and genuinely
      workspace-less callers on their current allocate path."
  - id: preallocate-shell-followups
    title: Pre-allocation survives shell follow-up launches
    depends_on: []
    size: medium
    description:
      "preallocate-shell-followups: record the starter's VCS workflow ref in shell
      member meta and thread it through `launch_shell_followup`,
      `spawn_family_successor`, and `spawn_detached_child` so a gate-, monitor-, or
      proc-shell follow-up whose composed prompt still carries `#gh:`/`#git:` is spawned
      with the `SASE_<VCS>_PRE_ALLOCATED` env the launcher already knows how to emit,
      instead of silently re-running VCS setup with no pre-allocation."
  - id: single-workspace-identity
    title: One workspace identity per runner
    depends_on:
      - adopt-runner-workspace
    size: medium
    description:
      "single-workspace-identity: when a VCS workflow legitimately allocates a workspace
      for a runner that had none (deferred/`#0` launches), rebind the runner's own
      workspace identity to it so `done.json`, the checkout occupant record, monitor
      start, and shell member meta all name the directory the agent actually works in,
      rather than leaving `workspace_num` pointing at the launcher's slot while only
      `step_output.meta_workspace` knows the truth."
  - id: handoff-safe-vcs-release
    title: VCS release never frees a workspace the family still holds
    depends_on:
      - single-workspace-identity
    size: medium
    description:
      "handoff-safe-vcs-release: make the `#git:`/`#gh:` release step identity-checked
      and handoff-aware -- release only a claim this run's pid still owns, clear only an
      occupant record naming this run, and skip both entirely when the turn ended by
      handing off mechanically to a monitor, gate, proc shell, pipe, or plan proposal
      whose follow-up will continue in the same checkout."
  - id: one-workspace-invariant-coverage
    title: Coverage for the one-workspace invariant
    depends_on:
      - adopt-runner-workspace
      - preallocate-shell-followups
      - single-workspace-identity
      - handoff-safe-vcs-release
    size: small
    description:
      "one-workspace-invariant-coverage: add a doctor check that reports any single live
      pid holding more than one numbered RUNNING claim for a project, plus regression
      tests that replay the incident shape -- a gate-shell follow-up carrying `#gh:`, a
      monitor handoff, and a subsequent pool allocation -- and assert exactly one
      numbered claim per runner and no release of a checkout whose family is still live."
proposed_by: bbugyi200.athena.0ft
status: done
bead_id: sase-vd
create_time: 2026-09-09 19:50:59
---

- **PROMPT:**
  [prompts/202608/one_workspace_per_agent_family.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/one_workspace_per_agent_family.md)
- **BEAD:**
  [sase-vd](https://github.com/sase-org/sase--beads/blob/main/pages/sase-vd/README.md)

# Plan: One workspace per agent family

## Outcome

A SASE agent runner holds exactly one numbered workspace, and every mechanism that needs
to know which one -- the RUNNING claim, `done.json`, the checkout occupant record, the
monitor/gate/proc shell handoff, and the follow-up successor -- agrees on the same
number for the whole family lifetime. No workspace that still holds a live agent
family's working tree is ever returned to the free pool.

## The incident this plan closes

On 2026-08-28, agent `sase-um.9.4` started working in workspace `#23` while that
checkout still held agent `0fq`'s work. The `~/.sase/logs/workspace_claims.jsonl` ledger
and the `#23` checkout's git reflog give the whole chain:

1. `16:58:31` -- the launcher spawns `0fq--code` (pid 100538) from a plan-approval gate
   shell and takes a `spawn-claim` on workspace `#12` (`ace(run)-260828_165830`).
2. `16:58:39` -- the follow-up prompt the gate shell composed still begins with
   `#gh:gh_sase-org__sase`, so the `gh` embedded workflow's `setup` step runs. The
   gate-shell spawn passed no `vcs_ref`, so no `SASE_GH_PRE_ALLOCATED` env was emitted
   and `gh_setup` fell through to `claim_next_axe_workspace`, claiming a **second**
   workspace `#23` under workflow `gh-gh_sase-org__sase` with `should_release=true` and
   `caller_tag="gh-setup"`. `setup` printed `_chdir=<workspace 23>`, so the LLM ran in
   `#23` while the agent record, `done.json`, and every shell mechanism still said
   `#12`.
3. `17:20:34` -- `0fq--code` ends its turn by handing off to a monitor. Because the
   runner's identity is `#12`, `sase monitor start` transferred the `#12` claim and ran
   `just check-full` in `#12` -- not the tree the agent had actually edited. The
   follow-up `0fq--1` later resumed in `#12` as well.
4. `17:20:35` -- the `gh` workflow's post-prompt `release` step ran unconditionally,
   calling `release_workspace(..., 23, "gh-gh_sase-org__sase", ...)` and
   `clear_occupant_record(<workspace 23>)`. Both halves of the sase-q0 occupancy guard
   were erased and `#23` went back into the free pool while it still held `0fq`'s
   uncommitted work.
5. `17:22:06`-`17:22:15` -- `sase-um.9.4` (pid 3038800), a deferred-workspace agent
   whose bead wait had just cleared, took a `lease(bead_claim)` on the lowest free
   number, `#23`, destructively prepared it (the `#23` reflog shows
   `branch: Reset to origin/master` at `17:22:08`, and `git stash list` in that checkout
   still holds two `gh_sase-org__sase-ace` stashes), released the lease, and then
   `deferred-claim`ed `#23` for the run. Two agents, one checkout.

The occupancy guard from epic sase-q0 could not help: `gh_setup` correctly calls
`ensure_workspace_not_occupied`, but by `17:22` the RUNNING row was gone and the
occupant marker had been deleted by the release step, so both sources of truth honestly
reported the checkout as free. The defect is the **lease lifetime and identity**, not
the guard.

This is not a rare path. The ledger for 2026-08-28 alone contains 35
`gh-setup`/`gh-release` records: every gate-shell or shell follow-up launch whose
composed prompt still carries a VCS tag double-allocates the same way. `0fr--code` was
doing it concurrently -- holding `#19` as `ace(run)-...` while its LLM child ran with
`--cwd` pointed at `#20`.

## Invariants preserved by every phase

- A runner holds exactly one numbered workspace claim. Two live RUNNING rows with the
  same pid and two different numbered workspaces is a bug, not a mode.
- The workspace a runner's LLM actually works in is the workspace named by its RUNNING
  claim, its `done.json`, its checkout occupant record, and the member meta every shell
  kind hands to its follow-up. `step_output.meta_workspace` may restate that number but
  must never be the only place it is correct.
- A workspace is released only by the mechanism that owns the claim, and only when no
  follow-up in the same agent family will continue in that checkout. Ending a turn by
  creating a monitor, gate, proc shell, pipe, or plan proposal is a handoff, not a
  completion.
- The occupancy guard's two sources of truth stay honest: a RUNNING row and an occupant
  marker are cleared together, by the run that owns them, and never on behalf of a
  different pid.
- Workspace `#0` keeps its meaning as the primary checkout / deferred placeholder.
  Adoption logic applies only to numbered pool workspaces (`UNIFIED_MIN_WORKSPACE` and
  above); a runner at `#0` still legitimately allocates.
- Behavior changes land in both VCS implementations together. `#git:` setup/release live
  in this repo at `src/sase/scripts/git_setup.py`; `#gh:` setup/release live in the
  `sase-github` linked repo (open it with `/sase_repo`) at
  `src/sase_github/scripts/gh_setup.py` and its sibling release step. They are
  deliberate mirrors and must not drift.

## Phase details

### 1. VCS setup adopts the runner's existing workspace

Both setup scripts already have exactly the branch this needs: when
`SASE_GIT_PRE_ALLOCATED` / `SASE_GH_PRE_ALLOCATED` is set they reuse the launcher's
workspace, write no second claim, write no duplicate occupant record, and print
`should_release=false`. The bug is that the flag is the _only_ way to reach that branch.

Add a second, environment-independent way in: before falling through to
`claim_next_axe_workspace`, look up the calling runner's own live claim. Both scripts
already use `os.getppid()` as the claiming pid precisely because setup is a short-lived
subprocess of the runner, so the runner's row is found by scanning
`get_claimed_workspaces(project_file)` for a claim whose `pid` is that parent and whose
`workspace_num` is a numbered pool slot. When such a row exists, resolve its directory,
materialize the SDD store, and take the pre-allocated branch: `should_release=false`, no
new claim, and no occupant record rewrite (the launcher already wrote one naming the
same lineage).

Leave the other two paths alone. An explicit `n=<num>` pin is a deliberate request for a
specific workspace and must keep its current claim-or-fail behavior with the occupant
named on failure. A caller with no numbered claim at all -- a runner at `#0`, a deferred
launch before its claim lands, a chop, a bare CLI invocation -- still allocates, and
phase 3 makes that allocation the runner's real identity.

Keep `ensure_workspace_not_occupied` in front of `prepare`/`checkout` on the adoption
branch, matching how the existing pre-allocated branch already runs it, and keep the
"release what this step claimed" error handling scoped to the branch that actually
claimed something. Add tests for: adoption when the parent holds a numbered claim; no
adoption when the parent's only claim is `#0`; no adoption when `n=` is given; and
`should_release=false` on every adopted run.

### 2. Pre-allocation survives shell follow-up launches

`spawn_agent_subprocess` builds the pre-allocation env from its `vcs_ref` argument
through `_preallocated_workspace_env`, and `spawn_detached_child` already accepts
`vcs_ref` and forwards it -- but `spawn_family_successor` has no such parameter, and
neither the gate-shell nor the monitor follow-up `_spawn` closure supplies one. Every
shell follow-up therefore spawns with `vcs_ref=None`, which is precisely how `0fq--code`
was launched. `run_agent_retry_spawn` already does this correctly and is the reference
shape to copy.

Record the starter's VCS workflow ref (workflow type plus ref, the same
`tuple[str, str]` the launcher uses) in shell member meta when the shell is created,
alongside the workspace number and directory already stored there. Thread it through
`launch_shell_followup` in `src/sase/shells/followup.py`, add a `vcs_ref` parameter to
`spawn_family_successor`, and pass it from the gate-shell and monitor follow-up spawn
closures. Where a shell has no recorded ref -- older shells, and follow-ups composed
without a VCS tag -- recover it from the composed prompt using the registry's
`get_embedded_vcs_tag_pattern()` rather than guessing, and pass `None` when there is
genuinely no VCS workflow to pre-allocate for.

The three degraded workspace fallbacks `launch_shell_followup` already implements
(transfer, fresh claim on the same number, `#0`) must keep working: the pre-allocation
env has to describe the workspace the follow-up actually got, so build it from the
number and directory the successful spawn used, never from the number the shell hoped
for. Add tests asserting a `#gh:`-carrying gate-shell follow-up is spawned with
`SASE_GH_PRE_ALLOCATED=1` and matching `_WORKSPACE_NUM`/`_WORKSPACE_DIR`, that a
degraded `#0` fallback advertises `#0`, and that a non-VCS follow-up sets none of the
three variables.

### 3. One workspace identity per runner

When a VCS workflow does legitimately allocate -- a runner that started at `#0`, which
is the normal shape for deferred and land agents -- the allocation currently reaches
only two places: `os.chdir()` plus `SASE_ACTIVE_PROJECT_DIR` via `apply_chdir_output`,
and `step_output.meta_workspace`. The runner's own
`RunnerRunState.workspace_num`/`workspace_dir`, parsed once from argv, never learns
about it. That split is why `done.json` recorded `workspace_num: 12` for a run whose
work was in `#23`, and why the monitor ran the landing gate in the wrong tree.

Make the workflow executor's `_chdir` application publish the new workspace identity,
and have the runner rebind its own state from it: the number and directory used for
`done.json`, for the occupant record it writes, for the artifacts/agent metadata the TUI
loads, and for the member meta every shell kind records. `sase monitor start` already
resolves a workspace number for its cwd through `_resolve_monitor_workspace_num` and
`resolve_workspace_num_for_dir`; once the runner's identity is correct, the monitor's
`request.cwd` and lane workspace agree and the existing claim-transfer logic in
`sase/monitor/start_claim.py` moves the right slot. Verify that path rather than adding
a parallel one.

The rebind must be a move, not a copy: after it, exactly one numbered claim names the
runner's pid. If the runner arrived holding a placeholder `#0` row, it is released as
the numbered claim is taken -- the `deferred-placeholder-release` / `deferred-claim`
pair already in the ledger is the existing precedent for that sequencing, and the pair
must stay atomic enough that no window exists where the runner holds neither. Add tests
covering: `done.json` naming the allocated workspace; the occupant record naming it; a
monitor started after a `_chdir` transferring the allocated claim; and a follow-up
successor resuming in the same checkout.

### 4. VCS release never frees a workspace the family still holds

The release step is a plain post-prompt step that runs whenever the turn ends. It must
become conditional on two independent facts.

First, identity. `release_workspace` matches on workspace number, workflow, and
`cl_name` -- not on pid -- so a release can remove a row that a different process now
owns. Refuse to release unless the current RUNNING row for that number still names this
run's pid, and refuse to `clear_occupant_record` unless the marker on disk names this
run. A mismatch is a no-op plus a ledger record with an explicit reason, never a silent
removal; both cases must be visible in `workspace_claims.jsonl` so a future incident
stays attributable.

Second, handoff. A turn that ends by creating a monitor, gate shell, proc shell, pipe,
or plan proposal has not finished with its checkout -- a family successor will continue
there. The runner already knows which of these happened, because each terminates the
turn mechanically; surface that as an explicit "the workspace was handed off" signal the
release step consults, and skip release entirely when it is set. After phase 1 and phase
3 most runs reach release with `should_release=false` anyway; this phase makes the
remaining ones safe rather than relying on that.

Mirror the change in `sase-github`'s release step and keep the two implementations
byte-for-byte equivalent in behavior. Add tests for: release skipped under each handoff
kind; release refused when the RUNNING row's pid differs; occupant record left intact
when it names another pid; and a normal completing turn still releasing exactly once.

### 5. Coverage for the one-workspace invariant

Add a doctor check next to the existing occupancy-conflict reporting in
`src/sase/doctor/checks_workspace.py` and
`src/sase/workspace_provider/occupancy_conflicts.py`: for each project, report any
single live pid that holds more than one numbered RUNNING claim, annotated with the
ledger's last mutation timestamp and caller tag for each row the same way the existing
conflict codes are. Report-only, never auto-repair, consistent with the sase-q0 detect
phase. Run it against the live host state during verification -- before this epic lands
it should fire on real double-claim rows, and after it should be silent.

Add regression tests that replay the incident shape end to end with fakes rather than
live agents: a gate-shell follow-up whose composed prompt carries `#gh:` produces
exactly one numbered claim; a monitor handoff transfers that same claim and runs in that
same checkout; and a pool allocation issued while the family is still live cannot select
that workspace. Add a focused test asserting the `workspace_claims.jsonl` record shapes
the doctor check depends on, so the ledger stays the authoritative post-mortem surface
it was built to be.

## Verification and landing

Every phase runs `just check` before handing off, per the two-speed verification rule;
`just check-full` is the landing gate and runs from a monitor, not inline. Ephemeral
workspace clones need `just install` first. Phases 1, 2, and 4 change files in the
`sase-github` linked repo as well as this one -- open it with `/sase_repo`, and cover
both repositories in the turn's final declaration.

The landing agent must additionally confirm on the live host that
`sase agent list --json` and the project's RUNNING field agree on one numbered workspace
per running agent, and that the new doctor check reports no double-claimed pids.

## Out of scope

- Recovering `0fq`'s clobbered work. It survives as two `gh_sase-org__sase-ace` stashes
  in the `#23` checkout and is a manual, one-time recovery, not epic work.
- Changing how `lease(bead_claim)`, `lease(plan-archive)`, and `lease(chop:...)`
  operational leases destructively prepare the checkout they claim. That behavior is
  correct for a slot they legitimately own; this epic stops the slot from being wrongly
  free, which is what made the preparation destructive here.
- Reworking `remove_vcs_workspace_claims` and the TUI's `meta_workspace` reconciliation.
  Those exist to paper over the two-workspace shape; once phases 1 through 3 land they
  should become dead weight, but retiring them is follow-up work, and a task should be
  filed for it rather than folded in here.
