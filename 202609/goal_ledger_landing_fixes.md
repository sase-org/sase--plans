---
tier: epic
title: 'Goals G1 landing fixes: ledger correctness in sase-core and CLI honesty in
  sase'
goal: 'Every goal-ledger defect found while landing G1 (sase-1bu) is fixed and tested
  before the frozen contract ships. Actions on unknown ids refuse instead of minting
  phantom goals, criteria keep stable ids, one corrupt file never takes down the whole
  ledger, and the I/O probe proves what it claims. The `sase goal` CLI does what its
  help says, and sase''s core pin covers the fixes.

  '
phases:
- id: core-fixes
  title: Ledger correctness fixes in sase-core
  depends_on: []
  size: medium
  description: 'core-fixes: fix sase-core goal append/reduce/read/projection/doctor/probe
    defects (unknown-id refusals, id normalization, stable criterion ids, criteria
    validation, corrupt-file isolation, projection rebuild and header preservation,
    reopen/claim reducer gaps, fixture and basis wire fidelity, a real I/O probe)
    with Rust tests.'
- id: cli-fixes
  title: CLI, reconcile, and pin fixes in sase
  depends_on:
  - core-fixes
  size: medium
  description: 'cli-fixes: ratchet the sase-core pin past core-fixes, then fix the
    sase goal CLI (-x numbering, -s choices, cross-project ids, doctor context and
    wording), locked reconcile commits, the acceptance test that leaks into the real
    SASE home, and a stale docstring, with tests.'
proposed_by: bbugyi200.athena.sase-1bu.land
parent_bead: sase-1bu
create_time: 2026-09-28 11:36:38
status: done
bead_id: sase-1bu.8
---

- **PROMPT:** [prompts/202609/goal_ledger_landing_fixes.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202609/goal_ledger_landing_fixes.md)
- **PARENT:** [202609/goal_ledger.md](https://github.com/sase-org/sase--plans/blob/main/202609/goal_ledger.md)
- **BEAD:** [sase-1bu.8](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1bu/sase-1bu.8.md)

# Goals G1 landing fixes

## Context

Epic `sase-1bu` (SASE Goals G1) finished all seven phases, but its land agent's audit
found real defects in the epic's own code. The contract is frozen at the moment G1
lands, so these are fixed now as G1 epic work, not deferred. This child epic's
`parent_bead` is `sase-1bu`. When it lands, its land agent resumes and closes the
`sase-1bu` landing. Do **not** close `sase-1bu`, edit its plan file, or run its closeout
from a phase.

Each defect below was reproduced against the real binding. For example, a scratch ledger
through `sase.core.goal_ledger_facade.goal_ledger_append`:

- `edit` of the unknown id `zzzzz` returned `applied` and created `live/zzzzz`, a
  phantom active goal.
- `criteria_removed: ["1"]` returned `applied`, removed nothing, and left only an
  `unknown_criterion` diagnostic.
- `goal_id: "4AH92"` for the existing `4ah92` wrote a second `items/4AH92/` plus
  `live/4AH92`.

Out of scope, already triaged by the `sase-1bu` land agent:

- the projection-aware hot read and the warm-list perf miss (task `sase-1c3`);
- the PyPI `sase-core-rs` window move (normal release process);
- a numeric `↑ N unpublished` count (the outbox has no count source by design);
- editor/LSP goal payload completion (kind-only in G1 by design).

The sibling epics G2–G6 stay out of scope.

## Rules for both phases

- Work in sase-core happens in the linked checkout opened with
  `sase repo open sase-core`. Read its `AGENTS.md` first. Rust files stay at 1,500 lines
  or fewer, and Python files under the 1,000-line `toobig` cap.
- Stable refusal codes are snake_case. New codes are additive, and existing codes keep
  their meaning.
- The committed fixtures under `crates/sase_core/src/goal/tests/fixtures/` freeze the
  vocabulary once G1 lands. G1 has not landed yet, so this epic may correct them.
- Verify with `sase tool run check` in every repo the phase touched, never `check-full`.
  Route long runs through `/sase_monitor`. Failures that reproduce identically on a
  clean base tree are recorded as `PROPOSED FOLLOW-UP:` notes on the phase bead, not
  fixed here.
- Phase workers never create beads. Anything genuinely out of scope becomes a
  `PROPOSED FOLLOW-UP:` note on the phase bead.

## Phase: core-fixes

All of this happens in sase-core, mostly in `crates/sase_core/src/goal/`. Every item
needs a Rust test in the existing `goal/tests/` or `goal/ledger/tests.rs` suites.

1. **Unknown ids refuse; they never mint phantoms** (`ledger/append.rs`, around the
   `plan_state` / `GoalStateWire::empty` fallback).
   - A touched goal with zero events plans against `None`. Then:
     - `edit`, `drop`, and `reopen` refuse with `goal_not_found`;
     - `merge` with an unknown source refuses with `goal_not_found`;
     - `merge` with an unknown target refuses with `target_not_found`.
   - A `new` whose minted or `new_goal_id`-overridden id already has events refuses with
     `goal_already_exists`.
   - A refused append writes nothing: no `items/<id>/` directory and no `live/<id>`
     marker.
   - Test each refusal, including the no-write guarantee.
2. **Normalized ids everywhere.** The planners and the write step use the id from
   `parse_goal_id`, not the raw input string. That covers the envelope `goal_id`, the
   `items/` and `live/` paths, and the merge `into`/`from`. `merge_into_self` compares
   normalized ids.
   - Test that `Edit{goal_id:"ABCDE"}` on the existing `abcde` writes only under
     `abcde`.
   - Test that merging `ABCDE` into `abcde` refuses with `merge_into_self`.
3. **Criterion ids are `<event_id>.<index>`, as the contract says.** In `reduce.rs`'s
   edited branch, the id currently comes from `self.state.criteria.len()`. It must be
   the criterion's index within that event's `criteria_added`, which is how `created`
   already does it.
   - Test with concurrent edits: two edits share one basis and each adds a criterion.
     Reducing E1+E3 and then E1+E2+E3 gives E3's criterion the same id both times.
4. **Edit validation tells the truth** (`actions.rs` `plan_edit`, and `plan_new` for the
   empty checks).
   - Every `criteria_removed` id must name a criterion currently on the goal. Otherwise
     refuse with `criterion_not_found`.
   - The 10-criteria cap counts current criteria, minus valid removals, plus additions.
   - An edit where every supplied field equals its current value, and nothing else
     changes, refuses with `no_changes`.
   - An empty title or outcome after trimming refuses with `title_empty` /
     `outcome_empty` instead of `title_too_long` / `outcome_too_long`.
5. **One corrupt event file never breaks the ledger** (`ledger/read.rs`,
   `ledger/doctor.rs`).
   - An unparseable event JSON makes only its goal `readable: false`, with an
     `unreadable_reason` naming `unparseable_event` and the file. `goal_ledger_list`,
     `goal_ledger_show`, `goal_ledger_history`, and `goal_ledger_doctor` all keep going,
     and doctor reports that goal as unreadable.
   - The "file vanished between readdir and read" skip must match on the I/O error kind.
     Today it matches the string prefix `"vanished"`, which the `"goal ledger io: "`
     Display prefix makes unreachable.
   - Test a corrupt goal next to healthy neighbours for list, show, and doctor.
6. **A corrupt `goals-hot.json` is rebuildable** (`ledger/projection.rs`). An
   unparseable projection reports a non-Fresh status (`SchemaMismatch` is fine) instead
   of a hard `Data` error, and `refresh_goal_projection` rebuilds it.
   - Test with garbage bytes in the projection file.
7. **Doctor repair keeps the projection header** (`ledger/doctor.rs`, around the
   `refresh_goal_projection(... "", "", GOAL_DEFAULT_FETCH_TTL_SECONDS)` call).
   - Add optional `watermark_path`, `outbox_path`, and `fetch_ttl_seconds` to the doctor
     request wire (serde default).
   - When a field (these three, or `project`/`mode`) is absent, keep the existing
     projection header's value rather than blanking it or defaulting to `local`/`""`.
   - Test that a repair of a stale marker leaves the header unchanged.
8. **Reducer gaps** (`reduce.rs`).
   - `reopened` clears `merged_into`; the target's `merged_from` stays as history.
   - A claim beaten by a concurrent human `canceled` settlement is no longer shown as
     the active claim: mark it superseded, or record its timeline effect as not applied,
     and keep the `canceled_beats_claim` diagnostic.
   - Test both.
9. **Wire fidelity** (`wire.rs`, fixtures).
   - Serialize `basis` always, so a `created` event writes `"basis": null` as the
     contract example shows. Drop the `skip_serializing_if` on `GoalEventWire.basis`;
     readers already accept both forms.
   - Every fixture event id is currently 25 characters. Correct all twelve fixtures to
     valid 26-character ids, and keep each fixture's basis pointing at the corrected
     ids.
   - Add a fixture test asserting that every fixture `event_id` and non-null `basis`
     passes `parse_event_id`.
   - Add a test that a serialized `created` event contains `"basis":null`.
10. **An I/O probe that can fail** (`ledger/probe.rs`, `ledger/read.rs`).
    - Record the goal id of every `items/<id>/events` directory actually opened during
      the read. Compute `settled_event_opens` as the opened ids that have no `live/`
      marker. Today it is computed from the returned goals, so it is zero by
      construction.
    - Count `store_reads` where `STORE.json` is really read.
    - The probe test seeds 1,000 settled plus 10 live goals, as the plan specified;
      write event files directly if seeding through append is too slow in debug. Assert
      zero settled opens and exactly 10 event-dir opens.
    - Add a negative test (for example, a history scan through the same probe) that
      shows the probe does count settled opens.
11. **Bindings.** Run `goal_render_list` and `goal_render_card` inside
    `py.allow_threads` like the other goal bindings
    (`crates/sase_core_py/src/goals/mod.rs`). Add or update binding round-trip tests for
    any changed request wire (the doctor request fields).
12. **sase stays green against the new core.** Rebuild the local extension
    (`just install` in sase rebuilds `sase_core_rs` from the linked checkout) and run
    `tests/goals/ tests/test_bead/test_sync_remote_push.py` in sase. If a core fix
    changes behavior a sase test asserts, update that sase test in this phase. Leave
    every other sase change to `cli-fixes`.
13. Verify with `sase tool run check` in sase-core.

## Phase: cli-fixes

All of this happens in sase. By now the host has committed and pushed `core-fixes`'s
sase-core work.

1. **Pin first.**
   - Run `just ratchet-core-revision`, then confirm the new `sase-core-revision.txt` SHA
     contains the `core-fixes` commit (`git merge-base --is-ancestor` in the sase-core
     checkout).
   - Rebuild the local extension with `just install`.
   - If the `core-fixes` commit is not on sase-core's remote yet, stop and record that
     on the phase bead instead of pinning an older SHA.
2. **`edit -x/--remove-criterion N` uses the number `sase goal show` prints.**
   - Today the option is an untyped string passed through as a criterion id
     (`src/sase/main/parser_goal.py` around its `-x` definition, and
     `src/sase/goals/cli.py` `handle_goal_edit`). The card numbers criteria from 1, but
     stored ids are `<event_id>.<index>`.
   - Make it `type=int`, read the goal's current state, and map each 1-based N to the
     criterion id.
   - An out-of-range N prints one stderr line and exits 2 without writing.
   - Update the help text and any `-x` wording in `docs/cli.md` / `docs/goals.md`.
   - Tests: `-x 1` removes the first criterion; `-x 9` on a two-criterion goal exits 2
     and writes no event.
3. **`list -s/--status` validates.**
   - Wire `choices=` to the existing unused `_GOAL_STATUSES` in `parser_goal.py`.
   - Regenerate the completion snapshot with `just sync-completion-spec`, and keep
     `tests/completion/` (including kind coverage) green.
   - Test that `-s bogus` exits 2 from argparse.
4. **Write verbs honor `goal:<project>@<id>`.**
   - Today `edit`/`drop`/`reopen`/`merge` drop the project part (`cli.py`, the handlers
     that call `_normalize_goal_id_token` and keep only the id). Resolve the named
     project's ledger exactly the way `show` already does.
   - `merge` refuses with exit 2 when the source and target name different projects.
   - Test that a cross-project `drop` settles the goal in the named project's ledger and
     leaves the current project's ledger untouched.
5. **Doctor context and wording** (`handle_goal_doctor`).
   - Pass `projection_path`, `project`, `mode`, `watermark_path`, `outbox_path`, and
     `fetch_ttl_seconds` from the resolved `GoalLedger` / goals config, so the
     projection check runs and a repair keeps the header.
   - Pass the same new fields from `src/sase/goals/reconcile.py`.
   - The agent refusal for `doctor --repair` must read
     `sase goal doctor --repair is a human verb: ...`. Today it says `sase goal repair`.
   - Test that after `sase goal doctor --repair`, the goal fast path still handles
     `sase goal list`, because the projection header is intact.
6. **Reconcile commits under the store lock** (`src/sase/goals/reconcile.py`).
   - Today its `git add`/commit runs after `apply_goal_action` has released
     `store_git_write_lock`. Take the hidden clone's `store_git_write_lock` around the
     commit, with the same `authorize_store_mutation` context `write.py` uses. When the
     lock is busy, skip the commit and fail open with a diagnostic.
   - The bead-link publisher hook (`src/sase/sdd/_artifact_link_publication_retry.py`,
     where it calls the goals reconcile) commits after its push and never publishes the
     fix. Make that fix reach the remote, either with one bounded re-push or by marking
     the goals outbox pending so the auto-sync leg publishes it.
   - Tests: a held lock means no commit and no exception; a reconcile commit from the
     bead-link path ends up published or outbox-pending.
7. **Stop the acceptance test from writing into the real SASE home.**
   - `tests/goals/test_goal_acceptance.py`'s offline test calls `monkeypatch.undo()`,
     which also undoes the autouse `_isolate_sase_home`, so the retry writes
     `goals-sync-stats.json` under the real `~/.sase/projects/acme_goals_accept/`.
   - Restore only the publish stub (with `monkeypatch.context()` or by re-setting the
     original), and assert that `SASE_HOME` still points into `tmp_path` before the
     retry.
   - Then inspect `~/.sase/projects/acme_goals_accept/`. If it holds only the leaked
     `goals-sync-stats.json`, delete that directory.
8. **Stale docstring.** Remove the sentence in `src/sase/goals/fetch_worker.py` claiming
   the function is "whitelisted for epic sase-1bu until that phase lands". No such
   whitelist exists, and the CLI calls it.
9. **CLI regression for phantom goals.** `sase goal edit <unknown-id> -t x` and
   `sase goal drop <unknown-id> -w x` exit non-zero, report `goal_not_found`, and create
   no `live/` marker, in both local and shared mode.
10. Verify: `sase tool run check` in sase is green; `just symvision` is clean; and
    `sase bead epic-symbols <this epic>` is empty.

## Definition of done

1. Every numbered item in both phases is implemented and tested.
2. `sase tool run check` passes in sase-core and in sase.
3. sase's `sase-core-revision.txt` contains the `core-fixes` sase-core commit.
4. `tests/goals/` passes and no longer writes outside `tmp_path`.
5. The leaked `~/.sase/projects/acme_goals_accept/` directory is gone.
6. There are no `--epic-symbol` entries for this epic.

After this epic lands, its land agent resumes the interrupted `sase-1bu` landing through
`parent_bead`.
