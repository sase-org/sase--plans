---
tier: tale
title: Viewer remote-node render parity
goal: Remote fleet nodes render and group like equivalent local nodes, differing only
  by deliberate remote affordances.
size: medium
proposed_by: bbugyi200.athena.sase-133.3
bead: sase-133.3
status: done
---

- **PARENT:**
  [202609/remote_dispatch_agents_tab_parity.md](https://github.com/sase-org/sase--plans/blob/main/202609/remote_dispatch_agents_tab_parity.md)
- **BEAD:**
  [sase-133.3](https://github.com/sase-org/sase--beads/blob/main/pages/sase-133/sase-133.3.md)

# Plan: Viewer Remote-Node Render Parity

Complete phase bead `sase-133.3` by making a remote fleet snapshot render through the
same Agents-tab projection, grouping, counting, and row-formatting paths as the
equivalent local snapshot. The only intentional row difference is the remote machine
chip; existing remote-only feed-health, last-seen, and action affordances must remain.

## Context and constraints

- The approved epic design is `plan:202609/remote_dispatch_agents_tab_parity.md`.
- Phase `sase-133.2` has landed fleet contract schema v4 in `sase-core`. Its summary
  `status` is the owner-resolved display status and it now carries resolved family
  linkage, tribe/clan facts, separate family and run start times, and terminal times.
  Current viewers must still tolerate older summaries that omit additive fields.
- The Python viewer already maps most fields in
  `src/sase/ace/tui/models/_fleet_agents_rows.py`, normalizes remote lineages in
  `_fleet_agents_nodes.py`, and then reuses `apply_status_overrides()` and
  `sort_and_reorder()`. Preserve that shared path rather than adding a parallel remote
  renderer.
- Existing remote-only presentation (machine chip, freshness/feed diagnostics,
  `WAS RUNNING`/last-seen behavior, and capability-gated actions) is deliberate and must
  not be removed for parity.
- Keep rendering and projection pure and in-memory; do not add filesystem, process, or
  network work to the TUI event loop.
- Follow `tui.md`, `tui_screenshot.md`, and `tui_perf.md`. If tracked files change, read
  `lint_and_test.md` before final verification.

## Implementation

1. Strengthen the fleet fixture and projection coverage around the newly landed owner
   facts.
   - Allow fixtures to express exact owner-presented statuses such as `TESTING`,
     `WORKING TALE`, `TALE DONE`, and `EPIC CREATED √` without reducing them to the
     coarse lifecycle bucket.
   - Correct the currently failing run-start fixture so its `observed_at_unix` is not
     earlier than `run_started_at_unix`, and assert both family/agent elapsed time and
     current-run elapsed time survive projection.
   - Retain an explicit legacy-summary case proving omitted schema-v4 fields degrade to
     the pre-v4 behavior instead of dropping the row or raising.

2. Make remote lineage normalization prefer real owner linkage and synthesize only as a
   compatibility fallback.
   - Resolve owner-local `parent_timestamp` and family identities to host-qualified row
     identities, including completed/historical shells.
   - When a real root/container is present, attach member, historical, monitor, gate,
     and proc rows beneath it and do not insert a lossy synthetic container.
   - Keep the existing synthesis path for older gateways that lack root/linkage facts.
   - Preserve owner `status`, `status_bucket`, `tribe`/`clan_tribe`, and the two runtime
     anchors through any clan/family aggregation; do not overwrite a rich status with a
     coarse remote bucket.

3. Exercise the shared presentation path with equivalent local and serialized-remote
   rosters.
   - Expand `tests/ace/tui/test_fleet_agents_display_parity.py` (and focused projection
     tests as appropriate) to include a running multi-shell family, a completed family
     with nested historical shells, a proc, a gate, a monitor, a non-default tribe, and
     rich owner status text.
   - Compare tree/parent signatures, panel keys, rendered status/runtime text, collapsed
     shell counts (`×N`), and proc/gate/monitor glyphs or count chips between local and
     remote rows after stripping only the machine chip.
   - Assert child shells do not repeat the machine chip while their rendered top-level
     owner nodes do.
   - Assert the machine-filtered remote visible-node count equals the equivalent local
     visible-node count.

4. Ensure tribe grouping and banner metrics count remote rows exactly as local rows.
   - Owner-resolved `tribe` and `clan_tribe` must place remote roots and descendants in
     the same `@default`, `@epic`, or other panels as their local equivalents.
   - Test a mixed panel with running, waiting, and done lanes and require the panel chip
     to include the done metric (for example `[R1 W1 D4]`). Counts must use the shared
     family/clan projection so nested shells do not inflate agent totals.
   - If a shared counter or renderer currently omits done or nested shell metrics, fix
     that shared implementation and add a focused regression test; do not special-case
     fleet rows.

5. Add dedicated visual coverage in
   `tests/ace/tui/visual/test_ace_png_snapshots_agents_fleet.py` for a remote tribe
   containing a running family with `×N` plus proc/gate/monitor presentation and a
   completed family with nested historical shells. Pin fixture time/data, assert key SVG
   text before accepting the PNG, and keep existing fleet-health snapshots intact.

## Verification and closeout

1. Run the focused non-visual tests for fleet projection/display parity, family shell
   lanes, panel counts/titles, and tribe summaries.
2. Run targeted `just fix-tui-screenshots -- <selector>` for the new/changed fleet
   visual case. Inspect the retained report and every created or updated golden; do not
   accept an unexplained image delta. Run the corresponding check-only visual test.
3. Run `just fix` and `just check`. If the scoped gate escalates or identifies a
   broadening-set change, follow `lint_and_test.md` and use `/sase_monitor` for
   `just check-full` rather than running it inline.
4. Re-read `sase bead show sase-133.3`, then run `sase bead epic-symbols sase-133.3`.
   Resolve every remaining symbol or re-key its Justfile line to the still-open
   parent/later phase; do not close with leftovers.
5. Record any out-of-scope discovery only as
   `sase bead note sase-133.3 'PROPOSED FOLLOW-UP: <summary — detail>'`; do not create a
   bead.
6. Close only this phase with
   `sase bead close sase-133.3 --note "<specific tests, visual inspection, and just check evidence>"`.
   Do not close `sase-133` or another ancestor.
