---
tier: tale
size: medium
title: Finish athena publication recovery and close sase-11o.2
goal:
  Full agent sync handles a publication larger than 128 MiB, athena backfills its
  missing agent pages with a slim owner manifest, and sase-11o.2 closes with verified
  recovery evidence.
proposed_by: bbugyi200.athena.sase-11o.2.f0
create_time: 2026-09-16 13:14:57
status: wip
---

# Finish athena publication recovery

## Scope and ownership

Complete the already assigned, in-progress phase **sase-11o.2**. Its original design is
`plan:202609/agents_sync_manifest_slim.md`, phase `athena-recovery-backfill`; read it
with `sase artifact read`. Phase sase-11o.1 is closed and its slim-manifest change is
present in this checkout. Read `sase bead show sase-11o.2` again before implementing
because other agents and the installed runtime can change while this plan is reviewed.

This is a medium tale: one implementer can add the small deterministic batch policy,
wire it to the existing filesystem transaction, test it, and perform recovery. It does
not need separately staffed feature phases. This plan extends the recovery phase to fix
the concrete blocker discovered by its prior worker.

Do not manually set the phase status, create follow-up beads, close sase-11o, close
sase-10x, or close any ancestor plan. Record unrelated discoveries as
`sase bead note sase-11o.2 'PROPOSED FOLLOW-UP: ...'`. Keep the phase open until the
operational acceptance criteria below pass. Host finalizers own source commits,
branches, and PRs; operational `sase agent sync` owns its normal sidecar commits and
pushes.

## Evidence and remaining failure

The prior worker installed phase 1 on athena, dropped 71 retired requests, reset more
than 646 quarantined requests, and repaired 47 missing manifest entries. Its recorded
sidecar state had 2,091 hood directories and approximately 311.5 MB of agent payload.
The `sase-11l` hood was absent entirely. Treat these as historical measurements, not
current acceptance evidence. During planning, `sase agent sync --check -p sase --json`
reported cached diagnostics unavailable and 665 quarantine diagnostics; obtain fresh
evidence during recovery.

The failure is distinct from the repaired manifest limit:

- `src/sase/agents_sync/publication.py:reconcile_agent_hoods` sends every eligible hood
  to `_publish_hoods`.
- `publication_planning.py:plan_hoods` constructs each hood's run files and snapshot,
  merges them into one dictionary even when the snapshot digest is unchanged, then adds
  the manifest and all rendered browsing pages.
- `v2_io.py:apply_payload_atomic` rejects the entire dictionary above
  `MAX_PAYLOAD_BYTES` (128 MiB) **before** comparing existing files. Thus even an
  unchanged large sidecar cannot complete a full reconcile.
- `rendering.py:render_browsing_payload` renders all owners' pages. Merely publishing
  one hood at a time repeats this work and still includes global rendering overhead;
  skipping unchanged snapshots alone misses page repairs.
- `git_sync_transaction.py` already owns the sidecar lock, pull, one Git commit, push,
  and non-fast-forward recomputation. Its retry drops exactly its one rejected commit.
  Introducing one Git commit per batch would break that assumption and enlarge this task
  unnecessarily.

`sase bead epic-symbols sase-11o.2` reported no entries during planning. Repeat that
check immediately before closure.

## Implementation

### 1. Add a deterministic byte-batch policy in Rust

Open `sase-core` using `/sase_repo` and
`sase repo open sase-core -r "Implement bounded agents publication planning"`; use only
its returned checkout path. Read its AGENTS.md. Current project instructions require new
shared backend policy in `crates/sase_core`, exposed by `crates/sase_core_py`; the older
parent plan's assertion that all publication changes should be Python is not the
boundary for this new work.

Add a narrowly scoped batch planner, preferably a separate publication-payload module
rather than expanding the existing push-retry policy. Accept unique relative
path/byte-length records and a positive byte budget. Return a versioned, deterministic
ordered list of batches, each no larger than the budget, with each input path present
exactly once. Include zero-byte files; empty input returns no batches. Reject duplicate
paths, invalid lengths or budget, arithmetic overflow, and any single file larger than
the budget with a useful path/size diagnostic. Use checked integer arithmetic.

The binding transports metadata only, not the hundreds of megabytes of file content. Add
the Rust export, PyO3 callable and binding-contract tests, then a thin typed Python
facade using `require_rust_binding`. Missing/stale bindings must fail explicitly; do not
add a Python algorithm fallback. Preserve the existing push-retry wire rather than
unnecessarily bumping unrelated APIs.

### 2. Apply bounded batches with rollback across the whole publication

Keep `apply_payload_atomic`'s existing cap and public contract for a single batch. Add a
publication-level filesystem adapter and call it from `publication.py:_publish_hoods`,
covering both full reconcile and targeted publication. Continue to run `plan_hoods` once
per publication pass so manifest construction, scoped integrity verification, rendering,
counts and diagnostics retain their existing semantics.

The adapter must:

1. Validate every supplied relative path and resolved destination against the repository
   root before any destination write, including unchanged files. Preserve
   traversal/symlink escape protection and pre-write manifest caps.
2. Compare planned bytes with actual on-disk bytes. Batch only changed or missing files,
   including rendered pages and manifest-only fat-to-slim changes. Never use snapshot
   digest equality alone to suppress repairs.
3. Ask the Rust planner to partition those actual writes using the unchanged
   `MAX_PAYLOAD_BYTES` budget. Preflight every batch before applying the first. Raising
   the limit, removing it, and filtering alone are insufficient fixes.
4. Preserve the existing all-or-restored behavior on ordinary exceptions over the
   **entire publication**, not just the last batch. Use transaction-scoped disk-backed
   original-byte backups and an absent-file ledger, then apply each bounded batch
   through the existing primitive. On a later write failure, restore prior bytes and
   remove files created by all earlier batches. Remove newly created empty directories
   when safe, without touching unrelated files. Do not rely solely on Git cleanup:
   direct publication callers also need rollback, and untracked hood files must not
   survive a failed publication.
5. Clean temporary staging/backup data after success or successful rollback. If rollback
   itself fails, report both errors and retain usable recovery backups; never erase the
   only copies of original bytes in a `finally` block. These are the existing
   exception-safety guarantees extended across batches, not a new crash-atomic
   filesystem or concurrency model.
6. Return an unchanged result without writes for a byte-identical payload. Preserve
   publication count meanings, diagnostics and slim flag behavior.

Keep all batches within the existing sidecar lock and one logical Git transaction; no
commit/push/outbox acknowledgment between batches. Do not add a primary-commit
dependency on publication. A successful full sync still restores deferred prompt
archives before acknowledging materialized requests. Failures must leave requests
retryable through the existing rules.

This fix bounds each apply operation and avoids writing unchanged content. The existing
all-hood plan/render dictionary remains in memory; eliminating that dictionary or making
index rendering incremental is outside this bounded repair. Avoid retaining an
additional in-memory copy of the entire rollback payload. Do not add new CLI options,
change the on-disk schema, prune hoods, raise limits, change the 120-second drain
timeout, or retire the existing flag.

### 3. Prove the failure and transaction guarantees

Use small byte budgets in fixtures instead of allocating production-size payloads. Add
meaningful tests alongside the existing publication, v2 I/O and Git sync suites:

- Rust and real PyO3 tests: deterministic partitioning independent of input order,
  exact-boundary sizes, zero-byte and empty input, duplicate paths, invalid metadata,
  arithmetic overflow, and a single oversized file.
- A complete planned payload exceeding the budget whose disk content is unchanged
  succeeds without writes. A small change to such a publication writes only that change.
- A clean sidecar with aggregate **changed** bytes larger than the budget reconciles
  successfully through multiple bounded writes. Assert all hoods, snapshots, manifest
  entries, run pages and counts, then assert rerun idempotence. This prevents a
  changed-file-only workaround from passing.
- Digest-unchanged hoods still repair missing/stale rendered pages, and legacy fat
  entries still slim even when their snapshot bytes are unchanged. Include global
  browsing pages in byte accounting.
- Inject a write failure in a later batch after earlier batches changed an existing file
  and created a new hood. All original bytes return, newly created files disappear, no
  manifest references a partial publication, and retry succeeds. Also cover rollback
  failure preserving recovery evidence.
- A malicious path or oversized single changed file late in the input fails before the
  first destination write. Keep the single-batch cap regression.
- A real temporary Git sidecar full-sync test crosses several batches, then exercises
  the existing non-fast-forward retry without extra per-batch commits, duplicate counts,
  or loss of another owner's publication. Failed publication must not acknowledge
  pending outbox work; deferred prompt restoration remains covered by its existing
  tests.
- Targeted publication still uses the same validation and preserves the phase-1 scoped
  run-file verification behavior.

Build the local Rust binding with the existing `just rust-install`/`just install`
workflow pointed at the opened core checkout. Run core `just check` or
`./scripts/check.sh`, including PyO3 tests with Python >=3.12; testing only
`cargo test -p sase_core` is insufficient. Run focused Python regressions and
`just check` in sase. Follow `lint_and_test.md`: run `just fix` before any verification
monitor, and run `just check-full` through `/sase_monitor` before landing as required by
the parent epic. Leave source commits to `/sase_final`.

Core release-plz owns crate versions; do not hand-bump them. Coordinate the
binding-first release and the Python minimum dependency using the repository's existing
release/floor workflow. Local binding tests alone do not prove the installed athena
runtime contains the new API.

For an intermediate verified source handoff, follow `/sase_final`'s prepared
monitor-completion procedure: obtain the host context, prepare commit decisions for both
changed repositories with `bead_action: keep`, and bind the returned completion
reference explicitly to the verification monitor. Preserve the remaining recovery work
and original phase ID in its checkpoint/follow-up policy. A verify profile alone does
not authorize host completion. Use the documented monitor completion/continuation
contract; do not wait for a release of source changes that are still uncommitted in the
worker's checkout.

## Operational recovery on athena

After the fix is tested and available through the supported installed runtime, complete
the following in order. Use the normal host finalization and monitor continuation
mechanisms if publication of the source/binding or a long command requires another turn.
Keep the phase open through that continuation; do not substitute a promise to resume for
a mechanical handoff. Never replace the global installation with an ephemeral workspace
checkout merely to bypass release ordering.

1. Confirm this is athena. Preview `sase update --dry-run` if an update is needed, then
   use supported `sase update` to pick up the Python fix and the matching core binding.
   Verify actual imported module/binding capabilities in the installed environment as
   well as `slim_agents_manifest` being ON. Account for the updater's active-runner
   safeguards; do not terminate other agents or bypass live process locks to create an
   idle window.
2. Capture fresh outbox counts and failure categories for `gh_sase-org__sase`. `--check`
   can show cached diagnostics and is not proof of recovery. Use existing read-only
   outbox APIs if more detailed live counts are needed. Open the agents sidecar through
   `/sase_repo` before inspecting its tree; read artifact bodies through
   `sase artifact read`, not raw sidecar reads. Record the manifest byte size, hood
   count and relevant missing-page state.
3. Re-run `sase agent sync --drop-retired -p sase` only if retired items remain, and
   `sase agent sync --retry-quarantined -p sase` if quarantined items remain. Both
   options also execute a normal full sync, so they must use the repaired
   implementation. Save their outcomes rather than interpreting a reset as a successful
   drain. Run `sase agent sync --repair-manifest -p sase` if manifest omissions remain;
   it is safe to repeat the supported repair.
4. Run a normal full `sase agent sync -p sase --json`. Use `/sase_monitor` for
   long-running sync or any wait/backoff and ensure its follow-up retains this plan and
   phase ID. A busy-lock skip is not success. Retry through the supported bounded lock
   path; do not delete active locks. Resolve a concrete new validation failure from its
   evidence rather than applying unrelated digest repairs or discarding data blindly.
5. Verify the pushed sidecar revision and fetch/read its remote-tracking state through
   the opened repository. The manifest for `bbugyi200.athena` must be well below 4 MiB
   (roughly hundreds of KB rather than the previous 4+ MB), contain `sase-11l`, retain
   the other owners and recovered hoods, and omit `files` on republished entries. Record
   exact byte and hood counts.
6. Confirm the actual GitHub page
   `https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-11l.2/README.md`
   is available and its file exists at the pushed remote revision. Spot-check
   `sase-11l.1` and the `sase-zn`/`sase-zw` pages from the parent design. Use
   authenticated Git/CLI access for a private repository; an unauthenticated 404 is not
   a reliable absence test. HTTP availability checks need not fetch repository file
   contents; artifact reads must remain audited.
7. Inspect fresh outbox state after successful sync. No active or quarantined request
   may still be failing on owner-manifest or aggregate payload byte limits. Record
   remaining counts and distinguish historical errors and unrelated concurrent lock
   contention from a new publication failure.
8. Observe a real post-fix single-hood publication for a commit-bearing agent,
   preferably a natural host-finalized commit from this repair or another active agent.
   Record the hood, primary revision, published page, outbox acknowledgment and elapsed
   time well below 120 seconds. Do not create a dummy primary commit to manufacture this
   evidence. A timed supported targeted drain of an existing eligible request is also
   suitable. A busy skip, clearing quarantine, or the full-sync runtime is not this
   timing proof.
9. Run a second normal sync to establish that recovery is repeatable and does not
   encounter the aggregate cap again. Account for concurrent genuine new work when
   interpreting counts; require no spurious rewrite for unchanged fixtures in tests
   rather than demanding a globally idle production host.

## Acceptance and closure

The source fix and both repositories' required checks must pass; the installed runtime
must have the compatible implementation; full sync and its repeat must succeed without a
busy skip or aggregate-size error; the remote missing pages and slim manifest must be
verified; relevant outbox failures must be cleared; and a real single-hood publish must
meet the original timing criterion.

Append a concise evidence note to sase-11o.2 with source/binding versions, check
results, successful sync outputs, pushed sidecar revision, manifest measurements, page
evidence, outbox categories and targeted timing. State explicitly that this resolves the
earlier aggregate-payload blocker.

Immediately before closure, run:

```sh
sase bead epic-symbols sase-11o.2
```

Resolve any entries or re-key still-needed Justfile exemptions to the still-open parent
epic with a reason and rerun affected checks. Then close **only** the original phase:

```sh
sase bead close sase-11o.2 --note "<concrete verification evidence from this run>"
```

Leave sase-11o, sase-10x and all other beads to their owners. Finish through
`/sase_final`, declaring both source repositories if modified. If a release or verified
operational failure still blocks any acceptance criterion, record the exact evidence and
use the appropriate mechanical continuation rather than closing the bead prematurely.
