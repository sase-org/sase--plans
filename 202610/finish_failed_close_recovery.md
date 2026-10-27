---
tier: tale
title: Finish failed-close recovery and land its epic
goal:
  Recovery preserves every eligible close episode, proves publication and landing
  accurately, and safely closes sase-1oh.4.1 and its parent phase.
size: medium
proposed_by: bbugyi200.athena.sase-1oh.4.1.land
proposed_with: codex/gpt-6-astra
bead: sase-1oh.4.1
status: done
---

- **PARENT:**
  [202610/failed_stitch_bead_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202610/failed_stitch_bead_recovery.md)
- **BEAD:**
  [sase-1oh.4.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1oh/sase-1oh.4.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1oh.4.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1oh.4.1.land.md)
- **COMMITS:**
  - [d940a57](https://github.com/sase-org/sase--plans/commit/d940a57a9479d74d4d956bcb429a94ee2970b733)
    — docs(plans): mark failed-close recovery plans done after landing

# Finish failed-close recovery and land its epic

## Scope and inherited contract

Finish the four reproduced landing gaps in epic `sase-1oh.4.1`, then close that epic and
its parent phase `sase-1oh.4` in this same implementation turn. This tale has no land
agent and nothing resumes the original landing after it finishes. Do not close outer
epic `sase-1oh` or source task `sase-yo`; the outer land agent owns those. Do not wait
for this turn's commit, SHA, push, or CI to do closeout.

Read `sase-1oh.4.1`, its children `.1`, `.2`, `.3`, and `sase-1oh.4` through
`sase bead read`. Read the approved scope through
`sase artifact read plan:202610/failed_stitch_bead_recovery.md "Need recovery and landing contracts"`.
Its linked plan is `202610/failed_stitch_bead_recovery.md` in the plans repo. Parent
decisions `secret_scan=build` and `dormant_flakes=close` remain final; neither changes
these repairs. No memory edits are authorized.

This is one bounded implementation: repair the existing Rust recovery mutation, the
finalizer evidence adapter, and publication retry, with regressions at their current
seams. Preserve the working collector, doctor, ACE, and waiter behavior. Shared
eligibility rules belong in `sase_core`; Python supplies host observations and calls
bindings. Open `sase-core` with `/sase_repo` and read its `AGENTS.md`.

## Landing evidence already collected

The audited source is sase `c74a32c5c4` and core `f1ea08fb`. Core recovery commit
`4f17b727` is an ancestor of that pin. Python host recovery is `eb365afe30`, and surface
integration is `13abbfb01e`. All three phase notes were reviewed.

These existing tests passed in the landing audit:

- 73 tests across `tests/test_finalizers_failed_close_recovery.py`,
  `tests/test_bead/test_close_provenance.py`,
  `tests/test_stranded_collector_recovery.py`,
  `tests/test_stranded_recovery_surfaces.py`, `tests/test_stranded_work_facade.py`, and
  `tests/test_axe_chop_wait_checks_stranded_beads.py`.
- 32 tests across `tests/doctor/test_checks_beads.py`,
  `tests/ace/tui/test_artifacts_beads_stranded.py`, and
  `tests/test_stranded_single_turn.py`.

Intervening work was inspected: `98cb07ebb8` adds single-turn name lookup; `fabfff06e0`
and `c74a32c5c4` add/reconcile stranded-wait notifications; `9cb35859c9` guards
near-miss bead mutations; `538736f0f2` filters closed plan parents; `e7d9f7a479`,
`cef4bb1199`, and `4a64ff9a28` update capacity accounting; core cache repair and
integration-test consolidation also landed. Retain these changes. The fetched base
additionally had `cf9b9dbf39` (Muse commentary), with no recovery overlap. Recheck later
drift before closing.

Follow-up triage is already complete; do not refile it:

- `sase-1oh.4.1.2` note #1's group-revival flake is task `sase-1on`.
- Its two claim-check failures are consolidated into task `sase-1oo`; initial draft
  `sase-1op` was closed superseded before launch, with its evidence copied to
  `sase-1oo`. The shared index-first prepass can bypass the fixture scan; the triggering
  isolation mechanism still needs investigation.
- `sase-1oh.4.1.3` note #1's capacity-fixture proposal was corroborated on active epic
  `sase-1oj`, which owns dead-PID pruning and adoption. On the audited clean tree, a
  five-file run gave 47 passed and only
  `test_capacity_blocked_waiter_does_not_park_the_queue_behind_it` failed: PIDs
  100/101/102 are pruned and `drained` is `unexpected` instead of `None`. The other two
  proposed capacity nodes pass after adoption. No new capacity task.
- The retired umbrella `sase-ct` was not reopened or given +1. The narrow flakes have
  related links to it where relevant and to the leak-investigation epic `sase-j7`; mere
  topical overlap was not treated as causal ownership.

## 1. Recover every eligible close before ancestor reopening can hide it

In `crates/sase_core/src/bead/mutation/completion_recovery.rs`, `run_recovery` walks
close events in event-ID order. Recovering a child immediately reopens its ancestors.
When a parent also has a standing done close from this run, its later eligibility check
now sees an open bead and skips it as `close_not_current`. The parent loses its own
structured completion-failure fact and `STITCH FAILED:` note.

Reproduction with the real temporary store and existing helper `_close_phase_with_plan`:
close a phase and then its plan under the same provenance, then call
`recover_failed_closes`. The current report has one recovered close and one skipped
parent; both beads are open, but the parent has no failure fact and zero failure notes.
This violates the all-closes contract and removes the parent from open-recovery
candidate collection.

Compute the set of eligible current close episodes under the mutation lock before reopen
side effects, or order the mutation so an eligible ancestor receives its own recovery
event before descendants cause ordinary ancestor reopening. Retain the exact
current-event/provenance checks against concurrent human/successor changes. Do not emit
a later ordinary `IssueOpened` event that clears an ancestor's freshly recorded recovery
fact. Ancestors not themselves eligible still use normal reopen semantics. Preserve
close history, assignees, unrelated fields, and atomic replay.

Add core tests with phase/plan and a deeper ancestor chain closed by one run: every
eligible episode gets exactly one note and fact; a repeated request changes nothing;
canonical replay and cached reads agree. Keep controls for a human or different-run
close, non-done resolution, a newer close, and a later claim/reclose. Exercise the
existing binding and the Python collector on the repaired parent.

## 2. Require current repository evidence before suppressing recovery

`src/sase/finalizers/failed_close_recovery.py::_landing_proof` accepts the first
same-repository/same-run marker containing a SHA. It does not prove that marker covers
the outstanding attempt/obligation. A direct probe with two different operation IDs, an
older pushed marker followed by a newer unpushed marker, returns `landed=True`. A
successful earlier controller cycle followed by new dirty work and a failed
before-commit attempt can likewise be masked by the old marker.

Normalize evidence from existing commit ledgers, operation records, checkpoints,
accepted repository obligations and baselines. Use the available attempt inputs
(`repo_path`, `head`, dirty fingerprints, message digest) and operation identity to
prove coverage. Reuse existing reconciliation APIs where they express the rule; put any
new shared assessment rule and its wire/binding in the Rust finalizer domain rather than
adding another Python policy implementation. Keep host I/O in the adapter. Do not infer
landing from exit status, clean status alone, the first matching marker, an unrelated
repository, or a prior attempt.

Also exercise prepared completion: `source_identity_scope` attributes closes to the
source run, while `host_completion_state._bind_finalizer_process_identity` and
`workflows/commit/commit_tracking.py` bind generated markers to the monitor. Use the
authenticated prepared-intent relationship to compare source close identity with
executing-attempt proof. Do not globally relabel the monitor or accept arbitrary
cross-run markers. Check these paths end to end instead of testing only environment
restoration. Authentic handoffs, active repair, clean/no-code completion, successful
retry, and proven fully landed work followed by an unrelated failure leave closes alone.
Partial multi-repository landing still requires recovery.

Add focused regressions for older success/newer unpushed, older success/new dirty
attempt failing before a marker exists, unrelated operations/runs/repos, successful
retry, post-commit failure after all obligations land, and monitor source/executor
identity. Preserve the original terminal error in every recovery path.

Use that same attempt evidence when naming message files and hook output. Current
`_message_files_for_repo` globbing can mistake another run's preserved message for a
stitch in the current declaration-failure case. The current run's recorded attempts
should identify evidence; a declaration failure with no stitch must say so even if old
message files exist in the checkout.

## 3. Resume recovery-store commit as well as push

In `_run_machine_lane_recovery`, a failed `_commit_recovery_mutation` leaves the
canonical recovery event dirty. The next core call returns `changed=False` with
`already_applied`, so the coordinator skips committing, attempts a push, and returns
`committed=True`. A temporary Git-store probe confirmed: first call changes state and
reports commit failure; second call reports success with only one commit call total and
dirty recovery events remaining. Existing publication tests cover a push failure only.

On retry, prove that the exact recovery events are in the local commit before claiming
publication. Retry a pending recovery-store commit through the existing machine-owned
transaction lane, then push and check the actual result. An idempotent core result is
not a commit receipt. A false commit return must distinguish no new work already proven
committed from a real failure. Preserve unrelated dirty work; never reset/stash/discard
it while acquiring or refreshing the machine lane. Retain the actionable request/outcome
and original finalizer error on failure.

Add a real temporary Git remote fixture: force the first recovery-store commit to fail,
retry, and read the remote clone to prove its event exists, with exactly one failure
note/history episode. Repeat with a push failure and another retry. Assert reported
`committed`/`published` values match those proofs. Cover successor claim or reclose
before retry so publication cannot roll its lifecycle state backward.

## 4. Redact recovery diagnostics before durable publication

`_build_recovery_note` and `_hook_tail_for_repo` only use `bound_stream`; that helper
truncates the beginning and does not redact. A synthetic token-bearing diagnostic was
copied verbatim into the note, whereas existing
`sase.procs.text_bounding.bound_and_redact_text` removes it. No real credential was used
or inspected for this finding.

Reuse existing redaction and tail-bounding helpers on hook output and exception
diagnostics before writing the bead note, request/outcome artifacts, or journal. Keep
artifact locations and useful failure codes; do not copy raw transcripts. Bound the
combined diagnostic content as well as each repository's contribution. Use synthetic
credential lines in tests and assert their values are absent from all newly produced
durable surfaces, while safe diagnostic context and the true tail remain. The newer
agents-sidecar protection is defense in depth, not a reason to publish raw secrets into
bead events.

## 5. Verify the repaired composition and later integration

Read `lint_and_test` before finishing and any domain memories required by actual edits.
Do not weaken assertions, raise timeouts, skip tests, or add suppressions to hide
failures. Do not change memory, crate versions, or changelogs. Read `symvision` before
resolving whitelist or symbol failures. Avoid UI/doc changes unless a repair actually
changes their contract; if needed, read their memories.

Run core's focused mutation/replay/binding tests through its `just test` recipes, then
`sase tool run check` in the changed core checkout. Never use bare cargo. Build the
local binding through the supported workspace workflow before Python integration tests.
Run the 105 focused tests listed above plus the new regressions and relevant
monitor/declaration harness tests, then `sase tool run check` in sase (the wrapped
`just check`). Do not run `just check-full`.

Use `/sase_monitor` when a command needs handoff; its continuation must still finish
step 6. Do not end a normal turn with a live command. Existing independently verified
infrastructure failures belong to the triaged follow-ups, not this epic; investigate
genuinely new failures and retain exact evidence.

Recheck commits on the current base after the audit SHAs and integrate any new recovery
callers or conflicting logic. Preserve single-turn query/deduplication, targeted
stranded notifications, external waiter-release reconciliation, and the near-miss guard.
Record checks and drift review on `sase-1oh.4`.

For the core pin, `docs/contribute/rust-backend.md` explains that a declaration
committing both repos lets the host commit/push the linked core first and update
`sase-core-revision.txt` before committing sase. Declare both changed repositories; do
not invent a future SHA or wait for this turn's commit before closing. Ratchet manually
only for an already-landed required core revision, without regressing the current pin.

## 6. Finish the original landing in this turn

1. Re-read `sase-1oh.4.1`, all descendants and their notes, and the linked plan. Confirm
   the four repairs and the original acceptance contract are complete. Preserve the
   follow-up dispositions above in the close note. If this tale has its own descendant
   bead, complete and close it normally before closing the enclosing epic; never force a
   successful nested landing.
2. Run `sase bead epic-symbols sase-1oh.4.1`. Resolve every listed symbol by wiring,
   privatizing, legitimate non-test use per Symvision policy, or removal. Re-key an
   exemption only if a specific still-open later bead actually needs it. The audit found
   none, but rerun immediately before close.
3. Run
   `sase bead close sase-1oh.4.1 --note "<verified recovery, replay, publication retry, evidence safety, integration, check results, and all follow-up outcomes>"`.
   If refused, finish the named remaining work/whitelist cleanup and retry normally.
   Never use `--force` merely to succeed, and never force successful nested landing.
4. Run `just symvision` and confirm the whitelist remains clean. Open the plans repo
   with `/sase_repo`, read the artifact through `sase artifact read`, and set
   `status: done` in frontmatter of `202610/failed_stitch_bead_recovery.md` in the
   returned checkout. Include that changed repo in the final declaration.
5. Inspect the parent link with `sase bead read sase-1oh.4.1 -r "Need the parent link"`.
   The expected parent is phase `sase-1oh.4`. Read it, verify this child plan satisfies
   its stranded-close work, and run `sase bead epic-symbols sase-1oh.4`, resolving or
   validly re-keying any entries. Close only that phase normally with
   `sase bead close sase-1oh.4 --note "<child epic verification and source-task evidence>"`.
   Leave containing epic `sase-1oh` and source task `sase-yo` to the waiting outer land
   agent. If the phase is incomplete or ambiguous, leave it open and record the concrete
   blocker. Do not force it or close other ancestors.
6. Finish through `/sase_final`, declaring every changed repository. Report the actual
   closed beads and verification results. Closing happens before the host's commits; no
   step depends on this work's own future SHA, push, or CI.
