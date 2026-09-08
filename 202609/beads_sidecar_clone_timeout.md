---
tier: tale
title: Fix beads sidecar clone timeouts that hard-fail agent launches
goal:
  "Agent launches no longer hard-fail on beads sidecar clone timeouts: a timed-out
  reference clone retries once without the local reference, and the sidecar auto-sync
  chop keeps primary sidecar clones defragmented so reference-based clones stay fast."
size: medium
proposed_by: bbugyi200.athena.sase-xe.16.8.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.sase-xe.16.8.f0](https://github.com/sase-org/sase--agents/blob/main/families/bbugyi200.athena.sase-xe.16.8.f0.md)
  - [bbugyi200.athena.sase-yh.1](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-yh.1/README.md)
  - [bbugyi200.athena.sase-yh.3](https://github.com/sase-org/sase--agents/blob/main/agents/bbugyi200.athena.sase-yh.3/README.md)
- **COMMITS:**
  - [46f7f54](https://github.com/sase-org/sase/commit/46f7f549ef2edc9d4e7f9136d810786cb9792b48)
    — fix(sdd): retry unpublished artifact-link sidecars
  - [3ec9b78](https://github.com/sase-org/sase/commit/3ec9b78b2e128554f281409e80043f77888418db)
    — fix(workspace): reconcile managed clone origins before stitch
  - [a5e642c](https://github.com/sase-org/sase/commit/a5e642c929a07fde7fd76b3be34439ce42c897bf)
    — fix(sdd): harden sidecar clone materialization

# Fix Beads Sidecar Clone Timeouts That Hard-Fail Agent Launches

## Problem

An agent launch failed at bead-claim time with:

```
RuntimeError: Failed to claim bead '...' for agent '...': could not materialize
beads sidecar repository sase-org/sase--beads ... timed out cloning SDD store
git@github.com:sase-org/sase--beads.git into <workspace>/sase/repos/beads.
```

The launch path (`claim_bead_for_agent_launch` -> `_ready_launch_beads_store` ->
`ensure_sdd_kind_clone(..., "beads", strict=True)`) materializes a workspace-local beads
sidecar clone with `strict=True`, so a clone failure aborts the launch.

## Root Cause (confirmed from `~/.sase/logs/tui_git_ops.jsonl` telemetry)

Fresh beads sidecar materialization runs:

```
git clone --reference-if-able <primary checkout>/sase/repos/beads --dissociate \
    git@github.com:sase-org/sase--beads.git <workspace>/sase/repos/beads
```

Three compounding causes:

1. **The reference clone is severely fragmented.** The primary checkout's
   `sase/repos/beads/.git` currently holds ~20 packs plus ~3,350 loose objects totaling
   ~468 MiB, while a fresh clone of the same remote packs to ~71 MiB. Nothing in SASE
   ever runs `git gc`/repack on primary sidecar clones, and the beads clone churns
   constantly (fetch + commit cycles), so fragmentation grows without bound. Borrowing
   objects from this fragmented store and then `--dissociate`-repacking them is what
   makes the clone slow.
2. **The clone shares the flat 120s network git timeout** (`network_git_timeout()`,
   default `DEFAULT_NETWORK_GIT_TIMEOUT_SECONDS = 120.0` in `src/sase/sdd/_git.py`).
   Telemetry from a single day shows twelve reference-based beads clones over 60s
   (74-116s) and two that hit the 120s cap and timed out, while every _plain_
   (no-reference) clone of the same remote completed in 6-33s. Under concurrent agent
   load the reference+dissociate variant reliably flirts with the cap.
3. **A timeout gets zero retries.** In `clone_sdd_store`
   (`src/sase/sdd/_store_clone_ops.py`), `SddGitCommandTimeout` is caught _inside_ the
   retry loop but immediately returns via `handle_failed_sdd_clone` — only transient
   stderr failures (`_TRANSIENT_REMOTE_CLONE_ERRORS`) are retried. With `strict=True`
   this raises `SddMaterializationError` and the agent launch dies, even though
   telemetry shows an immediately-following plain clone succeeded in 17s.

Aggravation: workspace beads materializations serialize under
`materialization_lock(primary)` (`ensure_beads_sidecar_clone` in
`src/sase/sdd/_store_workspace.py`), so one 120s clone also delays every other launching
agent for that project.

## Constraints

- Keep the "refs come from the recorded remote" property of the current design (see the
  comment in `clone_sdd_store`): do **not** switch beads sidecar materialization to
  cloning from the primary clone's refs; unpublished local bead commits must not leak
  into fresh workspace clones. Keep `--reference-if-able`/`--dissociate` as the
  first-attempt strategy — it saves ~71 MiB of transfer per workspace when healthy.
- The Rust core boundary is not crossed: this is existing Python SDD infrastructure in
  this repo; fix it in place.
- Never run repository maintenance against a user's primary sidecar clone from tests;
  tests must use temporary repos and/or monkeypatched probes.

## Fix 1: Retry clone timeouts, dropping the reference on retry

File: `src/sase/sdd/_store_clone_ops.py` (`clone_sdd_store`).

- Treat `SddGitCommandTimeout` as retryable instead of immediately fatal:
  - On the **first** timeout, if any retry budget remains: clean the partial clone with
    `_remove_partial_sdd_clone`, log a warning that the clone timed out and is being
    retried **without** the local object reference, strip the
    `--reference-if-able <path> --dissociate` arguments from the clone argv, and retry.
    Rationale (from telemetry): when the reference-based clone stalls, the plain clone
    is fast; the reference is the slow component, not the network.
  - On a **second** timeout, fail through the existing
    `handle_failed_sdd_clone(..., cause=exc)` path unchanged (same error message), so a
    genuinely dead network still fails within ~2 x timeout instead of retrying
    indefinitely. Cap timeout retries at one; do not let timeouts consume the full
    `_REMOTE_CLONE_RETRY_DELAYS` schedule (each attempt costs up to 120s while holding
    the beads materialization lock).
- Transient stderr-failure retry behavior stays exactly as it is today.
- Keep the `op="sdd.clone.remote"` telemetry name so existing log analysis keeps
  working; the retried attempt is distinguishable by its argv (no
  `--reference-if-able`).

## Fix 2: Fragmentation-gated maintenance for primary sidecar clones

The reference clone must stop degrading. Add a small, best-effort maintenance step to
the sidecar auto-sync path so the primary sidecar clones (beads is the acute one;
plans/research benefit identically) get repacked when fragmented.

- Add a helper (suggested: `src/sase/sdd/_store_maintenance.py`) exposing roughly
  `maybe_gc_sidecar_clone(clone_dir: Path, primary: Path) -> bool`:
  - **Fragmentation probe** (cheap, no subprocess): count `*.pack` files under
    `clone_dir/.git/objects/pack/` and loose-object files under
    `clone_dir/.git/objects/<xx>/`. Trigger maintenance when packs > ~8 or loose
    objects > ~2000 (module constants; no new CLI surface).
  - When triggered, run `git gc` via `run_sdd_git` with a dedicated telemetry op (e.g.
    `op="sdd.maintenance.gc"`) and a generous dedicated timeout constant (~600s) — the
    existing local/network timeouts are too short for a first gc of a ~468 MiB store.
    Best-effort: log and return `False` on any failure or timeout; never raise into the
    caller.
  - Hold `materialization_lock(primary)` (from `sase.sdd._store_adoption`) for the
    duration of the gc so it cannot race a concurrent workspace beads materialization
    that is borrowing objects from this clone via `--reference-if-able`. If the lock API
    cannot be acquired without blocking launches for the whole gc, prefer skipping the
    gc when the lock is contended over queueing behind it — maintenance is always safe
    to defer.
- Call the helper from the sidecar auto-sync chop
  (`src/sase/scripts/sase_chop_sidecar_auto_sync.py`, `_run`), after a target syncs
  successfully (`refreshed` or `up_to_date`), and only when enough of the chop's work
  budget remains (compare against `work_deadline` before starting; skip and count it as
  deferred-style work otherwise). This keeps budget logic in the chop, where the
  deadline lives, and keeps `sync_primary_sidecar_role` side-effect-free beyond its
  documented fetch/fast-forward.
- The first chop run after this lands will gc the currently-bloated primary beads clone
  automatically; no manual operational step is required.

## Tests

Extend `tests/sdd_store/test_sidecar_clone.py` (model:
`test_sidecar_clone_retries_transient_transport_failures`) and add coverage for the
maintenance helper (new test module or alongside existing sdd_store tests):

1. Clone timeout on the first (reference-based) attempt -> retried exactly once without
   the reference arguments -> success; the partial clone directory is removed between
   attempts.
2. Two consecutive timeouts -> strict mode raises `SddMaterializationError` whose
   message still contains "timed out cloning SDD store"; non-strict mode returns
   `False`. No third git invocation occurs.
3. Transient stderr failures still retry per the existing schedule (guard against
   regression of current behavior; existing test should keep passing).
4. Maintenance probe: repo over the pack-count or loose-object threshold triggers gc;
   repo under both thresholds does not.
5. Maintenance failure/timeout is swallowed (returns `False`, no raise), and the chop
   proceeds.
6. Chop integration: gc is skipped when the remaining work budget is too small.

## Verification

- Run `just install` first if the workspace venv is stale, then `just check` (whole-repo
  lint gates + diff-scoped tests) before finishing. If the scoped run escalates or the
  change grows beyond these two modules + tests, run `just check-full` via a monitor per
  the two-speed verification rule.

## Out of Scope

- Changing where beads sidecar refs come from (remote-refs property stays).
- Server-side history growth of the beads repo itself (~78k objects upstream); a healthy
  71 MiB clone completes in well under the timeout, so no shallow/partial clone work is
  needed now.
- New CLI subcommands, options, or feature flags — thresholds and timeouts are module
  constants.
