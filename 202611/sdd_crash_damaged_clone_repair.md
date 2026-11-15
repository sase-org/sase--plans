---
tier: tale
title: Self-heal crash-damaged SDD sidecar clones
goal:
  Machine-managed SDD recovery repairs clones whose loose objects were truncated by an
  unclean shutdown, SDD clones are crash-hardened via repo-local fsync config, and the
  damaged bob-cli plans clone behind the 2026-11-15 axe digest error is repaired.
size: medium
decisions:
  live_repair:
    ask: How should the coder repair the live bob-cli plans clone once the fix is in?
    choices:
      gate:
        You approve the exact repair command via /sase_gate before bob-cli's checkout is
        touched
      run:
        The coder runs the one-shot repair itself after just check passes and reports it
      defer:
        The clone stays broken until you reinstall sase; the hourly digest error
        continues
    default: gate
    why:
      Lossless, but it writes to another project's primary checkout, so you see the
      command first
    answer: gate
  global_fsync:
    ask:
      Should the coder also crash-harden every repo on your machines via the chezmoi
      global gitconfig?
    default: false
    why:
      Only an unattended SASE clone was damaged; machine-wide fsync is a personal
      trade-off
    answer: false
proposed_by: bbugyi200.athena.1ch
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 06:16:55
status: wip
---

# Self-heal crash-damaged SDD sidecar clones and harden them against unclean shutdowns

## Problem

The axe digest `~/.sase/axe/error_digests/digest_20261115_051404.txt` reports one error.
It comes from routine `sdd-sidecar`, job `workspace_sdd_clone_recovery`, and it is
written from `_append_recovery_error` in `src/sase/sdd/_store_integration.py`:

```
SDD repository /home/bryan/projects/github/bobs-org/bob-cli/sase/repos/plans is not safe
to write: not a valid Git worktree; could not list unmerged index entries: error: object
file .git/objects/88/fd06dfb7bad3904efc2a47af7e68ddd5bb6b39 is empty ... fatal: unable to
read 88fd06df...; machine-managed recovery failed: the checkout is not a valid Git worktree
```

The report marker (`.git/sase-sdd-recovery-report.json`) is rewritten every hour. The
error will keep coming back until the clone is repaired, because SASE's machine-managed
recovery has no handling for object-store damage.

## Root cause (verified during planning)

1. **Unclean shutdown.** The host crashed at 2026-11-14 06:55:34. The journal for that
   boot (`journalctl --list-boots`, boot -1) ends with no shutdown sequence, and the
   next boot started at 07:19. At that moment the sidecar auto-sync chop was
   fast-forwarding bob-cli's `plans` sidecar to `3e16b719` ("Link approved epic plan to
   its bead: bank_badge_strip_family"). The call path is `sync_primary_sidecar_role` →
   `refresh_clean_linked_checkout` → `fetch_git_upstream` + `merge_git_ff_only`, which
   matches the `merge origin/main: Fast-forward` reflog lines.
2. **Git does not fsync loose objects by default.** The default `core.fsync` is
   `committed,-loose-object`, and no fsync config is set at any scope on this machine.
   The fetch was small, below `transfer.unpackLimit`, so it was unpacked into loose
   objects that never reached disk. The ref and index replacements did survive, because
   ext4 flushes data before a rename replaces an existing file. The result is on-disk
   state that is consistent but points at nothing:
   - 9 zero-byte loose objects, all with mtime 06:55: 2 commits, 4 trees, and 3 blobs.
   - `refs/heads/main` and `refs/remotes/origin/main` both point at the empty commit
     `3e16b719`.
   - Two worktree files truncated to 0 bytes: `202611/bank_badge_strip_family.md` and
     `202611/bank_badge_strip_family.plan-decisions.json`.
   - `logs/HEAD` and `logs/refs/heads/main` end in NUL padding. Git tolerates this.
3. **Recovery refuses instead of healing.** `inspect_sdd_repository` marks the clone
   `valid_worktree=False`. `managed_recovery_branch()` in
   `src/sase/sdd/_repository_recovery_git.py` then returns "the checkout is not a valid
   Git worktree", and recovery ends as `UNRECOVERABLE` before it ever fetches.

**The repair is lossless.** A fresh clone of
`git@github.com:bobs-org/bob-cli--plans.git` contains all 9 objects. `ORIG_HEAD`
(`18e7cb95`) is an ancestor of `3e16b719`. Compared against the damaged tip using a
temporary index, the live worktree differs only in the two truncated files, and it has
no untracked files. A scan of every clone under `~/projects`, `~/.local/state/sase`,
`~/.sase`, and the chezmoi source found zero-byte objects in this one clone only.

**Repair mechanics, tested on scratch repos with git 2.47.3:**

- While zero-byte object files are present, every fetch fails ("did not send all
  necessary objects"), because git believes it already has those objects.
- After removing them, a plain `git fetch` restores missing commits and trees. It does
  **not** restore a missing blob whose commit is still intact, because negotiation tells
  the server the client already has it.
- `git fetch --refetch` restores everything, after which `git fsck --connectivity-only`
  is clean.

## Goals

1. Machine-managed recovery self-heals crash-truncated loose objects. It quarantines the
   zero-byte files, refetches from the configured upstream, verifies the result, and
   then continues through the existing snapshot → reset → retry flow. That flow already
   restores truncated worktree files.
2. When the damage is not repairable from upstream, recovery fails closed with a message
   that names the actual problem, not the misleading "finish or abort the existing Git
   operation" text.
3. SASE-managed SDD clones are crash-hardened through repo-local git config, so this
   class of damage stops happening.
4. The live bob-cli clone is repaired (see the `live_repair` decision).

## Design

### 1. Object-store repair module (new: `src/sase/sdd/_repository_recovery_objects.py`)

Keep this Python-only, next to the existing recovery modules. This recovery logic has no
counterpart in `sase-core`, and porting it is out of scope.

- `detect_object_store_damage(git_dir) -> ObjectStoreDamage | None`:
  - Resolve the objects directory with `git rev-parse --git-path objects`, which works
    with a corrupt store. Fall back to `git_dir / "objects"`.
  - Report damage when either of these is true:
    - Some loose object is zero bytes: a regular file named
      `objects/<2 hex>/<38 or 62 hex>` (SHA-1 or SHA-256) with size 0. A valid loose
      object is a zlib stream and can never be zero bytes, so this test has no false
      positives.
    - A quarantine manifest from an earlier attempt is still `"status": "pending"`.
  - Reuse the loose-object bucket check `_is_loose_object_bucket` from
    `src/sase/sdd/_store_maintenance.py`. Move it to a small shared helper if importing
    it would cross a private boundary that symvision rejects.
- `repair_crash_damaged_object_store(root, *, git_dir, branch, runner, op_prefix, now) -> str | None`.
  It returns `None` on success, or an error detail.
  1. **Quarantine.** First write `manifest.json` atomically under
     `<git_dir>/sase-object-quarantine/<UTC stamp>-<short hash>/`, containing
     `clone_path`, `timestamp`, the object ids, and `"status": "pending"`. Then
     `os.replace` each zero-byte file into that directory, keeping its bucket/name
     layout. The files hold no data, so this loses nothing, and the move can be undone
     by hand.
  2. **Find the remote.** Call the existing
     `configured_upstream(root, branch, runner, op_prefix)`. If the remote is `"."` (a
     local upstream), fail closed: there is nothing to refetch.
  3. **Refetch.** Run `git fetch --refetch --prune <remote>` with `network=True` and
     `op=f"{op_prefix}.recovery.object_refetch"`. If it fails, return a detail like:
     "object store damaged by an unclean shutdown: quarantined N zero-byte loose objects
     under <dir>; refetch from <remote> failed: <redacted git error>". The manifest
     stays pending, so the next admitted attempt retries. `--refetch` needs git 2.36 or
     newer; an older git simply surfaces its error here.
  4. **Verify.** Run `git rev-list --objects --quiet HEAD <upstream_ref>`. If it fails,
     return "objects still missing after refetch; they existed only in this clone
     (unpublished commits or staged blobs): <stderr>; SASE did not reset the branch".
     That case must not reach the reset.
  5. **Finish.** On success, rewrite the manifest with `"status": "repaired"` and
     `repaired_at`.

### 2. Wire the repair into `recover_machine_managed_sdd_repository` (`src/sase/sdd/_repository_recovery.py`)

After the first `inspect_sdd_repository(...)` and **before**
`managed_recovery_branch(starting)`:

- Call `detect_object_store_damage(starting.git_dir)`. When it finds damage:
  - If `starting.branch` is `None`, return `UNRECOVERABLE` ("object-store repair refuses
    a detached HEAD"). `symbolic-ref` works with a corrupt store, so the branch is still
    known.
  - Apply the existing `expected_branch` check.
  - Run the recovery-attempt admission: the cooldown check plus the attempt-marker
    write. This must be the **same single admission** the normal path uses. Factor it
    into a helper and skip the second admission when the repair already ran it.
    Otherwise the marker written by the repair would immediately put the normal path
    into `RECOVERY_COOLDOWN`.
  - Call `repair_crash_damaged_object_store(...)`. If it returns an error, return
    `managed_recovery_failure(...)` with that detail. The clone is still invalid, so the
    status becomes `UNRECOVERABLE`, and the axe report carries the precise detail.
  - Re-inspect: `starting = inspect_sdd_repository(root, runner)`.
- Then continue unchanged: `managed_recovery_branch`, upstream resolution, recovery ref,
  fetch, snapshot (this stashes the truncated worktree files into the recovery
  snapshot), `reset --hard`, `verify_managed_reset`, and the integration retry, ending
  in `RECOVERED`.
- When no damage is detected, behavior must be identical to today. The existing recovery
  tests must pass untouched.

The store write lock is already held for this whole block, so no cooperating SASE writer
can race the quarantine.

### 3. Crash-hardening config for SDD clones (new: `src/sase/sdd/_git_durability.py`)

- `ensure_sdd_clone_durability(repo_root) -> bool` is best-effort: it never raises, logs
  at debug level on failure, and returns `True` when it changed config.
  - For each pair (`core.fsync`, `committed,index,reference`) and (`core.fsyncMethod`,
    `batch`), run `git -C <root> config --get <key>`:
    - Exit 1 (unset at every scope): run `git -C <root> config --local <key> <value>`.
    - Exit 0 (already set at any scope, including global): leave it alone, so an
      explicit user choice always wins.
    - Any other result: skip.
  - `batch` gives loose objects a single flush per git operation, so the cost stays
    small. These settings need no objects, so they work on a corrupt clone too.
- Repo-local config is used instead of per-command `-c` flags because the write that
  crashed came from `refresh_clean_linked_checkout`, which bypasses `sdd_git_command()`.
  Repo-local config hardens every writer: SASE Python, the chop, manual git, and agents.
- Call sites, each wrapped so a failure never changes the caller's outcome:
  1. `pull_sdd_clone` (`src/sase/sdd/_store_integration.py`), immediately before
     `integrate_machine_managed_sdd_repository`, after the freshness and cooldown
     short-circuits.
  2. `sync_primary_sidecar_role` (`src/sase/_sidecar_auto_sync.py`), immediately before
     `refresh_clean_linked_checkout`.
  3. The hidden-clone maintenance pass in
     `src/sase/scripts/sase_chop_sidecar_auto_sync.py`
     (`_maintain_hidden_sidecar_clones`), once per hidden clone.
  4. Every successful SDD clone materialization path in
     `src/sase/sdd/_store_clone_*.py`, at least the remote-clone success path in
     `_store_clone_remote.py`. Find the other creation paths with grep.

## Tests (new file `tests/sdd_store/test_repository_object_repair.py`, plus focused additions)

Use the real-git helpers in `tests/sdd_store/_helpers.py` and
`_repository_transaction_helpers.py`.

To simulate crash damage deterministically, commit in a clone, push it so the remote has
the objects, and then truncate that clone's own loose objects for the commit, its root
tree, and a blob (`chmod u+w` first). Truncate the worktree copy of the blob's file too.

1. Truncated commit, tree, and blob, with the remote advanced past the damaged tip:
   `integrate_machine_managed_sdd_repository(clone, recovery_cooldown_seconds=0)`
   returns `RECOVERED`. Afterward:
   - `git fsck --connectivity-only` is clean.
   - HEAD equals the upstream.
   - The truncated worktree file is restored.
   - No zero-byte files remain under `objects/`.
   - The manifest lists the objects with `status: repaired`.
2. The same damage with the remote **not** advanced (the remote tip equals the missing
   local tip) also returns `RECOVERED`.
3. Blob-only damage (commit and tree intact) returns `RECOVERED`. This proves the repair
   uses `--refetch`; a plain fetch would leave the blob missing.
4. An injected `fetch --refetch` failure through a wrapping runner returns
   `UNRECOVERABLE`:
   - The error contains "unclean shutdown", the quarantine directory, and the injected
     failure.
   - The manifest is `pending`, and the branch ref is unchanged.
   - A second call with a working runner repairs the clone through the pending manifest,
     even though no zero-byte files remain.
5. A truncated **unpushed** local commit fails closed as `UNRECOVERABLE` with the
   "existed only in this clone" detail. No reset happens and no recovery ref is created.
6. An upstream whose remote is `"."` fails closed without fetching.
7. A healthy clone never enters the repair path: a spy runner sees no
   `--refetch`/`rev-list --objects` call.
8. `ensure_sdd_clone_durability`:
   - Sets both local keys when they are unset everywhere.
   - Leaves them alone when a value is set through a temporary `GIT_CONFIG_GLOBAL` or
     locally.
   - Returns without raising when the path is not a repo.
   - One spy test for each call site in `pull_sdd_clone` and `sync_primary_sidecar_role`
     confirms the helper is invoked and that a raising helper does not change the
     outcome.

## Docs

Read the `docs_writing` reference memory first. In `docs/contribute/sdd-storage.md`:

- In the paragraph that starts "Managed Git commands disable `rerere`…", add that SASE
  sets repo-local `core.fsync=committed,index,reference` and `core.fsyncMethod=batch` on
  SDD clones unless a value is already configured.
- In the "Machine-managed disposable sidecar clones have one additional recovery path…"
  paragraph, describe the crash-damage repair: zero-byte loose objects are quarantined
  under `.git/sase-object-quarantine/`, the store is refetched with
  `git fetch --refetch`, and the repair is verified before the usual snapshot and reset.
  Damage that upstream cannot repair fails closed.

## Live repair of the bob-cli plans clone

Damaged clone: `/home/bryan/projects/github/bobs-org/bob-cli/sase/repos/plans` (bob-cli
primary checkout, `plans` sidecar).

The one-shot repair command runs the **new** code from the coder's workspace venv. For
example, `uv run python -c` calls
`sase.sdd._store_integration.pull_sdd_clone(Path(<clone>), fresh=True)`, prints its
result, and then prints:

- `git -C <clone> fsck --connectivity-only`
- `git -C <clone> rev-parse HEAD origin/main`
- `git -C <clone> status --porcelain`
- `git -C <clone> for-each-ref refs/sase/recovery`

Expected result:

- `pull_sdd_clone` returns `True`, with outcome `RECOVERED`.
- fsck is clean, and HEAD equals `origin/main`.
- `202611/bank_badge_strip_family.md` and `.plan-decisions.json` are non-empty again.
- One new recovery ref holds the two zero-byte worktree files.
- `.git/config` now has the two fsync keys.

> [!decision] live_repair = gate After `just check` passes, propose the repair command
> through `/sase_gate` with a short explanation (lossless: all 9 objects are upstream;
> only the two crash-truncated files differ). Then finish the turn. Do not run it
> yourself.

> [!decision] live_repair = run After `just check` passes, run the repair command
> yourself and include its output in your final report. If the outcome is not
> `RECOVERED`, stop. Do not attempt any manual git surgery on that clone. Report the
> detail instead.

> [!decision] live_repair = defer Make no live change. In the final report, state that
> the `sdd-sidecar` routine will self-heal the clone on its first pass after the user
> reinstalls sase (global install is human-only), and that the hourly digest entry
> continues until then.

> [!decision] global_fsync Open the chezmoi repo with `/sase_repo`
> (`sase repo open chezmoi -r "<reason>"`), and add `fsync = committed,index,reference`
> and `fsyncMethod = batch` under `[core]` in the chezmoi-managed global gitconfig
> source, keeping that file's existing style. Do not run `chezmoi apply`; mention that
> the user applies it. The repo becomes a commit obligation for `/sase_final`.

## Follow-up to file (do not fix here)

While investigating, planning found a nested clone at
`bob-cli/sase/repos/plans/sase/repos/plans`. It is ignored through the plans repo's
`/sase/repos/` ignore rule, but it was still being fetched and rebased at 06:53 on
2026-11-14, which means some process still resolves the plans clone itself as a project
root. Use `/sase_new_task` to file a `bug` task bead for it, or corroborate an existing
one. Include the path, the reflog evidence, and the related test
`tests/sdd_store/test_nested_repo_open_sdd_root.py`.

## Verification

- `sase tool run check` (the wrapped `just check`) passes. Do not run `check-full`.
- The new tests fail on today's code, by reverting the wiring locally, and pass with it.
- The `live_repair` branch has been carried out as decided.

## Out of scope

- Repairing corrupt **pack** files. Packs are fsynced by default, and the scan found
  none.
- Cleaning NUL padding out of reflogs. Git tolerates it.
- Making the conservative chop sync (`sync_primary_sidecar_role`) self-heal. It stays
  conservative; the machine-managed recovery path owns repair.
- Porting SDD recovery into `sase-core`.
