---
tier: epic
title: Finish exact evidence and bounded published-state repair
parent_bead: sase-1pu.5.1
goal: Published run repair reads only bounded exact-run evidence, preserves state
  on uncertainty, and advances checkpoints only on verified progress.
phases:
- id: evidence-policy
  title: Complete raw-evidence and bounded-discovery contracts in Rust
  size: medium
  depends_on: []
  description: 'evidence-policy: implement shared raw-marker classification and bounded
    hood-discovery contracts described below in sase-core, including registered bindings
    and tests. Reuse the existing terminal precedence and scheduler policy. Record
    the host-created core commit for the dependent Python phase.'
- id: repair-integration
  title: Integrate exact evidence, honest repair progress, and current publishers
  size: medium
  depends_on:
  - evidence-policy
  description: 'repair-integration: consume the preceding core commit, replace publication-inventory
    hydration in state repair with bounded exact-run observations, fix failure reporting
    and terminal preservation, integrate later publication changes, retire the parent
    checkpoint symbol exemption, and prove the real CLI and scheduler paths.'
proposed_by: bbugyi200.athena.sase-1pu.5.1.land
proposed_with: codex/gpt-6-astra
create_time: 2026-11-01 10:17:37
status: done
bead_id: sase-1pu.5.1.5
---

- **PROMPT:** [prompts/202611/published_state_landing_repairs.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/published_state_landing_repairs.md)
- **PARENT:** [202610/published_run_state_convergence.md](https://github.com/sase-org/sase--plans/blob/main/202610/published_run_state_convergence.md)
- **BEAD:** [sase-1pu.5.1.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pu/sase-1pu.5.1.5.md)

# Finish exact evidence and bounded published-state repair

## Scope and reason for two phases

This is only the remaining work from the landing audit of `sase-1pu.5.1`, whose plan is
`plan:202610/published_run_state_convergence.md`. Its parent is phase `sase-1pu.5` in
`plan:202610/task_backlog_epic_3.md`. The accepted ancestor decision is
`optional_scope = keep_five_seeds`. Preserve it: no unrelated features, memory edits,
live fleet backfill, global installations, or historical publication purge.

A tale was considered first. The missing classifier currently lives in Python, whereas
the Rust resolve binding accepts already-classified evidence. Correcting that backend
boundary and bounding discovery requires a small core contract extension, followed by a
Python consumer whose revision pin must contain the host-created core commit. These are
two dependent implementation phases, not a single coder waiting for its own commit. The
child lander handles ancestor readiness after those commits land; parent closeout is not
an implementation phase.

## Audit evidence

The audit read every note on the original epic and its four closed phases, both linked
plans, and actual code and commits at sase `5036f01eb2` (also origin/master at review
time). Core `7de361cc` implements the original policy and bindings; sase commits
`1d1c46262a`, `5c14ac6a54`, and `b1618348d4` implement publication safeguards, repair,
and scheduling. The current core pin `764317b9` contains `7de361cc`.

Existing evidence is useful but incomplete: 42 publication-preservation, repair,
scheduler, job-registration, and formerly failing scoped-run tests pass, plus 11
completion snapshot/kind tests. The scheduled convergence and scoped-preview tests
inject already-classified `InventoryRun` loaders. They do not write real completion
markers or observe the production loader's payload reads.

Confirmed remaining failures:

1. `inventory_sources._artifact_state(record, {})` returns `failed`; an unknown outcome
   also returns `failed`. A completion marker with a conflicting `name` returns
   `completed`; a literal `stopped` outcome returns `failed`. The claimed exact-run
   validation is a comment, not an implemented evidence check.
2. `_publication_state_repair_evidence.default_inventory_loader` calls
   `build_project_hood_inventory(requested_hoods=...)`. That performs historical commit
   discovery and hydrates prompts, chats, embedded workflows, and prompt steps for all
   runs in selected hoods. It cannot receive selected run IDs. `collect_published_rows`
   loads every owner snapshot before selection without a deadline. Page planning also
   re-reads all referenced content in changed hoods.
3. With one selected active run and completed evidence, an already-expired deadline
   returns `examined=1, failed=1, changed=0`, a non-null cursor, `pass_complete=true`,
   and no error. A corrupt owner manifest returns `failed=0, pass_complete=true`.
   `_push_ahead_convergence` also treats an unreadable ahead count as a successful
   no-op. The scheduler trusts these reports and can erase the retry position.
4. `resolve_preserved_run_state` returns the candidate on a Rust validation exception.
   Published `completed` with a malformed historical timestamp plus a candidate `active`
   becomes `active`, despite a diagnostic saying unchanged.
5. `sase bead epic-symbols sase-1pu.5.1` still lists
   `sase-1pu.5.1(AgentRunStateProjectCheckpointWire)` in local and upstream Justfiles.
   The type is currently constructed and referenced only inside its wire module. This
   must be resolved during implementation, before the child lander attempts ancestor
   closeout. The close guard unions local and upstream entries, so removing it only
   during the final landing turn would be too late.

The oversized repair module reported in original epic note 1 was already split by
`cf68c745f1`; `sase-1q1` is closed. Keep those focused modules.

## Phase evidence-policy

Open `sase-core` with `/sase_repo` and follow its instructions. Work in the
`agent_run_state` domain and its existing Python binding module.

Add an additive, typed observation-classification contract that the publication
inventory and repair adapters can share. Python supplies filesystem observations; Rust
decides whether they prove terminal state. Reuse the existing resolve policy for
precedence, supported timestamp parsing, later dismissal, identity conflicts, and
unchanged results.

- Represent missing, unreadable/malformed, unfinished, completed/failed/stopped, and
  dismissed evidence explicitly. An empty object or unknown outcome never means failed.
  Preserve the existing supported completion-outcome vocabulary; investigate the actual
  marker writers instead of assuming every non-completed outcome is failure.
  Repeated-run stop evidence must remain stopped.
- Bind evidence to project, owner, canonical global name, and source-run identity.
  Accept supported older markers whose identity is established by the exact artifact
  directory and metadata; reject conflicting identity fields when present. An absent
  optional finish time stays absent. Names alone, PID absence, age, and commit-history
  placeholders do not establish an outcome.
- Preserve separate candidate observations rather than choosing the first same-ID record
  or blindly preferring a live copy over a dismissed archive. Ambiguity and
  contradictory terminal evidence remain diagnostic and unchanged; verified later
  dismissal follows the existing Rust rule.

Extend the window policy just enough to let Python discover a bounded hood window before
hydrating its snapshots. Inputs include the validated owner manifest's hood names,
project/owner identity, and the opaque cursor. Rust owns cursor decoding, ordering,
bounds, and progress over examined empty or corrupt units. Keep the order
`(local_hood, source_run_id)`, default limit 100, public limits 1..500, and at most five
selected hoods. Preserve existing `ars1` cursor support; add a distinguishable cursor
form only if empty-unit progress needs it, and test old cursors explicitly. Do not add a
new corpus index or change public v2 snapshots.

The discovery/selection response must distinguish a completed pass from a partial scan.
A corrupt individual hood may be diagnosed and skipped so healthy units advance; an
unreadable owner manifest or an interrupted scan cannot pretend that the owner has no
work. A hood with more than the run limit must resume within that hood before advancing.
Bound diagnostics as well as the selected work.

Add core and real registered-binding tests for raw valid/invalid markers, identity
mismatch, supported missing fields, conflicting observations, stopped and later
dismissed states, pagination within a large hood, corrupt/empty hoods, inserted/deleted
rows, owner-bound cursors, and invalid limits. Reuse the existing
resolve/window/schedule tests. Run the linked repo's wrapped `sase tool run check`;
never bare cargo or a global install. Leave Python callers and the revision pin to the
dependent phase, after the host has created the core commit.

## Phase repair-integration

Read the preceding phase's notes and actual core commit, open the linked repo through
`/sase_repo`, and move `sase-core-revision.txt` to a containing commit. Use thin Python
wire/facade adapters for the new contracts, without a Python backend fallback or a
second implementation of terminal precedence.

Replace the repair inventory loader with a state-only reader that receives the selected
immutable run identities. Existing bounded artifact-index queries
(`query_agent_artifact_index_bounded`, its candidate filters, and exact record loads)
and dismissed-bundle summary lookup are available. Use them to locate candidates, then
read current bounded marker/metadata files only for those runs. Do not refresh/rebuild
the index, walk source history, read local transcripts or prompt archives, run Git
history, or build a full publication inventory to classify a selected run. Missing or
ambiguous lookup is unresolved, not a reason to fall back to an unbounded scan. Read
dismissed bundle bytes only for selected matches. Reject foreign/imported observations
and validate paths against their owning roots.

Use the same observation adapter in ordinary inventory publication. Resolve terminal
state before generating and scanning state bytes. For an unresolvable same-run merge,
retain all published state/timestamp fields or withhold that run; never publish the
active candidate while claiming unchanged. Keep independent run identities separate and
preserve missing timestamps without inventing them.

Make discovery and application honor the existing 90-second work budget, including
checks before expensive collection and publication steps. Apply existing byte bounds
before reading oversized files. Read only the selected hood snapshots for state planning
and reuse validated metadata where derived indexes require wider context. Updating a
complete snapshot may retain unselected records; it must not hydrate their local
prompts/chats or revalidate unrelated content merely to change state. Reuse existing
targeted rendering/protection infrastructure and preserve foreign-owner and
corrupt-sibling index/page entries. The normal publication protection boundary still
applies to every new byte.

Repair reporting must distinguish verified progress from retryable failure:

- Manifest corruption, budget exhaustion before verification, unknown push/ahead state,
  and failed/uncertain publication retain the usable starting position and report a
  failure, never a completed pass. Ensure no-write branches obey the same rule as
  writes. Keep valid empty manifests as successful empty passes.
- Live, missing, conflicting, or individually corrupt rows can be examined and diagnosed
  without starving later rows. Partial work must expose exactly the progress proven by
  the core window policy, so retry neither loses a row nor repeatedly traps healthy
  siblings behind one irreparable row.
- Recompute snapshots and fresh local evidence after non-fast-forward recovery. Do not
  retry a precomputed state patch or drain unrelated outboxes.
- Keep the scheduler's `86bec79ca4` pessimistic pre-charge, fair rotation, and failure
  backoff. Failed reports cannot reset cursors as a successful pass.

Integrate changes that landed since the original epic began:

- Background commit enqueue/drain (`4356cffaa1`, `e69360f49d`, `6d615680b8`) and planner
  linked scopes/archive restore (`9409312d1e`, `f1dcab6dae`, `4900d9a0f3`) still flow
  through shared hood planning. Preserve their fulfillment semantics; add coverage
  proving a later background/planner publication cannot regress a state already repaired
  to terminal. CLI repair modes remain mutually exclusive.
- Preserve published-byte protection reuse from `30e012baaa`. Changed state bytes still
  get normal protection checks; identical preserved bytes remain reusable.
- After reading the docs-writing memory, correct the obsolete immediate-drain paragraph
  in `docs/contribute/agents-sidecar-sync.md` to match background enqueue with its
  existing fallback. Keep the owner-local preview/apply/resume runbook accurate about
  partial progress and failed windows. No new CLI or config knobs are needed.

Resolve `AgentRunStateProjectCheckpointWire` according to Symvision's hierarchy: make it
private if it remains wire-module-internal, or consume it meaningfully in the
scheduler's typed checkpoint conversion. Remove its parent `--epic-symbol` line. No
test-only pragma, dummy import, or transfer to an unrelated bead. Run
`sase bead epic-symbols sase-1pu.5.1` and account for every entry; document the expected
upstream-only row until the host commits this phase. The child lander will see the
landed removal, not wait for its own commit.

Acceptance requires tests through the production reader, not only injected
`InventoryRun` objects. Seed temporary sidecars and artifact indexes, publish a mixed
live hood, then write actual success/failure/stop/dismissal evidence. Drive the real
repair and scheduler with controlled time. Verify remote state bytes, snapshot fields,
manifest digests, pages, live siblings, and a second no-op pass. Instrument filesystem
reads so `state_limit=1` with many same-hood siblings proves that only the selected
run's evidence is read and no prompt/chat/history paths are touched. Cover
empty/malformed/mismatched markers, an unreadable manifest, budget expiry, unreadable
ahead state, partial/corrupt units, and concurrent same-owner publication. Existing
foreign-owner, push-failure/retry, protection, fairness, restart, and cursor tests must
keep passing.

Run focused relevant tests, `just fix`, and `sase tool run check`. Do not run
`just check-full`. If a check needs handoff, follow `/sase_monitor` and include the
remaining phase work in its continuation. No full check pass has yet been claimed for
the original epic; record actual results instead of inheriting the old notes. Record
results and follow-up proposals on this phase; do not create task beads.

## Follow-up triage already completed

The original lander used `/sase_new_task` and recorded every outcome on `sase-1pu.5.1`
before this proposal:

- Phase `.2` note 1, missing miniature-repo test helper: declined as fixed by
  `86bec79ca4`; the exact node passes now.
- Phase `.3` note 1 and phase `.4` note 2, unobserved check passes/timeouts: declined as
  distinct tasks; they are historical verification debt, not a diagnosed new
  infrastructure defect. The `.3` ToolRun ultimately failed; its NEW helper/completion
  failures now pass. The `.4` run is signaled/undetermined. Existing tasks `sase-1pv`
  and `sase-1pw` were inspected but not corroborated because these runs do not reproduce
  their respective foreground-handoff and token-starvation causes. The core-floor
  advisory is superseded by the containing pin.
- Epic note 1, oversized repair module: addressed by `cf68c745f1` and already closed
  task `sase-1q1`.

No new task or +1 was appropriate. Include these outcomes in eventual parent close
notes. The reproduced product defects above remain epic work.

## Landing and ancestor continuation

The generated child epic has `parent_bead: sase-1pu.5.1`. Its lander must verify both
phases, all notes, their actual commits, the containing core pin, production evidence
tests, current check results, and changes since this plan began. Retire its own symbol
obligations and close its own epic normally first. Mark this plan done after its own
successful close.

Then resume the parent-plan landing under the user-requested ancestor procedure: re-read
`sase-1pu.5.1`, its prior landing notes, all descendants and notes, and
`plan:202610/published_run_state_convergence.md`; rerun descendant/linked-plan readiness
and post-child drift checks. If complete, resolve every entry from
`sase bead epic-symbols sase-1pu.5.1`, then run
`sase bead close sase-1pu.5.1 --note "<actual verification, integration, and follow-up outcomes>"`.
Run `just symvision` and set `status: done` in that epic's linked plan frontmatter,
opening the plans repo through `/sase_repo` and reading artifacts through
`sase artifact read`.

Its direct parent `sase-1pu.5` is a phase. Verify its completion/failure/live-run,
concurrency, and bounded-backfill acceptance against the finished child work. Resolve
`sase bead epic-symbols sase-1pu.5`, record
`TASK RESOLVED: sase-1cc — <actual convergence and preservation evidence>` on
`sase-1pu.5`, and close only that phase with
`sase bead close sase-1pu.5 --note "<what was verified>"`. Leave `sase-1pu`, its linked
plan, and seed task `sase-1cc` to their already-waiting lander.

If any ancestor is incomplete or ambiguous, record its concrete blocker and stop
ancestor closing there. Never force a successful nested landing, never close unfinished
named phases merely to satisfy the close command, and never wait for the current turn's
own commit/SHA/push/CI. Use `/sase_final` for host-owned commits after the turn's work;
no manual branches, commits, or PRs.
