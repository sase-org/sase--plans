---
tier: epic
title: Finish and land the AXE chop-overrun indicator
goal:
  The AXE chop-overrun feature satisfies its original classifier and rendering contracts
  on real first paint, on every selected run, and in published installs; all landing
  verification is recorded and epic sase-jx is closed cleanly.
phases:
  - id: repair_core_contract
    title: Repair the classifier's timestamp and per-run contract
    depends_on: []
    size: medium
    description:
      "repair_core_contract: in sase-core, reject every run whose started_at is
      unparsable and extend the versioned verdict so Python can associate an overrun
      ratio with each raw cached run, then verify and publish the corrected binding
      without hand-editing release-plz-owned versions."
  - id: integrate_tui_contract
    title: Integrate per-run and responsive rendering in AXE
    depends_on:
      - repair_core_contract
    size: medium
    description:
      "integrate_tui_contract: consume the corrected core verdict in sase, render the
      detail-header mark only for the raw run currently selected, and make the overview
      choose wide or compact layout after initial layout and immediately after terminal
      resize, with focused unit and PNG coverage."
  - id: publish_core_floor
    title: Ratchet the published core dependency contract
    depends_on:
      - repair_core_contract
      - integrate_tui_contract
    size: small
    description:
      "publish_core_floor: after the corrected sase-core release is fully available, use
      the repository's release-owned ratchet workflow to move pyproject.toml and uv.lock
      to the first published sase-core-rs version containing the complete corrected
      chop-overrun schema contract, and verify the floor-pinned binding inventory and
      schema probe."
  - id: close_epic
    title: Verify and close epic sase-jx
    depends_on:
      - publish_core_floor
    size: medium
    description:
      "close_epic: verify the combined cross-repo tree, record all verification,
      integration, and follow-up outcomes, close sase-jx without force, run post-close
      Symvision cleanup, and set status done in the original epic plan."
proposed_by: bbugyi200.athena.sase-jx.land
parent_bead: sase-jx
create_time: 2026-08-12 12:13:53
status: wip
---

- **PROMPT:**
  [prompts/202608/land_axe_chop_overrun.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/land_axe_chop_overrun.md)
- **PARENT:**
  [202608/axe_chop_overrun_indicator.md](https://github.com/sase-org/sase--plans/blob/main/202608/axe_chop_overrun_indicator.md)

# Plan: Finish and land the AXE chop-overrun indicator

## Context and verified evidence

Epic `sase-jx` has four closed phases and one commit for each phase:

- `sase-core` `c1a0a7361` (`sase-jx.1`) adds the Rust classifier and PyO3 bindings.
- `sase` `46773f606` (`sase-jx.2`) preserves `script_duration_ms` through launched-agent
  finalization and tolerates unknown persisted run keys.
- `sase` `2f1512c7c` (`sase-jx.3`) adds the typed Python facade and full/targeted AXE
  snapshot wiring.
- `sase` `d4c4efda5` (`sase-jx.4`) renders sidebar, overview, detail-header, onboarding,
  docs, and PNG coverage.

The land review read the original plan at `plans:202608/axe_chop_overrun_indicator.md`,
every source path changed by those commits, every child bead note and history entry, and
every commit after the epic started. Most reported behavior is present and covered, but
four remaining defects prevent an honest close:

1. `sase-core/crates/sase_core/src/axe_overrun/classify.rs` parses `started_at` only for
   `running` runs. A completed `success` run with `started_at="not-a-timestamp"` and
   `duration_ms=65000` is sampled and returns `level="over"`. The original plan
   explicitly says every run with an unparsable `started_at` is dropped, never fatal.
2. The Rust verdict reports only the newest _sampled_ ratio, while the TUI detail header
   keys it to raw `run_idx == 0`. With raw history `[skipped 1ms, success 65s]`, the
   classifier correctly skips the first entry and returns the older run's ratio, but the
   detail header paints `⚠ 1.1×` on the selected skipped run. It also cannot paint an
   older overrun when the user pages to it. This violates the plan's requirement that
   the segment follow the run currently being viewed.
3. `AxeOutputSection.update_lumberjack_overview` receives `width=None` before first
   layout and selects the 68-cell wide table. No resize/layout handler repaints it after
   width settles or when the terminal resizes. The narrow PNG added by `sase-jx.4` works
   around the production bug by manually calling `_refresh_axe_display()` after layout.
4. `sase-core-rs` 0.26.3 is published with the new v1 bindings, but the `sase`
   dependency window remains `>=0.24.0,<0.25.0`.
   `just ratchet-core-window --report-only` reports a pending move to 0.26.3. A normal
   published install can therefore resolve a core without either binding and silently
   degrade to no indicator. Because the core correction below changes the versioned wire
   contract again, the final floor must be the first complete release, not necessarily
   0.26.3.

The only `PROPOSED FOLLOW-UP:` entries were on `sase-jx.4`:

- The stale-width issue is retained as epic work because the phase promised a working
  narrow layout and its new test contains a manual refresh workaround.
- The 11 unrelated PNG mismatches were independently reproduced by
  `just test-visual -k axe` (11 failed, 21 passed, 1 skipped), registered as
  `file:explicit:8668b25f99aed578e9b544a7`, and corroborated after close on the semantic
  duplicate task `sase-dl`, which reopened ready. Do not rebaseline those unrelated
  goldens in this epic.

Post-start commits were also reviewed. Dynamic artifact-pane and Chats-pane changes
touch ACE globally but do not overlap AXE collection/rendering. The post-feature `sase`
0.17.0 release and `sase-core` 0.26.3 release do overlap the feature's published
dependency contract and motivate phase `publish_core_floor`.

## Phase 1: Repair the classifier's timestamp and per-run contract

Work in the linked `sase-core` repository opened with `/sase_repo`. Preserve the
repository's release-plz ownership of Cargo versions.

1. In `crates/sase_core/src/axe_overrun/classify.rs`, parse and validate every
   otherwise-known run's `started_at` before deriving its blocking time. Invalid
   timestamps must produce an unsampled entry for completed, active, excluded, and
   action statuses alike; they must never fail the request. Keep negative blocking
   durations unsampled.
2. Evolve the chop-overrun wire schema so a single classifier call returns one
   `Option<f64>` ratio aligned with each raw request run (for example, `run_ratios`). An
   unsampled raw entry has `None`; a sampled entry has its blocking/interval ratio.
   Preserve existing summary fields (`level`, counts, worst values, and latest sampled
   ratio) for sidebar, roll-up, and overview consumers. Bump the schema version rather
   than changing a versioned shape under the old number.
3. Extend Rust unit tests to prove:
   - completed, running, excluded, and action entries with invalid timestamps are
     unsampled;
   - `[skipped, over-success]` returns aligned ratios `[None, Some(...)]` while
     retaining `level="over"` and the older run as `latest_ratio`;
   - an older overrun can be located after a healthy sampled newest run;
   - response field order/round-trip and structural errors match the new schema.
4. Update PyO3 binding round-trip and error tests plus the binding inventory for the new
   schema. The public binding names should remain stable unless a schema migration
   genuinely requires a new name.
5. Run `just check` from the `sase-core` root. Commit through the normal SASE workflow
   with `SASE_BEAD=[sase-jx]` provenance. Do not edit Cargo versions; let release-plz
   publish the resulting patch release.

## Phase 2: Integrate per-run and responsive rendering in AXE

Work in the primary `sase` repository after rebuilding the corrected linked core with
`just install`. Read `sase/memory/tui_perf.md` through `/sase_memory_read` before
editing responsive TUI behavior.

1. Update `src/sase/axe/chop_overrun.py` to the corrected schema version and rehydrate
   the aligned per-raw-run ratios into an immutable, typed shape. Validate list length
   and item types so malformed bindings degrade through the collector's existing
   exception boundary rather than corrupting display state.
2. Keep all classification in Rust and at collection time. Do not duplicate the sampling
   rule in Python and do not call the binding from a render path.
3. In `src/sase/ace/tui/widgets/_axe_dashboard_status.py`, select the ratio by the
   displayed raw `run_idx`. Render `⚠ N× of Ms interval` exactly when that selected
   entry's aligned ratio is at least 1.0. This must:
   - suppress the mark on a raw newest skipped/invalid entry even if an older sampled
     entry makes the chop's window level `over`;
   - show the mark when paging to any older raw entry that itself overran;
   - preserve no-verdict and out-of-range degradation without raising.
4. Fix responsive overview rendering at the widget that owns the actual output width.
   Cache only the already-collected snapshot needed for a presentation-only repaint,
   clear that cached view when another output mode is shown, and repaint when the width
   crosses the wide/compact threshold. Ensure the first non-zero layout width corrects
   pre-layout wide content and later terminal resizes update immediately. The handler
   must do no disk I/O, subprocess work, or async waiting.
5. Remove the manual `_refresh_axe_display()` workaround and its explanatory comment
   from `test_axe_chop_overrun_narrow_png_snapshot`; the test should pass through normal
   layout behavior.
6. Add focused tests for invalid/skipped newest entries, paging onto/off older overruns,
   schema/list validation, first-layout compact selection, resize in both directions,
   and clearing the cached overview when switching views. Keep the two overrun PNG
   snapshots and all six intentionally widened-table goldens stable unless visual
   inspection proves an intentional change.
7. Run focused facade, collector, dashboard, status, and visual tests, followed by
   `just check`. Do not alter the 11 unrelated AXE-editor goldens tracked by `sase-dl`.

## Phase 3: Ratchet the published core dependency contract

This phase is release integration, not a manual dependency edit.

1. Confirm the phase-1 `sase-core-rs` release is fully published with an sdist and all
   supported wheels, and that it exposes the corrected schema and bindings. If release
   publication is still in flight, wait for it rather than pinning an unpublished
   checkout version.
2. Run the repository-owned ratchet (`just ratchet-core-window`) so `pyproject.toml` and
   `uv.lock` move together to the newest complete release. Review the diff and verify
   the chosen floor is at least the first release that contains phase 1. Do not
   hand-edit either version window.
3. Run the core-version validation and floor-pinned binding inventory/smoke gates used
   by CI. Prove a clean install resolving exactly the declared minimum exposes
   `chop_overrun_wire_schema_version` and `classify_chop_overrun` with the schema the
   Python facade requires.
4. Run `just install` again from the combined tree and rerun the focused chop-overrun
   facade/collector tests.

## Phase 4: Verify and close epic sase-jx

This is the final phase. Do not close the epic until every prior phase is complete and
committed.

1. Re-run the land audit against the final tree:
   - `sase bead show sase-jx`, its full history, and every child show/history;
   - the linked original plan and all child notes;
   - every `sase-jx` commit in both repositories;
   - every non-epic commit since the first `sase-jx` commit, confirming no new overlap
     remains.
2. Run `just check` in `sase-core`. In `sase`, run `just install`, focused AXE tests,
   both overrun PNG nodes, `just check-full`, and `just test-visual -k axe`. The full
   visual subset may still fail only on the 11 exact nodes recorded on ready task
   `sase-dl`; verify their signatures and do not accept their goldens. Any new or
   feature-related failure remains epic work.
3. Perform the original plan's live `sase ace` checks where practical: sidebar and
   collapsed-parent chips, overview PACE/advisory, detail paging, immediate compact
   first paint and resize response, a fast agent-launching chop with no false mark, and
   the Guide legend. Record any environment limitation honestly.
4. Append a comprehensive close note that records:
   - phase-by-phase source and test verification;
   - the post-start integration audit and dependency-floor correction;
   - the invalid-timestamp, per-run detail, and responsive-layout repairs;
   - follow-up disposition: stale width completed as epic work; PNG drift
     corroborated/reopened as `sase-dl` with artifact
     `file:explicit:8668b25f99aed578e9b544a7`;
   - all final command results and any deliberately scoped unrelated failures.
5. Close with `sase bead close sase-jx --note "<verified close note>"`. Do not use
   `--force` merely to make the command succeed. If close names an unfinished
   descendant, finish or reopen it; use forced canceled/superseded resolution only for a
   deliberate lifecycle decision with a real reason.
6. **After the close**, run `just symvision` if available. Read
   `sase/memory/symvision.md` through `/sase_memory_read` before fixing findings; remove
   expired `sase-jx` whitelist entries and any newly exposed unused code, then rerun the
   relevant checks.
7. Open the plans sidecar through `/sase_repo` and change only the original epic plan's
   frontmatter at `plans:202608/axe_chop_overrun_indicator.md` from `status: wip` to
   `status: done`. Verify the plan link and sidecar status. Also ensure this landing
   plan reaches its normal completed lifecycle through the executing epic's land agent.

## Non-goals

- Do not implement the unrelated PNG drift fix or accept its 11 goldens here.
- Do not add CLI/notification overrun surfaces, scheduling changes, auto-tuning, static
  timeout-risk warnings, sorting/filtering, or new navigation commands.
- Do not move classification policy into Python or perform I/O in resize/render
  handlers.
