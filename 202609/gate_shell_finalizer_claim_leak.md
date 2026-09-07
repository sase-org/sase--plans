---
tier: tale
title: Stop gate shells from releasing a live finalizer run's workspace claim
goal:
  A gate created during a host finalizer turn is refused, and gate settlement never
  frees a workspace whose creator process is still alive.
size: medium
proposed_by: bbugyi200.athena.03z
create_time: 2026-09-07 14:26:28
status: wip
---

# Stop gate shells from releasing a live finalizer run's workspace claim

## Problem

A gate created from inside a host finalizer's provider turn steals the creating run's
workspace claim and then releases the workspace to the free pool while the run is still
alive and using the checkout. This destroyed the `sase-x7.4` run on 2026-09-07
(`ace(run)-260907_123316`, error
`commit finalizer hit a second unresolved conflict in main`).

Evidence chain (all timestamps EDT, from `~/.sase/logs/workspace_claims.jsonl` and
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202609/07/20260907123316/`):

1. `12:33` — the run's commit finalizer hit a legitimate rebase conflict in the main
   repo (`tests/test_agent_name_registry_lock.py`) and started its one-shot
   conflict-repair provider turn (`finalizers/commit/conflict_repair_prompt.md`).
2. `13:01:37` — the repair-turn model requested an agent launch. The LaunchApproval gate
   shell transferred workspace #29's claim to `ace-gate` (ledger
   `caller_tag=gate-shell-create`).
3. `13:01:44` — the gate settled (rejected) and the gate shell **released workspace #29
   to the pool** (ledger `caller_tag=gate-shell-settle`) even though the creating run
   (claim pid 2375248) was still alive, mid-repair.
4. `13:03` and `13:31` — two unrelated runs (`ace(run)-260907_130306`, then
   `ace(run)-260907_133109` / agent `sase-xy.5.1`) were allocated workspace #29. The
   first one's workspace preparation reset the checkout, destroying the paused rebase;
   the repair turn's transcript (`finalizers/commit/conflict_repair_response.md`)
   records the wipe, the recovery, and the eventual deadlock against the concurrent
   writer.
5. `14:00` — the finalizer's `sase stitch create --resume` still saw the conflict, the
   run failed with `second_unresolved_conflict`, and its `hold_workspace_claim` failed
   with `workspace #29 claim for ace(run)-260907_123316/... was not found`.

Root cause: the gate-shell claim protocol implements the `gates-never-block` decision —
"creating a shell gate hands off and kills the creating agent's turn immediately" — so
on settle the workspace is legitimately handed to a follow-up or released. But a
**finalizer-owned provider turn** (the commit finalizer's one-shot conflict-repair turn,
and the declaration-recovery turn) cannot end the agent run: the host process keeps
executing mechanically after the provider turn returns. Gate creation from such a turn
violates the protocol's precondition, and nothing currently detects or prevents it.

## Fix 1 (root): refuse gate-shell creation from finalizer-owned provider turns

- Add a small helper (suggested home: `src/sase/finalizers/` — e.g. a
  `finalizer_owned_turn()` context manager next to the existing
  `mint_finalizer_turn_nonce` machinery in `declaration.py`, or a sibling module) that
  sets a marker env var, suggested name `SASE_FINALIZER_OWNED_TURN=1`, and restores the
  prior value on exit. The env var propagates to the provider CLI subprocess and
  therefore to any `sase gate create` / launch-approval invocation the model makes.
- Wrap **every** provider invocation the finalizers own with it. Known sites (grep
  `provider.invoke` and `invoke` under `src/sase/finalizers/` to catch all):
  - `_run_conflict_repair_turn` in `src/sase/finalizers/commit_repair.py`
  - the declaration-recovery provider turn driven via
    `src/sase/finalizers/declaration_recovery.py`
- In `create_gate_shell` (`src/sase/gate_shell/transaction.py`), before any side effect
  (before `create_gate_shell_member` and before `move_gate_shell_claim`), check the
  marker and raise `GateShellError` with an instructive message along the lines of:
  "gate shells cannot be created from a host finalizer turn: this turn cannot end the
  agent run, so the gate could never hand off. Finish the finalizer's task in this turn,
  or report the blocker in your response so the host records the failure." The message
  is what the model sees, so it must steer the model back to the repair contract.
- Verify by grep that every gate kind (CustomGate, LaunchApproval, PlanApproval,
  EpicApproval, UserQuestion) funnels through `create_gate_shell`; if any creation path
  bypasses it, guard that path with the same check.

## Fix 2 (defense in depth): settle must not release a claim whose pid is alive

Even with Fix 1, any future path that settles a gate shell while the creator process
still lives would free a workspace in active use. Make settle liveness-safe:

- Gate-shell creation already persists the creator's original claim via
  `_record_creator_claim` (`src/sase/gate_shell/transaction.py`) and the restore
  machinery already exists (`restore_gate_shell_claim` in
  `src/sase/gate_shell/start_claim.py`).
- In `release_gate_shell_claim` (`src/sase/gate_shell/start_claim.py`, called from
  `src/sase/gate_shell/settlement.py`): before releasing the `ace-gate` claim, load the
  recorded creator claim for this shell and probe whether its pid is still a running
  process (reuse the existing process-liveness probe pattern from
  `src/sase/workspace_provider/inventory.py`). If alive, restore the creator's original
  claim (transfer back to the original workflow / artifacts timestamp) instead of
  releasing, and record the anomaly in the gate-shell log plus the workspace-claim
  ledger (distinct `caller_tag`, e.g. `gate-shell-settle-restore`). If dead, release
  exactly as today.
- Acceptable edge case: a creator runner that is in its final seconds of shutdown after
  a real handoff gets its claim restored and then dies; that leaves an unpinned dead-pid
  claim. Confirm the existing stale-claim cleanup reaps unpinned dead claims (the
  `pinned` docstring in `src/sase/running_field/_claim.py` implies it) so this cannot
  leak a workspace permanently; if no such reaping exists, note it as a follow-up rather
  than building it here.

## Non-goals

- No changes to Rust core (`sase-core`): claim planning wire calls are unchanged; both
  fixes only choose between existing host-side operations (refuse / release /
  transfer-restore), which is host orchestration, not shared domain behavior.
- No change to the one-shot conflict-repair budget or the `second_unresolved_conflict`
  semantics — those behaved as designed once the workspace was stolen.
- No operational recovery of the stranded `sase-x7.4` commits; that is being handled
  separately.

## Tests

- Gate-shell transaction test: with `SASE_FINALIZER_OWNED_TURN` set, `create_gate_shell`
  raises `GateShellError` before creating a member or moving a claim; without it,
  creation behaves as before.
- Finalizer test: the conflict-repair turn (and declaration-recovery turn) set the
  marker for the duration of the provider invocation and restore the prior env
  afterward, including on exception.
- Settle tests: with a live creator pid recorded, settle restores the original claim
  (ledger shows the restore tag, workspace stays claimed by the original workflow); with
  a dead pid, settle releases as today.

## Verification

Run `just check` (agent default lane). Follow the repo's lint/test memory note before
finishing.
