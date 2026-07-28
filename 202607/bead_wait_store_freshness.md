---
tier: tale
title: Refresh the canonical bead store so closed-bead waits release on their own
goal: An agent parked on `%wait(bead=<id>)` starts within roughly a minute of that
  bead being closed and pushed to the SDD plans sidecar remote, with no manual `git
  pull` in the primary workspace's plans clone.
---

- **AGENTS:**
  - [bbugyi200.athena.la](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.la/README.md)
  - [bbugyi200.athena.la--code](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.la.md#member-code)
- **COMMITS:**
  - [f429d11](https://github.com/sase-org/sase/commit/f429d118c2433f99522be4e6a7138aa071f5ea6e) — fix: refresh bead stores for active waiters

# Plan: Refresh the canonical bead store so closed-bead waits release on their own

## Problem

Agents parked on `%wait(bead=<id>)` — most visibly epic phase workers — stay parked indefinitely after the bead they
wait on is closed and pushed to GitHub. The only reliable way to release them today is to manually `git pull` the plans
sidecar clone in the project's **primary** workspace:

```
❯ cd sase/repos/plans && git pull
   07b527a9..69ef0e1c  main       -> origin/main
 beads/events/streams/sase-9q.jsonl | 1 +
 beads/issues.jsonl                 | 2 +-
```

Right after that pull, `wait_checks` sees the bead as closed and writes `ready.json`.

## Root cause

Bead waits are resolved against exactly one clone per project, and nothing integrates that clone on a schedule.

1. `wait_checks` (`src/sase/scripts/sase_chop_wait_checks.py:124`) resolves `wait_for_beads` by calling
   `closed_bead_ids_for_project(project_name)`.
2. `closed_bead_ids_for_project` (`src/sase/bead/store_locator.py:30`) reads the _canonical_ store from
   `canonical_beads_dir_for_project` → `get_project_beads_dirs_for_project` (`src/sase/bead/workspace.py:28`) →
   `_canonical_project_beads_dir` (`src/sase/bead/workspace.py:197`). For `sidecar_repos` storage that resolves to the
   **primary workspace's** plans clone — e.g. for this project, `<primary-workspace>/sase/repos/plans/beads`. It is a
   plain filesystem read: no fetch, no rebase.
3. A worker that closes a bead does so in **its own** numbered workspace's plans clone and pushes that commit to the
   sidecar remote. Nothing propagates it back into the canonical clone.
4. The canonical clone is only ever integrated opportunistically, never on a cadence:
   - `schedule_current_bead_refresh` (`src/sase/bead/sync.py:468`) fired from the CLI entry points
     (`src/sase/main/entry.py:124`, `src/sase/main/bead_fast_path.py:69`) — only when a `sase bead` command happens to
     resolve to that store;
   - workspace materialization (`ensure_sdd_kind_clone` → `_pull_sdd_clone`, `src/sase/sdd/_store_link.py:262`) — only
     when a new workspace is set up;
   - the missing-bead retry inside `claim_bead_for_waiting_agent` (`src/sase/bead/claims.py:171`) — only when a claim
     targets a bead the local store has never seen.

   None of these is correlated with "a bead someone is waiting on just closed elsewhere".

5. The runner-side backstop cannot rescue it either: `waiting_marker_dependencies_resolved`
   (`src/sase/axe/run_agent_wait_deps.py:77`), polled every 60s from `src/sase/axe/run_agent_wait.py:183`, re-resolves
   against the _same_ stale clone. It is designed for chop outages, not stale data.

So the fix is what you suspected: give the canonical bead store a periodic remote integration, driven by the existence
of unresolved bead waits.

## Non-goals

- Do **not** change how closed beads are detected, how `ready.json` is written, or the `%wait` directive surface.
- Do **not** change the SDD integration primitives themselves (fetch/rebase/recovery). This plan only _calls_ the
  existing `refresh_bead_store` entry point.
- Do **not** push from the refresh path. The refresh is read-only recovery: fetch + rebase, never push.
- Do **not** touch per-workspace clones. Only the canonical (primary-workspace) store matters for wait resolution.

## Must not conflict with the `sase-9r` epic

`sase-9r` ("Serialize bead-store writes with SDD sidecar integration") is in flight and rewrites the integration
internals. Per its plan file (`sase/repos/plans/202607/sdd_clone_integration_race.md`) it edits:

`src/sase/bead/claims.py`, `src/sase/bead/sync.py`, `src/sase/bead/project.py`, `src/sase/bead/conflict_resolver.py`,
`src/sase/bead/cli_work_from_plan_store.py`, `src/sase/axe/run_agent_exec_plan_accept.py`, and everything under
`src/sase/sdd/` (`_repository_integration.py`, `_repository_health.py`, `_repository_recovery*.py`,
`_repository_transaction.py`, `_git.py`, `_git_contention.py`, `_store_link.py`, `_repository_recovery_markers.py`).

**Hard constraint for this tale: do not edit any file in that list.** Treat `refresh_bead_store(beads_dir)`
(`src/sase/bead/sync.py:502`) as a stable public call — `sase-9r`'s own plan keeps it and cites it as the model for lock
hand-off, so calling it means this work automatically inherits every `sase-9r` hardening (lock serialization, rollback
verification, integration backoff) with zero merge surface.

Two consequences to honor:

- Call `refresh_bead_store` (which routes to the non-destructive `integrate_sdd_repository`). **Never** call
  `integrate_machine_managed_sdd_repository` / `_pull_sdd_clone` on the canonical clone: that clone is the user's
  primary workspace checkout and may hold real local work, and the destructive recovery path is exactly what `sase-9r`
  is fixing.
- Never raise out of the refresh. A failed integration must not become an axe error-digest entry — `sase-9r` exists
  because that digest is already noisy.

## Design

A new builtin chop, `bead_store_refresh`, in the existing `waits` lumberjack. It runs on its own cadence, decides which
projects actually need fresh bead data, and integrates only those canonical stores. `wait_checks` is left completely
unchanged — it keeps reading the local store, which is now kept fresh underneath it.

Why a separate chop rather than folding the fetch into `wait_checks`:

- `wait_checks` runs on a 10s tick. Script chops within a tick run concurrently in a thread pool
  (`src/sase/axe/lumberjack.py:220`) but a single chop's runs never overlap — `run_script_chop_once` returns
  `already_running` (`src/sase/axe/chop_runner_script.py:77`). A slow or hung `git fetch` inside `wait_checks` would
  therefore stall resolution of _all_ waits, including pure agent-name waits that need no network at all.
- A separate chop gets its own `run_every` cadence, its own timeout, and its own failure accounting.

Expected latency: ≤ `run_every` (30s) for the integration, plus ≤ 10s for the next `wait_checks` tick — under ~40s from
push to `ready.json` in the normal case.

### Chop behavior

1. **Kill switch.** If `bead_refresh_mode()` (`src/sase/sdd/_integration_marker.py:16`, re-exported as
   `sase.bead.sync.bead_refresh_mode`) is `"off"`, emit a summary with reason `bead_refresh_disabled` and do nothing.
   This reuses the documented `sdd.bead_refresh.mode` knob instead of inventing a new one, matching the precedent in
   `src/sase/scripts/sase_clan_summary_epic.py:240`.
2. **Find work.** Scan agent artifacts via the Rust scan facade —
   `scan_agent_artifacts(sase_projects_dir(), AgentArtifactScanOptionsWire(only_workflow_dirs=("ace-run",), include_waiting=True, ...))`
   — modeled on `_SCAN_OPTIONS` in `src/sase/scripts/sase_chop_bead_claim_checks.py:27`. Keep a record when all of these
   hold:
   - `record.waiting is not None` and `record.waiting.wait_for_beads` is non-empty;
   - `ready.json` does not exist in `record.artifact_dir` (the wait is already resolved otherwise);
   - the owning process is alive, using the same `is_process_alive(meta, artifact_dir)` check that
     `_claim_owner_is_alive` (`src/sase/scripts/sase_chop_bead_claim_checks.py:98`) applies to the identical pre-launch
     waiting population. **Fail open**: if liveness cannot be determined, keep the record. Dropping a live waiter is the
     one failure mode that reintroduces the bug.
3. **Refresh.** Collect the distinct `project_name` values, and for each resolve
   `canonical_beads_dir_for_project(project)` (`src/sase/bead/store_locator.py:12`). Skip `None`. Call
   `refresh_bead_store(beads_dir)` inside `try/except Exception`, logging failures with `runtime.log.warning` in the
   style of `src/sase/scripts/sase_chop_bead_claim_checks.py:164`.
4. **Failure backoff.** Persist a small JSON map (`{project: {"failures": int, "next_attempt_at": iso}}`) under the
   chop's state dir (`runtime.context.state_dir`) so a store that cannot integrate — remote down, credentials gone,
   diverged clone — is retried with exponential backoff (`run_every * 2**failures`, clamped to 15 minutes) instead of
   once per cadence forever. Clear a project's entry on success. Corrupt/unreadable state must be treated as empty.
5. **Summary.** Always return success from the chop itself. Emit
   `{"projects_waiting": n, "stores_refreshed": n, "stores_failed": n, "stores_backed_off": n}` with a `reason` when
   nothing was refreshed (`bead_refresh_disabled`, `no_bead_waits`, `no_canonical_stores`, `all_backed_off`,
   `refresh_failed`). Follow the `runtime.emit_summary(...)` contract used by every existing builtin chop.

### Runner-side outage backstop

`src/sase/axe/run_agent_wait.py` already re-resolves dependencies every 60s so "a chop outage cannot strand a waiting
agent forever" (`src/sase/axe/run_agent_wait_deps.py` module docstring). For bead waits that promise is currently
hollow, because the local store never changes while axe is down. Close it:

- Add `refresh_bead_wait_store(project_name)` to `src/sase/axe/run_agent_wait_deps.py`: no-op when
  `bead_refresh_mode() == "off"`; otherwise resolve `canonical_beads_dir_for_project(project_name)` and call
  `refresh_bead_store`, swallowing every exception.
- In `src/sase/axe/run_agent_wait.py`, next to the existing `_WAIT_DEPENDENCY_FALLBACK_INTERVAL = 60.0`, add
  `_WAIT_BEAD_REFRESH_FALLBACK_INTERVAL = 600.0`. Inside the dependency poll loop, when `wait_beads` is non-empty and
  that much time has elapsed, call `refresh_bead_wait_store(project_name)` immediately before the existing
  `waiting_marker_dependencies_resolved` fallback.

Ten minutes is deliberately generous: under normal operation the chop always wins and this never fires. A thundering
herd of waiters is already handled inside `refresh_bead_store` — it serializes on `store_git_write_lock` and returns
early when `integration_marker_generation` shows another process integrated while it waited
(`src/sase/bead/sync.py:521`), so N waiters collapse to one real fetch.

Do **not** add a refresh to `initial_dependencies_resolved`; it runs on the launch fast path and must stay network-free.

## Implementation steps

1. **New chop script** — `src/sase/scripts/sase_chop_bead_store_refresh.py`, registered with
   `@builtin_chop("bead_store_refresh")` and a `main()` calling `run_builtin_chop("bead_store_refresh")`, exactly like
   `src/sase/scripts/sase_chop_bead_claim_checks.py`. Implement the behavior above. Keep the backoff-state helpers in
   the same module unless the file grows past the repo's size norms.
2. **Console script** — add `sase_chop_bead_store_refresh = "sase.scripts.sase_chop_bead_store_refresh:main"` to the
   `[project.scripts]` block in `pyproject.toml` (alphabetical, next to `sase_chop_bead_claim_checks` at line 122).
3. **Default config** — in `src/sase/default_config.yml`, add to the `waits` lumberjack (line 448), _before_
   `wait_checks`:

   ```yaml
   - name: bead_store_refresh
     script: sase_chop_bead_store_refresh
     run_every: "30s"
     timeout: "2m"
     description: "Integrate the canonical bead store for projects with agents waiting on beads"
   ```

   `run_every` is the cadence gate (`src/sase/axe/lumberjack.py:210`) and `timeout` is required because the `waits`
   lumberjack sets no `chop_timeout` default — without it a hung fetch would leave a permanently "running" chop and the
   live-run dedupe would suppress every later run.

4. **Runner backstop** — the two edits described above in `src/sase/axe/run_agent_wait_deps.py` and
   `src/sase/axe/run_agent_wait.py`.
5. **Docs**:
   - `docs/axe.md`: add a `bead_store_refresh` row to the waits-chop table (line ~177) and extend the `wait_for_beads`
     paragraph (line ~187) to state that the canonical store is now integrated on a cadence while bead waits are
     outstanding, and that `sdd.bead_refresh.mode: off` disables that integration.
   - `docs/configuration.md`: add the new chop to the `waits` block of the lumberjack example (line ~1212), and note in
     the `sdd.bead_refresh.mode` row (line ~1800) that `off` also disables the `bead_store_refresh` chop and the
     runner's bead-wait backstop.

## Testing

New file `tests/test_axe_chop_bead_store_refresh.py`, following the fixture style of
`tests/_axe_chop_wait_checks_helpers.py` (which builds a `ChopScriptContext` with a `state_dir` and calls the chop's
`main()`) and `tests/test_axe_chop_bead_claim_checks.py`. Monkeypatch the module's `refresh_bead_store` and
`canonical_beads_dir_for_project` seams — no test may touch a real git remote. Cover:

- a live agent with unresolved `wait_for_beads` triggers exactly one refresh for its project;
- two waiting agents in the same project trigger exactly one refresh (dedupe by project);
- a waiting agent whose `ready.json` already exists triggers no refresh;
- an agent with only agent-name/artifact waits triggers no refresh;
- a dead waiting agent triggers no refresh, but an _undeterminable_ liveness result still does (fail open);
- `bead_refresh_mode() == "off"` short-circuits with the `bead_refresh_disabled` reason;
- a raising `refresh_bead_store` is contained: the chop still succeeds, counts the failure, and the next run within the
  backoff window skips that project; a later success clears the backoff;
- corrupt backoff state is treated as empty rather than crashing.

Also extend `tests/test_axe_chop_wait_checks_beads.py` (or add a focused test alongside it) proving the end-to-end seam:
a closed bead that appears in the canonical store only _after_ a refresh is what flips `wait_checks` to write
`ready.json`. Add a unit test for `refresh_bead_wait_store` covering the `off` mode and exception containment.

## Verification

- `just install && just check` from the workspace checkout (mandatory for this repo; the ephemeral workspace may have
  stale deps).
- Confirm the config parses and the chop is registered: `sase axe chop list` (or the equivalent listing command) shows
  `bead_store_refresh` under `waits`.
- Dry-run / manual single run: `sase axe chop run waits bead_store_refresh` and confirm the emitted summary.
- Empirical check of the reader assumption (worth doing once, cheaply): after a refresh brings a newer `issues.jsonl`
  into the canonical store, `closed_bead_ids_for_project` must observe the newly closed bead **without** any SQLite
  rebuild. This should hold — `BeadProject.list_issues` (`src/sase/bead/project.py:151`) delegates to the Rust read
  facade reading `beads_dir` directly, and `beads.db` is only the legacy compatibility mirror
  (`src/sase/bead/project.py:72`) — but confirm it rather than assume it, because the whole fix depends on it.
- Live confirmation: with an agent parked on `%wait(bead=…)`, close and push that bead from another workspace and verify
  the agent starts without any manual `git pull`.

## Risks and notes

- **Added integration traffic on the canonical clone.** Bounded by design: at most one fetch/rebase per project per 30s,
  and only while an unresolved bead wait exists. Zero extra traffic when nothing is waiting. This does raise contention
  on the store write lock that `sase-9r` is hardening — which is the argument for routing exclusively through
  `refresh_bead_store` rather than adding a parallel git path.
- **Refresh failures must stay quiet.** Warnings and summary counters only; no exceptions, no error-digest entries.
- **Rust core boundary.** This is axe/lumberjack chop logic, which lives in Python in this repo (every existing chop
  does), reading through the existing Rust scan and bead-read facades. No `sase-core` change is required.
