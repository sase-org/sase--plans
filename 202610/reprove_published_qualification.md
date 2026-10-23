---
tier: epic
title: Integrate the new core binding and retain deployed qualification proof
goal:
  A containing published core is pinned on master and retained aligned runs prove
  passing published qualification, a typed wrong-core refusal, and passing source
  blocking checks.
parent_bead: sase-1n6.5.1.5.2.1
phases:
  - id: published-pin
    title: Integrate a complete published core containing the current source pin
    depends_on: []
    description:
      "published-pin: wait for the authorized daily core release, integrate its exact
      dependency pin, and verify all current required bindings before deployed proof."
    size: medium
  - id: deployed-reproof
    title: Retain aligned green, incompatible-core, and source proofs
    depends_on:
      - published-pin
    description:
      "deployed-reproof: exercise the host-landed pin and observation repair, retain
      actual workflow observations, and prove the exact-SHA source comparison with
      bounded retries."
    size: medium
proposed_by: bbugyi200.athena.sase-1n6.5.1.5.2.1.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-23 18:27:25
status: wip
---

- **PROMPT:**
  [prompts/202610/reprove_published_qualification.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202610/reprove_published_qualification.md)
- **PARENT:**
  [202610/deployed_qualification_proof.md](https://github.com/sase-org/sase--plans/blob/main/202610/deployed_qualification_proof.md)

# Finish the deployed qualification proof

This is only the remaining work from landing `sase-1n6.5.1.5.2.1`. Its two children are
closed, but its live-proof child closed under the explicit repair-handoff branch, not
because the proof passed. The parent epic and its parent phase `sase-1n6.5.1.5.2` remain
open.

A tale was considered. It cannot finish this work directly: master now needs a new
published dependency pin, and the live workflow must run that host-committed pin. The
first phase ends with the pin ready for host finalization; the second starts after it is
on remote master. No worker waits for its own end-of-turn commit. The existing TUI
repair owners remain separate; do not duplicate their implementations.

Read the following audited context before work:

```sh
sase bead read sase-1n6.5.1.5.2.1 -r "Need landing audit and all follow-up dispositions"
sase bead read sase-1n6.5.1.5.2.1.2 -r "Need repair handoff and retained failed live evidence"
sase bead read sase-1n6.5.1.5.2 -r "Need parent phase acceptance"
sase bead read sase-1n6 -r "Need outer release issues and inherited decisions"
sase artifact read plan:202610/deployed_qualification_proof.md "Need original acceptance and once-only retry rules"
sase artifact read plan:202610/qualification_landing_repairs.md "Need producer/consumer and exact-SHA contracts"
sase artifact read file:explicit:88fe402c602e56f995848006 "Need retained failure acceptance mapping"
```

## Verified starting state

The land audit reviewed every note on both children and the three epic commits:
`3db3b51d53` (Full CI permissions, shared suffix parser, contract manifest),
`ba7ab4e96b` (identity version reader), and `2d1e7fd486` (qualification observation
reader and executable workflow tests). At reviewed master `2d1e7fd486`, 121 focused
release/core/CI/manifest/transport tests pass. The epic has no symbol exemptions.

The latest compared live SHA is `bdec291c77b0de1aab4c00abecef7430761db736`:

- Green rehearsal `40000653149`: lint and Python 3.12 pass; 3.13/3.14 fail modal resize.
- Wrong-core rehearsal `40000657197`: sealed 0.42.2, installed 0.42.1; lint reports nine
  missing bindings. No deployed typed observation survived.
- Source Full CI `40000661173`: lint passes, all three test legs fail modal resize; 3.12
  also fails the Files Ctrl-J anchor-scroll node.
- Both rehearsal observation uploads were broken by the inline asset reader, now
  repaired in `2d1e7fd486`. Local reconstructions are diagnostic, not deployed proof.
- All were attempt 1. No rerun was consumed. Original candidate ZIPs, API pages, logs,
  digests and reconstructions are in `file:explicit:d1d05f2daface15a4c425e9f`.

Drift already integrated: `b2a4c380cf` moves setup-uv after the source/published core
checkout; `53b55f1fd1` preserves permission tests through a module split; `b97e6c4b91`
raised the published dependency to 0.42.2; `bdec291c77` repaired pytest plugin-option
handling. The shared suffix parser remains the one used by candidate, core-ensure and
metrics. `62c5934eb6` repaired visual fixtures/goldens after the compared run; visual
status must be observed fresh but remains advisory here.

New integration blocker: `5d982d2bb9` (`sase-1no.2`) added a required
`stranded_work_reduce` binding and moved `sase-core-revision.txt` from
`246e28743538b6cd4d4e098b6960085c3e8ef5a9` to
`58aca5fe69158286092e3a33685687a52f8d2b8f`. The latter has no containing published tag
in the opened core checkout. The isolated command

```sh
uv run --no-project --isolated --with sase-core-rs==0.42.2 python tools/check_sase_core_rs_bindings
```

fails with `missing 1 of 876 required binding(s): stranded_work_reduce`. A fresh
rehearsal at current master is therefore predictably red until the published floor is
integrated. Do not reuse the old 875-binding proof or downgrade the source pin.

## Constraints and follow-up ownership

Inherited decisions are final: `rerun_once=once`, `steward_host=workflow`, and
`release_cadence=daily`. No new decision is needed. Do not publish to PyPI, create or
move tags, change repository/PyPI settings, weaken binding validation, substitute a
source-built wheel for published qualification, or change admission/rerun policy. Use
the already-authorized daily release path; any explicit cut requiring authority outside
these instructions belongs to the outer release owner. No memory edits.

The lander triaged the existing proposals before submitting this plan. Read the parent's
LAND TRIAGE note for every disposition and newly filed task ID:

- Import-budget evidence stays on `sase-1ic`. The ref-prefix false positive is fixed by
  `a9f4b3d1b7`, with a passing current test; no new report or false corroboration.
- Full CI cache ordering is fixed by `b2a4c380cf` and proved by later running jobs.
- Modal resize remains owned by `sase-1nn` / `.4`; independent recurrence was added to
  that epic. Files Ctrl-J recurrence corroborates `sase-1a7` after its prior close.
- Historical advisory visual reds predate `62c5934eb6`; no post-fix regression claimed.
- Wheel `UPGRADE_NO_INSTALLABLE_PREVIOUS` and build SC2012 findings were routed to outer
  `sase-1n6`, whose `.4` and `.2` introduced those paths. They are separate
  exercise/build obligations; wrapper failure alone does not negate qualification.
- Three local TUI failures share a proven diagnostic-environment isolation defect,
  tracked by `sase-1nq`; the distinct broad-suite descriptor ordering flake is
  `sase-1nr`. Neither belongs to this qualification implementation. Preserve the
  original check's nonzero exit and NEW classification.
- Source containment enforcement is still outer `sase-1n6` DISCOVERED ISSUE #1.
  Independently prove containment for the chosen pair; do not claim that closes the
  manifest/evaluator enforcement gap. New binding lag is also noted on `sase-1no`.

## published-pin

Owned changes are the exact `sase-core-rs` dependency in `pyproject.toml`, its `uv.lock`
resolution, and only directly necessary existing dependency contracts. No Rust behavior
change is expected. Read `lint_and_test` memory before verification.

1. Fetch remote master and inspect drift since `2d1e7fd486`. Open `sase-core` with
   `/sase_repo`, read its instructions, and refresh master/tags through that checkout.
   Re-read the source pin and required binding inventory. Check whether another release
   worker already landed the complete containing published pin; if so, verify and reuse
   that commit instead of repeating the update.
2. Find a published core release whose tag contains the **current** source pin and whose
   live PyPI files satisfy the suffix set read from that tag through the shared parser.
   A release PR, local tag, version string, or source-only wheel is insufficient. Verify
   ancestry locally and publication/file availability independently. If no such release
   exists, use `/sase_monitor` for the inherited bounded roughly 36-hour next-daily-cut
   retry. Carry the pin, missing bindings, deadline, owner notes, and these remaining
   steps into the monitor continuation. Timeout remains an explicit blocker with the
   phase open; do not loop indefinitely or declare readiness.
3. Raise the exact Python dependency to that complete containing release and refresh
   only its lock resolution using the established repo workflow. Preserve unrelated
   dependency versions. Re-run the isolated published-wheel binding preflight against
   current source; every binding must exist. Keep the source pin intact unless newer
   already-authorized source integration requires moving it forward. Record tag commit,
   source pin, version, wheel digest, completeness files and binding count.
4. Recheck `sase-1nn` modal-size repair progress and `sase-1a7`; record landed fixes and
   outstanding failures for the next phase. Do not take over unrelated repairs or claim
   focused tests prove a live CI pass. Preserve the new environment and descriptor-order
   task ownership. A still-open external issue is a readiness warning for the proof
   worker, not permission to remove a blocker from the inventory.
5. Run focused release/core/build-core/dependency contracts as appropriate, format, and
   run `sase tool run check` (the guarded `just check`) for file changes. Never run
   `just check-full`. Use `/sase_monitor` for long verification. Retain genuine
   failures/triage without changing exit codes. Note the stable next-phase contract;
   close only this phase when the integration is verified. Host finalization commits at
   turn end; do not wait for this phase's own SHA, push, or CI run.

## deployed-reproof

This phase depends on `published-pin` and its changes being on remote master. It is
evidence work; do not repeat the first phase or the already-landed observation repair.

1. Fetch master and prove it contains the integration commit and `2d1e7fd486`. Recheck
   source/tag containment, live PyPI completeness, isolated green preflight, and drift
   in release/CI/full/build-core workflows, inventory, collector, writer and shared
   evaluator. If newly landed code requires an unpublished binding, retain the exact
   failure and return to the bounded core-readiness path; never silently relax published
   qualification. Review `sase-1nn` and `sase-1a7` evidence before spending another full
   run on a known persistent failure. Waits for an active external repair use
   `/sase_monitor`, with a stated deadline and continuation, not a normal final response
   promising to resume.
2. Choose the nearest older complete published core that demonstrably lacks a current
   required binding. After a containing successor to 0.42.2 lands, 0.42.2 is the first
   candidate probe because it lacks `stranded_work_reduce`; verify instead of assuming.
   Record the actual missing names and keep the intended version separate. The old
   plan's 0.41.0/mobile_pairing_issue example was already superseded by its changed-pin
   rule; do not require that obsolete name.
3. Ensure no release.yml run at master is running or pending. Then dispatch back to
   back, retaining dispatch time, run IDs, URLs, events, attempts and exact headSha:

   ```sh
   gh workflow run release.yml -R sase-org/sase --ref master -f mode=rehearse
   gh workflow run release.yml -R sase-org/sase --ref master -f mode=rehearse -f probe_core_version=<verified-probe>
   gh workflow run full.yml -R sase-org/sase --ref master
   ```

   All three SHAs must match. Never dispatch a third Release while the probe is pending.
   Full CI has cron `17 */2 * * *` and group `full-ci`; an existing or scheduled run is
   usable only at the exact compared SHA. If alignment is lost, follow the original
   plan's at-most-three alignment attempts, retaining every abandoned attempt. Never
   substitute a later green source run.

4. Use `/sase_monitor` for bounded polling, under four hours for each aligned batch. Its
   continuation must carry IDs, compared SHA, versions, retry counts and steps 5-8. Do
   not end a provider turn while the monitor-start handoff is still running.
5. Download the actual `candidate-build` and
   `qualification-observation-run-<run>-attempt-<n>` artifacts for green and probe.
   Verify manifest identity, candidate ID/generation, SHA, intended/tested/installed
   core, inventory digest, canonical observation filenames, artifact digests and
   run/attempt identity. Retain complete paginated jobs/artifact metadata, required job
   IDs/URLs/conclusions, positive collection counts, failures/errors, advisories and
   FLAKY lines. Missing deployed observation is failure, not permission to replace it
   with a locally reconstructed one.
6. Re-evaluate archived original manifests and actual uploaded observation histories
   with `python3 tools/release_candidate evaluate --manifest ... --observations ...`.
   Green qualification must pass with lint and test 3.12/3.13/3.14 successful and
   positive collected counts/zero failures/errors. Probe must be typed `QUALIFY_RED`
   naming lint and the actual missing binding(s), intended version unchanged, tested
   version equal to the probe, `rerun_eligible=false` and its reason. Source Full CI
   must pass lint and all three test legs at the same SHA and record the actual
   source-built core revision and bindings check. Source wheel package metadata may lag
   the source revision; do not confuse that wheel with the older PyPI probe.
7. Honor `rerun_once=once`: at most one `gh run rerun <id> --failed` per eligible run.
   Rehearsal eligibility must come from the evaluator on retained first-attempt
   evidence; source eligibility requires every failed blocking job to be a test failure.
   Preserve both attempts. Missing/setup/collection/harness/lint failures and a second
   red are ineligible. Do not use redispatches to evade this bound. Assess
   qualification, exercise and publication separately. Failed wheel exercise or advisory
   visuals remain reported to their owners; rehearsals do not publish.
8. Register durable original observations/manifests, evaluator inputs and outputs, job
   summaries, source proof and one acceptance mapping via `sase artifact create`. Note
   refs, URLs, compared SHA, versions and every acceptance result on this phase,
   `sase-1n6.5.1.5.2.1`, `sase-1n6.5.1.5.2`, `sase-1n6.5.1`, and `sase-1n6.5`. Close
   only this phase when all three proof obligations are met. For a further
   qualification-caused code defect, fix/test it and record `REPROOF REQUIRED` using the
   original repair-handoff rule; never call that deployed acceptance. Unrelated failures
   become phase proposals for the lander, with honest blocker status.

## Landing handoff, outside the phases

The child land agent reviews both phases and all notes, source changes, post-start
drift, decisions, follow-ups, and actual retained proofs. It must not close this child
epic on another repair-only handoff, a timed-out wait, or red/missing proof.

Once this child is complete, its `parent_bead` resumes the interrupted landing of
`sase-1n6.5.1.5.2.1`. Re-read that epic's landing note, all descendants and notes,
`plan:202610/deployed_qualification_proof.md`, and post-child drift; rerun descendant
and linked-plan readiness checks. Resolve or re-key every entry from
`sase bead epic-symbols sase-1n6.5.1.5.2.1` to a genuinely still-open consuming bead,
then close that parent epic normally with the evidence-backed verification note, run
`just symvision`, and mark its linked plan `status: done` through the sanctioned plans
checkout. Its PLAN currently resolves to `plan:202610/deployed_qualification_proof.md`;
resolve it again rather than guessing a machine path. Never force a successful nested
close.

Then re-read the parent link: it is phase `sase-1n6.5.1.5.2`. Verify these retained
proofs meet its scope and close **only that phase** normally with refs, URLs, SHA and
acceptance mapping. Leave containing epic `sase-1n6.5.1.5` and the higher ancestors to
their waiting land agents. Keep outer exercise and containment enforcement issues
explicitly open. If any parent is incomplete or ambiguous, note the concrete blocker
there and stop the ancestor closeout.
