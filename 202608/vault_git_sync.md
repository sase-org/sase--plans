---
tier: epic
title: Make the bobs-org/bob GitHub repo the only Bob vault sync channel
goal: "The Bob Obsidian vault stays converged between athena and the MacBook through git
  alone, driven by a new `bob vault-sync` command that pulls, merges, commits, and
  pushes unattended without ever leaving conflict markers or a halted merge; Obsidian
  Sync is switched off on both devices; and large PDF trees that must not enter the repo
  reach the MacBook over an explicit rsync bridge invoked by a new configurable `bob
  highlights` pre-scan command.

  "
phases:
  - id: reconcile
    title: Converge both vaults and fix what the first `git add -A` would sweep in
    depends_on: []
    size: medium
    description: "reconcile: back up both vaults, close the five remaining
      athena/MacBook differences, gitignore `lit_review/` and `xlib/`, add
      `.gitattributes`, run `git gc`, and land the 206-path commit backlog so both
      machines start from one known SHA.

      "
  - id: vaultsync
    title: Implement `bob vault-sync` and its conflict-copy policy
    depends_on: []
    size: medium
    description: "vaultsync: add the `bob vault-sync run|status` subcommand implementing
      the lock-protected reconcile cycle, quarantined conflict copies under
      `_conflicts/`, the 95 MiB preflight guard, interrupted-merge recovery, and a
      machine-readable status record.

      "
  - id: retire
    title: Retire `bulk-git-commit` and rewire `bob nightly`
    depends_on:
      - vaultsync
    size: small
    description: "retire: delete the `bulk-git-commit` subcommand and its `bob_sync`
      script and binary aliases, drop the `ob sync` gate from `bob nightly`, and make
      nightly run vault-sync, move-done-tasks, vault-sync.

      "
  - id: highlights
    title: Add a configurable pre-scan command to `bob highlights`
    depends_on: []
    size: medium
    description: "highlights: teach `bob highlights scan` to run a configured command
      before it inspects the library, wired through `~/.config/bob/config.yml` and an
      environment override, so an off-band PDF bridge can deliver `xlib/` intake.

      "
  - id: machines
    title: Provision credentials, triggers, and the MacBook git clone
    depends_on:
      - reconcile
      - retire
      - highlights
    size: medium
    description: "machines: create a passphraseless deploy key for athena, add SSH
      ControlMaster, ship the chezmoi rsync bridge and config, convert the MacBook's
      `~/bob` into a clone in place, and install the systemd and launchd triggers
      without enabling them.

      "
  - id: cutover
    title: Prove the acceptance matrix end to end, then cut over
    depends_on:
      - machines
    size: medium
    description:
      "cutover: run every acceptance test across both real machines, disable Obsidian
      Sync on athena and the MacBook, restore nightly maintenance, write the runbook,
      and supersede the competing Obsidian Sync restoration epic."
proposed_by: bbugyi200.athena.0ez
create_time: 2026-09-09 20:00:35
status: wip
---

- **PROMPT:**
  [prompts/202608/vault_git_sync.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/vault_git_sync.md)

# Plan: Make the bobs-org/bob GitHub repo the only Bob vault sync channel

## Outcome

Replace Obsidian Sync with git. A new `bob vault-sync` command owns one idempotent
reconcile cycle (stage → commit → fetch → merge → resolve → push) and runs on both
machines from a platform-native trigger. Every conflict resolves automatically as a
quarantined conflict copy, so the channel never stalls and no `<<<<<<<` marker ever
lands in a note the user has open. Obsidian Sync is stopped on both devices and the
subscription can be cancelled after a soak.

Two things stay off the git channel by explicit decision: the `lit_review/` PDF library
and the `xlib/` staging directory. `xlib/` reaches the MacBook over an rsync bridge that
a new configurable `bob highlights` pre-scan command invokes, which is the behavior
requested in the annotations on the research report.

## Source material

- Research: `research:202608/obsidian_vault_git_sync/obsidian_vault_git_sync.md`
- User annotations: `~/bob/ref/chat/obsidian_vault_git_sync.md` (five comments)

The research is the design's backbone. This plan overrides it in four places, each
called out below with the measurement that settled it.

## Verified state, 2026-08-27 ~12:40 EDT

Everything here was measured on the live machines while writing this plan. Several facts
contradict the research report, which was written earlier the same day.

### The two vaults are already converged

A full cross-machine inventory (6,173 files on athena, 6,263 on the MacBook, excluding
`.git/`, `old_lib/`, `.sase/`) found **exactly five differences**:

| #   | Path                                                        | State                                                                                  |
| --- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 1   | `lit_review/**` (91 files, ~430 MB)                         | MacBook only                                                                           |
| 2   | `xlib/chat/ace_refresh_loop_and_link_rail_regression.pdf`   | athena only; **stale** — `lib/chat/` already holds the processed copy on both machines |
| 3   | `lib/chat/obsidian_vault_git_sync.pdf`                      | differs; the MacBook copy carries the user's annotations                               |
| 4   | `bob_git.md`                                                | differs; the MacBook copy is newer (user is editing it)                                |
| 5   | `.obsidian/plugins/block-id-prompt/{main.js,manifest.json}` | differs; custom plugin, gitignored                                                     |

All 5,518 `.md` files outside `old_lib/` are **byte-identical** on both machines, as are
all 67 tracked `.obsidian/` config files apart from row 5. `old_lib/` matches on both
(700 files, 1,731,114,877 bytes) per commit `aedc9f5`.

**This voids research §1.1 and §7.2 step 0.** There is no hand reconciliation of a
diverged vault to perform; there is a five-item checklist. Epic `bob-cli-1l` phases `.1`
and `.2` already did that work.

### All automation is already stopped on athena

- `ob-sync-bob.service`: `inactive (dead)` since 10:23 EDT today.
- The 03:30 `bob nightly` crontab line is commented out by `bob-cli-1l.2`'s quiesce.
  Restore text is preserved at `/tmp/crontab_bak_bob-cli-1l2_20260827T142313Z.txt` —
  **copy it somewhere durable before `/tmp` is cleared.**
- `~/bob` is on `master`, 0 ahead / 0 behind `origin/master` (`aedc9f5`), with 206 dirty
  paths.

The MacBook still runs three cron jobs every 15 minutes: `maybe_bob_highlights_sync -w`,
`bob projects sync`, and `bob task-status-hooks`. These mutate the vault and must be
accounted for by the sync loop.

### `lit_review/` is a bigger landmine than `xlib/`

The research flagged `xlib/` as the untracked-and-un-ignored directory the first
`git add -A .` would sweep in. It missed the much larger one:

| Path               | athena            | MacBook            | Tracked? | Ignored? |
| ------------------ | ----------------- | ------------------ | -------- | -------- |
| `lit_review/`      | 722 MB / 185 PDFs | 1.15 GB / 276 PDFs | no       | **no**   |
| `xlib/`            | 141 KB / 1 PDF    | absent             | no       | **no**   |
| `lib/**` untracked | ~66 MB            | same               | no       | no       |

`git check-ignore -v lit_review/vibe_coding_101_1.pdf` returns `.gitignore:23:!*.pdf`,
confirming it is un-ignored. The GitHub repo is already **1,247,309 KB (~1.19 GB)** per
the GitHub API, so committing `lit_review/` would roughly double an already-oversized
repo.

### Credentials: athena is broken for unattended use, the MacBook is not

```text
athena:   env -i git ls-remote git@github.com:bobs-org/bob.git
          → git@ssh.github.com: Permission denied (publickey)
MacBook:  env -i git ls-remote git@github.com:bobs-org/bob.git
          → aedc9f5e7ad01c006b2fa00c9038869bb4d96c4e  refs/heads/master
MacBook:  env -i ssh -o BatchMode=yes home 'hostname; ls ~/bob/xlib'
          → athena / chat
```

All three athena keys (`id_ed25519`, `id_rsa`, `id_internal`) are passphrase-protected.
athena's GitHub access depends entirely on an ssh-agent captured in
`~/.ssh-agent-thing`, which is why `ob.rs::source_ssh_agent_env()` exists. **After a
reboot no agent holds an unlocked key until the user logs in** — precisely the
`pass_git_sync` failure mode the research warned against inheriting. The MacBook's
`~/.ssh/id_rsa` is already passphraseless and authenticates as `bbugyi200`, so launchd
needs nothing extra there.

### Measured cycle cost (athena, live vault)

| Operation                       | Measured       | Note                                                   |
| ------------------------------- | -------------- | ------------------------------------------------------ |
| `git status --porcelain`        | **32 ms**      |                                                        |
| `git ls-remote origin master`   | **507–539 ms** | no ControlMaster yet                                   |
| `git add -A --dry-run .`        | **2,193 ms**   | dominated by hashing 722 MB of untracked `lit_review/` |
| `bob highlights scan --dry-run` | **7.7 s**      | 113 PDFs in `lib/`                                     |

The research measured `git add -A` at 20 ms and built the cycle around running it every
pass. That number no longer holds. **The cycle must gate `git add -A` behind
`git status --porcelain`** (see `vaultsync` step 3).

### Platform inventory

|                        | athena                                  | MacBook                                                                        |
| ---------------------- | --------------------------------------- | ------------------------------------------------------------------------------ |
| OS                     | Debian 13, `Port 34857` sshd            | macOS 26.5, arm64                                                              |
| Reachable as           | `ssh home` from the Mac (192.168.1.156) | `ssh mac` from athena                                                          |
| `~/bob` is a git repo  | yes                                     | **no**                                                                         |
| Free disk              | 384 GB                                  | 107 GB                                                                         |
| `bob` / `cargo`        | yes                                     | `~/.cargo/bin/bob`, cargo 1.95                                                 |
| `bob-cli` checkout     | this repo                               | `~/projects/github/bbugyi200/bob-cli`                                          |
| `bob-plugins` checkout | yes                                     | **no**                                                                         |
| chezmoi                | yes                                     | `/opt/homebrew/bin/chezmoi` + `~/.local/share/chezmoi` (not on non-login PATH) |
| `inotifywait`          | **no** (needs `inotify-tools`)          | n/a                                                                            |
| `fswatch`              | n/a                                     | **no**                                                                         |
| Obsidian               | snap 1.13.7, rarely run                 | 1.12.7, **running now**                                                        |
| git identity           | Bryan Bugyi / bryanbugyi34@gmail.com    | same                                                                           |
| `core.excludesfile`    | `~/.gitignore_global`                   | byte-identical file                                                            |

No vault filename contains a non-ASCII byte and there are no case-only filename
collisions, so converting the MacBook's `~/bob` into a clone on APFS is safe.

## Decisions

Each of these is a decision the epic will act on. Overrule any of them at approval time
and the affected phase changes accordingly.

**D1 — `lit_review/` is gitignored, not committed.** 722 MB–1.15 GB of PDFs would
roughly double a 1.19 GB repo, and the annotations say PDFs should not go into the repo.
`reconcile` gitignores it and rsyncs the 91 MacBook-only files to athena once so both
machines hold the same 276. Future `lit_review/` growth is then out of band; `cutover`
documents that.

**D2 — `xlib/` is gitignored and bridged by rsync.** This is the annotated instruction.
`highlights` adds the pre-scan hook; `machines` ships the script.

**D3 —
`lib/**`PDFs stay on the git channel.** 113 PDFs on athena, ~66 MB of them untracked today, and 23 already tracked. They are the Highlights pipeline's inputs, their`.md`sidecars carry a`source_pdf_sha256`that must match, and Obsidian embeds them in`ref/`
notes. Untracking the 23 already-tracked ones would delete them from the MacBook on its
first pull. This is a deliberate, bounded exception to "no PDFs in the repo"; a broader
PDF-tracking policy is filed as a follow-up rather than attempted mid-cutover.

**D4 — the six custom `bob-*` plugins stay gitignored.** Per the annotations, manual
`bob plugins sync` on the MacBook is acceptable. This overrides research §7.1, which
recommended tracking the built artifacts. `machines` clones `bob-plugins` to
`~/projects/github/bobs-org/bob-plugins` on the MacBook so the command can run there.

**D5 — no sparse-checkout on the MacBook.** Research §6.3 spends significant effort on a
`--no-cone` recipe to keep `old_lib/` off the laptop. `old_lib/` is _already_ on the
laptop — 700 files, 1.6 GB, and the MacBook was the source of truth for 40 of them.
Sparse-checkout would therefore be a deletion decision, not an avoidance, and it adds
skip-worktree semantics underneath a daemon that runs `git add -A`. With 107 GB free, a
full clone is simpler and strictly safer.

**D6 — conflict copies are quarantined under `_conflicts/`.** Verified still true today:
the Tasks plugin's `globalQuery` is empty, Dataview has no folder exclusions, and
`dash.md` runs unscoped vault-wide queries. All three surfaces get closed in
`vaultsync`/`machines`.

**D7 — this epic supersedes `bob-cli-1l`.** That in-progress epic ("Restore Bob Obsidian
Sync and establish a sub-1 GB footprint policy") is trying to repair the channel this
one removes. Its phases `.1` and `.2` are done and their output is reused. `cutover`
closes `bob-cli-1l.3/.4/.5` and the epic bead with resolution `superseded`.

**D8 — `old_lib/` is left exactly as it is.** Tracked, 698 files in the index, with
`old_lib/docs/{vim_help_netrw,nvim_help_lua_intro}.pdf` gitignored for exceeding
GitHub's 100 MiB limit. Off-site coverage for those two is already tracked by
`bob-cli-1k`.

**D9 — athena watches, the MacBook polls.** athena gets a systemd user service running
`inotifywait -t 15` (one blocking wait that returns on either a file event or the
timeout, giving edge-triggered push and a 15-second poll from one process). The MacBook
gets a launchd LaunchAgent with `StartInterval 15` and **no watcher**. The research
recommended `fswatch` under `KeepAlive` on the Mac; `StartInterval` needs no Homebrew
dependency, has no resident process to leak, and launchd already handles sleep/wake and
missed intervals natively. The policy lives in one place — the Rust command — and only
the trigger differs. If a 15-second outbound delay from the MacBook ever becomes
annoying, adding `fswatch` is a follow-up, not a redesign.

## Non-goals

- Git LFS, or any rewrite of the 1.19 GB of existing history.
- Moving `old_lib/` out of the vault.
- Mobile Obsidian. `.obsidian/workspace-mobile.json` does not exist in either vault; if
  a phone enters the picture this design does not serve it.
- A general PDF-tracking policy for the vault repo (follow-up, per D3).
- Editing `sase/memory/obsidian.md`. It documents the Obsidian Sync topology and will be
  stale after this epic, but per `CLAUDE.md` a plan file does not authorize a SASE
  memory edit. `cutover` records this as a task needing the user's explicit approval,
  and `bob-cli-1h` already tracks the note.

---

## Phase `reconcile`: Converge both vaults and fix what the first `git add -A` would sweep in

Bring both machines to a single, fully-committed SHA and make the ignore rules correct
**before** any daemon exists to act on them.

Do this with all sync automation stopped, which is already the case on athena. Stop the
MacBook's three cron jobs for the duration (`crontab -l > backup`, comment out, restore
at the end) so nothing mutates the vault mid-reconcile.

1. **Durable backups first.** Snapshot both vaults outside git — for example
   `rsync -a --delete ~/bob/ /home/bryan/var/backups/bob-pre-gitsync-<date>/` on athena
   and the equivalent on the MacBook — and copy
   `/tmp/crontab_bak_bob-cli-1l2_20260827T142313Z.txt` out of `/tmp`. Record both paths
   on the phase bead. Obsidian Sync is **not** a valid rollback target; its remote is
   over quota.
2. **Close the five differences** listed in "The two vaults are already converged":
   - Delete athena's stale `xlib/chat/ace_refresh_loop_and_link_rail_regression.pdf`
     after confirming `lib/chat/` holds the processed copy on both machines. Today this
     file makes `bob highlights scan` exit 1 with an intake collision.
   - Copy the MacBook's annotated `lib/chat/obsidian_vault_git_sync.pdf` and its
     `bob_git.md` to athena. The MacBook is the source of truth for both.
   - Rsync the 91 MacBook-only `lit_review/**` PDFs to athena so both hold 276.
   - Leave `.obsidian/plugins/block-id-prompt/` alone; D4 keeps it off git and
     `machines` gives the MacBook the means to resync it. Re-run the full inventory
     comparison afterwards and require zero differences outside the gitignored set.
3. **Fix `.gitignore`.** Add `lit_review/` and `xlib/` with comments explaining that
   each is carried out of band, not lost. Then prove the result:
   `git status --porcelain | grep '^??'` must list no path under either, and
   `git add -A --dry-run .` must not name a `lit_review/` or `xlib/` file.
4. **Add `.gitattributes`** at the vault root, per research §5.6, so an unattended merge
   can never attempt a textual merge on a binary:

   ```gitattributes
   * text=auto eol=lf
   *.png binary
   *.jpg binary
   *.jpeg binary
   *.gif binary
   *.webp binary
   *.pdf binary
   *.PDF binary
   *.mp3 binary
   *.wav binary
   *.m4a binary
   *.flac binary
   *.ogg binary
   *.mp4 binary
   *.mov binary
   *.m4v binary
   *.webm binary
   *.avi binary
   *.mkv binary
   *.xmind binary
   ```

   `*.PDF` matters — commit `ac31423` exists because the case variant is real here.
   After adding it, confirm `git status` does not suddenly report mass modifications
   from the `text=auto eol=lf` line; every file in the vault is already LF.

5. **`git gc`** before the MacBook clones anything. `git count-objects -vH` reports
   7,566 loose objects / 565 MiB alongside 689 MiB in two packs.
6. **Commit and push the backlog.** Land the 206 dirty paths plus the newly-swept
   untracked content (~66 MB of `lib/` PDFs, `img/20260827_070810.png`, the `.md`
   backlog) in a small number of well-messaged commits. Do not use
   `bob bulk-git-commit`; it is being deleted. Finish with `git push` and confirm
   `git rev-list --left-right --count HEAD...origin/master` is `0 0`.
7. **Restore the MacBook's crontab.**

Done when: both vaults' non-ignored file sets are identical, `~/bob` on athena is clean
and level with `origin/master`, `.gitignore` and `.gitattributes` are committed, and
both backup paths are recorded on the bead.

## Phase `vaultsync`: Implement `bob vault-sync` and its conflict-copy policy

Add `src/native/vault_sync.rs` and register `vault-sync` in `src/native.rs` and the
`SUBCOMMANDS` table in `src/runner.rs` (keep that table alphabetically sorted; the
`subcommands_are_sorted` test guards it). Reuse `ob::acquire_lock()`, `ob::child_env()`,
and `ob::git_command()` rather than reinventing them.

### CLI surface

```text
bob vault-sync [run] [OPTIONS]     run one reconcile cycle (default subcommand)
bob vault-sync status [OPTIONS]    report the last cycle's outcome
```

Follow `sase/memory/cli_rules.md`: alphabetically-sorted subcommands and options, a
short alias for every public long option, and colored output when it aids reading (reuse
`native::style::Styler`, which already auto-disables color off a TTY and under
`NO_COLOR`).

| Option                    | Applies to | Meaning                                                           |
| ------------------------- | ---------- | ----------------------------------------------------------------- |
| `-h, --help`              | both       |                                                                   |
| `-j, --json`              | `status`   | machine-readable record                                           |
| `-m, --message <MESSAGE>` | `run`      | override the generated commit message                             |
| `-n, --dry-run`           | `run`      | report the cycle without staging, committing, merging, or pushing |
| `-q, --quiet`             | `run`      | suppress per-step logging; errors and conflicts still print       |

Environment: `BOB_DIR`, `BOB_VAULT_SYNC_LOCK_FILE` (defaulting to the existing
`${XDG_RUNTIME_DIR:-/tmp}/bob_sync.lock` so it stays the _same_ lock `bob nightly`
takes), and `BOB_VAULT_SYNC_STATE_FILE`.

### The cycle

Numbered so acceptance tests can name a step. This is research §5.1 with step 3
corrected for the measured `git add -A` cost.

0. If `.git/MERGE_HEAD`, `.git/rebase-merge/`, `.git/rebase-apply/`, or
   `.git/CHERRY_PICK_HEAD` exists, abort that operation, log loudly at warning level,
   record it in the status file, and **continue**. Do not refuse and wait for a human: a
   SIGKILL, an OOM, or a closed lid leaves the worktree mid-merge, and step 9 guarantees
   the next pass can finish without help.
1. Acquire the lock non-blocking. If held, exit 0 silently — another cycle or
   `bob nightly` owns the vault.
2. Verify `bob_dir` is a git worktree (reuse `sync::verify_bob_worktree`).
3. `git status --porcelain` (32 ms). **Only if it reports changes** run the size
   preflight and then `git add -A .`. Skipping `add -A` on an idle vault is what keeps
   the idle cycle at ~250 ms instead of ~2.2 s.
4. Preflight: refuse the cycle if any path about to be staged is ≥ 95 MiB, and warn at ≥
   50 MiB. Fail locally with the path and size rather than letting GitHub reject the
   push at 100 MiB.
5. If anything is staged, commit. Message format — the annotations ask for "a good
   commit message", so replace `bob bulk-git-commit <date>`:

   ```text
   vault(athena): 6 files — 2026/20260827.md, bob.md, dev.md (+3)

   M  2026/20260827.md
   M  bob.md
   ...
   ```

   Subject: `vault(<hostname>): <n> file(s) — <up to 3 paths>[ (+k)]`. Body: the
   `git diff --cached --name-status` list, capped at 30 lines with a `… and N more`
   tail. `-m/--message` replaces the whole thing.

6. `git ls-remote origin master`; if its SHA equals the cached `origin/master`, skip the
   fetch entirely. Otherwise `git fetch --no-tags origin master`.
7. If `HEAD == origin/master`, go to step 10. If `HEAD` is an ancestor of
   `origin/master`, `git merge --ff-only origin/master` and go to step 10.
8. `git merge --no-edit origin/master`. **Merge, never rebase**: a failed rebase leaves
   a partially-replayed detached HEAD with an unbounded number of remaining conflicts,
   which is the worst possible thing to hand a running Obsidian. A merge has one
   conflict point, one resolution pass, one commit.
9. On conflict, resolve every conflicted path with no human input, then `git add` and
   commit the merge:

   | Conflict class                                 | Action                                                                                                                                                               |
   | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | Both-modified text (`.md`, `.canvas`, `.base`) | `git checkout --theirs` into place; write the local (stage 2) version to `_conflicts/<relpath>.<host>-<ISO8601>.<ext>`                                               |
   | Both-added (`AA`, no common ancestor)          | Same shape; `git checkout --theirs` is verified to extract stage 3 correctly here. This is the common daily-note case where both machines create `2026/YYYYMMDD.md`. |
   | Binary                                         | Same shape, whole file. Matches Obsidian Sync's "last modified wins" for non-markdown.                                                                               |
   | Delete/modify                                  | **Keep the file.** Never let an unattended daemon win a delete race. Log it.                                                                                         |
   | Rename/rename, or anything unhandled           | Abort the merge, record `conflict_state` in the status file, `bob notify`, and exit non-zero. Do not guess.                                                          |

   Remote wins in place because the remote version is what the _other_ machine already
   considers committed, while the local version is the one the user is sitting in front
   of and can most easily re-apply. Nothing is ever lost. This is Obsidian Sync's own
   "create conflict file" strategy.

   Then append one line per conflict to `_conflicts/sync_conflicts.md` and fire
   `bob notify`.

   **Never** use `reset --hard`, `push --force`, `-X ours`, `-X theirs`, or
   `merge=union`. `merge=union` silently concatenates both sides of a hunk, which on
   this vault means duplicated task lines, duplicated `^blockid`s, and double-counted
   Pomodoros. A conflict you can see beats a corruption you cannot.

10. `git push`. On a non-fast-forward rejection, go back to step 6, bounded to three
    retries with a short backoff, then fail with a clear message.
11. Write the status record.

### Status record

`${XDG_STATE_HOME:-$HOME/.local/state}/bob-cli/vault-sync.json`, updated every cycle:
`last_attempt_at`, `last_success_at`, `local_sha`, `remote_sha`, `files_committed`,
`push_retries`, `duration_ms`, `conflicts` (list of quarantined paths from the last
cycle), `interrupted_merge_recovered` (bool), and `last_error`. `bob vault-sync status`
renders it as a short human panel, or as JSON under `-j`. The design intent is that "are
my notes current?" is answerable without reading journals or git internals — the kind of
question that would have surfaced the Obsidian Sync outage in minutes instead of hours.

### Quarantine plumbing

Add `_conflicts` to `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES` in `src/native.rs` so
`move-done-tasks`, `projects`, `note_tasks`, `capture`, and the dataview walkers all
skip it. This is the exclusion that cannot be fixed from plugin settings and the one
that matters most, because `move-done-tasks` mutates what it finds. Add a test that
walks a fixture vault containing `_conflicts/` and asserts nothing under it is seen.

### Tests

Unit and integration tests in `tests/` against scratch bare repos and two clones — never
the real vault:

- No-change cycle creates no commit and no push.
- Local-only change commits and pushes.
- Remote-only change fast-forwards.
- Non-overlapping edits to one file merge cleanly.
- Same-line edits produce a conflict copy under `_conflicts/`, the working file matches
  the remote version, and **no file contains `<<<<<<<`**.
- Both machines create the same new file (`AA`) → conflict copy.
- Delete-versus-modify keeps the file.
- A binary changed on both sides produces an uncorrupted conflict copy.
- A ≥ 95 MiB file is refused locally, before any push.
- A worktree left mid-merge is recovered by the next cycle.
- A push race retries and succeeds without intervention.
- A second concurrent invocation exits 0 without touching the repo.

Run `just all` (fmt, clippy, test) before the phase is done.

## Phase `retire`: Retire `bulk-git-commit` and rewire `bob nightly`

The annotation is explicit: _"Let's get rid of this command in favor of the
`bob vault-sync` command."_ `bulk-git-commit` never fetches, which was safe only because
Obsidian Sync converged the tree first; keeping it alive after cutover would be a
correctness bug.

1. Delete `src/native/sync.rs`, the `NativeCommand::BulkGitCommit` variant, its
   `SUBCOMMANDS` entry, the `"bob_sync" => BulkGitCommit` mapping in
   `command_for_script`, `src/bin/bob_sync.rs`, `scripts/bob_sync`, its
   `scripts.rs`/`embedded_assets` registration, and its `justfile` `check-scripts` and
   `install-smoke` lines. Add a `vault-sync` `install-smoke` line.
2. Retire the `BOB_BULK_GIT_COMMIT_*` and `BOB_SYNC_*` environment names in favor of
   `BOB_VAULT_SYNC_*`. This is a single-user tool; make it a clean break rather than
   carrying another generation of deprecated aliases, and say so in the help text. Keep
   the lock _path_ default unchanged so the mutual-exclusion gate is preserved.
3. Rewrite `bob nightly` (`src/native/nightly.rs`): drop `run_sync_gate` and the
   `ob::sync_vault` call entirely, and make the step list **`vault-sync` →
   `move-done-tasks` → `vault-sync`**. The leading vault-sync pulls the MacBook's day
   before maintenance rewrites task lines; the trailing one publishes the maintenance
   commit. Keep the framed section output and summary.
4. Prune `src/native/ob.rs` down to what survives: `acquire_lock`, `child_env`,
   `git_command`, and `ChildEnv`. Remove `sync_vault`, `run_sync_status`,
   `load_ob_command`, `load_ob_from_nvm`, `SyncOutcome`, and the
   `SYNC_ALREADY_RUNNING_MESSAGE` constant. Renaming the module is optional; if it is
   renamed, do it in one mechanical commit.
5. Update `docs/` and `README.md` wherever `bulk-git-commit` or `bob_sync` appears.
6. `just all` must pass, and `bob nightly --help` must no longer mention `ob sync`.

Note for the worker: the chezmoi copy of the `bob_sync` shim
(`home/bin/executable_bob_sync`) is removed in `machines`, not here, because that repo
must be opened through `/sase_repo`.

## Phase `highlights`: Add a configurable pre-scan command to `bob highlights`

From the annotations: _"Let's add support for a configurable command that the
`bob highlights` command will use, if configured, prior to searching the `~/bob/lib/`
directory."_

`src/native/highlights_ref/mod.rs::scan_library()` already runs `plan_xlib_intake()` →
`execute_xlib_intake()` → `collect_pdf_paths()`. Insert the hook at the very top of
`scan_library`, **before** `plan_xlib_intake`, so anything the command drops into
`xlib/` is picked up by the existing intake machinery. Reusing `execute_xlib_intake` for
the actual `xlib/` → `lib/` move is deliberate: it already handles companion files
(`.md` sidecars, textbundles) and pre-flight collision detection, which a shell script
would not.

### Configuration

Extend `src/native/config.rs` — which today parses only `properties:` from
`~/.config/bob/config.yml` — with an optional block:

```yaml
highlights:
  # Runs from the vault root before `bob highlights scan` inspects the library.
  # Use it to deliver PDFs that do not travel on the git sync channel.
  pre_scan_command: bob_xlib_pull
```

- Environment override: `BOB_HIGHLIGHTS_PRE_SCAN_COMMAND`. An empty value disables the
  hook even when the file configures one.
- Executed with `sh -c <command>` from `bob_dir`, inheriting stdout/stderr so its output
  lands in the cron log.
- **A non-zero exit is a hard error**: print the exit code and abort `scan`. A silent
  failure would mean PDFs quietly never arrive. The script owns the "remote is
  unreachable, that's fine" policy and exits 0 in that case (see `machines`).
- Skipped entirely under `--dry-run`; `--dry-run` reports that it _would_ run and with
  what command.
- `bob highlights doctor` prints the configured command and whether it is executable,
  but does not run it.
- Keep the parser tolerant: an unknown top-level key in `config.yml` must not break the
  existing `properties:` path, and vice versa.

Tests: config parsing (present, absent, env override, empty override), the hook running
before intake, a non-zero exit aborting the scan with the command's output visible, and
`--dry-run` not executing it.

## Phase `machines`: Provision credentials, triggers, and the MacBook git clone

Everything here is deployment. **Install the triggers but leave them disabled** —
`cutover` turns them on. Open the chezmoi repo with `/sase_repo` before touching it.

### 1. Fix athena's GitHub credentials (blocking)

`env -i git ls-remote` fails today. Without this, the systemd service dies on every
reboot until the user logs in — the exact anti-pattern the research called out.

```bash
ssh-keygen -t ed25519 -N '' -C 'bob-vault-sync@athena' -f ~/.ssh/id_bob_vault
```

Register the public half as a **deploy key with write access** on `bobs-org/bob`
(`gh repo deploy-key add`), scoping it to that one repository. Add a dedicated host
alias to athena's `~/.ssh/config` and point the vault remote at it, leaving the existing
`Host github.com` stanza (which uses `ssh.github.com:443`) untouched:

```sshconfig
Host github-bob
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_bob_vault
  IdentitiesOnly yes
  AddKeysToAgent no
  ControlMaster auto
  ControlPath ~/.ssh/cm-%r@%h:%p
  ControlPersist 10m
```

```bash
git -C ~/bob remote set-url origin git@github-bob:bobs-org/bob.git
```

**Acceptance:**
`env -i HOME=/home/bryan PATH=/usr/bin:/bin git -C ~/bob ls-remote origin master` prints
the master SHA. Note that `~/.ssh/config` is _not_ chezmoi-managed on either machine, so
this edit is machine-local; record it in the runbook.

Add the same `ControlMaster`/`ControlPath`/`ControlPersist` three lines to the MacBook's
`github.com` handling. The measured win is roughly 2× on poll cost (507 ms → ~220 ms).
Do not add an `ensure_usable_ssh_agent`-style probe anywhere; that is the
`pass_git_sync` failure mode this design exists to avoid.

### 2. Convert the MacBook's `~/bob` into a clone, in place

Do not delete and re-clone; the MacBook holds 2.9 GB including `lit_review/` and
`old_lib/`, and it is the source of truth. Convert without touching a single file:

```bash
cd ~/bob
git init -b master                                    # unborn branch; touches no file
git remote add origin git@github.com:bobs-org/bob.git # sets the standard fetch refspec
git fetch origin master                               # creates refs/remotes/origin/master
git reset --mixed FETCH_HEAD                          # sets HEAD + index; worktree untouched
git branch --set-upstream-to=origin/master master
```

`reset --mixed` sets the index from the remote tree and leaves the worktree untouched,
so nothing is overwritten. Then `git status --porcelain` should be empty or contain only
genuine MacBook edits made since `reconcile` ran — `reconcile` already converged the
content, `.gitignore` already covers `lit_review/` and `xlib/`, and the six custom
plugin directories are ignored. Commit any such edits normally. Treat anything else as a
real divergence and investigate it; never `git checkout --` over it.

Confirm `git config core.precomposeunicode` is `true` (git sets it on macOS) and verify
`git rev-parse HEAD` matches athena's.

### 3. Ship the chezmoi changes

In the chezmoi repo (`/sase_repo` → linked `chezmoi`):

- **`home/bin/executable_bob_xlib_pull`** — the rsync bridge, invoked as
  `pre_scan_command` on the MacBook. It must:
  1. Exit 0 immediately if it is not running on the MacBook, or if `ssh home` is
     unreachable within a short `ConnectTimeout` (roaming laptop; unreachable is normal,
     not an error).
  2. `rsync -a --remove-source-files -e ssh home:bob/xlib/ "$HOME/bob/xlib/"`.
     `--remove-source-files` deletes each source file on athena only after it has
     transferred successfully, which is safer than a blind post-hoc `rm`; macOS 26's
     `openrsync` supports the flag (verified).
  3. `ssh home 'find ~/bob/xlib -mindepth 1 -type d -empty -delete'` to clear the
     directory husks `--remove-source-files` leaves behind — this is the "ssh command to
     delete that file from the `~/bob/xlib/` directory on this machine" the annotation
     asks for.
  4. Leave the `xlib/` → `lib/` move to `bob highlights`'s existing intake. Guard it
     with a lock directory so overlapping cron runs cannot race, mirroring
     `maybe_bob_highlights_sync`.
- **`home/dot_config/bob/config.yml`** — add the
  `highlights.pre_scan_command: bob_xlib_pull` block from the `highlights` phase.
- **`home/bin/executable_maybe_bob_highlights_sync`** — its gate is
  `find "$lib_dir" -type f -mtime 0`, which short-circuits **before** the pre-scan hook
  can ever run, because a new athena PDF has not reached the MacBook's `lib/` yet.
  Chicken-and-egg: fix it or the bridge never fires. `bob highlights scan --dry-run`
  measures 7.7 s over 113 PDFs, which is 0.85 % of a core at the 15-minute cron cadence,
  so **drop the mtime gate whenever a `pre_scan_command` is configured**. Re-measure on
  the MacBook first; if it exceeds ~15 s there, fall back to gating on `lib/`
  mtime-today **or** a non-empty local `xlib/` and run `bob_xlib_pull` from the wrapper
  ahead of the gate.
- **Delete `home/bin/executable_bob_sync`**, matching the `retire` phase.

Apply with `chezmoi apply` on athena and on the MacBook. chezmoi is installed on the
MacBook at `/opt/homebrew/bin/chezmoi` with its source at `~/.local/share/chezmoi`, but
it is **not on the non-login PATH** — invoke it by absolute path over SSH.

### 4. Clone `bob-plugins` on the MacBook

`bob plugins sync` defaults to `~/projects/github/bobs-org/bob-plugins`
(`env.rs::plugins_dir`), which does not exist on the MacBook. Clone it there so D4's
manual workflow is actually available, and verify `bob plugins list` runs. The MacBook's
`block-id-prompt` build is currently stale relative to athena's; running
`bob plugins sync` there is the fix.

### 5. Install `bob` on both machines

`cargo install --path . --locked` from each machine's `bob-cli` checkout
(`~/projects/github/bbugyi200/bob-cli` on the MacBook). Verify `bob vault-sync --help`
and `bob highlights --help` on both, and that `bob bulk-git-commit` is gone.

### 6. Install the triggers, disabled

**athena** — `apt install inotify-tools`, then
`~/.config/systemd/user/bob-vault-sync.service` (`Type=simple`, `Restart=always`,
`RestartSec=10`) running a small loop script:

```bash
while true; do
  inotifywait -q -r -t 15 -e modify,create,delete,move \
    --exclude '(/\.git/|/\.obsidian/workspace|/\.trash/|/\.sase/|/old_lib/|/lit_review/|/_conflicts/)' \
    "$BOB_DIR" >/dev/null
  sleep 5                # debounce: Obsidian writes ~2 s after typing stops
  bob vault-sync -q
done
```

`inotifywait -t 15` returns on either a file event or the timeout, so one blocking wait
gives both edge-triggered push and a 15-second remote poll.
`fs.inotify.max_user_watches` is 505,837, far above the ~6.4 k needed. Do **not** enable
the unit yet.

**MacBook** — `~/Library/LaunchAgents/com.bbugyi.bob-vault-sync.plist` with
`StartInterval` 15, `RunAtLoad` true, `StandardOutPath`/`StandardErrorPath` under
`/var/tmp/`, and an `EnvironmentVariables` dict setting an explicit `PATH` (including
`/Users/bbugyi/.cargo/bin`) and `HOME`. The directory does not exist yet; create it. Do
**not** `launchctl bootstrap` it yet.

Both triggers call the identical command; only the trigger differs (D9).

### 7. Close the remaining query surfaces (D6)

- Tasks plugin: set `globalQuery` to `path does not include _conflicts`. It is currently
  empty, so this is free and covers every `tasks` block with no per-query edits.
  (`globalFilter` stays `#task`.)
- Dataview: add `_conflicts` to its excluded folders in
  `.obsidian/plugins/dataview/data.json`, which currently has none.
- The `bob` walker skip already shipped in `vaultsync`.

Verify by creating a throwaway `_conflicts/probe.md` containing a `- [ ] #task probe`
line and confirming it appears in **neither** `dash.md` nor `blocked.md`, then deleting
it.

Done when: `env -i` git access works on athena, the MacBook's `~/bob` is a clone with a
clean `git status` at athena's SHA, chezmoi is applied on both, both machines run the
new `bob`, and both triggers are installed and **not running**.

## Phase `cutover`: Prove the acceptance matrix end to end, then cut over

The user's instruction is to get this working end to end with strong confidence before
landing. Every check below runs against the **real** two machines over SSH, not a
fixture. Record each result on the phase bead as it passes.

### 1. Dry-run window

With both triggers still disabled, run `bob vault-sync -n` manually on each machine and
confirm the reported plan matches reality. Then run one real cycle on each, serially,
and confirm both land at the same SHA.

### 2. Acceptance matrix

| #   | Test                                                          | Pass condition                                                                                              |
| --- | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| 1   | Edit a note on the MacBook                                    | Appears on athena within 30 s                                                                               |
| 2   | Edit a note on athena                                         | Appears on the MacBook within 30 s                                                                          |
| 3   | Concurrent edits to _different_ notes                         | Both arrive on both machines                                                                                |
| 4   | Concurrent _non-overlapping_ edits to one note                | Merged, both arrive                                                                                         |
| 5   | Concurrent _same-line_ edits to one note                      | Quarantined conflict copy; **no `<<<<<<<` anywhere**; no halted merge; the copy is invisible in `dash.md`   |
| 6   | Both machines create the same daily note                      | `AA` conflict → conflict copy                                                                               |
| 7   | Push race                                                     | Retried automatically, no user action                                                                       |
| 8   | Both machines edited offline, then reconnected                | Reconciles                                                                                                  |
| 9   | Delete on one, edit on the other                              | File kept                                                                                                   |
| 10  | Binary changed differently on both                            | Uncorrupted conflict copy                                                                                   |
| 11  | Drop a 96 MiB file into the vault                             | Refused **locally**, before GitHub sees it                                                                  |
| 12  | `kill -9` the loop mid-merge                                  | Next cycle auto-recovers                                                                                    |
| 13  | No-change cycle                                               | No commit, no push                                                                                          |
| 14  | **Reboot athena; sleep/wake the MacBook**                     | Both resume with **no interactive SSH login**                                                               |
| 15  | `bob nightly` on athena                                       | Pulls before maintenance, pushes the maintenance commit                                                     |
| 16  | Drop a PDF into athena's `xlib/`, wait for the MacBook's cron | It lands in the MacBook's `lib/`, is removed from athena's `xlib/`, and a `ref/` note is produced           |
| 17  | `bob plugins sync` on the MacBook                             | Refreshes `block-id-prompt` to match athena                                                                 |
| 18  | Idle CPU over 10 minutes on both                              | At or under the ~1.5 % of one core the measurements predict, and below `ob-sync-bob.service`'s measured 4 % |

Test 14 is the one most likely to fail and the one that matters most for "frictionless";
do not accept a pass that relied on a warm ssh-agent.

### 3. Cut over

**Exactly one sync engine may run.** With both live, a delete propagated by one is
resurrected by the other, forever.

1. Enable and start `bob-vault-sync.service` on athena; `launchctl bootstrap` the
   LaunchAgent on the MacBook.
2. `systemctl --user disable --now ob-sync-bob.service`. Leave the unit and
   `~/.local/bin/ob-sync-bob-poll` on disk, disabled, through the soak.
3. Turn Obsidian Sync off in the MacBook GUI (Settings → Sync). There is no supported
   CLI for this.
4. Restore the 03:30 `bob nightly` crontab line from the backup taken in `reconcile`,
   adjusted for the new step sequence.
5. Confirm the MacBook's three 15-minute cron jobs still run and that their vault writes
   propagate.

### 4. Soak and document

- Soak for two weeks. **On day one, deliberately create a conflict** and confirm a
  quarantined conflict copy rather than markers.
- The rollback target is **git**, not Obsidian Sync — the Sync remote is over quota and
  rejecting writes. Tag the pre-cutover commit and rely on the filesystem backups from
  `reconcile`.
- Rewrite `docs/obsidian-sync-exclusions.md`, or add a sibling `docs/vault-git-sync.md`,
  covering: the cycle, the conflict-copy policy and where copies land, the deploy-key
  and ControlMaster setup on each machine, both trigger units, the `xlib/` rsync bridge,
  the `lit_review/` out-of-band policy, the manual `bob plugins sync` step, and how to
  read `bob vault-sync status`.
- Only after a clean soak: `ob sync-unlink --path ~/bob`, `ob logout`, remove
  `ob-sync-bob-poll` and its unit, and cancel the subscription. Keep this last group out
  of the epic's completion criteria — it is the user's call on the user's timeline.

### 5. Bead hygiene

- Close `bob-cli-1l.3`, `.4`, `.5` and then the `bob-cli-1l` epic bead with
  `-R superseded`, citing this epic (D7). Closing does not cascade, so close the phases
  first.
- Close `bob-cli-1i` (re-check Obsidian Sync quota daily) as `superseded`.
- Re-evaluate `bob-cli-1j` (let `bob nightly` run when the sync gate fails) — the
  `retire` phase deletes the gate, so it is likely `superseded` too.
- Re-evaluate `bob-cli-1g` (96 vault files sync to Obsidian but are untracked in git);
  `reconcile` should have closed the underlying gap.
- File follow-ups through `/sase_new_task`: the vault-repo PDF-tracking policy (D3), the
  `sase/memory/obsidian.md` update (needs the user's explicit approval, see Non-goals;
  `bob-cli-1h` may already cover it), and optionally an `fswatch` upgrade for the
  MacBook trigger (D9).

Done when: every acceptance test has passed on the real machines and is recorded on the
bead, Obsidian Sync is off on both devices, `bob-vault-sync` is running on both,
`bob nightly` is restored, the runbook is committed, and the superseded beads are
closed.
