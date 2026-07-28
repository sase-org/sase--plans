---
tier: epic
title: Stop sase test and tooling scratch from exhausting the /tmp tmpfs
goal: 'A full `just check` sweep across every sase workspace no longer grows the /tmp
  tmpfs without bound: pytest scratch lives off tmpfs with bounded retention, per-test
  scaffolding stops copying multi-megabyte binary assets, ChangeSpec lock and archive
  siblings stay inside pytest-managed temp trees, production sase temp files are cleaned
  up, deleting something under /tmp actually reclaims its space, and a regression
  guard fails the suite if system-temp leakage returns.

  '
phases:
- id: basetemp
  title: Move pytest scratch off the /tmp tmpfs and bound its retention
  depends_on: []
  size: medium
  description: '''Move pytest scratch off the /tmp tmpfs and bound its retention''
    section: point the pytest base temp directory at a per-workspace on-disk cache
    instead of the shared 32G tmpfs, set an explicit tmp_path retention policy and
    count, and add a bounded reaper that prunes stale run directories without waiting
    out pytest''s 3-day lock timeout.

    '
- id: assets
  title: Stop copying multi-megabyte PNG assets into every scaffolded test home
  depends_on: []
  size: medium
  description: '''Stop copying multi-megabyte PNG assets into every scaffolded test
    home'' section: find the scaffolding path that installs the sdd and memory directory-map
    PNGs, and make asset installation skippable or redirectable so each per-test temp
    home costs kilobytes instead of ~2.3 MB.

    '
- id: locks
  title: Stop leaking ChangeSpec lock and archive siblings into the system temp dir
  depends_on: []
  size: medium
  description: '''Stop leaking ChangeSpec lock and archive siblings into the system
    temp dir'' section: migrate the test call sites that create ProjectSpec temp files
    in the default system temp dir onto pytest''s tmp_path, so the `.lock`, `-archive`,
    and `-archive.lock` siblings that sase creates alongside them land inside a directory
    pytest can collect.

    '
- id: prodleaks
  title: Give production sase temp files a cleanup path
  depends_on: []
  size: small
  description: '''Give production sase temp files a cleanup path'' section: audit
    the mkstemp/mkdtemp/NamedTemporaryFile call sites in src/sase that land in the
    default system temp dir and ensure each one removes its artifact on both success
    and failure.

    '
- id: trashenv
  title: Stop `rm` from parking deleted /tmp data inside /tmp
  depends_on: []
  size: small
  description: '''Stop `rm` from parking deleted /tmp data inside /tmp'' section:
    change the dotfiles-level `rm`-to-`trash` alias so paths under /tmp and $TMPDIR
    are deleted outright instead of moved into a trash directory that lives on the
    same tmpfs, and document the rule for agents.

    '
- id: guard
  title: Regression guard against system-temp leakage
  depends_on:
  - basetemp
  - assets
  - locks
  size: small
  description: '''Regression guard against system-temp leakage'' section: add a session-scoped
    check that snapshots the system temp directory around a suite run and fails when
    the suite leaves new entries behind, so the earlier phases cannot silently regress.

    '
- id: reclaim
  title: One-time reclamation of stuck space and orphaned dirents
  depends_on:
  - trashenv
  size: small
  description: '''One-time reclamation of stuck space and orphaned dirents'' section:
    behind an explicit user confirmation gate, reclaim the multi-gigabyte trash directory
    and sweep the orphaned zero-byte lock files and bare temp entries out of /tmp.

    '
create_time: 2026-07-25 08:14:59
status: done
bead_id: sase-96
---

- **PROMPT:** [202607/prompts/tmp_space_exhaustion.md](prompts/tmp_space_exhaustion.md)

# Plan: Stop sase test and tooling scratch from exhausting the /tmp tmpfs

## Problem

`/tmp` on this host is a **32 GB tmpfs** (`rw,nosuid,nodev,nr_inodes=1048576`) — RAM-backed, so everything written there
is resident memory, and nothing survives reboot. It filled to 100% and returned `ENOSPC` **during the investigation that
produced this plan**, which is direct confirmation of the failure mode rather than a reconstruction from logs.

Measured state at the start of that investigation: **4.1 GB used across 29,272 entries**. The composition is what makes
this a sase problem rather than generic host clutter.

### Where the bytes are

| Path                                     | Size    | Note                                                      |
| ---------------------------------------- | ------- | --------------------------------------------------------- |
| `/tmp/.Trash-1000`                       | 3.9 GB  | ~95% of baseline usage; already-"deleted" data            |
| `/tmp/.Trash-1000/files/pytest-of-bryan` | 1.77 GB | **239** numbered run dirs (`pytest-103` … `pytest-1824`)  |
| `/tmp/.Trash-1000/files/plainci-XyO7`    | 1.06 GB | contains a full `sase` clone                              |
| `/tmp/.Trash-1000/files/plainci-GrvR`    | 1.06 GB | contains a full `sase` clone                              |
| `/tmp/pytest-of-bryan` (live)            | 172 MB  | 11 numbered dirs + `pytest-current` + one `garbage-*` dir |
| `/tmp/check.log`                         | 24 MB   | agent scratch                                             |
| `/tmp/node-compile-cache`                | 11 MB   | unrelated tooling                                         |

The single largest observed pytest run directory, `pytest-1719`, was **547 MB**, spread across ~20 xdist worker dirs
(`popen-gw0` … `popen-gw18`); `popen-gw7` alone held 187 MB. **24 numbered sase workspaces share this one tmpfs**, and
each runs `just check` — so one round of checks across workspaces can alone account for well over 10 GB.

The burst behavior was captured directly: within a few minutes, `/tmp` went from **4.1 GB → full (32 GB, `ENOSPC`) → 4.2
GB** as concurrent runs allocated and then released roughly **28 GB** of scratch. So the steady-state figures above
understate the risk considerably — the peak, not the average, is what breaks the host, and it is driven by how many runs
overlap. Note also what the outage broke: `ENOSPC` on a tmpfs takes down anything that needs a scratch file, including
agent tooling itself (Claude Code's own `/tmp/claude-1000` writes failed, leaving the agent with no working shell).
Fixes must therefore reduce the _peak_, not merely clean up afterwards.

### Four independent root causes

**RC1 — pytest scratch lives on the tmpfs and is pinned there for three days.** `tools/run_pytest` passes neither
`--basetemp` nor `TMPDIR`, so pytest defaults to `$TMPDIR/pytest-of-$USER` = `/tmp/pytest-of-bryan`. pytest keeps the 3
newest numbered dirs, but its garbage collector additionally _refuses_ to remove any older run dir whose `.lock` file is
younger than `_pytest.pathlib.LOCK_TIMEOUT`. On pytest 9.1.1 that constant is **259200 seconds = 3 days**:

```python
# _pytest/pathlib.py — ensure_deletable()
if lock_time < consider_lock_dead_if_created_before:   # now - LOCK_TIMEOUT
    with contextlib.suppress(OSError):
        lock.unlink()
        return True
return False        # <-- lock younger than 3 days => dir is NOT collectable
```

Every run inside a rolling 3-day window is therefore pinned. The live directory proves GC is failing _right now_: 11
numbered dirs survive where `keep=3` should leave 3, plus a `garbage-*` dir that pytest created because it could not
delete something.

**RC2 — deleting anything under /tmp reclaims zero bytes.** `rm` is aliased to `trash` (trash-cli at
`~/.local/bin/trash`). The XDG trash spec requires the trash to sit on the _same filesystem_ as the file, so removing
something under /tmp moves it to `/tmp/.Trash-1000` and frees nothing. That is why 3.9 GB of nominally-deleted data was
still occupying the tmpfs, and **2,349 of the 2,358 trash entries were created on a single day**. Because agent shells
are initialized from the user's profile, this alias applies to agents too — every `rm -rf /tmp/...` cleanup an agent ran
was silently a no-op for space.

**RC3 — ~24k orphaned lock/archive siblings (dirents, not bytes).** `/tmp` holds **24,272 `*.lock` files, every one zero
bytes** — 20,288 `*.sase.lock`, 2,998 `*.md.lock`, 460 `*-archive.sase.lock`, 230 `*.md-archive.lock` — plus 230 orphan
`*.md-archive` files and 4,370 bare `tmpXXXXXXXX` entries. _This is not the space cause_ (total 0 blocks); it is dirent
and inode bloat that slows every scan of /tmp. The mechanism: `changespec_lock()` creates a lock file and never unlinks
it —

```python
# src/sase/ace/changespec/locking.py:159-167,182-184
lock_file = f"{project_file}.lock"
fd = os.open(lock_file, os.O_RDWR | os.O_CREAT, 0o644)
...
finally:
    fcntl.flock(fd, fcntl.LOCK_UN)
    os.close(fd)          # <-- lock_file itself is never removed
```

For the real `~/.sase/projects/<key>/<key>.sase` this is harmless (one stable file). But **135** call sites across
`tests/` and `src/` create temp files via `tempfile.NamedTemporaryFile(suffix=".sase"|".md", delete=False)` with no
`dir=` argument, so each lands in `/tmp` under a fresh random name; the test unlinks its own file but never the siblings
sase created next to it. Concentrated offenders include `tests/test_changespec_add_operations.py` (11 sites),
`tests/test_mentors_wip.py`, `tests/test_status_state_machine_field_updates.py`,
`tests/test_changespec_reservation_operations.py`, `tests/test_bare_git_workspace.py`, `tests/test_hooks_operations.py`,
and `tests/test_clipboard_raw_text.py`.

**RC4 — per-test scaffolding copies megabytes of binary assets.** Each per-test temp dir measured **~2.3 MB** (2348–2360
KB), and the bulk is packaged PNGs copied into the fake home — observed at
`<testdir>/.sase/sdd/assets/sdd-directory-map.png` and `<testdir>/home/sase/memory/assets/memory-directory-map.png`. The
repo ships 5,033,085 bytes of these:

- `src/sase/sdd/assets/research-directory-map.png` — 1.3 MB
- `src/sase/sdd/assets/plans-directory-map.png` — 1.3 MB
- `src/sase/sdd/assets/sdd-directory-map.png` — 1.2 MB
- `src/sase/memory/assets/memory-directory-map.png` — 1.2 MB

This is the multiplier that turns a run into half a gigabyte: ~2.3 MB × hundreds of scaffolding tests × ~20 workers.

### Also present, lower priority

Production sase leaves temp artifacts behind: 179 trashed plus 3 live `sase_xprompts_catalog_*` dirs, many
`sase_sdd_msg_*.txt`, `sase-gh-*.diff`, `sase-core-py-bead-*`, and `sase-pdf-properties-*`. The `plainci-*` clones (1.06
GB each) were **not** produced by any code in this repo, in chezmoi, or on `PATH` — they are ad-hoc agent scratch, and
are covered by the generic mitigations rather than a code fix.

## Non-goals

- Resizing or re-mounting the tmpfs. That hides the leak instead of fixing it, and the tmpfs is RAM.
- Reworking `changespec_lock()` into a self-deleting lock. Unlinking a file you hold an `flock` on is racy — a second
  process can create and lock a fresh file at the same path and both then believe they hold the lock. The `locks` phase
  deliberately fixes _where_ the temp specs live instead.
- Changing how many xdist workers `tools/run_pytest` uses. Its shared-token governance is intentional.

---

## Move pytest scratch off the /tmp tmpfs and bound its retention

Owner phase: `basetemp`.

This is the highest-leverage fix; do it first if phases are serialized.

1. In `tools/run_pytest`, set the pytest scratch root to a **per-workspace, on-disk** directory rather than the shared
   tmpfs. Prefer exporting `TMPDIR` for the pytest child process over passing `--basetemp`, because it preserves
   pytest's numbered-dir scheme, the `pytest-current` symlink, and xdist's `popen-gwN` layout, and it also catches
   libraries that call `tempfile` directly. A good target is a gitignored per-workspace path such as
   `<repo-root>/.pytest_cache/tmp` (the workspace clone is on disk, and workspaces are ephemeral, so it is discarded
   with the workspace). Verify how the script invokes pytest before wiring this up — it builds an argument list and
   governs `--numprocesses` from a shared token, and it rejects caller-supplied `-n`/`--numprocesses`.
   - Create the directory if missing, and make sure `just test`, `just test-cov`, and `just test-visual` all inherit it.
   - Leave an escape hatch: honor a pre-existing `SASE_PYTEST_TMPDIR` (or similar) override so CI can redirect it.
2. In `pyproject.toml` under `[tool.pytest.ini_options]` (line 249), add explicit retention settings. Defaults today are
   `all` / `3`, which keeps per-test dirs for passing tests:
   - `tmp_path_retention_policy = "failed"` — keep per-test dirs only for tests that failed. Nearly all tests pass, so
     this alone removes most of the per-run footprint.
   - `tmp_path_retention_count = "1"` — keep one generation instead of three.
   - Note `--strict-config` is already in `addopts`, so a misspelled key fails loudly. Good.
3. Add a bounded reaper to `tools/run_pytest` that runs before pytest starts and prunes run dirs under the scratch root
   older than a short horizon (hours, not pytest's 3 days). It must not fight concurrent runs: skip any dir whose
   `.lock` is younger than the horizon, and ignore `OSError` so a losing race is harmless.
4. Because 955 test files use `tmp_path`, do not change fixture semantics — only the root location and retention.

**Verification.** Record `df /tmp` and the `/tmp` entry count, run `just test` twice back to back, and confirm neither
number grows. Confirm run dirs now appear under the new root, and that `pytest-of-bryan` gains nothing.

## Stop copying multi-megabyte PNG assets into every scaffolded test home

Owner phase: `assets`.

1. Locate the code that installs the packaged PNGs into a scaffolded home. The evidence is the on-disk layout
   (`.sase/sdd/assets/sdd-directory-map.png`, `home/sase/memory/assets/memory-directory-map.png`), so start from the
   asset directories `src/sase/sdd/assets/` and `src/sase/memory/assets/` and follow their readers; a plain grep for
   `assets` in `src/sase/sdd/` and `src/sase/memory/` did not pin the copy site, so trace the init/scaffolding entry
   points (`sase init`, `sase repo init`, `sase memory init`) instead of assuming a helper name.
2. Make asset installation avoidable for tests without weakening production behavior. In preference order:
   - Skip asset installation unless it is actually requested, so scaffolding that never asserts on the PNGs never pays
     for them. Give the test fixtures the cheap path by default.
   - If some tests must see the files, write a tiny placeholder byte string in tests rather than the real PNG, or
     hardlink/symlink the packaged asset instead of copying it (a hardlink on the same filesystem costs one dirent).
3. Keep at least one test that asserts the real assets _are_ installed on the production path, so this optimization
   cannot silently break `sase init`.

**Verification.** Measure a single per-test scaffolding dir before and after; it should drop from ~2.3 MB to kilobytes.
Then measure a full run directory — the ~547 MB worst case should fall by roughly an order of magnitude.

## Stop leaking ChangeSpec lock and archive siblings into the system temp dir

Owner phase: `locks`.

The fix is to relocate the temp specs, not to make the lock self-deleting (see Non-goals).

1. Migrate the offending call sites from `tempfile.NamedTemporaryFile(..., delete=False)` in the default temp dir onto
   pytest's `tmp_path` / `tmp_path_factory`. Once the spec lives inside the pytest temp tree, every sibling sase creates
   next to it (`.lock`, `-archive`, `-archive.lock`) is collected with the parent directory.
   - Enumerate the work with a grep for `NamedTemporaryFile|mkstemp` filtered to sites passing a `suffix` and no `dir=`;
     there were 135 such sites across `tests/` and `src/` at plan time.
   - Many are module-level helper factories shared by several tests, so the edit count is far smaller than 135; fix the
     helper and its callers follow. Start with `tests/test_changespec_add_operations.py`, `tests/test_mentors_wip.py`,
     `tests/test_status_state_machine_field_updates.py`, `tests/test_changespec_reservation_operations.py`,
     `tests/test_bare_git_workspace.py`, `tests/test_hooks_operations.py`, and `tests/test_clipboard_raw_text.py`.
   - Where a helper cannot take a fixture (for example it is called at import time), pass an explicit `dir=` pointing at
     a per-test directory rather than leaving the default.
2. Do not change `changespec_lock()`'s locking protocol. If you want defense in depth, the safe option is to place lock
   files in a dedicated subdirectory keyed by the spec path so they are bounded in number, but treat that as optional
   and out of scope unless it falls out naturally.
3. Leave the historical 24,272 orphans alone here — the `reclaim` phase sweeps them.

**Verification.** Snapshot `ls /tmp | grep -c '\.lock$'` before and after a full `just test`; the delta must be 0. Same
for `*.md-archive` and bare `tmpXXXXXXXX` entries.

## Give production sase temp files a cleanup path

Owner phase: `prodleaks`.

1. Audit `src/sase` for `tempfile.mkstemp`, `tempfile.mkdtemp`, and `tempfile.NamedTemporaryFile` calls that do not pass
   `dir=` and therefore land in the system temp dir. Known leaked families, by the leftovers they produced:
   - `sase_xprompts_catalog_*` — 179 trashed plus 3 live directories; the worst offender, fix first.
   - `sase_sdd_msg_*.txt`
   - `sase-gh-*.diff`
   - `sase-core-py-bead-*`
   - `sase-pdf-properties-*`
2. For each, guarantee removal on both success and failure — a `try`/`finally`, a `with tempfile.TemporaryDirectory()`,
   or `contextlib.ExitStack`. Several sites use the `fd, path = mkstemp(...)` idiom for atomic-replace writes; those are
   correct only if the temp file is removed when the replace raises, so check the failure path specifically.
3. Where a file is intentionally handed to an external process or editor and cannot be deleted immediately, give it a
   deterministic parent directory under the sase state dir instead of the system temp dir, so it is reapable.

**Verification.** Exercise the affected commands and confirm no new `sase_*` or `sase-*` entries persist in the system
temp dir afterward.

## Stop `rm` from parking deleted /tmp data inside /tmp

Owner phase: `trashenv`.

This is a dotfiles change, **not** a change to this repo. Open the chezmoi repo with the `/sase_repo` skill — do not
edit `~/.zshrc` or `~/.local/bin` directly, and do not clone or web-fetch chezmoi any other way.

1. Replace the bare `rm` → `trash` alias with a small wrapper function that deletes outright when the target is under
   `/tmp` or `$TMPDIR`, and otherwise falls through to `trash`. Rationale to preserve in a comment: the XDG trash must
   live on the same filesystem as the file, so trashing something on a tmpfs reclaims nothing and merely relabels it.
   - Handle relative paths by resolving before the prefix test, and keep the wrapper safe when given no arguments.
   - Keep the safety intent of the alias for everything outside /tmp — this is a targeted carve-out, not a revert.
2. Add a periodic or login-time `trash-empty` for the /tmp trash volume as a backstop, so any trash that does land there
   cannot accumulate across a long uptime.
3. Record the rule where agents will see it, since agent shells inherit the user's profile and this bit them: a cleanup
   command aimed at /tmp must reach real `rm`. This must go in a normal document, **not** in `sase/memory/*.md` or any
   generated instruction shim — editing those requires explicit user permission that this plan does not grant.

**Verification.** `rm` a scratch file under /tmp and confirm `df /tmp` drops and nothing appears in `/tmp/.Trash-1000`;
`rm` a scratch file under `$HOME` and confirm it still lands in the home trash.

## Regression guard against system-temp leakage

Owner phase: `guard`.

Depends on `basetemp`, `assets`, and `locks` — the guard can only be switched on once the suite is actually clean.

1. Add a session-scoped guard that records the set of entries in the _system_ temp dir (`tempfile.gettempdir()` as seen
   by the parent, i.e. the real `/tmp`, not the redirected pytest root) at session start, compares at session end, and
   fails when the suite added entries.
2. Make it robust in the real environment rather than flaky:
   - Under xdist, only the controller should assert, or the check must tolerate sibling workers' concurrent activity.
     Comparing sets and reporting only entries this session created is safer than comparing counts.
   - Other processes on this host also write to /tmp. Ignore entries the suite provably did not create, and prefer an
     allowlist of known-foreign prefixes over a strict count equality.
   - Give it an env-var opt-out so a developer debugging a leak can disable it.
3. Report the offending entry names in the failure message — a bare count tells the next person nothing.

**Verification.** Confirm the guard passes on the fixed suite, then deliberately reintroduce one `NamedTemporaryFile`
leak and confirm it fails and names the entry.

## One-time reclamation of stuck space and orphaned dirents

Owner phase: `reclaim`.

Depends on `trashenv`: emptying the trash before the alias is fixed just invites it to refill, and the reclamation
itself needs real deletion rather than the trash alias.

**This phase destroys data and must not run unattended.** Propose the commands through a confirmation gate (the
`sase_gate` skill) and let the user approve them.

1. Reclaim `/tmp/.Trash-1000` (3.9 GB at plan time, and growing). Prefer `trash-empty` for the /tmp volume so
   trash-cli's own bookkeeping in `.Trash-1000/info` stays consistent; otherwise use an absolute `/usr/bin/rm -rf` so
   the alias cannot intercept it. Everything in there is already-deleted data, but surface a summary of the largest
   entries for the user before deleting.
2. Sweep the orphaned dirents: 24,272 zero-byte `*.lock` files, 230 `*.md-archive` files, and 4,370 bare `tmpXXXXXXXX`
   entries. Batch the deletion (`find … -delete`, or `xargs`) because a single glob over ~29k entries can exceed
   `ARG_MAX`. Restrict the patterns tightly — `*.sase.lock`, `*.md.lock`, `*-archive.sase.lock`, `*.md-archive.lock`,
   `*.md-archive` — and add an age filter so a concurrently-running test's lock is not removed.
3. Do **not** delete anything whose owner is unclear, `.X11-unix`/`.ICE-unix`/`.font-unix`, active sockets such as
   `bryan-supervisor.sock`, `tmux-1000`, or the live `claude-1000` directory.
4. Report reclaimed bytes and the final entry count.

**Verification.** `df -h /tmp` shows the space returned; `ls /tmp | wc -l` drops from ~29k to a small number; the
desktop session, tmux, and supervisord are all still healthy afterward.

---

## Suggested execution order

`basetemp`, `assets`, `locks`, `prodleaks`, and `trashenv` are mutually independent and can run in parallel. `guard`
joins after `basetemp` + `assets` + `locks`. `reclaim` follows `trashenv`.

If capacity forces serialization, `basetemp` alone removes most of the growth, and `reclaim` (after `trashenv`) is what
returns the space that is stuck today.

## Repo-wide requirements

- Every phase that changes files in this repo must run `just install` and then `just check` before reporting done —
  workspace virtualenvs are ephemeral and may hold stale dependencies.
- The `trashenv` phase touches only the chezmoi repo, opened via `/sase_repo`.
- No phase may edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims (`CLAUDE.md`, `GEMINI.md`,
  `OPENCODE.md`, `QWEN.md`). This plan does not constitute user permission for those edits.
