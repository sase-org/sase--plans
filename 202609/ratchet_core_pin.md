---
tier: tale
title: Ratchet the pinned sase-core revision to fix Master Gate
goal: Master Gate on sase master is green again because CI builds a sase_core_rs wheel
  that exposes every required binding.
size: small
proposed_by: bbugyi200.athena.0gi
status: done
---

# Ratchet The Pinned sase-core Revision To Fix Master Gate

## Problem

Master Gate is failing on sase master (run 33993789878, commit `457041681`):

- The `lint` job fails at step "Check pinned core bindings":
  `sase_core_rs 0.32.16 is missing 14 of 406 required binding(s)`.
- Test shards 2, 3, 4, 6, 7, and 8 fail in "Run sharded fast suite" with
  `AttributeError: module 'sase_core_rs' has no attribute 'normalize_owned_agent_name'`
  (and siblings) raised through `require_rust_binding` in `src/sase/core/rust.py`.

The 14 missing bindings are: `artifact_link_ref_parts`, `artifact_row_index_keys`,
`artifact_row_ref_lookup_keys`, `artifact_row_resolution_wire_schema_version`,
`artifact_row_resolve`, `foreign_agent_owner_root`, `globalize_owned_agent_name`,
`normalize_owned_agent_name`, `parse_owned_agent_name`,
`validate_agent_archive_capabilities`, `validate_agent_archive_key`,
`validate_agent_archive_visibility`, `validate_owned_agent_name`, and
`validate_owner_root`.

## Root Cause (already diagnosed — do not re-derive)

CI builds the `sase_core_rs` wheel from the git SHA pinned in `sase-core-revision.txt`
(see the `core-wheel` job in `.github/workflows/master-gate.yml`), not from sase-core
HEAD. The pin currently reads `51df9061fd8576145cb4226be1999d6f9499d99c`, which is
sase-core's v0.32.16 release commit. Python code on sase master (recent agent-identity
and artifact-row work) now calls 14 Rust bindings that were only added to sase-core in
releases v0.32.17 through v0.32.19, so the wheel built from the stale pin lacks them.
The lint step's own remedy text names the fix: bump the pin (see
`just ratchet-core-revision`).

Facts already verified during diagnosis (no need to repeat this research):

- sase-core's remote HEAD is `fe9a643cd295b692922c55cc206375692cac6db8` (the v0.32.23
  release commit). A static cross-check of the pyo3 registrations in
  `crates/sase_core_py/src/lib.rs` at that revision against the full output of
  `tools/check_sase_core_rs_bindings --list` (406 names) confirms every required binding
  is exposed there, including all 14 missing ones (`AtReferenceInventory` is a
  `#[pyclass]`, not a `#[pyfunction]`, and is present).
- sase-core commit `1416679` (in v0.32.23) removed three legacy bindings
  (`classify_agent_ownership`, `globalize_legacy_agent_name`, `localize_agent_name`);
  none of them appear in the required-binding list, so the removal cannot introduce a
  new failure.
- `pyproject.toml` already requires `sase-core-rs>=0.32.19,<0.33.0`, and all 14 bindings
  exist by v0.32.19, so the `release-core-floor-smoke` job in `.github/workflows/ci.yml`
  is unaffected. Do NOT touch the pyproject floor or `uv.lock`.
- No `core-pin-ratchet-*` PR is currently open. The scheduled Core Pin Ratchet workflow
  (`.github/workflows/core-pin-ratchet.yml`) exits cleanly once master's pin matches
  sase-core HEAD, so landing this fix does not race the bot.

## Change

Exactly one tracked file changes: `sase-core-revision.txt`.

1. Run the ratchet in apply mode from the repo root:

   ```bash
   just ratchet-core-revision
   ```

   This rewrites `sase-core-revision.txt` from
   `51df9061fd8576145cb4226be1999d6f9499d99c` to sase-core's current remote HEAD and
   exits 2 (2 means "ratchet applied" for this tool — not an error). The expected new
   pin is `fe9a643cd295b692922c55cc206375692cac6db8`; if sase-core HEAD has advanced
   past that by implementation time, the newer SHA is fine and expected — the tool
   always pins remote HEAD, and newer sase-core releases are supersets for the bindings
   at issue (still, the verification below must pass against whatever SHA gets pinned).

Do not edit any workflow file, `pyproject.toml`, or `uv.lock`. Do not hand-edit the SHA;
use the just recipe so the value is fetched, validated, and formatted by
`tools/ratchet_core_revision`.

## Verification

1. `just install` — the ephemeral workspace venv may be stale; this builds
   `sase_core_rs` from the linked sase-core checkout. If it reports the sase-core
   checkout is behind the floor, refresh it with
   `sase repo open sase-core -r "refresh linked checkout to build the pinned core"` and
   rerun `just install`.
2. `./.venv/bin/python tools/ratchet_core_revision --check` — must exit 0 ("already
   matches sase-core HEAD").
3. `./.venv/bin/python tools/check_sase_core_rs_bindings` — must report that the
   installed `sase_core_rs` exposes all required bindings (success message, exit 0).
4. `just check` — whole-repo lint gates plus the diff-scoped test lane. Hand it to
   `/sase_monitor` if it runs long. `just check-full` is not required for this change:
   the blast radius is the CI wheel build input, which Master Gate itself re-proves on
   the landed commit.

## Out Of Scope

- Bumping the `sase-core-rs` floor in `pyproject.toml` or regenerating `uv.lock` (the
  existing floor already covers every required binding; the version-window ratchet has
  its own tooling in `tools/ratchet_core_window`).
- Any change to `.github/workflows/` (the pin file is the designed control surface).
- Any sase-core repository change (the needed release already shipped).
