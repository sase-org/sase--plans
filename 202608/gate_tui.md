---
tier: tale
title: Gate shells in ACE
goal:
  ACE renders gate shells as first-class family members with correct status semantics,
  live decision output, folds, and visual coverage.
size: medium
proposed_by: bbugyi200.athena.sase-ud.6
bead: sase-ud.6
create_time: 2026-08-26 18:12:10
status: wip
---

- **PARENT:** [202608/gate_shells.md](gate_shells.md)
- **BEAD:**
  [sase-ud.6](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ud/sase-ud.6.md)

# Plan: Gate shells in ACE

## Goal

Teach ACE to treat a gate shell as a first-class non-LLM family shell without conflating
it with a monitor: project the existing Rust wire fields into the TUI model, show
gate-specific row chrome, family lanes and counts, render the structured gate decision
plus selected-shell live output, and preserve agent-only accounting and family status
semantics.

## Context and constraints

- Phase `sase-ud.6` implements the `gate-tui` slice of `plan:202608/gate_shells.md`; the
  Rust read model and durable `gate.log` execution pipeline are already present from its
  dependencies.
- Keep all work in the Python presentation/adaptation boundary. Do not duplicate scanner
  or domain behavior that belongs in `sase-core`.
- The gate glyph is `⋔` (U+22D4). Pending uses the gate status-pair accent, settled uses
  `#9E9E9E`, and failed/timeout/lost uses `#FF5F5F`. Use `⊣` only if inspection against
  the pinned visual fixture demonstrates that `⋔` is unreadable, and record that choice
  in the phase close note.
- Follow `tui_perf.md`: gate metadata comes from the scanned structured projection; only
  the selected shell tails its artifact log; existing cached artifact-file and
  `render_axe_output` paths are reused; render code performs no new stat/glob/JSON or
  bundle reads.
- Do not create follow-up beads. Record any discovered work on `sase-ud.6` as a
  `PROPOSED FOLLOW-UP:` note.

## Implementation

1. Extend the integration and ACE row models with the flat gate-shell wire fields and a
   strict `is_gate` predicate based on role plus `gate_id`. Enrich both the indexed wire
   loader and filesystem fallback through a shared gate metadata helper, derive gate
   state/status buckets and effective status pairs, and map `gate_output_path` to the
   existing artifact-output cache so `Agent.get_live_reply_content()` can serve
   `gate.log` without render-path I/O. Cover meta and done-marker settlement sources.

2. Audit all ten `is_monitor` decisions in `models/agent_family_members.py` individually
   and encode the gate answer next to each decision. Gate shells must participate where
   the newest family shell supplies status (including `concrete_agent_statuses`) and be
   excluded from agent/runner/ completion counts (including
   `concrete_family_member_rows`). Preserve chronological concrete-shell ordering for
   roster and phase rendering, and add focused tests for both opposing directions plus
   the pending/settled newest-shell family status projection.

3. Generalize monitor-only family lane accounting and glyph chips into per-shell-kind
   counts while retaining compatible monitor helpers where useful. Render combined
   family chips such as `⚙2 ⋔1`, give gate rows and gate chips state-derived styles, add
   the running/settled gate entries beside monitor entries in the help modal, and add
   `_GateShellLane` beside `_MonitorShellLane` with a bounded decision title and a
   human-readable pending deadline/countdown.

4. Add a gate detail renderer as the structural peer of the monitor renderer. Define
   `GATE_PHASE_LABEL = "GATE"` and `GATE_SECTION_ID = "gate"`; render, from already
   projected/cached data, the phase divider, decision title/kind/id, compiled branches
   and selected branch, reviewer note, per-option JSON results, state/status/elapsed/
   deadline, inspection pointer, follow-up disposition, and bounded live output via
   `render_axe_output(f"gate:{gate_id}", ..., "ansi")`. Reuse notification-gate
   summary/branch-layout helpers and shell follow-up attention styling rather than
   rebuilding decision semantics in the TUI.

5. Wire the gate section into both single-member and family `AGENT REPLY` render paths,
   the selected-gate-shell pane, phase labels/roster rows, hint-mode flattening, and
   fold override/step rendering beside the monitor section. Ensure only a selected gate
   shell reads/tails output and that collapsed, expanded, and narrow layouts keep stable
   fold and jump behavior.

6. Add focused unit/integration coverage for wire/filesystem enrichment, every
   family-members filtering choice, glyph hues, per-kind counts, shell lanes, structured
   gate fields, output cache identity, and fold registration. Add ACE PNG scenarios for
   pending, executing, approved, failed, long-output, and narrow-width gate
   shells/families; inspect the generated diffs and accept only intentional rebaselines.

## Verification

1. Run the focused gate and affected ACE model/widget tests while iterating.
2. Run `just test-visual`; if snapshots intentionally change, update them with
   `--sase-update-visual-snapshots`, then rerun `just test-visual` cleanly.
3. Run `just check` as the repository gate. If it escalates or the touched surface
   qualifies as broadening, run `just check-full` through `/sase_monitor`.
4. Run `sase bead epic-symbols sase-ud.6` and resolve or re-key every remaining entry
   before closing only `sase-ud.6` with a note naming the unit, visual, and repository
   checks that passed.
