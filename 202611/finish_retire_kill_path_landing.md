---
tier: tale
size: medium
title: Finish the staged-handoff kill-path retirement landing
goal:
  Make staged handoffs from in-process agent-session successors adopt correctly, retire
  the dead kill-only adopter branches, and close sase-1pj.8.1 and its parent phase
  sase-1pj.8.
proposed_by: bbugyi200.athena.sase-1pj.8.1.land
proposed_with: claude/opus
bead: sase-1pj.8.1
status: done
---

- **PARENT:**
  [202611/retire_staged_handoff_kill_path.md](https://github.com/sase-org/sase--plans/blob/main/202611/retire_staged_handoff_kill_path.md)
- **BEAD:**
  [sase-1pj.8.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1pj/sase-1pj.8.1.md)
- **AGENTS:**
  - [bbugyi200.athena.sase-1pj.8.1.land](https://github.com/sase-org/sase--agents/blob/main/sessions/bbugyi200.athena.sase-1pj.8.1.land.md)
- **COMMITS:**
  - [ced5901](https://github.com/sase-org/sase/commit/ced5901dd5612f00f767b3e7bc695a0936bdc14f)
    — feat(axe): finish staged-handoff kill-path landing

# Finish the staged-handoff kill-path retirement landing

This tale holds the remaining work found by the land agent for epic `sase-1pj.8.1`
("Retire the staged-handoff kill protocol", plan
`plan:202611/retire_staged_handoff_kill_path.md`). All seven phases (`sase-1pj.8.1.1`
through `.7`) are closed. Their commits are `f4254582d8`, `11147ccf2f`, `671431ee80`,
`8caa3fd1f5`, `d3518e567d`, and `fdad1aa7bf`. The `staged_handoffs` flag and its bead
`sase-1pk` are gone, provider text teaches Staged then `/sase_final`, and the readout
classifier is in Rust core. Integration with commits that landed after the epic started
is clean: the agent-hijack work (`01a28287c8`, `f163ba74c0`, `8ac56e2a09`, `aa7eedb1c5`)
already builds on the retired kill path, and the readout and runner shutdown splits
(`323a91d69f`, `d191279c26`) kept the new code. Two gaps remain, and both count as epic
work. No coder should land without them.

Read the epic plan with
`sase artifact read plan:202611/retire_staged_handoff_kill_path.md` and the epic bead
with `sase bead read sase-1pj.8.1`. The epic's invariants still apply. Never add a
kill-based fallback. Keep stage-before-declare, the one-handoff locking, user-kill
discard, external-kill evidence, and idempotent replay. Shared domain policy belongs in
`sase-core`. The identity fix below is a narrow Python adapter correction next to the
existing Python checks, so it adds no new Rust API. This tale edits no memory, generated
skills, or user docs; `sase-1pj.9` owns those. Do not run `just check-full`.

## 1. In-process successors cannot adopt a second staged handoff

**Regression (caused by this epic).** Pipe, plan-coder, and question successors continue
inside the same runner through `continue_as_successor`
(`src/sase/axe/run_agent_successor_staged.py`). The loop keeps the runner's
`AgentExecContext` (`ctx.agent_name` is the session base, e.g. `fakey-e2e`), and
`LoopState.current_role_suffix` and `current_artifacts_dir` move to the member
(`fakey-e2e--1`). When that member stages any later handoff, staged dispatch reads the
member's `agent_meta.json` as `run_metadata`. The adopters in
`src/sase/axe/run_agent_exec_staged_adopters.py` then refuse it:

- `_runner_ctx_from_metadata` (used by the pipe and gate adopters, and by plan and
  questions through `_runner_ctx_from_approval_context`) raises `StagedHandoffError`
  with "persisted agent identity differs from the active runner identity" whenever
  `run_metadata["name"] != ctx.agent_name`.
- `_runner_ctx_from_monitor_metadata` accepts only the exact root promotion to `--0` or
  `--plan`.

The land agent reproduced this on `bdbbce02e3`. It ran a two-hop staged pipe through the
fakey harness (`tests/fakey/test_pipe_e2e.py` helpers): the root pipes, the successor
`fakey-e2e--1` pipes again, and adoption of the second stage raises at
`run_agent_exec_staged_adopters.py:28`. Before this epic, the default (flag-off) kill
path adopted that second hop using the loop `ctx`. Phase `sase-1pj.8.1.1` hit the
failure, shrank `test_two_link_chain_then_bound_leaves_the_agent_running` to the one-hop
`test_pipe_chain_bound_leaves_the_agent_running`, and recorded a follow-up.

Required behavior:

1. A staged record whose persisted identity is the runner's **current in-process
   agent-session member** must adopt. Recognize that member from durable facts the
   runner already owns: the persisted name equals
   `plan_chain_agent_name(ctx.agent_name, state.current_role_suffix)` for a non-empty
   canonical suffix, and the persisted agent-session value matches the runner's session
   base (`sase.plan_chain` helpers). Put the predicate in one shared helper used by
   `_runner_ctx_from_metadata` and `_runner_ctx_from_monitor_metadata`. Keep the
   existing root-promotion acceptance, and keep rejecting any other mismatched identity
   with the same error.
2. For an accepted in-process member, keep `ctx.agent_name` as the runner's session
   base, as the legacy loop did. Take per-member facts (`agent_meta`, model, provider,
   VCS provider) from the persisted metadata. Successor naming then continues the
   session: the second pipe hop creates `fakey-e2e--2` with
   `piped_from == "fakey-e2e--1"` and `pipe_depth == 2`, never a doubly suffixed
   `fakey-e2e--1--2`. For plan and questions, check that
   `_runner_ctx_from_approval_context`'s frozen `execution` overrides still name
   successors from the session base. If the frozen approval context names the member
   instead, normalize it so the coder or question successor names match what the legacy
   loop produced.
3. Confirm the settlement identity check in
   `src/sase/axe/run_agent_exec_staged_settlement.py` (final context `agent_id` vs
   persisted name) already accepts the member. Replay must still produce one logical
   successor: the existing `prepare_staged_pipe_successor` and staged-successor identity
   checks must hold on replay after a crash.

Tests:

- Restore two-hop coverage in `tests/fakey/test_pipe_e2e.py`. Return the bound test to
  `max_agent_pipe_chain=2`: root pipes, `fakey-e2e--1` pipes, `fakey-e2e--2` exists with
  depth 2 and `piped_from` `fakey-e2e--1`, and then a third pipe is refused at the bound
  with no `STAGED_HANDOFF_FILENAME` written and the agent still running. Use the
  pre-epic version of that test (`git show f4254582d8~1:tests/fakey/test_pipe_e2e.py`)
  as the behavioral reference, with staged `_pipe` calls in place of the killed ones.
- Add focused adopter tests: an in-process member accepted for pipe, gate, and monitor
  (and plan or questions through the approval context); a foreign identity still
  rejected; root promotion still accepted. Extend an existing staged-adoption module
  such as `tests/test_questions_pipe_staged_adoption.py` or
  `tests/monitor/test_monitor_start_activation.py`, or add a new focused module.

## 2. Dead kill-only branches survive in the staged adopters

The epic's runtime-retirement section required simplifying "kill-only wrappers and
`interrupted`/`persist_chat` branches in the
`axe/run_agent_exec_{plan,questions,pipe,gate,monitor}*` adopters and helpers" and
replacing marker-driven fixtures with real staged adoption. Phase `sase-1pj.8.1.4`
instead kept those branches and drove them from tests through
`tests/_legacy_handoff_markers.py`. No production caller reaches them:

- `interrupted` parameter and branches (`normalize_handoff_interruption_state`, early
  `finalize_handoff_artifacts_as_completed`, `reset_killed()`) in `_handle_plan_handoff`
  (`src/sase/axe/run_agent_exec_plan.py`), `_handle_questions_handoff`
  (`run_agent_exec_questions.py`), and `_handle_pipe_handoff`
  (`run_agent_exec_pipe_handoff.py`). The only production entry points,
  `handle_staged_*_marker`, always pass `interrupted=False`.
- The `provider_response is None` / `planner_response is None` synthetic-chat branches,
  plus `persist_chat=... is None` at `run_agent_exec_plan.py` and
  `run_agent_exec_questions.py` calls into `handle_gate_marker`. Also
  `handle_gate_marker`'s `persist_chat` (default `True`) synthetic gate chat in
  `run_agent_exec_gate.py`: the staged gate adopter passes `False`, and staged plan and
  questions always have a real response.
- `capture_monitor_handoff`'s `reset_killed_state` and `require_capture=False` branches
  and its "legacy marker path" docstring in `run_agent_exec_monitor.py`. The staged
  monitor adopter always passes `require_capture=True`.
- The `staged_id is None` / `staged_plan is None` paths in `_handle_pipe_handoff`. The
  staged pipe adopter always supplies a staged ID, so make it required.

Remove these branches and parameters, and make the staged entry points the only adoption
API. Follow `symvision.md` for any helper this leaves unused (for example
`normalize_handoff_interruption_state` or `_format_gate_response`): delete genuinely
dead code rather than whitelisting it. Keep live host behavior even where a function is
still named `handle_*_marker`. Preserve real final replies, plan preview extras,
lifecycle phases, metadata, receipt replay, and inline-continuation `None` semantics.

Migrate the test consumers of `tests/_legacy_handoff_markers.py`: `handle_plan_marker`
(about 40 calls in 13 files), `handle_questions_marker` (17 in 5), `handle_pipe_marker`
(5 in 1), and `handle_monitor_marker` (4 in 3, including
`tests/test_turn_handoff_outcome_parity.py`). Point them at the staged entry points
(`handle_staged_plan_marker`, `handle_staged_questions_marker`,
`handle_staged_pipe_marker` with a real staged ID and record, and
`capture_monitor_handoff` with `require_capture` semantics) and a real provider
response. A thin shared helper renamed for staged adoption is fine if it keeps per-test
edits small. Assertions about synthetic hand-off chat should check the real response
plus the hand-off extra sections instead. Keep `write_legacy_handoff_marker`, or an
equivalent fixture writer, only for tests that seed archived marker files to prove
historical readers and leftover-marker non-adoption. Delete tests that exist only for
the removed kill branches.

Afterwards, audit with
`rg -n 'interrupted=|persist_chat|reset_killed_state|require_capture|_legacy_handoff_markers' src tests`.
Remaining hits must be live staged behavior or archived-evidence fixtures.

## 3. Verification

1. If `sase_core_rs` lacks `classify_instruction_readout`
   (`tests/instructions/test_readout_adapter.py` fails with "installed wheel is stale"),
   run `just install-venv` first. It is long; hand it to `/sase_monitor` if needed.
2. Run focused suites: `tests/fakey/test_pipe_e2e.py`,
   `tests/test_questions_pipe_staged_adoption.py`,
   `tests/test_staged_handoff_acceptance_matrix.py`, `tests/monitor/`,
   `tests/axe/test_run_agent_exec_pipe.py`, `tests/test_turn_handoff_outcome_parity.py`,
   every migrated `_legacy_handoff_markers` consumer, the staged reconcile and crash
   suites (`tests/test_staged_reconcile_*.py`,
   `tests/test_staged_handoff_reconcile.py`), and `tests/instructions/`.
3. Read `lint_and_test.md` and run `sase tool run check`. Run `just fix` inline first.
   Use a prepared-completion monitor for the final check if it will not fit in the turn.

## 4. Closeout (final step; this tale finishes the landing)

Close in the same turn as the final code. Do not wait for this work's own commit, SHA,
push, or CI.

1. Run `sase bead epic-symbols sase-1pj.8.1` and `sase bead epic-symbols sase-1pj.8`.
   Both are empty at planning time. Resolve any entry that appears (wire it up,
   privatize it, add a non-test pragma, or delete it per `symvision.md`). Re-key an
   entry only when a still-open later bead genuinely needs it.
2. Close the epic: `sase bead close sase-1pj.8.1 --note "<verification>"`. The note
   should cover:
   - All seven phases verified against their commits.
   - Flag and `sase-1pk` retired.
   - Readout pin and binding verified.
   - Post-start commits (agent-hijack, runner shutdown split, readout split, flag-gate
     unblock) integrated with no conflicts.
   - Inline-successor identity regression fixed, with two-hop pipe coverage restored.
   - Dead kill-only adopter branches retired.
   - Follow-up outcomes: the multi-hop pipe was fixed as epic work. The 8.1.7
     prepared-monitor identity issue was recorded as a DISCOVERED ISSUE on `sase-1pj`.
   - Check ToolRun ID and focused test counts.

   Never use `--force` merely to make the close succeed.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path
   `sase bead read sase-1pj.8.1` shows for
   `plan:202611/retire_staged_handoff_kill_path.md`).
5. The epic's `parent_bead` is phase `sase-1pj.8` ("Remove the flag and the kill-path
   machinery"). Verify its description is satisfied: Off branch deleted, flag bead
   `sase-1pk` closed, runner kill and coping code removed, provider adapter text
   updated, readout classifier taught the new protocol. Then close only that phase:
   `sase bead close sase-1pj.8 --note "<what you verified>"`. Do not close `sase-1pj` or
   `sase-1pj.9`; the `sase-1pj` land agent owns them.
