---
tier: tale
title: Isolate zsh PTY completion tests from the live candidate backend
goal:
  Resolve sase-13a for phase sase-1qn.4 while preserving real-shell assertions and
  timeout budgets.
size: medium
proposed_by: bbugyi200.athena.sase-1qn.4
proposed_with: codex/gpt-6-astra
bead: sase-1qn.4
create_time: 2026-11-01 15:15:28
status: wip
---

- **PARENT:**
  [202611/task_backlog_epic_4.md](https://github.com/sase-org/sase--plans/blob/main/202611/task_backlog_epic_4.md)
- **BEAD:**
  [sase-1qn.4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1qn/sase-1qn.4.md)

# Isolate the zsh completion PTY smoke tests from the live candidate backend

Complete phase `sase-1qn.4`, which owns the fix for flake task `sase-13a` under
`plan:202611/task_backlog_epic_4.md`. Keep the existing real-zsh alias completion
assertions and timeout budgets. Close only `sase-1qn.4` after verification; the parent
epic's land agent owns closing `sase-13a` and the epic.

## Scope and sizing

This is one medium coding task: fix the smoke-test subprocess boundary, add a regression
that detects accidental use of the inherited executable, and verify the affected module
under load. The phase has no earlier implementation or remaining-work notes. The parent
decision `optional_scope = keep_five_seeds` is final: do not add adjacent features or
edit memory. No source changes were made while authoring this plan.

## Evidence from the current tree

Investigation used clean sase commit `078e3da1d6a784cfc8fd2ab783273246318660c2` and zsh
5.9 on athena. The last change to `tests/completion/test_zsh_smoke.py` was `82fae375ba`
on 2026-09-15, before the first report of this flake. No later fix to this test has
landed. The phase currently has no `--epic-symbol` entries.

`_pty_capture_completion` in `tests/completion/test_zsh_smoke.py` starts an interactive
zsh, waits for `READY>` for up to eight seconds, sends the typed text plus TAB and the
capture widget binding, then waits up to five seconds for a unique capture marker. Each
invocation already has its own `ZDOTDIR`, and `compinit -u -D` avoids a shared compdump.
The installed zsh's `_retrieve_cache` and `_store_cache` default to
`${ZDOTDIR:-$HOME}/.zcompcache`, so that shell cache is also already per invocation.

The unexpected dependency is downstream of readiness. The static alias test supplies no
`bin_dir`, so its shell inherits the real executable search path. The generated
`_arguments` completion can request positional candidates while completing an option:

| Typed buffer     | Completed buffer     | Calls to a logged fixture executable |
| ---------------- | -------------------- | ------------------------------------ |
| `sbd sh`         | `sbd show `          | None                                 |
| `sbd show --for` | `sbd show --format ` | `sase completion candidates bead`    |

The call flows through `__sase_candidates`, `__sase_candidate_lines`, and `__sase_run`
in `src/sase/completion/emit_zsh_preamble.py`. `__sase_run` deliberately skips ephemeral
workspace virtualenv executables, making the inherited global executable and its backend
state relevant. Neighboring dynamic completion tests already use
`_write_logged_fixture_sase` or `_write_fixture_sase` to isolate this boundary.

An in-memory probe loaded the unchanged test module, wrote its normal generated script
to a disposable directory, and used `_write_logged_fixture_sase` with no candidates. It
timed the existing `_read_until` calls without changing them. Delaying only the
fixture's candidate response by six seconds produced the reported failure:

| Probe                              | Startup wait | Completion wait | Result                                                    |
| ---------------------------------- | ------------ | --------------- | --------------------------------------------------------- |
| Immediate empty candidate response | 0.343 s      | 0.281 s         | Exact `sbd show --format ` buffer                         |
| Six-second candidate response      | 0.339 s      | 5.025 s         | Capture-marker timeout; received only `b'sbd show --for'` |

Both probes logged exactly `<completion><candidates><bead>`. This proves that a slow
candidate subprocess can reproduce the precise symptom after shell startup. It is a
controlled reproduction, not proof of which operation inside the real backend stalled in
every historical full lane. The inherited executable dependency itself makes this static
shell test depend on unrelated host state and load.

## Implementation

1. Re-read `sase-1qn.4`, `sase-13a`, and the parent design through their audited
   commands. Check current history and notes for a landed fix before editing. Read
   `lint_and_test.md`. Preserve the parent's prohibition on `just check-full`.

2. Confirm the boundary on the current base before changing source. Run the affected
   parameters with a logged candidate executable; compare immediate and delayed replies,
   recording startup separately from the completion-marker wait. Exercise the unchanged
   module with bounded xdist concurrency using the repository's governed test runner. Do
   not repeatedly run the full suite to hunt for a probabilistic failure. Record the
   base SHA, worker count, runtime, exact failure text, and whether the natural flake
   was observed. If the natural failure does not recur, retain the controlled
   delayed-reply reproduction as evidence and state that limit explicitly.

3. Make `_pty_capture_completion` always use a test-owned `sase` executable. When the
   caller supplies `bin_dir`, keep that explicit fixture and its current behavior. When
   the caller omits it, supply a per-test fixture that records invocations and returns
   empty candidate output without invoking the real CLI. Reuse the existing fixture
   machinery where practical. Prepend the selected fixture directory to the child's PATH
   in both cases; keep the rest of PATH available for shell support commands. Ensure
   repeated captures do not collide on fixture directories or logs.

   Keep the generated live command specification, actual zsh/compsys execution, alias
   expansion, TAB input, and capture-widget assertion. Do not stub `_sase`,
   `_arguments`, the capture buffer, or the emitter. Do not suppress candidate calls
   globally: zsh legitimately asks for them in this context. The default empty provider
   makes that incidental request deterministic while explicit dynamic fixtures still
   test values.

4. Add a fast behavioral regression for the missing isolation. Put a logged sentinel
   `sase` executable in an inherited PATH directory, separate from the helper's local
   fixture, and invoke `_pty_capture_completion` without `bin_dir` for `sbd show --for`.
   Assert the exact `sbd show --format ` result and that the inherited sentinel was
   never invoked. A sentinel that logs and returns immediately is enough: this
   regression must fail on the old helper because it reaches the sentinel, without
   requiring a multi-second sleep in the permanent test suite. Also retain coverage that
   an explicitly supplied candidate fixture is used, including its existing call logs.

5. Review every caller of `_pty_capture_completion` for this same dependency. The
   default fixture should isolate the other static probes automatically; existing
   dynamic probes must continue to receive their supplied candidate values. Preserve all
   assertions for alias expansion, root print options, typed and nonmatching prefixes,
   embedded prompt markers, extendedglob restoration, and cache on/off call counts. Keep
   all current timeout budgets and shell-cache settings. No production backend, runtime
   grammar, Rust binding, CLI option, documentation, memory, or flake-baseline change is
   expected for this fixture defect.

## Verification and acceptance

- Show the new sentinel regression fails on the original helper and passes with the fix.
  The exact static alias assertion must still fail if completion is broken; do not
  weaken it to a prefix or merely assert that the capture marker arrives.
- Re-run both static alias parameters and the complete
  `tests/completion/test_zsh_smoke.py` module. Exercise cache enabled/disabled and
  dynamic fixture call counts unchanged.
- Run a bounded series of at least three complete module runs through the governed
  pytest runner with up to four xdist workers, respecting any lower admitted worker
  count. Use distinct temporary directories and record admitted concurrency, duration,
  and pass/fail counts. Repeat the inherited slow-provider probe after the fix to show
  it is bypassed and the completion remains within the existing five-second budget.
  Measure comparable startup/completion stages before and after; do not claim that
  injected delay is a natural loaded-lane reproduction.
- Run formatting as needed and `sase tool run check`. Do not run `just check-full`, add
  skips/xfails, raise timeouts, prewarm a shared live backend as the fix, or add flake
  suppressions. Use `/sase_monitor` if a verification command will outlast the turn. Do
  not finish while an inline command is still running.
- For any check failure, investigate it. If it reproduces identically on the clean base,
  append a `PROPOSED FOLLOW-UP:` note to this phase, citing an existing tracking task
  when known, and close the phase once its own work is complete. Do not create task
  beads or leave a verified phase open solely because of that baseline failure.

## Completion

Record the implementation and verification evidence on `sase-1qn.4`, including a
`TASK RESOLVED: sase-13a — ...` note with the base, before/after timings, sentinel
regression, loaded-run results, check result, and limits. Run
`sase bead epic-symbols sase-1qn.4` immediately before closing and resolve or re-key any
remaining entries to a still-open owner. Then run
`sase bead close sase-1qn.4 --note "<what was verified>"`. Leave `sase-13a`, `sase-1qn`,
and all ancestor beads to the epic lander. Submit the root agent's `/sase_final`
declaration for host-owned completion; do not create commits manually.
