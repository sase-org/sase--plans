---
tier: epic
title: Durable plan-archive publication for approved epic plans
goal: "Approved plan files always reach the plans sidecar remote: the epic-launch
  archive path publishes its plans-repo commits, workspace re-provisioning can no longer
  destroy the only copy of an unpushed plans commit, and every plan lost to this defect
  (starting with 202609/unified_agents_across_machines.md) is backfilled and verified
  against the sidecar remote.

  "
phases:
  - id: launch-plan-push
    title: Publish plans-repo commits made by the epic launch path
    depends_on: []
    size: medium
    description:
      "launch-plan-push: make `sase bead work <plan.md>` publish its plan archive and
      bead-link commits to the plans sidecar remote instead of leaving them local-only,
      and surface publication failures instead of swallowing them."
  - id: prep-clone-protection
    title: Protect unpushed sidecar commits from workspace re-provisioning
    depends_on: []
    size: medium
    description:
      "prep-clone-protection: find the code path that deletes and re-creates a
      workspace's sase/repos/<role> sidecar clones, and extend the existing
      publish-or-rescue protection (today bead-store-only) to every sidecar role so an
      unpushed plans commit can never be silently destroyed."
  - id: plan-archive-doctor
    title: Detect and repair missing archived plans
    depends_on: []
    size: medium
    description:
      "plan-archive-doctor: add a doctor-style check that cross-references plan beads,
      sidecar links/ indexes, and the canonical plans directory against the plans
      sidecar, reporting approved plans whose archived copy is missing, with a confirmed
      repair mode that re-archives recoverable plans (restoring bead_id frontmatter) and
      pushes."
  - id: backfill-athena
    title: Backfill the lost plans on this machine and verify
    depends_on:
      - launch-plan-push
      - plan-archive-doctor
    size: small
    description:
      "backfill-athena: run the repair for every recoverable missing plan on this
      machine — 202609/unified_agents_across_machines.md first — verify the sidecar
      remote now serves them with correct bead_id frontmatter, and record follow-ups for
      plans only recoverable from another machine."
proposed_by: bbugyi200.athena.0i2
bead_id: sase-z2
create_time: 2026-09-09 19:52:19
status: wip
---

- **PROMPT:**
  [prompts/202609/durable_plan_archive_publication.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/durable_plan_archive_publication.md)
- **BEAD:**
  [sase-z2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-z2/README.md)

# Plan: Durable plan-archive publication for approved epic plans

## Problem

The approved epic plan `202609/unified_agents_across_machines.md` (epic bead
`sase-xe.16.11.7`) exists in the canonical plans directory and is linked from its bead,
but the plan file was never committed to the plans sidecar repository
(`sase-org/sase--plans`). The sidecar holds an orphaned link index
(`links/202609/unified_agents_across_machines.md.json`) for a plan markdown file it does
not contain. This is one instance of a systemic defect: the sidecar currently has **57
orphaned `links/<month>/<plan>.md.json` entries** whose plan markdown is missing, and
several hundred canonical plan files that never reached the sidecar at all.

## Root cause (diagnosed, with durable evidence)

The loss chain has four links:

1. **Epic approvals delegate archiving to the launch, by design.** The epic-plan gate
   response for this plan (interaction request `d45b62e0-ccae-4bd2-a09d-a1a4ec4cbbd4`)
   recorded `plan_archive_owner: "none"` and `plan_archive_state: "not_requested"`.
   `prepare_plan_terminal_response` in `src/sase/plan_approval_actions.py` only runs the
   host-owned synchronous archive (`sase._plan_archive_approval`, which pushes and
   verifies) for `commit`/`tale` actions. For `action == "epic"` the archive is
   deliberately left to the host-owned epic launch.

2. **The epic launch commits the plan locally but never pushes the plans repo.** The
   launch proc (`s5b6fr4cwdaa`) ran
   `sase bead work ~/.sase/plans/202609/unified_agents_across_machines.md` and reported
   `✓ Archived ... (committed)` with exit code 0. Under the hood:
   - `commit_plan_file` in `src/sase/bead/cli_work_from_plan_store.py` commits the
     archived plan with `push_after_commit=False`.
   - `epic_from_plan.py` makes a second local commit
     (`Link approved epic plan to its bead: <stem>`) that writes `bead_id:` into the
     archived copy.
   - The only post-launch publication, `push_store_after_launch`, calls
     `sdd_commit_targets(store, paths=[store.kind_root("beads")])`. With split sidecar
     storage, `sdd_commit_targets` partitions paths per repo and **drops targets with no
     matching paths** — so only the beads repo is pushed. The two plans-repo commits
     stay local to the workspace's `sase/repos/plans` clone forever.

3. **Workspace sidecar clones are ephemeral, and only bead stores are protected.**
   Numbered-workspace `sase/repos/<role>` clones are routinely destroyed and re-cloned
   (the clones in active workspaces on this machine were all re-created within hours of
   each other on 2026-09-09). Workspace preparation (`src/sase/axe/runner_workspace.py`)
   calls `_protect_unpushed_sidecar_bead_commits` to publish-or-rescue unpushed **bead**
   commits before destructive steps — its docstring even says "the caller is about to
   destroy the clone that holds the only copy" — but there is **no equivalent protection
   for the plans sidecar** (or any other role). The clone holding the only copy of this
   plan's archive commit was destroyed the same day, a few hours after the launch.

4. **Survival today is luck.** Sidecar history proves it: commits `27ed360f`
   (`Archive approved plan double_star_model_completion`) and `b86ab701`
   (`Link approved epic plan to its bead: double_star_model_completion`) have author
   date 2026-09-09 10:57 but committer date 18:04 — they sat unpushed for seven hours
   and only reached the remote because a later, unrelated plans-repo push
   (`7acba3d0 chore: Add links`) from the same clone replayed and carried them. When no
   such push happens before the clone is re-provisioned, the archive is silently lost.
   Nothing raises, so no `plan-archive` failure notification fires.

A second, smaller loss class exists on the tale path: nine `plan-archive` failure
notifications (for example `6c9f6596-9b42-4af2-b438-f18ca951172f` for
`wait_checks_perf.md` and `012386f3-68e6-4c1c-a13d-e9c19cf20216` for
`hidden_clone_machine_writes.md`) show approval-time archives failing with
`OperationalLeaseError`/`ResetReplayError`. Those failures are already surfaced, and
their plans remain recoverable from the canonical directory; the backfill phase covers
them. Fixing the underlying lease failures (stale `sase_core_rs` wheel, leftover
`index.lock`) is out of scope here.

### Collateral damage worth naming

The `Link approved epic plan to its bead` commit died with the clone, so the canonical
`~/.sase/plans/202609/unified_agents_across_machines.md` has **no `bead_id:`
frontmatter** even though epic `sase-xe.16.11.7` is running. A future `sase bead work`
against that file would not find a linked epic and could create a duplicate epic. The
repair must restore `bead_id` when re-archiving.

## Phase design

### launch-plan-push (no dependencies)

Make the epic-launch path publish its plans-repo commits.

- In `src/sase/bead/cli_work_from_plan_store.py` /
  `src/sase/bead/cli_work_from_plan.py`: after the archive and bead-link commits
  succeed, publish the plans sidecar repo. Prefer extending `push_store_after_launch` so
  its `sdd_commit_targets` call includes the plans root (for example
  `paths=[beads_root, plans_root]`, or pushing every configured target when the launch
  made commits in it). Respect `no_push`.
- Publication failure must not be silent: the launch already treats bead-graph
  publication as fatal (`publish_epic_graph_before_launch_result`); a plans push failure
  may stay non-fatal for the launch, but must then produce a notification of the same
  shape as `report_plan_archive_failure` in `src/sase/_plan_archive_approval.py` (the
  "plan of record exists only on this machine" warning) instead of a swallowed log line.
- Cover `epic_from_plan.py`'s bead-link commit too: both commits must be on the remote
  after a successful launch.
- Tests: a launch-from-plan-file test against a split sidecar store asserting the plans
  remote contains both the archive commit and the bead-link commit; a test that a
  failing plans push produces the notification and does not abort an otherwise
  successful launch; `no_push` behavior unchanged.

### prep-clone-protection (no dependencies)

Close the destruction window for every sidecar role.

- First, identify the actual code path that deletes and re-creates numbered-workspace
  `sase/repos/<role>` clones between agent claims (observed empirically; the plans clone
  that held this plan's archive commit was replaced by a fresh `git clone` from the
  remote). Candidate paths to audit: `src/sase/axe/runner_workspace.py` preparation,
  workspace claim/refresh in the workspace provider, and sidecar materialization in
  `src/sase/sdd/_store_link.py` / `src/sase/sdd/_sidecar_init.py` /
  `src/sase/_linked_repo_workspaces.py`. Document the found path in the phase bead
  notes.
- Generalize `_protect_unpushed_sidecar_bead_commits` (or add a sibling) so that before
  any step that deletes or re-creates a sidecar clone, every `sase/repos/<role>` git
  repo is checked for commits ahead of its upstream: publish them; if publication fails,
  rescue them (bundle or rescue ref under `~/.sase/rescue/`, mirroring the existing
  bead-store rescue behavior) and notify — never destroy the only copy silently.
- The check must be cheap for the common all-published case (one
  `rev-list upstream..HEAD --count` per role).
- Tests: preparation over a plans clone with an unpushed commit publishes it (or rescues
  it when the remote rejects) before the clone is replaced.

### plan-archive-doctor (no dependencies)

Detection and confirmed repair for plans that should be in the sidecar but are not.

- Extend the existing doctor surface (`sase bead doctor` already covers bead-store,
  plan-link, and artifact-reference health — keep the new check consistent with that UX;
  a `sase plan`-side home is acceptable if cleaner) with a check that reports, for the
  current project:
  1. plan beads whose linked plan file is missing from the plans sidecar;
  2. orphaned `links/<month>/<name>.md.json` entries whose plan markdown is missing;
  3. canonical-plans-directory files missing from the sidecar, flagged separately since
     unapproved drafts may legitimately be local-only — bead-linked plans are the
     authoritative "must archive" set.
- Add a confirmed `--fix`-style repair that, for each recoverable missing plan:
  re-archives the canonical copy through the existing archive machinery
  (`archive_plan_file` + the epic bead-link/header projection helpers from
  `epic_from_plan.py` / `sase.sdd.plan_header_writes`, not a raw `git add`), restoring
  `bead_id:` frontmatter when the owning bead is identifiable (bead plan link or link
  index), commits with a clear backfill message, and pushes synchronously with
  verification, following the reset-and-replay pattern in `sase._plan_archive_approval`.
- Plans whose content exists nowhere on this machine (not canonical, not in an
  interaction-request bundle snapshot) are reported as unrecoverable-here, naming the
  likely source machine when the bead's creator implies one.
- New CLI subcommands/options require reading the `cli_rules.md` reference memory first;
  the worker must do so.
- Tests: fixture store with a bead-linked plan missing from the sidecar → detected and
  repaired with bead_id restored and remote verified; orphaned link index detected;
  drafts not falsely flagged as must-archive.

### backfill-athena (depends on launch-plan-push, plan-archive-doctor)

Run the repair on this machine and prove the original loss is healed.

- Run the doctor check, then the confirmed repair, for the current project.
- `202609/unified_agents_across_machines.md` is the acceptance case: after repair, the
  plans sidecar remote must contain the plan markdown with `bead_id: sase-xe.16.11.7`,
  and `sase bead show sase-xe.16.11.7` must resolve its epic plan against the store. The
  canonical copy must also carry the restored `bead_id` so a future `sase bead work`
  cannot create a duplicate epic.
- Sweep the remaining findings: the 57 orphaned link indexes and the notification-listed
  tale-path failures (`wait_checks_perf.md`, `hidden_clone_machine_writes.md`,
  `agents_all_panel_fold_sweep.md`, `gate_shell_finalizer_claim_leak.md`,
  `axe_systemd_scope_escape.md`, `machine_cli_enrollment.md`, `phantom_running_proc.md`,
  `recover_artifacts_conformance_phase.md`, `ctrl_space_stale_prompt_context.md`).
  Repair every plan recoverable on this machine; for the rest, record a
  `PROPOSED FOLLOW-UP:` note on this phase's bead naming the plan and its likely source
  machine so the owner can run the doctor there.
- Report before/after counts (missing bead-linked plans, orphaned link indexes) in the
  phase bead notes.

## Out of scope

- Root-cause fixes for the tale-path `OperationalLeaseError` failures (stale
  `sase_core_rs` wheel repair, leftover `index.lock` cleanup) — already surfaced by
  notifications and separately trackable.
- Backfilling plans whose only copy lives on another machine (follow-ups recorded
  instead).
- Any change to Rust core: this is host-side git/workspace orchestration in the Python
  runner, not frontend-shared domain behavior.

## Risks

- **Duplicate archives**: the repair must go through the existing archive/header
  machinery and `preserve_existing` semantics so a plan already in the sidecar is never
  overwritten with a stale canonical copy; where both exist and differ, the sidecar copy
  wins and the difference is reported, not clobbered.
- **Push contention**: synchronous verified pushes must reuse the existing sync-worker
  lock waits and reset-and-replay bounds rather than raw `git push`.
- **Prep-time cost**: the generalized protection must stay O(one rev-list per role) when
  everything is already published.
