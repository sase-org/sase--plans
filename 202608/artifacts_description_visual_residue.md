---
status: done
tier: epic
title: Finish Artifacts description visual coverage
goal:
  Every visual snapshot affected by the Artifacts pane-description brief reflects the
  shipped default summary row, with the eleven missed Stitches goldens repaired and the
  wider visual corpus audited so no sase-u6-caused drift remains hidden inside the
  standing unrelated visual backlog.
parent_bead: sase-u6
phases:
  - id: stitches
    title: Repair the missed Stitches goldens
    depends_on: []
    size: small
    description:
      "stitches: regenerate the eleven artifacts_stitches_* PNG snapshots through their
      dedicated visual test module, inspect every actual/expected/diff artifact, accept
      only the pane-brief row and its resulting vertical layout shift, and rerun that
      module without snapshot-update mode."
  - id: audit
    title: Audit the remaining visual corpus and verify the repair
    depends_on:
      - stitches
    size: medium
    description:
      "audit: run the full visual lane, distinguish the standing sase-r5 and
      node-specific failures from panes affected by sase-u6, repair only additional
      snapshots whose delta is caused by the pane brief, and finish with the focused
      description/tooltip tests plus the repository verification gate."
proposed_by: bbugyi200.athena.sase-u6.land
bead_id: sase-u6.5
create_time: 2026-09-09 19:49:56
---

- **PROMPT:**
  [prompts/202608/artifacts_description_visual_residue.md](https://github.com/sase-org/sase--agents/blob/main/prompts/202608/artifacts_description_visual_residue.md)
- **PARENT:**
  [202608/artifacts_subtab_descriptions.md](artifacts_subtab_descriptions.md)
- **BEAD:**
  [sase-u6.5](https://github.com/sase-org/sase--beads/blob/main/pages/sase-u6/sase-u6.5.md)

# Plan: Finish Artifacts description visual coverage

## Problem

Epic `sase-u6` added a default one-row pane brief beneath the Artifacts sub-tab strip.
Its visual phase (`sase-u6.4`, commit `2cbe2f17d0`) added four dedicated description
snapshots and updated 32 existing `artifacts_*` PNGs, but it did not update any of the
eleven `artifacts_stitches_*` PNGs. Those eleven files last changed in pre-epic commit
`4dd2995028`; direct inspection of `artifacts_stitches_empty_120x40.png` confirms that
the checked-in image jumps directly from the sub-tab strip to the query bar and omits
the default Stitches brief. The corresponding test does not disable the description
mode, so this is stale expected output caused by the epic rather than optional coverage.

The repository also has a standing, independently tracked visual backlog (`sase-r5` and
its node-specific related beads). A broad update would conceal unrelated failures. This
repair therefore needs an attribution pass, not a blanket rebaseline.

## Phase `stitches`: Repair the missed Stitches goldens

Run the dedicated Stitches visual module with snapshot updates enabled:

```bash
just test-visual -- \
  tests/ace/tui/visual/test_ace_png_snapshots_commits.py \
  --sase-update-visual-snapshots
```

Inspect the generated actual, expected, diff, and source artifacts under
`.pytest_cache/sase-visual/` for all eleven snapshot names:

- `artifacts_stitches_timeline_detail_120x40`
- `artifacts_stitches_origin_legend_120x40`
- `artifacts_stitches_merge_row_120x40`
- `artifacts_stitches_empty_120x40`
- `artifacts_stitches_persistent_filter_80x24`
- `artifacts_stitches_jump_hints_120x40`
- `artifacts_stitches_filter_bar_prefilled_120x40`
- `artifacts_stitches_filter_completion_120x40`
- `artifacts_stitches_sidecar_filter_120x40`
- `artifacts_stitches_narrowed_filter_chips_120x40`
- `artifacts_stitches_filter_parse_error_120x40`

For each image, require the new Stitches summary brief directly under the strip and
limit acceptance to that row plus the consequent vertical displacement or resize of the
existing content. Treat any unrelated content change as a test or product defect and fix
it before accepting the image. Rerun the same module without the update flag and require
exact passes.

## Phase `audit`: Audit the remaining visual corpus and verify the repair

Run `just test-visual` through `/sase_monitor`. Use its per-node artifacts to classify
every failure:

- A snapshot whose only new delta is the Artifacts description brief is still `sase-u6`
  work. Regenerate it narrowly by node, inspect the diff, and rerun it exact.
- Failures already matching `sase-r5` or its node-specific related beads remain that
  backlog. Do not update those goldens and do not turn them into duplicate tasks.
- A distinct failure not caused by `sase-u6` goes through `/sase_new_task`, with the
  proposing phase recorded as this child epic's audit phase.

After the attribution pass, rerun the focused non-visual feature coverage:

```bash
.venv/bin/python -m pytest -q \
  tests/ace/tui/test_artifacts_pane_descriptions.py \
  tests/ace/tui/test_artifacts_pane_brief.py \
  tests/ace/tui/test_artifacts_description_modes.py \
  tests/ace/tui/test_panel_tab_strip_tooltips.py \
  tests/ace/tui/artifacts_contract/test_contract_compiler.py \
  tests/ace/tui/artifacts_contract/test_pane_declarations.py \
  tests/main/test_artifact_pane.py \
  tests/test_config_schema_extensions.py \
  tests/ace/tui/test_artifacts_scaffold.py::test_subtab_strip_labels_and_accents_cover_all_panes
```

Finish with `just check`. If that lane escalates or reports unusual selection, use the
required `/sase_monitor` flow for `just check-full` and record unrelated pre-existing
gate failures precisely rather than weakening the gates.

## Constraints

- Do not blanket-update the PNG suite; `sase-r5` documents why unrelated renderer and
  fixture failures must be attributed node by node.
- Do not change the description feature merely to preserve stale images. The resolved
  default is intentionally `summary`, and the pane brief must remain visible for every
  built-in and provider pane.
- Do not close `sase-u6`, run its final Symvision cleanup, or mark its parent plan done
  in either child phase. The `parent_bead` link hands the completed child back to the
  waiting `sase-u6` land agent for those landing actions.
