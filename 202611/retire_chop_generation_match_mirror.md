---
tier: tale
title: Finish sase-1tc.3 by retiring the Python clan-generation mirror
goal:
  Clan-kill generation gating trusts the pinned sase-core policy with no Python fallback
  or veto, and the bead store records the finished closeout evidence.
size: small
decisions:
  close_sase_1tm:
    ask: Once this cleanup lands and its regressions pass, close task sase-1tm as done?
    choices:
      close:
        Close sase-1tm with commit and test evidence; its last open ask is this cleanup
      leave_open: Only append an evidence note to sase-1tm and leave it for later triage
    default: close
    why: Its nodes pass with causal fixes on master; note
    answer: close
proposed_by: bbugyi200.athena.1ea
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 06:14:04
status: wip
---

# Finish sase-1tc.3: retire the Python clan-generation mirror and record closeout evidence

## Context

Phase bead **sase-1tc.3** ("A scheduler stop never silently drops a chop's launch batch
(sase-1c2)") is already **closed** (resolution `done`, "delegated work landed",
2026-11-19T10:22:57Z). Its child epic **sase-1tc.3.1** and every descendant
(sase-1tc.3.1.1–.5 and nested epic sase-1tc.3.1.6 with its four phases) are closed.
Closeout note #6 on sase-1tc.3.1 confirms these are done:

- every epic-symbol whitelist row (`789e5d0fc5`, `150fce3735`);
- the mypy fix and the oversized-test split (`5467217dd0`, `6200573ecc`);
- a green full `sase tool run check` `f901dd4e9b90438ba51eab179fd85144`.

One piece of the phase's own work is still unfinished. The phase plan says to "keep any
shared domain logic … behind the Rust boundary", and the `rust-core-required` decision
forbids a "Python reimplementation of a ported operation". The clan-generation kill gate
breaks both rules:

- `src/sase/axe/_chop_batch_recovery_cancel.py::_generation_matches` calls the Rust
  binding `chop_batch_generation_matches`. If that call raises anything, a bare
  `except Exception` falls back to a complete Python copy of the policy.
- When Rust returns `True`, the wrapper also layers a Python "older pins" timestamp veto
  on top of it.
- The private helpers behind that copy are `_scoped_clan_generations`,
  `_unrecorded_older_generation`, `_older_generation_predates_batch`,
  `_batch_started_at`, and `_generation_started_at`. Together they are about 130 lines.

That workaround was added during recovery (sase-1tm note #2). That note asks for core
`chop_batch_generation_matches` to be fixed "so the wrapper can go back to trusting a
Rust True". The core fix has landed: sase-core `6f8f016e` ("refuse older kills of
unrecorded clan generations") is the current pin in `sase-core-revision.txt`, moved in
`5467217dd0`. At that pin,
`crates/sase_core/src/axe_chop/batch.rs::chop_batch_generation_matches` implements the
whole contract:

- a blank generation matches;
- only member generations recorded for the requested clan can satisfy or veto a match;
  members of other clans are ignored;
- the top-level `clan_generation` counts only when the batch has no member clans and its
  top-level clan is blank or equals the requested clan;
- with no scoped generation, an older kill that predates the batch's `created_at`
  (falling back to `run.run_started_at`) does not match.

The Python copy is now dead weight, and it is a second implementation that can drift.
The core's `settle_cancelled_dispatch` transition has also existed since core
`b1a03639`, so the "Older core pins lack the settlement transition" comment in
`_chop_batch_recovery_cancel_settlement.py` is stale.

No sase-core change is needed, and the pin does not move.

## Changes

### 1. `src/sase/axe/_chop_batch_recovery_cancel.py`: trust the core decision

- Import `CHOP_BATCH_SCHEMA_VERSION` and `chop_batch_generation_matches` from
  `sase.core.axe_chop_facade` at module top. Sibling store modules already import that
  facade at top level. The lazy import existed only to tolerate older pins.
- Rewrite `_generation_matches(manifest, clan, generation)` so it returns
  `chop_batch_generation_matches({...})` directly, using the same request it builds
  today: `schema_version`, `manifest` as a `dict`, `clan`, `generation`. Remove the
  `rust_matched` tri-state, the veto on `True`, and the Python fallback branch.
- Error handling follows the store's pattern in `_chop_batch_store_batches.py`. Catch
  only `ValueError`, which is how core `ChopEngineError` surfaces, and re-raise it as
  `ChopBatchError(f"cannot match clan generation {generation!r} of {clan!r} against batch {batch_id!r}: {exc}")`,
  where `batch_id` comes from `manifest.get("batch_id")`. Let every other exception
  propagate. Never fall back and never guess a match.
- Why this fails closed: the only caller, `cancel_pending_batches_for_clan_selection` in
  `src/sase/ace/tui/actions/agents/_clan_cleanup.py`, already turns any exception into a
  visible `RuntimeError` that stops clan cleanup before member expansion. Retrying the
  kill is idempotent, because batches already cancelled no longer pass
  `batch_needs_resume`. This path is effectively unreachable: `load_chop_batch` has
  already validated the manifest through the same Rust validator, and the blank-clan
  case returns early.
- Update the docstring. The shared sase-core policy decides: only generations recorded
  for the requested clan can satisfy or veto, and an older kill never cancels a batch
  whose selected clan has no recorded generation. Delete the sentence saying Python
  mirrors the policy for older pins.
- Delete `_scoped_clan_generations`, `_unrecorded_older_generation`,
  `_older_generation_predates_batch`, `_batch_started_at`, and `_generation_started_at`.
  Also delete the now-unused `from datetime import UTC`, plus any other import that
  becomes unused. Check first with `grep -rn "<name>" src tests` that nothing else
  references them; today nothing does.

### 2. `src/sase/axe/_chop_batch_recovery_cancel_settlement.py`: fix the stale comment

Keep the `except Exception as exc:` hold around the `settle_cancelled_dispatch` event.
It correctly holds the member visibly when the core refuses, with `batch_not_cancelled`
or a non-dispatching member state. Only rewrite the comment ("Older core pins lack the
settlement transition; …") so it says the core can refuse this settlement, and that we
then hold visibly instead of retiring unsettled evidence. Do not change behavior.

### 3. Tests: `tests/axe/test_axe_chop_batch_recovery_cancel.py`

- In `test_rust_generation_match_binding_scopes_to_clan` and
  `test_rust_generation_match_binding_rejects_older_unrecorded_generation`, delete the
  `if not hasattr(sase_core_rs, "chop_batch_generation_matches"): pytest.skip("core binding not yet built; fallback covers old pins")`
  guard. The pinned core always has the binding, and the fallback it names is gone. Keep
  `pytest.importorskip("sase_core_rs")`, which is the repo-wide convention.
- Add `test_generation_gate_trusts_the_core_decision`, parametrized over `True` and
  `False`. Monkeypatch `chop_batch_generation_matches` in
  `sase.axe._chop_batch_recovery_cancel` to return the parameter. Use an
  older-generation batch with no recorded generation; this is the case the removed veto
  used to override. Assert that `cancel_chop_batches_for_clan(..., generation=...)`
  cancels the batch exactly when the core says `True`. This guards against a Python veto
  being layered back on.
- Add `test_generation_gate_core_refusal_fails_closed`. Monkeypatch the same name to
  raise `ValueError("schema_version_mismatch")`. Assert that
  `cancel_chop_batches_for_clan` raises `ChopBatchError` naming the batch id, and that
  the batch manifest's status is still `open`: nothing was cancelled and nothing was
  guessed.
- Build fixtures with the existing `manifest` / `record_spawn` helpers from
  `tests/axe/_test_axe_chop_batch_recovery_shared.py` and the `isolated_home` fixture,
  the way the neighboring cancel tests do. Add the new names to the module's `__all__`,
  because `tests/axe/test_axe_chop_batch_recovery.py` re-exports it. The file stays far
  under the toobig limit.
- Do not weaken or skip any existing assertion. The end-to-end generation tests in this
  file must pass unchanged against the core alone:
  `test_cancel_targets_only_matching_clan_generation`,
  `test_cancel_matches_a_secondary_clan_in_the_batch`,
  `test_older_generation_does_not_cancel_a_batch_without_recorded_generation`, and
  `test_cancel_scoped_to_requested_clan_generation`.

## Verification

1. If the workspace venv is stale, run `just install-venv` so `sase_core_rs` matches the
   `6f8f016e` pin.
2. Focused regressions:
   `.venv/bin/pytest tests/axe/test_axe_chop_batch_recovery_cancel.py tests/axe/test_axe_chop_batch_recovery.py tests/axe/test_axe_chop_batch_store.py tests/axe/test_axe_chop_batch_replay.py tests/axe/test_axe_chop_batch_process_recovery.py tests/ace/tui/test_agent_collapsed_panel_kill.py -q`.
   The process-recovery file holds the subprocess clan-kill acceptance scenarios
   (paused, recovery-active, typed, multi-clan, stale generation).
3. `just fmt`, then `sase tool run check` (read the `lint_and_test` reference memory
   first). Do not run `just check-full`. ruff and symvision must stay clean after the
   helper and import deletions.

## Bead closeout

Read each bead with `sase bead read <id> -r "<why>"` before writing to it.

- **sase-1tc.3** (closed phase): do **not** reopen it. No phase work is left to track,
  and a reopened phase could make the waiting `sase-1tc.land` runner reassign it.
  Re-closing is a no-op. Append one note with `sase bead note sase-1tc.3 "..."` that
  records:
  - the follow-up commit;
  - that the clan-generation kill gate now trusts sase-core
    `chop_batch_generation_matches` (pin `6f8f016e`) with no Python fallback;
  - the focused test counts;
  - the `sase tool run check` run id.

  This gives the outer lander current evidence when it verifies note #4's
  `TASK RESOLVED: sase-1c2`.

- **sase-1c2**: do **not** close or modify it. Under the outer epic plan
  (`plan:202611/backlog_epic_cmdsub_retries_launches.md`, "Bead hygiene" and "Landing"),
  only the `sase-1tc` lander closes task beads after verifying `TASK RESOLVED:` notes.
  `sase-1tc.land` is alive and waiting on phases sase-1tc.4 and .5.
- **sase-1tm**:

  > [!decision] close_sase_1tm = close Run `sase bead close sase-1tm -n "<evidence>"`.
  > The evidence must cite:
  >
  > - its original nodes
  >   (`test_clan_cancellation_during_launch_settles_current_and_stops_next`,
  >   `test_cancellation_priority_and_reconcile_selection`), which have passed since the
  >   paired primary/core contract in `4e5c1b9730` (its note #1);
  > - the +1 node
  >   `test_older_generation_does_not_cancel_a_batch_without_recorded_generation`, fixed
  >   by core `6f8f016e` and pin move `5467217dd0`;
  > - this commit, which retires the sase-side veto that note #2 asked to remove;
  > - a passing serial run of all three nodes on the final tree.

  > [!decision] close_sase_1tm = leave_open Append the same evidence with
  > `sase bead note sase-1tm "..."` and leave its status unchanged.

- Do not touch epic `sase-1tc` or its other phases. Create no new beads unless
  verification finds a real, unrelated failure. In that case, use `/sase_new_task`
  first.

## Out of scope

- No sase-core edits and no `sase-core-revision.txt` change.
- The other `except Exception` handlers in chop-batch recovery are hold or notice paths,
  not policy mirrors. Leave them alone.
