---
tier: tale
title: Extract the family-shell substrate
goal:
  Shared family-shell mechanics live in a reusable sase.shells package while all monitor
  APIs remain compatible and monitor follow-ups fork the whole family transcript.
size: medium
proposed_by: bbugyi200.athena.sase-ud.2
bead: sase-ud.2
create_time: 2026-08-26 14:08:12
status: wip
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.2](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.2.md)

# Extract the family-shell substrate

## Goal

Complete phase `sase-ud.2` as a pure refactor: introduce a kind-parameterized
`sase.shells` package for the family-shell mechanics currently embedded in
`sase.monitor`, preserve every existing monitor import and behavior through thin
facades, and make the one approved behavior change that monitor follow-ups fork the
whole family transcript.

No gate-shell schema, execution, UI, Rust, or migration work belongs in this phase.

## Implementation

1. Create focused `sase.shells` modules for reusable shell mechanics: family-member
   artifact creation and inherited metadata, suffix allocation, pending-marker handoff
   and runner termination, workspace settlement, follow-up launching and prompt
   scaffolding, output capture, status-pair presentation, and shell-state
   bucketing/predicates. Parameterize each boundary with explicit shell inputs
   (kind/role, suffix constants, marker name and payload, metadata fields, status/state
   configuration) instead of adding monitor/gate branches.
2. Rebuild the monitor-specific entry points as thin adapters and re-exports. Keep
   existing public names, signatures, exception translation, monkeypatch seams, module
   invocation paths, and import paths under `sase.monitor`, `sase.monitor_status`, and
   `sase.monitor_state` working so callers outside the monitor package do not need to
   migrate in this phase. Keep monitor-only command supervision, storage,
   reconciliation, proc integration, request models, and CLI behavior in `sase.monitor`.
3. Separate generic member creation from the monitor metadata block: the substrate
   creates the inherited family member for an explicit shell kind/role, while the
   monitor adapter layers the existing `monitor_*`, proc, lineage, and execution fields
   without changing the serialized monitor record.
4. Generalize suffix allocation and pending handoff around caller-supplied suffix
   constants/templates and marker configuration, then have monitor adapters pass the
   current `--mon` rules and `.sase_monitor_pending` payload. Preserve refresh-pulse,
   error, and kill ordering semantics.
5. Move shared settlement/follow-up machinery and prompt helpers behind shell-generic
   APIs while retaining monitor-specific persistence keys and messages in adapters.
   Change prompt routing to use `#fork:<family>` when the starter settled, instead of
   `#fork:<starter-name>`; preserve the no-fork degraded path and model/effort routing.
6. Add focused substrate tests for each parameterized seam and update monitor facade,
   lifecycle, status/state, naming, member, handoff, settlement, follow-up, and golden
   prompt tests. Assert both API compatibility and the new family-level fork prefix.

## Verification

- Run focused tests for the new `sase.shells` package and `tests/monitor/`, including
  the follow-up prompt goldens and facade-import coverage.
- Run `just check` while iterating and address only failures caused by this phase.
- Run the required exhaustive `just check-full` through `/sase_monitor` before closing
  the phase.
- Run `sase bead epic-symbols sase-ud.2`; resolve every remaining phase symbol or re-key
  it to an open epic/later phase.
- Close only `sase-ud.2` with a note naming the compatibility, focused-test, and full
  verification evidence. Record any out-of-scope discovery only as a
  `PROPOSED FOLLOW-UP:` note on this phase bead.
