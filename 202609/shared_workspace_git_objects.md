---
tier: tale
title: Share Git objects across managed workspace checkouts
goal: Managed checkouts persistently share primary Git objects, compact safely, and
  recover stale dependencies.
size: medium
proposed_by: bbugyi200.athena.sase-zw.6
bead: sase-zw.6
status: done
---

- **PARENT:**
  [202609/bound_sase_disk_footprint.md](https://github.com/sase-org/sase--plans/blob/main/202609/bound_sase_disk_footprint.md)
- **BEAD:**
  [sase-zw.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-zw/sase-zw.6.md)

# Plan: Share Git objects across managed workspace checkouts

## Goal

Implement phase `sase-zw.6` so numbered managed checkouts borrow the primary checkout's
Git object database persistently, existing safe checkouts can be compacted, and
`sase workspace repair` diagnoses and repairs a stale object-store dependency. The
implementation must preserve checkout correctness, the existing clone/origin/fetch
healing behavior, and the rule that a claimed, occupied, or dirty checkout is never
compacted.

## Mechanism and safety decision

Use Git's persistent alternates mechanism, created for new local clones with
`git clone --shared` and represented by the checkout's resolved
`objects/info/alternates` file. An alternate points at the primary checkout's resolved
object directory, so it survives either repository repacking. Reject clone-time
hardlinks because the next repack replaces them with private packs, and reject
`--reference-if-able ... --dissociate` because dissociation deliberately copies the
borrowed objects and therefore preserves correctness by giving up the disk saving.

Protect the dependency explicitly:

- Before creating or retrofitting a borrower, set the primary repository's local
  `gc.pruneExpire` to `never`, so ordinary automatic/manual GC cannot prune objects that
  remain reachable only through a managed checkout.
- Mark SASE-managed sharing in local Git config and disable automatic borrower
  maintenance (`gc.auto=0`, `maintenance.auto=false`) so routine commands do not
  silently copy the alternate's objects back into a private pack. Explicit user Git
  commands remain possible and repair/compact can restore the intended state.
- Resolve object paths with `git rev-parse --git-path objects` instead of assuming a
  literal `.git/objects` layout, canonicalize the primary path, and write alternates
  atomically with a trailing newline.
- For retrofit, use `git repack -a -d -l`: `--local` excludes objects available from the
  alternate and `-d` removes superseded private packs. The planning probe against a
  standalone local clone reduced its object directory from 3096 KiB to 12 KiB and still
  passed `git fsck --connectivity-only`.
- Treat deletion or relocation of the primary as an explicit dependency break. Repair
  rewrites a stale alternate to the current configured primary and verifies
  connectivity. If sharing has been disabled, repair first points at the current primary
  as needed, performs a non-local `git repack -a -d` to copy all reachable objects,
  removes the alternate, and verifies the checkout standalone. If required objects are
  no longer available, fail with an actionable error and leave the dependency visible;
  do not claim a recovery Git cannot prove.

## Implementation

1. Add a focused workspace-provider object-sharing module and expose only the helpers
   needed by checkout materialization and maintenance. It should:
   - inspect and classify alternate state (absent, expected, stale/broken, unexpected);
   - configure the primary and borrower protections above;
   - install/repoint/remove the SASE-owned alternate safely;
   - measure local object bytes before/after without following symlinks;
   - run Git through the existing non-interactive environment and lock-retry
     conventions;
   - provide compact and repair operations whose success is contingent on
     `git fsck --connectivity-only`.

2. Add `workspace.share_git_objects` (boolean, default `true`) to `WorkspaceStore`,
   `src/sase/default_config.yml`, and `src/sase/config/sase.schema.json`. Pass that
   resolved choice from `ensure_workspace_checkout()` into `ensure_git_clone_at()`
   rather than making every incidental local clone share objects. When enabled, new
   numbered workspace clones use `--shared`; configure the primary before cloning and
   the borrower immediately after cloning, while retaining the existing target healing,
   canonical origin rewrite, non-interactive fetch, and clone-failure recovery. When
   disabled, retain today's standalone clone behavior. Add store/config-schema and real
   Git integration tests for both modes, including fetch plus repack followed by
   connectivity checks.

3. Add `sase workspace compact` to the sorted workspace parser/dispatch surface with
   `-n/--dry-run` and `-p/--project`. The command operates on registry-owned numbered
   checkouts only and reports every compacted or skipped checkout and the measured
   before/after bytes. Dry-run performs classification and reports intended work but
   changes neither Git config nor objects.

4. Make the compact eligibility check fail closed and race-resistant. Skip the primary,
   missing/non-Git checkouts, every workspace number present in the RUNNING claims, any
   checkout with a live `.sase/occupant.json` PID, and any checkout for which
   `git status --porcelain` is non-empty or cannot be read. On apply, take the existing
   ProjectSpec `patch_lock`, repeat all claim/occupant/dirty checks under that lock,
   then install the alternate, run the local-only repack, prune packed duplicates if
   still needed, and verify connectivity before reporting success. Continue across
   independently skipped entries but return nonzero for an attempted compaction that
   fails.

5. Extend `sase workspace repair` beyond registry/filesystem reconciliation. For an
   existing checkout with an alternate that no longer resolves to the configured primary
   object store, preview or apply a repoint and require connectivity to pass. When
   `workspace.share_git_objects` is false, preview or apply safe dissociation of
   SASE-managed borrowers as described above. Repair must never silently ignore a broken
   alternate or remove it before objects have been copied locally. Preserve the
   command's existing missing-entry and live-claim rematerialization behavior, and make
   repair return nonzero with per-checkout diagnostics when recovery cannot be proven.

6. Add unit and real-repository integration coverage under `tests/workspace_provider/`
   and `tests/main/` for:
   - fresh shared clones, disabled sharing, origin rewrite/fetch compatibility, and
     post-repack connectivity;
   - compact dry-run immutability and apply-time reclamation;
   - skip behavior for raw claims, live occupant records, dirty trees, missing/corrupt
     repositories, and an eligibility change detected by the locked recheck;
   - repair of a moved primary, an irrecoverable missing primary/object, and clean
     dissociation when sharing is disabled;
   - parser help/ordering and the short aliases for every public long option.

7. Document the config choice, dependency/lifetime contract, opt-out/dissociation path,
   and compact/repair workflows in `docs/workspace.md`, `docs/configuration.md`, and the
   workspace command tables in `docs/cli.md`. Keep the default config comments and
   public schema descriptions aligned.

## Verification and bead completion

1. Run focused workspace-provider, CLI, and config-schema tests while iterating.
2. Read the required `lint_and_test.md` reference memory after tracked edits, run its
   prescribed fast checks, then run `just check-full` through `/sase_monitor` as
   required for a workspace-provider change.
3. On an unclaimed, unoccupied, clean real managed checkout, first run
   `sase workspace compact -n`, record its local pack/object size and connectivity,
   apply `sase workspace compact`, then verify and record the after-size,
   `git fsck --connectivity-only`, `git log`, `git status`, a branch checkout and
   return, `git fetch`, and `just install`. Do not use the current or any claimed
   checkout for this validation.
4. Record on `sase-zw.6` the alternates choice, rejected alternatives, actual reclaimed
   bytes, and the real-checkout command results. Record any out-of-scope discovery only
   as a `PROPOSED FOLLOW-UP:` phase note.
5. Immediately before closing, run `sase bead epic-symbols sase-zw.6` and resolve or
   re-key every remaining symbol. Close only `sase-zw.6` with a concise verification
   note; never close `sase-zw` or another ancestor.
