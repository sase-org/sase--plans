---
tier: epic
title: Remove old_lib from Obsidian Sync and restore the vault under quota
goal: "The `old_lib/` directory no longer syncs through Obsidian Sync from athena, its
  851 MB is gone from the remote vault, every old_lib byte still exists on disk and in a
  durable backup, and `ob-sync-bob.service` completes sync cycles without `Vault limit
  exceeded`.

  "
phases:
  - id: backup
    title: Close the backup gap and gate the destructive window
    depends_on: []
    size: small
    description:
      "backup: capture a baseline of remote sync state, prove every old_lib byte exists
      outside Obsidian Sync, fix the vault .gitignore case gap that leaves one
      tax-return PDF untracked, and confirm the user decisions this epic depends on."
  - id: evacuate
    title: Evacuate old_lib and push the deletions to the remote vault
    depends_on:
      - backup
    size: medium
    description:
      "evacuate: quiesce the sync service and nightly cron, rename old_lib to a
      dot-prefixed staging directory so the sync client sees it as absent, canary-test
      that the server accepts deletes while over quota, then drain all 660 old_lib
      entries from the remote."
  - id: exclude
    title: Set the device-local exclusion and restore old_lib in place
    depends_on:
      - evacuate
    size: small
    description:
      "exclude: write ignoreFolders via ob sync-config, restore the staging directory to
      old_lib while sync is still stopped, then resume the service and cron and confirm
      across several cycles that no old_lib path is re-uploaded."
  - id: verify
    title: Verify quota recovery and run the fallback if version history holds
    depends_on:
      - exclude
    size: medium
    description:
      "verify: confirm sync cycles go green, and if attachment version history keeps the
      vault over 1 GB, execute the documented fresh-remote-vault rebuild with exclusions
      configured before the first sync."
  - id: document
    title: Document the exclusion and file the discovered follow-ups
    depends_on:
      - exclude
    size: small
    description:
      "document: record old_lib and its sync exclusion in the bob-cli README vault
      layout, add a runbook for the procedure, and file task beads for the unbounded
      sync log and the sase memory note that omits the sync topology."
proposed_by: bbugyi200.athena.0en
status: done
create_time: 2026-09-09 20:00:34
---

- **PROMPT:**
  [prompts/202608/unsync_old_lib.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/unsync_old_lib.md)

# Plan: Remove old_lib from Obsidian Sync and restore the vault under quota

## Context

`~/bob/` is Bryan's Obsidian vault. It syncs through Obsidian Sync via
`obsidian-headless` (`ob`), driven on athena by a systemd user service
`ob-sync-bob.service`, which runs `~/.local/bin/ob-sync-bob-poll`: a bash loop that
invokes `ob sync --path /home/bryan/bob` in a **fresh process** every 30 s with a 120 s
timeout.

The request is to stop syncing `~/bob/old_lib/`. Investigation showed that the literal
one-line change does not accomplish the underlying goal, so this plan covers both the
exclusion and the quota recovery it exists to enable. The scope expansion is deliberate
and is called out in "Scope note" below.

### Measured state (2026-08-27)

Sync is currently **failing**. `systemctl --user status ob-sync-bob.service` reports:

```
Sync failed: Error: Vault limit exceeded.
```

Remote vault contents, read from the sync client's own state database
(`~/.config/obsidian-headless/sync/8a259ad922718b6d8400c1f0e3ba8abe/state.db`, table
`server_files`):

| Top-level path  | Remote size  | Files |
| --------------- | ------------ | ----- |
| `old_lib`       | **851.1 MB** | 660   |
| `img`           | 65.3 MB      | 187   |
| `_meta`         | 16.1 MB      | 49    |
| `.obsidian`     | 10.6 MB      | 66    |
| `lib`           | 6.1 MB       | 27    |
| `xmind`         | 4.1 MB       | 20    |
| everything else | ~12 MB       | ~5455 |
| **total live**  | **965.7 MB** | 6464  |

Obsidian Sync **Standard** allows 1 GB total storage, 5 MB maximum file size, and 1
month of version history; version history and attachments both count toward the limit.
The vault is at 965.7 MB of live content plus history, hence the error. `old_lib/` is
88% of the remote vault. Removing it takes the remote to ~115 MB.

`old_lib/` is 814 MB on disk across 660 files, **all PDFs, zero Markdown**. It is a
legacy predecessor of `lib/` and is not referenced anywhere in bob-cli: it appears in no
source file, and the README "Vault layout" table documents `xlib/`, `lib/`, and `ref/`
but not `old_lib/`.

### Why excluding the folder is necessary but not sufficient

`ob sync-config --excluded-folders old_lib` writes `ignoreFolders: ["old_lib"]` into
`~/.config/obsidian-headless/sync/<vault-id>/config.json`. The filter in
`obsidian-headless/cli.js` is a plain prefix test — deobfuscated:

```js
_allowSyncFile(path, isFolder) {
  for (const r of this.ignoreFolders)
    if ((isFolder && path === r) || path.startsWith(r + "/")) return false;
  ...
```

So the value must be exactly `old_lib` — no leading or trailing slash, case-sensitive —
and it covers the folder and everything beneath it.

But the bidirectional remote-deletion branch in the same file reads:

```js
for (const p in serverFiles) {
  const m = serverFiles[p];
  if (!Object.hasOwn(localFiles, p)) {
    if (m.deleted) { ...; continue; }
    if (!filter.allowSyncFile(p, m.folder)) continue;   // <-- excluded: skipped forever
    ...select for remote deletion
  }
}
```

A server file that fails the filter is **never** selected for deletion. Obsidian's own
documentation states the same rule: "Adding a file to the Excluded files list does not
remove it from the remote vault if it has already been synced."

**Consequence:** setting the exclusion first permanently strands 851.1 MB on the remote,
sync stays broken, and the only remaining remedy is rebuilding the remote vault. The
deletions must therefore be pushed **before** the exclusion is set. That ordering is the
spine of this plan and must not be reordered.

### Backup gap discovered during investigation

The vault is also a git repo (`git@github.com:bobs-org/bob.git`). 659 of the 660 old_lib
files are tracked and present in `origin/master`. The 660th is **not**:

```
old_lib/gov/2024_tax_return.PDF   (1.14 MB)
$ git check-ignore -v old_lib/gov/2024_tax_return.PDF
.gitignore:3:*  old_lib/gov/2024_tax_return.PDF
```

The vault `.gitignore` ignores everything and re-includes `!*.pdf` in lowercase only;
git's ignore matching is case-sensitive on Linux, so the uppercase `.PDF` extension
falls through. That file **is** on the Obsidian Sync server, which means Obsidian Sync
is currently its only off-machine copy. Deleting it from the remote before it is backed
up would destroy that copy. Phase `backup` closes this gap and is a hard gate on
everything after it.

### Other devices

`server_files` records these device attributions: `athena-headless` (4206), `Kellys-MBP`
(1821), `athena` (225), `Kellys-MacBook-Pro.local` (205), `MacBookPro.lan` (6), and one
Verizon mobile client. These are last-writer attributions, not a live device roster, but
they establish that athena is not the only device on this vault.

This matters for a non-obvious reason. When phase `evacuate` pushes the deletions:

- A device that has **already** excluded `old_lib` filters those paths out entirely, so
  it neither uploads them nor applies the deletion — **it keeps its local copy.**
- A device that has **not** excluded `old_lib` receives the deletion and **removes its
  local copy.**

So the user-facing instruction is: on every other device that should keep its local
`old_lib/`, add `old_lib` to Settings → Sync → Excluded folders **before** phase
`evacuate` runs. Confirming this is a precondition of phase `backup`.

The exclusion is device-local in both clients: obsidian-headless stores it in its own
config directory, and the desktop app stores it in `.obsidian/sync.json`, which is
neither synced (the filter rejects it) nor tracked (the vault `.gitignore` lists it
under "Obsidian Sync device-local state").

### Scope note

The literal request — "stop syncing `~/bob/old_lib/`" — is one command. This plan is
larger because that command on its own leaves Obsidian Sync permanently broken and 851
MB permanently stranded, which is the opposite of the outcome the request is serving. If
the intent really is only "stop athena from uploading further old_lib changes" and the
stranded remote data and dead sync are acceptable, phase `exclude` alone is the whole
job and the rest of the epic can be closed. That is a decision for the approval gate,
not for the implementing agents.

### Alternatives considered and rejected

- **Upgrade to Sync Plus** (10 GB, 200 MB max file size, 12-month history, $8/mo). Zero
  data risk and no cross-device deletion, but it pays indefinitely to keep syncing 851
  MB of archival PDFs that no note or command references, and it does not satisfy the
  actual request. Worth surfacing to the user as the no-risk option.
- **Exclude by file type** (`--file-types image,audio,video`, dropping `pdf`). Wrong
  granularity: `lib/` holds 23 PDFs and `xlib/` 1, both actively managed by
  `bob highlights`. The exclusion must be by folder.
- **Rename to a permanent dot-directory** (`~/bob/.old_lib`). Obsidian Sync would ignore
  it forever, but Obsidian itself hides dot-directories, so the PDFs would become
  unopenable from the vault UI and invisible to `bob query`. Rejected as the end state;
  used only as transient staging in phase `evacuate`.
- **Rebuild the remote vault immediately.** Resets version history and guarantees quota
  recovery, but forces re-onboarding of every device and discards a month of history.
  Held as the phase `verify` fallback rather than the first move.

---

## Phase `backup`: Close the backup gap and gate the destructive window

Purely additive. No sync configuration or vault content is removed in this phase.

1. **Record the baseline** so later phases have something to diff against. Read the sync
   state DB directly (it is a plain SQLite file; `server_files.data` is JSON with
   `path`, `size`, `folder`, `deleted`):

   ```bash
   python3 - <<'EOF'
   import sqlite3, json, collections
   D = '/home/bryan/.config/obsidian-headless/sync/8a259ad922718b6d8400c1f0e3ba8abe/state.db'
   c = sqlite3.connect(D)
   tot = 0; by = collections.Counter(); n = collections.Counter()
   for p, d in c.execute("select path, data from server_files"):
       j = json.loads(d)
       if j.get('folder') or j.get('deleted'):
           continue
       tot += j.get('size', 0); by[p.split('/')[0]] += j.get('size', 0); n[p.split('/')[0]] += 1
   print(f"remote live total: {tot/1e6:.1f} MB")
   for k, v in by.most_common(10):
       print(f"  {k:12s} {v/1e6:9.1f} MB ({n[k]} files)")
   EOF
   ```

   Expected now: total ~965.7 MB, `old_lib` ~851.1 MB / 660 files. Save the output.

2. **Fix the `.gitignore` case gap.** Open the vault through `/sase_repo` if the
   implementing agent is not permitted to write `~/bob` directly. Add an uppercase
   re-include next to the existing `!*.pdf` line in `~/bob/.gitignore`:

   ```
   !*.pdf
   !*.PDF
   ```

   Then confirm the file is no longer ignored and commit it to the vault repo:

   ```bash
   git -C ~/bob check-ignore -v old_lib/gov/2024_tax_return.PDF   # expect: no match, exit 1
   git -C ~/bob add .gitignore old_lib/gov/2024_tax_return.PDF
   ```

   Commit through `/sase_git_commit`. Push to `origin/master`.

3. **Prove the backup is complete.** Every old_lib path on disk must now be in
   `origin/master`:

   ```bash
   git -C ~/bob fetch origin
   diff <(git -C ~/bob ls-tree -r --name-only origin/master -- old_lib | sort) \
        <(cd ~/bob && find old_lib -type f | sort)
   ```

   This must print nothing. **If it prints anything, stop the epic here** — do not start
   phase `evacuate` until it is empty.

4. **Take a second, independent copy** of `old_lib/` outside the vault and outside the
   git repo, on a path that is not a candidate for later cleanup.
   `rsync -a ~/bob/old_lib/ <destination>/` is sufficient; record the destination in the
   phase notes. Rationale: phase `evacuate` deletes 851 MB from the remote, and a single
   backup channel is not enough for an irreversible cross-device deletion.

5. **Confirm the user decisions.** These are questions for the user, not assumptions to
   make. Use `/sase_questions`:
   - Are there other active devices on this vault (a MacBook Pro, an iPhone)? For each
     one that should keep its local `old_lib/`, has `old_lib` been added to Settings →
     Sync → Excluded folders on that device? Devices without the exclusion will have
     `old_lib/` deleted locally when the deletions propagate.
   - Confirm that deleting 851 MB from the Obsidian Sync remote — including the month of
     version history for those files — is intended, given the alternative of upgrading
     to Sync Plus.

**Done when:** the diff in step 3 is empty, the independent copy exists and has been
verified by file count and byte size, the baseline numbers are recorded, and the user
has answered both questions affirmatively.

## Phase `evacuate`: Evacuate old_lib and push the deletions to the remote vault

This is the destructive phase. It runs with sync automation stopped and finishes with
the remote holding zero `old_lib` entries.

1. **Quiesce.** Two independent things run `ob sync` against this vault:

   ```bash
   systemctl --user stop ob-sync-bob.service
   ```

   and the 03:30 cron entry `bob nightly`, which runs `ob sync` and then commits and
   pushes the vault git repo. Comment out that crontab line for the duration of this
   phase and phase `exclude`, and restore it in phase `exclude`. Note that `bob nightly`
   and `bob bulk-git-commit` share an exclusive lock through `ob::acquire_lock()` in
   `src/native/ob.rs`, but the systemd poll service does **not** participate in that
   lock — stopping the service and gating the cron are two separate, both-required
   actions.

   Confirm nothing is running before continuing:

   ```bash
   systemctl --user is-active ob-sync-bob.service   # expect: inactive
   pgrep -af 'obsidian-headless/cli.js'             # expect: no output
   ```

2. **Canary first.** Do not move all 851 MB before confirming the server accepts
   deletions while the vault is over quota. The observed failure was on an _upload_;
   deletes send a null payload and should be accepted, but this is the one unverified
   assumption in the plan, so test it cheaply.

   Move one small subtree out of the sync client's view. Rename it to a **dot-prefixed**
   path inside the vault — the filter rejects any path beginning with `.`, so the client
   treats it as locally absent, while git records it as a rename rather than 660
   deletions:

   ```bash
   mkdir -p ~/bob/.old_lib_migrating
   mv ~/bob/old_lib/slides ~/bob/.old_lib_migrating/slides
   ```

   Then run one foreground sync and watch it:

   ```bash
   ob sync --path ~/bob
   ```

   Expect `Deleting remote file old_lib/slides/...` lines and no `Vault limit exceeded`.
   **If deletions are rejected, stop and escalate to the phase `verify` fallback** —
   move `slides` back first, then hand off.

3. **Evacuate the rest** once the canary succeeds:

   ```bash
   mv ~/bob/old_lib/* ~/bob/.old_lib_migrating/
   rmdir ~/bob/old_lib
   ```

   Use `mv` on the same filesystem so this is an instant rename, never a copy.

4. **Drain the remote.** The engine deletes one entry per internal iteration — files
   first (longest path first), then folders — and `ob sync` loops until it reports
   `Fully synced`. With 660 files plus 11 folders this takes a while, and the 120 s poll
   timeout does not apply here because the service is stopped. Run it in the foreground;
   if it exits before draining, run it again. Use `/sase_monitor` rather than blocking
   the turn.

   Track progress with the baseline script from phase `backup`, or directly:

   ```bash
   python3 -c "
   import sqlite3, json
   D='/home/bryan/.config/obsidian-headless/sync/8a259ad922718b6d8400c1f0e3ba8abe/state.db'
   rows=[json.loads(d) for p,d in sqlite3.connect(D).execute(
       \"select path,data from server_files where path like 'old_lib%'\")]
   live=[r for r in rows if not r.get('deleted')]
   print(len(live), 'live old_lib entries,', sum(r.get('size',0) for r in live)/1e6, 'MB')
   "
   ```

**Done when:** the query above reports 0 live `old_lib` entries and 0 MB, and the remote
live total from the phase `backup` baseline script has dropped from ~965.7 MB to ~115
MB. Do not start phase `exclude` before both hold — a partial drain that is then
excluded strands whatever is left.

## Phase `exclude`: Set the device-local exclusion and restore old_lib in place

The ordering here is what prevents the 851 MB from going straight back up. The exclusion
must land **while the folder is still staged** and **while sync is still stopped**.

1. **Set the exclusion**, with the service still stopped:

   ```bash
   ob sync-config --path ~/bob --excluded-folders old_lib
   ```

2. **Verify it landed** before restoring anything:

   ```bash
   ob sync-config --path ~/bob | grep 'Excluded folders'
   # expect: Excluded folders: old_lib

   python3 -c "
   import json
   p='/home/bryan/.config/obsidian-headless/sync/8a259ad922718b6d8400c1f0e3ba8abe/config.json'
   print(json.load(open(p)).get('ignoreFolders'))
   "
   # expect: ['old_lib']
   ```

   The value must be exactly `old_lib`. `old_lib/`, `/old_lib`, and `Old_lib` all fail
   the prefix test and would silently leave the folder syncing.

   Note that `--excluded-folders` is a whole-list replacement, not an append. The list
   is empty today, so passing `old_lib` alone is correct; if a future change adds
   others, pass the full comma-separated list.

3. **Restore the folder**, still with sync stopped:

   ```bash
   mkdir -p ~/bob/old_lib
   mv ~/bob/.old_lib_migrating/* ~/bob/old_lib/
   rmdir ~/bob/.old_lib_migrating
   ```

   Confirm the tree is intact against the phase `backup` baseline: 660 files, 814 MB,
   and the `origin/master` diff from phase `backup` step 3 still empty.

4. **Resume automation:**

   ```bash
   systemctl --user start ob-sync-bob.service
   ```

   and restore the commented-out `bob nightly` crontab line.

5. **Confirm no re-upload** across at least three poll cycles (~2 minutes). The sync log
   is at `~/.config/obsidian-headless/sync/8a259ad922718b6d8400c1f0e3ba8abe/sync.log`;
   it is append-only and currently 982 MB, so tail it rather than reading it:

   ```bash
   tail -f ~/.config/obsidian-headless/sync/8a259ad922718b6d8400c1f0e3ba8abe/sync.log \
     | grep -i old_lib
   ```

   Any `Uploading file old_lib/...` line means the exclusion is not in effect — stop the
   service immediately and recheck step 2.

   Because `ob-sync-bob-poll` spawns a fresh `ob sync` process every cycle, the config
   is re-read on each run and no service restart is required for the exclusion to take
   effect; the stop/start above exists to close the race window during the restore, not
   to reload configuration.

**Done when:** `ignoreFolders` is `["old_lib"]`, `~/bob/old_lib/` holds all 660 files on
disk, the service is active, the cron line is restored, and three consecutive sync
cycles log no `old_lib` path.

## Phase `verify`: Verify quota recovery and run the fallback if version history holds

1. **Check whether sync is green:**

   ```bash
   systemctl --user status ob-sync-bob.service --no-pager | tail -20
   ```

   Success looks like cycles completing without `Vault limit exceeded`. Confirm a
   round-trip actually works by touching a scratch note in the vault and watching it
   upload.

2. **If it is still red**, the cause is version history: on Standard, attachments are
   retained in version history for up to two weeks and history counts toward the 1 GB
   limit, so the 851 MB of just-deleted PDFs may still be charged. Two options, in
   order:
   - **Wait.** Re-check daily. Space is expected to free within roughly two weeks as
     attachment history expires. Sync stays broken for that window, which also means
     `bob nightly` cannot sync — acceptable only if the user agrees.
   - **Rebuild the remote vault.** This resets version history and is deterministic.
     Requires the user's decision because it re-onboards every device and discards a
     month of history:

     ```bash
     ob sync-create-remote --name bob-v2 --encryption e2ee
     ob sync-setup --vault bob-v2 --path ~/bob --device-name athena-headless
     # Configure BEFORE the first sync, so nothing unwanted is ever uploaded:
     ob sync-config --path ~/bob --excluded-folders old_lib
     ob sync-config --path ~/bob --file-types image,audio,pdf,video
     ob sync-config --path ~/bob --configs community-plugin,community-plugin-data,hotkey,appearance,appearance-data
     ob sync-config --path ~/bob    # verify all four settings before syncing
     ob sync --path ~/bob
     ```

     Those `--file-types` and `--configs` values reproduce the current configuration
     exactly; re-verify them against the phase `backup` baseline rather than trusting
     this plan's copy. Standard allows only 1 synced vault, so the old remote vault must
     be deleted to make room — do that only after the new one has fully synced and been
     verified. Then update `~/.local/bin/ob-sync-bob-poll` if it hardcodes anything
     vault-specific (it currently hardcodes only `VAULT_PATH`, so it should need no
     change), and re-point the other devices.

3. **Record the final numbers**: remote live total, `old_lib` remote entry count (must
   be 0), local `old_lib` file count (must be 660), and the service state.

**Done when:** `ob sync` completes cleanly end to end and a test edit round-trips, with
zero `old_lib` entries on the remote and all 660 files present locally.

## Phase `document`: Document the exclusion and file the discovered follow-ups

1. **README.** Add `old_lib/` to the "Vault layout" table in `README.md`, describing it
   as the archival predecessor of `lib/`, excluded from Obsidian Sync and backed up only
   through the vault git repo. The table currently documents `xlib/`, `lib/`, and `ref/`
   but not `old_lib/`, which is why its 851 MB went unnoticed.

2. **Runbook.** Add a short document covering the procedure this epic proved out: that
   excluding a folder never removes already-synced data, that deletions must be pushed
   before the exclusion is set, that the exclusion is device-local and must be set on
   every device, and that the exact `ignoreFolders` value is prefix-matched and
   case-sensitive. Place it alongside the other bob-cli docs.

3. **File task beads** through `/sase_new_task`, one per finding — do not fold these
   into this epic:
   - The sync log at `~/.config/obsidian-headless/sync/<vault-id>/sync.log` has grown to
     **982 MB** and is append-only with no rotation. `ob-sync-bob-poll` writes to it
     every 30 s. It needs truncation and a rotation policy.
   - 82 files totalling 23.6 MB are on the Obsidian Sync remote but untracked in the
     vault git repo, mostly under `_meta/migration/` and `xmind/`. Obsidian Sync is
     their only off-machine copy. Separate from the `.PDF` case gap that phase `backup`
     fixes.
   - `sase/memory/obsidian.md` describes `~/bob/` and `obsidian-headless` but records
     nothing about the sync topology: the `ob-sync-bob.service` poll loop, the
     device-local exclusion list, or the 1 GB Standard-plan ceiling. File a `memory`
     task bead proposing the addition. **Do not edit the memory note directly** — memory
     changes require the user's explicit approval, and approval recorded in a plan file
     does not count.

**Done when:** the README row exists, the runbook is committed, and the three task beads
are filed.

## Risks

| Risk                                                                                        | Mitigation                                                                                                                                                      |
| ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `old_lib/gov/2024_tax_return.PDF` is destroyed — Obsidian Sync is its only off-machine copy | Phase `backup` fixes the `.gitignore` case gap, commits and pushes the file, and gates the epic on an empty `origin/master` diff plus a second independent copy |
| Exclusion set before deletion strands 851 MB on the remote permanently                      | Phase ordering is explicit and phase `exclude` is blocked on a verified-empty drain                                                                             |
| `old_lib/` is deleted on other devices                                                      | Phase `backup` requires the user to exclude `old_lib` on each device that should keep it, before phase `evacuate` runs                                          |
| `bob nightly` fires mid-window and commits a mass rename, or races `ob sync`                | Cron line is gated for the duration; staging uses a dot-directory so git sees a rename, not 659 deletions                                                       |
| Sync service re-uploads old_lib during the restore                                          | Service is stopped for the whole staging window and only restarted after `ignoreFolders` is verified                                                            |
| Server rejects deletes while over quota                                                     | Phase `evacuate` canaries one small subtree first and escalates to the phase `verify` fallback rather than proceeding blind                                     |
| Version history keeps the vault over 1 GB after deletion                                    | Phase `verify` carries the wait-or-rebuild fallback, with the rebuild configured before its first sync                                                          |
| Wrong exclusion value silently no-ops                                                       | Phase `exclude` verifies both the CLI output and the raw `ignoreFolders` JSON, and the plan states the exact prefix-match semantics                             |
