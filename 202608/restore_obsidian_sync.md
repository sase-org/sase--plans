---
tier: epic
title: Restore Bob Obsidian Sync and establish a sub-1 GB footprint policy
goal: "Obsidian Sync completes end to end on athena and the MacBook from a fully
  reconciled, backed-up vault whose charged storage is below the Standard plan's 1 GB
  limit, with explicit and repeatable decisions about every excluded or unsupported
  content class.

  "
phases:
  - id: audit
    title: Reconcile live, local, remote-only, and historical storage
    depends_on: []
    size: medium
    description:
      "audit: produce a read-only, cross-device inventory that distinguishes live bytes,
      stranded remote entries, pending uploads, and charged version history."
  - id: protect
    title: Preserve every copy and approve the new sync policy
    depends_on:
      - audit
    size: medium
    description:
      "protect: capture independent snapshots, reconcile divergent device state, and
      obtain explicit approval for the rebuild and optional exclusions."
  - id: recover
    title: Rebuild the remote vault with exclusions preconfigured
    depends_on:
      - protect
    size: medium
    description:
      "recover: replace the history-bloated remote vault, seed it from the reconciled
      tree, and reconnect each device without uploading excluded content."
  - id: verify
    title: Prove quota headroom, round trips, and data completeness
    depends_on:
      - recover
    size: medium
    description:
      "verify: demonstrate green sync cycles, cross-device round trips, a sub-limit
      storage reading, and manifest-level preservation."
  - id: document
    title: Record the footprint policy and operational audit
    depends_on:
      - verify
    size: small
    description:
      "document: update the runbook with measured recommendations, the rebuild
      procedure, and a reproducible redacted footprint audit."
proposed_by: bbugyi200.athena.0er
create_time: 2026-09-09 20:00:26
status: wip
---

- **PROMPT:**
  [prompts/202608/restore_obsidian_sync.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/restore_obsidian_sync.md)

# Plan: Restore Bob Obsidian Sync and establish a sub-1 GB footprint policy

## Outcome and recommendation

Rebuild the Obsidian Sync remote vault after first making complete, independently
verified copies of the athena vault, the MacBook vault, and every live remote-only file.
Configure exclusions and attachment types before the new remote's first upload, seed
from a reconciled canonical tree, then reconnect the other devices one at a time.

This is the recommended immediate recovery because the current failure is not caused by
the live vault being near 1 GB. The old remote has only **114.528 MB across 5,804 live
files**, but every upload still fails with `Vault limit exceeded`. Official Obsidian
documentation says that version history counts toward storage, deleted attachments are
retained for up to two weeks, and the quickest way to remove that charge is a new remote
vault; see [Plans and storage limits](https://obsidian.md/help/sync/plans) and
[Version history](https://obsidian.md/help/sync/version-history).

No additional live-file exclusion is required to fit under the 1 GB Standard limit. The
current live set has about **885 MB of nominal headroom**. Excluding and deleting more
attachments from the existing remote would not repair today's outage: their bytes would
enter attachment history rather than disappear immediately. The plan still records
sensible longer-term no-sync candidates, but treats them as policy choices—not as a
substitute for clearing the retained history.

The previous epic `bob-cli-1e` remains the authoritative context for the `old_lib/`
evacuation and its ordering constraints. Its follow-up `bob-cli-1i` records the chosen
wait-until-history-expires path. Approving this plan supersedes that wait with the
documented immediate-rebuild fallback; it does not reopen or repeat the completed
`old_lib/` work.

## Evidence collected on 2026-08-27

The plan author performed read-only checks against the headless client's state database,
the running service, the current sync configuration, recent sync logs, and a freshly
opened clone of the vault's Git remote.

- `ob-sync-bob.service` is active but fails about every 30 seconds with
  `Vault limit exceeded`.
- Each observed cycle reaches a new 0.101 MB `xlib/chat/AGENTS.pdf`, attempts to upload
  it, and fails. The failing object is far below Standard's 5 MB per-file ceiling, so
  this is the total-storage gate.
- `old_lib/` has zero live remote entries and remains excluded on athena. The completed
  epic separately verified the same exclusion on the MacBook.
- Athena's attachment filter is `image,audio,pdf,video`; `unsupported` is disabled. Its
  configuration-sync categories are community plugins, community-plugin data, hotkeys,
  appearance, and appearance data.
- The remote live index reports the following material roots:

| Path                                     |     Live files |           Live size | Audit finding                                                                                                                                                                                                         | Default recommendation                                                                      |
| ---------------------------------------- | -------------: | ------------------: | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `img/`                                   |            187 |           65.269 MB | 182 files / 59.985 MB have textual references in the Git-backed Markdown snapshot; all five apparent orphans are recent July/August files, including one 2.513 MB file not yet in Git because nightly sync is blocked | Keep syncing; review apparent orphans individually only after live-device reference checks  |
| `_meta/migration/`                       |             47 |           16.059 MB | One-time migration material; 31 files / 15.831 MB are absent on athena, and 36 files / 15.833 MB are not in the Git remote; several dashboards reference the tree                                                     | Best optional no-sync candidate, but archive and verify it before exclusion                 |
| `.obsidian/plugins/`                     |             52 |           10.552 MB | Mostly reproducible plugin code, but it carries active cross-device Bob/community-plugin behavior; 15 files / 1.365 MB are not in the vault Git remote                                                                | Secondary candidate only if per-device plugin provisioning replaces Sync; otherwise keep    |
| `lib/`                                   |             27 |            6.134 MB | Every file is referenced and Git-backed; this is the active Highlights library                                                                                                                                        | Keep syncing                                                                                |
| `xmind/`                                 |             20 |            4.139 MB | Every file is referenced, every file is absent on athena, and none is in the Git remote; athena's disabled `unsupported` type means these are live remote residue rather than active athena sync                      | Preserve first; explicitly decide whether the rebuilt vault should sync `unsupported` files |
| `_generated/`                            |          1,131 |            3.389 MB | Fully Git-backed, 255 files are directly referenced, and Git shows no changes under this root in the last 30 days                                                                                                     | Keep syncing; there is no current history-amplification evidence                            |
| Year/daily-note trees and other Markdown |      thousands | under 9 MB combined | Core note content, small relative to the limit                                                                                                                                                                        | Keep syncing                                                                                |
| `xlib/`                                  | 1 pending file |    0.101 MB pending | Active Highlights intake; the current upload failure exposes the quota problem                                                                                                                                        | Keep syncing                                                                                |
| `old_lib/`                               |  0 remote live |    0 MB remote live | 1.7 GB remains local and backed up; excluded on each verified device                                                                                                                                                  | Keep excluded                                                                               |

The remote index also contains roughly 20 MB of files absent from athena—principally
`_meta/migration/` reports/tools and all 20 XMind files. Because many are neither in the
athena tree nor the vault Git remote, deleting the old remote before retrieving or
verifying another copy would lose the only confirmed off-machine instance. This is the
same durability class tracked by `bob-cli-1g` and must be closed as a hard gate, not
accepted as cleanup collateral.

The current state database contains live-file metadata, not billed historical versions.
Therefore its 114.528 MB total must never be presented as the account's charged usage.
Phase `audit` obtains the actual **Storage usage** reading from desktop Obsidian's
Settings → Sync view and preserves that reading separately from the live-index total.

## Sync-footprint policy proposed for approval

Use the following policy for the rebuilt remote unless the user changes it at phase
`protect`'s decision gate:

1. **Always exclude `old_lib/`.** It is archival, already evacuated, and its local
   preservation has been verified by `bob-cli-1e`.
2. **Keep Markdown, images, audio, PDFs, and video enabled.** This preserves core notes,
   referenced images, the active `lib/` library, and `xlib/` intake. Their combined live
   footprint is comfortably below the plan limit after history is reset.
3. **Leave `unsupported` disabled by default.** Before rebuilding, archive all XMind and
   migration artifacts outside Obsidian Sync. The user may instead opt in to
   `unsupported` after the audit demonstrates that the desired XMind files exist on the
   canonical seed device; they add only 4.139 MB and are all referenced.
4. **Consider excluding `_meta/migration/` after archival.** It is the strongest
   additional folder candidate because it is one-time migration output rather than
   current notes. Preserve a browsable archive and identify the handful of notes that
   link into it before adding the exclusion on every device. The rebuild itself does not
   depend on accepting this optional exclusion.
5. **Keep community plugin code/data syncing for now.** The 10.552 MB is small and the
   active Bob experience depends on consistent plugins. Turning off `community-plugin`
   and `community-plugin-data` is a reasonable later optimization only after
   `bob plugins sync` plus a third-party plugin bootstrap is available on every device.
6. **Do not blanket-exclude `img/`, `lib/`, `xlib/`, `_generated/`, year folders, or
   XMind content as a deletion shortcut.** The evidence shows active references or
   workflow ownership. The five apparently unreferenced images total 5.283 MB and are
   too recent for an automated orphan decision; review them visually and against the
   live, not merely Git-backed, Markdown set.

## Phase `audit`: Reconcile live, local, remote-only, and historical storage

This phase is read-only. It must not change sync settings, create/delete a remote vault,
move vault files, or run a bidirectional sync against a temporary tree.

1. Stop treating the state DB's `server_files` table as a billing report. Capture four
   distinct inventories with UTC timestamps:
   - live remote entries from the athena headless state DB;
   - athena local entries from `local_files` and a filesystem manifest;
   - the MacBook filesystem manifest over SSH;
   - the Git remote's tracked paths and blob identities, opened through `/sase_repo`.
2. For each path record byte size, content hash, modification time, top-level root,
   extension, current-device sync eligibility, remote presence, athena presence, MacBook
   presence, Git presence, and independent-backup presence. Redact the headless config's
   encryption material and never copy credentials into logs, plans, bead notes, or
   documentation.
3. Extract the actual billed **Storage usage** from desktop Obsidian Settings → Sync on
   the MacBook. Record the UI reading next to, but never merge it with, the 114.528 MB
   live-index measurement. If the UI cannot be inspected over SSH, ask the user for the
   displayed value rather than guessing.
4. Recheck the active filters on athena and MacBook. Classify every live remote path as:
   actively eligible on both devices, eligible on only one device, filtered but still
   stranded remotely, or already excluded. In particular, verify the MacBook's
   attachment-type toggles and not just its `ignoreFolders` value.
5. Re-run the reference analysis against the union of current Markdown from athena and
   MacBook, not only the Git snapshot. Resolve Obsidian basename links as well as full
   paths. Produce a review list for apparent attachment orphans, but do not delete or
   move them.
6. Quantify recent history pressure using the last month of Git changes and available
   Sync history. Report file-version events by root, especially generated notes and
   plugin payloads. Preserve the observed result that `_generated/` had no Git changes
   in the prior 30 days unless the live manifests disprove it.

**Done when:** there is one redacted inventory that accounts for every live remote path
and every unsynced local path, the actual billed usage is recorded or explicitly marked
as awaiting the user's UI reading, and each recommendation has measured reclaim,
reference, activity, and durability evidence.

## Phase `protect`: Preserve every copy and approve the new sync policy

This phase may create additive backups, but the old remote and every vault tree remain
unchanged until all gates pass.

1. Quiesce every writer: stop `ob-sync-bob.service`, prevent `bob nightly` from running,
   close or pause Sync on the MacBook and mobile clients, and confirm no headless client
   process remains. Record how each writer will be restored.
2. Take independent, timestamped snapshots of the complete athena and MacBook vaults,
   including ignored files. Generate sorted path/size/hash manifests and verify the
   snapshots against their sources. Do not count two paths on the same filesystem as
   independent durability.
3. Retrieve a complete plaintext copy of the old remote's **live** contents before it
   can be deleted. Prefer a fresh temporary checkout configured as pull-only before its
   first sync, with all attachment types—including `unsupported`—enabled. It must have
   an empty source tree so there is nothing to upload. If the client cannot prove that
   ordering, use verified MacBook copies and the remote index instead; never risk a
   bidirectional merge merely to obtain a snapshot.
4. Reconcile device divergence created during the outage. Build a canonical seed tree
   containing the newest non-conflicting copy of each path and an explicit conflict
   report for paths changed differently on athena and MacBook. Preserve both versions of
   every unresolved conflict and ask the user which one is canonical.
5. Close the remote-only durability gap: every `_meta/migration/`, `xmind/`, plugin, and
   other live remote path must appear in at least one verified snapshot independent of
   the old remote. Coordinate with existing tasks `bob-cli-1g` and `bob-cli-1k` rather
   than filing semantic duplicates.
6. Use `/sase_questions` for the destructive decision gate. Obtain explicit answers to:
   - replace the current remote now, losing its retained version history, rather than
     continue the `bob-cli-1i` wait;
   - accept the proposed attachment/config policy, including whether `unsupported` XMind
     files should resume syncing;
   - accept or decline `_meta/migration/` as an additional device-local folder
     exclusion;
   - confirm which active devices must be reconnected and which local trees are
     canonical.

**Hard stop:** do not unlink or delete the old remote unless all manifests reconcile,
all remote-only bytes have a verified copy, conflicts are resolved, the encryption
password needed for the new E2EE remote is available without exposing it, and the user
has explicitly approved the destructive rebuild.

**Done when:** the three snapshots/manifests are verified, one canonical seed tree is
defined, all device divergence is resolved or preserved, and every policy/rebuild
decision is recorded.

## Phase `recover`: Rebuild the remote vault with exclusions preconfigured

1. Keep all automated and desktop sync clients stopped. Record the old remote's ID,
   name, region, encryption version, live manifest, charged usage, and device roster; do
   not record encryption secrets.
2. Because Standard permits one remote vault, disconnect every device and delete the old
   remote only after phase `protect`'s hard gate. Use the supported Obsidian UI for any
   operation the headless CLI does not expose. Treat this deletion as irreversible loss
   of version history.
3. Create a new E2EE remote. On the canonical seed device, configure **before the first
   sync**:
   - bidirectional mode and the intended conflict strategy;
   - the full device-local exclusion list, always including exact lowercase `old_lib`
     and optionally `_meta/migration` if approved;
   - attachment types `image,audio,pdf,video`, plus `unsupported` only if approved;
   - the approved configuration categories, retaining the existing community-plugin,
     community-plugin-data, hotkey, appearance, and appearance-data set by default.
4. Print and inspect the effective configuration before exposing the canonical tree to
   the client. Remember that `--excluded-folders`, `--file-types`, and `--configs` are
   whole-list replacements. Verify exact values rather than incrementally appending.
5. Seed the new remote from the reconciled canonical tree. Keep `old_lib/` staged out of
   view until the exclusion is proven, then restore it locally. If `_meta/migration/` is
   excluded, use the same evacuate → drain/initial-sync → set and verify exclusion →
   restore ordering established by `docs/obsidian-sync-exclusions.md`.
6. Compare the new remote live manifest with the policy-derived expected set before
   reconnecting another device. Investigate every missing or unexpected path. The
   expected total is far below 1 GB—roughly 95–115 MB depending on whether remote-only
   unsupported content and optional plugin/migration categories are retained.
7. Reconnect the MacBook and then each mobile device one at a time. Configure the same
   exclusions and attachment-type decisions locally before allowing its first merge.
   Keep its pre-rebuild snapshot until the final manifest comparison passes. On the
   MacBook, verify the stored IndexedDB values with the procedure already documented in
   `docs/obsidian-sync-exclusions.md`; the absence of `.obsidian/sync.json` is not
   evidence.
8. Restore the systemd poller and the nightly job only after every active device is
   connected and no device attempts to upload an excluded path.

**Done when:** the new remote contains exactly the approved eligible set, its live bytes
match the phase `audit` projection, every device uses the intended local filter policy,
and no old/excluded data was uploaded.

## Phase `verify`: Prove quota headroom, round trips, and data completeness

1. Record the new remote's actual **Storage usage** from desktop Settings → Sync and the
   independently calculated live-index total. The UI reading must be below 1 GB and
   should be close to the initial live set because the remote has no inherited history.
2. Observe at least three consecutive athena poll cycles completing without
   `Vault limit exceeded`, upload retries, or excluded-path activity.
3. Exercise two bidirectional round trips:
   - a scratch Markdown edit from athena to the MacBook and back;
   - the pending `xlib/chat/AGENTS.pdf` from athena to the MacBook, followed by its
     normal Highlights workflow or a non-destructive presence/hash check. Remove only
     the purpose-created scratch note after its deletion also round-trips.
4. Compare athena, MacBook, new-remote, Git, and snapshot manifests. Differences must be
   entirely explained by the approved device-local exclusions or unsupported-type
   policy. Confirm that all `old_lib/`, archived migration, and XMind bytes remain in
   their promised local/backup locations even when absent from the new remote.
5. Run `bob bulk-git-commit` or the normal nightly path as appropriate and verify the
   vault Git remote catches up. Do not mistake a green Obsidian round trip for restored
   Git durability.
6. Recheck the service and account after another ordinary user edit so success is not
   merely the empty initial-sync state. Retain all pre-rebuild snapshots until this
   check and the user's MacBook confirmation pass.

**Done when:** billed usage is demonstrably under the plan limit, normal and attachment
round trips work across devices, the poll service and nightly workflow are healthy, and
every pre-rebuild path is accounted for by the approved policy or a verified backup.

## Phase `document`: Record the footprint policy and operational audit

1. Add a concise, redacted audit report to the bob-cli documentation. Include the
   measured category table, the distinction between live and billed/history bytes, the
   accepted exclusions, the expected baseline size, and the date/device scope. Never
   include E2EE keys, passwords, tokens, or raw config fields containing them.
2. Extend `docs/obsidian-sync-exclusions.md` with the fresh-remote recovery path, the
   remote-only/unsupported-file backup gate, the requirement to reconcile divergent
   device trees, and the actual verification results.
3. Add a reusable read-only footprint command or checked script only if it can read the
   state DB without credentials, emit top-level/extension/eligibility/durability
   summaries, and clearly label live bytes as **not billed storage**. Otherwise document
   the audited command sequence rather than adding brittle product code.
4. Record the final disposition of existing related tasks. Do not create duplicate task
   beads: update `bob-cli-1i` with the rebuild result, and add corroborating evidence to
   `bob-cli-1g`/`bob-cli-1k` where the backup audit overlaps their scope. Use the
   required bead procedures before any task mutation.

**Done when:** a future operator can reproduce the audit, understand why further live
deletions were not the immediate remedy, see which content is intentionally local-only,
and rebuild/reconnect safely without consulting this epic's chat transcript.

## Risks and mitigations

| Risk                                                                                          | Mitigation                                                                                                                     |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| More exclusions fail to clear the current error because deleted attachments remain billed     | Use the official fresh-remote recovery path for immediate service restoration; treat optional exclusions as future policy only |
| The old remote is deleted while it is the only confirmed copy of XMind or migration artifacts | Require a pull-only remote snapshot plus athena/MacBook snapshots and hash-manifest coverage before deletion                   |
| Athena and MacBook have diverged while Sync is red                                            | Reconcile both local manifests into a canonical seed and preserve both sides of every conflict                                 |
| `old_lib/` or another local-only folder uploads into the new remote                           | Configure and verify the complete exclusion list before the first upload and reconnect devices one at a time                   |
| A whole-list CLI option silently removes another filter/config choice                         | Capture baselines and verify the full effective `excluded-folders`, `file-types`, and `configs` sets after every change        |
| Apparent image orphans are actually referenced by unsynced notes                              | Analyze the union of live device Markdown, then require visual/user review; no automated image deletion is in scope            |
| Disabling plugin sync breaks Bob behavior on another device                                   | Keep plugin sync by default; allow it to be disabled only with a verified per-device provisioning replacement                  |
| Live-index totals are mistaken for charged usage                                              | Always report state-DB live bytes and desktop Storage usage as separate measurements                                           |
| A new remote restores Obsidian but nightly Git backup remains broken                          | Verify both an Obsidian round trip and the independent Git commit/push path before completion                                  |
