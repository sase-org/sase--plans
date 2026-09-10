---
tier: tale
title: Retire the obsolete Obsidian Sync restoration epic
goal:
  Retire bob-cli-1l safely as superseded by bob-cli-1n while preserving necessary
  recovery evidence and transferring operational ownership.
size: medium
proposed_by: bbugyi200.athena.0f0
create_time: 2026-09-09 20:00:26
status: wip
---

# Plan: Retire the obsolete Obsidian Sync restoration epic

## Outcome

Retire `bob-cli-1l` in favor of the active git-only vault-sync epic `bob-cli-1n` without
reviving the broken Obsidian Sync path or discarding the only safe copy of any vault
content. The unfinished `bob-cli-1l` phases and their parent will be closed as
`superseded`; the two completed audit/protection phases will remain historically `done`.
Durable audit and rollback evidence will transfer to `bob-cli-1n`, while only the
obsolete, non-final phase-3 seed will be removed after a content-uniqueness gate.

This is one bounded operational cleanup rather than a new epic: one agent can inventory
the exact state, perform the guarded rollback, update the related beads, force-close the
obsolete tree, and verify the resulting host and bead state in one coherent pass.

## Established state and constraints

- `bob-cli-1l` is `in_progress`; phases `.1` and `.2` are closed `done`, while `.3`,
  `.4`, and `.5` remain `in_progress`. `bob-cli-1n` is still `in_progress`.
- No live SASE agent currently has a `bob-cli-1l*` name, the old land/phase agents are
  dismissed or completed, and `sase bead epic-symbols bob-cli-1l` reports no entries.
  Recheck both facts immediately before closing because they can change.
- There is no `bob-cli-1l` Patch or `bob-cli` code commit to revert. Current `bob-cli`
  commits `86bff41`, `eee290a`, and `4051bf5` belong to `bob-cli-1n` and must remain.
  Vault commits `3ab1ab31` and `9fce6eb7` also belong to `bob-cli-1n` and must remain.
- Research commit `f7717b15abb05a8ae910abf868caf1ff0fb477eb` is the read-only
  `bob-cli-1l.1` footprint audit. It documents that every live remote path existed on a
  device and is useful evidence for the replacement epic; do not revert it merely
  because its source epic is superseded.
- `/mnt/hercules/backup/bob-cli-1l2-protect/20260827T142411Z` is a 5.9 GB verified,
  independent athena/MacBook snapshot with reconciliation manifests. It is rollback
  material for `bob-cli-1n` and must remain until that epic's cutover/soak policy
  permits deletion.
- `/mnt/hercules/backup/bob-cli-1l3-recover/20260827T150057Z` is a 4.2 GB non-final
  canonical-seed draft plus manifests. Its phase note explicitly says it became stale
  before the destructive gate cleared. It is the only `bob-cli-1l` payload eligible for
  cleanup, and only after proving that none of its file versions are unique.
- `/tmp/crontab_bak_bob-cli-1l2_20260827T142313Z.txt` is byte-identical to the durable
  copy at
  `/home/bryan/var/backups/bob-pre-gitsync-metadata-20260827/athena-crontab-pre-quiesce-20260827T142313Z.txt`.
- The user systemd unit `ob-sync-bob.service` is enabled but inactive. The 03:30
  `bob nightly` cron entry is commented out with a `bob-cli-1l.2` quiesce marker. The
  old Obsidian remote `8a259ad922718b6d8400c1f0e3ba8abe` remains configured. Do not
  restart the service, reactivate the old nightly command, delete the remote, unlink a
  device, or change Sync filters: `bob-cli-1n` requires quiescence and owns cutover.

## Implementation

1. **Freeze the obsolete epic and refresh the inventory.** Re-read `bob-cli-1l..` and
   `bob-cli-1n..`, their dependency trees, and the relevant plan artifacts. Query live
   agents and processes for every `bob-cli-1l*` worker/monitor. If any have reappeared,
   terminate them by exact SASE agent name, wait for each to reach a terminal state, and
   confirm that no process can continue phase-3 remote polling or write a late bead
   update. Re-run `sase bead epic-symbols bob-cli-1l` and stop if any symbol remains.
   Capture the current user-service state, crontab, Sync remote/status, exact backup
   paths, repository heads, and worktree cleanliness before mutating anything.

2. **Classify and roll back only `bob-cli-1l`-exclusive effects.** Confirm from commit
   trailers and repository history that there is still no `bob-cli-1l` code/Patch change
   in `bob-cli` or the vault repository; never revert the identified `bob-cli-1n`
   commits. Preserve research commit `f7717b1` and the full phase-2 verified snapshot.
   Verify the phase-3 seed against its recorded SHA-256 manifest, then compare every
   seed file digest with the current vault, the phase-2 athena/MacBook snapshots, and
   the durable `bob-cli-1n` backups. If a digest exists only in the phase-3 draft,
   extract that exact file version plus path/hash provenance into a small retained
   handoff area under the durable `bob-cli-1n` backups and record it on `bob-cli-1n.1`;
   do not delete the draft until the uniqueness set is empty. Once the gate passes,
   delete only
   `/mnt/hercules/backup/bob-cli-1l3-recover/20260827T150057Z/canonical_seed_full`,
   retaining the phase-3 manifests as audit evidence. Remove the `/tmp` crontab copy
   only after re-proving it matches the durable backup.

3. **Transfer the quiesced operational state to the replacement epic.** Disable and stop
   `ob-sync-bob.service` at the user level so a reboot cannot resurrect the old sync
   engine. Keep the 03:30 nightly command commented out, but replace its obsolete
   `bob-cli-1l.2` marker with a `bob-cli-1n` cutover marker that points to the durable
   crontab backup; do not enable the new or old nightly schedule in this cleanup. Verify
   that the old remote ID and local Sync configuration are otherwise unchanged. The
   `bob-cli-1n.6` cutover remains responsible for enabling `bob-vault-sync`, restoring
   the revised git-only nightly command, disabling Obsidian Sync on the MacBook, and
   deciding later unlink/subscription cleanup.

4. **Leave explicit handoff notes before closure.** Append a concise summary to the
   `bob-cli-1n` epic recording that `bob-cli-1l` is being superseded early at the user's
   direction, which assets were retained, which phase-3 payload was removed, the old
   remote's unchanged state, and the exact service/cron state. Add focused notes to:
   - `bob-cli-1n.1`: phase-1 audit commit, phase-2 snapshot/manifests, any extracted
     unique phase-3 versions, and the fact that the phase-3 canonical seed was stale and
     must never be used as a convergence source;
   - `bob-cli-1n.5`: inherited disabled service/cron state and the requirement to leave
     deployment triggers disabled until cutover;
   - `bob-cli-1n.6`: this cleanup has already performed the plan's `bob-cli-1l` bead
     hygiene, so it must treat re-closing as a verification/no-op and retain ownership
     of the revised nightly schedule, final Obsidian disable/unlink steps, and backup
     retention after acceptance/soak.

   Also append notes to `bob-cli-1l.3`, `.4`, and `.5` explaining the selective rollback
   and identifying `bob-cli-1n` as the superseding epic. Add a parent note explaining
   why completed phases `.1` and `.2` remain `done` and why their evidence was retained.
   Use append-only bead notes; do not rewrite existing phase evidence.

5. **Force-close the obsolete bead tree with accurate resolutions.** After notes and
   cleanup are durable, invoke one user-authorized force close on `bob-cli-1l` with
   resolution `superseded` and a reason naming `bob-cli-1n`. This must sweep unfinished
   phases `.3`, `.4`, and `.5` to `closed/superseded` and close the parent as
   `closed/superseded`. Do not reopen or re-close `.1` and `.2`: their completed audit
   and protection work remains truthfully `closed/done`, and SASE rejects conflicting
   resolutions for already-closed beads. Do not close or otherwise change `bob-cli-1n`,
   and leave `bob-cli-1i` to the already-authored `bob-cli-1n.6` cleanup unless the user
   separately expands scope.

## Verification

- `sase bead show bob-cli-1l..` shows the parent and phases `.3`-`.5` closed with
  resolution `superseded`, phases `.1` and `.2` still closed `done`, and notes that name
  `bob-cli-1n` and accurately describe retained/removed state.
- `sase bead show bob-cli-1n..` shows `bob-cli-1n` still `in_progress`, with handoff
  notes on the epic and phases `.1`, `.5`, and `.6`; no phase status or dependency was
  disturbed. `sase bead doctor` reports a healthy store.
- `sase agent list -j` and a process scan show no live `bob-cli-1l*` worker or monitor;
  `sase bead epic-symbols bob-cli-1l` remains empty.
- `systemctl --user is-enabled ob-sync-bob.service` reports `disabled` and
  `systemctl --user is-active ob-sync-bob.service` reports `inactive`. The crontab has
  no active `bob nightly` entry and names `bob-cli-1n` plus the durable restore source.
  `ob sync-list-remote` and `ob sync-status --path /home/bryan/bob` still report the
  same old remote/configuration, proving this cleanup did not perform a premature Sync
  cutover or deletion.
- The phase-2 snapshot, manifests, and research audit remain readable. The phase-3
  `canonical_seed_full` directory is absent only after its manifest verified and the
  unique-digest set was empty or extracted durably; its manifests remain. The transient
  `/tmp` crontab copy is absent only after equality with the durable copy was verified.
- The `bob-cli`, research, and vault worktrees are clean apart from pre-existing user
  changes, the `bob-cli-1n` commits remain reachable, and no revert commit was created
  for evidence that the replacement epic still uses.
