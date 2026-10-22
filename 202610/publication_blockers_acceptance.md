---
tier: epic
title: Fix the confirmed publication blockers and finish recovery acceptance
goal: Remove the three confirmed publication blockers, install the fixes on both hosts,
  recover Apollo's unpublished bob-cli content, and prove all five recovery checks
  with bounded canaries and a natural automatic publication.
parent_bead: sase-1nf.10.1.9.1
decisions:
  canary_budget:
    ask: After install, should each host/project get a fresh limit-1 canary plus one
      batch of at most 25?
    choices:
      fresh: Every host/project restarts at limit 1, then may run one batch of at
        most 25
      remaining: Only unspent budget remains; athena and bob-cli go straight to one
        batch of 25
    default: fresh
    why: Earlier canaries ran on defective code, and a limit-1 probe is the safer
      first step
    answer: fresh
phases:
- id: drain-bound
  title: Remove the quadratic name scan and the cached-snapshot digest regression
  depends_on: []
  description: 'drain-bound: compute reserved agent-session names once per registry
    snapshot during inventory builds, digest reused validation snapshots by their
    on-disk bytes, add regressions, and measure scoped drains on athena.'
  size: medium
- id: replay-policy
  title: Make stale-replay recovery binary-safe and merge-base aware
  depends_on: []
  description: 'replay-policy: read upstream blobs as bytes in the stale-replay helper,
    add an optional merge-base keep-one-side rule to the sase-core replay byte policy,
    wire it through report and apply, add fixtures, and bump the core pin.'
  size: medium
- id: install-replay
  title: Install the fixes on both hosts and recover Apollo bob-cli content
  depends_on:
  - drain-bound
  - replay-policy
  description: 'install-replay: confirm both fixes landed, gate sase update on athena
    and apollo, then run the backup-first replay report, apply, and repeat apply on
    Apollo bob-cli with content and remote proof.'
  size: medium
- id: canaries
  title: Prove bounded publication for each repaired class on both hosts
  depends_on:
  - install-replay
  description: 'canaries: run the bounded athena timeout, apollo bob-cli rebase-conflict,
    and apollo sase manifest-mismatch canaries on the installed fixes with full evidence
    and remote validation.'
  size: medium
- id: acceptance
  title: Prove fixtures and automatic publication, and publish the acceptance report
  depends_on:
  - canaries
  description: 'acceptance: rerun the real-Git recovery fixtures, correlate a natural
    post-install automatic publication, and publish the five-check research report
    attached to sase-1nf.10.1.9.'
  size: medium
proposed_by: bbugyi200.athena.sase-1nf.10.1.9.1.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-21 10:45:29
status: done
bead_id: sase-1nf.10.1.9.1.6
---

- **PROMPT:** [prompts/202610/publication_blockers_acceptance.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/publication_blockers_acceptance.md)
- **PARENT:** [202610/publication_recovery_proof.md](https://github.com/sase-org/sase--plans/blob/main/202610/publication_recovery_proof.md)
- **BEAD:** [sase-1nf.10.1.9.1.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1nf/sase-1nf.10.1.9.1.6.md)

# Fix the confirmed publication blockers and finish recovery acceptance

This child epic carries the remaining work of `sase-1nf.10.1.9.1` ("Finish Apollo
recovery and prove bounded publication"). That epic covers the `recovery-proof` phase
`sase-1nf.10.1.9` of `plan:202610/publication_drain_fixes.md`. All five of its phases
closed, but none of its five acceptance checks has passing production evidence. Its land
agent diagnosed three code defects that block acceptance. Each is confirmed with
read-only evidence attached to `sase-1nf.10.1.9.1`:

- Landing audit report: `file:explicit:684de02c487788fab67ad837`.
- Apollo snapshot digest probe: `file:explicit:b2ebf67eabeaef7b5e0a1265`.
- cProfile of a scoped inventory build: `file:explicit:07eac2cef30caada4c976c16`.
- Profile JSON: `file:explicit:1e757f4f4eadfca380da107c`.
- Athena drain-timing log tail: `file:explicit:f3576bf6104be8530cc534e2`.

Read the parent epic `sase-1nf.10.1.9.1`, its five phases and their notes, and
`plan:202610/publication_recovery_proof.md` with audited commands. Do not repeat
completed investigation. The ancestor decision `gc_soak = no` remains final. No memory
edits are required or authorized.

## Confirmed blockers

1. **Cached-snapshot digest regression.** Commit `8c6699f413` (sase-1nf.10.1.5) added
   `base_snapshots` reuse to `load_validated_publication` in
   `src/sase/agents_sync/publication_validation.py`. A reused snapshot is digested as
   `v2_json_bytes(snapshot.to_json_dict())` instead of its on-disk bytes. At agents
   revision `b3da77e44a`, 40 of 98 legacy Apollo sase snapshots have raw bytes that
   match their manifest digest but a canonical re-serialization that does not (the first
   is `03`). The second hood planned in one transaction is served from the cache that
   `publication_planning.py` populates, so every multi-hood Athena drain fails with
   `snapshot digest mismatch for 'bbugyi200.apollo.03'` (sase) or
   `'bbugyi200.apollo.0p'` (bob-cli). Apollo's uncached validation passes. The manifest
   is valid and the validator is wrong. Because of this regression, hood `sase-1nf`, and
   with it every worker of the parent epic, never published.
2. **Quadratic reserved-session-name scan.** Athena's scoped `inventory_build` stage
   takes 106–148 s, which alone exceeds the 120 s drain bound. During a
   `--hood sase-1nf` profile the call chain `sase_agent.sase_agent_ref_for_name` →
   `_is_reserved_agent_session_name` → `get_reserved_agent_session_names_for_display` →
   `_registry_queries.get_reserved_agent_session_names` ran 8,689 times. Each call
   rebuilt the full reserved set from the whole name registry, 221.5M
   `is_agent_session_container_kind` calls in total. That accounts for about 229 s of
   268 s under the profiler. Callers on the path are
   `content_layout.discover_project_root` and
   `inventory_history.historical_associations`. Git calls numbered only 25.
3. **Stale-replay helper defects** (sase-1nf.10.1.9.1.1, evidence
   `file:explicit:05af6c55b55e24008cbd9f2f`):
   - `_git_optional_bytes` in
     `src/sase/agents_sync/_publication_replay_recovery_backup.py` re-encodes text-mode
     Git output, so bare CR bytes become LF. This produced two false refusals on
     byte-identical chats.
   - The prefix-only `decide_replay_byte_divergence` policy in sase-core
     (`crates/sase_core/src/agent_publication_recovery/replay.rs`) cannot carry
     one-sided terminal transitions, where upstream is unchanged since the merge-base.
     Examples: `state` moving active to completed, and extended `prompt_steps`. Apply
     therefore refused, with the backup intact.

   sase-1nf.10.1.9.1.4 reproduced the stale `2abeff3f9` rebase conflict. That class
   cannot pass until the replay lands.

## Shared execution and evidence contract

Follow the "Shared execution and evidence contract" of
`plan:202610/publication_recovery_proof.md` (items 1–9) unless this section changes it.
That covers repo access through `/sase_repo`, the Apollo login-shell invocation,
existing helper/CLI APIs only, `/sase_monitor` for long commands, per-mutation evidence
fields, remote validation at a fetched revision through `load_validated_publication`,
stop-on-regression, and `PROPOSED FOLLOW-UP:` notes. The changes:

- Source changes are expected in `drain-bound` and `replay-policy`. Shared policy
  belongs in sase-core, with its binding and the `sase-core-revision.txt` pin updated
  per `docs/rust_backend.md`. Read `lint_and_test` and the domain memories you need, and
  run `sase tool run check` in every changed repo. Never run `check-full`.
- Canary budget: see the decision below. The budget belongs to this whole epic, not to
  each continuation. Never retry all, drop requests, increase the 120 s bound, or revive
  a retired class whose cause is unresolved.
- Every phase saves its raw evidence as explicit artifacts attached to its own phase
  bead.
- Before closing a phase, run `sase bead epic-symbols` for this epic and for the phase.
  Resolve or re-key any remaining exemption.

> [!decision] canary_budget = fresh Each host/project (athena/sase, apollo/bob-cli,
> apollo/sase) gets one limit-1 call and at most one later batch of at most 25, all
> spent in `canaries`.

> [!decision] canary_budget = remaining athena/sase and apollo/bob-cli get only their
> single unspent batch of at most 25, with no new limit-1 probe. apollo/sase keeps its
> unspent limit-1 call and one batch.

## Phase drain-bound

Fix blockers 1 and 2 in the sase repo. No production mutation.

1. **Reserved-name scan.** Make repeated `_is_reserved_agent_session_name` lookups reuse
   one reserved-name set per loaded registry snapshot instead of rebuilding it per call.
   For example, memoize in the display-tier facade keyed on the registry snapshot or
   file signature, or within `name_registry_load_session`. Preserve the existing
   semantics: best-effort, display tier, never forcing a rebuild or taking the
   allocation lock. A registry change must still be observed. Check whether sibling
   per-call registry queries on the same inventory path have the same shape, and fix
   them only if they show up in the profile. Add a regression test showing that repeated
   lookups against one snapshot build the set once, and that a changed registry is seen.
2. **Digest of reused snapshots.** Cached unselected snapshots must be verified against
   the digest of their on-disk bytes. For example, cache the raw digest recorded when
   the snapshot was first read and validated, and compare that with `entry.digest`.
   Freshly built override snapshots keep their canonical serialization. Do not weaken
   validation: a cached snapshot whose recorded digest does not match the manifest entry
   must still fail, and so must identity and explicit file-set checks. Add regressions
   in `tests/agents_sync/test_publication_targeted.py`:
   - A foreign-owner snapshot stored with non-canonical bytes, with two requested hoods
     planned in one transaction. This fails before the fix and passes after.
   - A tampered cached snapshot still fails.
3. **Measure, read-only, under `/sase_monitor`.** On athena run
   `tools/agents_publication_profile --project sase --hood sase-1nf --json` and a
   two-hood run (`--hood sase-1nf --hood 12r`) against the hidden sase agents clone.
   Compare with the landing-audit baseline (inventory ≈110 s in production drain rows,
   268.5 s under cProfile). Record inventory and planning seconds, Git counts, and
   whether any digest mismatch occurs. Target: a 1–3 hood drain fits within the 120 s
   bound with clear margin. If inventory still dominates after the fix, profile once
   more. Fix the next bounded hotspot, or record the measured remainder precisely.
4. Run `sase tool run check`. Attach the before/after profile evidence to the phase.

## Phase replay-policy

Fix blocker 3. No production mutation.

1. **Binary-safe reads.** Read upstream and merge-base blobs as bytes, so CR bytes
   survive. Extend the existing `GitRunner` path rather than hand-rolling subprocess
   calls. Audit every helper read of Git blob content, in report and apply alike, for
   the same text-mode re-encoding.
2. **Merge-base rule in sase-core.** Extend `ReplayByteDivergenceRequestWire` with an
   optional merge-base payload. When it is absent, today's behavior is unchanged. Rules:
   - previous == base: keep current. Upstream is untouched since the merge-base, and the
     local change is retained.
   - current == base: keep previous.
   - Otherwise apply the existing identical/empty/prefix rules, then refuse.

   Add Rust unit tests, update the binding, add an optional `base` argument to the
   Python facade `sase.core.agent_publication_recovery.decide_replay_byte_divergence`,
   and bump the pin.

3. **Wire through report and apply.** In `refused_same_run_divergences` and the apply
   path's `_choose_payload_bytes`, plus any state or metadata merge, pass merge-base
   bytes when the path exists at the merge-base. Report and apply must reach the same
   verdict for every path.
4. **Fixtures.** Extend `tests/agents_sync/test_publication_replay_recovery.py` without
   duplicating existing tests:
   - Identical CR-containing chats are not refused.
   - A one-sided terminal transition (`state` active to completed, extended
     `prompt_steps`, upstream == merge-base) applies and publishes the local bytes.
   - A two-sided non-prefix divergence still refuses.
   - The existing HEAD-drift and same-run-divergence refusal coverage is preserved.
5. Update the policy description in
   `docs/contribute/agents-sidecar-sync.md#stale-replay-recovery`, which currently
   documents prefix-only refusal. Run `sase tool run check` in sase-core and sase.

## Phase install-replay

1. Fetch and confirm that both code phases' commits are on sase `master` and that the
   core commit is on the sase-core default branch. Record full SHAs.
2. Global install is human-only. Use `/sase_gate` to propose `sase update` on athena
   and, through the documented login shell, on Apollo. The gate's follow-up continues
   this phase. After the update, record `sase version --json` on both hosts: clean SASE
   and core revisions that contain both fixes. Do not mutate production on stale code.
3. On Apollo, open the bob-cli agents target with `sase repo open` and confirm the
   hidden machine clone. Capture full HEAD, upstream, merge-base, and worktree state. If
   HEAD is no longer `2abeff3f9`, validate the earlier recovery instead of resetting a
   healthy clone.
4. Run `tools/agents_publication_recover_replay` without `--apply`, using the observed
   full HEAD.
   - Prefer the preserved backup
     `/home/bryan/.sase/recovery-backups/bob-cli-replay-sase-1nf.10.1.9.1.1/` if the
     helper accepts it.
   - Otherwise use a new durable phase-specific directory under
     `~/.sase/recovery-backups/`.
   - Never edit or delete the old backup.

   Require `stage=verified`, verified checksums and bundle, and empty `unknown_paths`
   and `refused_divergences`. Apply with the identical backup and expected HEAD. Then
   repeat the apply and require `already_complete` with no new publication.

5. Prove that the unique content inventoried in `file:explicit:05af6c55b55e24008cbd9f2f`
   is published with identical bytes at a fetched remote revision:
   - Runs `bbugyi200.apollo.bob-cli-7c.2--2` and `bbugyi200.apollo.bob-cli-7c.2--mon-0`.
   - The `2--1` and `6` chat additions.
   - The completed state of `2--1` and `6`, and the extended `6` prompt steps.
   - The three archive objects.
   - The byte-identical `1--mon` and `2--mon` chats.

   Also prove that upstream-only content survives and that `load_validated_publication`
   passes at that revision.

6. Read-only, on athena after the install, re-run the two-hood targeted profile. Confirm
   no `bbugyi200.apollo.03` mismatch. The Apollo sase manifest needs no repair
   (sase-1nf.10.1.9.1.2). Run `--repair-manifest` only if fresh evidence shows a real
   on-disk mismatch.

## Phase canaries

1. Refresh doctor summaries and complete lock-free queue snapshots on both hosts. Record
   whether automatic drains since the install have already cleared active rows. Use the
   Rust selector, without mutation, to identify the rows that the first call will select
   and at least one stopped historical row of each class.
2. Run the bounded canaries in this order, within `canary_budget`. Allow at most one
   follow-up batch per host/project, and only when the limit-1 call succeeded on an
   active row without reaching a stopped row of the class:
   - `sase agent sync --project sase --retry-quarantined --retry-limit 1 --retry-error-class timeout --json`
     on athena.
   - `sase agent sync --project bob-cli --retry-quarantined --retry-limit 1 --retry-error-class rebase_conflict --json`
     on Apollo.
   - `sase agent sync --project sase --retry-quarantined --retry-retired --retry-limit 1 --retry-error-class manifest_mismatch --json`
     on Apollo.
3. For each host/project, prove that a named, previously failing, stopped request of its
   class published within the 120 s drain bound. The proof includes push, a fetched
   remote SHA, a checked tree matching that SHA, `load_validated_publication`, the
   requested revision/page, and the prompt obligation. Preserve stage timings, Git
   counters, payload files/bytes, and selected keys. Compare unselected rows before and
   after, and explain concurrent additions. Confirm that bob-cli unique content still
   holds after drain cleanup.
4. Stop on regression and keep durable obligations. Record final per-host/project counts
   and exact bounded follow-up commands for `sase-1nf.11`. Do not bulk drain.

## Phase acceptance

1. Verify that the workspace binding matches its pin, preparing under a monitor if a
   build is needed. Run the focused
   `tests/agents_sync/test_publication_replay_recovery.py` through the normal test
   recipe. Capture passes for:
   - `test_report_only_writes_a_verified_backup_without_resetting`
   - `test_apply_keeps_both_sides_unique_content_and_pending_objects`
   - `test_interrupted_apply_resumes_and_second_apply_is_already_complete`
   - the new CR and merge-base fixtures
   - HEAD-drift and divergence refusal
2. Identify an agent that completed after the verified install and published through its
   automatic host-completion path. Earlier workers of this epic, whose hood is
   `sase-1nf`, are natural candidates. Correlate:
   - the completion timestamp
   - the requested primary revision
   - the drain timing row and acknowledgment
   - the remote run, page, and prompt at a fetched remote SHA

   A manual canary publication or an empty outbox is insufficient. Only if no qualifying
   completion exists, use `/sase_run` for one minimal publication-eligible canary agent
   with a harmless deliverable, and wait through a monitor.

3. Open the research sidecar through `/sase_repo` and author the acceptance report. It
   includes the five-check table:
   1. A named formerly failing request publishes within 120 s.
   2. Remote manifests and referenced files agree.
   3. Repeat recovery is a no-op, with fixture and production evidence.
   4. An interrupted real-Git publication resumes without content loss.
   5. A newly completed agent publishes automatically after installation.

   Link the immutable raw evidence from `sase-1nf.10.1.9.1.*` and this epic's phases.
   Include:
   - the scoped/full measurement comparison: 2026-10-20 full baseline 209.84 s / 9,903
     Git calls, the 2026-10-21 targeted `research` profile 129.956 s, the landing-audit
     cProfile, and the post-fix measurements
   - backup locations and checksums
   - remote SHAs and representative paths
   - per-host/project remaining backlog with bounded commands for `sase-1nf.11`

   Carry forward, without treating them as passing, the known issues already recorded
   for the `sase-1nf.10.1` landing:
   - Apollo `job`/`tailnet` "Repository not found" fetch failures.
   - The tagged-commit "prompt archive is unresolved" failure, including its production
     occurrences in Athena drain rows.

   Publish the report through host-owned finalization and attach its canonical reference
   to `sase-1nf.10.1.9` and to this phase. Missing proof must be labeled as missing,
   never as passed.

## Landing handoff

This epic's land agent closes this epic normally. It then resumes the interrupted
landing of its parent plan, `sase-1nf.10.1.9.1`:

1. Recheck that plan's phases, its `LANDING AUDIT` note, and this epic's evidence.
2. Close `sase-1nf.10.1.9.1`, then set `status: done` in
   `plan:202610/publication_recovery_proof.md`.
3. Carry out that plan's delegated enclosing-phase closure:
   - Confirm the research report is attached to `sase-1nf.10.1.9`.
   - Run `sase bead epic-symbols sase-1nf.10.1.9` and resolve or re-key every entry.
   - Close only `sase-1nf.10.1.9`, with a note citing the verified acceptance and the
     research reference.

Never close `sase-1nf.10.1`, `sase-1nf.10`, `sase-1nf`, `sase-1nf.11`, or `sase-10x`.
