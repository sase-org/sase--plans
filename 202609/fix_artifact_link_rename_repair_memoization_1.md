---
tier: tale
title: Fix the artifact_link_backfill chop's per-ref rename-scan loop
goal:
  The housekeeping artifact_link_backfill chop completes scheduled ticks well under its
  300s timeout because the rename-repair job runs its full-history git scan once per
  kind instead of once per dangling ref, and the chop deadline can defer the repair loop
  instead of being SIGKILLed by it.
size: medium
proposed_by: bbugyi200.kellys_mbp.m
---

# Plan: Fix the `artifact_link_backfill` chop's per-ref rename-scan loop (closes sase-u9)

## 1. The symptom

The `housekeeping` lumberjack's `artifact_link_backfill` chop now times out on **every**
scheduled tick:

```
Lumberjack: housekeeping
Job:        artifact_link_backfill
Error:      timed out after 300s
Traceback:  <no python traceback: subprocess error>
```

Five consecutive scheduled runs on 2026-09-03 died this way (12:44, 14:26, 15:31, 16:36,
16:44 EDT; see `~/.sase/axe/error_digests/digest_20260903_173941.txt` and
`~/.sase/axe/logs/lumberjack-housekeeping.log`). This is the recurrence that plan
`202608/artifact_link_backfill_chop_timeout.md` (commit `cc2d907`) predicted when it
filed **sase-u9**: that plan fixed the reconcile pass (job 3) and added the
`_CHOP_WORK_BUDGET_SECONDS = 240.0` guardrail, but jobs 3+4 are one un-budgeted call
(`reconcile_and_repair_artifact_links`), so the chop-level deadline cannot interrupt job
4 once it starts.

## 2. Root cause (confirmed on a live hung run)

Sampling a live timed-out run's process tree showed the chop spawning a **fresh**

```
git log --all --format= --name-status --find-renames -M --diff-filter=R --
```

child every ~1–2 seconds for 3.5+ minutes, cwd = the sase project's plans sidecar (6,222
commits across all refs; one scan measured at ~2.2s wall). sase-u9's profiling had
already counted 208 such subprocesses in one run — one per candidate dangling ref.
`208 × ~2.2s ≈ 460s`, alone far beyond the 300s axe timeout.

The per-ref scanning is not by design — it is a classic eager-`setdefault` bug in
`repair_historical_artifact_renames` (`src/sase/sdd/_artifact_link_renames.py:89`):

```python
history = by_kind.setdefault(
    kind,
    _historical_rename_map(root, kind=kind),
)
```

`dict.setdefault` evaluates its default argument **eagerly**, so
`_historical_rename_map` — the full-history git rename scan — runs on every loop
iteration even when `kind` is already cached. The intended once-per-kind memoization
(`by_kind` exists for exactly this) is dead code. There are only two kinds (`plan`,
`research`), so a correct memo bounds job 4 at **two** scans (~5s) per project instead
of one scan per dangling ref.

**Why the failure surfaced now:** while the retroactive sweep (job 1) still had
remaining documents, `_run_project` returned early ("deferred outbox/reconcile after
sweep budget") and job 4 never ran for the sase project. The sweep converged on
2026-09-03 (13 documents remaining at 13:22, then 0), so from then on every tick reaches
job 4 for the sase project and dies inside it. The per-project sweep checkpoint persists
mid-run, so this is a stable, self-sustaining failure state: every future tick times out
until job 4 is fixed.

Reproduce the scan cost directly (read-only):

```bash
time git -C "$(sase repo path plans)" -c rerere.enabled=false \
  log --all --format= --name-status --find-renames -M --diff-filter=R -- >/dev/null
```

## 3. Changes

### 3.1 `src/sase/sdd/_artifact_link_renames.py` — the fix

In `repair_historical_artifact_renames`, replace the eager `setdefault` with an explicit
guard so the scan runs at most once per kind:

```python
if kind not in by_kind:
    by_kind[kind] = _historical_rename_map(root, kind=kind)
history = by_kind[kind]
```

This also fixes the identical cost on the interactive path: `sase artifact doctor --fix`
calls the same function via `src/sase/artifact_cli/link_health.py`
(`inspect_artifact_link_health`).

### 3.2 Deadline threading — the sase-u9 hardening

sase-u9's ask is a time budget on this loop, and it stays valid even after 3.1:
`reconcile_and_repair_artifact_links` remains a single un-budgeted call, so a future
slow path inside it could again sail past the chop budget into the SIGKILL. Thread the
existing chop deadline through:

1. `repair_historical_artifact_renames(store, refs)` gains a keyword-only
   `deadline: float | None = None` (a `time.monotonic()` timestamp, matching the chop's
   existing budget style). Check it at the top of each ref iteration; on expiry, stop
   resolving further refs but still apply whatever renames were already resolved. Add a
   `deferred_refs: int = 0` field to `_ArtifactLinkRenameReport` counting the refs not
   examined.
2. `reconcile_and_repair_artifact_links(store)` gains the same keyword-only `deadline`
   parameter, passed through to the repair call, and its `_ArtifactLinkReconcileReport`
   gains `deferred_refs: int = 0` copied from the repair report.
3. `src/sase/scripts/sase_chop_artifact_link_backfill.py::_run_project` passes
   `deadline=chop_deadline` and, when the returned report has `deferred_refs > 0`,
   appends a warning naming the project and the deferred count (mirroring the existing
   deferral warnings) so run history shows the degradation.

The `link_health.py` caller keeps the default `deadline=None` — the interactive doctor
should finish its work, not defer it. Existing callers and tests must keep working
without signature changes at their call sites.

## 4. Tests

New file `tests/sdd/test_artifact_link_rename_repair.py` (none exists for this module
today; `tests/sdd/test_artifact_link_reconcile.py` only mocks it). Follow that file's
fixture idiom: `redirect_sase_home` plus a real `ArtifactLinkStore` with tmp-path
`plan`/`research` sidecar roots.

1. **Regression: one scan per kind, not per ref.** Monkeypatch
   `sase.sdd._artifact_link_renames._historical_rename_map` with a counting fake
   returning `{}`. Call `repair_historical_artifact_renames` with three `plan:` refs and
   two `research:` refs. Assert the fake ran exactly twice (once per kind). This test
   MUST fail against today's code (it currently runs five times).
2. **Deadline defers un-examined refs.** With the same counting fake, pass
   `deadline=time.monotonic() - 1.0` and several refs; assert zero scans ran and the
   report's `deferred_refs` equals the number of eligible refs. Also assert
   `deadline=None` (default) examines everything, and that renames already resolved
   before an expiring deadline are still applied.
3. **Chop passes its budget through.** In
   `tests/test_axe_chop_artifact_link_backfill.py`, following its existing monkeypatch
   pattern, capture the kwargs `reconcile_and_repair_artifact_links` receives from
   `_run_project` and assert `deadline` is the chop deadline; assert a
   `deferred_refs > 0` report produces a warning naming the project.

## 5. Verification

1. `just check` (run `just install` first if the venv is stale — ephemeral workspace
   clones may sit unused while pinned dependencies change). Hand it to `/sase_monitor`
   if it runs long.
2. **Real-store proof** that the repair now completes. From the project checkout, run
   the sase-u9 reproduction:

   ```python
   import time
   from pathlib import Path
   from sase.sdd.artifact_link_store import resolve_artifact_link_store
   from sase.sdd.artifact_link_backfill import reconcile_and_repair_artifact_links

   store = resolve_artifact_link_store(
       cwd=Path.home() / "projects" / "github" / "sase-org" / "sase"
   )
   t0 = time.monotonic()
   report = reconcile_and_repair_artifact_links(store)
   print(f"{time.monotonic() - t0:.1f}s", report)
   ```

   Expect well under 60s total (previously unbounded: 100s–460s+). Note this is the real
   repair — it may legitimately commit rewritten `links/` indexes to the plans/research
   sidecars with an async push, exactly as the healthy chop would; that one-time healing
   of the dangling backlog is desired.

3. **Real chop run** end to end:

   ```bash
   sase axe chop run artifact_link_backfill -L housekeeping -f -V
   ```

   Then read the newest run record under
   `~/.sase/axe/lumberjacks/housekeeping/chops/artifact_link_backfill/runs/` and confirm
   `status == "success"` with `duration_ms` far below 300000, and that the per-project
   progress log line reports a small `reconcile_repair` elapsed time.

4. This change touches shared `sase.sdd` store code on `sase artifact doctor`'s path, so
   run `just check-full` through `/sase_monitor` (`TESTING`/`TESTED` status pair — never
   inline) before landing.

## 6. Bookkeeping

- After verification, close the corroborated task bead with evidence:
  `sase bead close sase-u9 --note "<measured before/after + chop run outcome>"`.
- Do NOT widen this plan into the adjacent known issues: the
  `init_sys_streams`/`Bad file descriptor` chop-launch failures are tracked separately
  as **sase-wd**, and `run_sdd_git`'s bytes-stderr defect is **sase-va**. Leave both
  alone here.
