---
create_time: 2026-07-13 12:21:30
status: wip
prompt: 202607/prompts/stale_runner_code_refresh.md
tier: tale
---
# Plan: Prevent stale-code `SddMaterializationError` launch failures after dependency waits

## Incident summary (2026-07-13, agent `sase-5w.4`)

Agent `sase-5w.4` failed at launch (RUN 11:59:08 → FAILED 11:59:09) with:

```
SddMaterializationError: SDD store record
/home/bryan/projects/github/sase-org/sase/.sase/sdd-store.json was written by a newer or
unknown sase version. Upgrade and restart sase; refusing to touch it.
```

The record on disk was **valid** for current code. The failing component was the agent's own long-lived runner process,
whose in-memory code predated a schema change that landed while the runner was blocked in a dependency wait.

### Exact failure chain

1. The `sase-5w` epic launched all phase agents up front. Each phase got a detached `run_agent_runner.py` process that
   imports sase code **once at spawn time**, then blocks in `wait_for_dependencies()`
   (`src/sase/axe/run_agent_runner.py:237`) until its upstream phase completes. `sase-5w.4`'s runner was spawned with
   pre-rename code and waited ~2 hours (WAIT 10:01:20). The traceback line
   (`prepare_linked_repo_workspaces_if_needed(..., cl_name=cl_name)` at line 340) matches the pre-rename source,
   confirming the frozen-code window.
2. Phase `sase-5w.2` — **a phase of the same epic** — landed the breaking `companion_repos → sidecar_repos` rename
   (commit `3cf8ea2bf`, 11:24). The primary checkout (which the uv-tool _editable_ install points at) fast-forwarded at
   11:26, so every _newly spawned_ process ran post-rename code.
3. At 11:56 a freshly spawned post-rename process re-materialized and rewrote `.sase/sdd-store.json` with the canonical
   new spelling (`storage: sidecar_repos`, `sidecars` key). New code intentionally reads legacy spellings but serializes
   canonical ones.
4. At 11:59:08 `sase-5w.3` finished; the waiting `sase-5w.4` runner woke and ran linked-repo prep, which reads the store
   record (`src/sase/linked_repos.py:247`). Its pre-rename `_STORAGE_VALUES` did not contain `sidecar_repos`, so
   `_load_sdd_store_record()` raised `_foreign_record_error()` (`src/sase/sdd/_store_records.py:170`) and the agent
   failed in 1s.

### Why this recurs

This is structural, not a fluke — it has forced three manual "upgrade + relaunch" rituals in four days (`747d9be32` Jul
10, `4c40d5af8` Jul 11, `3cf8ea2bf` Jul 13):

- SASE dogfoods itself: the editable install auto-fast-forwards on every merge, while runner processes live for hours in
  dependency waits.
- **Any epic that changes the SDD store record schema in phase N is guaranteed to strand phases N+1..**: their runners
  were spawned with old code at epic launch, and phase N's landing causes the record to be rewritten before they wake.
- More SDD epics are queued; more schema evolution is expected.

### What is and is not justified about the current validation

- **Justified:** refusing to _rewrite/clobber_ a record written by unknown code (`6df95bbec` "preserve unknown store
  records"). Keep this write guard fully intact.
- **Not justified:** a hard read failure that kills an agent launch, with a message ("Upgrade and restart sase") that
  misdiagnoses the situation — the on-disk install was already current; only the long-running process was stale. A fresh
  process (retry/relaunch) always succeeds.

## Goals

1. Agents that spend hours in dependency waits must not fail because sase code evolved while they waited — they should
   transparently run post-wait work with current code.
2. When a version-skew read failure _does_ still occur, the error must state the real cause and the real remedy (retry
   the agent), and be classified as retryable.
3. Do not weaken the foreign-record write protection.

## Non-goals

- No changes to the SDD store record schema or its read/write compatibility rules.
- No dual-spelling/transition-window serialization scheme (does not help `schema_version` bumps, only renames).
- No graceful-degradation fallback in linked-repo resolution that silently guesses sidecar names: masking version skew
  at one read site still leaves later same-process reads (e.g. the commit finalizer,
  `src/sase/llm_provider/commit_finalizer.py`) to fail _after_ the agent did all its work. Refreshing the process fixes
  every downstream read at once.
- No `sase-core` (Rust) changes — this is Python process-orchestration behavior local to this repo.

## Design

### 1. Primary fix: re-exec the runner at the wait→run boundary when its code is stale

In `run_agent_runner.main()`, immediately after `wait_for_dependencies()` returns (before `detect_repeat_stop()` /
`wait_for_runner_slot()` / any workspace mutation):

- **Capture a code-identity token at process start** (module import time of the runner): the git HEAD commit of the sase
  source checkout, resolved via the existing helpers (`sase.version._sources.source_root()` for the editable install
  root and the git probe in `sase.version._git`; a lightweight `.git/HEAD` read is acceptable if the full probe is too
  heavy). If the identity cannot be determined (non-editable install, no git metadata), the feature is inert and
  behavior is unchanged.
- **Re-check after the dependency wait.** If the identity differs and the guard env var is not set, log one line to the
  runner log (old sha → new sha) and re-exec in place: `os.execv(sys.executable, [sys.executable] + sys.argv)` with
  `SASE_RUNNER_CODE_REFRESHED=1` set in the environment.
- **Guard against loops:** when `SASE_RUNNER_CODE_REFRESHED=1` is present, never re-exec again; clear/ignore it for
  children so spawned sub-agents are unaffected.

Why re-exec (vs alternatives):

- `os.execv` keeps the **same PID**, so workspace claims (held by PID) and the detached process's stdout/stderr
  redirections survive without any handoff machinery. The spawn-on-retry flow (`retry_transfer_from_pid`) was considered
  and rejected as primary: it changes PID, requires claim transfer, and pollutes retry-attempt metadata.
- On the refreshed pass, `wait_for_dependencies()` takes its existing "dependencies already satisfied" fast path
  (`src/sase/axe/run_agent_wait.py:324-344`) — it does not rewrite `waiting.json` and returns immediately.
- Pre-wait `main()` steps re-run and are effectively idempotent: `setup_artifacts_directory` resolves the same dir,
  `write_submitted_xprompt_artifact` and `extract_directives_and_write_meta` rewrite identical content, telemetry
  exit-push never fired (exec replaces the process image before exit).

Implementation care points:

- Place the check _after_ `wait_for_dependencies()` but _before_ `detect_repeat_stop()` and `wait_for_runner_slot()` so
  the refreshed process redoes repeat-stop detection and slot claiming itself (neither has happened yet on the first
  pass).
- Only trigger when an actual blocking wait occurred (`has_dependency_wait` true). Do not re-exec on the fast "already
  satisfied" path — the process is seconds old.
- `_record_wait_completed_at` runs again on the refreshed pass; make it (or the caller) keep the first recorded value if
  already present so WAIT/RUN timestamps stay accurate.
- Suppress the duplicate start banner on the refreshed pass (cosmetic; keyed off the guard var).
- `preprocess_prompt_xprompts` re-resolves the submitted prompt on the refreshed pass. Document this: if xprompt
  definitions changed during the wait, the fresh resolution is used — this is consistent with "run post-wait work with
  current code".
- SIGTERM during wait (`was_killed()`) must keep its current semantics: check kill state before deciding to re-exec,
  never re-exec a killed runner.

### 2. Correct the misleading error message

Reword `_foreign_record_error()` in `src/sase/sdd/_store_records.py` to describe both real cases and the real remedy,
e.g.:

> SDD store record `<path>` uses a format this process does not understand. Either this long-running sase process
> predates the record (retry/relaunch the agent — a fresh process will read it), or this sase install is older than the
> record's writer (upgrade sase).

Keep the "refusing to touch it" write-guard semantics untouched.

### 3. Classify foreign-record read failures as retryable

Ensure `SddMaterializationError` raised during launch prep surfaces through the runner error path
(`src/sase/axe/run_agent_runner_errors.py` / `run_agent_exec_retry.py::is_retryable_error`) as a retryable failure so
the `r retry` affordance (and any configured auto-retry) is the obvious, working remedy for the residual windows the
re-exec cannot cover (e.g. a schema change landing mid-run before the commit finalizer's read). If the
retry-classification mechanism is config/pattern-driven, add the pattern to the default config (and remember
`src/sase/default_config.yml` if a config value is involved).

## Testing

- Unit tests for the code-identity helper: stable token for unchanged tree, changed token after simulated HEAD move,
  `None` (inert) when no git metadata is resolvable.
- Runner-level test with `os.execv` monkeypatched: after a simulated dependency wait with a changed identity token,
  assert exec is invoked with the original argv and the guard env var; assert no exec when the token is unchanged, when
  the guard var is already set, when no blocking wait occurred, or when the runner was killed during the wait.
- Test that the refreshed pass preserves the originally recorded wait-completion timestamp and does not rewrite
  `waiting.json`.
- Message/regression tests for `_foreign_record_error` and the retryable classification.
- Existing store-record read/write compatibility tests must pass unchanged (write guard intact).

## Risks

- **Non-idempotent pre-wait side effect missed in the audit** → mitigated by re-running only the well-reviewed `main()`
  prefix, gating on the guard var, and covering the pass with tests.
- **Identity probe cost** — one git read at spawn and one per completed wait; negligible next to a multi-hour wait, zero
  for non-waiting agents.
- **Exec fidelity** (env/argv/cwd): argv is replayed verbatim and the environment is inherited across exec. The pre-wait
  `enter_agent_workspace` chdir means cwd at exec time is already the agent workspace; the refreshed pass re-runs that
  chdir itself, so cwd drift is not a concern. Covered by the runner-level test.
