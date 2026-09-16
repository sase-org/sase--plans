---
tier: tale
title: Fix Master Gate by ratcheting the stale sase-core revision pin
goal:
  Master Gate goes green on the next master push because sase-core-revision.txt pins a
  sase-core commit that exposes all 631 required sase_core_rs bindings.
size: small
proposed_by: bbugyi200.athena.0lv.w0
create_time: 2026-09-16 09:44:10
status: wip
---

# Fix Master Gate: Ratchet The Stale sase-core Revision Pin

## Problem

Every Master Gate run on sase master has been red since commit `297e6122b0`
("feat(config): wire AXE routine job contract consumers", 2026-09-16 06:32 UTC; last
green was `5c1af84d91` at 06:07 UTC). Two symptoms, one root cause:

1. The lint job fails at "Check pinned core bindings":
   `tools/check_sase_core_rs_bindings` reports that `sase_core_rs 0.34.35` is missing 11
   of 631 required bindings: `agent_tribe_display_key`,
   `canonicalize_agent_tribe_metadata`, `canonicalize_public_tribe_name`,
   `is_reserved_tribe_name`, `parse_tribe_reference`, `project_axe_status_public`,
   `public_tribe_name`, `reserved_tribe_target_reason`,
   `resolve_agent_tribe_display_config`, `resolve_agent_tribe_identity`,
   `validate_tribe_name`.
2. Six of eight fast-suite shards fail (~1,630 FAILED lines in run 35097762781; shards 1
   and 4 hit the 20-minute job ceiling). 1,357 failures directly raise
   `AttributeError: module 'sase_core_rs' has no attribute '<binding>'` via
   `require_rust_binding` (`src/sase/core/rust.py`). The remaining ~273 are concentrated
   in tribe/axe/agent-runner test files downstream of the same code paths (bootstrap
   errors get caught and surface as secondary assertion or missing-file failures).

Root cause: `sase-core-revision.txt` pins sase-core commit
`a7d588263e5a1c69f49dddbb2f72a138382b53a5` (builds `sase_core_rs` 0.34.35). Master Gate
builds the Rust core wheel strictly from that pin (`.github/workflows/master-gate.yml`,
`core-wheel` job). Today's sase commits — starting with `297e6122b0` (axe routine job
contract consumers, needs `project_axe_status_public`), then `e4700fd747` and
`edde28a8dd` (agent-tribes, need the tribe bindings) — call bindings that only exist in
newer sase-core commits (`ad13940`, `fe1a17b`, `d0f9cf8`, `51c7c38`, all already merged
to sase-org/sase-core master; verified present in `crates/sase_core_py/src/lib.rs` at
HEAD `7ce99089300bc75127946b17c938f15d83b831e6`). The scheduled `core-pin-ratchet.yml`
workflow only fires every 6 hours and has not yet proposed the bump, and no ratchet PR
is open.

## Fix

Ratchet the pin to sase-core's current remote HEAD, exactly as the check tool's remedy
text instructs:

1. From the repo root, run:

   ```bash
   just ratchet-core-revision
   ```

   This runs `tools/ratchet_core_revision`, which queries
   `https://github.com/sase-org/sase-core.git` HEAD and rewrites
   `sase-core-revision.txt`. Exit code 2 with output
   `sase-core pin ratchet a7d588263e5a -> <new sha> applied` is SUCCESS (2 means "a
   ratchet was applied", 0 means "already current"). Exit code 3 is failure. The new SHA
   is expected to be `7ce99089300bc75127946b17c938f15d83b831e6` or a descendant of it;
   all 11 missing bindings were verified present at that commit. If the applied SHA is
   NOT `7ce9908...` or a descendant, stop and verify the 11 bindings above exist in
   `crates/sase_core_py/src/lib.rs` at the applied SHA (open the repo with
   `sase repo open sase-core -r "..."` first) before proceeding.

2. The only tracked-file change in this repo is the one-line content of
   `sase-core-revision.txt`. Do not edit `pyproject.toml`'s published sase-core-rs
   version window — the release-branch reconciler ratchets that at release time
   (`just rust-install`'s own note says no action is needed when the checkout is ahead
   of the window).

## Validation

1. Rebuild the local Rust core so the venv wheel matches the new pin:

   ```bash
   just rust-install
   ```

   This refreshes the linked sase-core checkout (`sase/repos/linked/sase-core`) to
   remote HEAD and builds/installs `sase_core_rs` from it via maturin. (Run
   `just install` first if the venv itself is stale.)

2. Confirm the binding gate that CI runs now passes (must exit 0, no missing bindings):

   ```bash
   ./.venv/bin/python tools/check_sase_core_rs_bindings
   ```

3. Spot-check test files that failed on the gate for reasons that did not literally name
   a missing binding, to confirm they were downstream of the same root cause:

   ```bash
   ./.venv/bin/python -m pytest \
     tests/ace/tui/test_agent_tribe_assignment.py \
     tests/ace/tui/test_agent_toggle_approve.py \
     tests/ace/tui/test_agent_marking_save.py \
     tests/main/test_monitor_handler_list.py \
     tests/test_fork_workflow.py \
     tests/test_run_agent_runner_bootstrap_errors.py \
     tests/test_agent_launch_validation.py
   ```

   All are expected to pass with the rebuilt core. If any test still fails with the new
   wheel installed, it is a genuinely separate regression from today's landing window:
   fix it here only if the cause is immediate and obvious; otherwise record it as
   discovered follow-up work (use `/sase_new_task` first, per standing instructions).

4. Run the standard agent verification gate:

   ```bash
   just check
   ```

   Hand it to `/sase_monitor` if it runs long. It must pass before finishing.

Do not create commits, branches, or PRs yourself; completion is host-owned and the
finalizer acts on your declaration.

## Expected Post-Land Signal

The next Master Gate run on master (which builds the core wheel from the new pin) goes
green: the "Check pinned core bindings" lint step passes and the eight fast-suite shards
stop raising `sase_core_rs` AttributeErrors.
