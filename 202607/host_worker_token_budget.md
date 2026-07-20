---
tier: tale
title: Host-global pytest worker-token budget
goal: 'Parallel pytest runs share a crash-safe host-wide xdist worker budget, allowing
  a solo suite to use substantially more workers while keeping concurrent runs bounded,
  promptly admitted, observable, and compatible with the existing gate escape hatches.

  '
create_time: 2026-07-20 11:05:16
status: wip
prompt: 202607/prompts/host_worker_token_budget.md
---

# Plan: Host-global pytest worker-token budget

## Context and outcome

The current pytest protection in `tests/_suite_gate.py` leases one of two whole-suite slots. It prevents the
resource-exhaustion incident that motivated it, but it also serializes a third suite even though each admitted suite is
currently capped well below the host's safe aggregate worker capacity. `tools/run_pytest` independently chooses one
worker count from CPU and currently available memory, then `exec`s pytest; raw `pytest -n ...` runs enter through the
conftest hook and share only the coarse suite-slot limit.

Replace those two independent mechanisms with one host-global pool whose lock files each represent one xdist worker. The
pool must never admit more workers than its computed or explicitly configured budget, must release every grant when its
holder exits or is killed, and must preserve useful timeout/status diagnostics. A normal `tools/run_pytest` launch will
wait only for a small minimum grant, then greedily lease additional free tokens up to a per-run ceiling and use the
actual grant as `-n`. The ceiling will remain below the host budget so a solo suite becomes much faster without
monopolizing every token needed by the expected concurrent-agent workload. Direct parallel pytest controllers will lease
their requested worker count from the same pool.

This phase changes test infrastructure and documentation only. It does not change test selection, markers, assertions,
coverage settings, xdist distribution mode, or production runtime behavior, and it does not close or otherwise mutate
the parent epic.

## Capacity model and environment contract

Evolve the gate implementation around a multi-token lease abstraction. Compute the default host budget from CPU count,
reserved host capacity, `/proc/meminfo` `MemAvailable`, a conservative measured GiB-per-worker allowance, and a hard
upper bound no greater than the existing safe aggregate ceiling. Degrade safely when host facts are missing and clamp
small machines so at least a viable serial-or-small-parallel grant can be made. Before selecting constants, sample the
controller and worker RSS during a representative real suite run on the target host and record the conservative
calibration in code comments or documentation.

Keep the established controls usable while documenting their worker-token semantics:

- `SASE_TEST_GATE_DISABLED=1` remains the deliberate ungoverned escape hatch.
- `SASE_TEST_GATE_DIR` selects the shared pool directory; the default becomes a UID-scoped pytest-token directory.
- `SASE_TEST_GATE_TIMEOUT` controls the bounded wait and retains actionable holder metadata in status and timeout text.
- `SASE_TEST_GATE_SLOTS`, when set, remains honored as an explicit host-wide capacity override, now counted in worker
  tokens rather than whole-suite admissions.
- `SASE_PYTEST_WORKERS` retains precedence over automatic per-run sizing but not over the global safety bound: acquire
  exactly that positive worker request or report an actionable configuration error when it cannot fit the configured
  pool, rather than silently oversubscribing or waiting forever.

Add narrowly named controls for the automatic run floor and ceiling only if they are needed for operational tuning;
validate every numeric override and document it alongside the retained variables. An internal governed-run marker must
distinguish tokens already leased by `tools/run_pytest` from raw pytest controllers. A governed parent must also export
the existing disabled marker to descendants so nested pytest calls cannot deadlock against inherited locks. Invalid or
internally inconsistent settings fail before pytest starts with the relevant variable names and safe recovery options.

## Crash-safe token leases

Refactor `tests/_suite_gate.py` from a single-slot `SuiteGate` into a lease that can own several numbered lock files.
Each candidate token uses nonblocking `fcntl.flock`, writes PID/start/argv and grant metadata, and remains open for the
lease lifetime. Acquisition waits until the effective floor can be acquired atomically from the caller's perspective; if
a partial attempt cannot reach the floor, release that attempt before polling so contenders do not deadlock or hoard
fragments. Once the floor is secured, greedily take currently free tokens up to the request ceiling. The pool directory
and numbered token set are shared by every checkout for the same UID.

Preserve the current periodic status line and bounded timeout, updated to show requested/granted token counts and
deduplicated holder metadata. Release restores inherited environment values and closes every lock. For the runner path,
clear close-on-exec on leased descriptors before `os.execv` so the pytest controller owns the locks for exactly its
lifetime; kernel lock release remains the SIGKILL-safe cleanup mechanism. Ensure acquisition failures and normal test
shutdown close partial/full leases deterministically.

## Runner and direct-pytest integration

Update `tools/run_pytest` to choose whether parallelism is allowed before building the pytest command. For normal
parallel invocations, obtain a lease first, derive `-n` from the granted token count, mark the environment as already
governed, sanitize the existing workflow variables, and then `exec` pytest with inheritable lock descriptors. Serial
inline-snapshot update modes continue to skip both xdist and token acquisition. Keep marker selection, caller-relative
selector normalization, coverage arguments, and the current `--dist=loadfile` default unchanged for the later scheduling
phase.

Retain the conftest backstop for raw parallel pytest. A controller not already governed determines its effective xdist
worker request, leases exactly that many tokens from the same budget, and holds them until `pytest_unconfigure`.
Explicitly exempt serial runs and xdist worker subprocesses. Handle xdist's supported numeric/automatic worker forms in
the same way xdist resolves them, or fail clearly when an exact safe grant cannot be established; never let a raw
parallel invocation bypass accounting merely because its option was non-numeric.

## Verification

Expand `tests/test_suite_gate.py` with isolated temporary pools and tiny polling intervals to cover:

- budget arithmetic across CPU, memory, explicit capacity, missing/malformed meminfo, small hosts, and invalid values;
- floor acquisition, greedy growth to the per-run ceiling, exact explicit requests, partial-attempt rollback, and a
  proof that simultaneous leases never exceed the pool;
- prompt admission of multiple scaled-down runs when the pool/ceiling leaves floor capacity, plus timeout/status output
  that identifies holders and requested capacity;
- release and environment restoration on normal teardown, and token reacquisition after a subprocess holding several
  inherited descriptors is killed with `SIGKILL`;
- governed-run, disabled, xdist-worker, and serial exemptions, plus raw-controller acquisition for numeric and automatic
  xdist requests.

Expand `tests/test_run_pytest_tool.py` to prove that the granted lease, not the old local heuristic, determines `-n`;
that `SASE_PYTEST_WORKERS` requests exact governed capacity; that serial snapshot modes never acquire; that descriptors
and internal markers are prepared before exec; and that existing command construction, selector normalization, marker
selection, environment sanitization, and coverage behavior stay intact. Use dependency injection or small pure helpers
for host facts and acquisition decisions so unit tests never touch the real host pool.

Run the focused gate/runner tests first, followed by related conftest smoke coverage. Exercise several concurrent tiny
subprocess runs against an override directory and capacity to verify admission, aggregate bounds, timeout behavior, and
crash release without consuming the real host pool. Run `just install` before repository checks as required for an
ephemeral workspace, then run `just check`; address all failures and rerun the affected checks and `just check` until
green.

## Documentation and completion

Update `docs/development.md` to explain automatic worker grants, the solo-versus-concurrent behavior, raw pytest
backstop, safety ceiling, and escape hatch. Update the environment table in `docs/configuration.md` for the retained and
new variables, including the changed meaning of `SASE_TEST_GATE_SLOTS` and the exact-request behavior of
`SASE_PYTEST_WORKERS`. Mention the measured memory calibration and distinguish this capacity work from the later epic
phases that optimize scheduling and test cost.

Review the final diff for accidental test-selection, assertion, coverage, or distribution changes. Confirm the working
tree contains only this phase's intended implementation/docs/tests, record concise implementation and verification notes
on `sase-86.1` if useful, and close `sase-86.1` only after every required check passes. Leave `sase-86` and every other
child bead unchanged.
