---
tier: tale
title: Seal the effective finalizer config for the whole turn
goal:
  A host config or code change that lands while an agent is mid-turn no longer fails
  that turn's finalizers; the turn runs the configuration that was sealed at launch and
  records any live drift as a non-fatal diagnostic.
size: medium
proposed_by: bbugyi200.athena.0cr
create_time: 2026-08-24 14:06:49
status: wip
---

# Seal the effective finalizer config so a mid-turn host change cannot kill the turn

## Symptom

Agent turns die at the very end with:

```
WorkflowExecutionError: Step 'main' failed: Error: live refusal policy for 'commit'
drifted from the sealed plan
```

The agent has already done all of its work. The turn is destroyed at finalizer time, no
commit happens, and the workspace is left dirty with nothing disclosed. On the reporting
host this killed five agents in one window, including two epic agents that had been
running for 3h16m and 4h21m.

## Root cause

`src/sase/finalizers/plan.py` seals a finalizer plan at the _start_ of a turn and then
re-derives the effective configuration from live, mutable sources at the _end_ of the
same turn, then hard-fails on any difference.

1. `resolve_and_persist_finalizer_plan()` (`src/sase/finalizers/plan.py:85`) runs from
   inside `invoke_llm` (`src/sase/llm_provider/_invoke.py:154`) before the model is
   called. It calls `load_finalizer_config()`, resolves the plan through the Rust core,
   and writes `finalizer_plan.json` plus the host-owned `finalizer_plan.authority.json`.
2. `authenticate_resolved_finalizer_plan()` (`src/sase/finalizers/plan.py:126`) runs at
   turn end - repeatedly from the controller
   (`src/sase/finalizers/controller.py:77,108,136,247`) and once per `sase final submit`
   (`src/sase/finalizers/declaration.py:339`). It calls `load_finalizer_config()`
   **again**, and `_compare_live_configuration()` (`src/sase/finalizers/plan.py:207`)
   raises `FinalizerPlanIntegrityError` if any of `provider_ref`, `max_attempts`,
   `refusal`, `after`, `config_digest`, or `provenance_id` differs from the sealed
   entry.
3. `load_finalizer_config()` -> `load_config_layers()` (`src/sase/config/core.py:416`)
   re-reads the layer files from disk on every call, and the bundled defaults layer is
   `src/sase/default_config.yml` of the _installed_ package.

The host `sase` install is editable against a live working checkout of this repo (see
`sase version`, "Code directory"). So `src/sase/default_config.yml` is not a frozen
resource - it is a file that changes under running agents every time that checkout
fast-forwards. The same is true of `~/.config/sase/sase.yml`, the selected overlays, and
the project-local `sase/sase.yml`.

The concrete trigger for the reported failures: commit `2b046b174` ("feat(finalizers):
defer commit on refusal instead of failing the turn") changed

```yaml
finalizers:
  instances:
    commit:
      refusal: fail # ->  refusal: defer
```

and the host checkout fast-forwarded past it at 13:45:25. Every agent that had sealed
its plan before 13:45:25 and reached its finalizer after 13:45:25 sealed
`refusal: fail`, re-read `refusal: defer`, and was failed by
`_compare_live_configuration`. Agents launched after 13:45:25 seal `defer` and pass.

Nothing was tampered with. The check cannot tell a legitimate host upgrade from an
attack, so it treats a routine `git merge` as tampering and discards the turn. The
exposure window is the entire turn, so the longer and more valuable the agent run, the
more likely it is to be destroyed.

This is the second instance of this failure family on this host. The first is recorded
on epic bead `sase-sp`: a `FINALIZER_WIRE_SCHEMA_VERSION` bump in the sibling Rust core
made every launch on the host fail with
`unsupported finalizer plan input schema_version 1; expected 2`, for the same underlying
reason - the finalizer stack reads live host state that agents in flight do not control.

### Why the check exists

`_compare_live_configuration` is not gratuitous. The turn-end path really does consume
live config values rather than the sealed plan:

- `src/sase/finalizers/controller.py:138` calls `load_finalizer_config()` and uses
  `instance.max_attempts` (`controller.py:163,206`) and `instance.refusal`
  (`controller.py:189`, the deferred-and-non-failing decision).
- `src/sase/finalizers/executor.py:131` (`validate_external_declaration_payload`, the
  `sase final submit` path, a different process) calls `load_finalizer_config()` and
  resolves the instance from it.
- `src/sase/finalizers/executor_command.py:49,59` and
  `src/sase/finalizers/executor_plugin.py:58,95` consume `instance.config` and
  `instance.max_attempts`.

The sealed plan carries only a `config_digest` for each instance, never the config body,
so the executor genuinely cannot run from the plan alone today. The comparison is a
guard around that gap. The fix is to close the gap rather than to keep guarding it.

## Approach

Seal the effective configuration, not just its digest, and make the turn hermetic.

1. **Seal the config bodies** into the host-owned authority artifact at plan-resolve
   time.
2. **Execute from the sealed snapshot** everywhere in the turn-end path, so live config
   is never consulted for a sealed field.
3. **Demote live drift to a non-fatal diagnostic**, since it can no longer change what
   executes. Keep hard failure only for genuine authentication failures.

The sealed snapshot is self-verifying against the existing digest chain, so this needs
**no Rust wire change**: `config_digest` on each plan entry is already
`finalizer_json_digest(config)`, and the plan itself is already authenticated against
`SASE_FINALIZER_PLAN_DIGEST`. Rebuilding an instance from the snapshot and checking that
its `to_wire()` matches the authenticated entry proves the snapshot was not edited. This
keeps the change inside this repo without reimplementing core logic, per the Rust core
backend boundary.

## Implementation

### 1. Serialize and deserialize a configured instance

`src/sase/finalizers/config.py`

- Add round-trip helpers for `ConfiguredFinalizerInstance` and `FinalizerConfig`:
  `configured_instance_to_json(instance) -> dict` and
  `configured_instance_from_json(payload) -> ConfiguredFinalizerInstance`, plus
  `finalizer_config_to_json(config)` / `finalizer_config_from_json(payload)`.
- Cover every field the turn-end path reads: `instance_id`, `provider_ref`, `after`,
  `max_attempts`, `refusal`, `config`, and `provenance` (each `FinalizerFieldProvenance`
  as `{"layer": ..., "path": ...}`). Carry `defaults`, `required`, and the config-level
  `provenance` too.
- Deserialization must be strict about shape and must not silently default a missing
  field to `"fail"` / `1`; a malformed snapshot is a tamper signal, not a fallback.

### 2. Persist the snapshot into the authority artifact only

`src/sase/finalizers/plan.py`

- Add `FINALIZER_CONFIG_SNAPSHOT_KEY = "config_snapshot"` and a
  `FINALIZER_CONFIG_SNAPSHOT_SCHEMA_VERSION = 1`.
- `_persist_plan()` gains the resolved `FinalizerConfig` and writes
  `{"schema_version": 1, "config": finalizer_config_to_json(config)}` under that key
  into `finalizer_plan.authority.json` **only**. Do not add it to the model-visible
  `finalizer_plan.json`: config bodies can carry commands and env allowlists, and the
  model has no reason to read or edit them.
- `_authenticate_resolved_finalizer_plan()` compares the two artifacts with
  `finalizer_wire_to_json_dict(authority) != finalizer_wire_to_json_dict(visible)`,
  which only compares the `plan` sub-object, so an authority-only key does not break
  that check. Confirm this while implementing and add a test that pins it.

### 3. Return the sealed config, and verify it

`src/sase/finalizers/plan.py`

- Introduce a frozen `AuthenticatedFinalizerPlan` dataclass with
  `plan: FinalizerPlanWire`, `config: FinalizerConfig`, and
  `drift: tuple[FinalizerConfigDiagnostic, ...]`.
- Add
  `authenticate_resolved_finalizer_plan_full(artifacts_dir, *, config=None) -> AuthenticatedFinalizerPlan`.
  Keep `authenticate_resolved_finalizer_plan()` returning `FinalizerPlanWire` as a thin
  wrapper so existing call sites and tests keep working.
- Verification of the snapshot, all fatal:
  - every entry in the authenticated plan has an instance in the snapshot;
  - for each entry, the rebuilt instance's `to_wire()` matches the entry on
    `provider_ref`, `after`, `policy.max_attempts`, `policy.refusal`, `config_digest`,
    and `provenance_id`. Because the plan is digest-authenticated first, this is what
    proves the snapshot authentic. Word these failures with both values, e.g.
    `sealed config for 'commit' does not match the authenticated plan: refusal sealed=fail snapshot=defer`.
- Replace `_compare_live_configuration()` with
  `_diagnose_live_configuration(plan, live_config) -> tuple[FinalizerConfigDiagnostic, ...]`.
  Same field comparisons, but it returns `severity="warning"`,
  `code="plan_config_drift"` diagnostics instead of raising. Each message must name the
  instance, the field, the sealed value, the live value, and the live layer/path from
  `instance.provenance` so the drift is diagnosable from the artifact alone - for
  example:
  `finalizer 'commit' refusal drifted after the plan was sealed: sealed=fail live=defer (default:finalizers.instances.commit.refusal); this turn ran the sealed value`.
- Loading live config for the diagnostic must never fail the turn: wrap
  `load_finalizer_config()` and downgrade any exception to a single
  `plan_config_unreadable` warning.

### 4. Compatibility with turns sealed by an older sase

An agent that is mid-turn when this lands has an authority artifact with no
`config_snapshot`. Handle it explicitly:

- If the key is absent, fall back to `load_finalizer_config()` for the returned config,
  emit a `plan_config_snapshot_missing` **warning**, and **do not** compare live config
  against the sealed plan at all. Failing those turns is exactly the bug being fixed.
- If the key is present but malformed or fails the digest check above, that is fatal.

### 5. Execute from the sealed config

- `src/sase/finalizers/controller.py`: replace `config = load_finalizer_config()`
  (`controller.py:138`) with the sealed config from the authenticated result. Remove the
  now-unused `load_finalizer_config` import if nothing else needs it. The controller
  re-authenticates several times per cycle; take the sealed config from each of those
  results rather than caching it once, so the code keeps reading as "one authenticated
  source per step".
- `src/sase/finalizers/executor.py`: `validate_external_declaration_payload()` must use
  the sealed config for `artifacts_dir` (`SASE_ARTIFACTS_DIR`) instead of
  `load_finalizer_config()`. This runs in the `sase final submit` process, which is the
  other place the old check fired.
- `src/sase/finalizers/declaration.py:339`: `load_finalizer_plan()` should use the full
  authenticated result and expose the sealed config to its callers rather than letting
  them re-read live config.

### 6. Surface the drift

- `src/sase/finalizers/artifacts.py` / `_write_plan_integrity_failure` neighbours: carry
  the drift diagnostics into `finalizer_result.json` `diagnostics` (severity `warning`)
  so a drifted-but-successful turn is auditable.
- Project the same list into `agent_meta.json` through
  `ResolvedFinalizerPlan.agent_meta_projection()`'s counterpart on the authenticated
  side, using the existing `update_meta_field` helper, so ACE can show it. Keep this
  additive - no ACE work is required for the fix to be correct.

## Tests

`tests/test_finalizers_plan_integrity.py`

- `test_live_configuration_drift_fails_before_execution` (line 246) encodes the old
  behaviour and must be **rewritten**, not deleted. Turn it into
  `test_live_configuration_drift_runs_the_sealed_policy`: seal a plan, mutate the live
  config the same way it does today, assert the turn **succeeds**, assert the executed
  behaviour matches the sealed value, and assert a `plan_config_drift` warning naming
  the field, both values, and the layer landed in `finalizer_result.json`.
- New: `test_sealed_config_snapshot_survives_a_refusal_flip` - the exact reported
  regression. Seal with `refusal: fail`, flip live config to `refusal: defer`, run the
  controller, assert no `FinalizerPlanIntegrityError` and that the sealed `fail` is what
  the controller used at `controller.py:189`.
- New: `test_tampered_config_snapshot_is_fatal` - edit `config_snapshot` in
  `finalizer_plan.authority.json` (change `refusal`, and separately a `config` body
  value) and assert `plan_integrity_failed` with a message naming the field and both
  values.
- New: `test_missing_config_snapshot_falls_back_without_failing` - delete the key from
  the authority artifact, mutate live config, assert success plus a
  `plan_config_snapshot_missing` warning.
- New: `test_config_snapshot_is_not_model_visible` - assert `finalizer_plan.json` has no
  `config_snapshot` key and that the visible-vs-authority comparison still passes.
- Extend the `test_plan_artifact_mutations_fail_before_execution` mutator table
  (line 185) with a `config_snapshot` mutator so snapshot tampering is covered by the
  same matrix as plan tampering.

`tests/test_finalizer_declaration_channel.py`

- Add a cross-process style case: seal a plan, mutate live config, run the
  `sase final submit` path, and assert it validates against the sealed instance instead
  of raising `plan_integrity_failed`.

## Verification

```bash
just install
just check
```

Then, because this touches the finalizer stack that every agent turn runs through, hand
the exhaustive lane to a monitor rather than running it inline:

```bash
sase monitor start --command 'just check-full' \
  --start-status TESTING --stop-status TESTED \
  --next 'Report just check-full results for the finalizer sealed-config change'
```

Manual reproduction of the original failure, which must now pass:

1. Seal a plan while the effective config has
   `finalizers.instances.commit.refusal: fail`.
2. Change the effective config to `refusal: defer` before the turn ends.
3. Run the finalizer controller and `sase final submit`.

Before: `live refusal policy for 'commit' drifted from the sealed plan`, turn failed.
After: the turn completes on the sealed `fail` policy and records one
`plan_config_drift` warning.

## Non-goals

- **Do not** change what happens to the workspace when a finalizer genuinely fails.
  Stranded dirty trees after a real failure are a separate concern.
- **Do not** seal provider/plugin resolution. `src/sase/finalizers/providers.py:397`
  reads `load_merged_config()` and `_resolve_provider` reads live entry points at
  execution time, so a mid-turn plugin change is a related but distinct exposure. Note
  it as follow-up; do not widen this change to cover it.
- **Do not** attempt the Rust wire version-skew problem already recorded on epic bead
  `sase-sp`. It is the same failure family but a different mechanism (a released-crate
  floor versus a checkout-built binding) and belongs with that epic.
- **Do not** add a Rust core wire type for the snapshot. The existing `config_digest`
  already authenticates it.
- **Do not** weaken the fatal paths that stay fatal: missing or malformed plan
  artifacts, digest mismatch, model-visible artifact diverging from host authority, a
  sealed entry with no snapshot instance, or a snapshot that fails its digest check.
