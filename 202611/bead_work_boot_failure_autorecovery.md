---
tier: tale
title: Auto-recover sase bead work from boot-failure agent artifacts
goal:
  sase bead work auto-recovers nameless boot-failure owners and both stranded epics are
  relaunched.
size: medium
decisions:
  relaunch_lane:
    ask:
      Should the coder itself relaunch the two stranded epics using its workspace venv's
      fixed sase?
    choices:
      workspace:
        Coder relaunches both epics now with its venv's fixed sase, live-validating the
        recovery path
      after_land:
        Only land the fix; Bryan reruns the relaunch command after the installed CLI has
        the fix
    default: workspace
    why:
      it recovers the epics now and is the only live validation of the new auto-recovery
      path
    answer: workspace
proposed_by: bbugyi200.athena.1br
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 06:19:04
status: wip
---

# Auto-recover `sase bead work` from boot-failure agent artifacts

## Problem

On 2026-11-12 at 05:29–05:30, a `sase bead work sase-1sd` launch spawned 8 agents that
all crashed during bootstrap with an `ImportError` (the editable sase install changed
underneath the launch — see the auto-restart scan signature
`ImportError cannot import name`). Each crash happened inside `bootstrap_agent_run` →
`extract_directives_and_write_meta`, i.e. **before** the agent's `name`, `bead_id`,
`phase_bead_id`, `epic_bead_id`, or `agent_clan` fields were written to
`agent_meta.json`. The stranded artifact directories (for example
`~/.sase/projects/gh_sase-org__sase/artifacts/ace-run/202611/12/20261112052956`) contain
only launch-time metadata: `pid`, `process_identity`, `lifecycle_phase: "booting"`,
`stopped_at`, an `error_report.md`, and **no** `name` and **no** bead ids. The
agent-name registry still holds the reservation for each owner (for example
`sase-1sd.3`) pointing at that artifact directory, and the failed run holds a workspace.

Retrying with `sase bead work sase-1sd bob-cli-kq -Y` now wedges permanently:

```
BLOCKED (blocked) sase-1sd.3 bead=sase-1sd.3  agent owner 'sase-1sd.3' is not
associated with expected bead sase-1sd.3; observed bead association(s): none; ...
Error: ... Dismiss the listed agent(s) in sase tui before retrying.
```

The relaunch path is supposed to auto-clean dead owners, but the association guard
refuses these records, so the only recovery today is manual per-agent dismissal in the
TUI. Make `sase bead work` auto-recover from this whole class: launches that died before
writing identity metadata.

## Root cause

In `src/sase/bead/cli_work_cleanup_targets.py`, `_require_record_association` decides
whether an existing artifact record may be cleaned up for the slot's owner name:

- If the record carries bead ids matching/related to the expected bead → allow.
- If the record carries **no** bead ids:
  - `session` / `clan` membership → allow (container membership is the evidence).
  - `registry` membership → allow **only** when the record's `agent_meta.name` key
    equals the owner name key.
- Otherwise raise `ForcedReuseCleanupError`, which `classify_slot_owner` converts into a
  `BLOCKED` target; any blocked target aborts the whole launch
  (`format_blocked_cleanup_error` in `cli_work_cleanup_types.py`).

A boot-failure record has no bead ids **and no name**, so the registry-membership escape
never matches and the record blocks forever, even though the only way such a record was
found is through the owner's **own** registry reservation.

Key invariant that makes auto-recovery safe: a nameless record can never be returned by
a name-keyed lookup. In every `membership="registry"` call site the record was located
through the owner's registry entry `artifacts_dir` pointer:

- `classify_slot_owner` → `_concrete_records_for_owner`: merges the record at
  `owner["artifacts_dir"]` (registry pointer) with name-keyed records (which always
  carry a matching name).
- `_select_preserved_slots_from_registry` (fast path, `cli_work_cleanup_selection.py`):
  matches by the owner's `artifacts_dir` first.
- `_fresh_cleanup_target` (verify-before-wipe, `cli_work_cleanup_apply.py`): rescans the
  previewed target's own `artifacts_dir`.

So "registry membership + zero bead ids + no agent name" implies the registry itself
asserts this artifact belongs to the owner name, and trusting it is exactly as safe as
the existing name-match escape.

## Change 1 — relax the association guard for nameless records

File: `src/sase/bead/cli_work_cleanup_targets.py`, function
`_require_record_association`.

In the `if not bead_ids:` branch, after the existing name-match escape, also allow the
record when it has **no** agent name (`_record_agent_name(record)` is `None`) and the
membership is `registry`. Do **not** loosen anything else:

- A record with a **different, non-empty** name and no bead ids must keep blocking
  (enshrined by `test_work_direct_registry_name_mismatch_without_beads_blocks` in
  `tests/test_bead/test_cli_work_epic_launch_cleanup_sessions.py` — that test must stay
  green unchanged).
- A record with unrelated bead ids must keep blocking.

No new cleanup actions are needed. Once association passes, the existing state ladder in
`classify_artifact_record` / `_record_current_state` already does the right thing:

- dead process, no done marker → `interrupted` → `REMOVE` (the shared wipe in
  `sase/agent/names/_forced_reuse.py` → `_wipe_execute.py` releases the held workspace,
  removes the artifact dir, clears the registry reservation, and dismisses related
  notifications);
- live process → `PRESERVE` (`active_status_for_record` falls back to `STARTING`, so a
  still-booting nameless agent is never killed);
- `waiting.json` present → `KILL` (replace a queued stale agent).

## Change 2 — distinguish the nameless case in the preview detail

File: `src/sase/bead/cli_work_cleanup_targets.py`, function `classify_artifact_record`.

The `if not _record_bead_ids(record):` block appends
`"(no bead metadata; matched by registry owner name)"` for registry membership. For a
nameless record that message is wrong (nothing matched by name). Append a distinct
suffix instead, e.g.
`"(no agent identity metadata; boot-failed launch owned by its registry reservation)"`,
so previews and tests can tell the recovery lane apart.

## Change 3 — tests

Add tests beside the existing ones in
`tests/test_bead/test_cli_work_epic_launch_cleanup_sessions.py` (or a sibling
`test_cli_work_boot_failure_recovery.py` if the file gets unwieldy). Use the existing
fake-home harness (`write_bead_agent_meta`, `seed_diamond`, `make_args`,
`claim_registered_name`) — never the real `~/.sase`. The helper always writes a `name`;
either add a `nameless: bool = False` parameter or rewrite the written `agent_meta.json`
in the test (the existing ancestor-epic test at line ~211 already edits meta JSON after
the fact).

1. **Dead boot-failure stub auto-recovers.** Registry entry claimed for a phase owner
   name (via `claim_registered_name(phase_id, artifact_dir, replace_existing=True)`)
   pointing at an artifact whose `agent_meta.json` has pid/`stopped_at` but no `name`
   and no bead ids, and no `done.json`.
   `handle_bead_work(make_args(epic_id, yes_to_all=True))` must not print
   `not associated with expected bead`, must wipe that owner, and must launch the phase.
   Mirror the stub pattern of
   `test_work_beadless_agent_session_members_do_not_wedge_retry` (stub
   `wipe_agent_name_for_reuse`, `rebuild_name_registry`, and the launcher). A dead stub
   needs `is_process_alive` to be false: write a `stopped_at` timestamp and a non-live
   pid, or monkeypatch `sase.agent.names.is_process_alive` the way neighboring tests
   handle liveness.
2. **Live boot stub is preserved, not killed.** Same setup but the process reads as
   alive (e.g. `pid=os.getpid()` with no `stopped_at`): the slot must classify
   `PRESERVE` (state `STARTING`), nothing wiped, no replacement launch for that slot,
   and no `BLOCKED` abort.
3. **Land-agent slot variant.** Same dead-stub recovery for the land owner name
   (`<epic>.land`, expected bead = the epic id) to cover the `slot_id=f"{epic_id}:land"`
   path.
4. **Named mismatch still blocks.** Covered by the existing
   `test_work_direct_registry_name_mismatch_without_beads_blocks`; do not weaken it.

## Change 4 — live recovery of the two stranded epics

> [!decision] relaunch_lane = workspace

After `just check` passes, recover the real epics from the coder's checkout so the new
code path is the one doing the recovery (the globally installed `sase` does not have the
fix yet):

```bash
.venv/bin/sase bead work sase-1sd bob-cli-kq -Y
```

Expectations, as of planning time (recompute from the command's own preview — live state
drifts):

- `sase-1sd`: six nameless boot-failure artifacts (`sase-1sd.3`–`sase-1sd.7`,
  `sase-1sd.land`) become `REMOVE`; `sase-1sd.1` was RUNNING (preserved); `sase-1sd.2`
  was WAITING (killed and relaunched). The command should relaunch the non-preserved
  phases plus the land agent.
- `bob-cli-kq`: phases `.1`, `.2`, `.4`, `.5` were already RUNNING again (preserved);
  only the dead `bob-cli-kq.land` artifact is removed and the land agent relaunched.
  This epic already recovers without the fix; it is included because the user asked for
  both.

Operational notes:

- The command is slow (owner discovery and registry stages took 1–2 minutes each on this
  host); run it in the foreground with a generous timeout (at least 15 minutes) and run
  it **once** — do not loop it.
- A `Local commands in .../sase/sase.yml were skipped (no-acknowledgment)` warning on
  every sase invocation from the checkout is expected noise; do not "fix" it.
- If the run still reports `BLOCKED` targets, inspect their detail lines: targets of the
  name-mismatch or unrelated-bead class are genuine conflicts — do **not** force or
  manually delete anything; finish the tale and report the residual blockers verbatim in
  the final answer.
- Confirm the relaunch with `sase agent list` (the relaunched phase/land agents should
  appear as RUNNING/WAITING/QUEUED) and summarize per epic what was removed, preserved,
  and launched.

## Verification

- `just check` (do not run `just check-full` unless explicitly repairing CI).
- The live recovery in Change 4 doubles as end-to-end validation.

## Out of scope

- The `sase agent auto-restart` subsystem (it classified these failures as verdict `ask`
  and is a separate lane; no changes there).
- Any relaxation for records with a mismatched non-empty name or unrelated bead ids.
- New CLI flags or subcommands (none added, so no CLI-rules churn).
- Rust core: the artifact scan already provides everything needed; the cleanup policy
  being changed is bead-work-CLI-specific Python and stays in this repo.
- Preventing the original `ImportError` (live-update skew during launch) — that is the
  auto-restart/update problem, not this tale.
