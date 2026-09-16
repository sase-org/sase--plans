---
tier: epic
title: Restore agents-sync hood publication by slimming the owner manifest
goal: 'New agent hoods publish to the sase--agents sidecar again on athena (including
  the backfilled sase-11l hood and its sase-11l.2 run page), single-hood publishes
  finish well inside the drain timeout, and the owner manifest format no longer has
  an imminent byte or hood-count ceiling.

  '
phases:
- id: slim-manifest-fix
  title: Slim owner manifest, raise manifest caps, scope validation
  depends_on: []
  size: medium
  description: 'slim-manifest-fix: make per-hood manifest `files` optional on read
    and omitted on write behind a new slim_agents_manifest sunset flag, add dedicated
    (larger) manifest byte and hood-count read caps plus a pre-write size guard, scope
    run-file re-hashing to the hoods being written, and lock old-reader lenient-skip
    compatibility in with tests.'
- id: athena-recovery-backfill
  title: Recover the athena outbox and backfill unpublished hoods
  depends_on:
  - slim-manifest-fix
  size: small
  description: 'athena-recovery-backfill: with the fixed sase installed on athena,
    drop retired and retry quarantined publication outbox items, repair the owner
    manifest''s missing on-disk hoods, run a full reconcile sync, and verify the sase-11l.2
    agent page is live on GitHub and the manifest shrank.'
proposed_by: bbugyi200.athena.0lt
create_time: 2026-09-16 09:17:26
status: wip
bead_id: sase-11o
---

- **PROMPT:** [prompts/202609/agents_sync_manifest_slim.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/agents_sync_manifest_slim.md)
- **BEAD:** [sase-11o](https://github.com/sase-org/sase--beads/blob/main/pages/sase-11o/README.md)

# Restore agents-sync hood publication: slim the owner manifest and scope validation

## Problem

The published page for the `sase-11l.2` sase agent
(`https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-11l.2/README.md`)
returns a 404 even though the agent committed. The agent-hood publication pipeline for
owner `bbugyi200.athena` on project `sase` (`gh_sase-org__sase`) is permanently bricked
for every NEW hood, and has been intermittently failing for weeks.

Existing task bead **`sase-10x`** ("Agent-hood publication hard-stuck: owner manifest
exceeds byte limit for sase-zn/sase-zw") tracks this same defect and carries a +1 with
this diagnosis; this epic is its fix, and the land agent should close `sase-10x` on
phase-2 evidence.

## Root cause (fully diagnosed; do not re-derive)

The v2 agents sidecar keeps ONE `manifest.json` per owner machine
(`users/<user>/machines/<machine>/manifest.json` in the `sase--agents` sidecar) that
embeds every hood's full published-file list (`V2OwnerHoodEntry.files`). This grows
without bound and has now hit hard caps:

1. The athena/sase owner manifest holds **2043 hoods and 71,899 file paths = 4,193,856
   bytes — 448 bytes under the 4 MiB `MAX_JSON_BYTES` read cap**
   (`src/sase/agents_sync/v2_validation.py:23`, enforced by `read_json`, used by
   `read_owner_manifest` at `src/sase/agents_sync/v2_manifest_io.py:68`).
2. The cap is enforced on **read but not write**. Publishing any NEW hood writes a
   manifest over the cap; the post-write re-read in `_prepare_publications`
   (`src/sase/agents_sync/commit_publication_transaction.py:258-267`) — or the next
   attempt's `read_owner_manifest` in `plan_hoods`
   (`src/sase/agents_sync/publication_planning.py:56`) — then raises
   `AgentsSyncFormatError("owner manifest exceeds the byte limit")`. Because
   `AgentsSyncFormatError` nominates a `terminal_reason`
   (`commit_publication_transaction.py:244-255`), the outbox item is retired
   permanently. That is exactly what happened to `sase-11l.2` (see item with
   `terminal_reason: "could not publish agent hood 'sase-11l': owner manifest exceeds the byte limit"`
   in `~/.sase/projects/gh_sase-org__sase/agents-publication-outbox.json`). Worktree
   cleanup then reverts the manifest to just-under-cap, so the failure repeats forever.
3. Aggravator (timeouts): `load_validated_publication`
   (`src/sase/agents_sync/publication_validation.py:45-114`) re-reads ALL owner
   manifests, re-reads all ~2043 hood snapshots, and **re-hashes every one of the
   ~71,899 referenced payload files on every single-hood publish** (`verify_run_files`
   at line 100). On athena this exceeds the 120s drain bound
   (`DEFAULT_PUBLICATION_DRAIN_TIMEOUT_SECONDS` in
   `src/sase/agents_sync/commit_publication.py`); 215 outbox items failed with
   "agent-hood publication did not complete within 120s" and were quarantined —
   including `sase-11l.2`'s earlier requests.
4. Aggravator (count cap): manifest decode rejects more than `MAX_CONTAINERS` (2048)
   hoods (`src/sase/agents_sync/v2_manifest_io.py:133-134`). The manifest holds 2043 and
   there are 2091 hood directories on disk (48 unreferenced, left behind by failed
   transactions whose tracked-file cleanup does not remove new untracked hood dirs), so
   `sase agent sync --repair-manifest` would immediately trip "owner manifest has too
   many hoods", and organic growth hits the cap within ~5 new hoods regardless of the
   byte fix.
5. Outbox state today: 719 items, 646 quarantined, 71 terminal, for this one project.

**Key fact that makes the fix cheap:** `V2OwnerHoodEntry.files` has exactly ONE consumer
— the equality cross-check at `src/sase/agents_sync/publication_validation.py:95` — and
the expected value is derived on the previous line (line 94) from the hood's own
`snapshot.json`, which is already digest-pinned by `entry.digest`. The manifest `files`
lists (~90% of the manifest's bytes) are pure denormalization and can be dropped with no
loss of integrity.

## Constraints (why the fix is shaped this way)

- **Never block agent commits.** Publication must remain an auxiliary durable-outbox
  drain; nothing here may run on, or add failure modes to, the primary-commit path.
- **No cross-machine hard breakage.** Owners `bbugyi200.apollo` and
  `bbugyi200.kellys_mbp` publish to the same sidecar and will run an older sase until
  they update. Audit result: every _strict_ manifest read (`read_owner_manifest`) is
  same-owner only; all _cross-owner_ reads go through `read_all_owner_manifests_lenient`
  (`v2_manifest_io.py:87-120`), which skips an undecodable manifest with a diagnostic.
  Therefore a new manifest format is safe iff OLD readers fail to decode it (lenient
  skip) rather than decode it and then hard-fail validation. Concretely: omitting the
  `files` key makes old `compatible_object` decode fail → lenient skip → other machines'
  publishes keep working (their index-page renders transiently omit athena's hoods until
  they update — acceptable and self-healing). Writing `"files": []` instead would decode
  on old readers and then hard-fail their whole publish at the line-95 cross-check —
  never do that.
- **No merge conflicts.** All writes stay inside the existing pull-rebase/push
  publication transaction; manifests are owner-sharded so machines never write each
  other's manifest.
- **Backward-compatible branches need a sunset feature flag** (per the sase_flags
  memory): the legacy fat-manifest writer is the flag's Off branch.

## Phase 1 — `slim-manifest-fix` (size: medium)

All code in `src/sase/agents_sync/` (pure Python; this module does not touch
`sase_core_rs`, so no Rust-core change is needed).

1. **Tolerant reader.** In `decode_owner_manifest`
   (`src/sase/agents_sync/v2_manifest_io.py:123-162`): make the per-hood `files` key
   optional. Entries WITH `files` keep today's validation (string list, sorted-unique,
   `MAX_FILES`); entries WITHOUT decode to an "unrecorded file set" state (e.g.
   `files=None` — adjust `V2OwnerHoodEntry` in `src/sase/agents_sync/v2_models.py:272`
   accordingly, keeping `to_json_dict` able to emit both shapes).
2. **Dedicated manifest read caps.** Replace the manifest read path's use of
   `MAX_JSON_BYTES` and `MAX_CONTAINERS` with new dedicated constants in
   `v2_validation.py`, e.g. `MAX_MANIFEST_JSON_BYTES = 16 MiB` and
   `MAX_MANIFEST_HOODS = 16384` (slim entries are ~100 bytes, so 16384 hoods ≈ 2 MB).
   This keeps today's 4.19 MB fat manifest readable (so it can be slimmed by the next
   write) and lets `--repair-manifest` re-adopt all 2091 on-disk hoods. Do not change
   the caps for snapshots or other payloads.
3. **Validation derives file sets.** In `load_validated_publication`
   (`src/sase/agents_sync/publication_validation.py:94-98`), when an entry has no
   recorded `files`, the derived `hood_file_set(snapshot)` is authoritative and the
   equality check is skipped; when `files` is present (legacy manifests, including other
   owners'), keep the check exactly as today. Update `repair_owner_hood_digests` and
   `_recovered_hood_entry` (`src/sase/agents_sync/publication_repair.py`) to build
   entries consistent with the flag state below.
4. **Slim writer behind a sunset flag.** Create the flag with
   `sase flag new slim_agents_manifest -k sunset ...` (author `--when-enabled`,
   `--when-disabled`, and `--remove-when`; suggested remove-when: every owner manifest
   in every agents sidecar has been rewritten without per-hood `files` and all owner
   machines run a sase whose reader tolerates slim manifests). Follow the command's
   printed registry-entry and both-states-test instructions. Flag ON (default for
   `sunset`): manifest writes (`plan_hoods` at
   `src/sase/agents_sync/publication_planning.py:109-110` and both repair writers) omit
   per-hood `files`. Flag OFF: byte-for-byte legacy fat writer. The slim shape MUST be
   one that old readers fail to decode (lenient skip) — omitting the `files` key
   achieves this; a dedicated manifest schema marker on top is the implementer's choice,
   but old-reader lenient-skip behavior must be locked in by a test that runs the OLD
   decode rules against a slim manifest.
5. **Scoped run-file verification.** In `load_validated_publication`, call
   `verify_run_files` only for hoods actually being written in this publication (the
   `override_snapshots` keys), not for every hood of every owner. Keep the cheap
   per-hood snapshot digest checks for all hoods so rendering inputs stay digest-pinned.
   This removes ~72k file reads + SHA256 per publish (the 120s-timeout driver) and stops
   one drifted file in an unrelated hood from blocking every publish. Update the
   now-stale module docstring in `src/sase/agents_sync/publication_repair.py:1-11` which
   documents the old whole-owner blocking behavior.
6. **Write guard.** Before `apply_payload_atomic`, fail the publication with a clear
   diagnostic if a manifest about to be written exceeds its read cap (defense in depth;
   unreachable once slim, but converts any future recurrence from a post-write brick
   into a pre-write error).
7. **Tests** (extend `tests/agents_sync/`): decode of fat and slim entries; old-reader
   lenient-skip of a slim manifest (other-owner publish keeps working); new manifest
   caps; scoped verification (a drifted file in an unpublished hood no longer fails an
   unrelated publish, and still fails a publish of its own hood); write guard; flag ON
   and OFF states for writer + repair paths. Run `just check` while iterating;
   `just check-full` must pass before landing (two-speed verification).

## Phase 2 — `athena-recovery-backfill` (size: small, depends on: slim-manifest-fix)

Operational recovery on athena after phase 1 has landed on master AND the installed
`sase` on athena includes it (verify first — e.g. `slim_agents_manifest` appears in the
flag registry of the installed sase; do NOT run recovery with a pre-fix sase).

1. For project `sase`: `sase agent sync --drop-retired`, then
   `sase agent sync --retry-quarantined` (clears the 71 terminal and 646 quarantined
   outbox items' blockage; retry restores quarantined items with a fresh budget).
2. `sase agent sync --repair-manifest` — re-adopts the ~48 on-disk hoods the manifest
   omits (works now that the hood cap is raised).
3. Full `sase agent sync` — `reconcile_agent_hoods` republishes every locally owned hood
   with a primary commit, which backfills `sase-11l` (its hood directory does not exist
   in the sidecar at all; local agent state for it is present on athena), and drains the
   requeued outbox.
4. Verify success, in order:
   - `users/bbugyi200/machines/athena/manifest.json` in the agents sidecar clone
     (`~/.sase/projects/gh_sase-org__sase/repos/agents/`) is now far below 4 MiB
     (expected ~250 KB) and contains a `sase-11l` hood entry;
   - the publication outbox for `gh_sase-org__sase` has no active items failing on
     manifest errors;
   - `https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-11l.2/README.md`
     returns 200 (also spot-check `sase-11l.1` and pages for the `sase-zn` and `sase-zw`
     hoods named by bead `sase-10x`);
   - a fresh single-hood publish (any new agent commit on the sase project) drains well
     inside the 120s bound.
5. Known transient: until apollo and the mac update their installed sase, their
   publishes lenient-skip athena's slim manifest, so index pages they render omit
   athena's hoods; this self-heals once they update. Do not "fix" it by reverting the
   manifest format.

## Out of scope (do not implement here)

- Retention/pruning of dead hoods: growth is still unbounded, just with a far higher
  ceiling (~16k slim hoods). Tracked separately as task bead `sase-11n`.
- Index-page rendering cost, which remains O(total hoods) per publish.
