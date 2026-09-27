---
tier: epic
title: Land the fleet claims-cache fix and reconverge Apollo and Athena builds
goal:
  The sase-core claims-cache fix that the receipt live proof deployed only to Apollo is
  committed on sase-core master, both hosts run the same sase and sase-core build
  containing it, and fresh remote rows stay visible from Athena while they run.
parent_bead: sase-1aq.10.7.5.7
phases:
  - id: land_claims_fix
    title: Commit the host_liveness claims-cache recheck fix to sase-core
    depends_on: []
    size: small
    description:
      "land_claims_fix: apply the audited host_liveness.rs claims-cache recheck diff in
      the linked sase-core checkout with its regression test, run the focused cargo
      suites plus sase-core lint, and land it on sase-core master."
  - id: reconverge_deploy
    title: Reconverge both hosts on the fixed build and recheck fresh-row visibility
    depends_on:
      - land_claims_fix
    size: medium
    description:
      "reconverge_deploy: reconcile the dirty Apollo primary sase-core checkout against
      the landed fix, run the supported sase update on Apollo and Athena, restart the
      gateway stack, confirm matched builds and health, and prove from Athena that a
      fresh running Apollo dispatch is served while it runs."
proposed_by: bbugyi200.apollo.sase-1aq.10.7.5.7.land
create_time: 2026-09-27 05:04:13
status: wip
---

- **PROMPT:**
  [prompts/202609/1aq_land_claims_cache_fix_and_reconverge.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/1aq_land_claims_cache_fix_and_reconverge.md)
- **PARENT:**
  [202609/1aq_receipts_matched_live_proof.md](https://github.com/sase-org/sase--plans/blob/main/202609/1aq_receipts_matched_live_proof.md)

# Land the fleet claims-cache fix and reconverge Apollo and Athena builds

## Why this plan exists

Epic `sase-1aq.10.7.5.7` (plan `plan:202609/1aq_receipts_matched_live_proof.md`) has the
goal "Apollo and Athena run the same sase and sase-core build" containing the
`exact_ops_receipts` contract, and a live proof of that contract. All three phases
closed, and the proof passed. But its land audit found one piece of unfinished work that
the epic itself caused:

- Phase `receipt_live_proof` (`sase-1aq.10.7.5.7.3`) found that fresh RUNNING dispatch
  rows were invisible to Athena for their whole run. The root cause is in sase-core
  `crates/sase_core/src/host_liveness.rs`. `FilesystemRecordIdentityProbe` caches each
  project file's workspace claims the first time it reads them and keeps them for the
  whole gateway process lifetime. Workspace claims change on every run.
  `match_project_claims` returns `Mismatch` when no cached claim names the live PID of a
  record that has a `workspace_num`, and `decide_fleet_presentation` excludes
  mismatches. So every agent that claimed its workspace after the cache fill disappears
  from the served fleet until it dies or the gateway restarts.
- The phase fixed it with a re-read before excluding: on a cached `Mismatch`, re-read
  the project file once and re-match, and keep `Mismatch` only if the fresh read still
  disagrees (fail closed). It added the regression
  `filesystem_probe_rechecks_stale_claims_before_mismatch`. But it left the fix
  **uncommitted** in Apollo's primary sase-core checkout
  (`~/projects/github/sase-org/sase-core`, on top of `b57cd21`). It rebuilt only
  Apollo's `sase_core_rs` extension (`maturin develop --release`) and restarted only
  Apollo's gateway. Athena was not rebuilt.
- As a result, the two hosts no longer run the same build, which breaks the epic's goal.
  Also, Apollo's dirty primary sase-core checkout will make the next `sase update` skip
  the sase-core fast-forward.

The exact diff (one file, +79/-9, fix plus regression test) is saved as the audited
artifact `file:explicit:b02a5484229526d3fdf7cb20`. Read it with
`sase artifact read file:explicit:b02a5484229526d3fdf7cb20 "<reason>"`. The phase's live
evidence is `file:explicit:1b071b131efc527dce16b7a1` (§0 describes the fix).

This plan does only that remaining work. It does not close `sase-1aq.10.7.5.7` or any
ancestor. Their land agents resume through `parent_bead` after this plan lands.

## Operating rules

- **Hosts.** Apollo owns the agents. Athena is the dispatch controller and viewer. Reach
  Athena with `ssh athena` (Tailscale SSH). Read the `tailnet.md` and `dispatch.md`
  reference memory with `/sase_memory_read`, and read `docs/remote_dispatch.md`, before
  the live phase.
- **Code placement.** Open sase-core with `sase repo open sase-core -r "<why>"`, and
  work only in the printed path. Read that repo's `AGENTS.md` first. This fix changes no
  binding or wire schema, so no `sase-core-revision.txt` pin move is required.
- **Verification.** Do not run `just check-full`. Known clean-base reds in sase (4 mypy
  errors owned by `sase-19i.7.3.3.3.3` and `sase-1ab`, and 13 symvision unused-public
  symbols owned by `sase-1ay`) are not this plan's work.
- **Safety.** Keep enrollment intact: `sase machine list` on Athena must still show
  `apollo` pinned. Leave unrelated work alone. Never replay an uncertain operation under
  a new key. Never close or reopen beads with `--force`. Stop or dismiss every test
  agent you create.

## Phase land_claims_fix

1. Open sase-core with `sase repo open sase-core`. Confirm its `origin/master` contains
   `b57cd21` and does not already contain an equivalent claims recheck (check
   `crates/sase_core/src/host_liveness.rs` for `refresh_claims`). If an equivalent fix
   has already landed, record that on this phase bead and stop.
2. Apply the diff from `file:explicit:b02a5484229526d3fdf7cb20` to
   `crates/sase_core/src/host_liveness.rs`. `git apply` should work cleanly. If it does
   not, re-create the same change by hand:
   - In `impl RecordIdentityProbe for FilesystemRecordIdentityProbe`, on a cached
     `Mismatch`, call `refresh_claims(record)` and re-match. If the refresh fails,
     return `Mismatch`.
   - Add `refresh_claims`, which re-reads the file and replaces the cache entry, and a
     shared `read_claims` helper.
   - Add the regression test `filesystem_probe_rechecks_stale_claims_before_mismatch`.
     It must fail without the fix.
3. Verify with `cargo test -p sase_core host_liveness` (13 tests),
   `cargo test -p sase_core fleet_presentation` (18 tests) and the `sase_gateway`
   `fleet_catalog_overlay` test. Also run the sase-core lint, fmt and clippy gates that
   its `AGENTS.md` names for a crate change.
4. Write a commit message in the style `fix(fleet): ...` that states the stale-claims
   exclusion root cause. Record a note on this phase bead with the test results. The
   host finalizer commits and lands the change on sase-core master. Do not touch hosts
   or deploy in this phase.

## Phase reconverge_deploy

Work on Apollo first, then on Athena over `ssh athena`.

1. **Record state.** On each host, record `sase version`, `sase update -n`, the running
   gateway, federation worker and scheduler processes with their start times, and
   `git status --short` for both primary checkouts (`~/projects/github/sase-org/sase`
   and `~/projects/github/sase-org/sase-core`).
2. **Reconcile Apollo's dirty sase-core checkout.** Run `git fetch`. Confirm that every
   hunk of the working-tree diff in `crates/sase_core/src/host_liveness.rs` is identical
   to what `land_claims_fix` landed on `origin/master`: after the fast-forward, the
   worktree file should match `origin/master` byte for byte. Then save the diff as a
   timestamped backup outside the checkout, register it with `sase artifact create`, and
   discard it with `git stash push -m "pre-reconverge <UTC>"` so it stays recoverable.
   If any hunk is not on master, stop and record it on this phase bead instead. If
   Athena's primary checkouts are dirty, check them the same way.
3. **Check running agents.** Run `sase agent list` on each host. Do not update while
   another agent on that host is committing or landing. WAITING or parked agents are
   fine.
4. **Update.** Run the supported `sase update` (dev mode, as configured) on both hosts.
   Confirm that both report the same sase commit and the same sase-core-rs build, which
   must contain the `land_claims_fix` commit, and that `sase update -n` shows nothing
   pending. Check that Apollo's `sase_core_rs` extension was rebuilt from the committed
   tree, not left over from the hand-built `.so`. If the update does not rebuild it, use
   the documented supported path.
5. **Restart.** Restart the gateway and federation worker on both hosts through the
   service host or the documented supported path. Restart schedulers where the update
   requires it. Record each gateway's version, protocol and fleet schema, and the new
   process start times.
6. **Health.** On Athena, run `sase machine list` (apollo still pinned),
   `sase machine status apollo` (ok, no version skew) and `sase doctor` (dispatch config
   OK). On both hosts, run `sase agent index status` and confirm it is clean.
   Pre-existing unrelated doctor errors (for example `project.artifact_links_aggregate`,
   tracked by `sase-ua`) are not blockers. Record them as they are.
7. **Live recheck of the fix on the reconverged build.** From Athena, dispatch a fresh
   long-running Apollo agent (`sase run %dispatch:apollo` with a sleep-free long task,
   or the same long `sleep` probe the previous phase used). Confirm the fix:
   - Athena serves it as RUNNING while it runs, within about the index plus
     snapshot-rebuild latency (about 70 s), not only after it is DONE.
   - It stays served as RUNNING across at least two later snapshot rebuilds, without a
     gateway restart.
   - An exact stop by full dispatch key settles `applied`.

   Capture the receipts and `sase machine agent` output as an audited artifact with UTC
   timestamps. Then stop or dismiss every test agent, and leave both hosts' Agents panes
   clean.

8. Record a note on this phase bead with the matched versions, identities, the backup
   artifact ref, health results and live-recheck evidence. Add a short note on
   `sase-1aq.10.7.5.7.3` saying that its in-phase fix has landed and been deployed on
   both hosts.
