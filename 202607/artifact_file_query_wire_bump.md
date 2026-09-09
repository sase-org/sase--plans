---
tier: tale
title: Close the artifact-file query wire drift that breaks `sase artifact list`
goal:
  Every artifact-file query surface works again against a core at the declared
  `sase-core-rs` floor, and a named test fails whenever a Python wire constant drifts
  from the binding it mirrors.
create_time: 2026-09-09 19:53:00
status: wip
---

# Plan: Unbreak `sase artifact list` by closing the artifact-file query wire drift

## Problem

Every artifact-file query surface is hard-broken on any environment carrying a published
`sase-core-rs` at or above the version `pyproject.toml` already requires:

```
❯ sase artifact list
Error: sase artifact list failed: sase_core_rs artifact-file query wire is stale: expected 2, got 3
```

`src/sase/core/artifact_file_query_facade.py:18` pins
`ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION = 2`, and `_require_query_schema()` (line 62)
asserts equality with the binding before every query. The installed core reports 3, so
the guard raises and no query ever runs.

The failure is not confined to `sase artifact list`. `query_artifact_files()` is the
single entry point for all of:

- `src/sase/artifact_cli/listing.py:38` — `sase artifact list`
- `src/sase/artifact_cli/references.py:172` — `sase artifact show` reference resolution
- `src/sase/artifact_ref_prompt.py:293` — `@`-reference expansion at agent launch
- `src/sase/ace/tui/widgets/artifacts/files_data.py:89` — the ACE Artifacts Files tab
- `src/sase/ace/tui/widgets/_artifact_ref_completion_catalog.py:245` — ACE `@`-reference
  completion

## Root cause

The bump is split across two phases of the `sase-b9` epic
(`@plan:202607/artifact_consumption_ledger.md`), and the phases landed in an order that
leaves master broken in between:

- Phase 3 (bead `sase-b9.1`, **closed**) raised
  `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` from 2 to 3 in the Rust core to add the
  `unused_only` / `consumption_log_path` filters. The plan states the bump is
  deliberate: "the new filter fields are `#[serde(default)]`, so a stale wheel would
  silently ignore `unused_only` and return unfiltered rows. The Python facade asserts
  version equality, so the bump converts silent wrong answers into a loud startup
  failure."
- Phase 4 (bead `sase-b9.2`, **closed**, commit `3a0a92d84`) raised the `pyproject.toml`
  floor from `sase-core-rs>=0.13.0` to `>=0.13.1` — the release that carries wire 3.
- Phase 5 (bead `sase-b9.3`, **in progress**) is where the plan schedules "raise
  `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` to 3 to match Rust".

Phase 4 therefore made wire 3 mandatory for every install while the Python constant
still said 2. Every commit from `3a0a92d84` onward is broken for any user whose core
matches the declared floor.

The floor in `pyproject.toml` is already correct and needs no change; only the Python
constant is behind.

### Why the existing gates did not catch it

Three independent gates each had a reason to stay quiet, and all three are worth
understanding because only the third is worth fixing here:

1. **Local `just check` builds its own core.** `Justfile:55-59` (`_core-overrides-arg`)
   writes a `sase-core-rs` override whenever a `sase-core` checkout with a `Cargo.toml`
   and a `cargo` binary are present, and `Justfile:151` passes that override to
   `uv pip install`. A dev workspace whose `sase-core` checkout predates the wire bump
   therefore builds and keeps a wire-2 core, and the pyproject floor is ignored
   entirely. `Justfile:91` already warns about exactly this ("dev installs build from
   ... regardless"). This is intended behavior for local Rust development and is not in
   scope to change.
2. **CI never reported.** Every master CI run from `3a0a92d84` onward was cancelled by
   the next push in a rapid series (`gh run list --branch master --workflow CI`), so the
   lane that installs the published floor never published a verdict.
3. **No gate names the contract.** The only check that would have failed is
   `tests/test_artifact_file_query_facade.py::test_real_rust_query_matches_python_reader_for_v1_v2_fixture`,
   which exercises the real binding incidentally and reports as a fixture-parity
   mismatch rather than as version drift. Neither `tools/validate_sase_core_rs` nor
   `tools/check_sase_core_rs_bindings` compares wire-schema versions at all — both only
   check that binding _names_ exist. This gap is worth closing, and doing so is cheap.

A fourth signal confirms the diagnosis:
`tests/test_artifact_ref_preprocessing.py:174-177` already monkeypatches
`ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` to `3`. Phase 4 hit the drift in a test and
patched around it locally instead of moving the constant, which is precisely why the
breakage reached master green.

## Scope boundary — read before editing

Bead `sase-b9.3` is **running right now** and its plan section owns the `--unused`
feature: the new `src/sase/core/artifact_consumption_query.py` facade, the `unused_only`
parameter on `query_artifact_files()`, the `-u/--unused` CLI flag, and consumption rows
on `sase artifact show`.

This plan deliberately implements **none** of that. It moves one constant and adds the
missing guard, so `sase-b9.3` finds the constant already at 3 and adds only its filter
plumbing. Do not add `unused_only`, `consumption_log_path`, `-u/--unused`, or any
consumption reporting here.

## Changes

### 1. `src/sase/core/artifact_file_query_facade.py`

Raise `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` from `2` to `3` (line 18). No other
change: the two new Rust filter fields are `#[serde(default)]`, so the filter dict the
facade already sends deserializes unchanged, and the query row wire is identical between
wire 2 and wire 3 (verify by reading `crates/sase_core/src/artifact_file.rs` after
`sase repo open sase-core`; commit `1bd3670` touched only `ArtifactFileQueryFiltersWire`
and the filter chain, never `ArtifactFileRowWire`).

### 2. `tests/test_artifact_file_query_facade.py`

- Change the four `lambda: 2` fake binding versions (lines 49, 120, 177, 198) to `3`.
- In `test_query_rejects_stale_handshake`, keep the stale binding at `2` — a genuinely
  stale core is now one reporting the previous version — and update the assertion to
  `match="expected 3, got 2"`.
- Leave `test_query_rejects_incompatible_rows`' `_wire_row(schema_version=3)` case
  alone. That parametrization is about the _index row_ schema
  (`ARTIFACT_FILE_INDEX_SUPPORTED_SCHEMA_VERSIONS`, currently `{1, 2}`), which is a
  separate version line from the query wire and is unaffected by this change. Add a
  brief comment at that case so the two 3s are not later mistaken for the same number.

### 3. `tests/test_artifact_ref_preprocessing.py`

Delete the `monkeypatch.setattr(...ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION, 3)` block at
lines 174-177. With the constant at 3 it is dead, and leaving it in place would re-hide
the next drift in this exact test.

### 4. New `tests/test_rust_wire_schema_agreement.py`

The durable guard, and the reason this is not a one-line commit. A single parametrized
test that asserts each Python wire constant equals the version its `sase_core_rs`
binding reports, against the **installed** core with no monkeypatching:

- Declare an explicit table of `(module path, constant name, binding name)` rows.
  Explicit, not discovered by scanning `dir(sase_core_rs)`: the core legitimately ships
  bindings ahead of Python (`artifact_consumption_wire_schema_version`,
  `artifact_file_lifecycle_wire_schema_version`, and
  `artifact_ref_list_resolution_wire_schema_version` are all landed in core and awaited
  by in-flight epics), so requiring full mirroring would fail for correct states.
- The table's nine current rows, all of which agree once change 1 lands:

  | Python constant                                                                | binding                                         |
  | ------------------------------------------------------------------------------ | ----------------------------------------------- |
  | `sase.core.agent_cleanup_wire:AGENT_CLEANUP_WIRE_SCHEMA_VERSION`               | `agent_cleanup_wire_schema_version`             |
  | `sase.core.agent_launch_wire:AGENT_LAUNCH_WIRE_SCHEMA_VERSION`                 | `agent_launch_wire_schema_version`              |
  | `sase.core.artifact_file_query_facade:ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` | `artifact_file_query_wire_schema_version`       |
  | `sase.core.commit_footer_facade:COMMIT_FOOTER_WIRE_SCHEMA_VERSION`             | `commit_footer_wire_schema_version`             |
  | `sase.artifact_ref_models:ARTIFACT_REF_WIRE_SCHEMA_VERSION`                    | `artifact_ref_wire_schema_version`              |
  | `sase.config.runner_limit_override:RUNNER_LIMIT_OVERRIDE_WIRE_SCHEMA_VERSION`  | `runner_limit_override_wire_schema_version`     |
  | `sase.llm_provider.effort_override:EFFORT_OVERRIDE_WIRE_SCHEMA_VERSION`        | `effort_override_wire_schema_version`           |
  | `sase.sdd.plan_header_block:PLAN_HEADER_BLOCK_WIRE_SCHEMA_VERSION`             | `sdd_plan_header_block_wire_schema_version`     |
  | `sase.sdd.plan_refs:PLAN_REFERENCE_RESOLUTION_WIRE_SCHEMA_VERSION`             | `plan_reference_resolution_wire_schema_version` |

- Resolve bindings through `sase.core.rust.require_rust_binding` so a missing binding
  fails with the repo's standard stale-wheel message rather than a bare
  `AttributeError`.
- The failure message must name both sides and the remedy, e.g.
  `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION is 2 but sase_core_rs.artifact_file_query_wire_schema_version() reports 3; update the Python constant in the same commit that raises the sase-core-rs floor`.
  The value of this test is that it names the contract; a bare `assert a == b` would be
  no better than the parity test that already exists.
- Add a module docstring stating the invariant: a commit that raises the `sase-core-rs`
  floor in `pyproject.toml` across a wire bump must carry the matching Python constant
  bump. That is the process rule this whole outage reduces to.

## Verification

Run from the workspace checkout. `just install` first — the workspace venv may carry a
core older than the pyproject floor, and the whole bug is invisible under a stale core:

```bash
just install
.venv/bin/python -c "import sase_core_rs as m; print(m.artifact_file_query_wire_schema_version())"   # must print 3
```

If that prints `2`, the environment is building core from a stale `sase-core` checkout
(see "Why the existing gates did not catch it", item 1) and cannot verify this fix.
Update that checkout to `origin/master` and reinstall before continuing.

Then:

```bash
.venv/bin/python -m pytest tests/test_rust_wire_schema_agreement.py \
    tests/test_artifact_file_query_facade.py tests/test_artifact_ref_preprocessing.py -q
.venv/bin/sase artifact list          # renders the table instead of the wire error
.venv/bin/sase artifact list -j       # exercises the JSON path
.venv/bin/sase core health
just check
```

Confirm the new test actually bites: temporarily set the constant back to `2`, confirm
`tests/test_rust_wire_schema_agreement.py` fails with the named message, then restore
`3`.

## Done when

- `sase artifact list` succeeds against a core built at the declared `pyproject.toml`
  floor.
- `ARTIFACT_FILE_QUERY_WIRE_SCHEMA_VERSION` is 3, the dead monkeypatch in
  `tests/test_artifact_ref_preprocessing.py` is gone, and no `unused_only` plumbing was
  added.
- `tests/test_rust_wire_schema_agreement.py` passes for all nine mirrored constants and
  fails with a contract-naming message when any one is reverted.
- `just check` passes.

## Out of scope

- The `--unused` filter and consumption reporting — owned by the running bead
  `sase-b9.3`.
- Changing `pyproject.toml`; the `>=0.13.1,<0.14.0` floor is already correct for wire 3.
- Relaxing the equality handshake into a floor or a supported-version set. The equality
  guard is a deliberate lockstep contract used identically by every other facade in
  `src/sase/core/`, and the `sase-b9` plan relies on it to turn a stale core into a loud
  failure. The defect was the split commit, not the guard.
- Changing the `Justfile` core-override behavior. Building core from a local checkout is
  the intended Rust development loop; the new test is the gate that makes the resulting
  skew visible.
