---
tier: tale
size: medium
title: Finish landing sase-1q4 - keep explicit %wait targets through typed admission
goal:
  An explicit `%wait(..., target=session|turn)` in an agent-initiated or typed launch is
  bound by the launched agent's runner instead of being silently dropped, the last
  sase-1q4 epic symbol is retired, and epic sase-1q4 is closed with its plan marked
  done.
proposed_by: bbugyi200.athena.sase-1q4.land
proposed_with: claude/opus
bead: sase-1q4
create_time: 2026-10-31 07:39:08
status: wip
---

- **PARENT:**
  [202610/fork_wait_session_turn_target.md](https://github.com/sase-org/sase--plans/blob/main/202610/fork_wait_session_turn_target.md)
- **BEAD:**
  [sase-1q4](https://github.com/sase-org/sase--beads/blob/main/pages/sase-1q4/README.md)

# Finish landing sase-1q4: keep explicit `%wait` targets through typed admission

This tale finishes the landing of epic **sase-1q4** ("Explicit session/turn targets for
fork and agent waits"). The land agent checked all eleven phases (sase-1q4.1 through
sase-1q4.11, all closed) against the epic plan
`plan:202610/fork_wait_session_turn_target.md` and the code on master `4a4b6fede8`. Two
things remain, plus the closeout:

1. **Epic-caused bug (fix it).** Typed launch admission silently drops an explicit
   `target=` on an agent unit's `%wait`.
2. **Leftover epic symbol (retire it).**
   `--epic-symbol 'sase-1q4(wait_targets_with_descriptions)'` is still in the
   `Justfile`.
3. **Closeout.** Close sase-1q4, run symvision, and mark the plan file done.

The follow-up triage is finished and recorded as note #2 on sase-1q4. Do not re-triage
it and do not create beads. sase-1q4 has no `parent_bead`.

## 1. Typed admission drops explicit `%wait` targets

### What is wrong

Every LaunchApproval request builds a typed plan, whatever the prompt
(`src/sase/agent/launch_request.py` `_typed_plan_payload`). After approval,
`src/sase/agent/launch_request_response.py` sends any plan with units through
`dispatch_typed_launch_request`. So every agent-initiated launch (`/sase_run`) goes
through typed admission. The same is true of user launches whose prompt contains `%if::`
or `%proc` (`src/sase/agent/direct_typed_launch.py`), and of AXE chop proposal launches.

In sase-core, `crates/sase_core/src/agent_launch/typed_units.rs` handles every `%wait`
in an agent unit like this:

- It always strips the `%wait` from the unit prompt (`regions_to_remove`).
- It stores the targets in `raw_waits`. `RawWaitTarget.wait_target` keeps the parsed
  target.
- `plan_resolution.rs` then turns each `RawWaitTargetKind::Agent` into a plain
  `WaitTargetWire::Agent { name }`. That wire has no target field.
- Python admission resolves the wait with
  `launch_admission_runtime._resolve_agent_wait(name)`, which calls
  `find_named_agent(name)`. That lookup matches one artifact by exact name and knows
  nothing about sessions or turns.
- `agent_unit_dispatch_prompt` (`admission.rs`) rebuilds the dispatched prompt without
  any waits.

So the `target=` is lost before the launched agent's runner (`resolve_wait_state`, the
shared `wait_target_binding` binder from sase-1q4.4/.5) could bind it. Probe on master:

```python
from sase.agent.launch_request import _typed_plan_payload
p = _typed_plan_payload('%wait(planner, target=turn)\nDo work')
# p['units'][0]['waits'] == [{'kind': 'agent', 'name': 'planner'}]
# p['units'][0]['payload']['prompt'] == 'Do work'
```

Effects:

- `%wait(16h, target=turn)` launched through `/sase_run` does not pin 16h's current
  turn.
- `%wait(16h--plan, target=session)` waits for one member, not the owning session.

sase-1q4.5 recorded this gap as a PROPOSED FOLLOW-UP. It is caused by the epic: before
sase-1q4, `target=` was a parse error, not something silently ignored.

### Fix: hand explicit-target agent waits to the launched agent's runner

Admission never needs these waits. They belong to the runner, which already binds,
persists, evaluates, displays (`wait_turn_bindings`), and edits them. So a `%wait`
occurrence with an explicit `target=` in an agent unit stays in that unit's prompt
verbatim. The dispatched agent's runner then binds it, exactly as on a direct
(non-typed) launch.

All of the following is in sase-core (`sase repo open sase-core -r "<why>"`; read its
`AGENTS.md`). Most of it is in `crates/sase_core/src/agent_launch/typed_units.rs`, with
some in `plan_resolution.rs`.

- **Forwarded occurrence.** A `%wait` occurrence is forwarded when all of these hold:
  - it carries a valid explicit `target=` (`session` or `turn`);
  - it is in an **agent** unit with no `%if` condition;
  - none of its agent targets names another unit's identity in the same plan;
  - it has no `unit=` or `proc=` key.
- **What happens to a forwarded occurrence.**
  - Do not add its span to `regions_to_remove`, so it stays in `AgentUnitWire.prompt`
    and the dispatched prompt.
  - Push none of its targets into `raw_waits`. Admission must neither wait on them nor
    arm `wait_for_epics_of` for them; the runner applies the epic-follow rules itself.
  - Any `bead=`, `hood=`, or `time=` keys in the same occurrence go to the runner along
    with it. The runner supports all three.
  - Keep every existing occurrence-level validation (`wait-target-*`, for_epic
    conflicts) exactly as it is.
- **Exact text.** The approval digest covers unit prompts. The kept text must be the
  original directive text, so a prompt without `target=` produces the same plan and the
  same `content_digest` as today. Add a test that pins this.
- **Mixed typed-only keys.** An occurrence that combines `target=` with `unit=` or
  `proc=` in an agent unit is an error. Use a new diagnostic
  `wait-target-typed-mixed-keys`, with a message telling the author to move the
  `unit=`/`proc=` waits into their own `%wait`.
- **Planned sibling unit.** `target=turn` naming the identity of another unit in the
  same plan (the `agent_names` map in `plan_resolution.rs`) is an error, with new code
  `wait-target-turn-planned-unit`. A unit that has not launched yet has no current turn.
  `target=session` naming a sibling keeps today's behavior: strip the occurrence and add
  a logical wait. The target is a no-op there, as it is on `%proc`/`%if`.
- **Unchanged.** `%proc` units and `%if` units keep their current behavior:
  `wait-target-turn-typed-unsupported` for `turn`, and `session` as a no-op.
- **Implementation detail.** Whether the unit is agent, proc, or conditional, and the
  sibling identities, are only fully known after the directive scan, and stripping
  happens at the `strip_prompt_regions` call. Record candidate forwarded occurrences
  (span, agent names, target) during the scan. Decide on them before stripping, and run
  the sibling check where `agent_names` is built. If deferring the sibling check past
  stripping is awkward, you may decide it from identities collected during the fanout
  scan, but the behavior above must hold.

Rust tests go in `crates/sase_core/src/agent_launch/tests/typed_plan.rs`:

- Update `typed_launch_wait_target_turn_skips_epic_follow`.
  `"%wait(planner, target=turn)\nDo work"` now gives no admission waits and no
  `wait_for_epics_of`, and a unit prompt (and `agent_unit_dispatch_prompt`) that
  contains `%wait(planner, target=turn)`.
- Explicit `target=session` on an external exact member, e.g.
  `%wait(16h--plan, target=session)`, is forwarded verbatim.
- `%wait(a, bead=sase-1, target=turn)` is forwarded whole.
- `%wait(unit=unit-1, a, target=turn)` reports `wait-target-typed-mixed-keys`.
- A two-unit plan where unit 2 has `%wait(<unit-1 identity>, target=turn)` reports
  `wait-target-turn-planned-unit`. The same plan with `target=session` keeps the logical
  wait.
- A prompt with no `target=` has the same `content_digest` before and after this change.
  Assert a stable fixture digest or compare against the stripped form.
- The `%proc`/`%if` rejection tests keep passing.

Run `sase tool run check` inside the sase-core checkout.

In sase:

- Run `just rust-install` so this checkout's `.venv` has the new core. It is a
  known-long command, so give it an explicit long timeout.
- Add a Python test, e.g. in `tests/test_directives_wait_target.py` or a new
  `tests/test_typed_admission_wait_target.py`:
  - Build the plan with `_typed_plan_payload('%wait(planner, target=turn)\nDo work')`.
  - Rehydrate it with `typed_plan_from_request`.
  - Check that `agent_unit_dispatch_prompt(plan.units[0].payload)` contains
    `%wait(planner, target=turn)` and that `plan.units[0].waits` is empty.
  - Check that `extract_prompt_directives` (or the runner's directive path the other
    `%wait` target tests use) on that dispatch prompt yields
    `wait_targets == {"planner": "turn"}`.
- Run the existing typed-admission tests and the sase-1q4 shared regression set (listed
  in the epic plan's "Shared regression set"), plus `tests/test_wait_target_binding.py`
  and `tests/test_directives_wait_target.py`.
- Docs:
  - In `docs/reference/macros/directive-launch.md`, in the `%wait` `target=` paragraph,
    replace the typed-launch sentence. It should say:
    - In a typed launch plan (every agent-initiated launch approved through
      LaunchApproval, and user launches containing `%if`/`%proc`), an agent unit's
      `%wait` with an explicit `target=` is not admitted by the coordinator. It stays in
      that unit's prompt, and the launched agent binds it when the unit dispatches.
    - `target=turn` cannot name another unit of the same plan.
    - `unit=`/`proc=` cannot share a `%wait` with `target=`.
    - `%proc` and `%if` units still reject `target=turn`.
    - Add the two new diagnostic codes to the list of codes.
  - In `docs/reference/macros/directive-syntax.md`, add a one-sentence pointer in the
    typed-launch-units section, next to the `%wait(agent=<name>)` paragraph.

No memory change is needed. The `macros.md` text about `%wait` `target=` stays true.

Commit both repos in this turn's declaration. When a declaration commits both repos, the
host commits sase-core first and moves `sase-core-revision.txt` itself
(`docs/contribute/rust-backend.md`). Do not run `just ratchet-core-revision` by hand.

## 2. Retire the `wait_targets_with_descriptions` epic symbol

The Python facade `wait_targets_with_descriptions()` in
`src/sase/core/wait_target_facade.py` has no non-test consumer. The `%wait target=`
suggestions use the Rust `WAIT_TARGETS_WITH_DESCRIPTIONS` constant directly, in the
editor and completion paths. The CLI `-T` choices are a plain tuple. Following the
Symvision hierarchy ("delete it if genuinely dead, and its tests"):

- Delete `wait_targets_with_descriptions` and its `__all__` entry from
  `src/sase/core/wait_target_facade.py`.
- Delete `test_wait_targets_with_descriptions_matches_catalog` from
  `tests/test_core_wait_target_facade.py`.
- Remove `--epic-symbol 'sase-1q4(wait_targets_with_descriptions)' \` from the
  `_lint-symvision` recipe in the `Justfile`. Rewrite the sase-1q4 lines in the comment
  block above it to say the land deleted the unused Python facade, so no sase-1q4 rows
  remain. Never put a comment line inside the continued command.
- Leave the sase-core pyo3 binding alone; it has its own core tests.

## 3. Verify

- `sase tool run check` in sase-core.
- `just fmt` and then `sase tool run check` in sase. Never run `just check-full`.
- `sase bead epic-symbols sase-1q4` lists nothing.
- If `sase tool run check` reports NEW/UNKNOWN failures in files this tale touched, fix
  them. KNOWN/FLAKY items are not yours.

## 4. Close out epic sase-1q4 (final step, same turn)

Do this after the code above is complete and verified, in the same turn. Do not wait for
this tale's own commit, push, or CI.

1. Run `sase bead epic-symbols sase-1q4`. If any entry remains, resolve it now: wire it
   up, privatize it, add a non-test pragma, or delete it, per the Symvision
   epic-whitelist policy. Re-key an entry only if a still-open later bead needs it; none
   is expected. Do not move on until the list is empty.
2. Close the epic. Pass `--note` with the verification text below, filling in the
   bracketed parts:

   ```bash
   sase bead close sase-1q4 --note "<verification>"
   ```

   The verification note must cover:
   - All 11 phases (sase-1q4.1 through sase-1q4.11) are closed and checked against the
     plan and the code at 4a4b6fede8:
     - Promotion-aware session seeds and the legacy heal (167af33530).
     - wait_target vocabulary, NamedEnum, and facade (core d7b00dd7; 271632e9d7,
       cb230adb5e).
     - keyword_only in core (18ad5fd6) and Python (9e6e2b329f).
     - Turn-pinned identity records (5ade7889b2).
     - `%wait target=` end to end (core 80f5af40; eed4ef925b).
     - `sase agent wait -T` (ef4bc2b34a).
     - turn= in coder options (1b01062d2e).
     - Fork history follows bound records (3fcb69fb3c).
     - `#fork(..., target=)` binding with history_only dedupe (dcee7f11e9).
     - TUI wait_turn_bindings (core dddcb0b7; 4a4b6fede8).
   - The `sase-core-revision.txt` pin covers every core commit.
   - The macros.md decision was honored.
   - The completion-test failures from epic note #1 were already fixed on master (139
     focused tests pass).
   - Integration: the 19 non-epic commits since 167af33530 are refactors, perf work, and
     test splits with no conflicts. eec20f6ad5 fixed sase-1q7, and e28334c45a fixed the
     pager teardown.
   - The landing fix: typed admission now forwards explicit-target `%wait`s to the
     runner (core [sha]; sase [summary]), and the dead `wait_targets_with_descriptions`
     facade was deleted.
   - Follow-up triage, as in note #2: sase-1qe, sase-1qf, and sase-1qg filed; +1 on
     sase-1gz and sase-15h; the rest declined with reasons; a DISCOVERED ISSUE was noted
     on sase-s6.
   - Verification: [sase tool run check results in both repos].

   Never use `--force` to make the close succeed. If the close is refused because of
   leftover `--epic-symbol` entries, finish that cleanup and close again.

3. Run `just symvision`, or `just _lint-symvision` if that recipe name is absent, and
   confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202610/fork_wait_session_turn_target.md` (the PLAN path that
   `sase bead read sase-1q4 -r "<why>"` prints; it currently says `status: wip`).
5. sase-1q4 has no `parent_bead`, so there is no parent to close.
