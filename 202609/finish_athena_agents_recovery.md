---
tier: tale
title: Finish athena agents publication recovery and close sase-11o.2
goal:
  Remove the full-reconcile payload-size blocker, verify athena publication recovery,
  and close only sase-11o.2 with current evidence.
size: medium
bead_id: sase-11o.2
proposed_by: bbugyi200.athena.sase-11o.2.f1
create_time: 2026-09-17 07:45:46
status: wip
---

- **BEAD:**
  [sase-11o.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11o/sase-11o.2.md)

# Finish athena agents publication recovery

## Scope and ownership

Finish the already assigned, in-progress phase bead `sase-11o.2`. Its prerequisite
`sase-11o.1` is closed. This is one bounded implementation and operational recovery, so
use a medium tale within the existing epic rather than creating another epic. The
current planning turn makes no implementation or recovery changes.

Read `sase bead show sase-11o.2` and the original design through
`sase artifact read plan:202609/agents_sync_manifest_slim.md "Need the original recovery acceptance criteria"`.
Do not set the bead's status by hand. Close only `sase-11o.2`, after all acceptance
checks below pass. Leave parent `sase-11o`, prerequisite `sase-11o.1`, related task
`sase-10x`, and all other beads to their owners. Record unrelated discoveries as
`PROPOSED FOLLOW-UP:` notes on `sase-11o.2`; do not create task beads.

## Verified starting point

The original defect was an oversized owner manifest. Phase 1 installed a tolerant
fat/slim reader, a slim writer behind `slim_agents_manifest`, dedicated manifest limits,
and scoped run-file validation. Those changes are present in this checkout.

The previous recovery agent reported clearing 71 retired requests, resetting more than
646 quarantined requests, and recovering 47 omitted on-disk hoods. Its next full sync
failed with `publication payload exceeds the byte limit`. Its recorded baseline was
approximately 2,091 hoods and 311.5 MB of agents payload. These are historical
measurements, not current guarantees. A read-only check on September 17 reported 681
quarantine diagnostics, including `sase-11l.2`; recapture the live baseline before
recovery. `sase bead epic-symbols sase-11o.2` currently reports none.

The blocker is still visible in the code:

- `src/sase/agents_sync/publication_planning.py::plan_hoods` includes every requested
  hood's payload even when its snapshot digest is unchanged, updates the owner manifest,
  and renders all browsing pages once.
- `publication.py::reconcile_agent_hoods` sends all eligible hoods to that planner and
  applies the resulting dictionary in one call.
- `v2_io.py::apply_payload_atomic` checks the sum of **all supplied bytes** against
  `MAX_PAYLOAD_BYTES` (128 MiB) before filtering unchanged files.
- `git_sync.py::_integrate_export_pass` invokes reconciliation during normal sync, then
  restores deferred prompt archives. `git_sync_transaction.py` already owns pull,
  commit, push, non-fast-forward recomputation, and cleanup. Outbox requests are
  acknowledged only after successful sync and materialization checks.

Keep those transaction and acknowledgement semantics. Merely ignoring unchanged
snapshots would miss missing or drifted payload files and leave a first-time or
genuinely large changed publication subject to the same aggregate failure.

## Implementation

### 1. Add a bounded full-reconciliation apply path

Keep snapshot construction, digest validation, fat-to-slim conversion, eligibility,
carried-forward runs, publication counts, and the final browsing render intact. Plan the
complete publication once. Add an explicit full-reconciliation apply path that filters
writes by actual destination bytes, partitions changed whole files into byte-bounded
batches, and applies them as one rollback transaction. Route `reconcile_agent_hoods`
through it; preserve the existing targeted publication and repair entry-point contracts
and size guards.

The Rust core boundary applies to new shared policy. Open `sase-core` through the
`sase_repo` skill and
`sase repo open sase-core -r "Implement bounded agents publication policy"`; use the
returned checkout and read its instructions. Add the deterministic batch policy to a
focused Rust module under `crates/sase_core/src/`, expose it through
`crates/sase_core_py`, and add a thin adapter under `src/sase/core/`. The existing
`sidecar_publication` policy/facade demonstrates the binding conventions but is a
push-retry policy; do not overload its wire version for unrelated batch data.

The new policy takes ordered relative-path/byte-size records and a positive byte budget
and returns deterministic batches. Use checked arithmetic, reject invalid or duplicate
records, handle zero-length files, accept the exact limit, and reject one file larger
than the limit with its path and size in the diagnostic. Pass metadata across the
binding, not the entire transcript corpus. Use 128 MiB as the production batch budget,
with a small injectable budget for tests. No Python fallback or duplicated Python
batching algorithm.

The original phase-1 design's Python-only implementation note described that already
completed change; it does not override the current Rust core boundary for this new
batching policy. Keep the new binding's schema and adapter validation explicit, and fail
clearly when the installed binding is stale.

Python may retain existing filesystem/Git orchestration as host glue. Refactor the
existing apply helper's staging and backup plumbing as needed, without copying the
publication planner into Rust or rewriting unrelated sync subsystems. Required
transaction behavior:

1. Preflight every supplied path with the existing relative-path and resolved-root
   containment checks, including paths whose bytes appear unchanged. Compute the changed
   file set and validate the whole batch plan before replacing destinations.
2. Stage changed bytes and preserve originals on disk, recording which paths did not
   previously exist. Keep backups until the complete apply succeeds. Avoid an additional
   all-payload in-memory backup. Preparation failure changes no payload destinations.
3. Apply bounded batches while the caller holds the existing publication lock. A batch
   boundary is internal filesystem work, not a Git commit or push boundary. No manifest,
   snapshot, run page, or index from a partial application is published remotely. This
   preserves the current sequential-write visibility model under the lock, not a new
   claim of instantaneous filesystem-wide atomicity.
4. If a later write fails, restore originals from **all** earlier batches and remove
   files created by this transaction. Surface a rollback failure explicitly and preserve
   recovery material if cleanup cannot complete. Successful rollback and successful
   apply remove temporary staging/backup files.
5. Return the same changed/no-change meaning as the existing applier. Leave the outer
   sync transaction responsible for committing, pushing, cleanup, and retry after remote
   divergence. A retry must regenerate the entire publication against the pulled state.
   Deferred prompt restoration and outbox acknowledgement must still follow successful
   complete publication.

Do not run `plan_hoods` once per hood or per batch: that would repeatedly validate and
render the complete sidecar. This fix bounds write batches and backup memory; the
existing inventory and planned payload still occupy memory. End-to-end streaming,
retention, schema changes, new flags, and index-rendering optimization are outside this
completion task unless measurements identify a direct blocker. Do not raise or remove
snapshot, manifest, text, or targeted-publication caps.

### 2. Lock in the failure and transaction guarantees

Use small deterministic fixtures and an injected budget rather than allocating a 311 MB
test corpus. Add Rust policy tests, PyO3 binding tests, and Python adapter coverage.
Extend the existing agents-sync reconciliation, I/O, and Git transaction tests to
establish:

- A complete supplied payload over budget succeeds when most bytes are unchanged; an
  all-changed payload over budget also succeeds through multiple bounded batches. Test
  empty input, zero-length entries, exact boundaries, deterministic ordering,
  overflow/invalid input, and an individually oversized file.
- Missing or drifted files are repaired even when the snapshot digest is unchanged.
  Legacy fat entries become slim with the existing flag on; flag-off compatibility,
  unrelated owners, carried-forward runs, and manifest guards still behave as before.
- An unsafe or oversized late entry fails before any destination changes. Inject a
  failure after an earlier batch has changed an existing file and created a new one;
  verify byte-for-byte rollback, removal of newly created files, and temp cleanup.
- A full sync of multiple batches creates one complete publish transaction. A
  non-fast-forward rejection recomputes against upstream and converges. A failed batch
  produces no successful sync or premature outbox acknowledgement, and the sidecar
  cleanup covers newly created hood directories. Deferred prompt archives remain
  required before their queued requests are acknowledged.
- A second reconciliation is idempotent: correct counts and no spurious payload changes
  or commit. Existing targeted-publication tests continue to pass.

Read the current `lint_and_test.md` memory before finishing implementation. Run focused
regressions and `just check` in sase. In the opened sase-core checkout run its full
`just check` or `./scripts/check.sh`, including bindings; a core-only
`cargo test -p sase_core` is insufficient. Build/install the matching Rust binding in
the workspace environment before Python validation. Run `just fix` before handing long
checks to a monitor. `just check-full` is required before landing and must run through
the `sase_monitor` skill with its TESTING/TESTED convention. Do not manually bump core
release versions or create commits/branches/PRs; host-owned finalization handles
repository changes.

### 3. Install the verified implementation and finish athena recovery

Carry implementation and recovery across mechanical finalizer/monitor continuations if
needed; leave the bead open until installed-runtime evidence is complete. Code in an
ephemeral workspace is not proof that the global `sase` contains the fix.

After the verified code and matching core binding have landed through the normal host
workflow, inspect `sase update --dry-run`, then use the supported update path on athena.
The user's recovery request authorizes the necessary update, but respect any actual
runtime refusal or approval requirement. Do not blindly swap a shared editable checkout
while its runners depend on it. Verify the installed Python and Rust versions/locations
include this change and that `slim_agents_manifest` is on. Use a monitor for long
commands and waits; do not kill competing agents or unlink locks to obtain access.

Capture a fresh baseline with `sase agent sync --check --json -p sase`, including
quarantine/retired counts and their reasons. Inspect the agents sidecar only via
`sase repo open agents` and audited artifact reads, using the returned path rather than
hard-coded primary or sibling paths. Record the athena manifest's byte size, hood count,
remaining fat entries, and whether `sase-11l` exists. Reads of published artifacts must
use `sase artifact read`; use repository metadata for size/count measurements. Keep
private prompt/chat contents out of reports.

Run the project's supported recovery commands with project scope:

```bash
sase agent sync --repair-manifest -p sase
sase agent sync --drop-retired --retry-quarantined -p sase
sase agent sync -p sase
```

Repair comes first so recovered on-disk hoods are part of the baseline manifest. The
second command both resets queue state and performs a full reconciliation; combining the
flags avoids two unnecessary full passes. The last command proves ordinary sync works
after recovery. Inspect exit codes and structured results, not just a printed
"synchronized" label. Distinguish lock contention from format, payload, rendering, and
materialization failures. Use bounded retries/monitors for contention; persist genuine
blockers as evidence on the bead without closing it.

Verify all of the following with timestamps and relevant SHAs:

1. Installed-runtime full sync succeeds without aggregate payload or manifest errors.
   Repeating ordinary sync succeeds with no spurious republishing of unchanged data;
   concurrent new publications may legitimately change counts.
2. The athena manifest is well below the former 4 MiB ceiling (the original design
   estimated about 250 KB; record the actual value), includes `sase-11l`, and retains
   all valid previously published/recovered hoods. Explain any legacy entries that
   cannot be converted because they are outside the eligible inventory.
3. The published `agents/bbugyi200.athena.sase-11l.2/README.md` exists at the pushed
   `sase-org/sase--agents` remote revision. Verify the GitHub page returns 200 and
   spot-check `sase-11l.1` and agents in `sase-zn`/`sase-zw`. Use the opened checkout
   plus its fetched remote ref for repository content, audited artifact reads for
   recorded context, and an HTTP status-only request for URL reachability.
4. No active outbox requests fail on manifest or aggregate-payload errors; the
   materialized `sase-11l` requests are acknowledged. Account explicitly for any
   remaining quarantined, retired, or prompt-archive failures instead of claiming the
   whole queue is clean just because it was reset.
5. Observe a real post-fix single-hood publication from a normal agent commit and record
   a successful drain comfortably below the 120-second bound. Observe the host-finalized
   implementation commit if suitable, or the next natural commit; do not manufacture a
   commit solely for this check. If latency still blocks this criterion, measure where
   time is spent and resolve the direct blocker before calling the phase complete;
   unrelated improvements remain follow-up notes.

## Completion

Append a concise verification note to `sase-11o.2` covering code/core revisions, checks,
installed versions, sync outcomes, before/after manifest size and hood counts, remote
page evidence, queue disposition, and observed drain time. Then run:

```bash
sase bead epic-symbols sase-11o.2
sase bead close sase-11o.2 --note "<actual verification evidence>"
```

If symbols have appeared since planning, resolve them or re-key justified entries to a
still-open owning bead before closing. Never close an ancestor or treat implementation
tests alone as production recovery. Submit the required SASE final declaration on normal
turn endings, including both repositories when modified.
