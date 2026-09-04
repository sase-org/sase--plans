---
tier: tale
title: Remove ACE agents-sync import surfaces
goal:
  ACE exposes only the manual publication-scoped agents-sidecar sync and no
  incoming-import UI, configuration, startup work, or comprehensive-update leg.
size: medium
proposed_by: bbugyi200.kellys_mbp.sase-ws.1
bead: sase-ws.1
create_time: 2026-09-04 14:16:54
status: wip
---

- **PARENT:** [202609/remove_agents_sync_import.md](remove_agents_sync_import.md)
- **BEAD:**
  [sase-ws.1](https://github.com/sase-org/sase--beads/blob/main/pages/sase-ws/sase-ws.1.md)

# Remove ACE agents-sync import surfaces

## Goal

Complete phase `sase-ws.1` of the publish-only agents-sidecar epic by removing every ACE
surface that detects, displays, previews, or applies incoming agent imports. Keep the
Admin Center Updates-pane `a` action as the sole manual agents-sidecar operation; it
must continue to run the full publication/reconciliation sync as a tracked proc.

## Constraints

- Preserve the `sase agent sync` publication path and the manual `a` binding in the
  Admin Center Updates pane. Retain its `agents-sync` deduplication/exclusive scope and
  refresh the Agents view after it completes.
- Remove the legacy import leg from the comprehensive update; “Everything” becomes
  SASE/core/plugins plus providers only. Do not substitute another import action.
- Keep the global `,U` / `update_sase` leader keymap, updating only its help/catalog
  label to describe SASE and providers.
- Do not change the agents-sync backend models or publication engine in this phase;
  later epic phases own those removals. Avoid unrelated `agents_sync` false positives
  such as publication modules, `ref_sync_gesture`, `sidecar_auto_sync`, and `#sync`.
- Keep keystroke and render paths allocation-only. Removing the periodic import status
  worker must not add any synchronous I/O or alternate refresh path.
- Update `src/sase/default_config.yml` whenever config/keymap-facing behavior changes,
  and keep config schema, help content, proc-site inventories, and tests synchronized.

## Implementation

1. Remove the indicator and periodic import-status subsystem.
   - Delete `agents_sync_indicator.py`, its lazy/type exports, top-bar mount, TCSS rule,
     unit/visual tests, and `agents_sync_indicator_pending_120x40.png` golden.
   - Delete `_agents_sync_config.py`, the `ace.agents_sync` default-config block and
     JSON-schema object, plus their schema/config tests.
   - Remove the post-mount scheduler call, late config parsing, runtime state seeding,
     startup declarations, and ACE test-harness patch points for `_agents_sync_*`.
   - Slim `AgentsSyncActionsMixin` to `action_sync_agents` and its existing coalesced
     Agents-view refresh helper. Remove all status scheduling/revalidation methods and
     `action_integrate_cached_agents`; update the surviving tracked-proc phase, display,
     summary, and duplicate text to say it publishes and reconciles agent hoods rather
     than importing them.
   - In `agents_sync_format.py`, retain only the full-sync outcome line and summary
     helpers; delete cached-hood/status formatting and their tests. Remove the
     `agents.cached` proc-producer site while keeping `agents.sync`.

2. Remove agents as a comprehensive-update leg.
   - Delete `UpdateLeg.AGENTS` / `UpdateScope.AGENTS` and make `ALL_LEGS` contain only
     SASE and providers.
   - Remove captured-agent fields, errors, runnable checks, planning, preview sections,
     execution, result fields/properties, completion refreshes, and all agent-import
     copy from the comprehensive-update model/preview/execution/action modules.
   - Remove the comprehensive update’s `agents-sync` exclusive scope and the matching
     `update.scoped` proc-site concurrency key. The manual publication sync retains its
     own `agents-sync` scope.
   - Update shared comprehensive-update fixtures/tests so Everything has two sections,
     selected legs and summaries cover only SASE/providers, and failure/restart logic no
     longer expects cached-agent outcomes.

3. Reduce the Update panel to three rows using only the update-status snapshot.
   - Remove the `agents` scope, row copy, accent/glyph, pending-count/detail/kind
     projection, and `agents_snapshot` argument. Compute Everything and freshness from
     the SASE/provider snapshot only.
   - Remove `a`/`A` panel bindings and actions, update the legend to `e s p` / `E S P`,
     and update docstrings plus selection/state tests for three rows.
   - Stop threading `_agents_sync_last_status` through `action_update_sase_shortcut`,
     update-panel refresh/recheck, busy-state calculation, duplicate-scope messaging,
     post-run revalidation, and Agents-view reloads.

4. Clean remaining ACE presentation and inventories.
   - Remove the Updates-pane agents freshness chip and associated status helpers/tests,
     while preserving `a sync agents` in `plugins_browser_pane.py`, layout delegation,
     status hints, and their existing assertions.
   - Update the `update_sase` command catalog and all three help-modal binding lists
     from “Update panel (SASE, providers, agents)” to “Update panel (SASE, providers)”,
     then update command/keymap tests. Keep the configured `update_sase: "U"` binding.
   - Adjust top-bar order, startup stopwatch/testing helpers, proc scope tests, config
     tests, and any other shared-file assertions identified by the epic design.

5. Update visual expectations and sweep the scoped surface.
   - Run the focused ACE/config/comprehensive-update tests while iterating.
   - Run `just test-visual`; accept the intentional three-row Update-panel changes with
     `just test-visual -- --sase-update-visual-snapshots` (or the Justfile-equivalent
     argument form), then rerun exact visual comparison. Confirm
     `config_center_comprehensive_update_preview_120x32.png` remains unchanged and
     inspect the regenerated Update-panel PNGs.
   - Search the touched ACE/config/test surface for removed indicator, cached-import,
     periodic-status, `UpdateLeg.AGENTS`, `UpdateScope.AGENTS`, `agents_snapshot`, and
     `agents.cached` references. Classify surviving `agents-sync` hits as the preserved
     manual publication action or later-phase backend work.

## Verification

1. Run `just install` before repository verification because this is an ephemeral
   workspace.
2. Run focused pytest coverage for agents sync actions/formatting, Update-panel state
   and shortcuts, comprehensive preview/execution, startup/test harness, proc-site
   inventories, top-bar order, config schema, keymap help, and command catalog.
3. Run and, where intentional, regenerate `just test-visual`; rerun it without update
   mode to prove exact snapshot stability and verify the config-center preview golden
   did not change.
4. Run `just check` as the required phase gate.
5. Immediately before closure, run `sase bead epic-symbols sase-ws.1`; resolve any
   unexpected entries without closing the parent epic, then close only `sase-ws.1` with
   a note naming the verification that passed.
