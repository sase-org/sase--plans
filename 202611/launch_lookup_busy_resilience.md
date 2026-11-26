---
tier: epic
title: Make agent launches ride out a busy agent artifact index
goal: "A detached agent runner survives a transiently busy agent artifact index or
  dismissed-bundle index instead of failing its launch, while hot callers keep bounded
  single-attempt lookups, fail-closed semantics hold, runner failure hints are typed and
  honest, and the next busy failure names its SQLite lock class.

  "
decisions:
  patience_budget:
    ask:
      How long should a background agent runner wait out a busy agent index before
      failing the launch?
    choices:
      s60:
        Covers checkpoint and WAL-recovery stalls on a loaded host; a wedged index fails
        in a minute
      s30: Fails a wedged index sooner but leaves less headroom on a loaded host
      s300:
        Tolerates very long stalls but hides a wedged index behind a long silent boot
    default: s60
    why: Far beyond any healthy lock hold, yet short next to a dependency wait
    answer: s60
phases:
  - id: runner-lookup-patience
    title: Opt-in busy patience at the lookup facades, installed by the agent runner
    depends_on: []
    size: medium
    description:
      "runner-lookup-patience: add a process-wide LookupPatience policy and
      retry_while_busy helper, wrap the artifact and dismissed-bundle lookup facades so
      busy-incomplete results retry with capped backoff under patience, install it in
      the agent runner entry with concise retry log lines, and test the default
      single-attempt path stays unchanged."
  - id: typed-transient-classification
    title: Typed host lookup failures, provider-retry exclusion, honest runner hint
    depends_on: []
    size: small
    description:
      "typed-transient-classification: add an IndexLookupUnavailable base with a
      transient flag for both lookup exceptions, exclude host lookup failures from
      provider retry via is_host_owned_failure, and make record_runner_error print a
      typed transient or index-repair hint instead of the coincidental provider match."
  - id: busy-diagnostics
    title: SQLite extended codes in sase-core busy lookup diagnostics
    depends_on: []
    size: small
    description:
      "busy-diagnostics: in sase-core, decorate busy/locked SQLite errors on the
      artifact-index and dismissed-bundle lookup read paths with the extended code,
      keeping busy classification, wire shapes, and bindings unchanged."
proposed_by: bbugyi200.athena.1gu
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 14:45:28
status: wip
---

- **PROMPT:**
  [prompts/202611/launch_lookup_busy_resilience.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202611/launch_lookup_busy_resilience.md)

# Make agent launches ride out a busy agent artifact index

## Incident

On 2026-11-26 at 14:24:21 an apollo `bob-cli` agent
(`%id(5, clan=bob-cli-qp, bead=bob-cli-qp.5)` with `%w(bob-cli-qp.3,bob-cli-qp.4)`)
failed 4 seconds after start, before any provider work. Its run output
(`~/.sase/workflows/202611/gh_bobs-org__bob-cli_ace-run-261126_142421.txt` on apollo)
has this traceback:

`run_agent_runner._run_agent` → `run_agent_runner_bootstrap.bootstrap_agent_run` →
`run_agent_directives_extract.extract_directives_and_write_meta` →
`run_agent_directives_flow.resolve_wait_state` → `wait_target_binding.bind_wait_target`
→ `_classify_reference` → `agent.names._lookup_named.find_named_agent` → raises
`AgentNamedLookupUnavailable: agent name lookup is incomplete (availability=busy; source=fallback; database is locked)`.

## Root cause

- `src/sase/core/agent_artifact_lookup.py::lookup_agent_artifacts` takes the
  process-local index lock with a 50 ms bound. It then calls the Rust
  `lookup_agent_artifacts` binding (sase-core
  `crates/sase_core/src/agent_scan/index/lookup.rs`). That binding opens the index
  read-only with `AGENT_ARTIFACT_INDEX_PROJECTION_BUSY_TIMEOUT` (250 ms).
- If SQLite stays locked past 250 ms, `recover()` runs the shared bounded fallback
  (`lookup_recovery.rs`, `LOOKUP_FALLBACK_BUDGET` = 200 directories). Apollo has about
  9.2k ace-run artifact directories and a 137 MB index, so the fallback is always
  truncated there. The result is `complete=false`, and `_find_named_agent_candidates`
  raises.
- The raise is deliberate. The indexed-lookup epic
  (`plan:202611/indexed_agent_lookups.md`, bead `sase-1t3.6.1`) requires that an
  unavailable lookup never become an authoritative miss. That contract stays. The defect
  is that a detached background runner, which has no latency constraint, gets one 250 ms
  attempt, the same budget as a TUI hot path. Every launch-time identity lookup has this
  exposure:
  - `%w` binding through `find_named_agent`.
  - Clan and session classification (`find_agent_clan` and `find_agent_session`, which
    raise `AgentArtifactGroupLookupUnavailable`).
  - Fork waits (`agent/fork_waits.py`).
  - Post-wait context resolution (`axe/run_agent_refs.py`).
  - The dismissed-bundle fallback (`core/dismissed_bundle_lookup.py`). It uses a
    separate archive SQLite store with its own short busy timeout.
- Apollo is a loaded host: load average 13–15 on 16 cores. Its other SQLite stores log
  `database is locked (extended_code=261)`, which is `SQLITE_BUSY_RECOVERY`. The
  artifact-index lookup diagnostic carries no extended code, so this incident's lock
  holder (WAL recovery, the TRUNCATE checkpoint in maintenance, or something else)
  cannot be identified after the fact.

## Secondary finding: the "retryable" hint was a coincidence

`src/sase/axe/run_agent_runner_errors.py::record_runner_error` printed "This runner
failure is retryable" because `find_retry_config_for_error(summary)` scans every
provider's patterns case-insensitively. agy's gRPC pattern `UNAVAILABLE` matched the
class name `AgentNamedLookupUnavailable`. The same cross-provider scan runs in
`src/sase/axe/run_agent_exec_retry.py`. As a result, a host lookup failure raised during
the exec phase can start a provider retry (agy waits 60/300/1800 s and sends the
"transient provider failure" nudge) for any provider. Host finalizer failures are
already excluded through `is_host_finalizer_failure`; host lookup failures are not.

## Goals

1. A background agent runner rides out a busy agent index for a bounded patience window
   instead of failing on its first 250 ms attempt.
2. Interactive and hot callers (TUI, gateway, axe chops, CLI commands) keep today's
   single-attempt bounded behavior.
3. The fail-closed contract holds. Once patience runs out, the same typed exception is
   raised with the last diagnostic. A busy or incomplete lookup is never turned into a
   miss.
4. Runner failure output describes retryability from typed classification, not by
   coincidence. Host lookup failures never trigger provider retry.
5. The next busy failure records the SQLite extended code, so its lock holder can be
   identified.

## Non-goals

- Automatic relaunch of failed agents. The auto-restart healer stays specific to update
  skew. If a lock outlives the patience window, the launch should fail loudly, and the
  TUI `R` retry stays the manual path.
- Raising the 200-directory fallback budget or making the fallback exhaustive. That
  would break the "never a source scan on the read path" contract.
- Tool-run-store busy failures. That is a separate store, and earlier work under
  `sase-1ou.2` covered it.
- Root-causing the specific lock holder on apollo. Phase `busy-diagnostics` makes the
  next occurrence diagnosable.
- Narrowing agy's `UNAVAILABLE` provider pattern itself.

## Governing decisions

- `decisions:rust-core-required`: lookup, ranking, and bounded recovery stay in
  sase-core, unchanged. Patience is caller policy layered around the unchanged binding
  call. It does not reimplement or fall back from any core behavior. The only core
  change (phase `busy-diagnostics`) is diagnostic text and needs no wire or binding
  change, so it needs no `sase-core-revision.txt` move.

## Phase runner-lookup-patience

Size: medium. Repo: sase.

Add an opt-in, process-wide busy-patience policy at the two Python lookup facades, and
install it in the agent runner.

1. Add a small module, for example `src/sase/core/index_lookup_patience.py`, with:
   - A frozen `LookupPatience` with `budget_seconds`, `initial_delay_seconds` (0.25),
     `max_delay_seconds` (5.0), and an optional `on_retry` callback that receives a
     small event: lookup label, attempt number, seconds waited, next delay, last
     diagnostic, and whether the attempt recovered or gave up.
   - Process-wide `set_process_lookup_patience(patience | None)`, a getter, and a
     context manager that restores the previous value. Use process-global state, not a
     contextvar: runner helper threads must see the policy.
   - `retry_while_busy(attempt, *, is_busy, diagnostic, lookup, sleep=time.sleep, monotonic=time.monotonic)`.
     With no patience installed (the default), run exactly one attempt, bit-for-bit
     today's behavior. With patience, retry while `is_busy(result)` until the deadline,
     using exponential backoff capped at `max_delay_seconds`, and never sleep past the
     deadline. Return the last result; callers raise exactly as they do today. Swallow
     and ignore exceptions from `on_retry`.
2. The busy predicate is `availability == "busy" and not complete`.
   - Return a complete busy result (an exhausted fallback) immediately.
   - Do not retry `missing`, `stale_schema`, `newer_schema`, or `corrupt`. Those need
     maintenance outside the lookup, and waiting does not fix them.
3. Wrap the single binding call in
   `sase.core.agent_artifact_lookup.lookup_agent_artifacts`. This one choke point covers
   name, session, clan, and pointer targets, and so both `AgentNamedLookupUnavailable`
   and `AgentArtifactGroupLookupUnavailable`. Wrap
   `sase.core.dismissed_bundle_lookup.lookup_named_dismissed_bundle` the same way.
   - Each attempt takes the process-local lock again with its existing 50 ms bound.
   - Never hold the process-local lock across a backoff sleep.
   - A forced-busy attempt (process lock not acquired) is retried like a SQLite busy
     one.
4. Install the policy early in `src/sase/axe/run_agent_runner.py::main()`, before
   `_run_agent`, so it covers bootstrap, the dependency wait, the refreshed post-wait
   pass, and launch-phase lookups. Use a module constant for the budget (see
   `patience_budget`).
   - The runner's `on_retry` prints one concise stderr line to the run output on the
     first busy retry, then at most one line about every 10 seconds, then a final
     "recovered after N s" or "still busy after N s; giving up" line.
   - Check whether the lean waiter entry (`src/sase/axe/run_agent_lean_wait.py::main`)
     reaches either facade. If it does, opt it in too.
   - Do not opt in the TUI, the gateway, axe chops, or CLI commands.
5. Tests, with an injected clock and sleep and no wall-clock sleeps:
   - `retry_while_busy`:
     - no patience: one attempt
     - busy then ready: recovers
     - busy throughout: last result returned, expected backoff sequence, deadline
       respected
     - complete-busy, corrupt, and missing results: not retried
     - a raising `on_retry`: ignored
     - the context manager: restores the previous policy
   - Facades: patch the Rust bindings to return busy-incomplete payloads followed by
     ready payloads. Assert a retry happens only under patience, and assert no
     process-lock hold across sleeps.
   - `find_named_agent`: busy once then a hit returns the `NamedAgent` under patience
     and raises `AgentNamedLookupUnavailable` without it. The existing raise test in
     `tests/test_agent_names_indexed_lookup.py` stays green. Add the same pair for one
     group lookup (`find_agent_clan` or `find_agent_session`).
   - Runner: the entry installs the patience policy, and the retry-line formatter's
     output is stable.

Acceptance: under runner patience, a lookup whose index is busy for less than the budget
returns the same result as an idle index. Without patience, behavior and tests are
unchanged. After the budget is exhausted, the same typed exception is raised with the
last diagnostic. Run `sase tool run check` in sase.

## Phase typed-transient-classification

Size: small. Repo: sase. Independent of the other phases.

Classify host lookup unavailability by type, keep it out of provider retry, and make the
runner's retry hint honest.

1. Add a shared base exception, for example `IndexLookupUnavailable(RuntimeError)` in
   `src/sase/core/index_lookup_errors.py`.
   - It exposes `availability`, `source`, `diagnostic`, `truncated`, and a `transient`
     property that is true only for `availability == "busy"`.
   - Make `AgentNamedLookupUnavailable` (`src/sase/agent/names/_lookup_named.py`) and
     `AgentArtifactGroupLookupUnavailable`
     (`src/sase/core/agent_artifact_group_lookup.py`) subclass it.
   - Keep their constructors, messages, and `RuntimeError` ancestry unchanged, so
     existing `except` sites and hijack handling behave identically.
2. In `src/sase/llm_provider/retry_config.py`, add `is_host_owned_failure(exc)`. It
   returns `is_host_finalizer_failure(exc)` or whether any exception in the
   cause/context chain is an `IndexLookupUnavailable`.
   - `src/sase/axe/run_agent_exec_retry.py` uses it where it now calls
     `is_host_finalizer_failure`, and logs "host lookup failure; skipping provider
     retry".
   - Keep `is_host_finalizer_failure` exported.
   - Genuine provider error text that contains "UNAVAILABLE" must still match agy's
     pattern.
3. In `src/sase/axe/run_agent_runner_errors.py::record_runner_error`, replace the
   pattern-only hint:
   - Transient `IndexLookupUnavailable` in the chain: print that the failure is
     transient (the agent index stayed busy), with the diagnostic, and that retrying the
     agent is safe once the index is idle.
   - Non-transient lookup unavailability: print the availability, and point to
     `sase agent index status` and the repair subcommand it recommends. Verify the
     actual command names first.
   - Other host-owned failures: print no provider retry hint.
   - Anything else: keep the existing provider-pattern hint.
4. Tests:
   - `tests/test_llm_provider_retry_config.py`: `is_host_owned_failure` for direct and
     chained (`__cause__` and `__context__`) lookup failures, plus a non-host exception
     whose text contains "Unavailable" still matching provider patterns.
   - Runner-error tests for each hint branch. Today's incident summary must no longer
     print the provider-retryable line.
   - An exec-retry test: an `AgentNamedLookupUnavailable` raised from an attempt returns
     `"raise"` without consulting provider configs or writing retry state.

Acceptance: a busy-lookup failure prints a typed transient hint and never the
coincidental provider line. Host lookup failures never enter provider retry. Provider
pattern behavior for genuine provider errors is unchanged. Run `sase tool run check` in
sase.

## Phase busy-diagnostics

Size: small. Repo: sase-core (open it with `sase repo open sase-core` and read its
`AGENTS.md`). Independent of the other phases.

Make busy lookup diagnostics identify the lock class.

1. Add a helper beside the index storage code, for example
   `sqlite_error_diagnostic(&rusqlite::Error) -> String`. For a `SqliteFailure` whose
   code is `DatabaseBusy` or `DatabaseLocked`, it formats
   `"{error} (extended_code={n})"`, matching the `tool_run` precedent in
   `crates/sase_core/src/tool_run/mod.rs`. Other errors keep their current text.
2. Use the helper only on the lookup read path:
   - In `agent_scan/index/lookup.rs` and its `storage.rs` helpers: the read-only open,
     busy-timeout setup, schema-version read, and query prepare/step.
   - In `agent_archive/store.rs`: `lookup_named_dismissed_bundle` and
     `open_named_lookup_store`.
3. `is_busy_error` and `named_lookup_error_availability` must still classify the
   decorated text (it still contains "database is locked").
4. Change no wire shape, binding, or availability semantics. No sase pin move is needed.
5. Tests: reuse the existing busy-index fixture approach in
   `agent_scan/index/tests/lookup.rs` and the archive store tests. Assert that a real
   SQLite busy failure reports `availability = busy` and a diagnostic containing
   `extended_code=`. A forced-unavailability result keeps no diagnostic.

Acceptance: a busy lookup diagnostic names the SQLite extended code. Classification is
unchanged. Run `sase tool run check` in sase-core. Use a conventional commit subject
such as `fix(agent-scan): report SQLite extended codes in busy lookup diagnostics`.

## Phase graph

All three phases are independent and can run in parallel. They touch disjoint files:

- `runner-lookup-patience`: the Python lookup facades and the runner entry.
- `typed-transient-classification`: the lookup exception classes, retry classification,
  and runner error output.
- `busy-diagnostics`: sase-core only.
