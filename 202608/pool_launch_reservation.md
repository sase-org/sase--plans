---
tier: tale
title: Reserve a load-balanced model-alias pool member at agent launch
goal:
  Every agent launched in one batch (an epic's phases, a chop, a swarm) reserves its own
  pool member at launch, so the recorded and displayed model honors pool membership,
  weights, and the persisted cursor instead of repeating one member across the batch.
size: medium
proposed_by: bbugyi200.athena.0cc
create_time: 2026-08-24 10:00:37
status: wip
---

# Plan

## Background: the reported symptom

When an epic is launched, its phase agents appear to ignore the configured model-alias
pool: every phase that shares a size alias shows the _same_ model, so the pool's
membership, its per-member weights, and the "which member was used last" cursor all look
like they were ignored.

The suspicion is **confirmed**, but the underlying rotation math is _not_ the culprit.

## Diagnosis

### What is actually broken

A pooled alias is resolved in two different places, with two different meanings:

1. `src/sase/axe/run_agent_directives.py::extract_directives_and_write_meta` runs once
   per launched agent, inside that agent's own runner process, during bootstrap. It
   calls `resolve_launch_selection(..., consume=False)` — a deliberately **non-consuming
   preview** — and writes the result into `agent_meta.json` as `model`, `llm_provider`,
   `reasoning_effort`, `model_alias`, `model_alias_trail`, and `model_alias_origin`.
2. `src/sase/xprompt/workflow_executor_steps_prompt.py` calls
   `resolve_launch_selection(..., consume=True)` immediately before the provider call.
   This is the only site that advances the machine-global cursor in
   `~/.sase/llm_lb.json`, and afterwards it reconciles `agent_meta.json` for anonymous
   workflows.

Between (1) and (2) the runner performs **two blocking waits** — see
`src/sase/axe/run_agent_runner.py::_run_agent`: `bootstrap_agent_run` →
`_wait_for_dependencies_phase` (`%w` dependencies and bead blockers) →
`_admit_and_launch` (`wait_for_runner_slot`, the global runner-concurrency queue) →
`launch_agent_run` → the workflow executor.

An epic launch renders one multi-prompt whose segments become N agents, launched roughly
one second apart. All N bootstrap and write their preview _before any of them reaches a
prompt step_. Because the preview never consumes, all N read the same cursor position
and record the same pool member. Every ACE row, `sase agent list` row, and alias-history
row for those agents keeps that identical model for the entire — often multi-hour —
waiting window. Agents killed while waiting keep it forever.

### Evidence gathered while diagnosing

Read from `~/.sase/agent_artifact_index.sqlite` and `~/.sase/llm_lb.json` on the host
that reported the problem:

- Epic clan `sase-sq` launched nine agents between `20260824093346` and
  `20260824093354`. The four `@large` phases all recorded `claude/opus`; the four
  `@medium` phases all recorded `grok/grok-4.6`. At that moment the persisted cursors
  were `large: 0` and `medium: 0`, i.e. exactly the pre-launch positions — every row was
  the same unconsumed peek.
- Epic clan `sase-sp` launched five `@medium` agents at `2026082409201x`; all five
  recorded `claude/sonnet`.
- Reproduced directly against the current tree: driving
  `extract_directives_and_write_meta` four times with `%model:@pool` against a
  `claude/sonnet | codex/gpt-5.5 | 2 grok/grok-4.6` pool records `grok/grok-4.6` four
  times and never creates a cursor entry for the pool at all.
- The rotation math itself is correct. Epic clan `sase-sn`'s `@medium` phases, which all
  ran to completion, landed on `codex/gpt-5.5`, `claude/sonnet`, `grok/grok-4.6` —
  exactly schedule positions 1, 2, 3 of the configured `codex | claude | 2 grok`
  weighted pool. Weighted-schedule generation, cursor persistence, fingerprinting, and
  the flock-guarded read/modify/write in `src/sase/llm_provider/load_balancing.py` are
  all sound and already covered by `tests/llm_provider/test_weighted_alias_pools.py` and
  `tests/llm_provider/test_load_balanced_alias_state.py`.

So: the _selection that eventually runs_ rotates correctly; the _selection recorded and
shown at launch_ does not, and for a batch launch it is systematically wrong for every
agent in the batch.

### Root cause

The pool cursor is consumed at provider-invocation time, but the model is _published_ at
launch time. Nothing reserves a member for an agent between those two moments, so a
batch of concurrent launches cannot be told apart.

## Design: reserve at launch, redeem at first invocation

Move the authoritative pool consumption to the launch bootstrap and let the first prompt
step redeem it. One launched agent consumes exactly one cursor slot, and the metadata it
publishes is the selection that will actually run.

The re-exec plumbing this needs already exists:
`src/sase/axe/run_agent_directive_metadata.py::preserved_agent_metadata` already carries
`model`, `llm_provider`, `reasoning_effort`, `model_alias`, `model_alias_trail`, and
`model_alias_origin` across a runner re-exec, and `extract_directives_and_write_meta`
already short-circuits resolution when that preserved metadata is present.

### Behavior after the change

- N agents launched in one batch take N successive positions in the pool's weighted
  schedule. Membership, weights, and the persisted cursor are all honored at the moment
  the user can first see them.
- `agent_meta.json` names the model that will actually run, from the instant the agent
  appears, including while it waits on `%w` dependencies or a runner slot.
- Long waits stay safe: a reservation whose target became unavailable is re-picked at
  redemption time.
- A pooled alias still advances the cursor exactly once per launched agent. Non-pooled
  targets (concrete models, `||` ordered fallback chains) are unaffected — they have no
  cursor.

## Implementation

### 1. Reserve during runner bootstrap

`src/sase/axe/run_agent_directives.py`

In the `else:` branch of `extract_directives_and_write_meta` (the branch that runs when
`preserved_metadata` carries no `model`/`llm_provider`), change
`resolve_launch_selection(directives, model_alias_overrides, consume=False)` to
`consume=True`, and rewrite the surrounding comment: this is no longer a preview, it is
the agent's authoritative reservation.

Record the reservation alongside the existing fields so the redeemer can validate it:

- `model_alias_reservation`: a small dict with `alias` (the pool-owning alias whose
  cursor advanced, or `null` when the resolution consumed no cursor), `target`
  (`"<provider>/<model>"`), `effort`, `alias_trail`, `alias_origin`, and
  `redeemed: false`.

Add `model_alias_reservation` to `preserved_agent_metadata`'s preserved keys in
`src/sase/axe/run_agent_directive_metadata.py` (it is a dict, so extend the preservation
loop rather than the string-only loop), and thread it through `AgentMetadataInputs` and
`build_agent_metadata` the same way `model_alias_trail` is threaded today.

The existing preserved-metadata short circuit already prevents a second reservation when
the runner re-execs after its wait (see
`src/sase/axe/run_agent_runner_refresh.py::refresh_runner_code_after_wait`); add a test
that pins this, because it is now the guard against double consumption.

### 2. Expose reservation lookup from the launch-selection seam

`src/sase/llm_provider/launch_selection.py`

Add two helpers next to `LaunchSelection` so no caller has to hand-roll the redemption
rules:

- `reservation_from_launch_selection(selection, *, alias) -> dict` — build the persisted
  shape.
- `launch_selection_from_reservation(reservation, *, directives, provider_disables) -> LaunchSelection | None`
  — return the reserved selection when it is still usable, and `None` when it is not. It
  returns `None` when any of these hold:
  - the reservation is missing, malformed, or already `redeemed`;
  - the step's effective `%model` no longer resolves to the reserved alias (a step-level
    `%model` override, or `model_alias_overrides` that changed the chain);
  - the reserved target is no longer available under the current provider-disable
    snapshot, per `resolved_target_is_available` / `resolved_target_availability` in
    `src/sase/llm_provider/model_alias_resolution_types.py`.

The pool-selection primitives in `src/sase/llm_provider/load_balancing.py` need no
behavior change; keep the change confined to the launch seam.

### 3. Redeem at the first prompt step

`src/sase/xprompt/workflow_executor_steps_prompt.py`

Where the step currently calls `resolve_launch_selection(..., consume=True)`:

1. Read `agent_meta.json` from `self.artifacts_dir` and try
   `launch_selection_from_reservation(...)`.
2. On a hit: mark the reservation `redeemed: true` in `agent_meta.json` **before**
   invoking the provider (so a crash or re-exec mid-step cannot redeem twice), and use
   the reserved `LaunchSelection` for the step marker, the provider call, and the saved
   chat history.
3. On a miss, including the "reserved member is no longer available" case: fall back to
   today's `resolve_launch_selection(..., consume=True)`. An unavailable reservation is
   a miss, and the fallback re-pick is what keeps a stale reservation from pinning an
   agent to a usage-limited provider after a long wait.
4. Subsequent prompt steps in a multi-step workflow are unchanged: the reservation is
   already redeemed, so they resolve fresh with `consume=True`, one consumption per real
   invocation.

The `if self.workflow.is_anonymous_workflow:` reconciliation block stays, but becomes a
no-op in the common case because bootstrap already wrote the same values. Keep it: it is
what repairs the row when redemption missed and the fallback picked a different member.

### 4. Update the module contracts

Rewrite the stale narrative in three docstrings so the invariant is stated once and
consistently:

- `src/sase/llm_provider/launch_selection.py` module docstring — currently claims
  `resolve_launch_selection` is "the single place that resolution happens with
  `consume=True`". Replace with: the runner bootstrap reserves, the first prompt step
  redeems, later steps resolve fresh.
- `src/sase/llm_provider/load_balancing.py` module docstring — document that a cursor
  slot is reserved at launch and that an agent killed before its first invocation leaves
  that slot spent. This is an accepted fairness blip, not a correctness bug: cursor
  accounting is already documented as best-effort.
- `src/sase/axe/run_agent_directives.py` — replace the "non-consuming preview ... will
  be reconciled afterward" comment with the reservation contract.

## Tests

Existing tests that encode the old contract and must be updated, not deleted:

- `tests/test_pooled_alias_single_consumption.py` — its whole premise is "the preview
  must not consume; the prompt step must". Re-point it at the new invariant: bootstrap
  consumes exactly once, the prompt step redeems without consuming again, and one
  composed launch advances the cursor by exactly one.
- `tests/test_reasoning_effort_metadata_persistence.py` and
  `tests/test_reasoning_effort_metadata_enrichment.py` — assert on the metadata the
  bootstrap writes for pooled aliases; confirm they still hold and adjust cursor
  expectations where they assume a non-consuming bootstrap.
- `tests/test_launch_default_indicator_pool_rotation.py` — the ACE launch-default
  indicator reads the same rotation state; confirm it still tracks.

New coverage:

- **Batch launch rotates.** Drive `extract_directives_and_write_meta` N times against a
  weighted pool (reuse
  `tests/llm_provider/_load_balanced_alias_helpers.py::configure_pool`) and assert the N
  recorded `model` values follow the weighted schedule, in order, and that the cursor
  advanced exactly N times. This is the regression test for the reported bug; it fails
  on today's code.
- **Redemption does not double-consume.** Bootstrap, then run the anonymous workflow's
  prompt step, and assert the cursor advanced once in total and the step marker, the
  `invoke_agent` kwargs, and `agent_meta.json` all name the same model.
- **Re-exec after the wait does not re-reserve.** Bootstrap, then bootstrap again with
  the first run's `agent_meta.json` in place, and assert the cursor did not move and the
  reservation is unchanged.
- **Unavailable reservation falls back.** Bootstrap, then make the reserved target
  unavailable via the provider-disable snapshot, then run the prompt step; assert the
  step picked an available member, that `agent_meta.json` was reconciled to it, and that
  no crash or hang occurred.
- **Non-pooled aliases are untouched.** A concrete `%model` and a `||` ordered fallback
  chain write no reservation and touch no cursor.

## Verification

- `just check` for the scoped lane during development.
- `just check-full` before landing, through `/sase_monitor` (it outruns a single turn):
  this change touches the launch seam and the workflow executor, so the scoped selection
  is not trustworthy on its own.
- Manual confirmation on a real batch: launch an epic with three or more phases that
  share one size alias and confirm the ACE rows show three different pool members
  immediately, matching the alias's weighted schedule from the current
  `~/.sase/llm_lb.json` cursor.

## Out of scope

- Any change to the weighted-schedule algorithm, cursor persistence format,
  fingerprinting, or the `~/.sase/llm_lb.json` state file — these were verified correct
  during diagnosis.
- Releasing a reservation when an agent is killed before its first invocation. The spent
  slot is a one-position fairness blip; add it only if it shows up in practice.
- Unrelated observation worth a separate task bead, not this plan: user config under
  `llm_provider.model_aliases.builtin` is filtered to `BUILTIN_MODEL_ALIAS_NAMES` in
  `src/sase/llm_provider/model_alias_config.py::get_builtin_model_aliases`, so retired
  names such as `medium_worker` are dropped **silently**. A host config still carrying
  them gets no warning that those entries do nothing.

## Feature flag

None. Per `sase/memory/sase_flags.md`, a flag is for behavior that reaches users before
it is ready, or for keeping a deprecated branch reachable while callers migrate. This is
a correctness fix that should land unconditional, and the old "publish an unreserved
peek" branch must not stay reachable.
