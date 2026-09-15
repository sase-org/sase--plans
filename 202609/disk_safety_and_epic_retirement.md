---
tier: epic
title: Retire unsafe disk cleanup paths and settle the sase-zw epic chain
goal: 'Artifact-run deletion fails closed, workspace reuse preserves object dependencies,
  cleanup failures remain visible, the tested core is pinned, and the abandoned sase-zw
  epics are closed with explicit scope and evidence.

  '
phases:
- id: core_safety
  title: Establish the three bounded Rust safety contracts
  depends_on: []
  size: medium
  description: 'core_safety: refuse artifact-run mutation, distinguish reuse from
    guarded maintenance, and normalize cleanup outcomes in Rust.'
- id: safe_callers
  title: Integrate preview-only run retention and safe workspace reuse
  depends_on:
  - core_safety
  size: medium
  description: 'safe_callers: pin the finalized core, route both manual prune paths
    through refusal, and preserve existing borrower dependencies during launch.'
- id: cleanup_results
  title: Propagate owner failures and preserve partial effects
  depends_on:
  - safe_callers
  size: medium
  description: 'cleanup_results: adopt the core outcome contract for discovery, subprocess,
    scratch, and proc failures without losing successful effects.'
- id: verify_delivery
  title: Remove unsupported test allowances and verify the integrated delivery
  depends_on:
  - cleanup_results
  size: medium
  description: 'verify_delivery: repair the fourteen-node baseline addition, verify
    the exact pinned wheel, and record the finite acceptance results.'
proposed_by: bbugyi200.athena.0ln.f0
create_time: 2026-09-15 19:35:45
status: wip
bead_id: sase-11h
---

- **PROMPT:** [prompts/202609/disk_safety_and_epic_retirement.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/disk_safety_and_epic_retirement.md)
- **BEAD:** [sase-11h](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11h/README.md)

# Retire unsafe disk cleanup paths and settle the sase-zw epic chain

## Decision for approval

Finish the demonstrated safety and reporting defects, then retire the old epic chain.
Use a **standalone epic with four serial, medium implementation phases**. Shared Rust
contracts land first so the Python integration can pin a real finalized core commit.
This exceeds one medium tale, but none of its phases needs another planning epic. There
is deliberately no `parent_bead`: approval replaces the old requirements rather than
entering another recursive landing audit.

The material product tradeoffs are part of this proposal:

1. **Artifact-run retention becomes preview-only.** Both manual apply commands and the
   Rust apply binding refuse deletion. Old run directories can consequently accumulate.
   Building a transaction across artifact references, bead/plan/gate state, and
   continuation publication is not justified for this cleanup feature now.
2. **Ordinary workspace reuse preserves existing object dependencies.** A healthy
   checkout can keep borrowing from its existing source. A broken borrower is preserved
   and reports a repair requirement. New shared clones and the existing, guarded manual
   maintenance commands remain supported.
3. **Cleanup reports partial effects and failures honestly.** A failed or incomplete
   owner observation cannot become a successful empty result. Accounting can explicitly
   be incomplete; it need not measure every historical byte to finish this work.

Approval authorizes retiring the newly introduced unsafe run-deletion behavior outright,
including its apply recommendations. This is an intentional scope reduction: do not
retain an opt-in unsafe legacy branch or add a flag whose removal creates another
unfinished project. Keep command parsing and an explanatory refusal for existing
callers. This explicit approved retirement takes precedence over the usual
temporary-deprecation flag convention. Other changes are ordinary safety fixes, not new
feature rollouts.

## Evidence and corrections to the earlier advice

Planning inspected main `f421051fdd`, core `a68ee7d`, and the current bead projections.
Both source checkouts were clean. The four open epics are `sase-zw`, `sase-zw.8`,
`sase-zw.8.7`, and `sase-zw.8.7.8`; the newest seven phases are still in progress with
no completion notes. All older phase beads are closed. No epic-symbol entries were
present for these four epics. Recheck these facts at execution; do not recover another
agent's uncommitted work by searching sibling workspaces.

Read context through audited commands when needed:

- `plan:202609/retention_landing_contracts.md` contains the superseded seven-phase
  scope.
- `file:explicit:b1aa712c5de680aad6436f99` records the prior disposable-fixture failures
  and node-level test-history dispositions. Its historic verdict does not control this
  replacement plan's exit criteria.
- `sase bead show sase-zw sase-zw.8 sase-zw.8.7 sase-zw.8.7.8` supplies the historical
  acceptance and scope decisions. The original approximately 340 GiB was a backlog
  estimate, not a verified reclaimed-byte result; do not repeat it as a measurement.

Source inspection confirms the important defects:

- `agent_artifact_run_retention.rs` accepts omitted protection fields as empty and
  trusts a caller snapshot through mutation. Python refreshes that snapshot before
  candidate collection, leaving changing references unprotected.
- Run deletion is reachable through **both** `sase artifact prune-runs --apply` and
  `sase disk reap --apply`. The unattended pressure chop explicitly excludes it.
- `ensure_git_clone_at` calls dependency-changing reuse and recovery helpers without the
  manual maintenance commands' eligibility barrier. The earlier occupied-checkout
  experiment proved an unexpected rewrite, not observed object loss; preserve this
  distinction in reports.
- `workspace_project_keys` swallows discovery errors. Workspace JSON `errors=1` with
  process exit zero is treated as success. Scratch exceptions escape the group; later
  proc failures can erase earlier mutation outcomes.
- Main pins `3566872b4916123fedf100b7c5684c701085655c`, behind required core APIs. Main
  accepts published core `0.34.35`, whose tag predates its required run2/sharing2/
  inventory1 contracts. A development wheel with that version label proves no published
  wheel compatibility.
- `05be391f7a` added fourteen blanket flake allowances. Passing those nodes once does
  not justify keeping the allowances or claim to fix them.

## Deliberately retired requirements

Cancel the old whole-pass scratch/deep-tree/process budget redesign and whole-inventory
discovery/root-resolution redesign. They are real limitations, but the reviewed evidence
does not establish a current victim that justifies expanding this repair. Preserve
existing limits, protective skips, and visible partial-coverage diagnostics. Do not
claim complete bounded latency or complete configured-root coverage afterward.

Also retire the host-wide no-unowned-directory-over-1-GiB proof, another disk
reclamation target, cold/no-op/one-crate build benchmark campaign, and new proc
concurrency proof campaign. Existing proc store locks, missing/malformed-store refusal,
4,000-directory convergence tests, scratch retention, symlink protection, and
nonincremental build settings remain in force and their relevant existing regressions
must pass.

There is no new operational deletion in this plan. Preserve earlier declined backup
removal. Disposable test fixtures suffice for safety verification. A host inventory or
free-space reading is optional diagnostic context, never a landing condition.

Do not create replacement backlog tasks for these intentionally declined requirements.
Record the decision in closure reasons. Existing independently owned work such as
`sase-115` (historical orphan logs), `sase-v4` (future shard creation), and `sase-10d`
(published core floor) retains its own scope and lifecycle.

## Repository and execution rules

Open `sase-core` with `/sase_repo` and use only its printed path. Read its `AGENTS.md`.
Other repositories, if needed, use the same access procedure; artifact context uses
`sase artifact read`. No absolute numbered-workspace paths belong in durable outputs.
Shared safety/result policy belongs in Rust; Python supplies host observations, runs Git
and filesystem effects, and renders results.

The user canceled the old agents. Do not run `sase bead work` on an old epic or resume
its land agent. If a conflicting old worker is actually still live, settle that conflict
before editing overlapping files. On approved execution, append a short replacement
scope note to `sase-zw.8.7.8`, linking this plan, so subsequent readers know its
original seven phases are no longer the execution contract.

Use host-owned finalizers for source commits. Do not manually edit core release
versions. Each implementation worker runs main `just check` when it changes main and
core `just check` when it changes core, with Python >=3.12 and PyO3 included. Long
commands use `/sase_monitor`. A later correction to the Rust contract returns to its own
phase for repair and finalization before the main pin moves; it does not spawn a
remaining-work epic.

## core_safety

Implement only these three concrete shared contracts in `sase-core`, including PyO3
exports/schema fixtures as required. Choose compatible wire evolution deliberately;
missing new safety evidence must never select a permissive default.

### Run deletion refusal

The public run-retention apply entry refuses **every** mutation request with a stable
machine-readable blocked reason indicating unavailable authoritative protection. Return
zero removals and zero reclaimed bytes. Refusal precedes filesystem mutation, including
empty-shard removal. Empty arrays, omitted fields, caller-authored completeness
booleans, or a recent preview never authorize deletion. Preview classification stays
available. This is the selected final behavior, not a temporary promise to build locking
later in this epic.

### Existing-checkout reuse

Make new materialization, ordinary existing reuse, and guarded maintenance distinct
policy contexts. Ordinary reuse cannot install, replace, remove, or normalize alternate
dependencies or sharing config. A healthy dependency that differs from the newly
configured primary can remain usable; return a usable/preserved outcome rather than
requiring `status == expected`. Broken connectivity yields a preserved/refused outcome.
Absent context must not implicitly authorize existing-checkout mutation.

Manual maintenance keeps its current project-lock and operation-specific eligibility
requirements. A fresh claim/occupant observation that is positive or unavailable refuses
mutation; compaction also requires a successful clean status. Preserve the existing
distinction for explicit broken-alternate repair: missing objects can make status
unreadable, so do not accidentally prohibit its existing recovery route by applying
compaction's clean-status rule to every repair. Keep its worktree-preservation,
connectivity and rollback checks, and do not label unreadable status as clean. This plan
does not introduce a new claim store or guarantee exclusion of arbitrary external
processes. New-clone setup remains possible without falsely rejecting its reservation.

### Cleanup outcome normalization

Add a small typed Rust contract over the existing owner observations and steps. It
decides failure/blocked/incomplete status, partial `changed`, and known reclaimed bytes.
Any owner error, nonzero exit, invalid result, or unavailable required observation
prevents success. Protective skips and ordinary capped batches are not errors merely
because they leave work for a later pass. Keep byte-accounting completeness distinct
from deletion eligibility and operational success. Python retains effect sequencing;
this does not require a new generic orchestration engine.

**Exit:** Core tests cover unconditional apply refusal including an empty shard; reuse
that cannot request dependency mutation; explicit eligible/ineligible maintenance; and
mixed successful/failed owner outcomes. Run core's complete `just check`, including
bindings. Record the finalized core commit for the next phase. No source release or host
cleanup is needed to close this implementation phase.

## safe_callers

Consume the finalized core through a wheel built from its actual commit. Move
`sase-core-revision.txt` with the supported `just ratchet-core-revision` workflow after
checking its proposed target; its exit 2 means a ratchet, not failure. If the remote
target contains additional changes, test that exact target. Never pin an imagined SHA
for dirty source or use a sibling checkout as a substitute.

Update the thin facades and relevant validator schemas to match. In
`src/sase/core/agent_artifact_run_retention.py`, route apply directly to the refusal
contract and avoid an expensive protection scan merely to refuse. Preserve the public
result shape and report why apply is blocked. No deindexing occurs after refusal.

Update `artifact_cli/prune_runs.py` to exit nonzero for blocked/errors returned at
execution, including when preview itself succeeded. Integrate the same result in
`disk_footprint_reap.py`; its other owners may still execute, with the blocked run step
visible. Update retention previews, the hourly notification, and relevant docs so they
describe estimates and preview-only operation, with no apply recommendation. Retain
notification deduplication and timezone-aware timestamps. The pressure chop must still
exclude artifact runs and workspace compaction.

In `workspace_provider/_utils_checkout.py` and `git_objects.py`, ordinary reuse only
validates/preserves object dependencies. It must not invoke repair, dissociation,
repoint, or the post-recovery install path. Preserve broken borrowers with an actionable
manual repair error; never fall through to `rmtree`/rematerialization for them. New
materialization keeps working. Route explicit compact/repair/dissociate through an
explicit maintenance context while retaining `workspace_handler_maintenance.py`'s locked
claim/occupant recheck, connectivity verification, foreign/relative alternates, and
rollback.

**Exit:** Test through the installed binding and actual Python callers:

- Direct Rust apply with omitted protection fields and both CLI apply commands remove
  neither an old terminal run nor an empty shard and return a visible failure.
- The prior reference-during-collection reproduction cannot delete anything. Check
  artifact-index rows and retained run files, not just mocked call counts.
- Real Git healthy reuse with a different proposed primary, live child cwd occupant,
  dirty worktree, or claim leaves alternates and sharing configuration byte-for-byte
  unchanged. A healthy existing dependency remains usable. A broken borrower and its
  unique local files survive refusal.
- Fresh shared clone creation and existing manual maintenance success, locked-recheck
  refusal, connectivity, and rollback regressions pass. Ordinary reuse's invariant does
  not depend on winning a process-scan race.
- Preview/notification consumers accurately present the reduced capability.

Use disposable repositories and synchronized child-process ready markers. Existing tests
are in `tests/core/test_agent_artifact_run_retention.py`,
`tests/main/test_artifact_cli_prune_runs.py`,
`tests/workspace_provider/test_git_object_sharing.py`, and
`tests/main/test_workspace_handler_cleanup_repair.py`; discover split chop tests with
`rg`. Update tests whose old expected behavior is deliberately retired and retain the
underlying no-data-loss assertions.

## cleanup_results

Adopt the Rust outcome contract in `disk_footprint_models.py` and
`disk_footprint_reap.py`; keep CLI and chop exit/status decisions derived from it.
Implement this finite failure matrix:

| Input or event                                                       | Required result                                                                 |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Workspace discovery raises                                           | Failed discovery step; no successful empty-project message                      |
| Workspace JSON has errors with process exit 0                        | Aggregate failure and CLI nonzero                                               |
| Nonzero exit, timeout, malformed JSON or invalid field types         | Structured failed owner step; other owner outcomes preserved                    |
| Workspace owner changed data and also failed                         | Both `changed=true` and failure; known reclaimed bytes retained                 |
| Scratch owner raises or cannot complete a required observation       | Failed/incomplete step; continue independent owners                             |
| Proc rows/logs/runtime changed before a later cleanup error          | Preserve earlier effects and errors; do not replace the whole result with zeros |
| Owner legitimately has no candidates or protects an active candidate | Successful no-op/protective skip with its reason                                |

Carry partial proc effects through `procs/store.py`, `procs/models.py`, and
`procs/logs.py`. Return bounded per-pass log outcomes for current and rotated
store-owned logs instead of discarding them; preserve external-log ownership rules. If
measurement fails, report unknown/incomplete accounting rather than claiming exact zero.
Do not scan historical rowless logs or add a new sweep. Reuse existing runtime owner
outcomes and avoid counting the same effect in the row-prune and orphan passes.

**Exit:** Focused Rust/Python tests cover each row above and JSON plus text CLI
behavior. A mixed-effect result must prove both the completed fixture mutation and the
reported failure. Existing proc missing/malformed-store, reservation protection, and
convergence tests still pass. Main `just check` passes with the intended core installed.

## verify_delivery

### Repair only the fourteen-node allowance addition

Review `git show 05be391f7a -- tests/reproducible_flake_baseline.txt` and the cited
audit. Remove the blanket `sase-zw.8.7.7` justification. The two Vim nodes have existing
owner `sase-ni`; the production query oracle has `sase-10g`. Retain a narrow existing
allowance only when its actual evidence and owner support it. The shell-writer and
incremental tests correspond to deterministic tasks `sase-10p` and `sase-10v`, not proof
of flakes. Claimed-show has a documented fixing commit, `981004d5e`.

Remove unsupported allowances. Use `fixed-at` only with a verified fixing commit and its
real timestamp. A changed environment or historic dirty-tree failure followed by a pass
is not a fix. Do not change the global effective date, erase health records, broaden
suppression, or increase test-cost budgets to turn the gate green. Keep the unrelated
fixture improvement from `05be391f7a`; this is not a wholesale revert.

### Finite verification and delivery

Record the main revision/diff identity, finalized core SHA, pin, interpreter, wheel
origin/hash, version, and schema probes. Build/install from the exact pin without using
an unrelated editable extension or cache entry. Run all required-binding checks and the
behavioral refusal/eligibility/result probes; a version label alone is insufficient.

Run main `just check`, then **one integrated `just check-full` through `/sase_monitor`
with TESTING/TESTED**. Core's complete check must cover the same final core commit.
Repeat affected verification only after a relevant change or diagnosed failure; do not
rerun full suites hoping for green. Preserve the current test environment when comparing
results. Existing normal test debt can remain under its established owner; new
allowances cannot be manufactured by this plan.

Check the published dependency path separately using the existing core-floor probe and
release reconciler. If a compatible published wheel exists, use the supported
`ratchet-core-window` workflow and verify it in a clean environment. If the needed core
is not published, record the exact `blocked_unpublished` result and required SHA/schemas
on existing release-floor owner `sase-10d` after reading its current state. The existing
release-floor CI gate must remain hard-failing for an incompatible published wheel. Do
not publish a sase release with that gap or claim PyPI compatibility.

**Source acceptance does not require waiting for release-plz/PyPI.** Published delivery
is either verified or explicitly release-blocked under the existing owner. This is an
authored boundary of this plan, not a reason to keep the disk epics alive or launch
another child. Resolve available release reconciliation in this pass; do not manufacture
a new release task or manually bump versions.

Save one concise acceptance artifact with a row for each safety/reporting exit above,
exact verification results, the fourteen baseline dispositions, and source/published
delivery status. No fresh host inventory, benchmark, byte target, or historical
whole-plan audit is required.

**Exit:** The changed safety behavior and exact pinned build pass their targeted tests
and required repository gates; unsupported blanket allowances are removed; release
status is correctly classified. An actual unresolved test failure is recorded as failed,
not accepted as success. Return a specific blocker to this plan's owner if needed; do
not propose another remediation epic or quietly reintroduce retired requirements.

## Landing and explicit settlement of the old beads

The new epic's land agent owns this section; it is not an extra coding phase. The user's
explicit instruction to close/cancel the `sase-zw*` epics authorizes closing those old
ancestors, overriding the usual instruction that phase workers leave parent closure to
the old land agent. Do not revive the old landing chain.

First verify this plan's finite acceptance table. Then re-read current `sase-zw` family
state and notes once, checking only for new evidence that contradicts the accepted
safety fixes or changes which beads remain open. Do not inherit additional historical
exit criteria from their linked plans. Snapshot the closure disposition into the
acceptance artifact before mutating bead state.

Use `sase bead close` with **`--reason` on every newly closed bead**. Do not use
`update --status closed`. Reasons must include the replacement plan reference, actual
verification/artifact reference where relevant, completed useful work, and the specific
scope being declined. The intended resolutions are:

| Bead              | Resolution | Required reason content                                                                                                                                                                     |
| ----------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sase-zw.8.7.8.1` | canceled   | Additional whole-pass scratch/process bounding declined; existing retention and limits retained; no claim of complete bounded observation                                                   |
| `sase-zw.8.7.8.4` | canceled   | Whole-inventory discovery and root redesign declined; partial coverage remains a documented limitation                                                                                      |
| `sase-zw.8.7.8.2` | superseded | Data-loss risk removed by unconditional apply refusal; proposed cross-store transaction deliberately not built; run retention is preview-only                                               |
| `sase-zw.8.7.8.3` | superseded | Ordinary reuse preserves dependencies and broken borrowers; explicit guarded maintenance retained; universal new mutation protocol not pursued                                              |
| `sase-zw.8.7.8.5` | superseded | Finite owner failure/partial-effect matrix implemented; historical orphan-log reconciliation and generic orchestration expansion excluded                                                   |
| `sase-zw.8.7.8.6` | superseded | Exact core pin verified; state separately whether published floor passed or remains owned by `sase-10d`                                                                                     |
| `sase-zw.8.7.8.7` | superseded | Baseline repaired and replacement gates recorded; host-wide inventory and build benchmark campaign canceled                                                                                 |
| `sase-zw.8.7.8`   | superseded | Seven-phase plan replaced by this bounded safety/retirement plan; link all above dispositions                                                                                               |
| `sase-zw.8.7`     | superseded | Useful safety fixes retained and remaining demonstrated defects addressed; original universal bounds/measurement contract retired                                                           |
| `sase-zw.8`       | superseded | Retention and cleanup improvements retained; its continuing full-footprint acceptance contract replaced                                                                                     |
| `sase-zw`         | canceled   | End the original broad disk-footprint program at the user's direction; retain completed improvements, record preview-only run retention and unproved original host-wide promises explicitly |

Close open leaf phases individually with their specific reasons, then epics deepest
first. Existing closed phases retain their resolution/history; append a note only if new
evidence materially qualifies their old completion claim. Never reopen them merely to
improve a missing historical reason.

For example, after substituting real plan/evidence references:

```sh
sase bead close sase-zw.8.7.8.1 --resolution canceled --reason @scratch_close_reason.txt
sase bead close sase-zw.8.7.8.2 --resolution superseded --reason @runs_close_reason.txt
sase bead close sase-zw.8.7.8 --resolution superseded --reason @replacement_close_reason.txt
sase bead close sase-zw --resolution canceled --reason @program_close_reason.txt
```

Those examples are not a complete batch: settle every row in the table in dependency
order. `--force` closes unfinished descendants as well. **When canceling or superseding
an epic that still has open descendants, use `--force`, a non-done resolution, and
`--reason`.** Enumerate and note each descendant's disposition first; do not sweep
unseen work under a generic reason. Never use force to assert successful completion.

After settlement, query all statuses and filter exact `sase-zw` or `sase-zw.` IDs;
assert every family bead is closed and each newly closed bead has the intended
resolution and a substantive reason. Fuzzy search also finds unrelated tasks that
mention these epics: leave their lifecycle alone. Verify the four old epics' symbol
exemptions are empty and the normal bead-derived plan projection reflects terminal
state; do not hand-mark abandoned plan requirements as implemented.

Finally close the new epic through its normal landing flow only if its own finite
acceptance passed. If a real source/gate blocker remains, the old broad programs may
still be canceled with accurate reasons and the necessary repair remains on this
standalone plan. Report that result as incomplete, with the exact remaining blocker. No
automatic nested replacement is authorized.
