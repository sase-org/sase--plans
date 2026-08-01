---
tier: epic
title: Close the remaining temp-scratch leaks sase-96 relocated but did not stop
goal: 'Nothing sase runs leaves unbounded scratch behind: the agent-launch prompt
  file lands in a reapable managed subdirectory instead of the bare $SASE_TMPDIR root,
  both sase-owned temp roots are reaped on a bounded horizon, the test suite can no
  longer write into the developer''s real managed root, the sase-github handoff diffs
  and sase-core Rust test directories clean up after themselves, the 94k stale entries
  stuck in the managed root are reclaimed, and epic sase-96 is closed out.

  '
phases:
- id: terminal_smoke
  title: Route the terminal-smoke lane through the pytest runner
  depends_on: []
  size: small
  description: '''Route the terminal-smoke lane through the pytest runner'' section:
    give tools/run_pytest a serial terminal-smoke mode, point the just recipe at it
    instead of a direct pytest invocation, and make the leak guard compare snapshots
    only when the runner has redirected the temp root. This work was authored during
    the sase-96 landing verification and may already be committed; confirm it is present
    and complete rather than redoing it.

    '
- id: prompt_leak
  title: Give the agent-launch prompt file a reapable home
  depends_on: []
  size: medium
  description: '''Give the agent-launch prompt file a reapable home'' section: route
    src/sase/agent/launch_spawn.py''s sase_tmpdir argument and ace_handler''s profile
    output through a managed subdirectory instead of the bare $SASE_TMPDIR root, and
    decide the fate of the now-unused get_sase_tmpdir helper. This is the largest
    active producer (32,595 files) and a grep-driven audit of src/sase could not see
    it because the tempfile call lives in sase-core.

    '
- id: sandbox_tmpdir
  title: Stop the test suite from writing into the developer's managed temp root
  depends_on: []
  size: medium
  description: '''Stop the test suite from writing into the developer''s managed temp
    root'' section: make get_sase_managed_tmpdir sandbox-aware under pytest by reusing
    sase-9l''s state_write_guard and SASE_PYTEST_SANDBOX_DIR, fail closed when no
    sandbox is published, and extend the leak guard to watch the managed root. A 31-test
    run currently adds six directories to the real root, unseen by either guard.

    '
- id: scratch_reaper
  title: Reap everything stale under the pytest scratch root
  depends_on: []
  size: small
  description: '''Reap everything stale under the pytest scratch root'' section: widen
    tools/run_pytest''s reaper so it prunes any stale top-level entry under the workspace-private
    scratch root rather than only pytest-<N> and garbage-* under pytest-of-*, which
    today leaves 99 inline-snapshot-* directories unreclaimed.

    '
- id: plugin_diffs
  title: Contain the sase-github handoff diff files
  depends_on: []
  size: small
  description: '''Contain the sase-github handoff diff files'' section: stop gh.yml
    from hardcoding mktemp /tmp/sase-gh-XXXXXX.diff and leaving it behind, and give
    new_pr_desc_get_context.py''s pr_desc_ diff an explicit managed directory. Both
    were named in the sase-96 plan but fell outside the prodleaks phase''s src/sase
    scope.

    '
- id: core_test_dirs
  title: Clean up the sase-core Rust test temp directories
  depends_on: []
  size: small
  description: '''Clean up the sase-core Rust test temp directories'' section: change
    the four sase_core_py test helpers that create directories under std::env::temp_dir
    and never remove them to use tempfile::TempDir, and sweep the rest of the crates
    for the same pattern. sase-core-py-bead-* was named in the sase-96 plan.

    '
- id: managed_reaper
  title: Reap the managed SASE temp root
  depends_on:
  - prompt_leak
  - sandbox_tmpdir
  size: medium
  description: '''Reap the managed SASE temp root'' section: add a bounded reaper
    for the root get_sase_managed_tmpdir returns, with per-subdirectory horizons that
    respect how long the ACE Agents tab reads artifacts back, and wire it to a path
    that actually runs without blocking an interactive command. Nothing reaps that
    root today despite the helper documenting it as reapable.

    '
- id: reclaim_managed
  title: One-time reclamation of the managed temp root
  depends_on:
  - managed_reaper
  size: small
  description: '''One-time reclamation of the managed temp root'' section: behind
    an explicit user confirmation gate, reclaim the 94,522 entries and 410 MB stuck
    in $SASE_TMPDIR, which sase-96.7 left untouched because it swept only /tmp.

    '
- id: land
  title: Land sase-96
  depends_on:
  - terminal_smoke
  - scratch_reaper
  - plugin_diffs
  - core_test_dirs
  - reclaim_managed
  size: small
  description: '''Land sase-96'' section: re-verify the sase-96 phases still hold,
    close sase-96, run just symvision afterwards and remove anything it reports, then
    set status done in both the sase-96 plan file and this one.'
parent_bead: sase-96
create_time: 2026-07-25 14:15:04
status: done
bead_id: sase-96.8
---

- **PROMPT:** [prompts/202607/managed_tmp_reaping.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202607/managed_tmp_reaping.md)
- **PARENT:** [202607/tmp_space_exhaustion.md](https://github.com/sase-org/sase--plans/blob/main/202607/tmp_space_exhaustion.md)
- **BEAD:** [sase-96.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-96/sase-96.8.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-96.8.land](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-96.8.land/README.md)

# Plan: Close the remaining temp-scratch leaks sase-96 relocated but did not stop

## Problem

Epic sase-96 moved pytest scratch off the 32 GB `/tmp` tmpfs, shrank per-test scaffolding from ~2.3 MB to ~10 KB,
contained ChangeSpec lock siblings, added a leak guard, and reclaimed the stuck space. Those wins are real and verified:
`/tmp` now sits at 639 MB / 2% with zero `*.lock` files, zero bare `tmpXXXXXXXX` entries, and no `.Trash-1000`; a full
`just test` (22,075 passing) writes its scratch to a per-workspace `/var/tmp/sase-<hash>` root and the leak guard
reports nothing.

But the `prodleaks` phase audited `src/sase` by grepping for `tempfile.mkstemp`/`mkdtemp`/`NamedTemporaryFile`, and that
audit could not see the two largest leaks. Both are still active today.

### RC1 — the agent-launch prompt file is the single biggest producer, and the audit could not find it

`src/sase/agent/launch_spawn.py:248` passes `sase_tmpdir=get_sase_tmpdir()` into sase-core's `prepare_agent_launch`.
`get_sase_tmpdir()` returns `$SASE_TMPDIR` **itself** — not a subdirectory — and the Rust side keeps the file:

```rust
// sase-core: crates/sase_core/src/agent_launch/mod.rs:1791-1810 — write_prompt_temp_file()
builder.prefix("sase_ace_prompt_").suffix(".md");
let mut file = match sase_tmpdir {
    Some(dir) if !dir.is_empty() => builder.tempfile_in(dir)?,
    _ => builder.tempfile()?,
};
...
let (_file, path) = file.keep()?;   // <-- delete-on-drop disabled; nobody unlinks it later
```

No Python `tempfile` call appears at this site, so a grep-driven audit of `src/sase` never reaches it. The prompt file
is handed to the runner as `argv[8]` and outlives the launching process, so it cannot be deleted immediately — it needs
the treatment the sase-96 plan already prescribed for handoff files: a deterministic, reapable parent directory.

Measured on this host: `$SASE_TMPDIR` (`/home/bryan/tmp/sase`) holds **32,595 `*.md` files**, the largest single bucket
in a directory of **94,522 entries / 410 MB**. New `sase_ace_prompt_*.md` files landed at 13:47, 13:51, 13:57, 13:59 and
14:04 on 2026-07-25 — hours after sase-96.4 landed at 09:45.

`src/sase/main/ace_handler.py:34` is the other `get_sase_tmpdir()` consumer and drops `ace_profile_*.txt` into the same
bare root.

### RC2 — nothing reaps the managed temp root, so `prodleaks` relocated the pile instead of bounding it

`get_sase_managed_tmpdir()` documents its purpose as putting handoff files somewhere "so it is reapable" — and sase-96.4
correctly routed 20-odd call sites through it. But no reaper exists. Grepping `src/sase` for any prune/clean/sweep of
that root returns nothing.

The consequence is measurable. `du -sh /home/bryan/tmp/sase` **timed out after two minutes**; the directory holds:

| Bucket                    |  Count | Note                                       |
| ------------------------- | -----: | ------------------------------------------ |
| `*.md`                    | 32,595 | RC1's prompt files                         |
| `workflow-tmp_abc123-*`   | 22,411 | pre-epic residue from a test workflow name |
| `*.json`                  | 17,589 |                                            |
| `*.sh`                    |  8,383 | hook wrapper scripts                       |
| `workflow-simple-*`       |  5,599 | pre-epic residue from a test workflow name |
| `workflow-refresh_docs-*` |  5,584 | pre-epic residue from a test workflow name |

Only 334 of the 94,522 entries were modified after 09:00 on 2026-07-25, so the bulk predates the epic — but sase-96.7
reclaimed `/tmp` only, and left this root untouched.

### RC3 — the test suite writes into the developer's real `$SASE_TMPDIR`, escaping both guards

`get_sase_managed_tmpdir()` honors `$SASE_TMPDIR` unconditionally, including under pytest. Because the leak guard
watches `tempfile.gettempdir()` and sase-9l's `state_write_guard` watches the real `~/.sase` tree, neither sees writes
to `$SASE_TMPDIR`. Demonstrated directly:

```
before: workflow-artifacts=342 wrappers=114 handoff=285
31 passed, 4 warnings in 2.60s
after:  workflow-artifacts=348 wrappers=114 handoff=285
DELTA:  workflow-artifacts=6
```

Three test files added six directories to the developer's real managed root, and the guard stayed silent. The
accumulated `workflow-artifacts/` contents confirm the pattern is long-running: 342 entries named after test workflows
(`tmp_abc123`, `simple`, `refresh_docs`), plus 280 `sase_local_xprompts_*.json` in `handoff/` and 114 `tmp*.sh` in
`wrappers/`.

### RC4 — the pytest scratch reaper only recognizes pytest's own run directories

`tools/run_pytest`'s `_reap_stale_pytest_runs()` walks `<root>/pytest-of-*` and prunes entries matching `pytest-<N>` or
`garbage-*`. Anything else under the workspace scratch root is never reclaimed. Today that leaves **99
`inline-snapshot-*` directories** across the `/var/tmp/sase-<hash>` roots (the plugin creates one per session via a
`TemporaryDirectory` dataclass field that frequently outlives collection) plus leaked `tmpXXXXXXXX/artifacts/` trees.
The scratch root is sase-private and exclusively owned by the reaper, so it can safely prune any stale top-level entry
rather than only the two names it recognizes.

### Two named families the `prodleaks` phase listed but never reached

The sase-96 plan named five "known leaked families ... fix first". Three were fixed. The other two live outside
`src/sase`, so the phase's stated scope excluded them:

- **`sase-gh-*.diff`** — `sase-github`'s `src/sase_github/xprompts/gh.yml:141` runs
  `diff_file=$(mktemp /tmp/sase-gh-XXXXXX.diff)`. It hardcodes `/tmp`, so it ignores `$TMPDIR` entirely, and it only
  removes the file when the diff is **empty**; the normal path emits `diff_path=$diff_file` and leaves it behind
  forever. 60 such files (1.7 MB) accumulated on 2026-07-25 alone, from 08:18 through 13:52.
  `src/sase_github/scripts/new_pr_desc_get_context.py:75` has the same shape in Python:
  `NamedTemporaryFile(mode="w", suffix=".diff", prefix="pr_desc_", delete=False)` with no `dir=` and no unlink. Both are
  genuine handoff files — `diff_path` is a declared workflow output of type `path`, consumed downstream by
  `new_pr_desc.yml` — so they need a reapable directory, not immediate deletion.
- **`sase-core-py-bead-*`** — `sase-core`'s `crates/sase_core_py/src/lib.rs` has four `#[test]` helpers
  (`temp_beads_dir`, `temp_notification_path`, `temp_telemetry_path`, `temp_agent_stats_root`) that each
  `fs::create_dir_all` under `std::env::temp_dir()` and never remove it. 16 `sase-core-py-bead-*` directories sit in
  `/tmp` and 5 more in `/var/tmp`.

### RC5 — one pytest lane bypassed the redirect, which also made the leak guard diff the shared /tmp

`just test-terminal-smoke` invoked `python -m pytest -m terminal_smoke` directly instead of going through
`tools/run_pytest`, so that lane never received the `TMPDIR` redirect and wrote its scratch to the tmpfs. The
second-order effect is worse: with `TMPDIR` unset, `tempfile.gettempdir()` resolves to the real `/tmp`, so the sase-96.6
leak guard diffed the **shared** directory — precisely what its own docstring says it avoids, because "many sase
workspaces, editors, and agents write there concurrently, so diffing it reports other processes' scratch instead of this
suite's." That lane is not part of `just check`, so nothing caught it.

The `terminal_smoke` phase below carries the fix. It was authored during the sase-96 landing verification, so it may
already be committed; the phase's first job is to check.

## Non-goals

- Reworking `changespec_lock()` into a self-deleting lock. Unchanged from sase-96: unlinking a file you hold an `flock`
  on is racy.
- Deleting the agent-launch prompt file at launch time. It is `argv[8]` of a process that outlives the launcher, and the
  ACE Agents tab reads it back for prompt display. Reaping on a horizon is the correct lifetime.
- Changing `_isolate_sase_home` to stop using `tmp_path_factory.mktemp("home")`. Its `homeN` dirs are not subject to
  `tmp_path_retention_policy`, but pytest's controller removes the whole numbered basetemp on a fully passing run, and
  the measured mid-run cost is 81 MB on disk. Not worth destabilizing an autouse fixture every test depends on.
- Fixing `tests/ace/tui/widgets/file_panel/test_diff_cache.py::test_get_agent_diff_invalidates_when_index_changes`. It
  fails deterministically when the rest of its file runs first and passes in isolation, at HEAD and at `899a257f2` alike
  — a pre-existing intra-file pollution bug unrelated to temp scratch. It should get its own bead.

---

## Route the terminal-smoke lane through the pytest runner

Owner phase: `terminal_smoke`.

Check first whether this is already committed — `git log --oneline -- tools/run_pytest` and a grep for `terminal-smoke`
in the `Justfile`. If the work below is present and its tests pass, verify and report rather than redoing it.

1. Add a `terminal-smoke` mode to `tools/run_pytest`: a `TERMINAL_SMOKE_MARKER_EXPRESSION = "terminal_smoke"` selector,
   the mode in the argparse `choices`, and a `SERIAL_MODES` set the mode belongs to. The PTY tests must not run under
   xdist, so the mode has to force serial in both `main()` and `_pytest_command()` — those two functions decide
   independently today, so introduce one shared `_requires_serial_run(mode, args)` helper rather than duplicating the
   check. A serial mode must also skip the worker-token grant entirely.
2. Point the `test-terminal-smoke` recipe at `tools/run_pytest terminal-smoke tests/ace/tui/terminal_smoke "$@"`, with
   `SASE_JUST_INVOCATION_DIR` set the way the other test recipes do.
3. Have `_prepare_pytest_tmpdir()` export a marker (`SASE_PYTEST_TMP_REDIRECTED=1`) once it has pointed `TMPDIR` at the
   private scratch root, and make `tests/_tmp_leak_guard.py:watched_temp_directories()` return `()` unless that marker
   is set. Update the function's docstring to say the guard is inert without a redirect — the current docstring already
   argues the shared `/tmp` must not be diffed, so this makes the code match the stated design. A bare
   `pytest tests/...` then becomes inert rather than flaky.
4. Cover all three in tests: the mode selects its marker and stays serial, `main()` in that mode redirects `TMPDIR` and
   never acquires tokens, `_prepare_pytest_tmpdir()` sets the marker, and the guard is inert when the marker is absent
   or `0`.

**Verification.** `tools/run_pytest terminal-smoke tests/ace/tui/terminal_smoke -q` collects the PTY test, runs
serially, and writes its scratch under `/var/tmp/sase-<hash>` rather than `/tmp`. `tests/test_run_pytest_tool.py` and
`tests/test_tmp_leak_guard.py` pass.

## Give the agent-launch prompt file a reapable home

Owner phase: `prompt_leak`.

This is the highest-value fix: it is the largest producer and it is still leaking.

1. Change `src/sase/agent/launch_spawn.py:248` to pass a managed **subdirectory** rather than the bare root — e.g.
   `sase_tmpdir=get_sase_managed_tmpdir("launch-prompts")`. Confirm the parameter reaches
   `write_prompt_temp_file(sase_tmpdir, ...)` in sase-core unchanged; the Rust side needs no edit, because it already
   honors an explicit directory and only falls back to the system temp dir when the argument is empty or `None`.
2. Audit the other `get_sase_tmpdir()` consumer, `src/sase/main/ace_handler.py:34`, which writes
   `ace_profile_<timestamp>.txt` into the bare root. Route it under a managed subdirectory too, and keep the
   `--profile <path>` override behavior intact.
3. Decide the fate of `get_sase_tmpdir()` itself. After the two call sites above move, it has no non-test production
   consumer. Either delete it in favor of `get_sase_managed_tmpdir()` or leave it and document that new code must not
   use it. Symvision will flag an unused public symbol, so pick one and make it consistent.
4. Keep the `prompt_file` value in the `PreparedAgentLaunch` wire unchanged in shape — only its parent directory moves.
   `tests/test_core_agent_launch_wire.py` asserts on the `sase_ace_prompt_` prefix and must keep passing.

**Verification.** Launch an agent (or exercise `prepare_agent_launch` through its existing test) and confirm the prompt
file appears under the managed subdirectory and no longer directly under `$SASE_TMPDIR`. Snapshot the entry count of
`$SASE_TMPDIR` itself across the run; the delta must be 0.

## Reap the managed SASE temp root

Owner phase: `managed_reaper`.

Depends on `prompt_leak` and `sandbox_tmpdir`: reaping is what makes those two safe, and it should not start pruning
while tests are still writing into the same root.

1. Add a bounded reaper for the root `get_sase_managed_tmpdir()` returns. Model it on
   `tools/run_pytest:_reap_stale_pytest_runs()` — prune entries older than a horizon of hours, skip anything with a
   recent mtime, and swallow `OSError` so a lost race is harmless.
2. Choose the horizon per subdirectory, not globally. `editors/`, `wrappers/`, `viewers/`, and `commit-messages/` hold
   scratch that dies with its command and can go after hours. `launch-prompts/` and `workflow-artifacts/` are read back
   by the ACE Agents tab for as long as the run is interesting, so give them a longer horizon — and check
   `src/sase/ace/tui/models/artifact_files.py` and `_revive_artifacts.py` for how long they expect artifacts to exist
   before choosing a number.
3. Wire it to a path that actually runs: a periodic hook, ACE startup, or `sase doctor`. Prefer somewhere that cannot
   block an interactive command — the measured root took over two minutes just to `du`, so a first pass on a
   long-neglected directory must not run inline on the TUI's startup path. Consider capping work per invocation.
4. Report what it reclaimed, so `sase doctor` output or a log line makes the cleanup visible rather than silent.

**Verification.** Seed a temporary managed root with entries straddling the horizon, run the reaper, and confirm only
the stale ones disappear. Confirm a fresh entry created by a concurrently running command survives.

## Stop the test suite from writing into the developer's managed temp root

Owner phase: `sandbox_tmpdir`.

1. Make `get_sase_managed_tmpdir()` sandbox-aware under pytest instead of honoring the developer's `$SASE_TMPDIR`.
   `src/sase/core/state_write_guard.py` already models exactly this decision — `pytest_context_detected()` plus
   `PYTEST_SANDBOX_DIR_ENV_VAR`, which `tests/conftest.py:144-153` publishes from `tmp_path_factory.getbasetemp()`.
   Reuse it rather than adding a second mechanism.
2. Fail closed the way sase-9l does: when a pytest process is detected and no sandbox root is published, raise with a
   message naming the fix, rather than silently falling back to the real root. `assert_bead_store_write_sandboxed()` is
   the precedent to copy for wording and shape.
3. Leave production behavior untouched — outside pytest, `$SASE_TMPDIR` must still win, then `~/.sase/tmp`.
4. Extend the leak guard in `tests/_tmp_leak_guard.py` to watch the managed root too, now that the suite should never
   add entries to it. Keep the existing single-root snapshot/diff/allowlist machinery and add the second directory to
   `watched_temp_directories()`; the guard already handles a tuple of roots.

**Verification.** Re-run the experiment that proved the leak — snapshot the counts of `$SASE_TMPDIR/workflow-artifacts`,
`handoff`, and `wrappers`, run `tests/test_workflows_runner.py`, `tests/test_xprompt_processor_workflow_execute.py`, and
`tests/test_multi_prompt_launcher_serialization.py`, and confirm every delta is 0 where it was previously +6. Then run
the full `just test` and confirm the extended guard passes.

## Reap everything stale under the pytest scratch root

Owner phase: `scratch_reaper`.

1. Widen `tools/run_pytest:_reap_stale_pytest_runs()` so it prunes stale top-level entries under the scratch root
   generally, not only `pytest-<N>` and `garbage-*` under `pytest-of-*`. The root is derived from the repo path hash and
   is used by nothing else, so the reaper may own everything in it.
2. Keep the existing safety properties: honor a `.lock` newer than the horizon, check mtime before removing, ignore
   `OSError`, and never follow symlinks. Concurrent runs from the same workspace must not lose live scratch.
3. Remove `inline-snapshot-*` from the leak guard's `FOREIGN_ENTRY_PATTERNS` only if it is genuinely no longer needed;
   the plugin creates its directory at configure time, so the allowlist entry may still be required for the _guard_ even
   once the _reaper_ cleans up afterwards. Decide deliberately and leave a comment either way.
4. Extend `tests/test_run_pytest_tool.py`'s reaper coverage with a non-pytest-named stale entry and a fresh one.

**Verification.** Seed a scratch root with a stale `inline-snapshot-abc`, a stale `tmpXXXXXXXX/artifacts` tree, a fresh
`inline-snapshot-def`, and a `pytest-1` whose `.lock` is fresh. Run the reaper and confirm only the two stale entries
go. Then run `just test` twice and confirm the scratch root's entry count does not grow across runs.

## Contain the sase-github handoff diff files

Owner phase: `plugin_diffs`.

This phase touches only the `sase-github` repo. Open it with the `/sase_repo` skill.

1. Fix `src/sase_github/xprompts/gh.yml:141`. Replace `mktemp /tmp/sase-gh-XXXXXX.diff` with a `mktemp` into a managed
   directory — resolve `${SASE_TMPDIR:-$HOME/.sase/tmp}` and a `gh-diffs` subdirectory, create it, and use `mktemp -p`.
   Never hardcode `/tmp`. Keep the existing `rm -f` for the empty-diff case.
2. Fix `src/sase_github/scripts/new_pr_desc_get_context.py:75` the same way. It is Python inside the plugin, so it can
   import `get_sase_managed_tmpdir` from `sase.core.paths` and pass `dir=`; check the plugin's existing imports from
   `sase` to confirm that dependency direction is already established before relying on it.
3. Both files are consumed downstream — `diff_path` is a `path`-typed workflow output and `new_pr_desc.yml:34`
   interpolates `@{{ get_context.diff_file }}` — so do not add immediate deletion. The `managed_reaper` phase is what
   bounds them; put them under a subdirectory that phase's horizon rules cover.
4. Run that repo's own check suite before reporting done.

**Verification.** Run the `gh` workflow's diff step (or exercise the script) and confirm the file lands under the
managed root, not `/tmp`. Confirm `ls /tmp | grep -c sase-gh-` stays at 0 across a PR-description run.

## Clean up the sase-core Rust test temp directories

Owner phase: `core_test_dirs`.

This phase touches only the `sase-core` repo. Open it with the `/sase_repo` skill.

1. In `crates/sase_core_py/src/lib.rs`, change the four `#[test]` helpers — `temp_beads_dir`, `temp_notification_path`,
   `temp_telemetry_path`, `temp_agent_stats_root` — so their directories are removed when the test finishes. The
   idiomatic fix is `tempfile::TempDir`, returned to the caller so the guard lives as long as the test body; a bare
   `PathBuf` return cannot clean up.
2. Check whether other crates in the repo have the same pattern before declaring the sweep done — grep for
   `std::env::temp_dir()` across `crates/`.
3. Keep `cargo test` green, and run the repo's own check command.

**Verification.** Record `ls /tmp | grep -c sase-core-py-` and the `/var/tmp` equivalent, run `cargo test`, and confirm
neither count grows.

## One-time reclamation of the managed temp root

Owner phase: `reclaim_managed`.

Depends on `prompt_leak`, `sandbox_tmpdir`, and `managed_reaper`: reclaiming before the producers stop just invites the
directory to refill.

**This phase destroys data and must not run unattended.** Propose the commands through a confirmation gate (the
`sase_gate` skill) and let the user approve them, exactly as sase-96.7 did.

1. Summarize before deleting: entry count, total size, and the largest buckets. The prior measurement was 94,522 entries
   / 410 MB, dominated by 32,595 `*.md`, 22,411 `workflow-tmp_abc123-*`, and 17,589 `*.json`.
2. Batch the deletion — `find … -delete` or `xargs`, never a single glob over ~94k entries, which can exceed `ARG_MAX`.
   Add an age filter so a concurrently running command's scratch is not removed.
3. Preserve the live subdirectories the reaper now manages (`editors/`, `handoff/`, `wrappers/`, `workflow-artifacts/`,
   `launch-prompts/`, …) and anything whose ownership is unclear. Only sweep entries the sase code paths are known to
   have produced.
4. Note that `du` on this directory previously took over two minutes; prefer a single `find` pass that both counts and
   deletes over repeated full scans.
5. Report reclaimed bytes and the final entry count.

**Verification.** `ls "$SASE_TMPDIR" | wc -l` drops from ~94k to a small number, `df` shows the space returned, and a
subsequent agent launch plus a `just test` both still succeed.

## Land sase-96

Owner phase: `land`.

Depends on every other phase. This is the epic's closeout, carried over from the sase-96 landing that deferred to this
plan.

1. Re-verify the sase-96 phases still hold after all of the above: `/tmp` gains nothing across two back-to-back
   `just test` runs, the leak guard passes, and `just check` is green apart from the known pre-existing
   `test_diff_cache.py` pollution failure (see Non-goals) if that has not yet been fixed under its own bead.
2. Close the epic: `sase bead close sase-96`.
3. **After** closing, run `just symvision`. sase-96 has no `--epic-symbol` whitelist entries in the Justfile today, so
   nothing should expire — but `prompt_leak` may have retired `get_sase_tmpdir()`, so confirm no unused symbol remains
   and remove anything the linter reports.
4. Set `status: done` in the frontmatter of the sase-96 plan file, `sase/repos/plans/202607/tmp_space_exhaustion.md`.
5. Set `status: done` in this plan file's frontmatter as well.

**Verification.** `sase bead show sase-96` reports the epic closed, `just symvision` passes, and both plan files carry
`status: done`.

---

## Suggested execution order

`terminal_smoke`, `prompt_leak`, `sandbox_tmpdir`, `scratch_reaper`, `plugin_diffs`, and `core_test_dirs` are mutually
independent and can run in parallel. `managed_reaper` joins after `prompt_leak` and `sandbox_tmpdir`. `reclaim_managed`
follows `managed_reaper`. `land` is last.

`terminal_smoke` and `scratch_reaper` both edit `tools/run_pytest`. They touch different functions, but sequence them or
expect a small merge if they run concurrently.

If capacity forces serialization, `prompt_leak` alone stops the largest active producer, and `reclaim_managed` (after
its dependencies) is what returns the 410 MB and 94k dirents that are stuck today.

## Repo-wide requirements

- Every phase that changes files in the sase repo must run `just install` and then `just check` before reporting done —
  workspace virtualenvs are ephemeral and may hold stale dependencies.
- The `plugin_diffs` and `core_test_dirs` phases touch only `sase-github` and `sase-core` respectively, opened via
  `/sase_repo`, and must run those repos' own check commands.
- No phase may edit `sase/memory/*.md`, `AGENTS.md`, or the generated provider shims (`CLAUDE.md`, `GEMINI.md`,
  `OPENCODE.md`, `QWEN.md`). This plan does not constitute user permission for those edits.
- Prompt-file and artifact paths are read by the ACE Agents tab. Any phase that moves them must check
  `src/sase/ace/tui/models/artifact_files.py`, `_loaders/_workflow_loaders.py`, and
  `src/sase/ace/tui/actions/agents/_revive_artifacts.py` for readers that assume the old location.
