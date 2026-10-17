---
tier: tale
title: Stop gate follow-ups from adopting a stale successor
goal:
  An answered sudo or custom gate created by a plan-chain coder always launches its
  follow-up agent.
size: medium
decisions:
  rust_guard:
    ask:
      Should sase-core also refuse to adopt on a first-pass settle that has no recorded
      launch attempt?
    default: true
    why:
      Puts the invariant in the Rust decision layer, guarding against future evidence
      bugs
    answer: true
proposed_by: bbugyi200.apollo.8j
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-17 09:11:10
status: wip
---

# Plan: Stop gate follow-ups from adopting a stale session member as their successor

## Problem

On 2026-10-17, agent session `8f` (project `gh_bobs-org__bob-cli`) ran an approved plan.
Its coder `8f--code` created a sudo gate ("Gate A") and correctly ended its turn. The
gate turn `8f--gate-0` ran the approved root commands and settled at 08:48 EDT. It
recorded `gate_followup_outcome = launched` and `gate_followup_agent = 8f--code`, yet
**no agent was launched**. The plan's Step 2 onward never ran, and the user had to fork
the session by hand.

The gate "adopted" its own already-finished creator as the follow-up successor. This
happens every time a non-plan gate (`sudo`, `custom`, and any other kind without
`next.suffix`) is created by a plan-chain coder (`<session>--code`). On this machine,
this was the first gate in that configuration out of 349 gate turns, and it failed.

### Root cause chain

1. `src/sase/gate_turn/handoff_launch.py::launch_or_record_followup` collects successor
   evidence and asks the Rust classifier for a decision before launching anything.
2. `src/sase/gate_turn/handoff.py::collect_successor_evidence` uses
   `expected_suffix or PLAN_CHAIN_CODER_SUFFIX`. Sudo and custom gate policies carry no
   suffix (`gate_next_suffix` is absent). Their real successor is allocated from the
   `--@` template (`spawn_turn_agent_session_successor` passes `suffix or "@"`), which
   yields numbered names such as `8f--0`. The `--code` fallback therefore matched the
   creator `8f--code`, which had completed when it handed off. Candidates are also not
   anchored to the gate in any way: there is no check that the record started after the
   gate member, and the creator is not excluded.
3. sase-core `crates/sase_core/src/gate_followup/classify.rs::decide_existing_successor`
   runs first. Seeing a completed match with no receipt, it returns `RECOVERY_ADOPT`,
   even on a first-pass `settle` where no launch attempt exists. Python's adopt branch
   then persists `launched` with `live=False` and returns without spawning.

The gate's metadata confirms the adopt branch. It has no
`gate_followup_attempt_owner_pid`, which only the real launch path writes, through its
live `preparing` attempt.

The reclaim diagnose path (`src/sase/gate_turn/reclaim.py`, around line 509) uses the
same evidence collector, so it would also accept the same false adoption.

Plan gates are unaffected today for two reasons. They carry an explicit
`gate_next_suffix: --code`, and their coder does not exist yet when they settle. Older
plan gates that lack `gate_next_suffix` still launched `--code` coders (133 such gates
exist), so plan kinds must keep the `--code` default.

## Changes

### 1. Anchor successor evidence to the gate (sase, Python)

In `src/sase/gate_turn/handoff.py`, extend `collect_successor_evidence` with these
keyword parameters:

- `gate_kind: str | None`
- `gate_timestamp: str | None`: the gate turn member's own artifact-directory timestamp
- `exclude_agents`: the gate creator and the gate member's own name

Then:

- **Exclusion:** drop any record whose name is in `exclude_agents`. A gate's creator can
  never be its own successor.
- **Time anchor:** when both values are 14-digit `YYYYmmddHHMMSS` strings, drop records
  whose `record.timestamp` is not strictly greater than `gate_timestamp`. A successor is
  only spawned after the gate member's directory exists, and launch timestamps are
  globally reserved, so the comparison is exact. If either timestamp is missing or
  malformed, skip this filter but keep the exclusion.
- **Suffix resolution:** replace the unconditional `--code` fallback with a resolver:
  - An explicit literal suffix (for example `--code`) is used as-is.
  - A suffix template containing `@` matches a final name segment where `@` becomes one
    or more digits. This mirrors `allocate_agent_name_template` in
    `src/sase/agent/names/_templates.py`; reuse a helper there if one fits rather than
    writing ad hoc regex.
  - No suffix: use `PLAN_CHAIN_CODER_SUFFIX` only when `gate_kind` is `plan` or
    `epic_plan`. Otherwise use the `--@` template, which is exactly what
    `spawn_turn_agent_session_successor` allocates.
  - Match against the name's final `--` segment, so gate members such as `--gate-0`
    never match the numbered template.
  - Keep the `expected_suffix` evidence field populated with the resolved pattern, for
    diagnostics.
- Apply recorded-agent narrowing (`recorded_agent`) after the anchor filters. A bogus
  receipt naming a pre-gate member must not resurface as a running or completed
  `attached_agent`.

Update both callers:

- `src/sase/gate_turn/handoff_launch.py::launch_or_record_followup`: pass
  `meta.get("gate_kind")`, `Path(artifacts_dir).name`, and
  `{meta.get("gate_creator_agent"), meta.get("name")}` (drop the empty values).
- `src/sase/gate_turn/reclaim.py` (the diagnose block that builds `meta` from `live`):
  pass `live.kind`, `live.timestamp`, and
  `{live.creator_agent, live.member_agent_name}`.

### 2. Refuse first-pass adoption without an attempt (sase-core, Rust)

> [!decision] rust_guard Open the linked repo with `sase repo open sase-core` and work
> in the printed path. Read its `AGENTS.md` first.
>
> In `crates/sase_core/src/gate_followup/classify.rs`, make `decide_ambiguous` and
> `decide_existing_successor` ignore successor evidence when all of these hold:
>
> - `mode == settle`
> - `!already_settled`
> - `matching_attempt(request).is_none()`
> - no receipt is present: `followup_agent` is blank and `followup_outcome` is not
>   `launched` or `launched_degraded`
>
> Python persists a `preparing`/`launching` attempt before any spawn, so a first-pass
> settle with no attempt cannot have produced a successor. Classification then falls
> through to the normal launch-allowed path.
>
> Add tests in `crates/sase_core/src/gate_followup/tests.rs`:
>
> - First-pass settle with completed or running evidence and no attempt: verdict is
>   `RECOVERY_RESUME` with `launch_allowed`. This is the incident replay.
> - First-pass settle with ambiguous evidence and no attempt: launch is still allowed.
> - Settle with a stale (not live) `launching` attempt and running evidence: still
>   `RECOVERY_ADOPT`, because crash recovery must keep working.
> - The existing diagnose-mode legacy adoption tests stay unchanged and passing.
>
> No wire or binding shape changes. Committing both repos in the same final declaration
> moves `sase-core-revision.txt` automatically.

> [!decision] rust_guard = no Skip section 2. The Python anchor alone fixes the
> incident.

### 3. Tests (sase, Python)

Add `tests/gate_turn/test_successor_evidence.py`. Build on-disk records and inject them
through `agent_session_records`; reuse the `record_from_disk` and `_coder_successor`
style fixtures from `tests/gate_turn/test_reclaim_reconcile.py`, and lift shared bits
into a helper module if they need it. Cover these cases:

1. **Incident replay.** Session `8f` has these members:
   - `8f--plan`
   - `8f--gate`, a plan gate
   - `8f--code`, done, with an earlier timestamp
   - `8f--gate-0`, a sudo gate member with creator `8f--code` and no suffix

   Expect `running`/`completed` false and not ambiguous. Also expect that
   `classify_gate_handoff(meta, mode="settle", already_settled=False, followup_requested=True, successor_evidence=...)`
   returns `launch_allowed` true.

2. A real successor `8f--0`, started after the gate and running, is reported as
   `attached_agent == "8f--0"`.
3. A plan gate with explicit `--code` and a coder started after the gate is still
   adopted (existing behavior).
4. A legacy plan gate with no suffix still matches a post-gate `--code` coder.
5. A `--code` member from an earlier cycle in the same session is not matched by a later
   plan gate.
6. Later `--gate-1` and `--mon` members never match the numbered template.
7. A malformed `gate_timestamp` still excludes the creator.

Add a settlement-level regression in `tests/gate_turn/test_settlement_followup.py` or
`tests/gate_turn/test_followup_launch.py`:

- Settle an answered gate (kind `sudo` or `custom`, no `next.suffix`) whose creator is a
  completed `lane--code` member.
- Assert that the spawn function was called, and that `gate_followup_agent` names the
  newly allocated member, not the creator.
- Reuse `make_gate_turn_member` from `tests/gate_turn/_settlement_followup_helpers.py`.
  Add a `creator_agent` parameter if needed; it currently hard-codes `lane--0`. Use the
  same spawn stubs as `test_tale_approve_commit_settlement_launches_coder_followup`.

If any existing fixture places a "successor" at or before its gate's timestamp, fix the
fixture rather than weakening the anchor. Check the gate and successor timestamps in
`test_reclaim_reconcile.py` and `test_answered_handoff_resume.py`.

## Out of scope

- **Repairing the existing `8f--gate-0` receipt.** The user already forked `8f.f0` to
  finish the incident work. Rewriting the receipt or making `--resume` relaunch Step 2
  would race that fork and start a duplicate root-level incident agent. After this fix,
  its stale `launched → 8f--code` receipt is harmless history.
- **Docs.** `docs/reference/automation/notification-gates.md` already promises that
  approving a sudo request "launches the request's `next.prompt` successor". This fix
  makes the code keep that promise, so no wording changes.

## Verification

- sase: run `just check` through `sase tool run`, following the repo's lint/test memory.
  Make sure the new and updated `tests/gate_turn/` modules pass.
- sase-core (only if `rust_guard` is accepted): run the repo's standard check recipe. At
  minimum, `cargo test -p sase_core gate_followup` must pass.
