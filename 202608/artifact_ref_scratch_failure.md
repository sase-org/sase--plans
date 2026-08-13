---
tier: epic
title: Fix the artifact-ref commit inventory's scratch-file failure and finish landing
  sase-fq
goal: The artifact-ref commit inventory stops returning an empty result on GitHub
  runners, the failure is fixed at the cause the runtime diagnostic actually names
  rather than at the budget hypothesis that was already disproved, every job in the
  sase CI workflow passes on master, and epic sase-fq is closed out.
phases:
- id: scratch-probe
  title: Identify the OS error behind the scratch-file failure on a CI runner
  depends_on: []
  size: medium
  description: 'scratch-probe: land a diagnostic that makes the next master CI run
    report the actual OS error and resource state behind CommitLogFailure::Scratch,
    and record the answer in the phase notes.'
- id: scratch-fix
  title: Fix the identified scratch-file failure at its source
  depends_on:
  - scratch-probe
  size: medium
  description: 'scratch-fix: fix the cause scratch-probe identified, carry the underlying
    io::Error into sase-core''s commit-inventory diagnostic, and confirm the parity
    test passes on master CI.'
- id: land-fq
  title: Close out epic sase-fq
  depends_on:
  - scratch-fix
  size: small
  description: 'land-fq: confirm every sase CI job is green on master, close epic
    sase-fq with a verification note, run just symvision, and mark both plan files
    done.'
proposed_by: bbugyi200.athena.sase-fq.land
parent_bead: sase-fq
create_time: 2026-08-06 07:04:59
status: done
bead_id: sase-fq.8
---

- **PARENT:** [202608/ci_master_red_recovery.md](https://github.com/sase-org/sase--plans/blob/main/202608/ci_master_red_recovery.md)
- **BEAD:** [sase-fq.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-fq/sase-fq.8.md)

# Plan: Fix the artifact-ref commit inventory's scratch-file failure and finish landing sase-fq

## Context

Epic sase-fq set out to restore master CI to green after six independent root causes. Five of the six are fixed and
confirmed green in real CI runs (see the LAND VERIFICATION note on bead sase-fq). The sixth, R6, is not fixed, and the
plan's hypothesis about it was wrong.

`tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py::test_commit_completion_rows_match_shared_inventory_and_resolve`
still fails on master at sase-fq's tip commit `7ffd5471a`:

- Run <https://github.com/sase-org/sase/actions/runs/31067580370>
- `test (3.13)` and `test (3.14)` both fail with `AssertionError: assert () == ('@commit:sas...6e833b27e730')` — the
  same empty-inventory shape the epic started with.
- `build-core`, `published-core-minimum-smoke`, `lint`, `visual-test`, and `perf-floors` are all green, so R1-R5 hold.

This is with `sase-core-rs` `0.18.2` installed (the release from phase sase-fq.6, which raised the commit-log budget
from 2s to 30s and added the `SASE_ARTIFACT_REF_COMMIT_TIMEOUT` override) and with the test itself setting
`SASE_ARTIFACT_REF_COMMIT_TIMEOUT=30` explicitly (phase sase-fq.7). **The budget was never the cause.**

### What the new diagnostic says

The one thing sase-fq.6 got unambiguously right was replacing the collapsed `Option` with a named `CommitLogFailure` and
reporting it on stderr. That paid off on the first CI occurrence. From the `test (3.13)` job log, captured stderr on the
failing test:

```
artifact-ref commit inventory: skipping repository sase at /var/tmp/sase-d1260045/pytest-of-runner/pytest-0/popen-gw1/test_commit_completion_rows_ma0/sase: could not open a scratch file for `git log` output; check that TMPDIR exists and is writable
artifact-ref commit inventory: skipping repository sase-core at /var/tmp/sase-d1260045/pytest-of-runner/pytest-0/popen-gw1/test_commit_completion_rows_ma0/sase-core: could not open a scratch file for `git log` output; check that TMPDIR exists and is writable
```

(twice each, because the test drives the inventory twice: once through the raw LSP binding and once through the prompt
completion rows.)

That message is `CommitLogFailure::Scratch` in `crates/sase_core/src/editor/completion.rs`, which is returned from
exactly two places in `commit_log_output`:

```rust
let mut stdout = tempfile::tempfile().map_err(|_| CommitLogFailure::Scratch)?;
let stdout_writer = stdout.try_clone().map_err(|_| CommitLogFailure::Scratch)?;
```

So either `tempfile::tempfile()` (an `open` under `std::env::temp_dir()`) or `try_clone()` (a `dup`) is returning an
error. **`CommitLogFailure::Budget` is not what fired.**

### What is already ruled out

- **The budget.** 30s, set two independent ways, and the `Budget` variant would have produced a different message.
- **A missing or unwritable `TMPDIR`, despite what the message guesses.** `tools/run_pytest`'s `_prepare_pytest_tmpdir`
  sets `TMPDIR` to the managed scratch root (`/var/tmp/sase-d1260045` in that run) and creates it. It demonstrably
  exists and is writable at the moment of failure, because pytest's own tmp directories for the very same test are
  inside it (`/var/tmp/sase-d1260045/pytest-of-runner/pytest-0/popen-gw1/test_commit_completion_rows_ma0/`). `TMPDIR` is
  not in `run_pytest`'s `PYTEST_ENV_UNSET_KEYS`, and no conftest fixture rewrites or deletes it.
- **A single bad worker.** It reproduces on `popen-gw1` under Python 3.13 and `popen-gw2` under Python 3.14 in the same
  run, and reproduced on all three legs at `1da5a3e27` and at `ab955c9ca`.
- **sase-side version skew.** `published-core-minimum-smoke` is green, so the installed core exports every required
  binding.

### What is not yet known

The current message throws the underlying `io::Error` away (`map_err(|_| ...)`), so the errno is unknown. The live
candidates, all consistent with "fails only on a 4-vCPU GitHub runner, never on a dev host":

- **`EMFILE`/`ENFILE`** — file-descriptor exhaustion. This fits both call sites: `open` and `dup` each fail with
  `EMFILE`. Worth checking against `RLIMIT_NOFILE` and the worker's open-fd count.
- **`ENOSPC`** — including inode exhaustion on the filesystem holding `/var/tmp`. This fits the timing correlation:
  `9672c5602` restored full worker parallelism on the runner (the 3.12 leg went from 3282s to 1405s), which multiplies
  peak concurrent scratch usage, and inode exhaustion specifically breaks creating a _new_ file while leaving reads and
  writes of existing files working.
- **An `O_TMPFILE`-related failure** in the `tempfile` crate on the runner's filesystem.

sase-fq.6 flagged exactly this gap in its own `PROPOSED FOLLOW-UP`: "spawn EAGAIN and tempfile EMFILE are equally
plausible and are now reported distinctly." This epic closes it with evidence instead of another hypothesis.

### Standing constraints inherited from sase-fq

- Do **not** loosen or retry the parity assertion in `tests/ace/tui/widgets/test_artifact_ref_completion_catalog.py`.
  The parity assertion is the point of that test.
- Do **not** silence the temp-leak guard or add suppression patterns to `tests/_tmp_leak_guard.py`.
- A silently empty inventory is a user-facing defect, not only a test failure: on a loaded machine a user typing
  `@commit:` gets no completions. Whatever fix lands must not restore a silent-empty outcome.

## Phases

### Identify the OS error behind the scratch-file failure on a CI runner

Land a diagnostic that makes the next master CI run answer the question, then read that run and record the answer.

The failure is deterministic in CI and has never reproduced locally, so the diagnostic has to reach CI. Prefer the
sase-side lever, because it needs no sase-core release round trip:

- In `test_commit_completion_rows_match_shared_inventory_and_resolve`, when the inventory comes back empty, emit the
  resource state before asserting. At minimum: the live `TMPDIR` value plus whether it exists and is writable,
  `os.statvfs(TMPDIR)` free blocks and free inodes, `resource.getrlimit(resource.RLIMIT_NOFILE)`, the open-descriptor
  count from `os.listdir("/proc/self/fd")`, and a direct Python attempt at `tempfile.TemporaryFile()` and `os.dup(fd)`
  reporting `OSError.errno` on failure. Attach it to the assertion message or print it, so it lands in the job log next
  to the existing stderr diagnostic.
- Keep the assertion itself unchanged: this phase adds evidence, it does not weaken or skip the test.
- Consider whether the same probe belongs behind an env flag rather than always-on, but do not let that stop it from
  running on master CI at least once.

Try to reproduce locally first anyway, now that the failing branch is known — for example by running the test with
`RLIMIT_NOFILE` lowered, and separately against a `TMPDIR` on a filesystem with no free inodes. A local reproduction
that matches the CI errno is worth more than another CI round trip, and it gives the next phase a regression test.

Then push to master, wait for the master CI run, and read the `test (3.13)` and `test (3.14)` job logs. Record in the
phase notes: the errno, the resource numbers, and which of the two `Scratch` call sites failed.

If the probe comes back showing something none of the candidates above predicted, stop and report that rather than
forcing a fix.

Verify with `just check-full` (not `just check` — this is landing into an epic's tree), and by confirming the probe
output actually appears in the CI job log.

### Fix the identified scratch-file failure at its source

Fix the cause `scratch-probe` named. The shape depends on the answer, so treat the following as the decision table
rather than a prescription:

- **Descriptor exhaustion** — find and fix what holds the descriptors (an fd leak in the suite is a real bug worth
  fixing on its own), and/or raise `RLIMIT_NOFILE` for the pytest run in `tools/run_pytest`. Prefer fixing a leak over
  raising the limit if a leak is what the numbers show.
- **Disk or inode exhaustion** — reclaim scratch space in the run. `tools/run_pytest` already reaps stale pytest run
  directories through `_reap_stale_pytest_runs`; check whether its horizon is long enough to leave several runs' worth
  of debris on a runner, and whether the four concurrent workers' peak usage is simply too high for the runner's disk.
- **An `O_TMPFILE` or filesystem-specific failure** — the fix belongs in sase-core: create the scratch file explicitly
  under a directory the caller controls, rather than relying on the `tempfile` crate's default probe.

Regardless of which branch applies, make one change in sase-core: **carry the underlying `io::Error` into the
diagnostic**. `CommitLogFailure::Scratch`, `::Spawn`, `::Wait`, and `::Read` all currently discard it via
`map_err(|_| ...)`, and the `Scratch` message's "check that TMPDIR exists and is writable" guess actively misled this
investigation — `TMPDIR` was fine. A real user hitting this deserves the errno too. This phase crosses the Rust core
backend boundary: open the checkout with `sase repo open sase-core -r "<reason>"` and use only the path it prints, land
through sase-core's normal review flow, get a release published (release-please cuts sase-core releases on merge to
master), then raise the `sase-core-rs` floor in `pyproject.toml` and refresh `uv.lock`, and record the released version
in the phase notes.

Add regression coverage for whatever the cause turns out to be, in whichever repo owns it. If a local reproduction was
found in `scratch-probe`, turn it into a test.

Verify with `just check-full`, and then by confirming on a master CI run that
`test_commit_completion_rows_match_shared_inventory_and_resolve` passes on all three `test` legs.

### Close out epic sase-fq

Finish the landing that sase-fq's land agent could not complete.

- Confirm every job in the sase CI workflow passes on master: `build-core`, `published-core-minimum-smoke`, `lint`,
  `test (3.12)`, `test (3.13)`, `test (3.14)`, `visual-test`, and `perf-floors`. Name the run in the close note.
- Close the epic: `sase bead close sase-fq --note "<what was verified>"`. The LAND VERIFICATION note already on sase-fq
  records the R1-R5 evidence and the follow-up disposition; the close note should add the R6 outcome and point at this
  epic.
- After closing, run `just symvision` (epic-symbol whitelist entries expire at close) and remove any stale entries and
  unused code it reports. A scan during the sase-fq landing found no `sase-fq` whitelist entries in the tree, so this is
  expected to be a no-op — but run it, and read `sase/memory/symvision.md` through `/sase_memory_read` before fixing
  anything it does report.
- Set `status: done` in the frontmatter of `@plan:202608/ci_master_red_recovery.md` (sase-fq's plan file) and in this
  plan file.

Do not force the close. If it is rejected, a phase bead is still open — finish or reopen it instead.

## Out of scope

- The load-flaky timing tests already tracked on beads sase-ct and sase-e2, and the contract-set runtime budget already
  routed to epic sase-fp. They are not caused by this work and each is corroborated on its own bead.
- Any further change to the commit-log budget itself. 30s with an override is fine and is not what fails.
- The pre-existing `RuntimeWarning: coroutine 'Timer._run_timer' was never awaited` and "changed the process working
  directory" warnings, carried over from sase-fq's out-of-scope list.
