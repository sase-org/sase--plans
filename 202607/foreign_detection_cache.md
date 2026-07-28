---
tier: tale
title: Foreign-only detection cache and no-network integration
goal: Validated foreign agent hoods are captured during explicit periodic fetches,
  integrated later with zero network or sidecar checkout mutation, and suppressed
  thereafter by durable per-hood receipts.
bead: sase-8v.7
create_time: 2026-07-24 16:01:35
status: done
---

- **PROMPT:** [202607/prompts/foreign_detection_cache.md](prompts/foreign_detection_cache.md)
- **PARENT:** [202607/global_agent_hoods.md](https://github.com/sase-org/sase--plans/blob/main/202607/global_agent_hoods.md)
- **BEAD:** [sase-8v.7](https://github.com/sase-org/sase--beads/blob/main/pages/sase-8v/sase-8v.7.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-8v.7](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-8v.7/README.md)
  - [bbugyi200.athena.sase-8v.7--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-8v.7.md#member-code)
- **COMMITS:**
  - [f76a9ed](https://github.com/sase-org/sase/commit/f76a9ede7738308fc89ca7cfe6f476e0a6598727) — feat: cache foreign agent state for offline integration (sase-8v.7)

# Foreign-only detection cache and no-network integration

## Objective

Complete phase `incoming-cache` for bead `sase-8v.7`: turn the existing periodic agents-sidecar fetch into a producer of
validated, immutable foreign-hood cache items; track durable per-source hood import receipts; expose a cached inbound
integration API that cannot perform network or mutate the sidecar checkout; and keep the existing `sync_agents()` path
as the explicit full-duplex transaction.

This is a `tale` because the work is one cohesive backend slice in the primary SASE repository. The linked `sase-core`
repository already supplies owner classification, name/hood parsing, and relationship validation/rewrite through the
Python facade, so no Rust change is needed. Phase 8 will consume the new immutable status items from ACE; this phase
must not change indicator, comprehensive-update, or Updates-tab behavior.

## Existing seams and constraints

- `src/sase/agents_sync/status.py` currently fetches the sidecar and then derives status from the mutable checkout's
  upstream/ahead/behind state. Its short revalidation still runs Git comparisons and the v1 unexported-artifact scan.
- `src/sase/agents_sync/v2_import_package.py` already performs strict whole-hood validation from a filesystem root, and
  `v2_importer.integrate_v2_hoods()` performs transactional, idempotent Phase 5 import.
- `src/sase/agents_sync/bundles.py` retains strict read-only v1 validation/import; v1 ownership remains username-unknown
  unless local evidence proves a current-owner round trip.
- `src/sase/agents_sync/git_sync.py` and commit-targeted publication share the bounded `.git/sase-agents-sync.lock`, but
  imported hoods do not yet advance a remote-digest receipt.
- Current status and sync model fields are public to CLI/TUI callers. New fields must be immutable and versioned while
  retaining the existing diagnostics until Phase 8 deliberately changes their presentation.
- Sidecar data is untrusted. Git object paths, JSON, counts, sizes, digests, UTF-8, owner/project agreement, and
  relationship graphs must be validated before anything becomes an integration candidate.

## Implementation

### 1. Define the versioned cached-update, receipt, status, and outcome contracts

- Extend `src/sase/agents_sync/models.py` with immutable records for:
  - a captured incoming hood (`project key/name`, fetched ref/SHA, cache ID, format version, source username or
    username-unknown v1 marker, machine, top hood, hood digest, run/family counts, and cache creation time);
  - one per-project import receipt keyed by exact source owner plus top hood and carrying the last applied digest,
    captured cache evidence, and application time;
  - typed cached integration results with dispositions `applied`, `already_applied`, `stale`, `missing`, `quarantined`,
    or `failed`, plus v1/v2 import counts and diagnostics.
- Bump the agents status schema and add fetched SHA/ref, pending immutable cache items, validated/exact-owner counts,
  quarantine diagnostics, and cache timing to `ProjectSyncStatus`. Provide a pure pending-foreign count/property for
  Phase 8 without changing the current widget predicate in this bead.
- Keep old ahead/behind/unexported fields as cached diagnostics for CLI/backward compatibility. Use strict exact-shape
  decoding for every persisted model and reject invalid identifiers, negative counts, malformed SHAs/digests, unsafe
  owner/hood values, and non-finite timestamps.

### 2. Add durable receipt and immutable cache storage

- Add a focused module under `src/sase/agents_sync/` for cache/receipt persistence below `${SASE_HOME}/agents_sync/`.
- Store receipts atomically as a versioned exact-shape document keyed by project, source owner kind/username/machine,
  and top hood. Receipt comparison is by validated hood digest, not Git behind count.
- Store each captured payload in an opaque digest-addressed cache directory. Build it under a private staging directory,
  validate the complete payload, write canonical metadata, and atomically rename it into place. If the content address
  already exists, verify it and reuse it; never rewrite a published cache object.
- Preserve current pending objects, the object referenced by the latest receipt, and a small bounded number of recent
  superseded generations per hood so a preview captured before a newer fetch remains usable. Prune only from worker/CLI
  paths, never short status projection, and refuse symlink/path escapes.
- For v2, cache only the manifest/snapshot and declared files required to reconstruct and revalidate one hood. For
  legacy v1, conservatively group validated manifest entries by unknown-username machine plus semantic top hood and
  cache a minimal strict manifest with its referenced bundles.

### 3. Read and validate the fetched commit without checking it out

- Add an injectable local Git-object reader that resolves the fetched upstream ref to an exact commit SHA, lists only
  owner-manifest/legacy-manifest paths at that SHA, and reads validated relative paths from that commit. These commands
  are local object reads; only the preceding explicit fetch is marked as a network operation.
- Refactor the strict v2 decoding boundary only as needed to decode an owner manifest/snapshot from captured bytes and
  reuse the existing whole-hood package validator. Do not duplicate identity, relationship, path, size, or digest rules
  outside the existing v2 validation layer.
- During an explicit/long-cadence refresh:
  - acquire the same bounded per-project agents lock used by full sync and targeted publication;
  - fetch once, resolve the exact upstream SHA/ref, and inspect data at that commit without changing `HEAD`, index, or
    worktree;
  - ignore exact-current v2 owners for pending purposes while recording observation counts;
  - treat same-user/other-machine, other-user (including the same machine token), and all username-unknown v1 groups as
    foreign;
  - skip already-receipted digests, reuse still-valid pending cache objects, atomically materialize newly validated
    candidates, and quarantine malformed owners/hoods independently so one bad source cannot suppress others.
- Preserve prior pending items if a fetch fails. A newer digest for one hood replaces that hood in the current status
  snapshot but does not immediately delete the older immutable object.

### 4. Make status refresh and short revalidation honor the three-mode contract

- Refactor `get_agents_sync_status()` so network occurs only for `refresh=True`. Plain `--check` and
  `revalidate_only=True` are cached/local by default even when the status file is absent or old.
- Long refresh may update cached ahead/behind/unexported diagnostics after fetch, but the short path must only resolve
  local configuration/targets and reconcile the persisted status items against cache existence and receipts. It must not
  fetch, enumerate manifests, scan agent artifacts, or mutate a sidecar checkout.
- Keep fetch/cache/receipt/status writes serialized with compatible project locks and atomic file replacement. Lock
  contention returns a truthful retry/skip diagnostic and never waits indefinitely.
- Make post-full-sync status rewriting network-free and receipt-aware so a successfully imported hood stops appearing
  pending even if the local sidecar remains behind or the old cache object is retained.

### 5. Implement cached inbound integration and receipt advancement

- Add and export `integrate_cached_agent_updates(captured_items)` from `sase.agents_sync`.
- Resolve only the projects named by the captured immutable items, require a current owner identity, group work by
  project, and acquire the existing agents sync lock. The function must never invoke a network-marked Git command and
  must not run pull, export, commit, push, or any sidecar worktree mutation.
- For each item under the lock:
  - return `already_applied` if its exact digest receipt is current;
  - return `stale` without importing if a newer receipt for the same source hood would otherwise be regressed;
  - return `missing` for a pruned/absent object;
  - strictly revalidate cache metadata, bytes, digests, owner/project/hood identity, and the complete v2 package or v1
    bundle group;
  - call the existing transactional v2 importer or conservative v1 importer;
  - return `quarantined` for validation/import quarantine and do not advance its receipt;
  - after successful/idempotent import, atomically write the hood receipt and return `applied`.
- After a project batch, rewrite its status from cache plus receipts without network. A crash after import but before
  receipt write must be safe: the importer is idempotent, so retry completes the receipt without duplicating artifacts.
- Update the full-duplex import pass to advance the same receipts for successfully completed v2 transactions and
  successfully validated v1 groups while already holding the shared lock. Exact-current v2 observations do not become
  foreign receipts. This ensures `sync_agents()` and targeted publication also clear cached pending state.

### 6. Add focused regression and integration coverage

- Add cache/object-reader tests using local bare remotes and real fetched commits to prove:
  - exactly one network fetch per explicit refresh and zero network calls for short revalidation/integration;
  - fetched objects are inspected without changing sidecar `HEAD`, index, or worktree;
  - exact-current v2 is ignored while same-user/other-machine and other-user/same-machine are pending;
  - multiple foreign owners in one commit are independently cached;
  - username-unknown v1 is conservatively grouped/detected;
  - corrupt manifests, paths, snapshots, referenced payloads, and digest mismatches are quarantined independently;
  - content-addressed objects are immutable and atomically published.
- Add receipt/integration tests for applied, already-applied, stale, missing/pruned, quarantined, and failed/lock-busy
  outcomes; v1 and v2 idempotence; receipt clearing; and full sync receipt advancement.
- Add a captured-SHA regression: capture digest A, fetch newer digest B, integrate the still-retained A object with
  every network operation configured to fail, then verify B remains the pending status item.
- Add concurrency coverage holding the existing sync lock while cached integration/detection attempts run, proving
  bounded typed failure and no partial cache, receipt, or import mutation.
- Update existing status/CLI tests for the new schema and explicit-refresh-only network behavior. Retain compatibility
  coverage for ahead/behind/unexported rendering until Phase 8 replaces it.

## Validation

1. Run `just install` before repository checks, as required for ephemeral workspaces.
2. Run focused tests first:

   ```bash
   .venv/bin/python -m pytest \
     tests/agents_sync/test_incoming_cache.py \
     tests/agents_sync/test_status.py \
     tests/agents_sync/test_git_sync.py \
     tests/agents_sync/test_v2_import_package.py \
     tests/agents_sync/test_v2_importer.py \
     tests/agents_sync/test_bundles.py \
     tests/agents_sync/test_cli.py -q
   ```

3. Run `just check` and fix every lint, type, unit, integration, and snapshot failure.
4. Re-run the focused incoming-cache/status tests after the full check to ensure no broad-suite isolation or cache-state
   interaction was masked.
5. Inspect `git diff --check` and the final diff. Confirm no SASE memory/generated instruction file, linked repository,
   TUI behavior, parent epic status, or unrelated bead was changed.

## Expected files

- `src/sase/agents_sync/models.py`
- `src/sase/agents_sync/status.py`
- `src/sase/agents_sync/git_sync.py`
- `src/sase/agents_sync/v2_io.py` and/or `v2_import_package.py` for reusable byte/object validation only
- new focused cache/receipt and Git-object modules under `src/sase/agents_sync/`
- `src/sase/agents_sync/__init__.py`
- `tests/agents_sync/test_incoming_cache.py`
- existing focused agents-sync status/full-sync/import/CLI tests as required

## Out of scope

- Do not change ACE indicator/click behavior, comprehensive update execution, or add the Updates-tab `a` action; those
  belong to dependent bead `sase-8v.8`.
- Do not change Rust identity semantics or bindings unless implementation proves an existing required Phase 1 API is
  actually absent.
- Do not edit SASE memory files, generated agent instruction/provider shim files, or linked repositories.
- Do not close parent epic `sase-8v`, create beads, invent tombstones, delete legacy v1 payloads, or republish imported
  foreign data.
